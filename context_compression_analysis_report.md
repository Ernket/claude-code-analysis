# Claude Code 项目上下文压缩实现分析报告

## 0. 结论摘要

这个项目的“上下文压缩”不是单一功能，而是一套分层的上下文治理体系，核心由 4 条路径组成：

1. `microcompact`
   在真正发起模型请求前，优先清理旧的工具结果，尽量少动主消息结构，降低 token 占用。

2. `autocompact`
   当上下文接近阈值时，自动把历史对话总结成一条“压缩摘要消息”，再配合少量保留消息和附件继续当前轮次。

3. `manual /compact`
   用户显式执行 `/compact` 时，走与自动压缩相近的摘要链路，但允许自定义总结指令。

4. `session memory compact` 与 `partial compact`
   前者优先复用会话记忆文件做摘要，后者允许从 REPL 里选中消息，只压缩前缀或后缀，而不是整段历史。

这套系统的核心思想可以概括为一句话：

> 先把“模型真正还需要看到的东西”投影出来，再把“可以被总结/清理的部分”变成结构化摘要、边界标记和少量恢复附件。

---

## 1. 分析范围

本报告基于当前仓库中可直接读取到的源码，重点覆盖以下文件：

| 文件 | 作用 |
| --- | --- |
| `src/commands/compact/index.ts` | `/compact` 命令注册 |
| `src/commands/compact/compact.ts` | 手动压缩入口 |
| `src/services/compact/compact.ts` | 传统摘要压缩、局部压缩、压缩后重建 |
| `src/services/compact/prompt.ts` | 压缩 prompt 与摘要包装逻辑 |
| `src/services/compact/autoCompact.ts` | 自动压缩阈值与触发器 |
| `src/services/compact/microCompact.ts` | 轻量压缩与旧工具结果清理 |
| `src/services/compact/sessionMemoryCompact.ts` | 基于 session memory 的压缩 |
| `src/services/compact/postCompactCleanup.ts` | 压缩完成后的状态清理 |
| `src/utils/messages.ts` | `compact_boundary` / `microcompact_boundary` 与边界切片 |
| `src/query.ts` | 查询主循环中串联 microcompact 和 autocompact |
| `src/QueryEngine.ts` | transcript 落盘、resume、压缩边界持久化 |
| `src/utils/hooks.ts` | `PreCompact` / `PostCompact` hooks |
| `src/constants/prompts.ts` | 对工具结果清理的系统提示协同 |
| `src/screens/REPL.tsx` | 局部压缩的 UI 触发点 |

另外，当前快照里存在若干“被调用但源码文件缺失”的 feature-gated 分支：

| 缺失模块 | 当前能确认的程度 |
| --- | --- |
| `src/services/compact/reactiveCompact.ts` | 能从调用点确认作用，但不能展开内部实现 |
| `src/services/compact/cachedMicrocompact.ts` | 能从接口使用方式推断缓存编辑策略，但不能查看内部算法 |
| `src/services/compact/snipCompact.ts` | 能确认其位于 microcompact 之前执行，但当前快照缺少实现 |
| `src/services/compact/snipProjection.ts` | 能确认它影响“边界后的可见视图”，但当前快照缺少实现 |

因此，本报告分为两类结论：

1. 可以直接从源码确认的实现事实。
2. 只能从调用点和接口推断的行为，我会明确标注为“推断”。

---

## 2. 整体架构图

### 2.1 查询主循环中的压缩顺序

```text
query.ts
  -> getMessagesAfterCompactBoundary(messages)
  -> applyToolResultBudget(...)
  -> snipCompactIfNeeded(...)           // feature-gated，当前快照缺少实现
  -> microcompactMessages(...)
  -> autoCompactIfNeeded(...)
       -> trySessionMemoryCompaction(...)
       -> compactConversation(...)
            -> streamCompactSummary(...)
            -> buildPostCompactMessages(...)
  -> 用压缩后的消息继续当前 query
```

### 2.2 手动压缩路径

```text
/compact
  -> commands/compact/compact.ts
  -> getMessagesAfterCompactBoundary(...)
  -> trySessionMemoryCompaction(...)    // 无自定义指令时优先
  -> microcompactMessages(...)
  -> compactConversation(...)
  -> 返回 compactionResult 给 REPL / SDK
```

### 2.3 局部压缩路径

```text
REPL 消息选择器
  -> partialCompactConversation(...)
  -> 只总结前缀或后缀
  -> 保留另一半消息
  -> 仍然写入 compact boundary
```

---

## 3. 关键抽象：压缩边界、摘要消息与保留段

上下文压缩能工作的根本，不是“把数组截断”，而是引入了一个明确的边界消息 `compact_boundary`。

### 3.1 `compact_boundary` 的数据结构

位置：`src/utils/messages.ts:4530`

```ts
export function createCompactBoundaryMessage(
  trigger: 'manual' | 'auto',
  preTokens: number,
  lastPreCompactMessageUuid?: UUID,
  userContext?: string,
  messagesSummarized?: number,
): SystemCompactBoundaryMessage {
  return {
    type: 'system',
    subtype: 'compact_boundary',
    content: `Conversation compacted`,
    isMeta: false,
    timestamp: new Date().toISOString(),
    uuid: randomUUID(),
    level: 'info',
    compactMetadata: {
      trigger,
      preTokens,
      userContext,
      messagesSummarized,
    },
    ...(lastPreCompactMessageUuid && {
      logicalParentUuid: lastPreCompactMessageUuid,
    }),
  }
}
```

这个对象承载了 4 类信息：

1. 压缩是自动还是手动触发。
2. 压缩前 token 数。
3. 可选的用户上下文。
4. 本次到底总结了多少条消息。

它不是 UI 装饰，而是整个会话恢复、切片和 resume 的锚点。

### 3.2 所有模型视图都从“最后一个压缩边界”开始

位置：`src/utils/messages.ts:4643`

```ts
export function getMessagesAfterCompactBoundary<
  T extends Message | NormalizedMessage,
>(messages: T[], options?: { includeSnipped?: boolean }): T[] {
  const boundaryIndex = findLastCompactBoundaryIndex(messages)
  const sliced = boundaryIndex === -1 ? messages : messages.slice(boundaryIndex)
  if (!options?.includeSnipped && feature('HISTORY_SNIP')) {
    const { projectSnippedView } =
      require('../services/compact/snipProjection.js')
    return projectSnippedView(sliced as Message[]) as T[]
  }
  return sliced
}
```

这个函数说明了项目的基本策略：

1. 不是“永久删除所有旧消息”。
2. 而是“之后所有真正面向模型的逻辑，只从最近一次 compact boundary 之后取消息”。

这也是为什么：

1. `/context` 能显示“模型实际看到的上下文”。
2. query 主循环能在压缩后立即继续。
3. transcript/resume 仍能保留完整历史，但活跃上下文只保留边界之后的那一段。

### 3.3 压缩结果的标准消息顺序

位置：`src/services/compact/compact.ts:330`

```ts
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,
    ...result.summaryMessages,
    ...(result.messagesToKeep ?? []),
    ...result.attachments,
    ...result.hookResults,
  ]
}
```

这段代码定义了“压缩后的活跃上下文”：

1. 边界消息。
2. 一条或多条压缩摘要消息。
3. 可选的原始保留消息。
4. 压缩后补回的附件。
5. SessionStart / Hook 结果。

换句话说，Claude Code 的压缩不是纯摘要，而是：

> `边界 + 摘要 + 必要原文 + 恢复附件 + hooks`

### 3.4 保留段 relink 机制

位置：`src/services/compact/compact.ts:349`

```ts
export function annotateBoundaryWithPreservedSegment(
  boundary: SystemCompactBoundaryMessage,
  anchorUuid: UUID,
  messagesToKeep: readonly Message[] | undefined,
): SystemCompactBoundaryMessage {
  const keep = messagesToKeep ?? []
  if (keep.length === 0) return boundary
  return {
    ...boundary,
    compactMetadata: {
      ...boundary.compactMetadata,
      preservedSegment: {
        headUuid: keep[0]!.uuid,
        anchorUuid,
        tailUuid: keep.at(-1)!.uuid,
      },
    },
  }
}
```

这说明压缩后并不是简单拼接文本，而是给“保留原文段”额外记录了一套链式 relink 元数据：

1. `headUuid`
2. `anchorUuid`
3. `tailUuid`

其作用是让 transcript / resume 能在恢复时正确连接摘要和保留消息，不把旧链路错误地串回来。

---

## 4. 手动 `/compact` 的实现

### 4.1 命令注册

位置：`src/commands/compact/index.ts:6`

```ts
const compact = {
  type: 'local',
  name: 'compact',
  description:
    'Clear conversation history but keep a summary in context. Optional: /compact [instructions for summarization]',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_COMPACT),
  supportsNonInteractive: true,
  argumentHint: '<optional custom summarization instructions>',
  load: () => import('./compact.js'),
}
```

从描述可以直接看出两点：

1. `/compact` 的语义不是“清空对话”，而是“保留 summary in context”。
2. 命令支持附带自定义 summarization instructions。

### 4.2 手动压缩入口逻辑

位置：`src/commands/compact/compact.ts:44`

```ts
messages = getMessagesAfterCompactBoundary(messages)

if (!customInstructions) {
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages,
    context.agentId,
  )
  if (sessionMemoryResult) {
    ...
    return {
      type: 'compact',
      compactionResult: sessionMemoryResult,
      displayText: buildDisplayText(context),
    }
  }
}

const microcompactResult = await microcompactMessages(messages, context)
const messagesForCompact = microcompactResult.messages

const result = await compactConversation(
  messagesForCompact,
  context,
  await getCacheSharingParams(context, messagesForCompact),
  false,
  customInstructions,
  false,
)
```

这段代码直接展示了手动 `/compact` 的优先级：

1. 先把上下文投影到最近一次 compact boundary 之后。
2. 如果用户没有传自定义指令，优先尝试 `session memory compact`。
3. 否则先做一次 `microcompact`，再执行真正的摘要压缩 `compactConversation(...)`。

这里能看出一个很重要的产品思路：

> 只要还有更便宜、更结构化的压缩方式，就先不要做昂贵的“全文摘要”。

### 4.3 自定义总结指令

`/compact [instructions]` 最终会传进 `getCompactPrompt(customInstructions)`，而且 `PreCompact hooks` 还能继续往里面追加指令。

位置：`src/services/compact/compact.ts:374`

```ts
export function mergeHookInstructions(
  userInstructions: string | undefined,
  hookInstructions: string | undefined,
): string | undefined {
  if (!hookInstructions) return userInstructions || undefined
  if (!userInstructions) return hookInstructions
  return `${userInstructions}\n\n${hookInstructions}`
}
```

也就是说，手动 `/compact` 最终看到的 prompt 指令来源有三层：

1. 用户在命令行里附带的指令。
2. `PreCompact` hook 输出的新指令。
3. 局部压缩时来自 UI 的 `feedback`。

---

## 5. 核心摘要压缩：`compactConversation(...)`

位置：`src/services/compact/compact.ts:387`

这是真正把长对话变成压缩摘要的主函数。

### 5.1 前置动作

进入函数后首先做的事：

1. 统计压缩前 token。
2. 执行 `PreCompact hooks`。
3. 合并 hook 返回的新指令。
4. 设置 UI / SDK 状态为 `compacting`。
5. 构造 summary request。

核心代码：

```ts
const preCompactTokenCount = tokenCountWithEstimation(messages)

const hookResult = await executePreCompactHooks(
  {
    trigger: isAutoCompact ? 'auto' : 'manual',
    customInstructions: customInstructions ?? null,
  },
  context.abortController.signal,
)
customInstructions = mergeHookInstructions(
  customInstructions,
  hookResult.newCustomInstructions,
)

const compactPrompt = getCompactPrompt(customInstructions)
const summaryRequest = createUserMessage({
  content: compactPrompt,
})
```

这说明 `compactConversation` 本质上不是在本地拼摘要，而是另外发起了一次“专用压缩请求”。

### 5.2 摘要请求不是直接裸调，而是带缓存优化的

位置：`src/services/compact/compact.ts:1136`

```ts
async function streamCompactSummary({
  messages,
  summaryRequest,
  appState,
  context,
  preCompactTokenCount,
  cacheSafeParams,
}: ...): Promise<AssistantMessage> {
  const promptCacheSharingEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_compact_cache_prefix',
    true,
  )

  if (promptCacheSharingEnabled) {
    try {
      const result = await runForkedAgent({
        promptMessages: [summaryRequest],
        cacheSafeParams,
        canUseTool: createCompactCanUseTool(),
        querySource: 'compact',
        forkLabel: 'compact',
        maxTurns: 1,
        skipCacheWrite: true,
        overrides: { abortController: context.abortController },
      })
      ...
      return assistantMsg
    } catch (error) {
      ...
    }
  }

  const streamingGen = queryModelWithStreaming({
    messages: normalizeMessagesForAPI(
      stripImagesFromMessages(
        stripReinjectedAttachments([
          ...getMessagesAfterCompactBoundary(messages),
          summaryRequest,
        ]),
      ),
      context.options.tools,
    ),
    systemPrompt: asSystemPrompt([
      'You are a helpful AI assistant tasked with summarizing conversations.',
    ]),
    thinkingConfig: { type: 'disabled' as const },
    ...
  })
}
```

这里有 4 个很关键的实现点：

1. 优先用 `runForkedAgent(...)` 复用主会话的 prompt cache。
2. 压缩 agent 被强制禁止工具调用。
3. 真正送给模型的消息会先经过 `stripImagesFromMessages(...)` 和 `stripReinjectedAttachments(...)`。
4. 如果缓存共享失败，再退回普通 streaming 调用。

### 5.3 为什么要剥离图片、文档和某些附件

位置：

1. `src/services/compact/compact.ts:145`
2. `src/services/compact/compact.ts:211`

项目里明确做了两类清理：

1. 把图片/文档替换成 `[image]` / `[document]` 文本标记，避免压缩请求本身再次触发 prompt-too-long。
2. 把后续本来就会重新注入的附件先从摘要上下文中剥掉，避免浪费 token。

这说明设计者把“压缩请求自己也可能过长”当成了真实生产问题处理。

### 5.4 压缩请求过长时，会继续截头重试

位置：`src/services/compact/compact.ts:227`, `:257`

关键常量：

```ts
const MAX_PTL_RETRIES = 3
const PTL_RETRY_MARKER = '[earlier conversation truncated for compaction retry]'
```

当摘要请求自己触发 `prompt too long` 时，并不是直接失败，而是：

1. 用 `groupMessagesByApiRound(...)` 按 API round 对消息分组。
2. 逐步删除最老的 group。
3. 在必要时插入 `PTL_RETRY_MARKER`。
4. 最多重试 3 次。

这说明压缩逻辑本身有“自保机制”，避免用户陷入“上下文太长，连压缩都压不了”的死锁。

### 5.5 压缩后的消息不是只有 summary

位置：`src/services/compact/compact.ts:122-130`, `:1415`

关键预算：

1. `POST_COMPACT_MAX_FILES_TO_RESTORE = 5`
2. `POST_COMPACT_TOKEN_BUDGET = 50_000`
3. `POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000`
4. `POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000`
5. `POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000`

压缩完成后，项目会尽量恢复这些上下文：

1. 最近读过的文件内容。
2. 异步 agent 相关附件。
3. plan 文件。
4. 已调用过的 skills。
5. deferred tools / agent listing / MCP instructions 的增量附件。
6. SessionStart hooks 结果。

代码片段：

```ts
const [fileAttachments, asyncAgentAttachments] = await Promise.all([
  createPostCompactFileAttachments(
    preCompactReadFileState,
    context,
    POST_COMPACT_MAX_FILES_TO_RESTORE,
  ),
  createAsyncAgentAttachmentsIfNeeded(context),
])

const planAttachment = createPlanAttachmentIfNeeded(context.agentId)
const planModeAttachment = await createPlanModeAttachmentIfNeeded(context)
const skillAttachment = createSkillAttachmentIfNeeded(context.agentId)
```

因此，Claude Code 的压缩策略不是“总结掉一切”，而是：

> 用摘要替代历史对话，但把继续工作时最贵、最容易丢失的运行态上下文重新挂回来。

### 5.6 压缩完成后会运行 `PostCompact hooks`

位置：`src/services/compact/compact.ts:716`

```ts
const postCompactHookResult = await executePostCompactHooks(
  {
    trigger: isAutoCompact ? 'auto' : 'manual',
    compactSummary: summary,
  },
  context.abortController.signal,
)
```

这意味着压缩结果并不是闭环终点，还允许外部扩展在摘要生成后执行观测、记录或补充动作。

---

## 6. 压缩 prompt 设计

`src/services/compact/prompt.ts` 是整个压缩质量的关键，它的设计非常“工程化”，不是一句简单的“请总结一下上文”。

### 6.1 强约束：禁止工具，只允许 `<analysis>` + `<summary>`

位置：`src/services/compact/prompt.ts:19`

```text
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

这个 preamble 解决的是两个实际问题：

1. 压缩 agent 只有一次机会，工具调用会浪费 turn。
2. 需要让模型先做草稿分析，再给正式摘要，但不希望草稿进入后续上下文。

### 6.2 基础压缩 prompt 要求非常细

位置：`src/services/compact/prompt.ts:61`

它要求摘要必须包含 9 个部分：

1. 用户主请求与意图。
2. 关键技术概念。
3. 文件与代码段。
4. 错误与修复。
5. 问题解决过程。
6. 全部用户消息。
7. 待办任务。
8. 当前正在做的事。
9. 下一个直接相关动作。

而且 prompt 里明确要求：

1. 尽量包含完整代码片段。
2. 记录用户反馈和纠正。
3. 最后一步尽量引用最近对话中的原话，避免任务漂移。

这实际上是在让“压缩摘要”兼做一个高保真 handoff 文档。

### 6.3 局部压缩 prompt 有两个版本

位置：

1. `src/services/compact/prompt.ts:145`
2. `src/services/compact/prompt.ts:208`

区别如下：

1. `PARTIAL_COMPACT_PROMPT`
   只总结“最近部分”，因为前文保留不动。

2. `PARTIAL_COMPACT_UP_TO_PROMPT`
   用于总结前缀，因为摘要会放在后续原文之前，必须额外强调“Context for Continuing Work”。

这说明 partial compact 不是复用同一套 prompt，而是针对“保留前缀”与“保留后缀”做了 prompt 语义分化。

### 6.4 摘要的 `<analysis>` 会被剥离，只把 `<summary>` 注入新上下文

位置：`src/services/compact/prompt.ts:311`

```ts
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary

  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )

  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    const content = summaryMatch[1] || ''
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${content.trim()}`,
    )
  }

  return formattedSummary.trim()
}
```

这里的设计很值得注意：

1. 压缩 agent 可以先“想得更完整”。
2. 但这些草稿不会污染压缩后的长期上下文。

这是一种典型的“让模型多想一点，但不把草稿暴露给后续轮次”的工程技巧。

### 6.5 注入回对话的不是裸摘要，而是一条续写指令

位置：`src/services/compact/prompt.ts:337`

```ts
let baseSummary = `This session is being continued from a previous conversation that ran out of context. The summary below covers the earlier portion of the conversation.

${formattedSummary}`

if (transcriptPath) {
  baseSummary += `\n\nIf you need specific details from before compaction ... read the full transcript at: ${transcriptPath}`
}

if (recentMessagesPreserved) {
  baseSummary += `\n\nRecent messages are preserved verbatim.`
}

if (suppressFollowUpQuestions) {
  let continuation = `${baseSummary}
Continue the conversation from where it left off without asking the user any further questions. Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with "I'll continue" or similar. Pick up the last task as if the break never happened.`
  ...
}
```

也就是说，压缩后的“摘要消息”实际承担了两层职责：

1. 给模型一个历史摘要。
2. 直接规定压缩后的行为模式：不要打断用户，不要重新开场，直接续上之前的任务。

---

## 7. 自动压缩：`autocompact`

### 7.1 自动压缩阈值计算

位置：`src/services/compact/autoCompact.ts:33`, `:72`

```ts
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,
  )
  ...
  return contextWindow - reservedTokensForSummary
}

export const AUTOCOMPACT_BUFFER_TOKENS = 13_000

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold =
    effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  ...
  return autocompactThreshold
}
```

这个计算方式说明设计者不是等到“刚好塞不下”才压缩，而是预留了两层缓冲：

1. 先预留最多 20k token 给压缩摘要输出。
2. 再在有效窗口上额外减去 13k buffer。

这是一种比较保守、偏生产稳定性的做法。

### 7.2 自动压缩不是每次都开

位置：`src/services/compact/autoCompact.ts:147`

```ts
export function isAutoCompactEnabled(): boolean {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) {
    return false
  }
  if (isEnvTruthy(process.env.DISABLE_AUTO_COMPACT)) {
    return false
  }
  const userConfig = getGlobalConfig()
  return userConfig.autoCompactEnabled
}
```

自动压缩受 3 层控制：

1. 全局禁用 `DISABLE_COMPACT`
2. 单独禁用 `DISABLE_AUTO_COMPACT`
3. 用户设置 `autoCompactEnabled`

### 7.3 自动压缩在主查询循环中的位置

位置：

1. `src/query.ts:365`
2. `src/query.ts:413`
3. `src/query.ts:453`
4. `src/query.ts:528`

```ts
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages

const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource,
  tracking,
  snipTokensFreed,
)

const postCompactMessages = buildPostCompactMessages(compactionResult)
```

这里的执行顺序是：

1. 先裁掉上一次压缩边界之前的内容。
2. 再做工具结果预算收缩和 microcompact。
3. 最后才决定是否走真正的摘要压缩。

这说明 autocompact 是“最后一层防线”，不是首选策略。

### 7.4 连续失败断路器

位置：`src/services/compact/autoCompact.ts:70`, `:241`

```ts
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

如果自动压缩连续失败 3 次，就停止继续尝试，避免每一轮都打出一记注定失败的 compaction API 调用。

这是很典型的生产级防抖逻辑。

### 7.5 自动压缩优先尝试 session memory

位置：`src/services/compact/autoCompact.ts:241`

自动压缩并不是无脑调用 `compactConversation(...)`，而是先试 `trySessionMemoryCompaction(...)`，因为这条路径更轻量，也更稳定。

---

## 8. `microcompact`：在真正摘要前先清理旧工具结果

`microcompact` 是整个系统里最像“上下文优化器”的模块。

### 8.1 只压某些工具结果

位置：`src/services/compact/microCompact.ts:41`

```ts
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

可见被清理的重点不是自然语言对话，而是“大块工具结果”，尤其是：

1. 文件读取结果。
2. shell 输出。
3. grep/glob 搜索输出。
4. web fetch / search 结果。
5. edit / write 的结果回显。

这很合理，因为这些内容通常又长、又容易重复读取、又比自然语言更占 token。

### 8.2 `microcompactMessages(...)` 的主逻辑

位置：`src/services/compact/microCompact.ts:253`

```ts
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    const model = toolUseContext?.options.mainLoopModel ?? getMainLoopModel()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  return { messages }
}
```

这段代码说明：

1. 先看是否满足“时间型 microcompact”触发条件。
2. 再看是否能走缓存编辑型 `cached microcompact`。
3. 两条路径都不满足则直接不做任何处理。

也就是说，当前快照里的 microcompact 是“条件式启用”的，不是每轮都动。

### 8.3 缓存编辑型 microcompact

虽然 `cachedMicrocompact.js` 不在当前快照里，但从 `microCompact.ts` 的接口调用能确认其行为特征。

位置：`src/services/compact/microCompact.ts:276`

```ts
const compactableToolIds = new Set(collectCompactableToolIds(messages))
...
if (
  block.type === 'tool_result' &&
  compactableToolIds.has(block.tool_use_id) &&
  !state.registeredTools.has(block.tool_use_id)
) {
  mod.registerToolResult(state, block.tool_use_id)
  groupIds.push(block.tool_use_id)
}
...
const toolsToDelete = mod.getToolResultsToDelete(state)
...
const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
pendingCacheEdits = cacheEdits

return {
  messages,
  compactionInfo: {
    pendingCacheEdits: {
      trigger: 'auto',
      deletedToolIds: toolsToDelete,
      baselineCacheDeletedTokens: baseline,
    },
  },
}
```

可以直接确认 4 个事实：

1. 这条路径按 `tool_use_id` 追踪哪些工具结果可以删。
2. 它会生成 `cache_edits`，而不是直接改本地消息内容。
3. `messages` 原样返回，说明本地 transcript 不被立即重写。
4. 真正的“删了多少 token”要等 API 返回后，根据 `cache_deleted_input_tokens` 计算。

### 8.4 microcompact 的边界消息是延迟生成的

位置：`src/query.ts:870`

```ts
if (feature('CACHED_MICROCOMPACT') && pendingCacheEdits) {
  const usage = lastAssistant?.message.usage
  const cumulativeDeleted = usage
    ? ((usage as unknown as Record<string, number>)
        .cache_deleted_input_tokens ?? 0)
    : 0
  const deletedTokens = Math.max(
    0,
    cumulativeDeleted - pendingCacheEdits.baselineCacheDeletedTokens,
  )
  if (deletedTokens > 0) {
    yield createMicrocompactBoundaryMessage(
      pendingCacheEdits.trigger,
      0,
      deletedTokens,
      pendingCacheEdits.deletedToolIds,
      [],
    )
  }
}
```

这段代码非常能说明作者的思路：

1. 不用本地估算的 token 节省值。
2. 等 API 回来以后，用服务端的真实 `cache_deleted_input_tokens` 增量。
3. 然后再生成 `microcompact_boundary`。

这是相当严谨的实现。

### 8.5 时间型 microcompact

位置：

1. `src/services/compact/microCompact.ts:422`
2. `src/services/compact/microCompact.ts:446`
3. `src/services/compact/timeBasedMCConfig.ts:30`

默认配置：

```ts
const TIME_BASED_MC_CONFIG_DEFAULTS: TimeBasedMCConfig = {
  enabled: false,
  gapThresholdMinutes: 60,
  keepRecent: 5,
}
```

触发逻辑：

```ts
const lastAssistant = messages.findLast(m => m.type === 'assistant')
const gapMinutes =
  (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
  return null
}
```

真正清理动作：

```ts
if (
  block.type === 'tool_result' &&
  clearSet.has(block.tool_use_id) &&
  block.content !== TIME_BASED_MC_CLEARED_MESSAGE
) {
  return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
}
```

而 `TIME_BASED_MC_CLEARED_MESSAGE` 是：

```ts
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

这条路径说明：

1. 如果距离上次 assistant 回复太久，服务端 prompt cache 很可能已经过期。
2. 既然反正要重写整段前缀，不如先把旧工具结果内容清掉。
3. 但仍保留工具调用结构，不是把整条消息删掉。

### 8.6 系统 prompt 还会提前教育模型：工具结果以后可能会被清掉

位置：`src/constants/prompts.ts:821`

```ts
return `# Function Result Clearing

Old tool results will be automatically cleared from context to free up space. The ${config.keepRecent} most recent results are always kept.`
```

位置：`src/constants/prompts.ts:841`

```ts
const SUMMARIZE_TOOL_RESULTS_SECTION = `When working with tool results, write down any important information you might need later in your response, as the original tool result may be cleared later.`
```

这说明 microcompact 不是孤立模块，它和 system prompt 协同工作：

1. 提前告诉模型“旧工具结果会被清掉”。
2. 鼓励模型把重要信息主动写进自然语言响应里。

这会显著降低工具结果被清除后的信息损失。

---

## 9. `session memory compact`：优先复用结构化会话记忆

这是当前实现里最“聪明”的一条路径，因为它并不重新让模型总结一遍对话，而是尽量复用已经存在的 session memory。

### 9.1 默认配置

位置：`src/services/compact/sessionMemoryCompact.ts:57`

```ts
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000,
}
```

含义很明确：

1. 压缩后仍至少保留一定数量的 recent context。
2. 不只看 token，还要求至少保留一定数量的“文本交互消息”。
3. 同时给保留段设置上限，避免保留过多导致刚压完又超阈值。

### 9.2 启用条件

位置：`src/services/compact/sessionMemoryCompact.ts:403`

```ts
export function shouldUseSessionMemoryCompaction(): boolean {
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) {
    return true
  }
  if (isEnvTruthy(process.env.DISABLE_CLAUDE_CODE_SM_COMPACT)) {
    return false
  }

  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory',
    false,
  )
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact',
    false,
  )
  return sessionMemoryFlag && smCompactFlag
}
```

所以 session memory compact 是 feature flag 驱动的实验能力，不一定对所有环境开启。

### 9.3 如何决定保留哪些 recent messages

位置：`src/services/compact/sessionMemoryCompact.ts:324`

```ts
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  ...
  if (
    totalTokens >= config.minTokens &&
    textBlockMessageCount >= config.minTextBlockMessages
  ) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }
  ...
}
```

它不是简单“保留最后 N 条”，而是动态向前扩展，直到同时满足：

1. 至少 `minTokens`。
2. 至少 `minTextBlockMessages`。
3. 但不能超过 `maxTokens`。

### 9.4 还会修复 API 级不变量

位置：`src/services/compact/sessionMemoryCompact.ts:232`

`adjustIndexToPreserveAPIInvariants(...)` 专门处理两类问题：

1. 不拆散 `tool_use` / `tool_result` 对。
2. 不丢掉与同一 `message.id` 关联的 thinking blocks。

这说明保留 recent tail 不是纯 UI 逻辑，而是深度考虑了 Anthropic 消息格式和 streaming 合并语义。

### 9.5 session memory 生成的压缩结果

位置：`src/services/compact/sessionMemoryCompact.ts:437`

```ts
function createCompactionResultFromSessionMemory(
  messages: Message[],
  sessionMemory: string,
  messagesToKeep: Message[],
  hookResults: HookResultMessage[],
  transcriptPath: string,
  agentId?: AgentId,
): CompactionResult {
  ...
  let summaryContent = getCompactUserSummaryMessage(
    truncatedContent,
    true,
    transcriptPath,
    true,
  )

  return {
    boundaryMarker: annotateBoundaryWithPreservedSegment(
      boundaryMarker,
      summaryMessages[summaryMessages.length - 1]!.uuid,
      messagesToKeep,
    ),
    summaryMessages,
    attachments,
    hookResults,
    messagesToKeep,
    ...
  }
}
```

关键点：

1. summary 来源不是“重新 summarize 对话”，而是 session memory 文件内容。
2. `recentMessagesPreserved = true` 会写入提示，告诉模型 recent messages 是原文保留的。
3. `annotateBoundaryWithPreservedSegment(...)` 让摘要和 recent tail 形成可恢复链路。

### 9.6 什么时候会放弃 session memory compact

位置：`src/services/compact/sessionMemoryCompact.ts:514`

如果出现以下情况，会返回 `null`，退回传统 `compactConversation(...)`：

1. feature flag 没开。
2. session memory 不存在。
3. session memory 还是空模板。
4. 找不到 `lastSummarizedMessageId`。
5. 压缩后 token 仍超过自动压缩阈值。

所以它是“优先路径”，不是“唯一路径”。

---

## 10. 局部压缩：`partialCompactConversation(...)`

位置：`src/services/compact/compact.ts:772`

### 10.1 触发入口在 REPL 消息选择器

位置：`src/screens/REPL.tsx:4918`

```ts
const result = await partialCompactConversation(
  compactMessages,
  messageIndex,
  context,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext: context,
    forkContextMessages: compactMessages
  },
  feedback,
  direction
)
```

说明局部压缩不是 slash command，而是从 UI 的消息选择器里触发。

### 10.2 `from` 和 `up_to` 两种模式

位置：`src/services/compact/compact.ts:779`

```ts
const messagesToSummarize =
  direction === 'up_to'
    ? allMessages.slice(0, pivotIndex)
    : allMessages.slice(pivotIndex)

const messagesToKeep =
  direction === 'up_to'
    ? allMessages
        .slice(pivotIndex)
        .filter(
          m =>
            m.type !== 'progress' &&
            !isCompactBoundaryMessage(m) &&
            !(m.type === 'user' && m.isCompactSummary),
        )
    : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

两种模式的语义：

1. `from`
   从选中消息开始往后总结，保留前缀。

2. `up_to`
   总结选中消息之前的前缀，保留后缀。

`up_to` 额外清掉旧的 `compact_boundary` 和 `isCompactSummary`，因为它们会干扰“最后一条 boundary 才是活跃边界”的切片逻辑。

### 10.3 局部压缩允许附带用户反馈

位置：`src/services/compact/compact.ts:826`

```ts
if (hookResult.newCustomInstructions && userFeedback) {
  customInstructions = `${hookResult.newCustomInstructions}\n\nUser context: ${userFeedback}`
} else if (hookResult.newCustomInstructions) {
  customInstructions = hookResult.newCustomInstructions
} else if (userFeedback) {
  customInstructions = `User context: ${userFeedback}`
}
```

这意味着局部压缩可以不是纯机械地总结，而是允许用户指定“总结时更关注什么”。

### 10.4 保留段 anchor 策略不同

位置：`src/services/compact/compact.ts:1079`

```ts
const anchorUuid =
  direction === 'up_to'
    ? (summaryMessages.at(-1)?.uuid ?? boundaryMarker.uuid)
    : boundaryMarker.uuid
```

这里的区别很重要：

1. `from`
   是“前缀保留”，所以保留段挂在 boundary 后面。

2. `up_to`
   是“后缀保留”，所以保留段挂在 summary 后面。

这说明 partial compact 的链路重建是经过仔细设计的，不是简单数组拼接。

---

## 11. hooks、状态和收尾逻辑

### 11.1 `PreCompact` / `PostCompact` hook 输入结构

位置：`src/entrypoints/sdk/coreSchemas.ts:569`, `:579`

```ts
export const PreCompactHookInputSchema = lazySchema(() =>
  BaseHookInputSchema().and(
    z.object({
      hook_event_name: z.literal('PreCompact'),
      trigger: z.enum(['manual', 'auto']),
      custom_instructions: z.string().nullable(),
    }),
  ),
)

export const PostCompactHookInputSchema = lazySchema(() =>
  BaseHookInputSchema().and(
    z.object({
      hook_event_name: z.literal('PostCompact'),
      trigger: z.enum(['manual', 'auto']),
      compact_summary: z.string(),
    }),
  ),
)
```

这两类 hooks 的目的不同：

1. `PreCompact`
   可以额外提供自定义总结指令。

2. `PostCompact`
   可以消费刚生成的压缩摘要。

### 11.2 hooks 执行逻辑

位置：`src/utils/hooks.ts:3961`, `:4034`

`executePreCompactHooks(...)` 会把成功 hook 的 stdout 拼接成新的 `customInstructions`，而 `executePostCompactHooks(...)` 主要负责构造展示给用户的结果提示。

### 11.3 压缩后的统一清理

位置：`src/services/compact/postCompactCleanup.ts:31`

```ts
export function runPostCompactCleanup(querySource?: QuerySource): void {
  resetMicrocompactState()
  ...
  if (isMainThreadCompact) {
    getUserContext.cache.clear?.()
    resetGetMemoryFilesCache('compact')
  }
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearSessionMessagesCache()
}
```

这个清理过程说明压缩不是简单“替换消息数组”，还必须同步清掉一批旧缓存：

1. microcompact 状态。
2. user context cache。
3. memory files cache。
4. system prompt section cache。
5. 各类权限/推测检查状态。
6. session message cache。

但它**故意不清** invoked skills，因为后续压缩可能仍要继续附带这些 skill 内容。

### 11.4 压缩后的状态标记

位置：`src/bootstrap/state.ts:771`

```ts
export function markPostCompaction(): void {
  STATE.pendingPostCompaction = true
}
```

作用是：

1. 标记“刚刚发生过压缩”。
2. 让下一次 API 成功事件带上 `isPostCompaction=true`。
3. 从而区分“压缩导致的 prompt cache 变化”和普通 TTL 失效。

---

## 12. transcript、resume 与会话恢复

### 12.1 `QueryEngine` 会显式持久化 compact boundary

位置：`src/QueryEngine.ts:691`

```ts
if (
  persistSession &&
  message.type === 'system' &&
  message.subtype === 'compact_boundary'
) {
  const tailUuid = message.compactMetadata?.preservedSegment?.tailUuid
  if (tailUuid) {
    const tailIdx = this.mutableMessages.findLastIndex(
      m => m.uuid === tailUuid,
    )
    if (tailIdx !== -1) {
      await recordTranscript(this.mutableMessages.slice(0, tailIdx + 1))
    }
  }
}
```

这里的意图很明确：

1. 在写入 compact boundary 之前，先把保留段尾部之前的消息落盘。
2. 避免 resume 时 `tailUuid` 指向一条实际上没写进 transcript 的消息。
3. 否则 preserved segment relink 会失败，进而把 pre-compact 历史错误地全部读回来。

### 12.2 压缩后内存里的旧消息会被剪掉

位置：`src/QueryEngine.ts:919`

```ts
if (
  message.subtype === 'compact_boundary' &&
  message.compactMetadata
) {
  const mutableBoundaryIdx = this.mutableMessages.length - 1
  if (mutableBoundaryIdx > 0) {
    this.mutableMessages.splice(0, mutableBoundaryIdx)
  }
  const localBoundaryIdx = messages.length - 1
  if (localBoundaryIdx > 0) {
    messages.splice(0, localBoundaryIdx)
  }
}
```

这一步说明压缩不仅影响模型上下文，也影响进程内消息缓存，目的是释放内存并让后续逻辑天然只处理边界之后的消息。

### 12.3 resume 读取的是“从压缩点开始”的 internal events

位置：`src/cli/transports/ccrClient.ts:842`

```ts
async readInternalEvents(): Promise<InternalEvent[] | null> {
  return this.paginatedGet('/worker/internal-events', {}, 'internal_events')
}
```

注释里已经写明：它返回的是 “from the last compaction boundary” 的 transcript entries，用于 session resume。

因此，从持久化层看，compact boundary 真的是一个“恢复点”。

---

## 13. 这个实现为什么有效

### 13.1 它不是靠一种方法硬顶，而是分层退化

从轻到重依次是：

1. 工具结果预算收缩。
2. `microcompact` 清老工具结果。
3. `session memory compact`。
4. 传统对话摘要 `compactConversation(...)`。
5. feature-gated 的 `reactiveCompact` / `snip` / `contextCollapse`。

这比“只在爆了时做一次全文总结”稳定得多。

### 13.2 它既压缩历史，也补回工作上下文

压缩完成后会恢复：

1. 最近读取的文件。
2. plan 文件。
3. skill 内容。
4. 工具和 MCP 指令增量。
5. SessionStart hooks 结果。

所以压缩后的会话不是只有一段摘要，而是一个“可继续操作的最小运行现场”。

### 13.3 它重视 Anthropic 消息协议的不变量

项目里多处显式避免：

1. 拆散 `tool_use` / `tool_result`。
2. 丢失同一 `message.id` 的 thinking blocks。
3. 让 resume 链路失去 preserved tail。

这也是很多“自己写的简易压缩器”最容易出 bug 的地方。

### 13.4 它把 prompt cache 也当成压缩系统的一部分

压缩系统里有多处明显围绕 cache 做优化：

1. `runForkedAgent(...)` 复用 prompt cache。
2. `cached microcompact` 用 `cache_edits` 而不是本地改消息。
3. `notifyCompaction(...)` / `notifyCacheDeletion(...)` 用于消除 cache-break 误报。
4. 会区分压缩后的首次请求和普通缓存失效。

因此，这套系统的目标不仅是“token 变少”，而是“减少 token 的同时尽量保住缓存命中率”。

---

## 14. 当前快照中无法完整展开的部分

下面这些能力在当前仓库里能看到调用点，但看不到实现文件：

1. `reactiveCompact`
   `src/commands/compact/compact.ts` 和 `src/query.ts` 都会在特定 feature flag 下调用它，说明它主要负责处理 prompt-too-long 或媒体大小错误后的“反应式压缩”。但当前快照缺少实现源码，所以只能确认角色，不能确认内部算法。

2. `cachedMicrocompact`
   `microCompact.ts` 已经能看出它会维护注册过的工具结果、生成 `cache_edits`、追踪 pinned edits，但真正“删哪些 tool result”的策略在缺失的 `cachedMicrocompact.js` 中。

3. `snipCompact` / `snipProjection`
   从 `query.ts` 和 `getMessagesAfterCompactBoundary(...)` 可看出它们在 microcompact 之前执行，用于进一步裁剪历史，但当前快照无法确认具体保留策略。

4. `contextCollapse`
   不是 `compact/` 目录的一部分，但在 query 主循环里与 autocompact 串联，明显也是更高层次的上下文治理方案。

因此，本报告可以完整解释当前仓库中“可见的上下文压缩主链路”，但对这些缺失模块只能做到“基于调用点的边界内推断”。

---

## 15. 最终总结

Claude Code 的上下文压缩实现，核心不是“把聊天记录概括一下”，而是建立了一套面向工程场景的上下文重写机制：

1. 用 `compact_boundary` 作为活动上下文的切割点。
2. 用专门的 no-tools prompt 生成高保真 handoff 摘要。
3. 用 `microcompact` 优先清理大块工具结果。
4. 用 `session memory` 优先替代重新摘要。
5. 用保留段 relink、附件恢复和 hooks 恢复运行现场。
6. 用 transcript 持久化和 resume 逻辑把压缩边界变成真正的恢复点。

从代码结构上看，这套实现已经不是“一个命令”，而是贯穿：

1. prompt 设计
2. query 执行链
3. UI
4. transcript 持久化
5. 状态缓存
6. session 恢复

的完整子系统。

如果只用一句话概括它的实现方式，可以写成：

> Claude Code 通过“边界消息 + 摘要消息 + 保留原文段 + 恢复附件 + 缓存编辑 + 恢复点持久化”来实现上下文压缩，而不是单纯靠一次总结覆盖全部历史。

