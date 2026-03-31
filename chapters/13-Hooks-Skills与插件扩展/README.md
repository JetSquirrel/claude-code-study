# 第十三章：Hooks、Skills 与插件扩展

## 13.1 引言

一个成熟的 Agent 系统不仅要有强大的核心能力，还需要提供灵活的扩展机制。Claude Code 设计了三层扩展体系：

1. **Hooks**：事件驱动的回调系统，允许在工具调用前后、会话生命周期事件中执行自定义逻辑
2. **Skills**：可复用的 Prompt 模板，封装特定领域的专业知识和工作流
3. **Plugins**：包含多个组件（Skills、Hooks、MCP 服务器）的可管理扩展包

此外，**Output Styles** 和 **Keybindings** 提供了用户界面层面的自定义能力。本章将逐一深入分析这些扩展机制的设计与实现。

## 13.2 Hook 事件系统

### 13.2.1 Hook 事件类型

Hook 事件定义在 `entrypoints/sdk/coreSchemas.ts` 中，覆盖了 Agent 生命周期的几乎所有关键节点：

```typescript
export const HOOK_EVENTS = [
  'PreToolUse',          // 工具调用前
  'PostToolUse',         // 工具调用后（成功）
  'PostToolUseFailure',  // 工具调用后（失败）
  'Notification',        // 通知
  'UserPromptSubmit',    // 用户提交 Prompt
  'SessionStart',        // 会话开始
  'SessionEnd',          // 会话结束
  'Stop',                // Agent 停止
  'StopFailure',         // Agent 停止失败
  'SubagentStart',       // 子 Agent 启动
  'SubagentStop',        // 子 Agent 停止
  'PreCompact',          // 上下文压缩前
  'PostCompact',         // 上下文压缩后
  'PermissionRequest',   // 权限请求
  'PermissionDenied',    // 权限被拒绝
  'Setup',               // 初始化
  'TeammateIdle',        // 团队成员空闲
  'TaskCreated',         // 任务创建
  'TaskCompleted',       // 任务完成
  'Elicitation',         // 信息采集
  'ElicitationResult',   // 信息采集结果
  'ConfigChange',        // 配置变更
  'WorktreeCreate',      // Worktree 创建
  'WorktreeRemove',      // Worktree 移除
  'InstructionsLoaded',  // 指令加载
  'CwdChanged',          // 工作目录变更
  'FileChanged',         // 文件变更
] as const
```

这套事件体系的设计遵循了"观察者模式"的经典范式，但针对 AI Agent 的特殊需求做了扩展。例如 `PreToolUse` / `PostToolUse` 对应了 AOP（面向切面编程）中的 before/after advice，而 `PermissionRequest` / `PermissionDenied` 则是权限系统的集成点。

### 13.2.2 Hook 类型

`schemas/hooks.ts` 定义了四种 Hook 类型，每种对应不同的执行机制：

**Command Hook**（Shell 命令）：

```typescript
const BashCommandHookSchema = z.object({
  type: z.literal('command').describe('Shell command hook type'),
  command: z.string().describe('Shell command to execute'),
  if: IfConditionSchema(),
  shell: z.enum(SHELL_TYPES).optional()
    .describe("Shell interpreter. 'bash' uses your $SHELL; 'powershell' uses pwsh."),
  timeout: z.number().positive().optional()
    .describe('Timeout in seconds for this specific command'),
  statusMessage: z.string().optional()
    .describe('Custom status message to display in spinner while hook runs'),
  once: z.boolean().optional()
    .describe('If true, hook runs once and is removed after execution'),
  async: z.boolean().optional()
    .describe('If true, hook runs in background without blocking'),
  asyncRewake: z.boolean().optional()
    .describe('If true, hook runs in background and wakes the model on exit code 2'),
})
```

**Prompt Hook**（LLM Prompt）：

```typescript
const PromptHookSchema = z.object({
  type: z.literal('prompt').describe('LLM prompt hook type'),
  prompt: z.string()
    .describe('Prompt to evaluate with LLM. Use $ARGUMENTS placeholder for hook input JSON.'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  model: z.string().optional()
    .describe('Model to use (e.g., "claude-sonnet-4-6"). Defaults to small fast model.'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

**HTTP Hook**（HTTP 请求）：

```typescript
const HttpHookSchema = z.object({
  type: z.literal('http').describe('HTTP hook type'),
  url: z.string().url().describe('URL to POST the hook input JSON to'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  headers: z.record(z.string(), z.string()).optional()
    .describe('Additional headers. Values may reference env vars using $VAR_NAME syntax.'),
  allowedEnvVars: z.array(z.string()).optional()
    .describe('Explicit list of env var names that may be interpolated in headers.'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

**Agent Hook**（Agent 验证器）：

```typescript
const AgentHookSchema = z.object({
  type: z.literal('agent').describe('Agentic verifier hook type'),
  prompt: z.string()
    .describe('Prompt describing what to verify (e.g., "Verify that unit tests ran and passed.")'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional()
    .describe('Timeout in seconds for agent execution (default 60)'),
  model: z.string().optional()
    .describe('Model to use. If not specified, uses Haiku.'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

### 13.2.3 条件过滤（If Condition）

```typescript
const IfConditionSchema = lazySchema(() =>
  z.string().optional().describe(
    'Permission rule syntax to filter when this hook runs (e.g., "Bash(git *)"). ' +
    'Only runs if the tool call matches the pattern. Avoids spawning hooks for non-matching commands.',
  ),
)
```

`if` 字段复用了权限系统的规则语法（如 `Bash(git *)`、`Read(*.ts)`），允许 Hook 只在匹配特定工具调用模式时触发。这避免了为每个不相关的工具调用都产生 Hook 进程。

### 13.2.4 Hook 配置结构

```typescript
// 匹配器配置
export const HookMatcherSchema = lazySchema(() =>
  z.object({
    matcher: z.string().optional()
      .describe('String pattern to match (e.g. tool names like "Write")'),
    hooks: z.array(HookCommandSchema())
      .describe('List of hooks to execute when the matcher matches'),
  }),
)

// 完整的 Hooks 配置
export const HooksSchema = lazySchema(() =>
  z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema())),
)
```

配置格式示例（在 `settings.json` 中）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to write a file'",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Bash command completed'",
            "if": "Bash(npm test *)"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Session started at $(date)'"
          }
        ]
      }
    ]
  }
}
```

### 13.2.5 Hook 的高级特性

**一次性执行**：`once: true` 使 Hook 在首次执行后自动注销，适用于初始化逻辑。

**异步执行**：`async: true` 使 Hook 在后台运行，不阻塞 Agent 主流程。这对于日志记录、通知推送等非关键操作很有用。

**异步唤醒**：`asyncRewake: true` 是异步执行的增强版 —— Hook 在后台运行，但如果返回 exit code 2（阻塞错误），会唤醒模型处理问题。这对于异步验证（如 CI 检查）很有价值。

**环境变量安全**：HTTP Hook 的 `headers` 支持 `$VAR_NAME` 语法引用环境变量，但必须通过 `allowedEnvVars` 显式声明哪些变量可以被引用。未声明的变量会被解析为空字符串，防止意外泄露敏感环境变量。

### 13.2.6 Discriminated Union 模式

四种 Hook 类型通过 Zod 的 `discriminatedUnion` 组合：

```typescript
export const HookCommandSchema = lazySchema(() => {
  const { BashCommandHookSchema, PromptHookSchema, AgentHookSchema, HttpHookSchema } = buildHookSchemas()
  return z.discriminatedUnion('type', [
    BashCommandHookSchema,
    PromptHookSchema,
    AgentHookSchema,
    HttpHookSchema,
  ])
})
```

`discriminatedUnion` 比普通 `union` 更高效：Zod 只需检查 `type` 字段就能确定使用哪个 schema 进行验证，而不需要尝试所有候选 schema。

### 13.2.7 与权限系统的集成

System Prompt 中对 Hooks 的说明将其视为用户的延伸：

```typescript
function getHooksSection(): string {
  return `Users may configure 'hooks', shell commands that execute in response to events
like tool calls, in settings. Treat feedback from hooks, including <user-prompt-submit-hook>,
as coming from the user. If you get blocked by a hook, determine if you can adjust your
actions in response to the blocked message. If not, ask the user to check their hooks
configuration.`
}
```

这是一个关键的信任模型决策：Hook 的输出被视为用户输入，模型应该尊重 Hook 的反馈并调整行为。

## 13.3 Skill 系统

### 13.3.1 BundledSkillDefinition 类型

内置技能（Bundled Skills）的定义在 `skills/bundledSkills.ts` 中：

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

字段解读：

- `whenToUse`：模型判断何时自动触发此技能的线索
- `allowedTools`：技能执行时可用的工具白名单
- `model`：技能使用的模型（可能使用比主模型更轻量的模型）
- `disableModelInvocation`：如果为 true，模型不能自动调用此技能（仅用户可通过 `/` 命令触发）
- `context`：`inline` 在当前上下文中执行，`fork` 创建新的上下文分支
- `files`：附带的参考文件，首次调用时提取到磁盘
- `getPromptForCommand`：核心方法，生成技能的 Prompt 内容

### 13.3.2 技能注册

```typescript
const bundledSkills: Command[] = []

export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition
  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  // 如果技能包含参考文件，需要惰性提取
  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand

    getPromptForCommand = async (args, ctx) => {
      // 闭包内 Memoization：每进程只提取一次
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }

  const command: Command = {
    type: 'prompt',
    name: definition.name,
    description: definition.description,
    // ... 其他字段映射
    source: 'bundled',
    loadedFrom: 'bundled',
    getPromptForCommand,
  }
  bundledSkills.push(command)
}
```

**惰性文件提取**：技能的参考文件不在启动时提取，而是在首次调用时才写入磁盘。通过 `extractionPromise ??=` 模式确保并发调用只触发一次提取。

### 13.3.3 参考文件的安全写入

```typescript
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'  // Windows 使用字符串标志
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW

async function safeWriteFile(p: string, content: string): Promise<void> {
  const fh = await open(p, SAFE_WRITE_FLAGS, 0o600)
  try {
    await fh.writeFile(content, 'utf8')
  } finally {
    await fh.close()
  }
}
```

安全措施：
- `O_EXCL`：文件已存在时失败（不覆盖）
- `O_NOFOLLOW`：不跟随符号链接
- `0o600`：只有所有者可读写
- 目录使用 `0o700` 权限

注释解释了威胁模型：

> The per-process nonce in getBundledSkillsRoot() is the primary defense against pre-created symlinks/dirs. Explicit 0o700/0o600 modes keep the nonce subtree owner-only even on umask=0.

### 13.3.4 路径遍历防护

```typescript
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (
    isAbsolute(normalized) ||
    normalized.split(pathSep).includes('..') ||
    normalized.split('/').includes('..')
  ) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

防止技能的文件路径通过 `..` 或绝对路径逃逸出技能目录。

### 13.3.5 用户自定义技能的加载

`loadSkillsDir.ts` 负责从文件系统加载用户定义的技能。技能文件是 Markdown 格式，存放在多个位置：

```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string {
  switch (source) {
    case 'policySettings':
      return join(getManagedFilePath(), '.claude', dir)     // 策略管理路径
    case 'userSettings':
      return join(getClaudeConfigHomeDir(), dir)            // ~/.claude/skills/
    case 'projectSettings':
      return `.claude/${dir}`                                // .claude/skills/
    case 'plugin':
      return 'plugin'                                        // 插件提供
    default:
      return ''
  }
}
```

技能加载会遍历 project 目录层级（从当前目录到 home 目录），在每层查找 `.claude/skills/` 和 `.claude/commands/`（向后兼容）目录。

### 13.3.6 技能 Frontmatter 解析

```typescript
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
  descriptionFallbackLabel: 'Skill' | 'Custom command' = 'Skill',
): {
  displayName: string | undefined
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  argumentHint: string | undefined
  argumentNames: string[]
  whenToUse: string | undefined
  version: string | undefined
  model: ReturnType<typeof parseUserSpecifiedModel> | undefined
  disableModelInvocation: boolean
  // ...
}
```

技能的 Markdown 文件示例：

```markdown
---
name: commit
description: Create a well-crafted git commit
when-to-use: When the user asks to commit changes
allowed-tools: Bash, Read, Grep
argument-hint: <commit message>
---

Review the staged changes and create a commit...
```

### 13.3.7 技能的 Token 开销估算

```typescript
export function estimateSkillFrontmatterTokens(skill: Command): number {
  const frontmatterText = [skill.name, skill.description, skill.whenToUse]
    .filter(Boolean)
    .join(' ')
  return roughTokenCountEstimation(frontmatterText)
}
```

这用于 System Prompt 的 Token 预算管理：每个技能的 frontmatter 会占用一部分上下文空间（用于在 System Prompt 中列出可用技能），所以需要估算其 Token 成本。

### 13.3.8 技能中的 Hooks

技能可以内嵌 Hooks 配置：

```typescript
function parseHooksFromFrontmatter(
  frontmatter: FrontmatterData,
  skillName: string,
): HooksSettings | undefined {
  if (!frontmatter.hooks) return undefined

  const result = HooksSchema().safeParse(frontmatter.hooks)
  if (!result.success) {
    logForDebugging(`Invalid hooks in skill '${skillName}': ${result.error.message}`)
    return undefined
  }
  return result.data
}
```

这意味着一个技能可以注册自己的 Hook —— 例如，一个"deploy"技能可以在 `PostToolUse(Bash)` 时检查部署状态。

### 13.3.9 技能的路径作用域

```typescript
function parseSkillPaths(frontmatter: FrontmatterData): string[] | undefined {
  if (!frontmatter.paths) return undefined

  const patterns = splitPathInFrontmatter(frontmatter.paths)
    .map(pattern => {
      return pattern.endsWith('/**') ? pattern.slice(0, -3) : pattern
    })
    .filter((p: string) => p.length > 0)

  if (patterns.length === 0 || patterns.every((p: string) => p === '**')) {
    return undefined
  }
  return patterns
}
```

技能可以通过 `paths` 字段限制其在特定文件路径范围内才激活，复用了与 CLAUDE.md 规则相同的 glob 语法。

### 13.3.10 技能的去重

```typescript
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)  // 解析符号链接
  } catch {
    return null
  }
}
```

使用 `realpath` 解析符号链接来检测重复文件。这处理了通过不同路径（如符号链接或重叠的父目录）访问同一文件的情况。注释中提到了一些文件系统的 inode 不可靠（NFS、ExFAT），所以选择了 `realpath` 而非 `stat` + inode 比较。

## 13.4 插件系统

### 13.4.1 BuiltinPluginDefinition 类型

插件在 `plugins/builtinPlugins.ts` 中管理：

```typescript
// 从 types/plugin.ts 导入的类型
export type BuiltinPluginDefinition = {
  name: string
  description: string
  version: string
  defaultEnabled?: boolean
  isAvailable?: () => boolean
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, unknown>
}
```

### 13.4.2 插件 vs 技能的区别

插件和技能是不同层次的抽象：

| 维度 | 技能（Skill） | 插件（Plugin） |
|------|-------------|-------------|
| 粒度 | 单个 Prompt 模板 | 多个组件的集合 |
| 组件 | 仅 Prompt | Skills + Hooks + MCP Servers |
| 管理 | 文件系统自动发现 | `/plugin` UI 显式管理 |
| 开关 | 无（始终可用） | 用户可启用/禁用 |
| ID 格式 | 文件名 | `{name}@builtin` 或 `{name}@{marketplace}` |

### 13.4.3 插件注册与发现

```typescript
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()

export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}
```

### 13.4.4 插件启用状态管理

```typescript
export function getBuiltinPlugins(): {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
} {
  const settings = getSettings_DEPRECATED()
  const enabled: LoadedPlugin[] = []
  const disabled: LoadedPlugin[] = []

  for (const [name, definition] of BUILTIN_PLUGINS) {
    // 可用性检查
    if (definition.isAvailable && !definition.isAvailable()) continue

    const pluginId = `${name}@${BUILTIN_MARKETPLACE_NAME}`
    const userSetting = settings?.enabledPlugins?.[pluginId]
    // 启用优先级：用户设置 > 插件默认 > true
    const isEnabled =
      userSetting !== undefined
        ? userSetting === true
        : (definition.defaultEnabled ?? true)

    const plugin: LoadedPlugin = {
      name,
      manifest: { name, description: definition.description, version: definition.version },
      path: BUILTIN_MARKETPLACE_NAME,
      source: pluginId,
      repository: pluginId,
      enabled: isEnabled,
      isBuiltin: true,
      hooksConfig: definition.hooks,
      mcpServers: definition.mcpServers,
    }

    if (isEnabled) enabled.push(plugin)
    else disabled.push(plugin)
  }

  return { enabled, disabled }
}
```

插件的 `isAvailable()` 方法允许插件在运行时决定是否可用（例如，某些插件可能依赖特定平台或特定配置）。

### 13.4.5 从插件获取技能

```typescript
export function getBuiltinPluginSkillCommands(): Command[] {
  const { enabled } = getBuiltinPlugins()
  const commands: Command[] = []

  for (const plugin of enabled) {
    const definition = BUILTIN_PLUGINS.get(plugin.name)
    if (!definition?.skills) continue
    for (const skill of definition.skills) {
      commands.push(skillDefinitionToCommand(skill))
    }
  }

  return commands
}
```

注意 `skillDefinitionToCommand` 将 BundledSkillDefinition 转换为 Command 时，使用的 source 是 `'bundled'` 而非 `'builtin'`：

```typescript
function skillDefinitionToCommand(definition: BundledSkillDefinition): Command {
  return {
    // ...
    // 'bundled' not 'builtin' — 'builtin' in Command.source means hardcoded
    // slash commands (/help, /clear). Using 'bundled' keeps these skills in
    // the Skill tool's listing, analytics name logging, and prompt-truncation
    // exemption.
    source: 'bundled',
    loadedFrom: 'bundled',
  }
}
```

这是一个命名约定的微妙之处：`Command.source === 'builtin'` 指的是硬编码的斜杠命令（`/help`、`/clear`），而不是内置插件提供的技能。

## 13.5 输出样式（Output Styles）

### 13.5.1 样式加载

`outputStyles/loadOutputStylesDir.ts` 从两个位置加载自定义输出样式：

```typescript
export const getOutputStyleDirStyles = memoize(
  async (cwd: string): Promise<OutputStyleConfig[]> => {
    const markdownFiles = await loadMarkdownFilesForSubdir('output-styles', cwd)

    return markdownFiles.map(({ filePath, frontmatter, content, source }) => {
      const fileName = basename(filePath)
      const styleName = fileName.replace(/\.md$/, '')

      const name = (frontmatter['name'] || styleName) as string
      const description = coerceDescriptionToString(frontmatter['description'], styleName)
        ?? extractDescriptionFromMarkdown(content, `Custom ${styleName} output style`)

      // keep-coding-instructions 标志
      const keepCodingInstructionsRaw = frontmatter['keep-coding-instructions']
      const keepCodingInstructions = keepCodingInstructionsRaw === true ||
        keepCodingInstructionsRaw === 'true'
        ? true
        : keepCodingInstructionsRaw === false || keepCodingInstructionsRaw === 'false'
          ? false
          : undefined

      return {
        name,
        description,
        prompt: content.trim(),
        source,
        keepCodingInstructions,
      }
    }).filter(style => style !== null)
  },
)
```

### 13.5.2 样式与编码指令的关系

`keepCodingInstructions` 是一个关键配置项。当用户启用自定义输出样式时，默认会替换 "Doing Tasks" 段落中的编码指令。但如果 `keepCodingInstructions: true`，两者会共存：

```typescript
// prompts.ts 中的相关逻辑
outputStyleConfig === null ||
outputStyleConfig.keepCodingInstructions === true
  ? getSimpleDoingTasksSection()
  : null,
```

### 13.5.3 样式在 System Prompt 中的注入

```typescript
function getOutputStyleSection(outputStyleConfig: OutputStyleConfig | null): string | null {
  if (outputStyleConfig === null) return null
  return `# Output Style: ${outputStyleConfig.name}
${outputStyleConfig.prompt}`
}
```

### 13.5.4 缓存管理

```typescript
export function clearOutputStyleCaches(): void {
  getOutputStyleDirStyles.cache?.clear?.()
  loadMarkdownFilesForSubdir.cache?.clear?.()
  clearPluginOutputStyleCache()
}
```

三层缓存（样式目录、Markdown 文件、插件样式）在需要时一起清除。

## 13.6 键绑定系统（Keybindings）

### 13.6.1 默认绑定

`keybindings/defaultBindings.ts` 定义了所有默认键绑定，按上下文（Context）组织：

```typescript
export const DEFAULT_BINDINGS: KeybindingBlock[] = [
  {
    context: 'Global',
    bindings: {
      'ctrl+c': 'app:interrupt',
      'ctrl+d': 'app:exit',
      'ctrl+l': 'app:redraw',
      'ctrl+t': 'app:toggleTodos',
      'ctrl+o': 'app:toggleTranscript',
      'ctrl+r': 'history:search',
    },
  },
  {
    context: 'Chat',
    bindings: {
      escape: 'chat:cancel',
      'ctrl+x ctrl+k': 'chat:killAgents',    // Chord（组合键序列）
      [MODE_CYCLE_KEY]: 'chat:cycleMode',     // 平台自适应
      'meta+p': 'chat:modelPicker',
      enter: 'chat:submit',
      up: 'history:previous',
      down: 'history:next',
      'ctrl+_': 'chat:undo',                 // 传统终端
      'ctrl+shift+-': 'chat:undo',           // Kitty 协议
      'ctrl+x ctrl+e': 'chat:externalEditor',
      'ctrl+g': 'chat:externalEditor',
    },
  },
  {
    context: 'Autocomplete',
    bindings: {
      tab: 'autocomplete:accept',
      escape: 'autocomplete:dismiss',
      up: 'autocomplete:previous',
      down: 'autocomplete:next',
    },
  },
  // ... 更多上下文
]
```

### 13.6.2 平台适配

```typescript
// 图片粘贴：Windows 用 alt+v（ctrl+v 被系统占用），其他平台用 ctrl+v
const IMAGE_PASTE_KEY = getPlatform() === 'windows' ? 'alt+v' : 'ctrl+v'

// 终端 VT 模式检测
const SUPPORTS_TERMINAL_VT_MODE =
  getPlatform() !== 'windows' ||
  (isRunningWithBun()
    ? satisfies(process.versions.bun, '>=1.2.23')
    : satisfies(process.versions.node, '>=22.17.0 <23.0.0 || >=24.2.0'))

// 模式切换：无 VT 模式支持时用 meta+m 替代 shift+tab
const MODE_CYCLE_KEY = SUPPORTS_TERMINAL_VT_MODE ? 'shift+tab' : 'meta+m'
```

这段代码揭示了终端兼容性的复杂现实：Windows Terminal 在没有 VT 模式时无法可靠处理 `shift+tab`，需要特定版本的 Node.js 或 Bun 来启用 VT 模式支持。

### 13.6.3 上下文分层

键绑定系统支持 15+ 个上下文层，每个上下文有独立的键绑定空间：

- **Global**：全局快捷键（`ctrl+c`、`ctrl+d` 等）
- **Chat**：聊天输入时
- **Autocomplete**：自动完成菜单
- **Settings**：设置面板
- **Confirmation**：确认对话框
- **Transcript**：Transcript 查看器
- **HistorySearch**：历史搜索（`ctrl+r`）
- **Task**：任务执行中
- **Scroll**：滚动操作
- **Select**：选择列表
- **MessageSelector**：消息选择器（rewind 对话框）
- **MessageActions**：消息操作菜单
- **DiffDialog**：差异查看器
- **Footer**：底部指示器导航
- **Attachments**：附件导航

### 13.6.4 特殊键处理

某些键有特殊处理逻辑：

```typescript
// ctrl+c 和 ctrl+d 使用基于时间的双击检测
// 它们在这里定义以便 resolver 能找到它们，但用户无法重新绑定
'ctrl+c': 'app:interrupt',
'ctrl+d': 'app:exit',
```

```typescript
// Undo 有两个绑定来支持不同终端行为：
// - ctrl+_ 用于传统终端（发送 \x1f 控制字符）
// - ctrl+shift+- 用于 Kitty 协议终端（发送带修饰符的物理键）
'ctrl+_': 'chat:undo',
'ctrl+shift+-': 'chat:undo',
```

### 13.6.5 Chord 键（组合键序列）

```typescript
// ctrl+x 前缀避免遮蔽 readline 编辑键（ctrl+a/b/e/f 等）
'ctrl+x ctrl+k': 'chat:killAgents',
'ctrl+x ctrl+e': 'chat:externalEditor',
```

Chord 键是两步组合键：先按 `ctrl+x`，再按第二个键。选择 `ctrl+x` 作为前缀是因为 Emacs readline 使用相同的约定，避免与单键快捷键冲突。

### 13.6.6 Feature-gated 绑定

```typescript
...(feature('KAIROS') || feature('KAIROS_BRIEF')
  ? { 'ctrl+shift+b': 'app:toggleBrief' as const }
  : {}),
...(feature('QUICK_SEARCH')
  ? {
      'ctrl+shift+f': 'app:globalSearch' as const,
      'cmd+shift+f': 'app:globalSearch' as const,
    }
  : {}),
...(feature('VOICE_MODE') ? { space: 'voice:pushToTalk' } : {}),
```

通过 feature flag 控制键绑定的激活，确保未发布的功能不会意外暴露快捷键。

## 13.7 扩展点总结

Claude Code 的扩展体系形成了一个清晰的层次结构：

```
                  ┌─────────────────┐
                  │    Plugin       │  最高抽象层：组合多个组件
                  │ (Skills+Hooks+  │
                  │  MCP Servers)   │
                  └────────┬────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
    ┌──────────────┐ ┌──────────┐ ┌──────────────┐
    │   Skills     │ │  Hooks   │ │  MCP Servers  │  中间层：独立的扩展机制
    │ (Prompts)    │ │ (Events) │ │ (Protocol)    │
    └──────┬───────┘ └────┬─────┘ └──────┬───────┘
           │              │              │
           ▼              ▼              ▼
    ┌──────────────┐ ┌──────────┐ ┌──────────────┐
    │ Output Style │ │Keybinding│ │  Settings    │  基础层：用户配置
    └──────────────┘ └──────────┘ └──────────────┘
```

每层都有自己的配置格式、加载机制和安全模型：

1. **Hooks**：通过 `settings.json` 配置，支持四种执行类型，与权限系统集成
2. **Skills**：通过文件系统发现（Markdown），支持 frontmatter 元数据和模板参数
3. **Plugins**：通过 UI 管理，聚合多个扩展组件，支持启用/禁用
4. **Output Styles**：通过 Markdown 文件定义输出风格
5. **Keybindings**：通过 JSON 配置覆盖默认绑定，支持平台适配和上下文分层

这套扩展体系遵循了"约定优于配置"的原则：默认行为覆盖大多数场景，自定义从简单的文件/配置修改开始，直到需要时才引入更复杂的插件机制。

