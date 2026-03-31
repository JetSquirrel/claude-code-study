---
title: "第六章：权限与安全模型"
weight: 60
bookCollapseSection: false
---


Claude Code 作为一个能够读写文件、执行 Shell 命令的 AI Agent，其权限与安全系统的设计直接决定了产品的可信度。本章将深入分析 Claude Code 的多层权限架构，从权限模式定义到 Bash 命令安全检测，从文件系统保护到 Auto 模式的 AI 分类器，全面剖析其"纵深防御"的安全哲学。

## 6.1 权限模式全景

Claude Code 的权限系统围绕 **PermissionMode** 展开，它决定了在每一次工具调用前，系统如何决策"放行、询问还是拒绝"。源码中定义了两个层级的模式集合：

```typescript
// types/permissions.ts
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]

export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

各模式的语义如下：

| 模式 | 含义 | 典型场景 |
|------|------|---------|
| `default` | 每次工具调用都需用户确认 | 日常交互使用 |
| `plan` | 规划模式，只读不写 | 代码审查、方案设计 |
| `acceptEdits` | 自动接受文件编辑，其他操作仍需确认 | 快速编码 |
| `bypassPermissions` | 绕过所有权限检查 | 完全信任场景 |
| `dontAsk` | 从不询问，无权限则直接拒绝 | 无头(headless)自动化 |
| `auto` | AI 分类器自动决策 | 内部/高级用户 |
| `bubble` | 冒泡到父终端的权限提示 | Fork 子 Agent |

`auto` 模式通过 feature flag `TRANSCRIPT_CLASSIFIER` 控制，在构建时条件编译：

```typescript
export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : ([] as const)),
] as const satisfies readonly PermissionMode[]
```

每个模式在 `PermissionMode.ts` 中都有对应的 UI 配置：

```typescript
const PERMISSION_MODE_CONFIG: Partial<Record<PermissionMode, PermissionModeConfig>> = {
  default: { title: 'Default', shortTitle: 'Default', symbol: '', color: 'text', external: 'default' },
  plan:    { title: 'Plan Mode', shortTitle: 'Plan', symbol: PAUSE_ICON, color: 'planMode', external: 'plan' },
  acceptEdits: { title: 'Accept edits', shortTitle: 'Accept', symbol: '⏵⏵', color: 'autoAccept', external: 'acceptEdits' },
  bypassPermissions: { title: 'Bypass Permissions', shortTitle: 'Bypass', symbol: '⏵⏵', color: 'error', external: 'bypassPermissions' },
  dontAsk: { title: "Don't Ask", shortTitle: 'DontAsk', symbol: '⏵⏵', color: 'error', external: 'dontAsk' },
  // auto 模式仅在 TRANSCRIPT_CLASSIFIER feature 开启时注入
}
```

## 6.2 PermissionRule 结构

权限规则是权限系统的基础数据单元。每条规则由三部分组成：

```typescript
export type PermissionRule = {
  source: PermissionRuleSource      // 规则来源
  ruleBehavior: PermissionBehavior  // 行为：allow | deny | ask
  ruleValue: PermissionRuleValue    // 匹配条件
}

export type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json
  | 'localSettings'     // .claude/settings.local.json
  | 'flagSettings'      // 管理策略
  | 'policySettings'    // 企业策略
  | 'cliArg'            // --allowed-tools CLI 参数
  | 'command'           // 命令定义
  | 'session'           // 会话中用户授权

export type PermissionRuleValue = {
  toolName: string       // 工具名称，如 "Bash"
  ruleContent?: string   // 可选的内容匹配，如 "git commit:*"
}
```

规则的匹配粒度非常灵活。例如：
- `Bash` —— 匹配所有 Bash 命令
- `Bash(git commit:*)` —— 匹配以 `git commit` 开头的 Bash 命令
- `mcp__server1` —— 匹配来自 MCP 服务器 `server1` 的所有工具
- `Agent(Explore)` —— 匹配特定的 Agent 类型

规则存储在 `ToolPermissionContext` 中，按来源和行为组织：

```typescript
export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  // ...
}
```

## 6.3 权限决策管线

当 Agent 发起一次工具调用时，权限检查遵循严格的管线流程。核心函数 `hasPermissionsToUseTool` 是整个管线的入口：

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool, input, context, assistantMessage, toolUseID,
): Promise<PermissionDecision> => {
  const result = await hasPermissionsToUseToolInner(tool, input, context)

  // 成功允许：重置连续拒绝计数器
  if (result.behavior === 'allow') {
    // ... 重置 denial tracking
    return result
  }

  // dontAsk 模式：将 ask 转为 deny
  if (result.behavior === 'ask') {
    if (appState.toolPermissionContext.mode === 'dontAsk') {
      return { behavior: 'deny', message: DONT_ASK_REJECT_MESSAGE(tool.name), ... }
    }

    // auto 模式：使用 AI 分类器替代用户提示
    if (feature('TRANSCRIPT_CLASSIFIER') && mode === 'auto') {
      // ... 分类器决策逻辑
    }
  }

  return result
}
```

决策管线可以概括为以下阶段：

**阶段1：安全检查 (Safety Check)**
每个工具的 `checkPermissions` 方法首先执行硬编码的安全检查。例如 Bash 工具检测破坏性命令、设备路径访问等。这些检查返回 `deny` 或 `ask`，某些结果被标记为 `classifierApprovable` —— 意味着 auto 模式下分类器可以覆盖。

**阶段2：规则匹配 (Rule Matching)**
系统从所有来源收集规则并进行匹配：

```typescript
export function getAllowRules(context: ToolPermissionContext): PermissionRule[] {
  return PERMISSION_RULE_SOURCES.flatMap(source =>
    (context.alwaysAllowRules[source] || []).map(ruleString => ({
      source,
      ruleBehavior: 'allow',
      ruleValue: permissionRuleValueFromString(ruleString),
    })),
  )
}
```

匹配优先级为：deny 规则 > ask 规则 > allow 规则。工具级匹配和内容级匹配分开处理：

```typescript
function toolMatchesRule(tool, rule): boolean {
  if (rule.ruleValue.ruleContent !== undefined) return false  // 内容规则不做工具级匹配
  const nameForRuleMatch = getToolNameForPermissionCheck(tool)
  if (rule.ruleValue.toolName === nameForRuleMatch) return true
  // MCP 服务器级权限：mcp__server1 匹配 mcp__server1__tool1
  // ...
}
```

**阶段3：模式决策 (Mode Decision)**
根据当前 PermissionMode 做最终决策。`bypassPermissions` 直接放行，`plan` 模式对写操作返回 `ask`，`dontAsk` 模式将所有 `ask` 转为 `deny`。

**阶段4：分类器 (Classifier)**
在 auto 模式下，`ask` 结果不直接发送给用户，而是交给 YOLO 分类器评估（详见 6.7 节）。

**阶段5：用户交互**
最终的 `ask` 结果展示给用户，附带权限更新建议（suggestions）。用户可以选择 "always allow" 将规则持久化。

权限决策的结果类型系统设计精密：

```typescript
export type PermissionDecision<Input> =
  | PermissionAllowDecision<Input>   // 允许，可带更新后的 input
  | PermissionAskDecision<Input>     // 需要用户确认
  | PermissionDenyDecision           // 直接拒绝

export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'sandboxOverride'; reason: string }
  // ...
```

每个决策都携带 `decisionReason`，记录"为什么做出这个决策"，这不仅用于 UI 展示，还用于分析追踪和安全审计。

## 6.4 Bash 命令安全

Bash 工具是权限系统中最复杂的子系统。源码在 `bashSecurity.ts` 和 `bashPermissions.ts` 中实现了多层防御。

### 6.4.1 命令解析与验证管线

`bashCommandIsSafe` 函数构建了一个验证器链：

```typescript
type ValidationContext = {
  originalCommand: string
  baseCommand: string
  unquotedContent: string
  fullyUnquotedContent: string
  fullyUnquotedPreStrip: string
  unquotedKeepQuoteChars: string
  treeSitter?: TreeSitterAnalysis | null
}
```

每个验证器接收 `ValidationContext` 并返回 `PermissionResult`：`passthrough` 表示该验证器不关心，`allow` 表示确认安全，`ask` 或 `deny` 表示发现问题。

关键验证器包括：

**空命令检测**：
```typescript
function validateEmpty(context: ValidationContext): PermissionResult {
  if (!context.originalCommand.trim()) {
    return { behavior: 'allow', ... }
  }
  return { behavior: 'passthrough', message: 'Command is not empty' }
}
```

**不完整命令检测**：
```typescript
function validateIncompleteCommands(context: ValidationContext): PermissionResult {
  if (/^\s*\t/.test(originalCommand)) {
    return { behavior: 'ask', message: 'Command appears to be an incomplete fragment (starts with tab)' }
  }
  if (trimmed.startsWith('-')) {
    return { behavior: 'ask', message: 'Command appears to be an incomplete fragment (starts with flags)' }
  }
  // ...
}
```

### 6.4.2 命令替换与注入防护

`bashSecurity.ts` 定义了大量的命令替换模式检测：

```typescript
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /\$\[/, message: '$[] legacy arithmetic expansion' },
  { pattern: /~\[/, message: 'Zsh-style parameter expansion' },
  // ...
]
```

Zsh 特有的危险命令也被单独枚举：

```typescript
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',    // 模块加载：gateway to zsh/mapfile, zsh/system, zsh/zpty 等
  'emulate',     // -c flag 是 eval 等价物
  'sysopen', 'sysread', 'syswrite', 'sysseek',  // zsh/system 模块
  'zpty',        // 伪终端命令执行
  'ztcp',        // 网络外泄
  'zsocket',     // Unix/TCP socket
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', ...  // zsh/files 内建命令
])
```

### 6.4.3 引号内容提取

安全检查需要理解引号语义。`extractQuotedContent` 函数维护三种提取视图：

```typescript
function extractQuotedContent(command: string, isJq = false): QuoteExtraction {
  let withDoubleQuotes = ''    // 去除单引号内容
  let fullyUnquoted = ''       // 去除所有引号内容
  let unquotedKeepQuoteChars = ''  // 去除内容但保留引号字符
  // ...逐字符状态机解析
}
```

`unquotedKeepQuoteChars` 看似冗余，实际上对检测 `'x'#` 这类"引号相邻 # 号"的攻击模式至关重要。

### 6.4.4 只读命令白名单

`readOnlyValidation.ts` 维护了一个庞大的只读命令白名单，每个命令都精确定义了安全的 flag：

```typescript
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: {
      '-I': '{}',
      '-n': 'number',
      '-P': 'number',
      '-0': 'none',
      '-t': 'none',
      '-r': 'none',
      // 注意：-i 和 -e 被有意排除——GNU getopt 可选参数语义存在歧义
    }
  },
  ...GIT_READ_ONLY_COMMANDS,
  ...DOCKER_READ_ONLY_COMMANDS,
  ...GH_READ_ONLY_COMMANDS,
  ...RIPGREP_READ_ONLY_COMMANDS,
  // ...
}
```

注释中详细记录了为什么某些 flag 被排除。例如 `xargs -i` 和 `-e` 被移除的原因：GNU getopt 的 `i::` (optional-attached-arg) 语义与空格分隔参数的解析歧义可能导致安全绕过。

### 6.4.5 安全重定向剥离

在分析命令安全性时，已知安全的重定向会被预先剥离：

```typescript
function stripSafeRedirections(content: string): string {
  // SECURITY: 所有模式必须有尾部边界 (?=\s|$)
  // 否则 `> /dev/nullo` 会匹配 `/dev/null` 前缀
  return content
    .replace(/\s+2\s*>&\s*1(?=\s|$)/g, '')
    .replace(/[012]?\s*>\s*\/dev\/null(?=\s|$)/g, '')
    .replace(/\s*<\s*\/dev\/null(?=\s|$)/g, '')
}
```

注释中的安全提示值得注意：如果没有尾部边界，`echo hi > /dev/nullo` 会被错误地剥离为 `echo hi o`，导致文件写入被绕过。

## 6.5 文件系统权限

### 6.5.1 敏感文件保护

`filesystem.ts` 定义了两组受保护的路径：

```typescript
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules',
  '.bashrc', '.bash_profile', '.zshrc', '.zprofile', '.profile',
  '.ripgreprc', '.mcp.json', '.claude.json',
] as const

export const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude',
] as const
```

对这些路径的写操作会触发 `ask` 而非 `allow`，即使在 `acceptEdits` 模式下也是如此。在 auto 模式中，这些检查被标记为 `classifierApprovable: true`，意味着分类器可以根据上下文自行决策。

### 6.5.2 大小写规范化

在 macOS/Windows 等大小写不敏感的文件系统上，路径大小写可能被用来绕过安全检查：

```typescript
export function normalizeCaseForComparison(path: string): string {
  return path.toLowerCase()
}
```

所有路径比较都经过此函数，防止 `.cLauDe/Settings.locaL.json` 这样的变体绕过 `.claude/settings.local.json` 的保护。

### 6.5.3 Claude 配置文件保护

系统对自身配置文件有特殊的保护逻辑：

```typescript
function isClaudeConfigFilePath(filePath: string): boolean {
  if (isClaudeSettingsPath(filePath)) return true
  // 检查 .claude/commands、.claude/agents、.claude/skills 目录
  const commandsDir = join(getOriginalCwd(), '.claude', 'commands')
  const agentsDir = join(getOriginalCwd(), '.claude', 'agents')
  const skillsDir = join(getOriginalCwd(), '.claude', 'skills')
  return pathInWorkingPath(filePath, commandsDir) ||
         pathInWorkingPath(filePath, agentsDir) ||
         pathInWorkingPath(filePath, skillsDir)
}
```

修改 Claude 的配置文件（settings、commands、agents 定义）始终需要用户确认，这阻止了 AI 自我修改权限规则的可能。

### 6.5.4 Skill 范围隔离

当 Claude 编辑 `.claude/skills/` 中的文件时，系统提供了精细的权限范围建议：

```typescript
export function getClaudeSkillScope(filePath: string):
  { skillName: string; pattern: string } | null {
  // 解析出 skill 名称
  // 返回仅覆盖该 skill 子目录的 pattern，如 "/.claude/skills/mySkill/**"
  // 防止授权泄漏到其他 skill 或 .claude/ 下的其他敏感目录
}
```

这避免了用户在同意编辑一个 skill 时意外授权了对整个 `.claude/` 目录的访问。

## 6.6 破坏性命令模式检测

### 6.6.1 危险 Bash 模式

`dangerousPatterns.ts` 集中定义了被认为"危险"的命令模式——它们可以执行任意代码，从而绕过分类器的安全评估：

```typescript
export const CROSS_PLATFORM_CODE_EXEC = [
  'python', 'python3', 'python2', 'node', 'deno', 'tsx', 'ruby', 'perl', 'php', 'lua',
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  'bash', 'sh', 'ssh',
] as const

export const DANGEROUS_BASH_PATTERNS: readonly string[] = [
  ...CROSS_PLATFORM_CODE_EXEC,
  'zsh', 'fish', 'eval', 'exec', 'env', 'xargs', 'sudo',
  // 内部用户额外模式：fa run, coo, gh, curl, wget, git, kubectl, aws, ...
]
```

这些模式用于 `isDangerousBashPermission` 函数，在 auto 模式激活时剥离过于宽泛的 allow 规则：

```typescript
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  if (toolName !== BASH_TOOL_NAME) return false
  if (ruleContent === undefined || ruleContent === '') return true  // Bash(*) 允许一切
  if (content === '*') return true

  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === lowerPattern) return true          // 精确匹配
    if (content === `${lowerPattern}:*`) return true    // 前缀语法
    if (content === `${lowerPattern}*`) return true     // 通配符
    if (content === `${lowerPattern} *`) return true    // 空格通配符
    // ...
  }
  return false
}
```

### 6.6.2 PowerShell 危险模式

类似地，PowerShell 有自己的危险模式列表，涵盖了 cmdlet 别名和 .NET 逃逸通道：

```typescript
const patterns: readonly string[] = [
  ...CROSS_PLATFORM_CODE_EXEC,
  'pwsh', 'powershell', 'cmd', 'wsl',
  'iex', 'invoke-expression', 'icm', 'invoke-command',
  'start-process', 'saps', 'start', 'start-job',
  'register-objectevent', 'register-engineevent',
  'add-type',    // Add-Type -TypeDefinition '<C#>' → P/Invoke
  'new-object',  // New-Object -ComObject WScript.Shell → .Run()
]
```

### 6.6.3 Agent 权限保护

任何 Agent allow 规则都被视为危险规则，因为它会在分类器评估子 Agent 的 prompt 之前就自动批准：

```typescript
export function isDangerousTaskPermission(toolName: string, _ruleContent: string | undefined): boolean {
  return normalizeLegacyToolName(toolName) === AGENT_TOOL_NAME
}
```

这体现了一个重要的安全原则：**防止委托攻击** (delegation attack)。恶意 prompt 可能通过让 Agent 产生子 Agent 来绕过分类器。

## 6.7 Auto 模式与 YOLO 分类器

Auto 模式是 Claude Code 权限系统中最前沿的组件。当权限决策为 `ask` 时，auto 模式不是询问用户，而是调用一个专门的 LLM 分类器来评估操作是否安全。

### 6.7.1 分类器架构

`yoloClassifier.ts` 实现了一个 2 阶段的分类流程：

```typescript
// 阶段 1：快速分类 (fast)
// 使用较轻量的模型快速判断，大多数安全操作在此阶段通过

// 阶段 2：思考分类 (thinking)
// 对阶段 1 不确定的结果，使用更强的模型进行深度分析
```

分类器使用 `sideQuery` 机制调用 API，与主对话循环分离：

```typescript
export const YOLO_CLASSIFIER_TOOL_NAME = 'classify_result'

const YOLO_CLASSIFIER_TOOL_SCHEMA: BetaToolUnion = {
  type: 'custom',
  name: YOLO_CLASSIFIER_TOOL_NAME,
  description: 'Report the security classification result for the agent action',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string', description: 'Brief step-by-step reasoning.' },
      shouldBlock: { type: 'boolean', description: 'Whether the action should be blocked (true) or allowed (false)' },
      reason: { type: 'string', description: 'Brief explanation of the classification decision' },
    },
    required: ['thinking', 'shouldBlock', 'reason'],
  },
}
```

分类器的 system prompt 包含三个可定制部分：

```typescript
export type AutoModeRules = {
  allow: string[]       // 允许规则
  soft_deny: string[]   // 软拒绝规则
  environment: string[] // 环境描述
}
```

用户可以在 `settings.json` 的 `autoMode` 字段中自定义这些规则，覆盖默认值。

### 6.7.2 对话上下文构建

分类器需要理解当前的对话上下文才能做出判断。`buildTranscriptEntries` 将对话历史转换为简化的 transcript：

```typescript
export function buildTranscriptEntries(messages: Message[]): TranscriptEntry[] {
  const transcript: TranscriptEntry[] = []
  for (const msg of messages) {
    if (msg.type === 'user') {
      // 提取用户文本
    } else if (msg.type === 'assistant') {
      // 只包含 tool_use blocks，排除 assistant text
      // 排除 text 的原因：模型生成的文本可能被精心构造来影响分类器决策
    }
  }
  return transcript
}
```

注意安全考虑：assistant 的文本内容被有意排除，只保留 tool_use 调用记录。这防止了模型通过生成影响性文本来操纵分类器。

### 6.7.3 熔断机制 (Denial Tracking)

auto 模式包含一个熔断器来防止分类器持续拒绝导致的死循环：

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number
  // ...
}

export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return state.consecutiveDenials >= DENIAL_LIMITS.MAX_CONSECUTIVE
}
```

当分类器连续拒绝超过阈值时，系统会降级回用户提示模式。每次成功的 `allow` 会重置计数器：

```typescript
if (result.behavior === 'allow') {
  if (mode === 'auto' && currentDenialState.consecutiveDenials > 0) {
    const newDenialState = recordSuccess(currentDenialState)
    persistDenialState(context, newDenialState)
  }
}
```

### 6.7.4 分类器结果结构

```typescript
export type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  durationMs?: number
  stage?: 'fast' | 'thinking'         // 哪个阶段做出的决策
  stage1Usage?: ClassifierUsage        // 阶段 1 token 使用
  stage2Usage?: ClassifierUsage        // 阶段 2 token 使用
  stage1RequestId?: string             // 用于审计追踪
  stage2RequestId?: string
  // ...
}
```

`transcriptTooLong` 标志表示对话上下文超出了分类器的 context window。这是确定性的（相同 transcript 总是触发），因此系统降级到正常提示而非重试。

## 6.8 Hook 集成

Hooks 系统允许在工具调用前后执行自定义逻辑，包括权限决策。

### 6.8.1 Hook 事件类型

```typescript
// schemas/hooks.ts 定义了四种 Hook 类型：
const BashCommandHookSchema = z.object({
  type: z.literal('command'),
  command: z.string(),
  if: IfConditionSchema(),   // 权限规则语法过滤
  shell: z.enum(SHELL_TYPES).optional(),
  timeout: z.number().positive().optional(),
  once: z.boolean().optional(),
  async: z.boolean().optional(),
  asyncRewake: z.boolean().optional(),
})

const PromptHookSchema = z.object({
  type: z.literal('prompt'),
  prompt: z.string(),       // 使用 $ARGUMENTS 占位符
  model: z.string().optional(),
})

const HttpHookSchema = z.object({
  type: z.literal('http'),
  url: z.string().url(),
  headers: z.record(z.string(), z.string()).optional(),
  allowedEnvVars: z.array(z.string()).optional(),
})

const AgentHookSchema = z.object({
  type: z.literal('agent'),
  prompt: z.string(),
  model: z.string().optional(),
})
```

### 6.8.2 PreToolUse / PostToolUse

Hook 通过 `PreToolUse` 和 `PostToolUse` 事件点介入权限决策。`PreToolUse` 钩子可以：
- 返回 `allow` 覆盖默认行为
- 返回 `deny` 阻止工具调用
- 不返回，让正常权限流程继续

`if` 条件使用权限规则语法进行过滤，避免为不匹配的命令派生 Hook 进程：

```typescript
const IfConditionSchema = lazySchema(() =>
  z.string().optional().describe(
    'Permission rule syntax to filter when this hook runs (e.g., "Bash(git *)").'
  ),
)
```

### 6.8.3 无头 Agent 中的 Hook

对于无法显示权限提示的 Agent（如异步后台 Agent），Hook 是唯一的权限决策途径：

```typescript
async function runPermissionRequestHooksForHeadlessAgent(
  tool, input, toolUseID, context, permissionMode, suggestions,
): Promise<PermissionDecision | null> {
  for await (const hookResult of executePermissionRequestHooks(...)) {
    if (hookResult.permissionRequestResult) {
      const decision = hookResult.permissionRequestResult
      if (decision.behavior === 'allow') {
        // 持久化权限更新
        if (decision.updatedPermissions?.length) {
          persistPermissionUpdates(decision.updatedPermissions)
        }
        return { behavior: 'allow', decisionReason: { type: 'hook', hookName: 'PermissionRequest' } }
      }
      if (decision.behavior === 'deny') {
        if (decision.interrupt) context.abortController.abort()
        return { behavior: 'deny', ... }
      }
    }
  }
  return null  // 没有 Hook 做出决策，调用方将自动拒绝
}
```

## 6.9 安全设计理念总结

通过对 Claude Code 权限系统的深入分析，我们可以总结出以下设计理念：

**1. 纵深防御 (Defense in Depth)**
从 Bash 命令解析器到规则匹配再到 AI 分类器，每一层都是独立的安全屏障。即使某一层被绕过，后续层仍然提供保护。

**2. 默认拒绝 (Default Deny)**
所有未明确允许的操作默认需要用户确认。`dontAsk` 模式下更是直接拒绝。系统宁可误报（false positive）也不放过潜在风险。

**3. 最小特权 (Least Privilege)**
权限规则支持精细粒度的内容匹配（如 `Bash(git commit:*)`），允许用户精确控制哪些操作被自动化。

**4. 防止权限升级**
- Agent allow 规则被视为危险规则（防止委托攻击）
- Claude 的配置文件始终需要确认（防止自我提权）
- 子 Agent 的权限被严格限制在父级范围内

**5. 大小写规范化与路径标准化**
所有路径比较都经过规范化处理，防止在大小写不敏感文件系统上通过大小写变体绕过安全检查。

**6. 安全注释文化**
源码中大量的 `SECURITY:` 注释记录了每个安全决策的原因和潜在的绕过途径。这不仅帮助代码审查，也为后续维护者提供了重要的安全上下文。

**7. 可观测性**
每个权限决策都附带 `PermissionDecisionReason`，支持审计追踪。分类器结果包含详细的 token 使用、耗时、请求 ID 等遥测数据。

这套权限系统展现了一个核心矛盾的优雅解决方案：如何在保持 AI Agent 强大能力的同时，确保用户对关键操作的控制权。Claude Code 的答案是——不是限制 Agent 能做什么，而是确保每一步都有适当的监督。

