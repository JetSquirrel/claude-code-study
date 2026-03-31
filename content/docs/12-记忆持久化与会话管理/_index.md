---
title: "第十二章：记忆、持久化与会话管理"
weight: 120
bookCollapseSection: false
---


## 12.1 引言

AI Agent 的一大局限在于"短期记忆"：每次对话结束后，Agent 对用户的了解、对项目的理解、以及从错误中学到的教训都会消失。Claude Code 通过一套完整的记忆系统（Memory System）解决了这个问题。这套系统不是简单的键值存储，而是一个带有类型分类、索引管理、相关性检索、老化机制和团队协作能力的持久化知识库。

同时，Claude Code 还实现了会话持久化（Session Persistence）、历史记录管理（History Management）和成本追踪（Cost Tracking），确保用户的工作不会因为意外中断而丢失。

## 12.2 记忆系统架构

### 12.2.1 总体设计

Claude Code 的记忆系统基于文件系统，核心思想是：**记忆就是文件**。每条记忆是一个 Markdown 文件，带有 YAML frontmatter 元数据。所有记忆文件存储在一个专用目录中，通过 `MEMORY.md` 索引文件组织。

核心常量定义在 `memdir/memdir.ts` 中：

```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000  // ~125 chars/line at 200 lines
```

`MEMORY.md` 是一个索引文件，每行是一个指向具体记忆文件的链接。200 行和 25KB 的双重限制确保索引不会无限膨胀。

### 12.2.2 记忆目录结构

目录路径的解析遵循明确的优先级链：

```typescript
export const getAutoMemPath = memoize((): string => {
  // 1. 环境变量覆盖（Cowork 使用）
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) {
    return override
  }
  // 2. 默认路径：~/.claude/projects/<sanitized-git-root>/memory/
  const projectsDir = join(getMemoryBaseDir(), 'projects')
  return (
    join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
  ).normalize('NFC')
}, () => getProjectRoot())
```

路径解析的安全性验证极为严格：

```typescript
function validateMemoryPath(raw: string | undefined, expandTilde: boolean): string | undefined {
  // 安全检查清单：
  // - 拒绝相对路径
  // - 拒绝根路径或近根路径（长度 < 3）
  // - 拒绝 Windows 驱动器根路径
  // - 拒绝 UNC 路径（网络路径）
  // - 拒绝包含 null byte 的路径
  if (
    !isAbsolute(normalized) ||
    normalized.length < 3 ||
    /^[A-Za-z]:$/.test(normalized) ||
    normalized.startsWith('\\\\') ||
    normalized.startsWith('//') ||
    normalized.includes('\0')
  ) {
    return undefined
  }
  return (normalized + sep).normalize('NFC')
}
```

**安全设计亮点**：`projectSettings`（项目目录下的 `.claude/settings.json`）被故意排除在记忆路径覆盖之外。注释解释了原因：

> A malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories via the filesystem.ts write carve-out.

这防止了恶意仓库通过记忆路径劫持来获取对敏感目录的写权限。

### 12.2.3 记忆启用条件

```typescript
export function isAutoMemoryEnabled(): boolean {
  const envVal = process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY
  if (isEnvTruthy(envVal)) return false
  if (isEnvDefinedFalsy(envVal)) return true

  // --bare 模式禁用
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false

  // CCR 无持久存储时禁用
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
      !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) return false

  // settings.json 中的配置
  const settings = getInitialSettings()
  if (settings.autoMemoryEnabled !== undefined) return settings.autoMemoryEnabled

  return true  // 默认启用
}
```

优先级链：`CLAUDE_CODE_DISABLE_AUTO_MEMORY` -> `CLAUDE_CODE_SIMPLE` -> `CLAUDE_CODE_REMOTE` -> `settings.json` -> 默认启用。

### 12.2.4 Worktree 共享

一个重要的设计决策是：同一 git 仓库的所有 worktree 共享同一个记忆目录：

```typescript
function getAutoMemBase(): string {
  // 使用 canonical git root 而非 project root
  return findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
}
```

`findCanonicalGitRoot` 会解析 worktree 的原始仓库根目录，确保从 `main` 和 `feature-branch` worktree 启动的 Claude Code 实例共享相同的项目记忆。

## 12.3 记忆类型系统

### 12.3.1 四类型分类法

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
export type MemoryType = (typeof MEMORY_TYPES)[number]
```

每种类型有明确的语义定义：

| 类型 | 描述 | 示例 |
|------|------|------|
| `user` | 用户的角色、目标、偏好 | "用户是数据科学家，正在研究日志系统" |
| `feedback` | 用户对工作方式的指导 | "不要在测试中 mock 数据库" |
| `project` | 项目的非代码上下文 | "周四后冻结非关键合并" |
| `reference` | 外部系统的指针 | "Pipeline bugs 在 Linear INGEST 项目中追踪" |

### 12.3.2 记忆 Frontmatter 格式

```typescript
export const MEMORY_FRONTMATTER_EXAMPLE: readonly string[] = [
  '```markdown',
  '---',
  'name: {{memory name}}',
  'description: {{one-line description — used to decide relevance...}}',
  `type: {{${MEMORY_TYPES.join(', ')}}}`,
  '---',
  '',
  '{{memory content — for feedback/project types, structure as:',
  'rule/fact, then **Why:** and **How to apply:** lines}}',
  '```',
]
```

`description` 字段特别重要 —— 它用于后续的相关性检索。模型被指示写出具体的、有助于判断相关性的描述。

### 12.3.3 什么不应该存储

```typescript
export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '',
  '- Code patterns, conventions, architecture... — these can be derived...',
  '- Git history, recent changes... — `git log` / `git blame` are authoritative.',
  '- Debugging solutions or fix recipes — the fix is in the code...',
  '- Anything already documented in CLAUDE.md files.',
  '- Ephemeral task details: in-progress work, temporary state...',
  '',
  // 即使用户显式要求保存也适用
  'These exclusions apply even when the user explicitly asks you to save.',
]
```

这是一个关键的设计原则：**记忆只存储不可从当前项目状态推导的信息**。代码模式可以通过 grep 发现，架构可以通过代码阅读理解，git 历史是权威来源。记忆系统存储的是"为什么"而不是"是什么"。

### 12.3.4 记忆的信任与验证

```typescript
export const TRUSTING_RECALL_SECTION: readonly string[] = [
  '## Before recommending from memory',
  '',
  'A memory that names a specific function, file, or flag is a claim that it existed',
  '*when the memory was written*. It may have been renamed, removed, or never merged.',
  'Before recommending it:',
  '',
  '- If the memory names a file path: check the file exists.',
  '- If the memory names a function or flag: grep for it.',
  '- If the user is about to act on your recommendation: verify first.',
  '',
  '"The memory says X exists" is not the same as "X exists now."',
]
```

这段指令经过 eval 验证（注释中提到了 `memory-prompt-iteration case 3, 0/2 -> 3/3`），解决了一个实际问题：模型会过度信任记忆中的代码引用，即使代码已经改变。

### 12.3.5 Feedback 类型的双向记录

`feedback` 类型的记忆有一个重要的设计决策：**不仅记录纠正，也记录确认**。

```typescript
// 指令中的关键段落
'Record from failure AND success: if you only save corrections, you will avoid
past mistakes but drift away from approaches the user has already validated,
and may grow overly cautious.'
```

如果只记录纠正（"别这样做"），模型会变得越来越保守，因为它只知道什么不该做，而不知道什么方法是有效的。记录确认（"是的，正是这样"）可以防止这种漂移。

## 12.4 记忆的构建与加载

### 12.4.1 MEMORY.md 索引的截断

```typescript
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const trimmed = raw.trim()
  const contentLines = trimmed.split('\n')
  const lineCount = contentLines.length
  const byteCount = trimmed.length

  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES

  if (!wasLineTruncated && !wasByteTruncated) {
    return { content: trimmed, lineCount, byteCount, wasLineTruncated, wasByteTruncated }
  }

  // 先按行截断，再按字节截断（在换行符处切割，不会截断到行中间）
  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed

  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }

  return {
    content: truncated + `\n\n> WARNING: ${ENTRYPOINT_NAME} is ${reason}.
Only part of it was loaded. Keep index entries to one line under ~200 chars;
move detail into topic files.`,
    ...
  }
}
```

截断后会附加警告消息，指导模型保持索引简洁。字节限制（25KB）是为了捕获"少于200行但每行很长"的情况（在 p100 观测到 197KB 的索引文件仅有 200 行以内）。

### 12.4.2 记忆提示词构建

`buildMemoryLines()` 构建完整的记忆系统提示词，不包含 MEMORY.md 内容：

```typescript
export function buildMemoryLines(
  displayName: string,
  memoryDir: string,
  extraGuidelines?: string[],
  skipIndex = false,
): string[] {
  const lines: string[] = [
    `# ${displayName}`,
    '',
    `You have a persistent, file-based memory system at \`${memoryDir}\`.
    ${DIR_EXISTS_GUIDANCE}`,
    '',
    "You should build up this memory system over time...",
    '',
    ...TYPES_SECTION_INDIVIDUAL,
    ...WHAT_NOT_TO_SAVE_SECTION,
    ...howToSave,
    ...WHEN_TO_ACCESS_SECTION,
    ...TRUSTING_RECALL_SECTION,
    // 记忆与其他持久化机制的关系
    '## Memory and other forms of persistence',
    '- When to use or update a plan instead of memory...',
    '- When to use or update tasks instead of memory...',
  ]

  lines.push(...buildSearchingPastContextSection(memoryDir))
  return lines
}
```

**`DIR_EXISTS_GUIDANCE`** 的设计很有意思：

```typescript
export const DIR_EXISTS_GUIDANCE =
  'This directory already exists — write to it directly with the Write tool
  (do not run mkdir or check for its existence).'
```

这是因为观察到模型在写入记忆前会浪费 turns 执行 `ls` 和 `mkdir -p`。通过在提示词中明确告知目录已存在，消除了这种低效行为。系统通过 `ensureMemoryDirExists()` 在加载阶段预先创建目录来兑现这个承诺。

### 12.4.3 记忆加载入口

`loadMemoryPrompt()` 是记忆系统的主入口，根据不同模式分发：

```typescript
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()

  // KAIROS 日志模式优先
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  // 团队记忆模式
  if (feature('TEAMMEM') && teamMemPaths!.isTeamMemoryEnabled()) {
    const autoDir = getAutoMemPath()
    const teamDir = teamMemPaths!.getTeamMemPath()
    await ensureMemoryDirExists(teamDir)
    return teamMemPrompts!.buildCombinedMemoryPrompt(extraGuidelines, skipIndex)
  }

  // 标准自动记忆模式
  if (autoEnabled) {
    const autoDir = getAutoMemPath()
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines('auto memory', autoDir, extraGuidelines, skipIndex).join('\n')
  }

  // 记忆被禁用
  return null
}
```

### 12.4.4 KAIROS 日志模式

对于长时间运行的助手会话（KAIROS 模式），记忆系统采用不同的策略 —— 追加日志而非维护索引：

```typescript
function buildAssistantDailyLogPrompt(skipIndex = false): string {
  const logPathPattern = join(memoryDir, 'logs', 'YYYY', 'MM', 'YYYY-MM-DD.md')

  const lines: string[] = [
    '# auto memory',
    '',
    `You have a persistent, file-based memory system found at: \`${memoryDir}\``,
    '',
    "This session is long-lived. As you work, record anything worth remembering by
    **appending** to today's daily log file:",
    '',
    `\`${logPathPattern}\``,
    '',
    "...Do not rewrite or reorganize the log — it is append-only.
    A separate nightly process distills these logs into MEMORY.md and topic files.",
  ]
  return lines.join('\n')
}
```

日志路径是模式（`YYYY/MM/YYYY-MM-DD.md`）而非当天的具体路径。注释解释了原因：

> This prompt is cached by systemPromptSection('memory', ...) and NOT invalidated on date change. The model derives the current date from the date_change attachment.

如果硬编码今天的日期，跨午夜时会导致缓存失效。使用模式让模型自己根据 `currentDate` 上下文推导当天日期。

## 12.5 相关记忆发现

### 12.5.1 记忆扫描

`memoryScan.ts` 负责扫描记忆目录并提取 frontmatter：

```typescript
export type MemoryHeader = {
  filename: string
  filePath: string
  mtimeMs: number
  description: string | null
  type: MemoryType | undefined
}

const MAX_MEMORY_FILES = 200
const FRONTMATTER_MAX_LINES = 30

export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  const entries = await readdir(memoryDir, { recursive: true })
  const mdFiles = entries.filter(
    f => f.endsWith('.md') && basename(f) !== 'MEMORY.md'
  )

  const headerResults = await Promise.allSettled(
    mdFiles.map(async (relativePath): Promise<MemoryHeader> => {
      const { content, mtimeMs } = await readFileInRange(
        filePath, 0, FRONTMATTER_MAX_LINES, undefined, signal
      )
      const { frontmatter } = parseFrontmatter(content, filePath)
      return {
        filename: relativePath,
        filePath,
        mtimeMs,
        description: frontmatter.description || null,
        type: parseMemoryType(frontmatter.type),
      }
    }),
  )

  return headerResults
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value)
    .sort((a, b) => b.mtimeMs - a.mtimeMs)  // 最新优先
    .slice(0, MAX_MEMORY_FILES)
}
```

**性能优化**：只读取每个文件的前 30 行（frontmatter 通常在文件开头），通过 `readFileInRange` 同时获取文件内容和 `mtimeMs`，避免了单独的 `stat` 调用。按修改时间排序后截取前 200 个，既保证了覆盖面又控制了 I/O 成本。

### 12.5.2 相关性选择

`findRelevantMemories.ts` 使用 Sonnet 模型来选择与当前查询相关的记忆：

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful
to Claude Code as it processes a user's query. You will be given the user's query
and a list of available memory files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful (up to 5).
Only include memories that you are certain will be helpful...
- If a list of recently-used tools is provided, do not select memories that are usage
  reference or API documentation for those tools (Claude Code is already exercising them).
  DO still select memories containing warnings, gotchas, or known issues about those tools...`
```

选择过程：

```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  // 1. 扫描记忆文件，排除已展示过的
  const memories = (await scanMemoryFiles(memoryDir, signal)).filter(
    m => !alreadySurfaced.has(m.filePath)
  )

  // 2. 让 Sonnet 选择最相关的
  const selectedFilenames = await selectRelevantMemories(
    query, memories, signal, recentTools
  )

  // 3. 映射回文件路径
  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }))
}
```

内部的 `selectRelevantMemories` 使用 `sideQuery`（旁路查询）调用 Sonnet 模型：

```typescript
const result = await sideQuery({
  model: getDefaultSonnetModel(),
  system: SELECT_MEMORIES_SYSTEM_PROMPT,
  skipSystemPromptPrefix: true,
  messages: [{
    role: 'user',
    content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
  }],
  max_tokens: 256,
  output_format: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: {
        selected_memories: { type: 'array', items: { type: 'string' } },
      },
      required: ['selected_memories'],
      additionalProperties: false,
    },
  },
  signal,
  querySource: 'memdir_relevance',
})
```

使用 JSON Schema constrained output 确保返回格式的可靠性。`recentTools` 参数防止了一个常见的误选：当模型正在使用某个工具时，该工具的参考文档不需要被选出（因为上下文中已经有工具的使用示例），但 warning 和 gotcha 仍然需要。

### 12.5.3 记忆清单格式化

```typescript
export function formatMemoryManifest(memories: MemoryHeader[]): string {
  return memories
    .map(m => {
      const tag = m.type ? `[${m.type}] ` : ''
      const ts = new Date(m.mtimeMs).toISOString()
      return m.description
        ? `- ${tag}${m.filename} (${ts}): ${m.description}`
        : `- ${tag}${m.filename} (${ts})`
    })
    .join('\n')
}
```

输出格式示例：
```
- [feedback] testing_policy.md (2026-03-28T10:30:00.000Z): Integration tests must use real database
- [user] role_profile.md (2026-03-25T08:15:00.000Z): Senior data scientist focused on observability
```

## 12.6 记忆老化

### 12.6.1 年龄计算

```typescript
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

注释解释了为什么使用自然语言而非 ISO 时间戳：

> Models are poor at date arithmetic — a raw ISO timestamp doesn't trigger staleness reasoning the way "47 days ago" does.

### 12.6.2 新鲜度警告

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''  // 今天/昨天的记忆不需要警告
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

这解决了一个用户报告的实际问题：模型会引用记忆中的过时代码位置（`file:line`），并以这些引用作为权威依据。加上天数标注后，模型更可能先验证再使用。

## 12.7 团队记忆

当启用团队记忆功能时，系统维护两个目录：

- **Private 目录**（`~/.claude/projects/<slug>/memory/`）：个人记忆
- **Team 目录**（`~/.claude/projects/<slug>/memory/team/`）：团队共享记忆

记忆类型中的 `<scope>` 标签指导模型决定存储位置：

```typescript
// TYPES_SECTION_COMBINED 中的片段
'<type>',
'    <name>user</name>',
'    <scope>always private</scope>',   // 用户信息永远是私有的
...
'<type>',
'    <name>feedback</name>',
'    <scope>default to private. Save as team only when the guidance is clearly
a project-wide convention...</scope>',  // feedback 默认私有，除非是团队规范
...
'<type>',
'    <name>project</name>',
'    <scope>private or team, but strongly bias toward team</scope>',  // 项目信息倾向于共享
```

Team 目录是自动记忆目录的子目录，所以 `mkdir` team 目录时会递归创建 auto 目录。注释中有一个精妙的提醒：

> If the team dir ever moves out from under the auto dir, add a second ensureMemoryDirExists call for autoDir here.

## 12.8 搜索过往上下文

记忆系统还提供了搜索能力，覆盖两个数据源：

```typescript
export function buildSearchingPastContextSection(autoMemDir: string): string[] {
  return [
    '## Searching past context',
    '',
    'When looking for past context:',
    '1. Search topic files in your memory directory:',
    '```',
    `${GREP_TOOL_NAME} with pattern="<search term>" path="${autoMemDir}" glob="*.md"`,
    '```',
    '2. Session transcript logs (last resort — large files, slow):',
    '```',
    `${GREP_TOOL_NAME} with pattern="<search term>" path="${projectDir}/" glob="*.jsonl"`,
    '```',
    'Use narrow search terms (error messages, file paths, function names)...',
  ]
}
```

记忆文件是首选搜索目标（小文件，快速），会话 transcript logs 是最后手段（大文件，慢）。

## 12.9 会话持久化

### 12.9.1 历史记录管理

`history.ts` 管理用户的输入历史，支持上下键浏览和 `ctrl+r` 搜索。

**存储格式**：JSONL（JSON Lines），每行一个历史条目：

```typescript
type LogEntry = {
  display: string                          // 显示文本
  pastedContents: Record<number, StoredPastedContent>  // 粘贴内容
  timestamp: number                        // 时间戳
  project: string                          // 项目路径
  sessionId?: string                       // 会话 ID
}
```

**粘贴内容的分层存储**：

```typescript
type StoredPastedContent = {
  id: number
  type: 'text' | 'image'
  content?: string      // 小内容内联存储
  contentHash?: string  // 大内容通过哈希引用外部存储
  mediaType?: string
  filename?: string
}
```

对于大的粘贴内容（> 1024 字符），系统计算内容哈希并存储到 paste store 中，历史条目只保留哈希引用：

```typescript
if (content.content.length <= MAX_PASTED_CONTENT_LENGTH) {
  storedPastedContents[Number(id)] = {
    id: content.id,
    type: content.type,
    content: content.content,  // 内联
  }
} else {
  const hash = hashPastedText(content.content)
  storedPastedContents[Number(id)] = {
    id: content.id,
    type: content.type,
    contentHash: hash,  // 引用
  }
  void storePastedText(hash, content.content)  // 异步写入，不阻塞
}
```

### 12.9.2 粘贴内容引用系统

```typescript
// 引用格式
export function formatPastedTextRef(id: number, numLines: number): string {
  if (numLines === 0) return `[Pasted text #${id}]`
  return `[Pasted text #${id} +${numLines} lines]`
}

export function formatImageRef(id: number): string {
  return `[Image #${id}]`
}

// 引用解析
export function parseReferences(input: string):
  Array<{ id: number; match: string; index: number }> {
  const referencePattern =
    /\[(Pasted text|Image|\.\.\.Truncated text) #(\d+)(?: \+\d+ lines)?(\.)*\]/g
  return [...input.matchAll(referencePattern)]
    .map(match => ({
      id: parseInt(match[2] || '0'),
      match: match[0],
      index: match.index,
    }))
    .filter(match => match.id > 0)
}
```

引用展开使用反向替换以避免位置偏移：

```typescript
export function expandPastedTextRefs(
  input: string,
  pastedContents: Record<number, PastedContent>,
): string {
  const refs = parseReferences(input)
  let expanded = input
  // 反向遍历，避免替换后的偏移影响前面的引用
  for (let i = refs.length - 1; i >= 0; i--) {
    const ref = refs[i]!
    const content = pastedContents[ref.id]
    if (content?.type !== 'text') continue
    expanded =
      expanded.slice(0, ref.index) +
      content.content +
      expanded.slice(ref.index + ref.match.length)
  }
  return expanded
}
```

### 12.9.3 历史记录的会话隔离

```typescript
export async function* getHistory(): AsyncGenerator<HistoryEntry> {
  const currentProject = getProjectRoot()
  const currentSession = getSessionId()
  const otherSessionEntries: LogEntry[] = []

  for await (const entry of makeLogEntryReader()) {
    if (!entry || typeof entry.project !== 'string') continue
    if (entry.project !== currentProject) continue

    if (entry.sessionId === currentSession) {
      yield await logEntryToHistoryEntry(entry)
      yielded++
    } else {
      otherSessionEntries.push(entry)
    }
    if (yielded + otherSessionEntries.length >= MAX_HISTORY_ITEMS) break
  }

  // 当前会话的条目优先，然后是其他会话
  for (const entry of otherSessionEntries) {
    if (yielded >= MAX_HISTORY_ITEMS) return
    yield await logEntryToHistoryEntry(entry)
    yielded++
  }
}
```

当前会话的条目优先于其他会话的条目，这样并发会话不会相互干扰上下键历史。

### 12.9.4 写入的可靠性

```typescript
async function addToPromptHistory(command: HistoryEntry | string): Promise<void> {
  // ... 构建 logEntry ...
  pendingEntries.push(logEntry)
  lastAddedEntry = logEntry
  currentFlushPromise = flushPromptHistory(0)
}

async function flushPromptHistory(retries: number): Promise<void> {
  if (isWriting || pendingEntries.length === 0) return
  if (retries > 5) return  // 5次重试后放弃

  isWriting = true
  try {
    await immediateFlushHistory()
  } finally {
    isWriting = false
    if (pendingEntries.length > 0) {
      await sleep(500)
      void flushPromptHistory(retries + 1)
    }
  }
}
```

写入使用文件锁（`lock(historyPath, { stale: 10000 })`）防止并发会话的写入冲突，并通过 `registerCleanup` 确保进程退出时刷新所有待写入的条目。

### 12.9.5 撤销最近的历史条目

```typescript
export function removeLastFromHistory(): void {
  if (!lastAddedEntry) return
  const entry = lastAddedEntry
  lastAddedEntry = null

  // 快速路径：从待写入缓冲区中移除
  const idx = pendingEntries.lastIndexOf(entry)
  if (idx !== -1) {
    pendingEntries.splice(idx, 1)
  } else {
    // 慢速路径：已刷入磁盘，记录到跳过集合
    skippedTimestamps.add(entry.timestamp)
  }
}
```

这用于"按 Esc 撤销"场景：用户提交了一个 prompt 但在响应到来前按 Esc 取消，此时历史条目也应该被撤销。

## 12.10 成本追踪

### 12.10.1 Token 使用统计

`cost-tracker.ts` 跟踪每个模型的 Token 使用量：

```typescript
function addToTotalModelUsage(cost: number, usage: Usage, model: string): ModelUsage {
  const modelUsage = getUsageForModel(model) ?? {
    inputTokens: 0,
    outputTokens: 0,
    cacheReadInputTokens: 0,
    cacheCreationInputTokens: 0,
    webSearchRequests: 0,
    costUSD: 0,
    contextWindow: 0,
    maxOutputTokens: 0,
  }

  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost
  return modelUsage
}
```

### 12.10.2 跨会话持久化

成本数据通过 project config 跨会话持久化：

```typescript
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastToolDuration: getTotalToolDuration(),
    lastLinesAdded: getTotalLinesAdded(),
    lastLinesRemoved: getTotalLinesRemoved(),
    lastTotalInputTokens: getTotalInputTokens(),
    lastTotalOutputTokens: getTotalOutputTokens(),
    lastTotalCacheCreationInputTokens: getTotalCacheCreationInputTokens(),
    lastTotalCacheReadInputTokens: getTotalCacheReadInputTokens(),
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        model,
        { inputTokens: usage.inputTokens, outputTokens: usage.outputTokens, ... },
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}
```

恢复时验证 sessionId 是否匹配：

```typescript
export function restoreCostStateForSession(sessionId: string): boolean {
  const data = getStoredSessionCosts(sessionId)
  if (!data) return false
  setCostStateForRestore(data)
  return true
}
```

### 12.10.3 成本格式化

```typescript
export function formatTotalCost(): string {
  const costDisplay = formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')

  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}\n` +
    `Total duration (wall): ${formatDuration(getTotalDuration())}\n` +
    `Total code changes:    ${getTotalLinesAdded()} lines added, ${getTotalLinesRemoved()} lines removed\n` +
    `${modelUsageDisplay}`,
  )
}
```

成本格式化区分了两个精度：大于 $0.50 的显示两位小数，小于 $0.50 的显示四位小数。这避免了小成本显示为 `$0.00` 而大成本显示不必要的精度。

### 12.10.4 Advisor 用量追踪

Claude Code 还追踪"advisor"（API 侧工具调用如 web search）的用量：

```typescript
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', {
    advisor_model: advisorUsage.model,
    input_tokens: advisorUsage.input_tokens,
    output_tokens: advisorUsage.output_tokens,
    cost_usd_micros: Math.round(advisorCost * 1_000_000),
  })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

成本以微美元（`cost_usd_micros`）为单位记录到 analytics，避免浮点精度问题。

## 12.11 小结

Claude Code 的记忆和持久化系统体现了以下核心设计原则：

1. **文件即记忆**：使用 Markdown 文件和 YAML frontmatter，人类可读、git 可追踪
2. **类型化分类**：四种记忆类型覆盖了不同的知识维度，防止存储冗余信息
3. **主动老化**：通过天数标注和新鲜度警告，防止模型过度信任过时记忆
4. **智能检索**：使用 Sonnet 模型进行相关性选择，而非简单的关键词匹配
5. **安全优先**：路径验证、项目设置排除、worktree 共享等都体现了安全意识
6. **性能考量**：Memoization、只读 frontmatter、异步写入等确保了低延迟
7. **跨会话连续性**：历史记录、成本追踪、记忆系统共同确保了工作的连续性

