---
title: "第九章：状态管理"
weight: 90
bookCollapseSection: false
---


状态管理是任何复杂应用的基石。Claude Code 作为一个同时涉及终端 UI 渲染、后台任务管理、多 Agent 协调和远程会话同步的系统，其状态管理设计尤为精巧。本章将深入分析 Claude Code 的双层状态架构——UI 驱动的 AppState 和全局单例的 Bootstrap State，以及围绕它们构建的 React Context 层次、Selector 模式和状态持久化策略。

## 9.1 双层状态架构

Claude Code 的状态管理分为两个独立的层次：

```
┌─────────────────────────────────────────────────┐
│            React Component Tree                  │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │          AppState (UI 驱动)                 │ │
│  │  Store<AppState> + React Context            │ │
│  │  - settings, tools, mcp, plugins            │ │
│  │  - tasks, inbox, notifications              │ │
│  │  - toolPermissionContext, thinkingEnabled    │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
├─────────────────────────────────────────────────┤
│            Bootstrap State (全局单例)             │
│  模块级变量 + getter/setter 函数                  │
│  - sessionId, cwd, modelUsage                    │
│  - registeredHooks, invokedSkills                │
│  - totalCostUSD, telemetry counters              │
│  - systemPromptSectionCache                      │
└─────────────────────────────────────────────────┘
```

**AppState** 是 React 生态内的响应式状态，通过 `createStore` 创建的 Store 驱动 UI 更新。它包含所有与用户界面、交互流程直接相关的状态。

**Bootstrap State** 是进程级的全局单例状态，以模块顶层变量的形式存在，通过导出的 getter/setter 函数访问。它包含会话 ID、成本追踪、遥测计数器等不需要触发 UI 重渲染的数据。

这种分离的核心理由是**性能**——不是所有状态变化都需要触发 React 重渲染。`totalCostUSD` 每次 API 调用都会更新，但只在特定时刻（如状态栏刷新）才需要读取；`modelUsage` 在 API 请求的 hot path 上累加，不应该触发整个 UI 树的协调。

## 9.2 AppState设计

### 9.2.1 DeepImmutable约束

AppState 的类型定义使用了 `DeepImmutable` 约束，确保状态对象在整个生命周期中保持不可变：

```typescript
// state/AppStateStore.ts
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  toolPermissionContext: ToolPermissionContext
  agent: string | undefined
  kairosEnabled: boolean
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  // ... 更多字段
}> & {
  // 以下字段排除在 DeepImmutable 之外，因为包含函数类型或 Map/Set
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { ... }
    needsRefresh: boolean
  }
  // ...
}
```

`DeepImmutable<T>` 将类型 T 的所有属性递归标记为 `readonly`。但注意 `tasks`、`mcp`、`plugins` 等字段被排除在 `DeepImmutable` 之外——这是因为它们包含函数类型（`TaskState` 中有回调）、`Map`/`Set` 等无法用 `readonly` 约束的类型。

### 9.2.2 关键字段详解

AppState 包含数十个字段，按功能可分为几大类：

**基础设置类**

```typescript
settings: SettingsJson            // 合并后的设置
verbose: boolean                  // 详细模式
mainLoopModel: ModelSetting       // 当前模型（alias 或 full name）
mainLoopModelForSession: ModelSetting // 本会话模型
```

**权限与安全类**

```typescript
toolPermissionContext: ToolPermissionContext  // 权限模式和规则
denialTracking?: DenialTrackingState         // 拒绝追踪（用于 auto mode 降级）
```

**MCP与插件类**

```typescript
mcp: {
  clients: MCPServerConnection[]    // 所有 MCP 连接
  tools: Tool[]                     // MCP 提供的工具
  commands: Command[]               // MCP 提供的命令
  resources: Record<string, ServerResource[]>  // MCP 资源
  pluginReconnectKey: number        // 插件重连触发器
}
plugins: {
  enabled: LoadedPlugin[]           // 已启用插件
  disabled: LoadedPlugin[]          // 已禁用插件
  errors: PluginError[]             // 插件错误
}
```

**任务与Agent类**

```typescript
tasks: { [taskId: string]: TaskState }      // 所有后台任务
agentNameRegistry: Map<string, AgentId>      // Agent 名称注册表
foregroundedTaskId?: string                   // 前台任务 ID
viewingAgentTaskId?: string                   // 正在查看的 Agent
teamContext?: {
  teamName: string
  teammates: { [id: string]: { name: string; ... } }
}
inbox: {
  messages: Array<{
    id: string; from: string; text: string;
    status: 'pending' | 'processing' | 'processed'
  }>
}
```

**推测执行与提示建议类**

```typescript
speculation: SpeculationState       // 推测执行状态
speculationSessionTimeSavedMs: number  // 推测节省的时间
promptSuggestion: {
  text: string | null
  promptId: 'user_intent' | 'stated_intent' | null
  shownAt: number
  acceptedAt: number
}
```

**远程与Bridge类**

```typescript
replBridgeEnabled: boolean          // Always-on bridge 开关
replBridgeConnected: boolean        // Bridge 已就绪
replBridgeSessionActive: boolean    // Bridge 会话活跃
replBridgeConnectUrl: string | undefined  // Bridge 连接 URL
```

### 9.2.3 默认状态

`getDefaultAppState` 函数提供了完整的初始状态：

```typescript
export function getDefaultAppState(): AppState {
  const initialMode: PermissionMode =
    teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
      ? 'plan'
      : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    mcp: {
      clients: [],
      tools: [],
      commands: [],
      resources: {},
      pluginReconnectKey: 0,
    },
    plugins: {
      enabled: [],
      disabled: [],
      commands: [],
      errors: [],
      installationStatus: { marketplaces: [], plugins: [] },
      needsRefresh: false,
    },
    thinkingEnabled: shouldEnableThinkingByDefault(),
    promptSuggestionEnabled: shouldEnablePromptSuggestion(),
    speculation: { status: 'idle' },
    speculationSessionTimeSavedMs: 0,
    notifications: { current: null, queue: [] },
    inbox: { messages: [] },
    activeOverlays: new Set<string>(),
    // ... 更多默认值
  }
}
```

注意初始权限模式的计算：如果当前进程是一个 teammate（团队成员 Agent）且要求 plan mode，则初始模式为 `'plan'`，否则为 `'default'`。

## 9.3 Store实现

Claude Code 的状态 Store 实现极简，仅 35 行代码：

```typescript
// state/store.ts
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等则跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

这个设计的几个关键特性：

1. **Updater模式**：`setState` 接受一个 `(prev: T) => T` 函数而非直接的新状态值。这确保了基于最新状态的原子更新，避免了并发写入时的数据丢失。

2. **引用相等检查**：使用 `Object.is(next, prev)` 判断状态是否真正变化。如果 updater 返回了同一个引用，则跳过所有通知——这是一个重要的性能优化。

3. **onChange回调**：在通知 listener 之前先调用 `onChange`，用于执行副作用（如持久化、同步到 CCR 等）。

4. **订阅/取消订阅**：`subscribe` 返回一个取消订阅函数，符合 React `useSyncExternalStore` 的接口要求。

## 9.4 状态变更监听

`onChangeAppState` 函数作为 Store 的 `onChange` 回调，负责在状态变化时执行副作用：

```typescript
// state/onChangeAppState.ts
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // 1. 权限模式变更 -> 同步到 CCR 和 SDK
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    const prevExternal = toExternalPermissionMode(prevMode)
    const newExternal = toExternalPermissionMode(newMode)
    if (prevExternal !== newExternal) {
      notifySessionMetadataChanged({
        permission_mode: newExternal,
        is_ultraplan_mode: isUltraplan ? true : null,
      })
    }
    notifyPermissionModeChanged(newMode)
  }

  // 2. 模型变更 -> 更新设置
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    if (newState.mainLoopModel === null) {
      updateSettingsForSource('userSettings', { model: undefined })
      setMainLoopModelOverride(null)
    } else {
      updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
      setMainLoopModelOverride(newState.mainLoopModel)
    }
  }

  // 3. 视图展开状态 -> 持久化到 globalConfig
  if (newState.expandedView !== oldState.expandedView) {
    saveGlobalConfig(current => ({
      ...current,
      showExpandedTodos: newState.expandedView === 'tasks',
      showSpinnerTree: newState.expandedView === 'teammates',
    }))
  }

  // 4. verbose -> 持久化
  if (newState.verbose !== oldState.verbose) {
    saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
  }

  // 5. settings 变更 -> 清除认证缓存、重新应用环境变量
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()
    }
  }
}
```

这个函数的源码注释中特别提到了一个重要的设计修复：在引入集中化的 `onChangeAppState` 之前，权限模式变更需要在 8+ 个不同的代码路径中手动同步到 CCR（Claude Code Runtime），导致许多路径遗漏了同步。通过在 Store 的 `onChange` 回调中统一处理，所有通过 `setState` 修改权限模式的代码路径都会自动触发同步。

## 9.5 Bootstrap State

Bootstrap State 位于 `bootstrap/state.ts`，是一个进程级的全局单例。它的设计哲学与 AppState 完全不同——不使用 React 响应式系统，而是直接通过模块级变量和 getter/setter 函数管理。

### 9.5.1 状态结构

Bootstrap State 的类型定义非常庞大，包含约 100 个字段：

```typescript
// bootstrap/state.ts
type State = {
  // 路径信息
  originalCwd: string           // 原始工作目录
  projectRoot: string           // 项目根目录（启动时确定，不随 cd 变化）
  cwd: string                   // 当前工作目录

  // 成本追踪
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number

  // 会话信息
  sessionId: SessionId
  parentSessionId: SessionId | undefined
  startTime: number
  lastInteractionTime: number

  // 模型使用统计
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting

  // 遥测
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  tokenCounter: AttributedCounter | null

  // 技能保存
  invokedSkills: Map<string, {
    skillName: string
    skillPath: string
    content: string
    invokedAt: number
    agentId: string | null
  }>

  // Hooks注册
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null

  // 缓存与延迟评估
  systemPromptSectionCache: Map<string, string | null>
  lastEmittedDate: string | null
  promptCache1hEligible: boolean | null

  // 安全 latches（粘性标志）
  afkModeHeaderLatched: boolean | null
  fastModeHeaderLatched: boolean | null
  cacheEditingHeaderLatched: boolean | null
  thinkingClearLatched: boolean | null

  // ... 更多字段
}
```

### 9.5.2 初始化

状态初始化在模块加载时执行，确保 `cwd` 经过了 symlink 解析和 NFC 规范化：

```typescript
function getInitialState(): State {
  let resolvedCwd = ''
  if (typeof process !== 'undefined' && typeof process.cwd === 'function') {
    const rawCwd = cwd()
    try {
      resolvedCwd = realpathSync(rawCwd).normalize('NFC')
    } catch {
      resolvedCwd = rawCwd.normalize('NFC')
    }
  }

  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    totalCostUSD: 0,
    sessionId: randomUUID() as SessionId,
    startTime: Date.now(),
    invokedSkills: new Map(),
    registeredHooks: null,
    systemPromptSectionCache: new Map(),
    // ... 更多初始值
  }
}
```

### 9.5.3 访问模式

Bootstrap State 通过导出的函数访问，而非直接暴露变量：

```typescript
// 典型的 getter 函数
export function getSessionId(): SessionId {
  return state.sessionId
}

export function getOriginalCwd(): string {
  return state.originalCwd
}

export function getProjectRoot(): string {
  return state.projectRoot
}

// 典型的 setter 函数
export function setMainLoopModelOverride(model: ModelSetting): void {
  state.mainLoopModelOverride = model
}

// 累加器
export function addTotalCost(cost: number): void {
  state.totalCostUSD += cost
}
```

源码中有一条醒目的注释："DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE"（不要在这里添加更多状态——对全局状态要保持审慎）。这反映了团队对全局状态膨胀的警惕。

### 9.5.4 技能保存

`invokedSkills` 是一个特殊的缓存，用于在上下文压缩（compaction）后保留已调用技能的内容：

```typescript
invokedSkills: Map<
  string,  // key = `${agentId ?? ''}:${skillName}`
  {
    skillName: string
    skillPath: string
    content: string      // 技能的完整内容
    invokedAt: number
    agentId: string | null
  }
>
```

键值使用 `agentId:skillName` 的复合格式，防止不同 Agent 之间的技能覆盖。在上下文压缩时，技能内容会从消息历史中被移除以节省 token，但通过 `invokedSkills` 缓存，系统可以在需要时重新注入。

### 9.5.5 粘性标志（Latches）

Bootstrap State 中有多个 "latch" 标志，它们遵循"一旦开启就不关闭"的语义：

```typescript
// 一旦 AFK mode 被激活，就持续发送 beta header
afkModeHeaderLatched: boolean | null

// 一旦 fast mode 被启用，就持续发送 header
fastModeHeaderLatched: boolean | null

// 一旦缓存编辑 beta 被启用，就持续发送 header
cacheEditingHeaderLatched: boolean | null
```

这些 latch 的目的是保护 prompt cache。Claude API 的 prompt cache 对请求头敏感——如果用户在会话中间切换了 fast mode 的开关，beta header 的变化会导致 50-70K token 的 prompt cache 失效。通过粘性标志，一旦首次启用就永远保持开启，避免了中途切换造成的缓存击穿。

## 9.6 React Context层次

Claude Code 使用了丰富的 React Context 来为组件树提供各类上下文数据。

### 9.6.1 AppStateProvider

`AppStateProvider` 是最顶层的 Context Provider，负责创建 Store 并将其注入组件树：

```typescript
// state/AppState.tsx
export const AppStoreContext = React.createContext<AppStateStore | null>(null)

export function AppStateProvider({ children, initialState, onChangeAppState }: Props) {
  // 防止嵌套
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error("AppStateProvider can not be nested within another AppStateProvider")
  }

  // 创建 Store（只在首次渲染时执行）
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )

  // 监听设置变化并应用
  const onSettingsChange = useEffectEvent(
    source => applySettingsChange(source, store.setState)
  )
  useSettingsChange(onSettingsChange)

  return (
    <AppStoreContext.Provider value={store}>
      <HasAppStateContext.Provider value={true}>
        <MailboxProvider>
          <VoiceProvider>
            {children}
          </VoiceProvider>
        </MailboxProvider>
      </HasAppStateContext.Provider>
    </AppStoreContext.Provider>
  )
}
```

注意 Provider 的嵌套顺序：`AppStoreContext` > `HasAppStateContext` > `MailboxProvider` > `VoiceProvider`。`HasAppStateContext` 用于检测错误的嵌套使用。

消费 AppState 的 hooks：

```typescript
// 获取 Store 实例
export function useAppStateStore(): AppStateStore {
  const store = useContext(AppStoreContext)
  if (!store) throw new Error('useAppStateStore must be used within AppStateProvider')
  return store
}

// 获取当前状态（订阅更新）
export function useAppState(): AppState {
  const store = useAppStateStore()
  return useSyncExternalStore(store.subscribe, store.getState)
}

// 获取 setState 函数
export function useSetAppState(): (updater: (prev: AppState) => AppState) => void {
  const store = useAppStateStore()
  return store.setState
}
```

`useAppState` 使用了 React 18 的 `useSyncExternalStore`，这确保了从外部 Store 读取状态时的一致性——在并发渲染模式下不会出现 "tearing"（撕裂）问题。

### 9.6.2 MailboxProvider

`MailboxProvider` 提供了进程间消息投递能力，用于 Agent 间通信：

```typescript
// context/mailbox.tsx
const MailboxContext = createContext<Mailbox | undefined>(undefined)

export function MailboxProvider({ children }: Props) {
  const mailbox = useMemo(() => new Mailbox(), [])
  return (
    <MailboxContext.Provider value={mailbox}>
      {children}
    </MailboxContext.Provider>
  )
}

export function useMailbox(): Mailbox {
  const mailbox = useContext(MailboxContext)
  if (!mailbox) {
    throw new Error('useMailbox must be used within a MailboxProvider')
  }
  return mailbox
}
```

`Mailbox` 实例在整个应用生命周期中只创建一次（`useMemo([], [])`），确保所有组件共享同一个消息队列。

### 9.6.3 ModalContext

`ModalContext` 为模态对话框提供了尺寸和滚动信息：

```typescript
// context/modalContext.tsx
type ModalCtx = {
  rows: number              // 模态框可用行数
  columns: number           // 模态框可用列数
  scrollRef: RefObject<ScrollBoxHandle | null> | null
}

export const ModalContext = createContext<ModalCtx | null>(null)

// 判断当前组件是否在模态框内
export function useIsInsideModal(): boolean {
  return useContext(ModalContext) !== null
}

// 获取可用尺寸（模态框内用模态框尺寸，否则用终端尺寸）
export function useModalOrTerminalSize(fallback: {
  rows: number; columns: number
}): { rows: number; columns: number } {
  const ctx = useContext(ModalContext)
  return ctx ? { rows: ctx.rows, columns: ctx.columns } : fallback
}
```

这个 Context 解决了一个关键问题：模态框的内部区域小于整个终端，组件需要知道自己的可用空间来正确计算分页和滚动。

### 9.6.4 QueuedMessageContext

`QueuedMessageContext` 为排队消息提供布局上下文：

```typescript
// context/QueuedMessageContext.tsx
type QueuedMessageContextValue = {
  isQueued: boolean
  isFirst: boolean
  paddingWidth: number     // 容器 padding（如 4 = paddingX * 2）
}

const PADDING_X = 2

export function QueuedMessageProvider({ isFirst, useBriefLayout, children }: Props) {
  const padding = useBriefLayout ? 0 : PADDING_X
  const value = useMemo(
    () => ({ isQueued: true, isFirst, paddingWidth: padding * 2 }),
    [isFirst, padding],
  )

  return (
    <QueuedMessageContext.Provider value={value}>
      <Box paddingX={padding}>{children}</Box>
    </QueuedMessageContext.Provider>
  )
}
```

### 9.6.5 NotificationContext

通知系统是一个功能完整的优先级队列：

```typescript
// context/notifications.tsx
type Priority = 'low' | 'medium' | 'high' | 'immediate'

type BaseNotification = {
  key: string
  invalidates?: string[]       // 使其他通知失效
  priority: Priority
  timeoutMs?: number
  fold?: (accumulator: Notification, incoming: Notification) => Notification
}

const DEFAULT_TIMEOUT_MS = 8000

export function useNotifications(): {
  addNotification: AddNotificationFn
  removeNotification: RemoveNotificationFn
} {
  const store = useAppStateStore()
  const setAppState = useSetAppState()

  const processQueue = useCallback(() => {
    setAppState(prev => {
      const next = getNext(prev.notifications.queue)
      if (prev.notifications.current !== null || !next) return prev

      // 设置超时自动清除
      currentTimeoutId = setTimeout(() => {
        setAppState(prev => ({
          ...prev,
          notifications: { queue: prev.notifications.queue, current: null }
        }))
        processQueue()
      }, next.timeoutMs ?? DEFAULT_TIMEOUT_MS)

      return {
        ...prev,
        notifications: {
          queue: prev.notifications.queue.filter(_ => _ !== next),
          current: next
        }
      }
    })
  }, [setAppState])

  const addNotification = useCallback((notif: Notification) => {
    if (notif.priority === 'immediate') {
      // 立即显示，将当前通知放回队列
      if (currentTimeoutId) clearTimeout(currentTimeoutId)
      setAppState(prev => ({
        ...prev,
        notifications: {
          current: notif,
          queue: [...(prev.notifications.current ? [prev.notifications.current] : []),
                  ...prev.notifications.queue]
            .filter(_ => _.priority !== 'immediate' && !notif.invalidates?.includes(_.key))
        }
      }))
      return
    }

    // 非即时通知：检查 fold（合并）逻辑
    setAppState(prev => {
      if (notif.fold && prev.notifications.current?.key === notif.key) {
        const folded = notif.fold(prev.notifications.current, notif)
        return { ...prev, notifications: { current: folded, queue: prev.notifications.queue } }
      }
      // 加入队列
      return { ...prev, notifications: { ...prev.notifications, queue: [...prev.notifications.queue, notif] } }
    })
    processQueue()
  }, [setAppState, processQueue])

  return { addNotification, removeNotification }
}
```

通知系统的高级特性包括：

- **优先级队列**：`immediate` 优先级可以抢占当前通知
- **通知合并**（fold）：相同 key 的通知可以通过 `fold` 函数合并，类似 `Array.reduce`
- **失效机制**（invalidates）：新通知可以声明它使哪些旧通知失效
- **自动超时**：每个通知默认 8 秒后自动消失

## 9.7 Selector模式

Claude Code 使用 Selector 模式从 AppState 中派生计算状态：

```typescript
// state/selectors.ts

/**
 * 获取当前正在查看的 teammate 任务
 */
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined {
  const { viewingAgentTaskId, tasks } = appState
  if (!viewingAgentTaskId) return undefined

  const task = tasks[viewingAgentTaskId]
  if (!task) return undefined
  if (!isInProcessTeammateTask(task)) return undefined

  return task
}

/**
 * 确定用户输入应路由到哪个 Agent
 */
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) {
    return { type: 'viewed', task: viewedTask }
  }

  const { viewingAgentTaskId, tasks } = appState
  if (viewingAgentTaskId) {
    const task = tasks[viewingAgentTaskId]
    if (task?.type === 'local_agent') {
      return { type: 'named_agent', task }
    }
  }

  return { type: 'leader' }
}
```

Selector 的设计原则：

1. **纯函数**：只做数据提取，不产生副作用
2. **窄类型参数**：使用 `Pick<AppState, ...>` 而非完整的 `AppState`，明确声明依赖
3. **Discriminated Union 返回值**：`ActiveAgentForInput` 是一个可判别联合类型，调用者可以通过 `type` 字段安全地分支

## 9.8 状态持久化策略

Claude Code 的状态持久化采用了分层策略：

### 9.8.1 即时持久化

某些状态变化会立即写入磁盘，在 `onChangeAppState` 中触发：

```typescript
// expandedView -> globalConfig
if (newState.expandedView !== oldState.expandedView) {
  saveGlobalConfig(current => ({
    ...current,
    showExpandedTodos: newState.expandedView === 'tasks',
    showSpinnerTree: newState.expandedView === 'teammates',
  }))
}

// verbose -> globalConfig
if (newState.verbose !== oldState.verbose) {
  saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
}
```

### 9.8.2 会话级持久化

会话相关数据在会话文件中持久化。Bootstrap State 中的 `sessionId` 标识会话，`originalCwd` 和 `projectRoot` 确定会话存储路径。

### 9.8.3 不持久化

大部分 AppState 字段不会持久化——它们是会话运行时状态（如 `notifications`、`speculation`、`activeOverlays`），在进程结束时自然消失。Bootstrap State 中的 telemetry 计数器会在会话结束时发送到后端，但不写入本地磁盘。

### 9.8.4 CCR同步

在远程模式下，关键状态变化会通过 CCR（Claude Code Runtime）同步到 Web UI：

```typescript
// 权限模式变化 -> 通知 CCR
notifySessionMetadataChanged({
  permission_mode: newExternal,
  is_ultraplan_mode: isUltraplan,
})
```

`notifySessionMetadataChanged` 函数通过 WebSocket 或 HTTP 将状态变化推送到 CCR 服务器，Web UI 可以实时反映 CLI 的状态。

## 9.9 状态更新的原子性保证

Claude Code 通过 updater 函数模式确保状态更新的原子性：

```typescript
// 正确：基于最新状态更新
setAppState(prev => ({
  ...prev,
  notifications: {
    current: notif,
    queue: prev.notifications.queue.filter(...)
  }
}))

// 错误：读取可能过时的快照
const state = store.getState()
store.setState(_ => ({ ...state, notifications: { ... } }))
```

第一种模式中，`prev` 参数保证是调用时的最新状态。在并发场景（如多个 MCP 服务器同时更新连接状态）下，这确保了每次更新都基于最新的基线。

## 9.10 本章小结

Claude Code 的状态管理系统体现了以下核心设计思想：

1. **关注点分离**：AppState 服务于 UI 渲染，Bootstrap State 服务于后台逻辑，两者互不干扰。
2. **极简 Store**：35 行代码实现了完整的 Store，`Object.is` 短路和 updater 模式提供了性能和正确性保证。
3. **集中化副作用**：`onChangeAppState` 作为唯一的状态变更监听点，解决了分散同步导致的遗漏问题。
4. **Context层次化**：从 AppState 到 Mailbox 到 Modal 到 Notification，每层 Context 服务于特定的功能域。
5. **粘性标志防缓存击穿**：Bootstrap State 中的 latch 标志是一个巧妙的性能优化，利用"只升不降"的语义保护 prompt cache。
6. **类型驱动的安全**：`DeepImmutable` 约束、Discriminated Union selector、窄类型参数，通过类型系统防止错误。

