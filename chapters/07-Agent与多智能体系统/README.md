# 第七章：Agent 与多智能体系统

Claude Code 不仅仅是一个单一的 AI 助手，它内建了一套完整的多智能体(multi-agent)架构。从单个子 Agent 的派生到团队(swarm)协作，从进程内通信到 tmux 终端隔离，Claude Code 展现了一个在生产环境中运行多 Agent 系统的工程蓝图。本章将从 AgentTool 的核心实现出发，逐层剖析这套多智能体系统的设计。

## 7.1 Agent 架构概览

Claude Code 的 Agent 系统呈现树状层次结构：

```
                ┌─────────────────┐
                │   Main Session  │
                │  (Coordinator)  │
                └────────┬────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────┴──────┐ ┌────┴────┐  ┌──────┴──────┐
   │  AgentTool  │ │  Fork   │  │  Teammate   │
   │  (同步/异步) │ │ (隐式)  │  │  (Swarm)    │
   └──────┬──────┘ └────┬────┘  └──────┬──────┘
          │              │              │
     ┌────┴────┐    直接执行     ┌──────┴──────┐
     │ 嵌套Agent│              │  独立进程    │
     │ (递归)  │              │  (Tmux/In-  │
     └─────────┘              │   Process)  │
                              └─────────────┘
```

系统中有三种 Agent 调用路径：
1. **AgentTool** —— 标准的子 Agent 调用，可同步或异步执行
2. **Fork 子 Agent** —— 隐式派生，继承父级完整对话上下文
3. **Teammate (Swarm)** —— 团队协作模式，多个 Agent 在独立进程中并行工作

## 7.2 AgentTool 核心实现

### 7.2.1 输入 Schema 设计

AgentTool 的输入 Schema 是根据 feature flag 动态构建的：

```typescript
// AgentTool.tsx
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe(
    'The type of specialized agent to use for this task'
  ),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional().describe(
    "Optional model override. Takes precedence over the agent definition's model."
  ),
  run_in_background: z.boolean().optional().describe(
    'Set to true to run this agent in the background.'
  )
}))
```

完整 Schema 还包含多 Agent 参数：

```typescript
const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional().describe(
      'Name for the spawned agent. Makes it addressable via SendMessage.'
    ),
    team_name: z.string().optional().describe(
      'Team name for spawning. Uses current team context if omitted.'
    ),
    mode: permissionModeSchema().optional().describe(
      'Permission mode for spawned teammate.'
    )
  })
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree']).optional().describe(
      'Isolation mode. "worktree" creates a temporary git worktree.'
    ),
    cwd: z.string().optional().describe(
      'Absolute path to run the agent in.'
    )
  })
})
```

Schema 通过 `.omit()` 动态裁剪不可用的字段：

```typescript
export const inputSchema = lazySchema(() => {
  const schema = feature('KAIROS') ? fullInputSchema() : fullInputSchema().omit({ cwd: true })
  return isBackgroundTasksDisabled || isForkSubagentEnabled()
    ? schema.omit({ run_in_background: true })
    : schema
})
```

这种设计确保模型在特定配置下不会看到不可用的参数，避免产生无效调用。

### 7.2.2 输出类型系统

AgentTool 的输出是一个判别联合(discriminated union)，根据执行路径返回不同结构：

```typescript
// 公开 Schema
const outputSchema = lazySchema(() => {
  const syncOutputSchema = agentToolResultSchema().extend({
    status: z.literal('completed'),
    prompt: z.string()
  })
  const asyncOutputSchema = z.object({
    status: z.literal('async_launched'),
    agentId: z.string(),
    description: z.string(),
    prompt: z.string(),
    outputFile: z.string(),
    canReadOutputFile: z.boolean().optional()
  })
  return z.union([syncOutputSchema, asyncOutputSchema])
})

// 内部类型（不导出到 Schema，用于 dead code elimination）
type TeammateSpawnedOutput = {
  status: 'teammate_spawned'
  teammate_id: string
  agent_id: string
  name: string
  tmux_session_name: string
  tmux_pane_id: string
  // ...
}

type RemoteLaunchedOutput = {
  status: 'remote_launched'
  taskId: string
  sessionUrl: string
  // ...
}
```

`TeammateSpawnedOutput` 和 `RemoteLaunchedOutput` 被有意排除在公开 Schema 之外，这是一种 dead code elimination 策略——构建时未开启相关 feature 的版本不会包含这些代码路径。

### 7.2.3 调用路由逻辑

`call` 方法的路由逻辑是理解 Agent 系统的关键：

```typescript
async call({ prompt, subagent_type, description, model, run_in_background,
             name, team_name, mode, isolation, cwd }, toolUseContext, ...) {

  // 路径 1：团队 Teammate 生成
  if (teamName && name) {
    const result = await spawnTeammate({ name, prompt, description, team_name, ... })
    return { data: { status: 'teammate_spawned', ...result.data } }
  }

  // 路径 2：Fork 子 Agent（实验性）
  const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)
  const isForkPath = effectiveType === undefined

  if (isForkPath) {
    // 递归保护
    if (isInForkChild(toolUseContext.messages)) {
      throw new Error('Fork is not available inside a forked worker.')
    }
    selectedAgent = FORK_AGENT
  } else {
    // 路径 3：标准 Agent 选择
    const agents = filterDeniedAgents(allAgents, appState.toolPermissionContext, AGENT_TOOL_NAME)
    selectedAgent = agents.find(agent => agent.agentType === effectiveType)
  }

  // ... 执行选定的 Agent
}
```

## 7.3 Agent 定义体系

### 7.3.1 内置 Agent 类型

Claude Code 预定义了多种专用 Agent，通过 `AgentDefinition` 类型系统管理：

```typescript
export type BaseAgentDefinition = {
  agentType: string          // 唯一标识
  whenToUse: string          // 使用场景描述
  tools?: string[]           // 可用工具列表
  disallowedTools?: string[] // 禁用工具列表
  skills?: string[]          // 预加载的 Skill
  mcpServers?: AgentMcpServerSpec[]  // Agent 专属 MCP 服务器
  hooks?: HooksSettings      // 会话级 Hook
  model?: string             // 模型选择
  effort?: EffortValue       // 推理力度
  permissionMode?: PermissionMode    // 权限模式
  maxTurns?: number          // 最大轮次
  color?: AgentColorName     // UI 颜色
  background?: boolean       // 始终后台运行
  isolation?: 'worktree' | 'remote'  // 隔离模式
  memory?: AgentMemoryScope  // 持久记忆
  omitClaudeMd?: boolean     // 省略 CLAUDE.md（优化 token）
}

// 内置 Agent
export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  getSystemPrompt: (params) => string  // 动态 system prompt
}

// 自定义 Agent（用户/项目配置）
export type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource
}

// 插件 Agent
export type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}
```

内置 Agent 类型包括：
- **general-purpose** —— 通用子 Agent，适用于大多数任务
- **Explore** —— 只读搜索 Agent，省略 CLAUDE.md 以节省 token
- **Plan** —— 规划 Agent，用于方案设计
- **verification** —— 验证 Agent，用于测试和确认
- **claude-code-guide** —— Claude Code 自身的使用指南 Agent

### 7.3.2 自定义 Agent 定义

用户可以在 `.claude/agents/` 目录下创建 Markdown 文件来定义自定义 Agent。文件通过 frontmatter 指定配置，正文作为 system prompt：

```typescript
const AgentJsonSchema = lazySchema(() =>
  z.object({
    description: z.string().min(1),
    tools: z.array(z.string()).optional(),
    disallowedTools: z.array(z.string()).optional(),
    prompt: z.string().min(1),
    model: z.string().trim().min(1)
      .transform(m => (m.toLowerCase() === 'inherit' ? 'inherit' : m))
      .optional(),
    effort: z.union([z.enum(EFFORT_LEVELS), z.number().int()]).optional(),
    permissionMode: z.enum(PERMISSION_MODES).optional(),
    mcpServers: z.array(AgentMcpServerSpecSchema()).optional(),
    hooks: HooksSchema().optional(),
    maxTurns: z.number().int().positive().optional(),
    skills: z.array(z.string()).optional(),
    initialPrompt: z.string().optional(),
    memory: z.enum(['user', 'project', 'local']).optional(),
    background: z.boolean().optional(),
    isolation: z.enum(['worktree']).optional(),
  }),
)
```

Agent 加载器实现了优先级覆盖机制：

```typescript
export function getActiveAgentsFromList(allAgents: AgentDefinition[]): AgentDefinition[] {
  const builtInAgents  = allAgents.filter(a => a.source === 'built-in')
  const pluginAgents   = allAgents.filter(a => a.source === 'plugin')
  const userAgents     = allAgents.filter(a => a.source === 'userSettings')
  const projectAgents  = allAgents.filter(a => a.source === 'projectSettings')
  const managedAgents  = allAgents.filter(a => a.source === 'policySettings')
  // 按优先级合并，projectSettings > userSettings > plugin > built-in
}
```

### 7.3.3 Agent MCP 服务器

每个 Agent 可以定义自己的 MCP 服务器，既支持引用已配置的服务器，也支持内联定义：

```typescript
export type AgentMcpServerSpec =
  | string                            // 引用："slack"
  | { [name: string]: McpServerConfig }  // 内联定义

// 初始化逻辑
async function initializeAgentMcpServers(agentDefinition, parentClients) {
  for (const spec of agentDefinition.mcpServers) {
    if (typeof spec === 'string') {
      // 按名称查找已配置的 MCP 服务器（memoized，可能共享父级客户端）
      config = getMcpConfigByName(spec)
    } else {
      // 内联定义：创建新连接，Agent 结束时清理
      isNewlyCreated = true
    }
    const client = await connectToServer(name, config)
    agentClients.push(client)
    if (isNewlyCreated) newlyCreatedClients.push(client)
  }

  // 清理函数只清理新创建的客户端，不影响共享客户端
  const cleanup = async () => {
    for (const client of newlyCreatedClients) {
      if (client.type === 'connected') await client.cleanup()
    }
  }
  return { clients: [...parentClients, ...agentClients], tools: agentTools, cleanup }
}
```

这种设计实现了资源的精确管理：引用的 MCP 服务器与父级共享，内联定义的服务器随 Agent 生命周期创建和销毁。

## 7.4 Agent 执行引擎

`runAgent.ts` 是 Agent 执行的核心引擎。它是一个 AsyncGenerator，逐步产出 Agent 生成的消息。

### 7.4.1 模型解析与权限配置

```typescript
export async function* runAgent({
  agentDefinition, promptMessages, toolUseContext, canUseTool,
  isAsync, canShowPermissionPrompts, forkContextMessages,
  querySource, override, model, maxTurns, availableTools,
  allowedTools, useExactTools, worktreePath, ...
}) {
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model,     // Agent 定义的模型
    toolUseContext.options.mainLoopModel,  // 父级模型
    model,                     // 调用时的显式覆盖
    permissionMode,
  )

  const agentId = override?.agentId ? override.agentId : createAgentId()
```

模型解析的优先级为：显式参数 > Agent 定义 > 父级继承。特殊值 `'inherit'` 表示使用父级的模型。

### 7.4.2 权限模式继承

子 Agent 的权限模式遵循严格的继承规则：

```typescript
const agentGetAppState = () => {
  const state = toolUseContext.getAppState()
  let toolPermissionContext = state.toolPermissionContext

  // 如果 Agent 定义了权限模式，使用它
  // 但 bypassPermissions、acceptEdits、auto 模式始终优先
  if (agentPermissionMode &&
      state.toolPermissionContext.mode !== 'bypassPermissions' &&
      state.toolPermissionContext.mode !== 'acceptEdits' &&
      !(feature('TRANSCRIPT_CLASSIFIER') && state.toolPermissionContext.mode === 'auto')) {
    toolPermissionContext = { ...toolPermissionContext, mode: agentPermissionMode }
  }

  // 异步 Agent 无法显示权限提示 -> 自动拒绝
  // bubble 模式例外：冒泡到父终端
  const shouldAvoidPrompts = canShowPermissionPrompts !== undefined
    ? !canShowPermissionPrompts
    : agentPermissionMode === 'bubble' ? false : isAsync

  if (shouldAvoidPrompts) {
    toolPermissionContext = {
      ...toolPermissionContext,
      shouldAvoidPermissionPrompts: true,
    }
  }

  // 限定工具权限范围
  if (allowedTools !== undefined) {
    toolPermissionContext = {
      ...toolPermissionContext,
      alwaysAllowRules: {
        cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,  // 保留 SDK 级权限
        session: [...allowedTools],  // 仅使用显式提供的工具权限
      },
    }
  }
  // ...
}
```

关键安全考虑：
- `bypassPermissions`、`acceptEdits` 和 `auto` 模式来自父级时优先于 Agent 定义，确保用户的全局选择不被子 Agent 覆盖
- SDK 级权限（`cliArg`）始终保留，会话级权限则重新设定
- 后台 Agent 但允许显示提示的，会设置 `awaitAutomatedChecksBeforeDialog`，让自动检查先跑完再打扰用户

### 7.4.3 FileState 缓存

Agent 维护独立的文件状态缓存，用于增量编辑和 diff 计算：

```typescript
// Fork Agent 继承父级缓存的克隆
const agentReadFileState = forkContextMessages !== undefined
  ? cloneFileStateCache(toolUseContext.readFileState)
  : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)
```

Fork Agent 克隆父级缓存是因为它继承了完整的对话上下文——父级提到的文件在子 Agent 中也应该有缓存。普通子 Agent 则从空缓存开始。

### 7.4.4 System Prompt 增强

Explore/Plan 等只读 Agent 会省略 CLAUDE.md 和 gitStatus：

```typescript
// 只读 Agent 不需要 commit/PR/lint 指南
const shouldOmitClaudeMd = agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)

// Explore/Plan 也不需要过时的 gitStatus（40KB+）
const resolvedSystemContext = agentDefinition.agentType === 'Explore' ||
  agentDefinition.agentType === 'Plan'
  ? systemContextNoGit : baseSystemContext
```

这个优化的规模令人印象深刻：源码注释提到"saves ~5-15 Gtok/week across 34M+ Explore spawns"——在数千万次 Agent 调用的规模下，每个 Agent 省下的 token 累积起来非常可观。

## 7.5 Fork 子 Agent 机制

Fork 是 Claude Code 中最精巧的 Agent 派生机制。它让子 Agent 继承父级的完整对话上下文，同时保持 prompt cache 的共享。

### 7.5.1 Fork Agent 定义

```typescript
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  whenToUse: 'Implicit fork — inherits full conversation context.',
  tools: ['*'],              // 继承父级全部工具
  maxTurns: 200,
  model: 'inherit',          // 继承父级模型
  permissionMode: 'bubble',  // 权限冒泡到父终端
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '',  // 不使用——由 override.systemPrompt 提供
} satisfies BuiltInAgentDefinition
```

注意 `getSystemPrompt` 返回空字符串——Fork Agent 通过 `override.systemPrompt` 直接传入父级已渲染的 system prompt 字节，避免因重新渲染导致的 prompt cache 失效。

### 7.5.2 消息构建与 Cache 共享

`buildForkedMessages` 是 Fork 机制的核心，它确保所有 Fork 子 Agent 生成字节相同的 API 请求前缀：

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 保留完整的 assistant 消息（所有 tool_use、thinking、text）
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID(), ... }

  // 2. 收集所有 tool_use 块
  const toolUseBlocks = assistantMessage.message.content.filter(
    block => block.type === 'tool_use',
  )

  // 3. 为每个 tool_use 生成相同的占位符 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 4. 最后一个 text block 是每个子 Agent 唯一不同的部分
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,           // 所有子 Agent 相同
      { type: 'text', text: buildChildMessage(directive) },  // 每个子 Agent 不同
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**Cache 共享原理**：API 的 prompt cache 基于请求前缀的字节匹配。所有 Fork 子 Agent 的请求前缀为 `[...历史消息, assistant(所有tool_use), user(占位符results...)]`，这部分完全相同。只有末尾的 directive 文本不同，不影响前缀匹配。这意味着第一个 Fork 子 Agent 填充缓存后，后续子 Agent 几乎零成本复用。

### 7.5.3 递归保护

Fork 子 Agent 的工具池中保留了 Agent tool（用于保持 API 请求的 tool 定义字节一致），但在运行时通过双重检查阻止递归 fork：

```typescript
// 主检查：querySource（抗消息压缩）
if (toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}`) {
  throw new Error('Fork is not available inside a forked worker.')
}

// 回退检查：扫描消息历史中的 fork 标记
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    return m.message.content.some(
      block => block.type === 'text' && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

### 7.5.4 Fork 子 Agent 指令

每个 Fork 子 Agent 收到的 boilerplate 是一套严格的行为规范：

```typescript
export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for the parent.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. If you modify files, commit your changes before reporting.
5. Do NOT emit text between tool calls.
6. Stay strictly within your directive's scope.
7. Keep your report under 500 words.
8. Your response MUST begin with "Scope:".
9. REPORT structured facts, then stop

Output format:
  Scope: <echo back assigned scope>
  Result: <answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list — only if there are issues>
</${FORK_BOILERPLATE_TAG}>

<fork-directive>${directive}`
}
```

## 7.6 团队/蜂群系统

Claude Code 的蜂群(swarm)系统允许多个 Agent 在独立进程中并行工作，通过文件级邮箱进行通信。

### 7.6.1 TeamCreateTool：团队创建

TeamCreateTool 负责创建团队的基础设施：

```typescript
export const TeamCreateTool = buildTool({
  name: TEAM_CREATE_TOOL_NAME,

  async call(input, context) {
    // 1. 检查不能同时领导多个团队
    if (existingTeam) {
      throw new Error('Already leading team. Use TeamDelete first.')
    }

    // 2. 生成唯一团队名
    const finalTeamName = generateUniqueTeamName(team_name)

    // 3. 创建 Team Lead
    const leadAgentId = formatAgentId(TEAM_LEAD_NAME, finalTeamName)

    // 4. 创建团队文件
    const teamFile: TeamFile = {
      name: finalTeamName,
      description,
      createdAt: Date.now(),
      leadAgentId,
      leadSessionId: getSessionId(),
      members: [{
        agentId: leadAgentId,
        name: TEAM_LEAD_NAME,
        agentType: leadAgentType,
        model: leadModel,
        joinedAt: Date.now(),
        cwd: getCwd(),
        subscriptions: [],
      }],
    }
    await writeTeamFileAsync(finalTeamName, teamFile)

    // 5. 初始化任务列表目录
    const taskListId = sanitizeName(finalTeamName)
    await resetTaskList(taskListId)
    await ensureTasksDir(taskListId)

    // 6. 更新 AppState
    setAppState(prev => ({
      ...prev,
      teamContext: { teamName: finalTeamName, teamFilePath, leadAgentId, teammates: { ... } }
    }))
  }
})
```

团队文件(TeamFile)是蜂群系统的持久化状态，存储在磁盘上。每个团队对应一个任务列表目录，任务编号从 1 开始。

### 7.6.2 SendMessageTool：队友间通信

SendMessageTool 实现了基于文件的消息传递系统：

```typescript
const inputSchema = lazySchema(() =>
  z.object({
    to: z.string().describe('Recipient: teammate name, or "*" for broadcast'),
    summary: z.string().optional().describe('5-10 word summary for UI preview'),
    message: z.union([
      z.string(),
      StructuredMessage(),  // 结构化消息：shutdown_request, shutdown_response, plan_approval_response
    ]),
  })
)
```

消息路由支持三种模式：

**点对点消息**：
```typescript
async function handleMessage(recipientName, content, summary, context) {
  await writeToMailbox(recipientName, {
    from: senderName,
    text: content,
    summary,
    timestamp: new Date().toISOString(),
    color: senderColor,
  }, teamName)

  return { data: { success: true, message: `Message sent to ${recipientName}'s inbox`, routing: { ... } } }
}
```

**广播消息**：
```typescript
async function handleBroadcast(content, summary, context) {
  const teamFile = await readTeamFileAsync(teamName)
  const recipients = teamFile.members.filter(m => m.name !== senderName)

  for (const recipientName of recipients) {
    await writeToMailbox(recipientName, { from: senderName, text: content, ... }, teamName)
  }

  return { data: { success: true, recipients: recipients.map(r => r.name), ... } }
}
```

**关闭协议**：
```typescript
// 结构化消息类型
const StructuredMessage = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('shutdown_request'),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('shutdown_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('plan_approval_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    feedback: z.string().optional(),
  }),
])
```

关闭协议允许一个 Agent 请求另一个 Agent 优雅退出。被请求方可以批准或拒绝，提供理由。这种协商式关闭避免了强制终止可能导致的数据不一致。

### 7.6.3 后端类型

蜂群系统支持多种后端来运行 Teammate：

```typescript
export type BackendType = 'tmux' | 'iterm2' | 'inprocess'
```

**Tmux 后端**：在 tmux session 的独立 pane 中启动 Claude Code 实例：

```typescript
async function ensureSession(sessionName: string): Promise<void> {
  const exists = await hasSession(sessionName)
  if (!exists) {
    await execFileNoThrow(TMUX_COMMAND, ['new-session', '-d', '-s', sessionName])
  }
}
```

Teammate 通过 CLI 参数继承父级的配置：

```typescript
function buildInheritedCliFlags(options?) {
  const flags: string[] = []
  // 继承权限模式
  if (permissionMode === 'bypassPermissions') flags.push('--dangerously-skip-permissions')
  else if (permissionMode === 'acceptEdits') flags.push('--permission-mode acceptEdits')
  else if (permissionMode === 'auto') flags.push('--permission-mode auto')

  // 继承模型选择
  if (modelOverride) flags.push(`--model ${quote([modelOverride])}`)

  // 继承设置文件和插件
  if (settingsPath) flags.push(`--settings ${quote([settingsPath])}`)
  for (const pluginDir of inlinePlugins) flags.push(`--plugin-dir ${quote([pluginDir])}`)

  return flags.join(' ')
}
```

**InProcess 后端**：在同一进程内运行 Teammate，共享终端的权限提示能力，但有生命周期限制：

```typescript
// InProcess Teammate 不能生成后台 Agent
if (isInProcessTeammate() && teamName && run_in_background === true) {
  throw new Error('In-process teammates cannot spawn background agents.')
}
```

### 7.6.4 邮箱系统

通信基于文件级邮箱——每个 Teammate 有一个以其名称命名的邮箱目录。`writeToMailbox` 将消息序列化为 JSON 写入文件，Teammate 通过轮询读取新消息。

这种设计的优点：
- **进程隔离**：不同 Teammate 可以在不同的 tmux pane 甚至不同机器上运行
- **崩溃恢复**：消息持久化在磁盘上，Teammate 重启后可以继续处理
- **简单可靠**：不需要 IPC socket 或网络协议

## 7.7 Git Worktree 隔离

EnterWorktreeTool 提供了基于 git worktree 的文件系统级隔离：

```typescript
export const EnterWorktreeTool = buildTool({
  name: ENTER_WORKTREE_TOOL_NAME,

  async call(input) {
    // 1. 检查不在已有 worktree 中
    if (getCurrentWorktreeSession()) throw new Error('Already in a worktree session')

    // 2. 回到主仓库根目录
    const mainRepoRoot = findCanonicalGitRoot(getCwd())
    if (mainRepoRoot && mainRepoRoot !== getCwd()) {
      process.chdir(mainRepoRoot)
      setCwd(mainRepoRoot)
    }

    // 3. 创建 worktree
    const worktreeSession = await createWorktreeForSession(getSessionId(), slug)

    // 4. 切换工作目录
    process.chdir(worktreeSession.worktreePath)
    setCwd(worktreeSession.worktreePath)
    setOriginalCwd(getCwd())

    // 5. 清除依赖 CWD 的缓存
    clearSystemPromptSections()
    clearMemoryFileCaches()
    getPlansDirectory.cache.clear?.()

    return { data: { worktreePath, worktreeBranch, message: '...' } }
  }
})
```

当 Fork Agent 在 worktree 中运行时，会收到上下文转换通知：

```typescript
export function buildWorktreeNotice(parentCwd: string, worktreeCwd: string): string {
  return `You've inherited the conversation context above from a parent agent
working in ${parentCwd}. You are operating in an isolated git worktree at
${worktreeCwd}. Paths in the inherited context refer to the parent's working
directory; translate them to your worktree root. Re-read files before editing
if the parent may have modified them.`
}
```

## 7.8 任务框架

### 7.8.1 核心类型

Agent 执行被抽象为"任务"(Task)，统一管理生命周期：

```typescript
// Task.ts
export type TaskType =
  | 'local_bash'            // 本地 Shell 任务
  | 'local_agent'           // 本地 Agent 任务
  | 'remote_agent'          // 远程 Agent 任务
  | 'in_process_teammate'   // 进程内 Teammate
  | 'local_workflow'        // 本地工作流
  | 'monitor_mcp'           // MCP 监控
  | 'dream'                 // Dream 任务

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

任务 ID 使用类型前缀加随机字符串：

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

export function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id  // 例如 "a7k2m9p1x3"
}
```

使用 36^8 的字符空间（约 2.8 万亿种组合），注释指出这"sufficient to resist brute-force symlink attacks"。

### 7.8.2 任务状态联合类型

`tasks/types.ts` 定义了所有具体任务状态的联合类型：

```typescript
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState

export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') return false
  if ('isBackgrounded' in task && task.isBackgrounded === false) return false
  return true
}
```

### 7.8.3 TaskHandle

`TaskHandle` 是任务的轻量级句柄，用于注册和清理：

```typescript
export type TaskHandle = {
  taskId: string
  cleanup?: () => void
}

export type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}
```

## 7.9 Coordinator 模式

Coordinator 模式是一种特殊的运行模式，将主 Agent 转变为纯粹的协调者，所有实际工作由 Worker Agent 完成。

### 7.9.1 模式检测

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

会话恢复时自动匹配模式：

```typescript
export function matchSessionMode(sessionMode: 'coordinator' | 'normal' | undefined): string | undefined {
  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'
  if (currentIsCoordinator === sessionIsCoordinator) return undefined

  // 翻转环境变量
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }
  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

### 7.9.2 Coordinator System Prompt

Coordinator 模式有一套精心设计的 system prompt，定义了协调者的行为规范：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible

## 2. Your Tools
- **${AGENT_TOOL_NAME}** - Spawn a new worker
- **${SEND_MESSAGE_TOOL_NAME}** - Continue an existing worker
- **${TASK_STOP_TOOL_NAME}** - Stop a running worker

## 4. Task Workflow
| Phase          | Who      | Purpose                        |
|----------------|----------|--------------------------------|
| Research       | Workers  | Investigate codebase           |
| Synthesis      | **You**  | Read findings, craft specs     |
| Implementation | Workers  | Make targeted changes          |
| Verification   | Workers  | Test changes work              |
...`
}
```

Coordinator prompt 中最重要的原则是 **Synthesis**（综合）："When workers report research findings, you must understand them before directing follow-up work." 协调者不能简单地转发信息，必须消化研究结果、提取具体的文件路径和行号，然后撰写精确的实施规格。

Worker 的工具范围被明确限定：

```typescript
export function getCoordinatorUserContext(mcpClients, scratchpadDir?) {
  const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
    .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
    .sort().join(', ')

  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to: ${workerTools}`

  // Scratchpad：跨 Worker 共享的临时目录
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\nScratchpad directory: ${scratchpadDir}
Workers can read and write here without permission prompts.`
  }
}
```

## 7.10 多 Agent 通信模式总结

Claude Code 的多 Agent 系统展现了三种不同的通信模式：

### 模式 1：同步调用 (AgentTool → runAgent)
- **特点**：调用方阻塞等待，子 Agent 在同一进程内运行
- **通信**：通过 AsyncGenerator yield 消息
- **适用**：需要即时结果的简单委托

### 模式 2：异步通知 (Background Agent → task-notification)
- **特点**：后台运行，通过 `<task-notification>` XML 回报结果
- **通信**：任务输出文件 + AppState 更新
- **适用**：长时间运行的独立任务

### 模式 3：邮箱通信 (Teammate → Mailbox)
- **特点**：独立进程，通过文件邮箱通信
- **通信**：`writeToMailbox` / 轮询读取
- **适用**：需要进程隔离的并行协作

三种模式之间的选择取决于：
- **隔离需求**：是否需要独立的文件系统和进程空间
- **交互需求**：是否需要与用户或其他 Agent 的持续交互
- **生命周期**：任务是短期还是长期运行

```
┌────────────────────────────────────────────────────────┐
│                    通信模式对比                          │
├──────────┬──────────────┬──────────────┬───────────────┤
│ 特性      │ 同步调用      │ 异步通知      │ 邮箱通信       │
├──────────┼──────────────┼──────────────┼───────────────┤
│ 进程模型  │ 同一进程      │ 同一进程      │ 独立进程       │
│ 通信方式  │ Generator    │ XML通知       │ 文件邮箱       │
│ 权限模式  │ 继承/override │ 自动拒绝/hook │ 独立配置       │
│ 缓存共享  │ 共享 prompt  │ 共享 prompt   │ 无共享         │
│ 崩溃隔离  │ 无           │ 无            │ 完全隔离       │
│ 文件隔离  │ 共享 CWD     │ 共享/worktree │ worktree/远程  │
└──────────┴──────────────┴──────────────┴───────────────┘
```

Claude Code 的多 Agent 架构证明了一个工程原则：**多 Agent 系统的核心挑战不在于"如何让多个 Agent 对话"，而在于"如何管理它们的资源隔离、权限边界和通信可靠性"。** 从 prompt cache 共享到文件邮箱，从递归保护到关闭协商，每一个设计决策都在平衡性能、安全和可靠性。

---
