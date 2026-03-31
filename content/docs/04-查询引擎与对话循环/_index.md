---
title: "第四章：查询引擎与对话循环"
weight: 40
bookCollapseSection: false
---


查询引擎（Query Engine）是 Claude Code 的"心脏"，它驱动了人与 AI 之间每一轮对话的完整生命周期。从用户输入到 API 调用，从流式响应到工具执行，从上下文压缩到错误恢复——所有核心逻辑都在这里汇聚。本章将深入剖析 `QueryEngine` 类与 `query()` 函数的设计，揭示一个工业级 AI Agent 对话循环的全部复杂性。

## 4.1 架构概览

查询引擎的核心分布在以下几个文件中：

| 文件 | 职责 |
|------|------|
| `QueryEngine.ts` | 会话级生命周期管理，消息持久化，SDK 接口 |
| `query.ts` | 核心查询循环（queryLoop），API 调用，工具执行 |
| `query/config.ts` | 不可变查询配置快照 |
| `query/deps.ts` | 依赖注入容器 |
| `query/stopHooks.ts` | 停止条件 Hook 评估 |
| `query/tokenBudget.ts` | Token 预算追踪与续传决策 |

它们的协作关系可以用下图表示：

```
┌─────────────────────────────────────────────────┐
│                  QueryEngine                     │
│  ┌─────────────┐  ┌──────────┐  ┌────────────┐ │
│  │ mutableMsgs │  │ totalUsage│  │ readFileState│ │
│  └──────┬──────┘  └────┬─────┘  └──────┬─────┘ │
│         │              │               │        │
│  ┌──────▼──────────────▼───────────────▼──────┐ │
│  │            submitMessage()                  │ │
│  │  ┌────────────────────────────────────────┐│ │
│  │  │  processUserInput → systemPrompt build ││ │
│  │  │  → query() → for await (message) {     ││ │
│  │  │       record + yield SDK messages      ││ │
│  │  │  }                                     ││ │
│  │  └────────────────────────────────────────┘│ │
│  └─────────────────────┬──────────────────────┘ │
└────────────────────────┼────────────────────────┘
                         │
         ┌───────────────▼───────────────┐
         │         query()                │
         │  ┌─────────────────────────┐  │
         │  │      queryLoop()         │  │
         │  │  while (true) {          │  │
         │  │    preprocess messages   │  │
         │  │    call API (streaming)  │  │
         │  │    execute tools         │  │
         │  │    evaluate stop conds   │  │
         │  │    state = next          │  │
         │  │  }                       │  │
         │  └─────────────────────────┘  │
         └───────────────────────────────┘
```

## 4.2 QueryEngine 类的设计

### 4.2.1 配置与状态

`QueryEngine` 通过 `QueryEngineConfig` 接收所有初始化参数：

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
  handleElicitation?: ToolUseContext['handleElicitation']
  includePartialMessages?: boolean
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

这个配置体现了几个重要的设计决策：

1. **状态管理外部化**：`getAppState` 和 `setAppState` 以回调方式注入，QueryEngine 不持有 AppState 的所有权。
2. **权限判断可注入**：`canUseTool` 作为函数注入，允许不同运行模式（REPL、SDK、Headless）使用不同的权限策略。
3. **模型可替换**：`userSpecifiedModel` 和 `fallbackModel` 支持运行时模型切换，这是实现 Fallback 策略的基础。
4. **Snip 回放注入**：`snipReplay` 回调由外层 `ask()` 注入，使 feature gate 后的字符串不渗入 QueryEngine。

QueryEngine 内部的可变状态严格精简：

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]          // 累积的完整消息历史
  private abortController: AbortController    // 中断信号
  private permissionDenials: SDKPermissionDenial[] // 权限拒绝记录
  private totalUsage: NonNullableUsage        // 累积 API 用量
  private hasHandledOrphanedPermission = false // 孤儿权限一次性处理标志
  private readFileState: FileStateCache       // 文件读取缓存（去重用）
  private discoveredSkillNames = new Set<string>() // Turn 级技能发现追踪
  private loadedNestedMemoryPaths = new Set<string>() // 已加载的嵌套记忆路径
}
```

### 4.2.2 submitMessage 生命周期

每次用户发送消息时，`submitMessage()` 启动一个新的 Turn：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
```

返回值是一个 `AsyncGenerator`，这意味着调用方可以 `for await` 地逐条消费生成的 SDK 消息。这是流式架构的关键——消息在生成时就被推送出去，而非等到全部完成。

`submitMessage` 的执行流程可分为以下阶段：

```
submitMessage(prompt)
  │
  ├── 1. 析构配置，设置工作目录
  ├── 2. 包装 canUseTool（注入权限拒绝追踪）
  ├── 3. 解析模型配置与 thinking 配置
  ├── 4. 构建 system prompt（default + memory + append）
  ├── 5. 注册结构化输出强制执行
  ├── 6. 处理孤儿权限（仅首次）
  ├── 7. processUserInput（处理 slash 命令、附件等）
  ├── 8. 持久化用户消息到 transcript
  ├── 9. 加载 skills 和 plugins
  ├── 10. yield buildSystemInitMessage（工具、模型、权限模式信息）
  ├── 11. 如果不需要查询（slash 命令已处理），直接返回结果
  ├── 12. 进入 query() 循环
  │      └── for await (message of query(...))
  │           ├── 记录到 mutableMessages
  │           ├── 持久化到 transcript
  │           ├── 回放用户消息确认
  │           └── yield 为 SDK 消息格式
  └── 13. 构造最终 result 消息
```

值得注意的是步骤 8 中的 transcript 持久化策略：

```typescript
// --bare / SIMPLE: fire-and-forget. Scripted calls don't --resume after
// kill-mid-request.
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise  // 不阻塞
  } else {
    await transcriptPromise  // 阻塞等待写入
  }
}
```

在非 bare 模式下，用户消息在进入查询循环**之前**就被写入 transcript。这保证了即使进程在 API 响应到达前被杀死，`--resume` 也能恢复到用户消息提交的时刻。

## 4.3 核心查询循环（queryLoop）

`queryLoop` 是整个系统最复杂的函数，实现了一个带有多种继续条件（Continue）和终止条件（Terminal）的状态机。

### 4.3.1 状态定义

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // 上一次迭代为何继续
}
```

`State` 是一个值对象——每次循环通过 `state = next` 整体替换而非局部修改。`transition` 字段记录了上一次迭代继续的原因，便于测试断言恢复路径是否被触发。

### 4.3.2 循环的完整流程

每次 `while (true)` 迭代执行以下步骤：

```
queryLoop iteration
  │
  ├── 1. 析构 state → 本轮局部变量
  ├── 2. 启动 Skill Discovery Prefetch（后台）
  ├── 3. yield { type: 'stream_request_start' }
  │
  ├── 4. 消息预处理管线
  │     ├── applyToolResultBudget   (工具结果大小预算)
  │     ├── snipCompactIfNeeded     (snip 压缩)
  │     ├── microcompactMessages    (micro 压缩)
  │     ├── applyCollapsesIfNeeded  (context collapse)
  │     └── autoCompactIfNeeded     (auto compact)
  │
  ├── 5. 检查 blocking limit（token 硬上限）
  │
  ├── 6. API 调用（streaming）
  │     ├── deps.callModel(messages, systemPrompt, ...)
  │     ├── 收集 assistantMessages, toolUseBlocks
  │     ├── 流式工具执行（StreamingToolExecutor）
  │     └── 错误拦截（FallbackTriggeredError → 模型切换重试）
  │
  ├── 7. Post-sampling hooks（异步）
  │
  ├── 8. 中断检查（abortController）
  │
  ├── 9. 如果 needsFollowUp == false（无工具调用）
  │     ├── PTL 413 恢复（context collapse drain → reactive compact）
  │     ├── max_output_tokens 恢复（escalate → recovery message）
  │     ├── Stop Hooks 评估
  │     ├── Token Budget 续传判断
  │     └── return Terminal
  │
  ├── 10. 工具执行
  │     ├── StreamingToolExecutor.getRemainingResults()
  │     │   或 runTools()
  │     └── 收集 toolResults
  │
  ├── 11. 附件处理
  │     ├── getAttachmentMessages（文件变更、队列命令）
  │     ├── Memory Prefetch consume
  │     └── Skill Discovery Prefetch consume
  │
  ├── 12. 检查 maxTurns 限制
  │
  └── 13. state = { ...next state }  →  continue
```

### 4.3.3 转换（Transition）类型

查询循环在不同情况下以不同原因继续。从代码中可以提取出所有可能的 `transition.reason`：

| Transition Reason | 触发条件 |
|---|---|
| `next_turn` | 正常的工具调用后继续 |
| `stop_hook_blocking` | Stop Hook 返回 blocking error |
| `max_output_tokens_recovery` | 输出 token 上限恢复（注入续传消息） |
| `max_output_tokens_escalate` | 输出 token 上限升级（从 8k 到 64k） |
| `reactive_compact_retry` | Reactive Compact 后重试 |
| `collapse_drain_retry` | Context Collapse 排空后重试 |
| `token_budget_continuation` | Token 预算未用完，继续执行 |

## 4.4 消息预处理管线

在每次 API 调用之前，消息需要经过一系列压缩和优化步骤。这条管线的设计体现了分层压缩的思想——从最轻量的局部截断到最重量的全量摘要，逐步升级。

### 4.4.1 管线阶段

```
原始消息 → applyToolResultBudget → snipCompact → microcompact
                                                      ↓
                                             context collapse
                                                      ↓
                                               autocompact
                                                      ↓
                                           最终 messagesForQuery
```

**阶段 1：Tool Result Budget**

对聚合的工具结果大小施加每消息预算。运行在 microcompact **之前**，因为 cached microcompact 纯粹按 `tool_use_id` 操作（不检查内容），所以内容替换对它是不可见的。

```typescript
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(...) : undefined,
  new Set(
    toolUseContext.options.tools
      .filter(t => !Number.isFinite(t.maxResultSizeChars))
      .map(t => t.name),
  ),
)
```

注意最后一个参数：`maxResultSizeChars` 为 `Infinity` 的工具被排除在预算之外（如 FileReadTool，因为持久化读取结果会导致循环读取）。

**阶段 2：Snip Compact**

Snip 是一种轻量级的历史裁剪，通过在消息数组中设置边界标记来移除早期消息。它的返回值包含 `tokensFreed`，这个值会传递给后续的 autocompact，使其阈值判断能反映 snip 移除的 token 数量。

```typescript
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
}
```

**阶段 3：Microcompact**

Microcompact 对工具结果执行局部压缩——例如截断过长的 Bash 输出、移除重复的文件内容。它支持缓存编辑模式（`CACHED_MICROCOMPACT`），此时实际的 token 删除数在 API 返回后才能确定。

```typescript
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages
```

**阶段 4：Context Collapse**

Context Collapse 是一种更细粒度的压缩策略——它不是对整个历史做摘要，而是选择性地折叠特定的对话片段。折叠的视图是一个"读时投影"，原始消息保留在 REPL 数组中，summary 存在 collapse store 里。

```typescript
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

设计要点：Context Collapse 运行在 autocompact **之前**。如果 collapse 已经将 token 数降到了阈值以下，autocompact 就不需要执行，从而保留了更细粒度的上下文而非单一摘要。

**阶段 5：Autocompact**

当 token 数超过阈值时，autocompact 将整个历史浓缩为一条摘要消息。这是最"重"的压缩手段：

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages },
  querySource,
  tracking,
  snipTokensFreed,
)
```

`consecutiveFailures` 实现了一个断路器模式——如果 compact 连续失败多次，系统会停止重试。

### 4.4.2 管线设计原则

1. **各阶段不互斥**：snip 和 microcompact 可以同时运行，它们不互斥。
2. **轻量优先**：先尝试代价最低的压缩，最重的全量摘要放在最后。
3. **token 传递**：每个阶段释放的 token 数会传递给后续阶段，避免基于过期计数做出错误判断。
4. **feature gate 隔离**：每个阶段都有独立的 feature gate，可以独立开关。

## 4.5 API 调用与流式响应处理

### 4.5.1 模型调用

模型调用通过 `deps.callModel` 执行，使用 `for await` 消费流式响应：

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    querySource,
    maxOutputTokensOverride,
    taskBudget: params.taskBudget && {
      total: params.taskBudget.total,
      ...(taskBudgetRemaining !== undefined && { remaining: taskBudgetRemaining }),
    },
    // ...更多选项
  },
})) {
  // 处理每个流式消息
}
```

`userContext` 通过 `prependUserContext` 注入到第一条用户消息中，而非添加到 system prompt。这是因为用户上下文（当前目录、平台信息等）在不同 turn 之间可能变化，放在 user message 中可以保持每次查询的上下文准确性。

### 4.5.2 流式消息处理

在流式循环中，消息处理分为几个关键分支：

**Streaming Fallback 处理**：当 API 在流式传输中途触发 fallback 时，需要清理已经生成的部分消息：

```typescript
if (streamingFallbackOccured) {
  // 为孤儿消息发出 tombstone
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false

  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
  }
}
```

Tombstone 是一种"删除标记"——通知 UI 和 transcript 移除先前已发出的部分消息。

**消息扣留（Withholding）**：某些可恢复的错误消息被暂时扣留，等恢复路径执行后再决定是否发出：

```typescript
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, ...)) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

这个设计解决了一个微妙的问题：如果先 yield 了错误消息再尝试恢复，SDK 调用方（如 Desktop 应用）可能在看到 `error` 字段后就终止了会话，而恢复循环还在继续——"nobody is listening"。

## 4.6 流式工具执行（StreamingToolExecutor）

传统的工具执行模式是：等 API 响应**完全结束**后，再开始执行工具。StreamingToolExecutor 改变了这一模式——它在 API 流式传输**过程中**就开始执行已完成参数的工具。

```typescript
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

工作流程：

1. 当流中出现 `tool_use` block 时，立即提交给 executor：
```typescript
if (streamingToolExecutor) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

2. 在流式循环中轮询已完成的结果：
```typescript
if (streamingToolExecutor) {
  for (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message
      toolResults.push(...)
    }
  }
}
```

3. 流结束后，消费剩余结果：
```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

这种设计显著减少了端到端延迟：模型输出第一个工具调用到结果返回之间的等待时间，从"流完成+工具执行"缩短为只有"工具执行"（因为执行在流传输期间已并行开始）。

## 4.7 停止条件评估

查询循环的终止由多层条件共同决定。

### 4.7.1 Stop Hooks

`handleStopHooks` 是一个 `AsyncGenerator`，它执行用户配置的 Stop Hook（settings.json 中定义的 shell 命令），并根据结果决定是否允许模型继续：

```typescript
const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
  stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  // 将 blocking error 作为用户消息注入，触发下一次循环
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...stopHookResult.blockingErrors],
    ...
    stopHookActive: true,
    transition: { reason: 'stop_hook_blocking' },
  }
  continue
}
```

Stop Hook 有两种结果：
- **preventContinuation**：完全停止循环（例如 hook 明确指示"不要继续"）
- **blockingError**：将错误注入消息流，让模型看到并修正（例如 linter 检查失败）

### 4.7.2 Token Budget 续传

当 `TOKEN_BUDGET` feature 开启时，系统会追踪 turn 级别的 token 消耗，并在预算未用完时自动续传：

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // 子 Agent 不做续传
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 边际收益递减检测：连续3次续传且每次增量不足500 token
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 未达到90%预算且没有边际递减 → 继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    return { action: 'continue', nudgeMessage: ..., ... }
  }

  return { action: 'stop', ... }
}
```

关键设计：
- **90% 阈值**（`COMPLETION_THRESHOLD = 0.9`）：达到预算的 90% 后停止续传。
- **边际收益递减保护**：如果连续 3 次续传每次增量不足 500 token，说明模型已经"无话可说"，此时即使预算未满也应停止。
- **子 Agent 不续传**：避免子 Agent 无限运行。

### 4.7.3 最大轮次限制

```typescript
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

## 4.8 错误恢复策略

查询循环实现了多层错误恢复机制，每种错误有独立的恢复路径。

### 4.8.1 Prompt Too Long (413) 恢复

当上下文超过模型限制时，API 返回 413 错误。恢复策略分两步：

**第一步：Context Collapse Drain**

如果有已暂存的 collapse（折叠），先尝试排空它们：

```typescript
if (feature('CONTEXT_COLLAPSE') && contextCollapse &&
    state.transition?.reason !== 'collapse_drain_retry') {
  const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
  if (drained.committed > 0) {
    state = { ..., transition: { reason: 'collapse_drain_retry' } }
    continue
  }
}
```

注意 `state.transition?.reason !== 'collapse_drain_retry'` 这个守卫——如果上一次已经尝试过 drain 且仍然 413，就不再重试，而是降级到 reactive compact。

**第二步：Reactive Compact**

如果 drain 不够，执行完整的 reactive compact（即紧急压缩）：

```typescript
const compacted = await reactiveCompact.tryReactiveCompact({
  hasAttempted: hasAttemptedReactiveCompact,
  querySource,
  aborted: toolUseContext.abortController.signal.aborted,
  messages: messagesForQuery,
  cacheSafeParams: { ... },
})

if (compacted) {
  state = { ..., hasAttemptedReactiveCompact: true,
             transition: { reason: 'reactive_compact_retry' } }
  continue
}

// 两步都失败 → 向用户展示错误
yield lastMessage
return { reason: 'prompt_too_long' }
```

`hasAttemptedReactiveCompact` 标志确保 reactive compact 只尝试一次——避免 compact → 仍超限 → 再 compact 的死循环。

### 4.8.2 Max Output Tokens (OTK) 恢复

当模型输出达到 token 上限时，恢复分两阶段：

**阶段 1：Escalate（升级）**

如果当前使用的是被限制的 8k 默认值，直接升级到 64k 重试，不注入任何恢复消息：

```typescript
if (capEnabled && maxOutputTokensOverride === undefined) {
  state = { ...,
    maxOutputTokensOverride: ESCALATED_MAX_TOKENS,  // 64k
    transition: { reason: 'max_output_tokens_escalate' },
  }
  continue
}
```

**阶段 2：Multi-turn Recovery（多轮恢复）**

如果 64k 也不够，注入恢复消息要求模型继续，最多重试 3 次（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`）：

```typescript
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap...`,
    isMeta: true,
  })
  state = { ...,
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: ... },
  }
  continue
}
```

恢复消息的措辞经过精心设计——"no apology, no recap"——因为 LLM 的自然倾向是在续传时先道歉再复述之前的内容，这会浪费宝贵的 token。

### 4.8.3 Fallback 模型切换

当主模型不可用时（例如高需求导致限流），API 层会抛出 `FallbackTriggeredError`，查询循环捕获后切换到 fallback 模型：

```typescript
} catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true

    // 清理已生成的部分消息
    yield* yieldMissingToolResultBlocks(assistantMessages, 'Model fallback triggered')
    assistantMessages.length = 0

    // 如果是 Ant 用户，剥离 thinking signature（因为它们是模型绑定的）
    if (process.env.USER_TYPE === 'ant') {
      messagesForQuery = stripSignatureBlocks(messagesForQuery)
    }

    yield createSystemMessage(
      `Switched to ${renderModelName(fallbackModel)} due to high demand...`,
      'warning',
    )
    continue
  }
  throw innerError
}
```

Thinking signature 的处理是一个重要细节：受保护的 thinking block 包含模型特定的密码学签名，将 Capybara 模型的签名发送给 Opus 模型会导致 400 错误。

## 4.9 依赖注入模式（query/deps.ts）

`QueryDeps` 将查询循环的外部依赖封装为一个可替换的接口：

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

这个模式的价值在于测试可替换性。代码注释中说得很直白：

> Passing a `deps` override into QueryParams lets tests inject fakes directly instead of spyOn-per-module — the most common mocks (callModel, autocompact) are each spied in 6-8 test files today with module-import-and-spy boilerplate.

使用 `typeof fn` 而非手写签名，保证了接口类型与实际实现自动同步。

## 4.10 查询配置快照（query/config.ts）

`QueryConfig` 在 `query()` 入口处被快照一次，在整个循环内保持不变：

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

设计要点：
- **不包含 `feature()` gate**：`feature()` 是编译时 tree-shaking 的边界，必须在使用点内联调用才能被正确消除。配置快照中只包含运行时 gate。
- **`CACHED_MAY_BE_STALE` 合约**：Statsig 的这些值本身就允许一定的陈旧性，所以每次 `query()` 调用快照一次并不违反其语义合约。

## 4.11 本章小结

查询引擎的设计揭示了构建生产级 AI Agent 对话系统的核心挑战：

1. **状态机复杂性**：一个看似简单的"调用 API → 执行工具 → 继续"循环，在加入错误恢复、上下文管理、权限控制后，状态转换图急剧膨胀。用 `State` 值对象 + `transition` 标记来管理这种复杂性是一个务实的选择。

2. **分层压缩策略**：从 tool result budget 到 snip、microcompact、context collapse 再到 autocompact，五层递进式压缩确保了在不同场景下都能找到最合适的压缩力度。

3. **流式优先架构**：从 `AsyncGenerator` 的返回值类型到 StreamingToolExecutor 的并行执行，整个设计都围绕"尽早产出、尽早处理"的流式理念构建。

4. **容错即常态**：413 恢复、OTK 恢复、模型 Fallback、streaming fallback tombstone——这些不是"异常处理"，而是系统正常运行的一部分。在 AI 应用中，不确定性是常态而非例外。

5. **可测试性设计**：依赖注入（QueryDeps）、配置快照（QueryConfig）、值对象状态（State）——这些模式共同使得一个复杂的有状态循环可以在单元测试中被可靠地验证。

