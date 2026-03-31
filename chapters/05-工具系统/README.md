# 第五章：工具系统

工具系统是 Claude Code 的"四肢"——模型通过工具与外部世界交互：读写文件、执行命令、搜索代码、派生子 Agent、获取网页内容。本章将从类型系统出发，深入每个核心工具的设计细节，揭示一个安全、可扩展、高性能的工具框架是如何构建的。

## 5.1 Tool 类型系统详解

### 5.1.1 核心接口

`Tool` 是一个带有三个类型参数的泛型接口：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  aliases?: string[]
  searchHint?: string
  readonly inputSchema: Input
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean
  readonly strict?: boolean
  isMcp?: boolean

  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  prompt(options): Promise<string>
  description(input, options): Promise<string>

  // 能力标记
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // 权限
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>

  // UI 渲染
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null

  // 分类与搜索
  toAutoClassifierInput(input): unknown
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }

  // 结果映射
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam

  // 可观测性
  backfillObservableInput?(input: Record<string, unknown>): void
  extractSearchText?(out: Output): string
  getPath?(input): string
}
```

三个类型参数的含义：
- **Input**：工具输入的 Zod schema 类型，用于参数验证
- **Output**：工具执行结果的类型
- **P**：进度事件的类型（不同工具报告不同格式的进度）

### 5.1.2 ToolResult 与消息注入

工具执行的返回值不仅仅是数据——它还可以注入额外的消息到对话流：

```typescript
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

- **data**：工具的主要输出，会通过 `mapToolResultToToolResultBlockParam` 转换为 API 格式
- **newMessages**：注入到消息流中的额外消息（例如 AgentTool 完成后注入子 Agent 的摘要）
- **contextModifier**：修改 `ToolUseContext` 的回调（例如切换工作目录）。注意：只对非并发安全的工具生效
- **mcpMeta**：MCP 协议的元数据透传

### 5.1.3 ToolUseContext

`ToolUseContext` 是工具执行的运行时上下文，包含工具运行所需的一切：

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    refreshTools?: () => Tools
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>
  messages: Message[]
  agentId?: AgentId
  agentType?: string
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }
  queryTracking?: QueryChainTracking
  contentReplacementState?: ContentReplacementState
  // ...更多字段
}
```

关键设计：
- **不可变选项 vs 可变状态**：`options` 内的字段倾向于只读配置，而 `readFileState`、`messages` 等是跨调用累积的可变状态。
- **Agent 作用域**：`agentId` 和 `agentType` 将工具执行绑定到特定 Agent，子 Agent 的工具调用不会影响主线程的状态。
- **文件读取限制**：`fileReadingLimits` 允许不同 Agent 类型有不同的读取上限，防止子 Agent 消耗过多 token。

## 5.2 buildTool 工厂模式

`buildTool` 是所有工具定义的入口，它填充默认值并确保类型安全：

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

默认值遵循"失败安全"（fail-closed）原则：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,   // 默认不允许并发
  isReadOnly: () => false,           // 默认假设有写操作
  isDestructive: () => false,
  checkPermissions: (input) =>       // 默认允许（交给通用权限系统）
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',   // 默认跳过安全分类
  userFacingName: () => '',
}
```

`ToolDef` 类型与 `Tool` 类型的关系：

```typescript
export type ToolDef<Input, Output, P> =
  Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>
```

工具定义可以省略 `DefaultableToolKeys` 中的方法，`buildTool` 会自动补全。这避免了每个工具都要写一堆 `isConcurrencySafe() { return false }` 的模板代码。

实际使用中，工具定义通过 `satisfies ToolDef<InputSchema, Output>` 获得类型检查：

```typescript
export const TaskCreateTool = buildTool({
  name: TASK_CREATE_TOOL_NAME,
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  // ... 具体实现
} satisfies ToolDef<InputSchema, Output>)
```

## 5.3 工具池组装（assembleToolPool）

工具池的组装是一个多阶段过程：

```
getAllBaseTools()  →  getTools(permissionContext)  →  assembleToolPool(permCtx, mcpTools)
      │                      │                              │
      │  (所有内置工具)       │  (模式过滤+deny 规则)       │  (+ MCP 工具, 去重排序)
      ▼                      ▼                              ▼
   ~50个工具             过滤后的内置工具            最终工具集
```

### 5.3.1 getAllBaseTools

这是所有内置工具的"全集"。它通过条件导入实现 feature gating 和 dead code elimination：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 嵌入式搜索工具检测：ant-native 构建中 bfs/ugrep 嵌入了 bun binary
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    // feature gate 控制的条件工具
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,
    ...(RemoteTriggerTool ? [RemoteTriggerTool] : []),
    ...(MonitorTool ? [MonitorTool] : []),
    // 环境变量控制的条件工具
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    // 运行时 gate
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    // 测试工具
    ...(process.env.NODE_ENV === 'test' ? [TestingPermissionTool] : []),
    // ...
  ]
}
```

工具列表使用 `feature()` 和环境变量进行三重门控：
1. **编译时**：`feature('HISTORY_SNIP')` 通过 `bun:bundle` 在构建时消除不可达代码
2. **加载时**：`process.env.USER_TYPE === 'ant'` 在模块加载时决定
3. **运行时**：`isTodoV2Enabled()` 在函数调用时检查

### 5.3.2 assembleToolPool

最终的工具池组装将内置工具和 MCP 工具合并：

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 分区排序以保持 prompt cache 稳定性
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

排序策略的精妙之处在于：内置工具和 MCP 工具**分别**排序后再连接，而非混合排序。这是因为 Anthropic API 的 cache 策略在内置工具结束后设置 cache breakpoint；如果 MCP 工具穿插到内置工具中间，每次 MCP 工具变化都会使所有下游的 cache key 失效。

`uniqBy('name')` 确保内置工具优先——当 MCP 工具与内置工具同名时，内置工具保留。

## 5.4 核心工具深度剖析

### 5.4.1 BashTool：命令执行引擎

BashTool 是使用频率最高、安全要求最严的工具。

**输入 Schema**

```typescript
const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional())
    .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional()
    .describe('Clear, concise description of what this command does...'),
  run_in_background: semanticBoolean(z.boolean().optional())
    .describe('Set to true to run this command in the background...'),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
    .describe('Set this to true to dangerously override sandbox mode...'),
  _simulatedSedEdit: z.object({
    filePath: z.string(),
    newContent: z.string()
  }).optional()
    .describe('Internal: pre-computed sed edit result from preview')
}))
```

注意几个设计细节：
- **`semanticNumber` / `semanticBoolean`**：这些是特殊的 Zod 类型，支持模型输出语义化的值（如 `"true"` 字符串被解析为 `true` 布尔值）
- **`_simulatedSedEdit` 被从模型可见 schema 中移除**：它是内部字段，由 SedEditPermissionRequest 在用户批准后设置。如果暴露给模型，模型可以绕过权限检查和沙箱
- **`run_in_background` 在禁用时被移除**：通过 `.omit()` 从 schema 中物理删除，而非让模型看到一个无效参数

**命令安全分类**

BashTool 维护了多个命令分类集合：

```typescript
// 搜索命令（可折叠显示）
const BASH_SEARCH_COMMANDS = new Set([
  'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
])

// 读取命令（可折叠显示）
const BASH_READ_COMMANDS = new Set([
  'cat', 'head', 'tail', 'less', 'more',
  'wc', 'stat', 'file', 'strings',
  'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
])

// 语义中性命令（不影响管道的读/搜索性质）
const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set([
  'echo', 'printf', 'true', 'false', ':'
])

// 静默命令（成功时无输出）
const BASH_SILENT_COMMANDS = new Set([
  'mv', 'cp', 'rm', 'mkdir', 'rmdir', 'chmod', 'chown', ...
])
```

`isSearchOrReadBashCommand` 对复合命令（pipeline）进行语义分析：

```typescript
export function isSearchOrReadBashCommand(command: string): {
  isSearch: boolean; isRead: boolean; isList: boolean
} {
  let partsWithOperators = splitCommandWithOperators(command)

  for (const part of partsWithOperators) {
    // 跳过重定向目标
    if (skipNextAsRedirectTarget) { ... }
    // 跳过操作符
    if (part === '||' || part === '&&' || part === '|' || part === ';') continue
    // 跳过语义中性命令
    if (BASH_SEMANTIC_NEUTRAL_COMMANDS.has(baseCommand)) continue
    // 如果任一非中性命令不是搜索/读取，整个管道都不是
    if (!isPartSearch && !isPartRead && !isPartList) {
      return { isSearch: false, isRead: false, isList: false }
    }
  }
  return { isSearch: hasSearch, isRead: hasRead, isList: hasList }
}
```

核心规则：**pipeline 中所有非中性命令都必须是搜索/读取命令**，整个命令才被视为搜索/读取。例如 `ls dir && echo "---" && ls dir2` 被正确识别为列表操作（echo 是中性的）。

**沙箱模式**

BashTool 支持沙箱执行，通过 `SandboxManager` 提供隔离环境：

```typescript
import { SandboxManager } from '../../utils/sandbox/sandbox-adapter.js'
```

`dangerouslyDisableSandbox` 参数允许显式关闭沙箱，但这个字段的命名本身就是一种安全提示——它明确告诉模型这是一个"危险"操作。

**设备路径阻断**

在 FileReadTool 中可以看到对设备路径的保护：

```typescript
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero', '/dev/random', '/dev/urandom', '/dev/full',  // 无限输出
  '/dev/stdin', '/dev/tty', '/dev/console',                   // 阻塞等待输入
  '/dev/stdout', '/dev/stderr',                                // 无意义读取
  '/dev/fd/0', '/dev/fd/1', '/dev/fd/2',                     // fd 别名
])
```

这防止了模型尝试读取 `/dev/random`（导致进程挂起）或 `/dev/zero`（导致内存耗尽）的情况。

### 5.4.2 FileReadTool：多格式读取引擎

FileReadTool 的输入 schema 支持文件路径、行范围和 PDF 页码：

```typescript
const inputSchema = lazySchema(() => z.strictObject({
  file_path: z.string().describe('The absolute path to the file to read'),
  offset: semanticNumber(z.number().optional())
    .describe('The line number to start reading from'),
  limit: semanticNumber(z.number().optional())
    .describe('The number of lines to read'),
  pages: z.string().optional()
    .describe('Page range for PDF files (e.g., "1-5", "3", "10-20")'),
}))
```

FileReadTool 支持多种文件格式，每种格式有独立的处理路径：

```
FileReadTool.call(file_path)
  │
  ├── 文本文件 → readFileInRange → addLineNumbers → 返回带行号文本
  ├── 图片文件 (png/jpg/gif/webp) → 读取 buffer → 可选压缩 → Base64 编码
  ├── PDF 文件 → extractPDFPages / readPDF → 提取文本或图片
  ├── Notebook (.ipynb) → readNotebook → mapNotebookCellsToToolResult
  └── 二进制文件 → 拒绝读取并建议使用 Bash
```

**macOS 截图路径兼容**

一个有趣的细节是对 macOS 截图路径的处理：

```typescript
function getAlternateScreenshotPath(filePath: string): string | undefined {
  const filename = path.basename(filePath)
  const amPmPattern = /^(.+)([ \u202F])(AM|PM)(\.png)$/
  const match = filename.match(amPmPattern)
  if (!match) return undefined

  const currentSpace = match[2]
  const alternateSpace = currentSpace === ' ' ? THIN_SPACE : ' '
  return filePath.replace(...)
}
```

不同版本的 macOS 在截图文件名中 AM/PM 前使用不同的空格字符（常规空格 vs. U+202F 窄不间断空格）。FileReadTool 会自动尝试替代字符，避免因为不可见的空格差异导致"文件不存在"。

**maxResultSizeChars 设计**

FileReadTool 的 `maxResultSizeChars` 是 `Infinity`：

```typescript
maxResultSizeChars: Infinity  // 永不持久化到磁盘
```

原因在注释中解释得很清楚：如果读取结果被持久化到磁盘文件，模型可能尝试读取那个磁盘文件，导致循环读取。FileReadTool 通过自身的 `fileReadingLimits` 来控制输出大小。

### 5.4.3 FileEditTool：精确文本编辑

FileEditTool 使用"查找并替换"模式（而非行号）进行编辑：

```typescript
const inputSchema = lazySchema(() => z.strictObject({
  file_path: z.string(),
  old_string: z.string().describe('The text to replace'),
  new_string: z.string().describe('The text to replace it with'),
  replace_all: z.boolean().optional().default(false)
    .describe('Replace all occurrences (default false)'),
}))
```

这个设计决策有深刻的原因：
1. **行号不稳定**：在多轮对话中，文件可能被其他工具或用户修改，行号会漂移
2. **上下文自含**：`old_string` 本身就提供了编辑点的上下文，无需额外信息

**模糊匹配（findActualString）**

当 `old_string` 在文件中找不到精确匹配时，FileEditTool 会尝试模糊匹配：

```typescript
import { findActualString, preserveQuoteStyle } from './utils.js'
```

`findActualString` 处理的常见场景包括：
- 引号风格差异（如模型将 `'` 改为 `"`，或反过来）
- 缩进差异（Tab vs. 空格）
- 行尾差异（CR/LF vs. LF）

**引号保持（preserveQuoteStyle）**

```typescript
import { preserveQuoteStyle } from './utils.js'
```

如果编辑的上下文中原文件使用单引号，而模型输出使用双引号，`preserveQuoteStyle` 会自动将 `new_string` 中的引号风格调整为与原文一致。

**输入验证**

```typescript
async validateInput(input: FileEditInput, toolUseContext: ToolUseContext) {
  // 1. 检查团队内存文件中的密钥泄露
  const secretError = checkTeamMemSecrets(fullFilePath, new_string)
  if (secretError) return { result: false, message: secretError, errorCode: 0 }

  // 2. 检查空操作
  if (old_string === new_string) return { result: false, ... }

  // 3. 检查 deny 规则
  const denyRule = matchingRuleForInput(fullFilePath, ..., 'edit', 'deny')
  if (denyRule !== null) return { result: false, ... }

  // 4. 安全：跳过 UNC 路径的文件系统操作（防止 NTLM 凭据泄露）
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//'))
    return { result: true }

  // 5. 文件大小检查（防止 OOM）
  const { size } = await fs.stat(fullFilePath)
  if (size > MAX_EDIT_FILE_SIZE) return { result: false, ... }
}
```

注意第 4 点：在 Windows 上，对 UNC 路径调用 `fs.existsSync()` 会触发 SMB 认证，可能导致 NTLM 凭据泄露到恶意服务器。这是一个安全相关的重要细节。

### 5.4.4 AgentTool：子 Agent 派生

AgentTool 是所有工具中最复杂的，它管理子 Agent 的完整生命周期。

**输入 Schema 的动态组装**

AgentTool 的 schema 根据 feature gate 动态组装：

```typescript
// 基础 schema
const baseInputSchema = lazySchema(() => z.object({
  description: z.string(),
  prompt: z.string(),
  subagent_type: z.string().optional(),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
}))

// 完整 schema = 基础 + 多 Agent 参数 + 隔离模式
const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional(),
    team_name: z.string().optional(),
    mode: permissionModeSchema().optional(),
  })
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree', 'remote']).optional(),
    cwd: z.string().optional(),
  })
})

// 根据 feature gate 移除不可用的参数
export const inputSchema = lazySchema(() => {
  const schema = feature('KAIROS') ? fullInputSchema() : fullInputSchema().omit({ cwd: true })
  return isBackgroundTasksDisabled || isForkSubagentEnabled()
    ? schema.omit({ run_in_background: true })
    : schema
})
```

这种 `.omit()` 模式比条件 spread 更安全——条件 spread 会导致 Zod 类型推断崩溃（字段类型坍缩为 `unknown`）。

**模型选择**

```typescript
async call({ prompt, subagent_type, model: modelParam, ... }, toolUseContext, ...) {
  const model = isCoordinatorMode() ? undefined : modelParam
  // ...
}
```

在 Coordinator Mode 下，模型参数被忽略——coordinator 模式有自己的模型分配策略。

**多种执行模式**

AgentTool 支持多种执行方式：

```
AgentTool.call()
  │
  ├── teamName && name → spawnTeammate()     // Tmux 窗格中的 teammate
  ├── isolation === 'remote' → teleportToRemote()  // 远程 CCR 环境
  ├── isolation === 'worktree' → createAgentWorktree() → runAgent()  // Git worktree 隔离
  ├── run_in_background → registerAsyncAgent() + runAsyncAgentLifecycle()  // 后台异步
  └── default → runAgent()  // 前台同步
```

**Guard 条件**

```typescript
// Teammate 不能派生 Teammate（团队名册是平的）
if (isTeammate() && teamName && name) {
  throw new Error('Teammates cannot spawn other teammates...')
}

// In-process teammate 不能跑后台任务（生命周期绑定在 leader 进程上）
if (isInProcessTeammate() && teamName && run_in_background === true) {
  throw new Error('In-process teammates cannot spawn background agents...')
}
```

### 5.4.5 GrepTool：基于 ripgrep 的搜索

GrepTool 将 ripgrep 的能力包装为一个结构化的工具接口：

```typescript
export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,
  searchHint: 'search file contents with regex (ripgrep)',
  maxResultSizeChars: 20_000,
  strict: true,

  isConcurrencySafe() { return true },   // 纯读操作，可并发
  isReadOnly() { return true },           // 不修改文件系统
  // ...
})
```

**head_limit 与分页**

GrepTool 有一个精心设计的分页机制：

```typescript
const DEFAULT_HEAD_LIMIT = 250

function applyHeadLimit<T>(items: T[], limit: number | undefined, offset: number = 0) {
  // 显式传 0 = 无限制（逃生舱）
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  // 只在实际发生截断时报告 appliedLimit
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

设计要点：
- 默认 250 条限制防止大量搜索结果填满上下文（每次搜索可能产生 6-24K token）
- `appliedLimit` 只在截断实际发生时报告，让模型知道还有更多结果可以通过 `offset` 翻页
- `limit === 0` 是显式的"无限制"逃生舱（但带有"use sparingly"的文档警告）

**VCS 目录排除**

```typescript
const VCS_DIRECTORIES_TO_EXCLUDE = ['.git', '.svn', '.hg', '.bzr', '.jj', '.sl']
```

自动排除版本控制系统的内部目录，避免在 `.git/objects` 中搜索二进制内容。

### 5.4.6 WebFetchTool：网页获取

WebFetchTool 的特色在于其权限模型和内容处理：

**预审批域名**

```typescript
import { isPreapprovedHost } from './preapproved.js'

async checkPermissions(input, context) {
  try {
    const { url } = input
    const parsedUrl = new URL(url)
    if (isPreapprovedHost(parsedUrl.hostname, parsedUrl.pathname)) {
      return { behavior: 'allow', ... }
    }
  } catch { }
  // 否则走正常权限流程
}
```

预审批列表中的域名（如 npm、PyPI 等公共注册表）不需要用户确认，减少交互摩擦。

**域名级权限**

WebFetchTool 的权限粒度是域名级别而非 URL 级别：

```typescript
function webFetchToolInputToPermissionRuleContent(input) {
  const { url } = parsedInput.data
  const hostname = new URL(url).hostname
  return `domain:${hostname}`
}
```

用户批准 `example.com` 后，该域名下所有 URL 都可以访问，无需逐个确认。

**shouldDefer 标记**

```typescript
shouldDefer: true,
```

WebFetchTool 被标记为可延迟加载——当 ToolSearch 启用时，它的完整 schema 不会出现在初始 prompt 中，需要模型先通过 ToolSearch 发现它。这减少了初始 prompt 的 token 消耗。

**Auth Warning**

```typescript
async prompt() {
  return `IMPORTANT: WebFetch WILL FAIL for authenticated or private URLs.
Before using this tool, check if the URL points to an authenticated service
(e.g. Google Docs, Confluence, Jira, GitHub). If so, look for a specialized
MCP tool that provides authenticated access.
${DESCRIPTION}`
}
```

这段 prompt 始终包含，不管 ToolSearch 是否启用。之前有过条件性包含的尝试，但发现 ToolSearch 启用状态在不同 `query()` 调用间可能变化，导致 tool description 闪烁，破坏 API prompt cache——"两次连续的 cache miss per flicker event"。

### 5.4.7 SkillTool：技能执行隔离

SkillTool 在 fork 出的子 Agent 中执行 Skill：

```typescript
async function executeForkedSkill(
  command: Command & { type: 'prompt' },
  commandName: string,
  args: string | undefined,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<Progress>,
): Promise<ToolResult<Output>> {
  const agentId = createAgentId()
  // ... 执行 runAgent 并收集结果
}
```

执行隔离的目的是：
1. Skill 有自己的 token 预算，不会无限消耗主线程的上下文
2. Skill 的执行不会干扰主线程的消息历史
3. Skill 可以使用不同的模型（通过 frontmatter 中的 `model` 配置）

**技能发现追踪**

```typescript
const wasDiscoveredField =
  feature('EXPERIMENTAL_SKILL_SEARCH') && remoteSkillModules!.isSkillSearchEnabled()
    ? { was_discovered: context.discoveredSkillNames?.has(commandName) ?? false }
    : {}
```

系统追踪每个 Skill 是否通过 Skill Discovery 被发现，用于分析发现机制的有效性。

### 5.4.8 TaskCreateTool：任务追踪

TaskCreateTool 是一个相对简单但展示了完整工具生命周期的范例：

```typescript
export const TaskCreateTool = buildTool({
  name: TASK_CREATE_TOOL_NAME,
  shouldDefer: true,
  maxResultSizeChars: 100_000,

  isEnabled() { return isTodoV2Enabled() },
  isConcurrencySafe() { return true },

  async call({ subject, description, activeForm, metadata }, context) {
    // 1. 创建任务
    const taskId = await createTask(getTaskListId(), { subject, description, ... })

    // 2. 执行 TaskCreated hooks
    const generator = executeTaskCreatedHooks(taskId, subject, ...)
    for await (const result of generator) {
      if (result.blockingError) {
        blockingErrors.push(...)
      }
    }

    // 3. Hook 失败则回滚
    if (blockingErrors.length > 0) {
      await deleteTask(getTaskListId(), taskId)
      throw new Error(blockingErrors.join('\n'))
    }

    // 4. 自动展开任务面板
    context.setAppState(prev => ({ ...prev, expandedView: 'tasks' }))

    return { data: { task: { id: taskId, subject } } }
  },

  mapToolResultToToolResultBlockParam(content, toolUseID) {
    const { task } = content as Output
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: `Task #${task.id} created successfully: ${task.subject}`,
    }
  },
} satisfies ToolDef<InputSchema, Output>)
```

关键设计：
- **Hook 驱动回滚**：如果 `TaskCreated` hook 返回 blocking error，任务会被删除。这允许外部验证逻辑阻止不合规的任务创建。
- **UI 联动**：创建任务后自动展开任务面板（`expandedView: 'tasks'`），这是工具影响 UI 状态的示例。
- **结果简洁化**：`mapToolResultToToolResultBlockParam` 将结构化输出转换为模型可理解的简洁文本。

### 5.4.9 ToolSearchTool：延迟工具发现

ToolSearchTool 是 Tool Deferred Loading 策略的核心：

```typescript
export const ToolSearchTool = buildTool({
  name: TOOL_SEARCH_TOOL_NAME,
  // ...
})
```

**搜索策略**

ToolSearchTool 支持两种查询模式：
1. **直接选择**：`"select:Read,Edit,Grep"` — 按名称精确匹配
2. **关键词搜索**：`"notebook jupyter"` — 基于工具名称和描述的关键词评分

```typescript
function parseToolName(name: string): { parts: string[]; full: string; isMcp: boolean } {
  if (name.startsWith('mcp__')) {
    const withoutPrefix = name.replace(/^mcp__/, '').toLowerCase()
    const parts = withoutPrefix.split('__').flatMap(p => p.split('_'))
    return { parts, full: withoutPrefix.replace(/__/g, ' ').replace(/_/g, ' '), isMcp: true }
  }

  // CamelCase 拆分
  const parts = name
    .replace(/([a-z])([A-Z])/g, '$1 $2')
    .replace(/_/g, ' ')
    .toLowerCase()
    .split(/\s+/)
    .filter(Boolean)
  return { parts, full: parts.join(' '), isMcp: false }
}
```

**描述缓存**

工具描述的获取通过 memoize 缓存：

```typescript
const getToolDescriptionMemoized = memoize(
  async (toolName: string, tools: Tools): Promise<string> => {
    const tool = findToolByName(tools, toolName)
    if (!tool) return ''
    return tool.prompt({ ... })
  },
  (toolName: string) => toolName,
)
```

当 deferred tools 集合变化时（MCP 服务器连接/断开），缓存自动失效：

```typescript
function maybeInvalidateCache(deferredTools: Tools): void {
  const currentKey = getDeferredToolsCacheKey(deferredTools)
  if (cachedDeferredToolNames !== currentKey) {
    getToolDescriptionMemoized.cache.clear?.()
    cachedDeferredToolNames = currentKey
  }
}
```

## 5.5 工具延迟加载（shouldDefer）策略

当工具总数超过阈值时，部分工具被标记为"延迟加载"。模型在初始 prompt 中只看到工具名称（`defer_loading: true`），需要通过 ToolSearchTool 获取完整 schema 后才能调用。

```typescript
// Tool 接口中
readonly shouldDefer?: boolean    // 工具自荐可被延迟
readonly alwaysLoad?: boolean     // 工具声明永不延迟
```

延迟加载策略：

| 标记 | 行为 |
|------|------|
| `shouldDefer: true` | 工具声明自己可以被延迟（如 WebFetchTool、TaskCreateTool） |
| `alwaysLoad: true` | 工具声明永不延迟（如核心文件操作工具）。MCP 工具通过 `_meta['anthropic/alwaysLoad']` 设置 |
| 无标记 | 由系统根据工具总数和重要性动态决定 |

`searchHint` 字段辅助延迟工具的发现：

```typescript
// BashTool
searchHint: undefined  // 不需要——永远不会被延迟

// WebFetchTool
searchHint: 'fetch and extract content from a URL'

// GrepTool
searchHint: 'search file contents with regex (ripgrep)'

// AgentTool
searchHint: 'delegate work to a subagent'
```

## 5.6 并发安全标记（isConcurrencySafe）

`isConcurrencySafe` 决定了工具是否可以与其他工具并行执行：

```typescript
// GrepTool — 纯读操作，安全并发
isConcurrencySafe() { return true }

// TaskCreateTool — 创建操作是幂等的，安全并发
isConcurrencySafe() { return true }

// FileEditTool — 默认 false（写操作可能冲突）
// 通过 buildTool 默认值：isConcurrencySafe: () => false
```

当工具不是并发安全时，StreamingToolExecutor 会串行执行它们。此外，非并发安全工具的 `contextModifier` 才会生效——它可以修改 `ToolUseContext`（例如在 `cd` 后更新工作目录），而这种修改在并发场景下是不安全的。

## 5.7 结果大小限制（maxResultSizeChars）

每个工具声明了其结果的最大字符数：

```typescript
// GrepTool
maxResultSizeChars: 20_000      // 20K 字符

// AgentTool
maxResultSizeChars: 100_000     // 100K 字符

// FileReadTool
maxResultSizeChars: Infinity    // 永不持久化

// BashTool
maxResultSizeChars: undefined   // 使用系统默认
```

当工具输出超过此限制时，结果会被持久化到磁盘文件，模型收到的是一个预览（前 N 字节）加上文件路径。`applyToolResultBudget`（在 query.ts 的预处理管线中）会进一步对所有消息中的工具结果施加聚合预算。

设为 `Infinity` 的工具（如 FileReadTool）被从聚合预算中排除：

```typescript
new Set(
  toolUseContext.options.tools
    .filter(t => !Number.isFinite(t.maxResultSizeChars))
    .map(t => t.name),
)
```

## 5.8 本章小结

Claude Code 的工具系统体现了以下核心设计原则：

1. **类型驱动设计**：`Tool<Input, Output, P>` 三参数泛型，配合 `buildTool` 工厂和 `satisfies` 类型守卫，在保持灵活性的同时确保了类型安全。Zod schema 既用于运行时验证，又驱动 TypeScript 类型推导。

2. **安全纵深防御**：从 schema 级别移除危险参数（`_simulatedSedEdit`）、设备路径阻断、UNC 路径保护、团队内存密钥检查、域名级权限——安全防线不是一层而是多层。

3. **渐进式信息披露**：`shouldDefer` + `ToolSearchTool` 实现了工具的延迟加载，只在模型需要时才展示完整 schema，有效控制了初始 prompt 的 token 消耗。

4. **并发与串行的精细控制**：`isConcurrencySafe` 标记让 StreamingToolExecutor 可以安全地并行执行无副作用的工具，而对有副作用的工具保持串行。

5. **可组合的工具池**：`assembleToolPool` 将内置工具、MCP 工具、feature gate 过滤、deny 规则过滤和 prompt cache 排序组合为一个干净的管道，每个环节独立可测。

6. **结果管理的经济学**：`maxResultSizeChars` + 磁盘持久化 + 聚合预算三级控制，在"给模型足够信息"和"不超出上下文窗口"之间找到平衡。`Infinity` 的逃生舱设计则避免了对读取类工具的循环引用。

