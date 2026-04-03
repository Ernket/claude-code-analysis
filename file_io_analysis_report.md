# 文件读写实现分析报告

这个项目的文件访问不是“让模型直接读磁盘”，而是走一套严格的 `Tool` 机制。模型只能通过 `Read / Write / Edit / Glob / Grep / Bash` 等工具间接访问文件系统；每次调用都会经过 `prompt` 约束、`Zod schema` 校验、业务校验、权限判断、hooks、执行器调度，最后才真正触发文件 I/O。

## 核心结论

- 文件读取核心是 `Read` 工具；它不仅能读文本，还能读图片、PDF、Jupyter Notebook。
- 文件写入分成两条路：`Write` 做整文件覆盖，`Edit` 做精确字符串替换。
- “先读再写/改”不是建议，而是硬约束；由 `readFileState` 缓存和文件修改时间校验共同保证。
- 文件搜索也被工具化了：找文件名用 `Glob`，找文件内容用 `Grep`，底层统一封装 `ripgrep`。
- 只读工具可并发执行，写工具默认串行执行；这是这个项目既快又稳的关键。

## 关键文件

| 能力 | 关键文件 | 作用 |
|---|---|---|
| Tool 基础框架 | `src/Tool.ts` | 定义 `ToolUseContext`、`buildTool()` 默认行为 |
| 工具执行链路 | `src/services/tools/toolExecution.ts` | schema 校验、业务校验、权限、hooks、调用工具 |
| 工具调度 | `src/services/tools/toolOrchestration.ts` | 把只读工具并发跑，把写工具串行跑 |
| 流式执行器 | `src/services/tools/StreamingToolExecutor.ts` | 流式接收 tool_use 时进行并发控制 |
| 文件读取 | `src/tools/FileReadTool/FileReadTool.ts` | `Read` 的 schema、校验、调用、结果映射 |
| 文件覆盖写入 | `src/tools/FileWriteTool/FileWriteTool.ts` | `Write` 的校验和整文件落盘 |
| 文件精确编辑 | `src/tools/FileEditTool/FileEditTool.ts` | `Edit` 的精确替换逻辑 |
| 文件搜索 | `src/tools/GlobTool/GlobTool.ts` | 用 glob 模式找文件 |
| 内容搜索 | `src/tools/GrepTool/GrepTool.ts` | 用 ripgrep 搜索文件内容 |
| 权限系统 | `src/utils/permissions/filesystem.ts` | 读写权限、工作目录、危险路径、UNC 安全 |
| 低层文本读取 | `src/utils/readFileInRange.ts` | 文本按行读取，支持 fast path 和 streaming path |
| 读状态缓存 | `src/utils/fileStateCache.ts` | 记录“读过什么文件、何时读的、是不是部分视图” |

## 整体链路

端到端流程是：

1. 系统 prompt 明确要求模型优先使用 `Read/Edit/Write/Glob/Grep`，不要直接用 shell 的 `cat/sed/rg/find`。
2. 模型输出一个 `tool_use(name, input)`。
3. `toolExecution.ts` 先做 `inputSchema.safeParse()`。
4. 再执行每个工具自己的 `validateInput()`。
5. 再跑 pre-tool hooks、权限判断、用户确认。
6. 权限通过后调用 `tool.call()`。
7. 返回结果后再映射成 `tool_result`，再跑 post-tool hooks。
8. 模型继续基于 `tool_result` 推理。

关键代码骨架在 `src/Tool.ts` 和 `src/services/tools/toolExecution.ts`：

```ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input, _ctx) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

```ts
const parsedInput = tool.inputSchema.safeParse(input)
const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)

const result = await tool.call(
  callInput,
  { ...toolUseContext, toolUseId: toolUseID, userModified: permissionDecision.userModified ?? false },
  canUseTool,
  assistantMessage,
)

const mappedToolResultBlock =
  tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
```

这里有两个很重要的设计点：

- `buildTool()` 的默认值是“保守”的：默认不可并发、默认不是只读、默认不 destructive，但也不会默认拒绝执行。
- 真正的安全边界不靠 prompt，而是 `validateInput()` 和 `checkPermissions()`。

## Prompt 层是怎么约束模型使用文件工具的

系统 prompt 明确要求模型优先使用专门工具，而不是 shell 命令。见 `src/constants/prompts.ts`：

```ts
`To read files use ${FILE_READ_TOOL_NAME} instead of cat, head, tail, or sed`
`To edit files use ${FILE_EDIT_TOOL_NAME} instead of sed or awk`
`To create files use ${FILE_WRITE_TOOL_NAME} instead of cat with heredoc or echo redirection`
`To search for files use ${GLOB_TOOL_NAME} instead of find or ls`
`To search the content of files, use ${GREP_TOOL_NAME} instead of grep or rg`
```

`Read` 工具自己的 prompt 也写得很细，见 `src/tools/FileReadTool/prompt.ts`：

```ts
return `Reads a file from the local filesystem...
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines starting from the beginning of the file
- This tool allows Claude Code to read images
- This tool can read PDF files
- This tool can read Jupyter notebooks (.ipynb files)
- This tool can only read files, not directories. To read a directory, use an ls command via the Bash tool.
`
```

所以这个项目不是“模型自己决定怎么读文件”，而是先通过 prompt 把行为收窄到固定工具，再由运行时去兜底。

## Read：文件读取是怎么实现的

`Read` 工具定义在 `src/tools/FileReadTool/FileReadTool.ts`。

它的输入 schema 很清晰：

- `file_path`: 绝对路径
- `offset`: 起始行
- `limit`: 读取行数
- `pages`: PDF 页范围

它还把默认读取限制做成了运行时可配置项，见 `src/tools/FileReadTool/limits.ts`：

- `maxSizeBytes`: 默认 256KB
- `maxTokens`: 默认 25000
- prompt 文案是否提示字节上限
- 是否鼓励用户做定向分段读取

`Read` 的校验逻辑在 `FileReadTool.ts`：

- 先解析 `pages`，防止非法 PDF 页范围。
- 再做 deny rule 检查。
- UNC 路径先短路，不提前触碰文件系统，避免 Windows 下的 NTLM 凭据泄漏。
- 非图片/PDF 的二进制文件直接拒绝。
- 特殊设备文件如 `/dev/zero`、`/dev/random`、`/dev/stdin` 会被拦掉，防止无限输出或阻塞。

核心校验片段：

```ts
if (hasBinaryExtension(fullFilePath) &&
    !isPDFExtension(ext) &&
    !IMAGE_EXTENSIONS.has(ext.slice(1))) {
  return {
    result: false,
    message: `This tool cannot read binary files...`,
  }
}

if (isBlockedDevicePath(fullFilePath)) {
  return {
    result: false,
    message: `Cannot read '${file_path}': this device file would block or produce infinite output.`,
  }
}
```

`Read` 真正执行时有一个非常关键的优化：重复读取去重。

```ts
const existingState = dedupKillswitch
  ? undefined
  : readFileState.get(fullFilePath)

if (existingState &&
    !existingState.isPartialView &&
    existingState.offset !== undefined) {
  const rangeMatch =
    existingState.offset === offset && existingState.limit === limit
  if (rangeMatch) {
    const mtimeMs = await getFileModificationTimeAsync(fullFilePath)
    if (mtimeMs === existingState.timestamp) {
      return {
        data: {
          type: 'file_unchanged',
          file: { filePath: file_path },
        },
      }
    }
  }
}
```

这说明它不是每次都把完整文件重新塞给模型；如果同一范围没变，就返回 `FILE_UNCHANGED_STUB`，避免浪费上下文 token。

文本读取底层走 `src/utils/readFileInRange.ts`。这个工具分两条路径：

- 小文件且 `< 10MB`：整文件读入内存再 split，快。
- 大文件、管道、设备：流式读取，只累计目标行范围，避免爆内存。

源码注释已经把设计写得很直白：

```ts
// Fast path (regular files < 10 MB):
//   readFile + in-memory split
//
// Streaming path (large files, pipes, devices, etc.):
//   Uses createReadStream with manual indexOf('\n') scanning.
//   Content is only accumulated for lines inside the requested range.
```

`Read` 文本分支的核心逻辑是：

- 调用 `readFileInRange()`
- 再做 token 上限验证
- 更新 `readFileState`
- 给返回文本加行号

行号是由 `src/utils/file.ts` 的 `addLineNumbers()` 实现的；`Edit` 的 prompt 会特别提醒模型，不要把行号前缀当成 `old_string` 的一部分。

`Read` 的返回结果还不只是“文件内容字符串”，而是经过格式化的 `tool_result`：

- 文本返回：`行号 + 内容 + system-reminder`
- 空文件：返回 warning
- offset 超出长度：返回 warning
- 自动记忆文件还会加 freshness note

它甚至会给 Read 结果附带一条恶意代码分析提醒：

```ts
export const CYBER_RISK_MITIGATION_REMINDER =
  '\n\n<system-reminder>\nWhenever you read a file, you should consider whether it would be considered malware...'
```

这说明 `Read` 结果不是纯 I/O，而是把“安全策略”直接注入到了模型上下文里。

## Read 对图片、PDF、Notebook 的分支

这部分是这个项目最像“产品级实现”的地方。

图片分支：

- 只读一次二进制 buffer
- 自动检测图片类型
- 按 token budget 压缩/缩放
- 返回多模态 image block
- 可附带图片尺寸 metadata

Notebook 分支：

- 用 `readNotebook()` 解析 `.ipynb`
- 把所有 cell 和 outputs 一起返回
- Notebook 内容过大时，提示改用 `jq` 只取部分 cells

PDF 分支：

- 如果给了 `pages`，走 `extractPDFPages()`，把对应页转成 JPG 再以 image blocks 发给模型
- 如果 PDF 很大，必须强制 `pages`
- 如果模型支持 PDF，则可能直接 `readPDF()`，再发 `document` block
- 如果模型不支持完整 PDF，也会退化成抽页图片

所以 `Read` 的本质不是“fs.readFile”，而是一个统一的“本地资源读取工具”，文本、图片、PDF、Notebook 都统一挂在这里。

## Write：整文件覆盖是怎么实现的

`Write` 工具定义在 `src/tools/FileWriteTool/FileWriteTool.ts`，对应 prompt 在 `src/tools/FileWriteTool/prompt.ts`。

它的 prompt 很明确：

```ts
return `Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one at the provided path.
- If this is an existing file, you MUST use the Read tool first to read the file's contents.
- Prefer the Edit tool for modifying existing files.
`
```

`Write` 的关键校验：

- 先做 team memory secret guard
- 再看 deny rule
- UNC 路径不提前访问磁盘
- 如果目标文件存在，必须已经读过，而且不能是 partial view
- 如果文件自从上次读取后被改过，必须重新读

关键代码：

```ts
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}

if (lastWriteTime > readTimestamp.timestamp) {
  return {
    result: false,
    message: 'File has been modified since read...',
  }
}
```

真正写入时：

- 先创建父目录
- 可选做 file history 备份
- 再读当前内容并做最后一次 staleness 检查
- 调 `writeTextContent()`
- 通知 LSP 和 VSCode
- 更新 `readFileState`
- 生成 diff patch 给 UI 展示

最值得注意的一点是这里：

```ts
writeTextContent(fullFilePath, content, enc, 'LF')
```

结合 `src/utils/file.ts` 的实现，这意味着：

- `Write` 不会强行继承旧文件的换行风格
- 它相信模型给出的 `content` 就是完整新文件内容
- 如果 `content` 里本来是 `LF`，最终文件就会是 `LF`
- 这跟 `Edit` 的“尽量保持原文件风格”是不同的设计哲学

所以可以把 `Write` 理解为“全量重写”，不是“保守补丁”。

## Edit：精确字符串替换是怎么实现的

`Edit` 工具定义在 `src/tools/FileEditTool/FileEditTool.ts`，对应 prompt 在 `src/tools/FileEditTool/prompt.ts`。

它的定位非常明确：不是 AST 编辑，不是 patch 语言，而是 exact string replace。

Prompt 原文已经说明了这一点：

```ts
return `Performs exact string replacements in files.

Usage:
- You must use your \`Read\` tool at least once in the conversation before editing.
- Never include any part of the line number prefix in the old_string or new_string.
- The edit will FAIL if \`old_string\` is not unique in the file.
- Use \`replace_all\` for replacing and renaming strings across the file.
`
```

输入 schema 也很简单：

- `file_path`
- `old_string`
- `new_string`
- `replace_all`

`Edit` 的校验比 `Write` 更严格：

- 文件超过 1GiB 直接拒绝
- `old_string === new_string` 拒绝
- 对不存在文件，只有 `old_string === ''` 时才允许，等价于“空文件创建”
- `.ipynb` 不允许用 `Edit`，必须转去 `NotebookEditTool`
- 必须已经做过完整读取
- 文件被外部改动后要重新读
- 如果 `old_string` 找不到，拒绝
- 如果匹配到多个位置但 `replace_all` 是 false，拒绝
- Claude 自己的 settings 文件还有额外校验逻辑

这段错误信息很有代表性，说明它是“精确替换”而不是“模糊编辑”：

```ts
message: `Found ${matches} matches of the string to replace, but replace_all is false.
To replace all occurrences, set replace_all to true.
To replace only one occurrence, please provide more context to uniquely identify the instance.`
```

真正的编辑过程：

- 先同步读取文件内容、编码、行尾风格
- 再做 staleness 校验
- 用 `findActualString()` 修正引号风格
- 用 `preserveQuoteStyle()` 保持原文件的 curly quotes 风格
- 生成 patch
- 用原始编码和原始 line endings 写回
- 通知 LSP / VSCode
- 更新 `readFileState`

这里的两个辅助函数最关键：

```ts
export function findActualString(fileContent: string, searchString: string): string | null {
  if (fileContent.includes(searchString)) {
    return searchString
  }

  const normalizedSearch = normalizeQuotes(searchString)
  const normalizedFile = normalizeQuotes(fileContent)

  const searchIndex = normalizedFile.indexOf(normalizedSearch)
  if (searchIndex !== -1) {
    return fileContent.substring(searchIndex, searchIndex + searchString.length)
  }

  return null
}
```

```ts
export function preserveQuoteStyle(
  oldString: string,
  actualOldString: string,
  newString: string,
): string {
  if (oldString === actualOldString) return newString
  ...
  if (hasDoubleQuotes) result = applyCurlyDoubleQuotes(result)
  if (hasSingleQuotes) result = applyCurlySingleQuotes(result)
  return result
}
```

这说明 `Edit` 并不是简单 `content.replace(old, new)`，而是：

- 先尝试做“引号归一化匹配”
- 再尽量把新字符串的引号风格贴合原文件
- 再生成结构化 diff

还有一个很重要的差异：`Edit` 会保留原文件的编码和换行风格。这跟 `Write` 的“全量重写”正好相反。

## readFileState：为什么它能强制“先读再写”

这一层在 `src/utils/fileStateCache.ts`：

```ts
export type FileState = {
  content: string
  timestamp: number
  offset: number | undefined
  limit: number | undefined
  isPartialView?: boolean
}
```

缓存本身是一个 LRU：

- 默认最多 100 个 entry
- 默认最多 25MB
- key 会被标准化，避免 Windows `/` 和 `\` 混用造成 miss

这层缓存承担了三件事：

- 记录模型已经看过哪个文件的什么内容
- 在 `Write/Edit` 时判断“你有没有先读”
- 在再次 `Read` 时做去重，直接返回 `file_unchanged`

所以它不是普通 cache，而是整个“文件一致性协议”的中心。

## Glob / Grep：文件搜索不是 shell，而是专门工具

`Glob` 的 prompt 很短，但方向很明确：

```ts
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Use this tool when you need to find files by name patterns
```

`Glob` 的底层实现不走 Node 递归，而是包了 ripgrep：

```ts
const args = [
  '--files',
  '--glob',
  searchPattern,
  '--sort=modified',
  ...(noIgnore ? ['--no-ignore'] : []),
  ...(hidden ? ['--hidden'] : []),
]

for (const pattern of ignorePatterns) {
  args.push('--glob', `!${pattern}`)
}

const allPaths = await ripGrep(args, searchDir, abortSignal)
```

这意味着：

- `Glob` 本质上是 `rg --files --glob ...`
- 默认包含 hidden
- 默认可以不遵守 `.gitignore`
- 会把 deny-rule 里的 read ignore patterns 下沉到 `--glob !pattern`

`Grep` 的 prompt更明确：

```ts
return `A powerful search tool built on ripgrep

Usage:
- ALWAYS use Grep for search tasks. NEVER invoke \`grep\` or \`rg\` as a Bash command.
- Supports full regex syntax
- Output modes: "content", "files_with_matches", "count"
`
```

`Grep` 的参数拼装：

```ts
const args = ['--hidden']

if (output_mode === 'files_with_matches') args.push('-l')
else if (output_mode === 'count') args.push('-c')

if (show_line_numbers && output_mode === 'content') args.push('-n')

if (multiline) args.push('-U', '--multiline-dotall')

if (pattern.startsWith('-')) args.push('-e', pattern)
else args.push(pattern)

if (type) args.push('--type', type)
if (glob) args.push('--glob', globPattern)
```

它还做了这些工程化处理：

- 过滤 `.git` 等 VCS 目录
- `--max-columns 500`，避免 minified/base64 污染结果
- 支持 `head_limit` 和 `offset`
- 返回结果时把绝对路径变成相对路径，节省 token
- 超时、buffer overflow、EAGAIN 重试都在 `src/utils/ripgrep.ts` 里统一处理

所以 `Glob/Grep` 不是语法糖，而是比直接 shell 更可控、更省 token、更安全的专门能力。

## 权限与安全模型

权限系统在 `src/utils/permissions/filesystem.ts`。

读权限 `checkReadPermissionForTool()` 的优先级大致是：

- UNC 路径先拦
- suspicious Windows path 先拦
- read-specific deny
- read-specific ask
- 如果 edit 已经 allow，则 read 也 allow
- 工作目录内默认 allow
- 内部路径如 session memory / plan / scratchpad / task / teams 等 allow
- 否则 ask

写权限 `checkWritePermissionForTool()` 的优先级更严格：

- edit deny
- 内部可写路径 allow
- `.claude/**` 的 session-scoped 特例
- 危险路径安全检查；`.git`、`.vscode`、`.idea`、shell config、UNC 都会提升为 ask
- edit ask
- 工作目录 + `acceptEdits` 模式 allow
- allow rule
- 否则 ask

这意味着安全边界主要是代码里的权限判定，不是 prompt。

## 并发与调度：为什么读操作快，写操作稳

`Read/Grep/Glob` 都显式标记了并发安全：

- `isConcurrencySafe() { return true }`
- `isReadOnly() { return true }`

调度器 `src/services/tools/toolOrchestration.ts` 会先按这个属性分批：

```ts
// 1. A single non-read-only tool, or
// 2. Multiple consecutive read-only tools
function partitionToolCalls(...) { ... }

if (isConcurrencySafe) {
  for await (const update of runToolsConcurrently(...)) { ... }
} else {
  for await (const update of runToolsSerially(...)) { ... }
}
```

并且流式执行器 `src/services/tools/StreamingToolExecutor.ts` 的目标也写得很清楚：

```ts
/**
 * Executes tools as they stream in with concurrency control.
 * - Concurrent-safe tools can execute in parallel with other concurrent-safe tools
 * - Non-concurrent tools must execute alone
 * - Results are buffered and emitted in the order tools were received
 */
```

这就是它“读得快、写得稳”的底层原因。

## 你可以把这套实现理解为下面这张图

```text
用户问题
  -> System Prompt / Tool Prompt
  -> 模型输出 tool_use(Read / Write / Edit / Glob / Grep ...)
  -> toolExecution.runToolUse()
       -> Zod schema 校验
       -> validateInput()
       -> pre-tool hooks
       -> checkPermissions() / canUseTool()
       -> tool.call()
       -> mapToolResultToToolResultBlockParam()
       -> post-tool hooks
  -> tool_result 回到模型
  -> 模型继续推理
```

## 最终判断

这个项目的文件读取/写入实现非常“工程化”：

- 上层用 prompt 强约束模型选正确工具。
- 中层用 Tool 框架统一 schema、权限、hooks、并发控制。
- 下层把文本读取、图片压缩、PDF 抽页、Notebook 解析、ripgrep 搜索、文件一致性校验都做成了独立能力。
- 最关键的一层是 `readFileState + mtime/content 校验`，它把“先读再写/改”从 prompt 提升成了运行时协议。

如果只用一句话概括：这个项目并不是“LLM 直接操作文件系统”，而是“LLM 驱动的一套严格受控的本地工具运行时”。
