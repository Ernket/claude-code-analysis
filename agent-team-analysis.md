# Agent Team 实现分析报告

## 1. 结论先行

这个项目里的“agent team”不是单纯的 `AgentTool` 多开几个 sub-agent，而是单独实现了一套 **swarm/team runtime**。它的核心组成可以概括为：

`TeamCreateTool + AgentTool(team_name + name) + utils/swarm/* + teammateMailbox + tasks + REPL hooks`

更准确地说，这个仓库里其实同时存在两套多 agent 机制：

1. **普通 sub-agent**
   - 入口：`src/tools/AgentTool/AgentTool.tsx`
   - 形态：一次性任务型 agent，偏“delegate / background worker”
   - 生命周期：跑完就返回结果，结果通过 task notification 回到主会话
   - 典型状态：`local_agent`

2. **team/swarm teammate**
   - 入口：`TeamCreateTool` 建队，然后仍由 `AgentTool` 触发 spawn
   - 形态：长期存活的 teammate，空闲时进入 idle，等待 leader 或 task list 派活
   - 生命周期：不是一次性 call-return，而是持续轮询 mailbox / task list
   - 典型状态：leader + teammate，围绕 team file、mailbox、task list 协作

所以，如果你问“这个项目怎么实现 agent team”，答案不是“AgentTool 起子进程”，而是：

- **用 `TeamCreateTool` 建立 team 元数据和 task list**
- **用 `AgentTool(name + team_name)` 分流到 teammate spawn**
- **用 `AsyncLocalStorage` 或 tmux/iTerm2 pane 承载 teammate 运行时**
- **用文件邮箱 `inboxes/*.json` 做 agent 间消息总线**
- **用 `.claude/tasks/{team}` 做协作任务队列**
- **用 `useInboxPoller` / `useSwarmInitialization` / `TeamsDialog` 把这一切接回 REPL UI**

---

## 2. 核心模块地图

### 2.1 关键入口

| 模块 | 作用 |
|---|---|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 创建 team、写 team file、初始化 task list、写入 `AppState.teamContext` |
| `src/tools/AgentTool/AgentTool.tsx` | 普通 sub-agent 与 team teammate 的统一入口；当 `team_name && name` 时走 teammate spawn |
| `src/tools/shared/spawnMultiAgent.ts` | 真正的 teammate spawn 实现，负责选择 in-process / tmux / iTerm2 backend |
| `src/tools/SendMessageTool/SendMessageTool.ts` | team 内部消息发送、shutdown / plan approval response 等协议消息 |
| `src/utils/swarm/*` | swarm 运行时核心：backend、identity、permission、runner、layout、cleanup |
| `src/utils/teammateMailbox.ts` | 文件邮箱协议与收发实现 |
| `src/utils/tasks.ts` | team 共享任务列表 |
| `src/hooks/useInboxPoller.ts` | leader/teammate 收件轮询，把 mailbox 消息注入对话流 |
| `src/hooks/useSwarmInitialization.ts` | 会话启动时初始化 teammate hooks 和上下文 |
| `src/components/teams/TeamsDialog.tsx` | 团队状态、关停、切换权限模式等 UI |

### 2.2 关键数据文件

| 路径 | 用途 |
|---|---|
| `~/.claude/teams/{team}/config.json` | team 配置和成员列表 |
| `~/.claude/teams/{team}/inboxes/{agent}.json` | agent inbox |
| `~/.claude/teams/{team}/permissions/` | permission request / resolved 持久化 |
| `~/.claude/tasks/{team}/` | team 共享任务列表 |

---

## 3. 总体架构判断

### 3.1 它不是 RPC/消息队列架构，而是“文件系统分布式”

这个实现最鲜明的特点是：**没有单独的 team server**。它主要靠以下机制拼装出 team runtime：

1. `config.json` 记录 team 成员、颜色、backend、mode、工作目录等元数据
2. `inboxes/*.json` 做 agent 间消息投递
3. `permissions/` 做权限请求和应答持久化
4. `tasks/{team}` 做共享任务列表
5. `AppState.teamContext` 做 leader 本地 UI 视图
6. `AsyncLocalStorage` 解决 in-process teammate 的身份隔离
7. `tmux` / `iTerm2` pane 解决 out-of-process teammate 的终端承载

换句话说，这个 team system 的核心不是“多模型调用”，而是 **多 agent runtime orchestration**。

### 3.2 这是一个“leader-centered”团队模型

这个 team 体系是标准的 **leader / teammate** 架构，不是 fully-peer mesh：

- leader 负责建队、派工、收消息、处理权限、看 UI
- teammate 主要通过 `SendMessage`、task list、idle notification 与 leader 协作
- teammate 之间可以互发消息，但系统默认仍以 leader 为主协调者
- team roster 是 **flat** 的，teammate 不能再 spawn teammate

`AgentTool.tsx` 里专门限制了这一点：

```ts
if (isTeammate() && teamName && name) {
  throw new Error(
    'Teammates cannot spawn other teammates — the team roster is flat. To spawn a subagent instead, omit the `name` parameter.'
  );
}
```

这说明作者明确不想把它做成递归 swarm。

---

## 4. 从 0 到 1：Team 是怎么创建出来的

### 4.1 `TeamCreateTool` 做了什么

`src/tools/TeamCreateTool/TeamCreateTool.ts` 并不负责真正跑 agent，它做的是 **建队初始化**：

1. 检查当前 leader 是否已经在一个 team 里
2. 生成唯一 team name
3. 生成 deterministic lead agent id：`team-lead@{team}`
4. 写入 team file
5. 重置并创建 task list 目录
6. 设置 leader 的 `teamContext`
7. 注册 session cleanup

核心代码：

```ts
const teamFile: TeamFile = {
  name: finalTeamName,
  description: _description,
  createdAt: Date.now(),
  leadAgentId,
  leadSessionId: getSessionId(),
  members: [
    {
      agentId: leadAgentId,
      name: TEAM_LEAD_NAME,
      agentType: leadAgentType,
      model: leadModel,
      joinedAt: Date.now(),
      tmuxPaneId: '',
      cwd: getCwd(),
      subscriptions: [],
    },
  ],
}

await writeTeamFileAsync(finalTeamName, teamFile)
await resetTaskList(taskListId)
await ensureTasksDir(taskListId)
setLeaderTeamName(sanitizeName(finalTeamName))
```

### 4.2 team file 结构

`src/utils/swarm/teamHelpers.ts` 定义的 `TeamFile` 大概长这样：

```ts
export type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    prompt?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

这说明 team file 既是：

- roster
- backend registry
- UI 状态来源
- teammate permission mode / active state 的共享真相源

### 4.3 team 和 task list 是绑定的

这个项目把 team 和 task list 设计成 1:1：

```ts
// TeamCreateTool
const taskListId = sanitizeName(finalTeamName)
await resetTaskList(taskListId)
await ensureTasksDir(taskListId)
setLeaderTeamName(sanitizeName(finalTeamName))
```

`src/utils/tasks.ts` 中 `getTaskListId()` 也明确把 teamName 作为 leader / teammate 的任务列表 id：

```ts
export function getTaskListId(): string {
  if (process.env.CLAUDE_CODE_TASK_LIST_ID) {
    return process.env.CLAUDE_CODE_TASK_LIST_ID
  }
  const teammateCtx = getTeammateContext()
  if (teammateCtx) {
    return teammateCtx.teamName
  }
  return getTeamName() || leaderTeamName || getSessionId()
}
```

这意味着：**agent team 的协作不是靠 leader 记忆分工，而是靠共享 task list 持久化分工。**

---

## 5. 真正的 teammate spawn 发生在哪里

### 5.1 `AgentTool` 是统一入口，但 team 会走特殊分支

`src/tools/AgentTool/AgentTool.tsx` 里真正把普通 sub-agent 和 team teammate 区分开：

```ts
const teamName = resolveTeamName({ team_name }, appState)

if (teamName && name) {
  const result = await spawnTeammate({
    name,
    prompt,
    description,
    team_name: teamName,
    use_splitpane: true,
    plan_mode_required: spawnMode === 'plan',
    model: model ?? agentDef?.model,
    agent_type: subagent_type,
    invokingRequestId: assistantMessage?.requestId
  }, toolUseContext)

  const spawnResult: TeammateSpawnedOutput = {
    status: 'teammate_spawned',
    prompt,
    ...result.data
  }
  return { data: spawnResult }
}
```

也就是说：

- `AgentTool(只有 prompt + subagent_type)` => 普通 sub-agent
- `AgentTool(team_name + name + prompt)` => team teammate

### 5.2 `spawnTeammate()` 再决定用哪个 backend

真正的 spawn 在 `src/tools/shared/spawnMultiAgent.ts`：

```ts
async function handleSpawn(input, context) {
  if (isInProcessEnabled()) {
    return handleSpawnInProcess(input, context)
  }

  try {
    await detectAndGetBackend()
  } catch (error) {
    if (getTeammateModeFromSnapshot() !== 'auto') {
      throw error
    }
    markInProcessFallback()
    return handleSpawnInProcess(input, context)
  }

  const useSplitPane = input.use_splitpane !== false
  if (useSplitPane) {
    return handleSpawnSplitPane(input, context)
  }
  return handleSpawnSeparateWindow(input, context)
}
```

这个分支非常关键，它说明 team teammate 支持三种运行方式：

1. **in-process**
   - 同进程运行
   - 用 `AsyncLocalStorage` 隔离身份
   - 默认在不在 tmux / iTerm2 环境时启用

2. **split pane**
   - 在 tmux / iTerm2 里开 pane
   - 用户可以直观看到 teammate pane

3. **separate window**
   - 较旧的 tmux window 模式

### 5.3 backend 自动选择策略

`src/utils/swarm/backends/registry.ts` 的逻辑很清楚：

```ts
export function isInProcessEnabled(): boolean {
  if (getIsNonInteractiveSession()) return true

  const mode = getTeammateMode()
  if (mode === 'in-process') return true
  if (mode === 'tmux') return false

  if (inProcessFallbackActive) return true

  const insideTmux = isInsideTmuxSync()
  const inITerm2 = isInITerm2()
  return !insideTmux && !inITerm2
}
```

可以总结为：

- 显式配置 `in-process` => 一定 in-process
- 显式配置 `tmux` => 一定 pane backend
- `auto` 模式下：
  - 在 tmux / iTerm2 中 => 优先 pane backend
  - 否则 => in-process
  - pane backend 不可用时 => fallback 到 in-process

这是一个很实用的工程决策：**有终端布局能力时给用户可视化 pane，没有时退化成同进程多 agent。**

---

## 6. 两条 spawn 路径：in-process 与 pane-based

## 6.1 in-process teammate

`handleSpawnInProcess()` 的关键步骤：

1. 生成 teammate name / id / color
2. 如果 `agent_type` 是 custom agent，则读取 agent definition
3. 调用 `spawnInProcessTeammate()`
4. 调用 `startInProcessTeammate()` 启动实际执行循环
5. 更新 `AppState.teamContext`
6. 将 member 写入 team file

关键代码：

```ts
const result = await spawnInProcessTeammate(config, context)

if (result.taskId && result.teammateContext && result.abortController) {
  startInProcessTeammate({
    identity: {
      agentId: teammateId,
      agentName: sanitizedName,
      teamName,
      color: teammateColor,
      planModeRequired: plan_mode_required ?? false,
      parentSessionId: result.teammateContext.parentSessionId,
    },
    taskId: result.taskId,
    prompt,
    description: input.description,
    model,
    agentDefinition,
    teammateContext: result.teammateContext,
    toolUseContext: { ...context, messages: [] },
    abortController: result.abortController,
    invokingRequestId: input.invokingRequestId,
  })
}
```

这里有两个重要点：

1. **不走 mailbox 发送初始 prompt**
   - in-process teammate 直接由 `startInProcessTeammate()` 启动
   - 代码里专门写了注释避免重复投递 prompt

2. **把父会话 `messages` 清空后再传入**
   - 避免把 leader 全量对话历史一直 pin 在子 agent 生命周期里
   - 这是一种内存与上下文污染控制

## 6.2 pane-based teammate

split pane / separate window 路径是另一种思路：

1. 生成 pane/window
2. 启动新的 Claude Code 进程
3. 通过 CLI 参数传 teammate 身份
4. 再把首条 prompt 写进 mailbox，让新进程启动后自行收取

核心代码片段：

```ts
const teammateArgs = [
  `--agent-id ${quote([teammateId])}`,
  `--agent-name ${quote([sanitizedName])}`,
  `--team-name ${quote([teamName])}`,
  `--agent-color ${quote([teammateColor])}`,
  `--parent-session-id ${quote([getSessionId()])}`,
  plan_mode_required ? '--plan-mode-required' : '',
  agent_type ? `--agent-type ${quote([agent_type])}` : '',
].filter(Boolean).join(' ')

await sendCommandToPane(paneId, spawnCommand, !insideTmux)

await writeToMailbox(
  sanitizedName,
  {
    from: TEAM_LEAD_NAME,
    text: prompt,
    timestamp: new Date().toISOString(),
  },
  teamName,
)
```

这说明 out-of-process teammate 的启动分两段：

- **启动进程**
- **投递首条任务**

这比把 prompt 直接塞到 CLI 启动参数里更灵活，因为 teammate 后续所有工作也都通过 mailbox 继续。

---

## 7. teammate 身份与上下文隔离

### 7.1 out-of-process：CLI args + dynamic team context

tmux / iTerm2 teammate 会带着下面这些参数启动：

- `--agent-id`
- `--agent-name`
- `--team-name`
- `--agent-color`
- `--parent-session-id`
- `--plan-mode-required`

之后 `src/utils/teammate.ts` 用 `dynamicTeamContext` 提供统一读取接口。

### 7.2 in-process：`AsyncLocalStorage`

in-process teammate 的关键实现不在 env var，而在 `src/utils/teammateContext.ts`：

```ts
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

export function runWithTeammateContext<T>(
  context: TeammateContext,
  fn: () => T,
): T {
  return teammateContextStorage.run(context, fn)
}
```

`TeammateContext` 结构：

```ts
export type TeammateContext = {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
  isInProcess: true
  abortController: AbortController
}
```

这一步非常关键，因为多个 teammate 在同一个 Node.js 进程里并发跑时，如果没有 `AsyncLocalStorage`，全局 `getAgentName()` / `getTeamName()` 一类 API 会串号。

### 7.3 会话启动时如何恢复 team context

`main.tsx` 会在首渲染前同步计算 `teamContext`：

```ts
teamContext: feature('KAIROS')
  ? assistantTeamContext ?? computeInitialTeamContext?.()
  : computeInitialTeamContext?.()
```

`src/utils/swarm/reconnection.ts` 会从 team file 里恢复：

```ts
export function computeInitialTeamContext(): AppState['teamContext'] | undefined {
  const context = getDynamicTeamContext()
  if (!context?.teamName || !context?.agentName) return undefined

  const teamFile = readTeamFile(teamName)
  return {
    teamName,
    teamFilePath,
    leadAgentId: teamFile.leadAgentId,
    selfAgentId: agentId,
    selfAgentName: agentName,
    isLeader,
    teammates: {},
  }
}
```

这就是为什么 tmux teammate 重启 / resume 后还能知道自己在哪个 team。

---

## 8. mailbox：整个 team 的消息总线

### 8.1 inbox 存储结构

`src/utils/teammateMailbox.ts` 明确写了：

```ts
export function getInboxPath(agentName: string, teamName?: string): string {
  const team = teamName || getTeamName() || 'default'
  const inboxDir = join(getTeamsDir(), safeTeam, 'inboxes')
  return join(inboxDir, `${safeAgentName}.json`)
}
```

即：

`~/.claude/teams/{team}/inboxes/{agent}.json`

写入时还用了 lockfile，避免并发写乱掉：

```ts
release = await lockfile.lock(inboxPath, {
  lockfilePath: lockFilePath,
  ...LOCK_OPTIONS,
})
```

### 8.2 mailbox 消息类型

这个文件里定义了大量协议消息：

- `idle_notification`
- `permission_request`
- `permission_response`
- `sandbox_permission_request`
- `sandbox_permission_response`
- `plan_approval_request`
- `plan_approval_response`
- `shutdown_request`
- `shutdown_approved`
- `shutdown_rejected`

这说明 mailbox 不是简单聊天，而是 team runtime 的统一 IPC 协议层。

### 8.3 普通 teammate 文本消息的格式

UI/模型侧接收到的 teammate message 会被包装成 XML：

```ts
return `<teammate-message teammate_id="${m.from}"${colorAttr}${summaryAttr}>
${m.text}
</teammate-message>`
```

也就是说，leader 实际“看到”的不是原始 inbox JSON，而是一个被注入到对话流里的伪 user message。

---

## 9. 消息是怎么回到 leader REPL 里的

### 9.1 `useInboxPoller()` 是消息接收中枢

`src/screens/REPL.tsx` 里挂了：

```ts
useSwarmInitialization(setAppState, initialMessages, { enabled: !isRemoteSession })
useInboxPoller({
  enabled: isAgentSwarmsEnabled(),
  isLoading,
  focusedInputDialog,
  onSubmitMessage: handleIncomingPrompt
})
```

`useInboxPoller()` 会：

1. 定时读 unread mailbox messages
2. 分类 permission / shutdown / plan approval / 普通消息
3. 普通消息转成 `<teammate-message>` XML
4. 如果主会话空闲，直接提交成新的 turn
5. 如果主会话正忙，排进 `AppState.inbox.messages`

这就是为什么 README/工具 prompt 里一直强调：

> Messages from teammates are automatically delivered to you.

### 9.2 leader 的 inbox poller 不只是收消息，还处理控制协议

`useInboxPoller.ts` 里除了普通消息，还直接处理了：

- leader 侧 permission request
- leader 侧 plan approval request
- leader 侧 shutdown approval
- teammate 侧 shutdown request
- teammate 侧 permission response

这是这个系统很“工程化”的一点：**leader 本身就是 team control plane。**

---

## 10. in-process teammate 的真正执行循环

这是整个实现里最重要的一段，位于 `src/utils/swarm/inProcessRunner.ts`。

### 10.1 teammate 不是跑一次，而是持续循环

`runInProcessTeammate()` 的主逻辑：

1. 构造 teammate system prompt
2. 注入 `TEAMMATE_SYSTEM_PROMPT_ADDENDUM`
3. 调 `runAgent()` 执行一次当前 prompt
4. 进入 idle
5. 发 idle notification 给 leader
6. 轮询 mailbox / task list / pending messages
7. 收到新消息后再跑下一轮

核心代码骨架：

```ts
while (!abortController.signal.aborted && !shouldExit) {
  for await (const message of runAgent({...})) {
    iterationMessages.push(message)
    allMessages.push(message)
    updateTaskState(...)
  }

  updateTaskState(taskId, task => ({ ...task, isIdle: true }), setAppState)

  await sendIdleNotification(identity.agentName, identity.color, identity.teamName, {
    idleReason: workWasAborted ? 'interrupted' : 'available',
    summary: getLastPeerDmSummary(allMessages),
  })

  const next = await waitForNextPromptOrShutdown(...)
  // next message / shutdown / task-claim
}
```

### 10.2 这个执行循环解决了什么问题

它不是普通 background task，而是解决了：

- teammate 长期在线
- 可以多轮对话
- 可以 idle 后被再次唤醒
- 可以在 leader 看 transcript 时插入 `pendingUserMessages`
- 可以在同一 teammate 上持续积累上下文 `allMessages`
- 可以自动 compact 历史，避免 token 爆炸

### 10.3 初始 prompt 与后续 prompt 的区别

初始 prompt 会被包装成：

```ts
const wrappedInitialPrompt = formatAsTeammateMessage(
  'team-lead',
  prompt,
  undefined,
  description,
)
```

后续 prompt 则来自三种来源：

1. mailbox 新消息
2. 用户在 teammate transcript 里直接发消息
3. team task list 中可认领的新任务

这个设计很像“agent actor loop”。

---

## 11. task list 如何驱动协作

### 11.1 teammate 会主动 claim task

in-process runner 在 idle 时不只是收消息，还会轮询 task list：

```ts
const taskPrompt = await tryClaimNextTask(taskListId, identity.agentName)
if (taskPrompt) {
  return {
    type: 'new_message',
    message: taskPrompt,
    from: 'task-list',
  }
}
```

`tryClaimNextTask()` 会：

1. `listTasks(taskListId)`
2. 找到 pending 且未 owner 且未 blocked 的任务
3. `claimTask(taskListId, task.id, agentName)`
4. 立刻 `updateTask(..., { status: 'in_progress' })`

这意味着 team 协作可以不全靠 leader 逐个 DM，teammate 能从共享 task list 自行拉活。

### 11.2 这个项目把 Task 和 Team 深度耦合

这套设计的本质是：

- **Team 负责 agent roster**
- **TaskList 负责 work queue**
- **Mailbox 负责即时通信**

三者组合起来，才是完整的 agent team。

---

## 12. 权限同步：这是实现里最复杂的一环

### 12.1 in-process teammate 的 `canUseTool` 被重写了

`createInProcessCanUseTool()` 做了两层处理：

1. 先跑正常 permission 检查
2. 如果结果是 `ask`
   - 优先尝试 leader UI 的 `ToolUseConfirmQueue`
   - 没有 bridge 时回退到 mailbox request/response

关键逻辑：

```ts
const result = await hasPermissionsToUseTool(...)
if (result.behavior !== 'ask') return result

const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
if (setToolUseConfirmQueue) {
  // 直接复用 leader 的 permission UI
}

// fallback: mailbox
const request = createPermissionRequest({...})
registerPermissionCallback({...})
void sendPermissionRequestViaMailbox(request)
```

### 12.2 leader 侧如何接住权限请求

`useInboxPoller.ts` 会识别 `permission_request`，再把它转成标准 `ToolUseConfirm`：

```ts
const entry: ToolUseConfirm = {
  tool,
  description: parsed.description,
  input: parsed.input,
  toolUseID: parsed.tool_use_id,
  workerBadge: {
    name: parsed.agent_id,
    color: 'cyan',
  },
  onAllow(updatedInput, permissionUpdates) {
    void sendPermissionResponseViaMailbox(...)
  },
  onAbort() {
    void sendPermissionResponseViaMailbox(...)
  },
}
```

这一步的价值很大：**worker 的权限弹窗不需要再造一套 UI，直接借 leader 原有权限系统。**

### 12.3 还有 sandbox permission

除了普通工具权限，网络 sandbox 权限也走 mailbox 协议：

- `sandbox_permission_request`
- `sandbox_permission_response`

说明这个 team system 对工具权限和网络权限都做了统一转发。

---

## 13. plan mode / approval 机制

### 13.1 teammate 支持 plan mode required

spawn teammate 时可以带：

- `mode: "plan"`
- `plan_mode_required: true`

相关状态会进 teammate identity / task state。

### 13.2 协议层面有 plan approval request/response

`teammateMailbox.ts` 定义了：

```ts
type: 'plan_approval_request'
type: 'plan_approval_response'
```

其中 request 还会附带：

- `planFilePath`
- `planContent`
- `requestId`

### 13.3 但当前实现里 leader 侧是自动批准

`useInboxPoller.ts` 有一个很值得注意的行为：

```ts
if (planApprovalRequests.length > 0 && isTeamLead(currentAppState.teamContext)) {
  // auto-approving
  const approvalResponse = {
    type: 'plan_approval_response',
    requestId: parsed.requestId,
    approved: true,
    permissionMode: modeToInherit,
  }
  void writeToMailbox(...)
}
```

也就是说：

- 协议设计上支持“leader 审核 teammate plan”
- 当前 leader inbox poller 实现却是 **自动批准**
- 并且会把 leader 当前 permission mode 继承给 teammate

这是一个很重要的实现细节，说明这里的“plan approval”更像 runtime gate，而不是真正的人工 review 流程。

---

## 14. shutdown / idle / lifecycle

### 14.1 teammate 默认不是结束，而是 idle

`initializeTeammateHooks()` 给 process-based teammate 注册了 Stop hook：

```ts
const notification = createIdleNotification(agentName, {
  idleReason: 'available',
  summary: getLastPeerDmSummary(messages),
})
await writeToMailbox(leadAgentName, { ... })
```

而 in-process teammate 则由 `inProcessRunner.ts` 直接发 idle notification。

这说明：

- process-based teammate：Stop hook 报 idle
- in-process teammate：runner loop 报 idle

两条实现不同，但行为统一。

### 14.2 graceful shutdown 是协议式，不是直接 kill

`SendMessageTool` 和 `InProcessBackend` 都支持 `shutdown_request` / `shutdown_approved` / `shutdown_rejected`。

对于 in-process teammate，`terminate()` 会：

1. 往 teammate mailbox 里发 `shutdown_request`
2. 把 task 标记为 `shutdownRequested`
3. 等 teammate 自己处理退出

如果批准了：

- leader 侧 `useInboxPoller` 会移除 roster
- 对 pane-based teammate 还会 kill pane
- 同时会 unassign tasks

这是一个比较完整的 graceful lifecycle。

### 14.3 TeamDelete 不会粗暴清空

`src/tools/TeamDeleteTool/TeamDeleteTool.ts` 在清理前会先检查 active member：

```ts
if (activeMembers.length > 0) {
  return {
    data: {
      success: false,
      message: `Cannot cleanup team with ${activeMembers.length} active member(s)...`,
    },
  }
}
```

也就是说 team cleanup 不是无脑删目录，而是要求先把 teammate 收干净。

另外，`init.ts` 还注册了 session cleanup，防止 team 遗留在磁盘上。

---

## 15. prompt 层是怎么约束 team 行为的

这一层非常重要，因为 runtime 只是“让多 agent 能活着协作”，真正让它们“像一个 team”还要靠 prompt。

## 15.1 coordinator mode system prompt

`src/coordinator/coordinatorMode.ts` 明确把主 agent 设定成协调者：

```ts
You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
```

这个 prompt 还明确要求：

- workers 异步并行是核心能力
- coordinator 不要替 worker 猜结果
- research 后必须先 synthesize，再发 follow-up prompt

这部分其实决定了“多 agent 团队”在模型侧的工作范式。

## 15.2 teammate system prompt addendum

`src/utils/swarm/teammatePromptAddendum.ts` 很短，但非常关键：

```ts
export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM = `
# Agent Teammate Communication

IMPORTANT: You are running as an agent in a team. To communicate with anyone on your team:
- Use the SendMessage tool with \`to: "<name>"\`
- Use the SendMessage tool with \`to: "*"\` sparingly

Just writing a response in text is not visible to others on your team - you MUST use the SendMessage tool.
`
```

这等于硬性告诉 teammate：

- 文本回答不会自动广播给其他 agent
- 你必须显式调用 `SendMessage`

如果没有这段 prompt，team runtime 再完整，模型也会经常“以为自己说话别人能听见”。

## 15.3 `TeamCreateTool` prompt

`src/tools/TeamCreateTool/prompt.ts` 里对 team workflow 的描述非常详细，重点包括：

- 什么时候应该主动建队
- team 与 task list 1:1
- teammate 进入 idle 是正常行为
- 不要手动检查 inbox，消息会自动投递
- teammate 应该通过 task list 协作

这里面有一句非常关键：

> Teams have a 1:1 correspondence with task lists (Team = TaskList).

这句基本就是这套实现的设计核心。

## 15.4 `SendMessage` tool prompt

`src/tools/SendMessageTool/prompt.ts` 明确规定：

- `to: "researcher"` 是发给 teammate name
- `to: "*"` 是广播
- 纯文本输出对其他 agent 不可见
- 协议响应如 `shutdown_response` / `plan_approval_response` 也通过这个 tool 发

它实际上承担了“告诉模型如何正确使用团队消息协议”的职责。

---

## 16. 这个实现最值得注意的工程设计

## 16.1 优点

### 优点 1：同一套 `AgentTool` 同时支撑普通 sub-agent 和 team teammate

这减少了模型侧工具面复杂度。用户/模型只要会用 `AgentTool`，加上 `name + team_name` 就能切进 team 模式。

### 优点 2：backend 可切换

同一套 teammate 语义既能：

- in-process 跑
- 也能在 tmux/iTerm2 里可视化跑

这让 UX 和 headless/CLI 环境都能兼容。

### 优点 3：消息、权限、任务三条通道分离

- mailbox：即时消息与控制协议
- task list：结构化任务协作
- team file：成员与状态元数据

这比把一切都塞进 prompt 更稳。

### 优点 4：in-process 用 `AsyncLocalStorage` 做身份隔离

这是正确解法。否则多 teammate 并发执行时，`getAgentName()`、`getTeamName()` 这类全局读取一定会串。

### 优点 5：leader 复用原权限 UI

权限同步没有新造一套复杂控制平面，而是把 worker request 接到 leader 已有的 permission dialog。

---

## 16.2 缺点 / 风险点

### 风险 1：大量依赖文件轮询

这套 team system 的实时性建立在：

- `readUnreadMessages`
- `markMessagesAsRead`
- `readPendingPermissions`
- `task list polling`

本质上是 polling + file lock，而不是 event-driven queue。规模大了会有：

- IO 抖动
- 锁竞争
- 状态延迟

### 风险 2：协议分散在多处

message types 虽然都定义在 `teammateMailbox.ts`，但处理逻辑分散在：

- `useInboxPoller.ts`
- `inProcessRunner.ts`
- `SendMessageTool.ts`
- `InProcessBackend.ts`
- `permissionSync.ts`

维护成本不低。

### 风险 3：plan approval 现在是“协议上支持，运行时自动批”

这会让人误以为真有 leader 审批环节，但实际 leader poller 里直接 auto-approve 了。名字上有“approval”，行为上更像 state transition。

### 风险 4：out-of-process teammate 在 UI task 层被归一成 `in_process_teammate`

`spawnMultiAgent.ts` 里 `registerOutOfProcessTeammateTask()` 也注册成 `in_process_teammate` 风格状态，这虽然统一了 UI，但命名语义已经不准确，后面容易让维护者误判。

### 风险 5：team roster 是 flat，扩展多层 team 会受限

现在明确禁止 teammate 再 spawn teammate。如果以后想做 hierarchy / subtree swarm，需要重构 roster 和 routing。

---

## 17. 我认为最核心的 5 个源码结论

1. **Team 不是 AgentTool 的附属功能，而是一套单独 runtime**
   - `TeamCreateTool`、`teamHelpers`、`mailbox`、`useInboxPoller`、`tasks` 都是独立层

2. **AgentTool 只是入口复用**
   - 真正进入 team 的条件是 `team_name && name`

3. **这个实现的本质是 leader-centered swarm**
   - coordinator/leader 是 control plane

4. **team 协作依赖文件系统 IPC**
   - team file、inbox、permissions、tasks 四类目录共同组成运行时

5. **in-process teammate 才是最完整、最先进的一条路径**
   - `AsyncLocalStorage` + `runInProcessTeammate()` 这套逻辑比 pane spawn 更像真正的 agent runtime

---

## 18. 建议的阅读顺序

如果你要继续深挖，我建议按这个顺序读：

1. `src/tools/TeamCreateTool/TeamCreateTool.ts`
2. `src/tools/AgentTool/AgentTool.tsx`
3. `src/tools/shared/spawnMultiAgent.ts`
4. `src/utils/swarm/backends/registry.ts`
5. `src/utils/swarm/spawnInProcess.ts`
6. `src/utils/swarm/inProcessRunner.ts`
7. `src/utils/teammateMailbox.ts`
8. `src/hooks/useInboxPoller.ts`
9. `src/utils/tasks.ts`
10. `src/components/teams/TeamsDialog.tsx`

按这个顺序最容易把“建队 -> spawn -> 运行 -> 收消息 -> 派任务 -> 清理”的完整链路串起来。

---

## 19. 一句话总结

这个项目的 agent team 实现，本质上是一个 **建立在文件系统 IPC、共享 task list、leader 协调和可切换 backend 之上的 swarm runtime**；`AgentTool` 只是触发入口，真正让它成为“team”的是 `swarm/* + mailbox + tasks + REPL hooks + prompt discipline` 这一整套组合。
