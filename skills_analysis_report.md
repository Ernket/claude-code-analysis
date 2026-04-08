# Claude Code `skills` 功能实现分析报告

## 1. 结论摘要

这个项目里的 `skills` 本质上不是一套单独的执行引擎，而是建立在统一 `Command`/`PromptCommand` 抽象之上的一类“可延迟展开的 Prompt 资产”。

核心结论如下：

1. `skill` 最终都会被编译成一个 `type: 'prompt'` 的 `Command` 对象，而不是特殊 AST 或 DSL。入口在 `src/skills/loadSkillsDir.ts:270` 的 `createSkillCommand(...)`。
2. 技能正文不会在启动时整体塞进模型上下文，而是只暴露 `name`、`description`、`when_to_use` 等前置信息；真正的 `SKILL.md` 内容只在调用时展开。这是这套设计最关键的节流点。
3. 技能来源很多，但最终都收敛到同一套命令接口：
   - 本地 `.claude/skills/<name>/SKILL.md`
   - 兼容旧格式 `.claude/commands`
   - 插件技能
   - bundled 内置技能
   - MCP prompt/skill
   - 实验性的远端 canonical skill
4. 模型侧并不是直接“知道所有技能正文”，而是通过 `SkillTool` 的提示词、`skill_listing` 附件、`skill_discovery` 附件、以及系统提示里的技能说明来决定何时调用技能。
5. 技能调用后，会被转成一条或多条“隐藏但模型可见”的 meta user message 注入对话流；后续模型其实是在执行技能展开后的 prompt。
6. 技能系统还做了几层增强：
   - 参数替换
   - `${CLAUDE_SKILL_DIR}` / `${CLAUDE_SESSION_ID}` 变量替换
   - `!` / ````!` 内嵌 shell 执行
   - `context: fork` 子 agent 执行
   - `paths` 条件激活
   - 文件操作触发的动态技能发现
   - compact/resume 后的技能恢复

一句话概括：**它把 skill 实现成了“可发现、可筛选、可延迟展开、可被工具调用的 Prompt Command”，而不是把 skill 设计成额外的一套 runtime。**

---

## 2. 总体架构

### 2.1 核心对象模型

`skills` 的底层抽象直接复用 `PromptCommand`。

参考 `src/types/command.ts:24`:

```ts
export type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  hooks?: HooksSettings
  skillRoot?: string
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
  paths?: string[]
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>
}
```

这个定义已经说明了 `skills` 的几个关键点：

- 技能本质是 `prompt` 命令。
- 技能的真正执行入口是 `getPromptForCommand(...)`。
- 技能可以声明额外工具权限、模型覆盖、执行上下文、agent 类型、条件路径等。

### 2.2 高层调用链

技能系统的主链路可以简化成下面这张图：

```text
SKILL.md / plugin / bundled / MCP
        -> 解析 frontmatter
        -> createSkillCommand()
        -> 注册进 commands registry
        -> getSkillToolCommands() / getCommands()
        -> 通过 system prompt + attachment 暴露给模型
        -> 模型调用 SkillTool(skill, args)
        -> processPromptSlashCommand()
        -> command.getPromptForCommand()
        -> 生成 meta user messages 注入对话
        -> Claude 按技能展开后的 prompt 继续执行
```

### 2.3 关键模块分工

| 模块 | 作用 |
| --- | --- |
| `src/skills/loadSkillsDir.ts` | 本地 skill/legacy command 的加载、解析、动态发现、条件激活 |
| `src/commands.ts` | 把 skill 与 bundled/plugin/workflow/builtin 命令合并成统一注册表 |
| `src/tools/SkillTool/SkillTool.ts` | 模型调用 skill 的工具入口，处理校验、权限、inline/fork/remote 分支 |
| `src/tools/SkillTool/prompt.ts` | SkillTool 的系统提示词、技能列表预算控制 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 真正把 skill 展开成对话消息 |
| `src/utils/promptShellExecution.ts` | 支持 skill markdown 中的 `!` 内嵌 shell |
| `src/services/compact/compact.ts` + `src/bootstrap/state.ts` | compact/resume 期间保留已调用技能内容 |
| `src/utils/attachments.ts` | 向模型发送 `skill_listing` / `dynamic_skill` / `invoked_skills` 等附件 |

---

## 3. Skill 的定义格式

### 3.1 目录约定

本地新式 skill 的目录格式是：

```text
.claude/skills/<skill-name>/SKILL.md
```

对应实现见 `src/skills/loadSkillsDir.ts:367` 附近的 `loadSkillsFromSkillsDir(...)`。它明确要求目录形式，且文件名固定为 `SKILL.md`。

旧格式兼容路径是：

```text
.claude/commands/**/*.md
或
.claude/commands/<dir>/SKILL.md
```

对应实现见 `src/skills/loadSkillsDir.ts:568` 的 `loadSkillsFromCommandsDir(...)`。

### 3.2 Frontmatter 语义

frontmatter 的通用定义在 `src/utils/frontmatterParser.ts:10`。

它支持的技能相关字段包括：

| 字段 | 作用 |
| --- | --- |
| `name` | 展示名 |
| `description` | 技能描述 |
| `allowed-tools` | 调用该技能时追加允许的工具权限 |
| `when_to_use` | 触发条件，主要给模型做决策 |
| `argument-hint` | 参数提示 |
| `arguments` | 命名参数列表 |
| `model` | 技能级模型覆盖 |
| `effort` | 思考强度覆盖 |
| `user-invocable` | 用户能否直接 `/skill-name` 调用 |
| `disable-model-invocation` | 模型是否禁止通过 SkillTool 调用 |
| `hooks` | 调用时注册 hook |
| `context` | `inline` 或 `fork` |
| `agent` | fork 模式下使用的 agent 类型 |
| `paths` | 条件激活路径模式 |
| `shell` | skill markdown 中 `!` 命令用 `bash` 还是 `powershell` |

### 3.3 Frontmatter 解析特点

`parseFrontmatter(...)` 在 `src/utils/frontmatterParser.ts:130`，它做了几件很实用的事情：

1. 支持 YAML frontmatter。
2. 如果 YAML 因 glob 或特殊字符解析失败，会先自动给“有问题的值”加引号再重试。
3. `paths` 支持逗号分隔和 YAML 数组两种写法，并支持 `{ts,tsx}` 这类 brace expansion。

相关代码：

```ts
export function splitPathInFrontmatter(input: string | string[]): string[] {
  if (Array.isArray(input)) {
    return input.flatMap(splitPathInFrontmatter)
  }
  ...
  return parts
    .filter(p => p.length > 0)
    .flatMap(pattern => expandBraces(pattern))
}
```

### 3.4 从 Markdown 提取 skill 元数据

真正把 frontmatter 变成 skill 元数据的是 `parseSkillFrontmatterFields(...)`，位置在 `src/skills/loadSkillsDir.ts:185`。

它会：

- 把 `description` 规范化；如果没写，就从 markdown 第一行提取摘要。
- 把 `allowed-tools` 解析成工具白名单数组。
- 把 `arguments` 解析成命名参数数组。
- 读取 `when_to_use`、`model`、`effort`、`hooks`、`context`、`agent`、`shell` 等字段。

核心片段如下：

```ts
export function parseSkillFrontmatterFields(...) {
  const validatedDescription = coerceDescriptionToString(
    frontmatter.description,
    resolvedName,
  )
  const description =
    validatedDescription ??
    extractDescriptionFromMarkdown(markdownContent, descriptionFallbackLabel)

  const userInvocable =
    frontmatter['user-invocable'] === undefined
      ? true
      : parseBooleanFrontmatter(frontmatter['user-invocable'])

  return {
    displayName:
      frontmatter.name != null ? String(frontmatter.name) : undefined,
    description,
    allowedTools: parseSlashCommandToolsFromFrontmatter(
      frontmatter['allowed-tools'],
    ),
    argumentHint: ...,
    argumentNames: parseArgumentNames(...),
    whenToUse: frontmatter.when_to_use as string | undefined,
    version: frontmatter.version as string | undefined,
    model,
    disableModelInvocation: parseBooleanFrontmatter(
      frontmatter['disable-model-invocation'],
    ),
    userInvocable,
    hooks: parseHooksFromFrontmatter(frontmatter, resolvedName),
    executionContext: frontmatter.context === 'fork' ? 'fork' : undefined,
    agent: frontmatter.agent as string | undefined,
    effort,
    shell: parseShellFrontmatter(frontmatter.shell, resolvedName),
  }
}
```

---

## 4. 技能是如何被加载和注册的

## 4.1 `createSkillCommand()`：统一编译入口

`src/skills/loadSkillsDir.ts:270` 的 `createSkillCommand(...)` 是整个技能系统最重要的函数之一。

它把“一个 skill 的静态定义”编译成统一的 `Command` 对象：

```ts
export function createSkillCommand({...}): Command {
  return {
    type: 'prompt',
    name: skillName,
    description,
    allowedTools,
    argumentHint,
    argNames: argumentNames.length > 0 ? argumentNames : undefined,
    whenToUse,
    version,
    model,
    disableModelInvocation,
    userInvocable,
    context: executionContext,
    agent,
    effort,
    paths,
    contentLength: markdownContent.length,
    isHidden: !userInvocable,
    progressMessage: 'running',
    source,
    loadedFrom,
    hooks,
    skillRoot: baseDir,
    async getPromptForCommand(args, toolUseContext) {
      let finalContent = baseDir
        ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
        : markdownContent

      finalContent = substituteArguments(
        finalContent,
        args,
        true,
        argumentNames,
      )

      if (baseDir) {
        const skillDir =
          process.platform === 'win32' ? baseDir.replace(/\\/g, '/') : baseDir
        finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
      }

      finalContent = finalContent.replace(
        /\$\{CLAUDE_SESSION_ID\}/g,
        getSessionId(),
      )

      if (loadedFrom !== 'mcp') {
        finalContent = await executeShellCommandsInPrompt(...)
      }

      return [{ type: 'text', text: finalContent }]
    },
  }
}
```

这段代码能看出三个非常重要的设计选择：

1. **技能正文延迟展开**：注册时只保存 metadata 和 `getPromptForCommand()`，不立即把正文塞进上下文。
2. **资源目录前缀注入**：所有基于目录的 skill 都会自动加上 `Base directory for this skill: ...`，便于模型后续通过 Read/Grep 访问相对文件。
3. **调用时二次求值**：参数替换、session 变量替换、嵌入 shell 执行都在真正调用时发生。

## 4.2 本地技能目录加载

`src/skills/loadSkillsDir.ts:638` 的 `getSkillDirCommands(...)` 负责扫描以下来源：

- policy settings 下的 `.claude/skills`
- 用户目录 `~/.claude/skills`
- 当前项目到 git root 之间每层目录的 `.claude/skills`
- `--add-dir` 额外目录里的 `.claude/skills`
- legacy `.claude/commands`

它的实现特点：

1. **并行加载多个来源**。
2. **通过 `realpath` 去重**，避免同一物理文件被符号链接或重复父目录扫描两次。
3. **把带 `paths` 的技能分离成 conditional skill**，先不直接暴露给模型。

相关逻辑：

```ts
const [
  managedSkills,
  userSkills,
  projectSkillsNested,
  additionalSkillsNested,
  legacyCommands,
] = await Promise.all([...])

const allSkillsWithPaths = [
  ...managedSkills,
  ...userSkills,
  ...projectSkillsNested.flat(),
  ...additionalSkillsNested.flat(),
  ...legacyCommands,
]

const fileIds = await Promise.all(
  allSkillsWithPaths.map(({ skill, filePath }) =>
    skill.type === 'prompt'
      ? getFileIdentity(filePath)
      : Promise.resolve(null),
  ),
)
```

## 4.3 兼容 legacy `/commands`

旧格式命令目录同样可以承载 skill。

`loadSkillsFromCommandsDir(...)` 会：

- 扫描 `.claude/commands`
- 支持普通 `.md`
- 如果某目录里存在 `SKILL.md`，则优先把目录视作 skill
- 用 `namespace:subname` 规则构造命名空间式命令名

这说明作者并没有把新旧格式拆成两套系统，而是通过同一个 `createSkillCommand(...)` 入口兼容过去的命令资产。

## 4.4 统一并入命令系统

`src/commands.ts:353` 开始的 `getSkills(...)` 会把 skill 源汇总成：

- `skillDirCommands`
- `pluginSkills`
- `bundledSkills`
- `builtinPluginSkills`

然后 `loadAllCommands(...)` 在 `src/commands.ts:449` 把它们与 workflow、plugin command、builtin command 合并：

```ts
return [
  ...bundledSkills,
  ...builtinPluginSkills,
  ...skillDirCommands,
  ...workflowCommands,
  ...pluginCommands,
  ...pluginSkills,
  ...COMMANDS(),
]
```

这里的关键是：**skill 并不是单独挂在另一棵树上，而是被并入统一的 commands registry**。

## 4.5 给模型看的 skill 列表和给 UI 看的 skill 列表不同

项目里有两个过滤函数：

- `getSkillToolCommands(...)`，位置 `src/commands.ts:563`
- `getSlashCommandToolSkills(...)`，位置 `src/commands.ts:586`

前者给 `SkillTool` 和模型用，后者给 slash-command/技能 UI 使用。

模型可调用 skill 的过滤条件更强调：

- `type === 'prompt'`
- `!disableModelInvocation`
- `source !== 'builtin'`
- bundled/skills/legacy 总是可见
- plugin/MCP skill 必须有显式描述或 `when_to_use`

这意味着项目显式区分了：

- “用户界面里想展示哪些技能”
- “模型运行时可自动决策调用哪些技能”

---

## 5. 模型是怎么知道有哪些 skills 的

## 5.1 SkillTool 自身提示词

`src/tools/SkillTool/prompt.ts:173` 的 `getPrompt()` 是模型调用 skill 的工具说明：

```ts
export const getPrompt = memoize(async (_cwd: string): Promise<string> => {
  return `Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match.
...
Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT:
  invoke the relevant Skill tool BEFORE generating any other response about the task
- NEVER mention a skill without actually calling this tool
- Do not use this tool for built-in CLI commands
- If you see a <command-name> tag in the current conversation turn, the skill
  has ALREADY been loaded - follow the instructions directly instead of calling
  this tool again
`
})
```

这个 prompt 很强势，核心约束有两条：

1. 命中技能时，必须先调 SkillTool，再回答。
2. 不能“口头提一下技能”，而不真正调用它。

## 5.2 system prompt 会提醒模型 slash command 就是 skill

`src/constants/prompts.ts:383` 还有一条显式提醒：

```ts
/<skill-name> (e.g., /commit) is shorthand for users to invoke a user-invocable skill.
When executed, the skill gets expanded to a full prompt.
Use the Skill tool to execute them.
IMPORTANT: Only use SkillTool for skills listed in its user-invocable skills section.
```

也就是说，系统提示层面已经把 `/commit` 这类 slash command 和 “skill” 进行了语义统一。

## 5.3 skill 列表通过 attachment 注入，而不是写死到系统 prompt

`src/utils/attachments.ts:2661` 的 `getSkillListingAttachments(...)` 会动态构造 `skill_listing` 附件。

它先取：

- 本地/插件/bundled 技能：`getSkillToolCommands(cwd)`
- MCP 技能：`getMcpSkillCommands(appState.mcp.commands)`

然后拼接、去重、预算裁剪后发给模型。

核心代码：

```ts
const localCommands = await getSkillToolCommands(cwd)
const mcpSkills = getMcpSkillCommands(
  toolUseContext.getAppState().mcp.commands,
)

const content = formatCommandsWithinBudget(newSkills, contextWindowTokens)

return [
  {
    type: 'skill_listing',
    content,
    skillCount: newSkills.length,
    isInitial,
  },
]
```

## 5.4 技能列表有严格预算控制

`src/tools/SkillTool/prompt.ts` 对 skill listing 预算做得非常细：

- 预算默认为上下文窗口的 `1%`
- `MAX_LISTING_DESC_CHARS = 250`
- bundled skill 的描述尽量保留完整
- 非 bundled skill 在预算不足时会截断描述，极端情况下只保留名字

核心逻辑见 `formatCommandsWithinBudget(...)`，位置 `src/tools/SkillTool/prompt.ts:70`。

这再次说明：**skill 的设计目标不是把正文预加载到模型，而是让模型先看到足够短的“技能索引”再按需展开。**

## 5.5 还有实验性的 `skill_discovery`

`src/constants/prompts.ts:338` 里还有一层引导：

```ts
Relevant skills are automatically surfaced each turn as "Skills relevant to your task:"
reminders. If you're about to do something those don't cover ... call
DiscoverSkills with a specific description of what you're doing.
```

这说明这个项目已经不满足于静态 skill listing，而是开始做“基于当前任务的 skill 检索”。

---

## 6. 技能真正被调用时发生了什么

## 6.1 SkillTool 是模型调用技能的统一入口

`src/tools/SkillTool/SkillTool.ts:331`：

```ts
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME,
  searchHint: 'invoke a slash-command skill',
  prompt: async () => getPrompt(getProjectRoot()),
  ...
})
```

输入非常简单：

```ts
{
  skill: string
  args?: string
}
```

## 6.2 调用前先校验 skill 是否存在、是否允许模型调用

`validateInput(...)` 在 `src/tools/SkillTool/SkillTool.ts:354`：

- 去掉开头的 `/`
- 支持实验性远端 canonical skill
- 从 `getAllCommands(context)` 查找 skill
- 确认命令存在
- 确认不是 `disableModelInvocation`
- 确认是 `prompt` 类型

这意味着 skill 并没有另一套查找器，而是从统一 command registry 里查。

## 6.3 权限模型

`checkPermissions(...)` 在 `src/tools/SkillTool/SkillTool.ts:432`，其逻辑是：

1. 先检查 deny rule。
2. 再检查 allow rule。
3. 如果 skill 只有“安全属性”，自动放行。
4. 否则向用户请求权限，并给出两种规则建议：
   - 精确 skill 许可
   - `prefix:*` 形式的前缀许可

这里还有一个值得注意的点：`SAFE_SKILL_PROPERTIES` 在 `src/tools/SkillTool/SkillTool.ts:875` 明确列出哪些字段算“安全”。

这说明 skill 调用权限不是粗粒度地“一刀切”，而是看 skill 是否引入了额外风险属性。

## 6.4 `call()` 的三条执行分支

`SkillTool.call(...)` 在 `src/tools/SkillTool/SkillTool.ts:580`，实际分三种情况：

1. **远端 canonical skill**
2. **forked skill**
3. **普通 inline skill**

### 6.4.1 inline skill

普通分支最终会调用：

```ts
const processedCommand = await processPromptSlashCommand(
  commandName,
  args || '',
  commands,
  context,
)
```

然后把 skill 展开后的消息作为 `newMessages` 回注入主对话。

### 6.4.2 forked skill

如果 skill frontmatter 指定了 `context: fork`，就进入 `executeForkedSkill(...)`，位置 `src/tools/SkillTool/SkillTool.ts:122`。

它会：

- 调 `prepareForkedCommandContext(...)`
- 把 skill prompt 作为子 agent 的初始 user message
- 应用 `allowedTools`
- 选择 `command.agent` 指定的 agent 类型
- 调 `runAgent(...)`
- 汇总结果文本返回

### 6.4.3 remote skill

实验性远端 skill 在 `executeRemoteSkill(...)`，位置 `src/tools/SkillTool/SkillTool.ts:969`。

这里会：

- 从 discovery 结果里取远端 URL
- 下载 skill
- 剥掉 YAML frontmatter
- 注入 `Base directory for this skill`
- 记录到 `addInvokedSkill(...)`
- 直接以 meta user message 形式注入对话

---

## 7. Skill 是怎么被展开成 Prompt 的

## 7.1 真正展开位置：`getMessagesForPromptSlashCommand(...)`

真正把 skill 变成消息的是 `src/utils/processUserInput/processSlashCommand.tsx:827` 的 `getMessagesForPromptSlashCommand(...)`。

它做的事非常关键：

```ts
const result = await command.getPromptForCommand(args, context)

if (command.hooks && hooksAllowedForThisSkill) {
  registerSkillHooks(...)
}

const skillPath = command.source ? `${command.source}:${command.name}` : command.name
const skillContent = result
  .filter((b): b is TextBlockParam => b.type === 'text')
  .map(b => b.text)
  .join('\n\n')
addInvokedSkill(command.name, skillPath, skillContent, getAgentContext()?.agentId ?? null)

const messages = [
  createUserMessage({ content: metadata, uuid }),
  createUserMessage({ content: mainMessageContent, isMeta: true }),
  ...attachmentMessages,
  createAttachmentMessage({
    type: 'command_permissions',
    allowedTools: additionalAllowedTools,
    model: command.model
  }),
]
```

这段逻辑说明：

1. skill 展开结果本质上是一条 **meta user message**。
2. 展开后的文本会被存进 `invokedSkills`，以便 compact/resume 后恢复。
3. 附带的 `allowed-tools` 和 `model` 不直接混在 prompt 正文里，而是通过 attachment/上下文修饰器生效。

## 7.2 为什么是“meta user message”

因为作者需要满足两个条件：

1. 让模型完整看到 skill 说明。
2. 不把 skill 展开过程当成普通用户自然语言输入去显示。

所以它把 skill 正文包装成 `isMeta: true` 的 user message。这样对于模型它仍是 prompt 内容，但对 UI 展示更可控。

## 7.3 技能加载元信息

skill 被展开前，还会生成一条元信息消息，格式由 `formatCommandLoadingMetadata(...)`（`src/utils/processUserInput/processSlashCommand.tsx:803`）决定。

它会注入：

- `<command-message>...`
- `<command-name>...`
- `<skill-format>true</skill-format>`

这有两个作用：

1. 让 UI 和消息渲染层知道这是一个 skill。
2. 让 SkillTool 自己识别“这一轮 skill 已经加载过”，防止重复调用。

---

## 8. Skill Prompt 在展开时做了哪些二次处理

## 8.1 参数替换

参数替换实现在 `src/utils/argumentSubstitution.ts`。

支持以下语法：

- `$ARGUMENTS`
- `$ARGUMENTS[0]`
- `$0`, `$1`
- `$name` 形式的命名参数

核心代码：

```ts
content = content.replace(/\$ARGUMENTS\[(\d+)\]/g, ...)
content = content.replace(/\$(\d+)(?!\w)/g, ...)
content = content.replaceAll('$ARGUMENTS', args)
```

如果 skill 没写任何参数占位符，又传了参数，还会自动在尾部追加：

```text
ARGUMENTS: ...
```

## 8.2 环境变量替换

`createSkillCommand(...)` 会替换：

- `${CLAUDE_SKILL_DIR}`
- `${CLAUDE_SESSION_ID}`

插件 skill 还会额外替换：

- `${CLAUDE_PLUGIN_ROOT}`
- `${CLAUDE_PLUGIN_DATA}`
- `${user_config.X}`

对应逻辑见 `src/utils/plugins/loadPluginCommands.ts:340` 之后。

## 8.3 skill markdown 内嵌 shell 执行

这是技能系统里非常特别的一层能力。

`src/utils/promptShellExecution.ts:69` 的 `executeShellCommandsInPrompt(...)` 支持两种语法：

```text
!`command`
```

和

````markdown
```! 
command
```
````

它会：

1. 选择 `BashTool` 或 `PowerShellTool`
2. 先做权限校验
3. 真正执行命令
4. 把输出替换回 skill prompt 文本

核心代码：

```ts
const shellTool: PromptShellTool =
  shell === 'powershell' && isPowerShellToolEnabled()
    ? getPowerShellTool()
    : BashTool

const permissionResult = await hasPermissionsToUseTool(
  shellTool,
  { command },
  context,
  createAssistantMessage({ content: [] }),
  '',
)

const { data } = await shellTool.call({ command }, context)
result = result.replace(match[0], () => output)
```

这意味着 skill markdown 不是纯静态文档，而是一种 **带有限度求值能力的 prompt 模板**。

## 8.4 MCP skill 的安全例外

`createSkillCommand(...)` 里有一条很重要的分支：

```ts
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(...)
}
```

也就是 **MCP skill 不允许执行内嵌 shell**。注释写得很清楚：远端/MCP 技能是不可信来源，不能在本地内联执行 shell。

这是技能系统里比较重要的一道安全边界。

---

## 9. 动态发现和条件激活

## 9.1 `paths` 条件技能

带 `paths` frontmatter 的 skill，在初次加载时不会直接暴露给模型，而是被放进 `conditionalSkills`。

实现见 `src/skills/loadSkillsDir.ts:997` 的 `activateConditionalSkillsForPaths(...)`：

```ts
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)
    ...
    if (skillIgnore.ignores(relativePath)) {
      dynamicSkills.set(name, skill)
      conditionalSkills.delete(name)
      activatedConditionalSkillNames.add(name)
      activated.push(name)
    }
  }
}
```

也就是说，这类 skill 的暴露条件是：**模型在本轮实际碰到了某些匹配路径的文件**。

## 9.2 文件操作触发动态 skill 发现

`FileReadTool`、`FileWriteTool`、`FileEditTool` 都会在触碰文件后触发 skill 动态发现。

例如 `src/tools/FileReadTool/FileReadTool.ts:579`：

```ts
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length > 0) {
  for (const dir of newSkillDirs) {
    context.dynamicSkillDirTriggers?.add(dir)
  }
  addSkillDirectories(newSkillDirs).catch(() => {})
}

activateConditionalSkillsForPaths([fullFilePath], cwd)
```

同样的逻辑在：

- `src/tools/FileWriteTool/FileWriteTool.ts:234`
- `src/tools/FileEditTool/FileEditTool.ts:408`

也存在。

## 9.3 动态 skill 目录发现策略

`discoverSkillDirsForPaths(...)` 会从被访问文件所在目录一路向上走到 `cwd`，但不包含 `cwd` 本身。

它还会：

- 跳过已扫描路径
- 跳过 gitignored 目录里的 `.claude/skills`
- 按路径深度排序，越靠近文件的目录优先级越高

这说明动态 skill 的设计意图是：**让离当前工作区域更近的 skill 在上下文上覆盖更远的 skill。**

## 9.4 动态 skill 加载是异步、非阻塞的

`addSkillDirectories(newSkillDirs).catch(() => {})` 故意不 `await`。

也就是说，在文件读写主流程里，动态 skill 发现只是“顺手触发”，不会阻塞当前工具调用。

## 9.5 动态 skill 变化会触发缓存失效和热更新

`src/utils/skills/skillChangeDetector.ts` 负责监听 skill/command 目录变化。

关键逻辑：

```ts
onDynamicSkillsLoaded(() => {
  clearCommandMemoizationCaches()
  skillsChanged.emit()
})
```

REPL 侧在 `src/cli/print.ts:1823` 订阅它：

```ts
const unsubscribeSkillChanges = skillChangeDetector.subscribe(() => {
  clearCommandsCache()
  void getCommands(cwd()).then(newCommands => {
    currentCommands = newCommands
  })
})
```

这意味着技能文件修改后，命令系统和技能列表会热更新。

---

## 10. Bundled Skill、Plugin Skill、MCP Skill 是怎么融入统一体系的

## 10.1 Bundled skill

bundled skill 的实现文件是 `src/skills/bundledSkills.ts`。

注册接口在 `src/skills/bundledSkills.ts:53`：

```ts
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const command: Command = {
    type: 'prompt',
    name: definition.name,
    description: definition.description,
    allowedTools: definition.allowedTools ?? [],
    userInvocable: definition.userInvocable ?? true,
    source: 'bundled',
    loadedFrom: 'bundled',
    hooks: definition.hooks,
    skillRoot,
    context: definition.context,
    agent: definition.agent,
    isEnabled: definition.isEnabled,
    isHidden: !(definition.userInvocable ?? true),
    progressMessage: 'running',
    getPromptForCommand,
  }
  bundledSkills.push(command)
}
```

这里和本地 skill 的统一性非常明显：最终还是注册成 `Command`。

### 10.1.1 bundled skill 的额外能力：内置参考文件惰性解压

如果 bundled skill 定义了 `files`，框架会在第一次调用时把这些参考文件安全地解压到磁盘，再把 `Base directory for this skill: ...` 前缀注入 prompt。

相关代码见：

- `getBundledSkillExtractDir(...)`，`src/skills/bundledSkills.ts:120`
- `extractBundledSkillFiles(...)`，`src/skills/bundledSkills.ts:131`
- `prependBaseDir(...)`，`src/skills/bundledSkills.ts:208`

它还做了安全写入：

- owner-only 权限
- `O_EXCL`
- `O_NOFOLLOW`
- 防目录穿越

这说明 bundled skill 并不只是纯字符串 prompt，它还可以携带一组配套文档/脚本文件。

## 10.2 启动时优先注册 bundled skill

`src/main.tsx:1919` 特地强调了这一点：

```ts
// Register bundled skills/plugins before kicking getCommands()
if (process.env.CLAUDE_CODE_ENTRYPOINT !== 'local-agent') {
  initBuiltinPlugins()
  initBundledSkills()
}
```

注释里写得很直接：如果晚于 `getCommands()` 注册，缓存里会拿到空的 bundled skill 列表。

### 10.2.1 bundled skill 示例

#### `verify`

`src/skills/bundled/verify.ts:17`：

```ts
registerBundledSkill({
  name: 'verify',
  description: DESCRIPTION,
  userInvocable: true,
  files: SKILL_FILES,
  async getPromptForCommand(args) {
    const parts: string[] = [SKILL_BODY.trimStart()]
    if (args) {
      parts.push(`## User Request\n\n${args}`)
    }
    return [{ type: 'text', text: parts.join('\n\n') }]
  },
})
```

这个 skill 的正文来自一个真实的 `SKILL.md` 模板，只不过被打包进源码里。

#### `skillify`

`src/skills/bundled/skillify.ts:22` 定义了一个很长的系统化 prompt：

```md
# Skillify {{userDescriptionBlock}}

You are capturing this session's repeatable process as a reusable skill.

## Your Session Context

Here is the session memory summary:
<session_memory>
{{sessionMemory}}
</session_memory>

Here are the user's messages during this session...

## Your Task

### Step 1: Analyze the Session
- What repeatable process was performed
- What the inputs/parameters were
- The distinct steps (in order)
...

### Step 2: Interview the User
- Use AskUserQuestion for ALL questions!
...

### Step 3: Write the SKILL.md
Create the skill directory and file at the location the user chose...
```

然后 `getPromptForCommand(args, context)` 会把 session memory 和历史 user messages 填进去。

这说明 bundled skill 的本质也是 prompt 工程，只是来源从磁盘换成了内置字符串。

#### `claude-api`

`src/skills/bundled/claudeApi.ts` 更进一步，调用时会：

1. 检测当前项目语言
2. 惰性加载大量文档内容
3. 将相关文档按 `<doc path="...">` 形式内嵌进 prompt

这已经很接近“prompt + 文档包 + 检索引导”的复合 skill 形态了。

## 10.3 Plugin skill

plugin skill 的加载逻辑在 `src/utils/plugins/loadPluginCommands.ts`。

它和本地 skill 几乎共用一套语义：

- 解析 frontmatter
- `loadedFrom: 'plugin'`
- 支持 `allowed-tools`
- 支持 `${CLAUDE_SKILL_DIR}`
- 支持 `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_DATA}`
- 支持 `${user_config.X}`
- 支持 `executeShellCommandsInPrompt(...)`

核心片段见 `src/utils/plugins/loadPluginCommands.ts:316` 之后：

```ts
loadedFrom: isSkill || config.isSkillMode ? 'plugin' : undefined,
...
finalContent = substitutePluginVariables(finalContent, ...)
finalContent = substituteUserConfigInContent(finalContent, ...)
finalContent = await executeShellCommandsInPrompt(...)
```

这说明插件 skill 不是另一套插件 runtime，而是扩展版的 skill source adapter。

## 10.4 MCP skill

MCP prompt 被转换成 `Command` 的逻辑在 `src/services/mcp/client.ts:2053`：

```ts
return promptsToProcess.map(prompt => {
  const argNames = Object.values(prompt.arguments ?? {}).map(k => k.name)
  return {
    type: 'prompt' as const,
    name: 'mcp__' + normalizeNameForMCP(client.name) + '__' + prompt.name,
    description: prompt.description ?? '',
    hasUserSpecifiedDescription: !!prompt.description,
    source: 'mcp',
    async getPromptForCommand(args: string) {
      const connectedClient = await ensureConnectedClient(client)
      const result = await connectedClient.client.getPrompt(...)
      ...
      return transformed.flat()
    },
  }
})
```

MCP 技能和本地 skill 的差别主要在于：

- 内容不是从磁盘 `SKILL.md` 读取
- `getPromptForCommand()` 会回调远端 MCP 服务取 prompt
- 不允许执行本地嵌入 shell

## 10.5 为避免循环依赖，MCP 通过一个 leaf registry 复用 skill builder

`src/skills/mcpSkillBuilders.ts` 里维护了一个 write-once registry：

```ts
let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error('MCP skill builders not registered ...')
  }
  return builders
}
```

`loadSkillsDir.ts` 在模块初始化时调用 `registerMCPSkillBuilders(...)`。这是一种很典型的“为了绕开循环依赖和 Bun bundle 限制而做的依赖图折中”。

---

## 11. Compact / Resume 时 skill 如何保留

## 11.1 已调用 skill 会被记录到全局状态

`src/bootstrap/state.ts:1510`：

```ts
export function addInvokedSkill(
  skillName: string,
  skillPath: string,
  content: string,
  agentId: string | null = null,
): void {
  const key = `${agentId ?? ''}:${skillName}`
  STATE.invokedSkills.set(key, {
    skillName,
    skillPath,
    content,
    invokedAt: Date.now(),
    agentId,
  })
}
```

这个设计很讲究：

- key 带 `agentId`
- 这样主线程和子 agent 的 skill 不会互相污染

## 11.2 compact 时转成 `invoked_skills` attachment

`src/services/compact/compact.ts:1494` 的 `createSkillAttachmentIfNeeded(...)` 会把这些 skill 恢复成附件：

```ts
const invokedSkills = getInvokedSkillsForAgent(agentId)
...
return createAttachmentMessage({
  type: 'invoked_skills',
  skills,
})
```

它还会：

- 按最近使用时间排序
- 对单个 skill 内容做 token 截断
- 受总 token budget 控制

因此 compact 之后，模型虽然失去了历史全文，但还能收到“最近调用过的技能摘要正文”。

## 11.3 resume 时恢复 skill 状态

`src/utils/conversationRecovery.ts:382`：

```ts
export function restoreSkillStateFromMessages(messages: Message[]): void {
  for (const message of messages) {
    if (message.attachment.type === 'invoked_skills') {
      for (const skill of message.attachment.skills) {
        if (skill.name && skill.path && skill.content) {
          addInvokedSkill(skill.name, skill.path, skill.content, null)
        }
      }
    }
    if (message.attachment.type === 'skill_listing') {
      suppressNextSkillListing()
    }
  }
}
```

这能避免两个问题：

1. resume 后再次 compact 时，之前已展开 skill 的内容丢失。
2. resume 后把旧的 `skill_listing` 再重复打一遍，浪费 token。

---

## 12. 这套实现的几个关键设计取舍

## 12.1 Skill 被实现成 PromptCommand，而不是特殊解释器

优点：

- 复用现有 command registry、权限、上下文、agent、tool attachment 体系。
- plugin、bundled、MCP、local skill 都可以落到同一个抽象。
- SkillTool 只需要处理“如何把 prompt command 展开”，而不是维护另一套 runtime。

代价：

- skill 的“执行”本质上仍然是 prompt 注入，不是结构化流程图。
- 复杂 skill 的可预测性依然取决于 prompt 质量，而不是状态机。

## 12.2 Skill 正文延迟加载是整个系统能工作的关键

如果一上来把所有 `SKILL.md` 全量塞进系统 prompt：

- token 会爆炸
- prompt cache 命中会变差
- 多来源 skill 数量一大，模型选择质量反而下降

当前做法是：

- 启动时只发索引信息
- 调用时才注入正文

这是非常合理的工程取舍。

## 12.3 Skill discovery 是“静态索引 + 动态发现 + 条件激活”的混合策略

它不是纯粹靠：

- 全量静态列表
- 或全量语义检索

而是把三者叠起来：

1. 初始 `skill_listing`
2. 文件路径触发的动态加载
3. `paths` 条件激活
4. 实验性的 `skill_discovery`

这使得 skill 既不会一开始过载，也能随任务推进逐步暴露更相关的技能。

## 12.4 安全边界做得比较细

我认为这个实现里几处安全处理很关键：

- MCP skill 不允许执行嵌入 shell
- 嵌入 shell 前先做权限校验
- bundled file 解压使用安全写文件策略
- `SAFE_SKILL_PROPERTIES` 控制自动放行的权限范围
- `plugin-only` / settings source 开关可以从源头关停某类 skill

这表明作者明确知道：skill 虽然表面是 prompt，但已经具备“半可执行配置”的性质，所以必须加边界。

---

## 13. 我对当前实现的总结判断

从工程实现看，这个项目的 `skills` 不是“把 prompt 文档堆到一个目录里”那么简单，而是一整套围绕 PromptCommand 的扩展系统：

- 定义层：`SKILL.md + frontmatter`
- 编译层：`createSkillCommand(...)`
- 注册层：`commands.ts`
- 暴露层：`SkillTool prompt + skill_listing + skill_discovery`
- 执行层：`SkillTool -> processPromptSlashCommand -> meta messages`
- 扩展层：plugin / bundled / MCP / remote
- 运行时增强：参数、变量、shell、fork、hooks
- 生命周期管理：dynamic discovery、hot reload、compact/resume recovery

如果要给它下一个技术定义，我会写成：

> **Claude Code 的 skill 系统，本质上是一套“多来源 PromptCommand 编译与按需展开框架”。**

它最强的地方不在于 prompt 本身，而在于它把 prompt 资产、命令系统、权限系统、agent 系统、上下文压缩系统做了统一。

---

## 14. 附录：最值得看的源码入口

如果只看 10 个文件，我建议按这个顺序读：

1. `src/types/command.ts`
2. `src/skills/loadSkillsDir.ts`
3. `src/commands.ts`
4. `src/tools/SkillTool/SkillTool.ts`
5. `src/tools/SkillTool/prompt.ts`
6. `src/utils/processUserInput/processSlashCommand.tsx`
7. `src/utils/promptShellExecution.ts`
8. `src/skills/bundledSkills.ts`
9. `src/services/compact/compact.ts`
10. `src/utils/attachments.ts`

---

## 15. 附录：代表性源码/Prompt 摘录

### 15.1 `createSkillCommand()` 的核心思路

```ts
async getPromptForCommand(args, toolUseContext) {
  let finalContent = baseDir
    ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
    : markdownContent

  finalContent = substituteArguments(
    finalContent,
    args,
    true,
    argumentNames,
  )

  if (baseDir) {
    const skillDir =
      process.platform === 'win32' ? baseDir.replace(/\\/g, '/') : baseDir
    finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
  }

  finalContent = finalContent.replace(
    /\$\{CLAUDE_SESSION_ID\}/g,
    getSessionId(),
  )

  if (loadedFrom !== 'mcp') {
    finalContent = await executeShellCommandsInPrompt(...)
  }

  return [{ type: 'text', text: finalContent }]
}
```

### 15.2 SkillTool 对模型的核心要求

```text
When a skill matches the user's request, this is a BLOCKING REQUIREMENT:
invoke the relevant Skill tool BEFORE generating any other response about the task

NEVER mention a skill without actually calling this tool

Do not use this tool for built-in CLI commands
```

### 15.3 `skillify` 的代表性 prompt 片段

```md
## Your Task

### Step 1: Analyze the Session
- What repeatable process was performed
- What the inputs/parameters were
- The distinct steps (in order)
- The success artifacts/criteria

### Step 2: Interview the User
- Use AskUserQuestion for ALL questions!
- Suggest a name and description for the skill
- Ask where the skill should be saved
- Ask whether the skill should run inline or forked

### Step 3: Write the SKILL.md
Use this format:

---
name: {{skill-name}}
description: {{one-line description}}
allowed-tools:
  {{list of tool permission patterns observed during session}}
when_to_use: {{detailed description...}}
argument-hint: "{{hint showing argument placeholders}}"
arguments:
  {{list of argument names}}
context: {{inline or fork -- omit for inline}}
---
```

### 15.4 Skill 在 compact 后如何恢复

```ts
export function createSkillAttachmentIfNeeded(
  agentId?: string,
): AttachmentMessage | null {
  const invokedSkills = getInvokedSkillsForAgent(agentId)
  ...
  return createAttachmentMessage({
    type: 'invoked_skills',
    skills,
  })
}
```

这段代码是 skill 能跨 compact/resume 存活的关键。
