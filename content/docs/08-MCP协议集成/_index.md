---
title: "第八章：MCP协议集成"
weight: 80
bookCollapseSection: false
---


Model Context Protocol（MCP）是 Claude Code 中最重要的扩展机制之一。它定义了一套标准化的协议，允许外部工具服务器以统一的方式向 AI Agent 暴露工具、资源和命令。本章将深入分析 Claude Code 中 MCP 协议的完整实现，涵盖客户端连接管理、多传输层适配、配置系统、认证机制、工具包装以及进程内通信等核心模块。

## 8.1 MCP协议简介与在Claude Code中的角色

MCP（Model Context Protocol）是 Anthropic 推出的开放协议，旨在标准化 AI 模型与外部工具、数据源之间的通信。在 Claude Code 中，MCP 扮演着"工具扩展总线"的角色——任何遵循 MCP 协议的服务器都可以无缝接入 Claude Code，将自身的能力注册为 Claude 可调用的工具。

Claude Code 的 MCP 子系统位于 `services/mcp/` 目录下，包含超过 25 个源文件，总代码量超过 40 万字符。其核心架构可以概括为以下层次：

```
┌─────────────────────────────────────────────┐
│                Model Layer                   │
│  (Claude 模型通过 tool_use 调用 MCP 工具)    │
├─────────────────────────────────────────────┤
│           Tool Wrapper Layer                 │
│  MCPTool / ListMcpResourcesTool /            │
│  ReadMcpResourceTool                         │
├─────────────────────────────────────────────┤
│              Client Layer                    │
│  connectToServer / fetchToolsForClient /     │
│  fetchResourcesForClient / ensureConnectedClient│
├─────────────────────────────────────────────┤
│            Transport Layer                   │
│  STDIO / SSE / HTTP / WebSocket / SDK /      │
│  InProcess / ClaudeAI-Proxy                  │
├─────────────────────────────────────────────┤
│           Config & Auth Layer               │
│  config.ts / auth.ts / types.ts              │
└─────────────────────────────────────────────┘
```

MCP 在 Claude Code 中承担三大职责：

1. **工具扩展**：将外部 MCP 服务器的工具注册为 Claude 可调用的 Tool，命名规则为 `mcp__<serverName>__<toolName>`。
2. **资源暴露**：让 Claude 能够列出和读取 MCP 服务器提供的资源（如文件、数据库记录等）。
3. **命令注入**：MCP 服务器可以注册 Prompt（命令），作为用户可用的 slash command。

## 8.2 MCP客户端实现

MCP 客户端的核心实现位于 `services/mcp/client.ts`，这是整个 MCP 子系统中最大的文件（约 119KB），负责连接管理、工具获取、资源获取以及工具调用等核心功能。

### 8.2.1 连接管理与Memoization

连接管理是 MCP 客户端最关键的能力。`connectToServer` 函数负责建立到 MCP 服务器的连接，其使用 lodash 的 `memoize` 进行缓存，确保同一服务器不会重复连接：

```typescript
// services/mcp/client.ts
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; stdioCount: number; ... },
  ): Promise<MCPServerConnection> => {
    const connectStartTime = Date.now()
    try {
      let transport

      if (serverRef.type === 'sse') {
        // 创建 SSE 传输层...
        const authProvider = new ClaudeAuthProvider(name, serverRef)
        const combinedHeaders = await getMcpServerHeaders(name, serverRef)
        transport = new SSEClientTransport(
          new URL(serverRef.url),
          transportOptions,
        )
      } else if (serverRef.type === 'http') {
        // 创建 Streamable HTTP 传输层...
        transport = new StreamableHTTPClientTransport(
          new URL(serverRef.url),
          transportOptions,
        )
      } else if (serverRef.type === 'ws') {
        // 创建 WebSocket 传输层...
        transport = new WebSocketTransport(wsClient)
      } else if (serverRef.type === 'sdk') {
        // SDK 控制传输层
        transport = new SdkControlClientTransport(name, sendMcpMessage)
      } else {
        // 默认：STDIO 传输层
        transport = new StdioClientTransport({
          command: serverRef.command,
          args: serverRef.args,
          env: { ...subprocessEnv(), ...serverRef.env },
        })
      }

      // 使用 MCP SDK 客户端建立连接
      const client = new Client(
        { name: 'claude-code', version: '1.0.0' },
        { capabilities: { roots: { listChanged: true } } },
      )
      await client.connect(transport)

      return {
        name,
        type: 'connected',
        client,
        capabilities: client.getServerCapabilities(),
        config: serverRef,
        cleanup: async () => { await client.close() },
      }
    } catch (error) {
      return { name, type: 'failed', config: serverRef, error: errorMessage(error) }
    }
  },
  // memoize resolver：基于服务器名和配置生成 cache key
  (name, serverRef) => getServerCacheKey(name, serverRef),
)
```

`memoize` 的 resolver 函数使用 `getServerCacheKey`，通过序列化服务器名称和配置对象来生成唯一的缓存键：

```typescript
export function getServerCacheKey(
  name: string,
  serverRef: ScopedMcpServerConfig,
): string {
  return `${name}-${jsonStringify(serverRef)}`
}
```

`ensureConnectedClient` 函数提供了一个安全的连接保障机制。对于已连接的客户端，它会通过 `connectToServer` 的 memoize 缓存直接返回（命中缓存时是 no-op）；但如果服务器的 `onclose` 事件已触发，memoize 缓存会被清除，此时会建立新连接：

```typescript
export async function ensureConnectedClient(
  client: ConnectedMCPServer,
): Promise<ConnectedMCPServer> {
  if (client.config.type === 'sdk') {
    return client // SDK 服务器走单独的管理路径
  }
  const connectedClient = await connectToServer(client.name, client.config)
  if (connectedClient.type !== 'connected') {
    throw new TelemetrySafeError(...)
  }
  return connectedClient
}
```

### 8.2.2 传输层选择

Claude Code 支持七种传输协议，类型定义在 `services/mcp/types.ts` 中：

```typescript
// services/mcp/types.ts
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
```

每种传输协议对应不同的配置 Schema：

| 传输类型 | 配置 Schema | 典型场景 |
|---------|------------|---------|
| `stdio` | `McpStdioServerConfigSchema` | 本地命令行工具 |
| `sse` | `McpSSEServerConfigSchema` | 远程 SSE 服务器 |
| `sse-ide` | `McpSSEIDEServerConfigSchema` | IDE 扩展集成 |
| `http` | `McpHTTPServerConfigSchema` | Streamable HTTP 服务器 |
| `ws` | `McpWebSocketServerConfigSchema` | WebSocket 服务器 |
| `sdk` | `McpSdkServerConfigSchema` | SDK 内嵌服务器 |
| `claudeai-proxy` | `McpClaudeAIProxyServerConfigSchema` | claude.ai 代理服务器 |

STDIO 传输是最常用的类型，适用于本地命令行工具。其配置结构为：

```typescript
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(), // 可省略，默认即 stdio
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)
```

HTTP 传输使用了 MCP 的 Streamable HTTP 规范，连接时会附加超时保护和 Accept 头规范化：

```typescript
export function wrapFetchWithTimeout(baseFetch: FetchLike): FetchLike {
  return async (url: string | URL, init?: RequestInit) => {
    const method = (init?.method ?? 'GET').toUpperCase()
    // GET 请求是长连接 SSE 流，不设超时
    if (method === 'GET') {
      return baseFetch(url, init)
    }
    // POST 请求设置 60 秒超时
    const controller = new AbortController()
    const timer = setTimeout(
      c => c.abort(new DOMException('The operation timed out.', 'TimeoutError')),
      MCP_REQUEST_TIMEOUT_MS,
      controller,
    )
    timer.unref?.()
    // ... 执行请求并清理
  }
}
```

### 8.2.3 工具获取与LRU缓存

工具获取通过 `fetchToolsForClient` 实现，它使用了自定义的 LRU（Least Recently Used）缓存策略：

```typescript
// 最大缓存大小
const MCP_FETCH_CACHE_SIZE = 20

export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.tools) return []

    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema,
    )

    // 清理 Unicode，防止不安全字符
    const toolsToProcess = recursivelySanitizeUnicode(result.tools)

    return toolsToProcess.map((tool): Tool => {
      const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
      return {
        ...MCPTool,
        name: fullyQualifiedName, // e.g., mcp__github__create_issue
        mcpInfo: { serverName: client.name, toolName: tool.name },
        isMcp: true,
        searchHint: tool._meta?.['anthropic/searchHint'],
        alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
        inputJSONSchema: tool.inputSchema,
        // 动态生成 description、prompt、call 等方法...
      }
    })
  },
  // LRU 缓存配置
  { maxSize: MCP_FETCH_CACHE_SIZE, resolver: (client) => client.name },
)
```

值得注意的是工具名称的构建规则。`buildMcpToolName` 会将服务器名和工具名拼接为 `mcp__<serverName>__<toolName>` 格式。这种命名空间机制确保了不同 MCP 服务器的工具不会发生命名冲突。

每个 MCP 工具都会动态生成其 `description` 和 `prompt`。描述长度被限制为 2048 字符，防止 OpenAPI 生成的巨型描述占用过多上下文：

```typescript
const MAX_MCP_DESCRIPTION_LENGTH = 2048

async prompt() {
  const desc = tool.description ?? ''
  return desc.length > MAX_MCP_DESCRIPTION_LENGTH
    ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '... [truncated]'
    : desc
},
```

### 8.2.4 资源获取

资源获取通过 `fetchResourcesForClient` 实现，同样使用 LRU 缓存：

```typescript
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.resources) return []

    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema,
    )

    return result.resources.map(resource => ({
      ...resource,
      server: client.name,
    }))
  },
  { maxSize: MCP_FETCH_CACHE_SIZE, resolver: (client) => client.name },
)
```

资源缓存在 `onclose` 和 `resources/list_changed` 通知时会被自动失效，确保结果不会过时。

### 8.2.5 Session过期重试

MCP 协议规定服务器可以返回 HTTP 404 + JSON-RPC 错误码 -32001 来表示 session 已过期。Claude Code 实现了专门的检测和重试机制：

```typescript
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error
    ? (error as Error & { code?: number }).code
    : undefined
  if (httpStatus !== 404) return false
  // 检查 JSON-RPC 错误码以区分普通 404
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
```

工具调用时会捕获 session 过期错误并重试：

```typescript
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    // 执行工具调用...
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // 清除缓存后重试
    }
    throw error
  }
}
```

## 8.3 MCP配置系统

### 8.3.1 多作用域配置

MCP 配置系统支持七种作用域（scope），定义在 `types.ts` 中：

```typescript
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',      // 本地项目配置（.claude/settings.local.json）
    'user',       // 用户全局配置（~/.claude/settings.json）
    'project',    // 项目配置（.mcp.json）
    'dynamic',    // 运行时动态注入（--mcp-config）
    'enterprise', // 企业管理配置（managed-mcp.json）
    'claudeai',   // claude.ai 在线连接器
    'managed',    // 受管理配置
  ]),
)
```

配置的加载和合并遵循严格的优先级规则。`getClaudeCodeMcpConfigs` 函数实现了完整的合并逻辑：

```typescript
export async function getClaudeCodeMcpConfigs(
  dynamicServers: Record<string, ScopedMcpServerConfig> = {},
): Promise<{
  servers: Record<string, ScopedMcpServerConfig>
  errors: PluginError[]
}> {
  // 1. 如果存在企业配置，它具有排他控制权
  const { servers: enterpriseServers } = getMcpConfigsByScope('enterprise')
  if (doesEnterpriseMcpConfigExist()) {
    return { servers: filtered, errors: [] }
  }

  // 2. 加载各作用域配置
  const { servers: userServers } = getMcpConfigsByScope('user')
  const { servers: projectServers } = getMcpConfigsByScope('project')
  const { servers: localServers } = getMcpConfigsByScope('local')

  // 3. 加载插件 MCP 服务器
  const pluginResult = await loadAllPluginsCacheOnly()
  // ... 处理插件服务器

  // 4. 去重：插件 vs 手动配置
  const { servers: dedupedPluginServers } = dedupPluginMcpServers(
    enabledPluginServers,
    enabledManualServers,
  )

  // 5. 按优先级合并：plugin < user < project < local
  const configs = Object.assign(
    {},
    dedupedPluginServers,
    userServers,
    approvedProjectServers,
    localServers,
  )

  // 6. 应用策略过滤
  for (const [name, serverConfig] of Object.entries(configs)) {
    if (!isMcpServerAllowedByPolicy(name, serverConfig)) continue
    filtered[name] = serverConfig
  }

  return { servers: filtered, errors: mcpErrors }
}
```

配置合并的关键设计是**后者覆盖前者**——local 配置优先于 project，project 优先于 user，user 优先于 plugin。Enterprise 配置则具有排他性，一旦存在就忽略所有其他来源。

### 8.3.2 配置解析

`parseMcpConfig` 函数负责验证和解析配置对象。它使用 Zod Schema 进行严格的类型校验，并支持环境变量展开：

```typescript
export function parseMcpConfig(params: {
  configObject: unknown
  expandVars: boolean
  scope: ConfigScope
  filePath?: string
}): {
  config: McpJsonConfig | null
  errors: ValidationError[]
} {
  const schemaResult = McpJsonConfigSchema().safeParse(configObject)
  if (!schemaResult.success) {
    return { config: null, errors: /* 格式化 Zod 错误 */ }
  }

  for (const [name, config] of Object.entries(schemaResult.data.mcpServers)) {
    if (expandVars) {
      const { expanded, missingVars } = expandEnvVars(config)
      if (missingVars.length > 0) {
        errors.push({ message: `Missing environment variables: ${missingVars.join(', ')}` })
      }
      configToCheck = expanded
    }
    validatedServers[name] = configToCheck
  }

  return { config: { mcpServers: validatedServers }, errors }
}
```

环境变量展开支持 `${VAR_NAME}` 语法，适用于 stdio 服务器的 command、args、env，以及远程服务器的 url、headers 等字段。

### 8.3.3 去重机制

当插件和手动配置指向同一个底层服务器时，系统会通过签名比对进行去重：

```typescript
export function getMcpServerSignature(config: McpServerConfig): string | null {
  // stdio 服务器：序列化 command + args
  const cmd = getServerCommandArray(config)
  if (cmd) return `stdio:${jsonStringify(cmd)}`
  // 远程服务器：使用 URL
  const url = getServerUrl(config)
  if (url) return `url:${unwrapCcrProxyUrl(url)}`
  return null
}
```

`dedupPluginMcpServers` 使用签名匹配，手动配置优先于插件配置，同一插件内先加载的优先：

```typescript
export function dedupPluginMcpServers(
  pluginServers: Record<string, ScopedMcpServerConfig>,
  manualServers: Record<string, ScopedMcpServerConfig>,
): { servers: Record<string, ScopedMcpServerConfig>; suppressed: Array<...> } {
  const manualSigs = new Map<string, string>()
  for (const [name, config] of Object.entries(manualServers)) {
    const sig = getMcpServerSignature(config)
    if (sig && !manualSigs.has(sig)) manualSigs.set(sig, name)
  }

  for (const [name, config] of Object.entries(pluginServers)) {
    const sig = getMcpServerSignature(config)
    const manualDup = manualSigs.get(sig)
    if (manualDup !== undefined) {
      suppressed.push({ name, duplicateOf: manualDup })
      continue
    }
    // 同一插件内先加载的优先
    const pluginDup = seenPluginSigs.get(sig)
    if (pluginDup !== undefined) {
      suppressed.push({ name, duplicateOf: pluginDup })
      continue
    }
    seenPluginSigs.set(sig, name)
    servers[name] = config
  }
  return { servers, suppressed }
}
```

## 8.4 MCP认证

MCP 认证系统位于 `services/mcp/auth.ts`，是整个子系统中第二大的文件（约 89KB），支持完整的 OAuth 2.0 流程。

### 8.4.1 ClaudeAuthProvider

`ClaudeAuthProvider` 实现了 MCP SDK 的 `OAuthClientProvider` 接口，为每个 MCP 服务器提供独立的 OAuth 凭证管理：

```typescript
export class ClaudeAuthProvider implements OAuthClientProvider {
  private serverName: string
  private serverConfig: McpSSEServerConfig | McpHTTPServerConfig

  // 核心方法
  async clientInformation(): Promise<OAuthClientInformationFull> { ... }
  async tokens(): Promise<OAuthTokens | undefined> { ... }
  async saveTokens(tokens: OAuthTokens): Promise<void> { ... }
  async redirectUrl(): Promise<URL> { ... }
  async saveCodeVerifier(codeVerifier: string): Promise<void> { ... }
  async codeVerifier(): Promise<string> { ... }
}
```

### 8.4.2 Token刷新

Token 刷新机制处理了多种边界情况，包括非标准错误码的规范化：

```typescript
// 某些 OAuth 服务器（如 Slack）在 HTTP 200 中返回错误
const NONSTANDARD_INVALID_GRANT_ALIASES = new Set([
  'invalid_refresh_token',
  'expired_refresh_token',
  'token_expired',
])

export async function normalizeOAuthErrorBody(
  response: Response,
): Promise<Response> {
  if (!response.ok) return response
  const text = await response.text()
  const parsed = jsonParse(text)

  // 如果是有效的 token 响应，直接返回
  if (OAuthTokensSchema.safeParse(parsed).success) {
    return new Response(text, response)
  }

  // 检测是否为 OAuth 错误响应
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)

  // 规范化非标准错误码
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: `Server returned: ${result.data.error}` }
    : result.data

  return new Response(jsonStringify(normalized), { status: 400 })
}
```

### 8.4.3 认证缓存

为避免每次连接都触发认证流程，系统维护了一个需要认证的服务器缓存，TTL 为 15 分钟：

```typescript
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000

async function isMcpAuthCached(serverId: string): Promise<boolean> {
  const cache = await getMcpAuthCache()
  const entry = cache[serverId]
  if (!entry) return false
  return Date.now() - entry.timestamp < MCP_AUTH_CACHE_TTL_MS
}

// 写入串行化，防止并发读-改-写竞争
let writeChain = Promise.resolve()
function setMcpAuthCacheEntry(serverId: string): void {
  writeChain = writeChain.then(async () => {
    const cache = await getMcpAuthCache()
    cache[serverId] = { timestamp: Date.now() }
    await writeFile(cachePath, jsonStringify(cache))
    authCachePromise = null // 使读缓存失效
  })
}
```

### 8.4.4 Claude.ai代理认证

对于通过 claude.ai 代理连接的 MCP 服务器，系统使用 OAuth Bearer Token 并实现了自动重试：

```typescript
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const currentTokens = getClaudeAIOAuthTokens()
      const headers = new Headers(init?.headers)
      headers.set('Authorization', `Bearer ${currentTokens.accessToken}`)
      const response = await innerFetch(url, { ...init, headers })
      return { response, sentToken: currentTokens.accessToken }
    }

    const { response, sentToken } = await doRequest()
    if (response.status !== 401) return response

    // 401 时尝试刷新 token 并重试
    const tokenChanged = await handleOAuth401Error(sentToken)
    if (!tokenChanged) return response

    return (await doRequest()).response
  }
}
```

## 8.5 工具包装：MCPTool

`MCPTool` 是连接 MCP 协议与 Claude Code 工具系统的桥梁。它定义在 `tools/MCPTool/MCPTool.ts` 中，作为所有 MCP 工具的模板：

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                    // 被 client.ts 覆盖为实际名称
  maxResultSizeChars: 100_000,     // 结果最大 100K 字符

  // 使用 passthrough schema：允许任意对象
  get inputSchema() {
    return z.object({}).passthrough()
  },

  // 权限检查：所有 MCP 工具都需要用户确认
  async checkPermissions(): Promise<PermissionResult> {
    return {
      behavior: 'passthrough',
      message: 'MCPTool requires permission.',
    }
  },

  // 以下属性由 client.ts 中的 fetchToolsForClient 动态覆盖
  async description() { return DESCRIPTION },
  async prompt() { return PROMPT },
  async call() { return { data: '' } },
  userFacingName: () => 'mcp',
  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolResultMessage,
})
```

关键设计点是 `MCPTool` 的属性在 `fetchToolsForClient` 中被逐一覆盖。这种"模板+覆盖"的模式让每个 MCP 工具都继承了标准 Tool 接口的渲染、权限检查等能力，同时拥有自己的名称、描述和调用逻辑。

`inputSchema` 使用了 `z.object({}).passthrough()`，这意味着它接受任意 JSON 对象作为输入。MCP 工具的 input schema 来自远程服务器的 `tools/list` 响应，以 `inputJSONSchema` 属性的形式附加到工具定义上，供模型生成正确格式的参数。

## 8.6 资源暴露

### 8.6.1 ListMcpResourcesTool

`ListMcpResourcesTool` 让 Claude 能够发现所有已连接 MCP 服务器提供的资源：

```typescript
export const ListMcpResourcesTool = buildTool({
  isConcurrencySafe() { return true },   // 可并发调用
  isReadOnly() { return true },           // 只读操作
  shouldDefer: true,                      // 延迟加载
  name: LIST_MCP_RESOURCES_TOOL_NAME,

  async call(input, { options: { mcpClients } }) {
    const { server: targetServer } = input
    const clientsToProcess = targetServer
      ? mcpClients.filter(client => client.name === targetServer)
      : mcpClients

    // fetchResourcesForClient 使用 LRU 缓存
    // 启动时已预热，onclose 和 resources/list_changed 时失效
    const results = await Promise.all(
      clientsToProcess.map(async client => {
        if (client.type !== 'connected') return []
        try {
          const fresh = await ensureConnectedClient(client)
          return await fetchResourcesForClient(fresh)
        } catch (error) {
          logMCPError(client.name, errorMessage(error))
          return [] // 单个服务器失败不影响整体
        }
      }),
    )

    return { data: results.flat() }
  },
})
```

### 8.6.2 ReadMcpResourceTool

`ReadMcpResourceTool` 通过 URI 读取特定资源的内容，并特别处理了二进制内容：

```typescript
export const ReadMcpResourceTool = buildTool({
  async call(input, { options: { mcpClients } }) {
    const { server: serverName, uri } = input
    const client = mcpClients.find(c => c.name === serverName)

    const connectedClient = await ensureConnectedClient(client)
    const result = await connectedClient.client.request(
      { method: 'resources/read', params: { uri } },
      ReadResourceResultSchema,
    )

    // 拦截 blob 字段：解码 base64，写入磁盘
    const contents = await Promise.all(
      result.contents.map(async (c, i) => {
        if ('text' in c) {
          return { uri: c.uri, mimeType: c.mimeType, text: c.text }
        }
        if (!('blob' in c) || typeof c.blob !== 'string') {
          return { uri: c.uri, mimeType: c.mimeType }
        }
        // 将二进制内容持久化到磁盘，避免 base64 占用上下文
        const persisted = await persistBinaryContent(
          Buffer.from(c.blob, 'base64'),
          c.mimeType,
          persistId,
        )
        return {
          uri: c.uri,
          mimeType: c.mimeType,
          blobSavedTo: persisted.filepath,
          text: getBinaryBlobSavedMessage(persisted.filepath, ...),
        }
      }),
    )

    return { data: { contents } }
  },
})
```

这里有一个重要的设计决策：二进制资源不会以 base64 编码直接放入上下文窗口，而是写入磁盘并返回文件路径。这避免了大型二进制内容（如图片、PDF）消耗宝贵的上下文空间。

## 8.7 InProcess传输

`InProcessTransport` 实现了同一进程内 MCP 服务器和客户端之间的通信，这在 Agent 间通信等场景中非常有用：

```typescript
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined
  private closed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  _setPeer(peer: InProcessTransport): void {
    this.peer = peer
  }

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.closed) throw new Error('Transport is closed')
    // 使用 queueMicrotask 异步投递，避免同步请求/响应循环中的栈溢出
    queueMicrotask(() => {
      this.peer?.onmessage?.(message)
    })
  }

  async close(): Promise<void> {
    if (this.closed) return
    this.closed = true
    this.onclose?.()
    // 同时关闭对端
    if (this.peer && !this.peer.closed) {
      this.peer.closed = true
      this.peer.onclose?.()
    }
  }
}

export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]
}
```

`createLinkedTransportPair` 创建一对互联的传输对象，一端用于客户端，一端用于服务器。消息通过 `queueMicrotask` 异步投递，这是一个关键的设计选择——它避免了同步请求-响应循环可能导致的栈深度问题。

## 8.8 SDK Control传输

`SdkControlTransport` 解决了一个更复杂的通信问题：当 MCP 服务器运行在 SDK 进程中时，需要通过控制消息在 CLI 进程和 SDK 进程之间桥接通信。

```
CLI Process                           SDK Process
┌───────────────────┐                ┌───────────────────┐
│   MCP Client      │                │   MCP Server      │
│         │         │                │         │         │
│  SdkControlClient │   stdout/stdin │  SdkControlServer │
│  Transport        │◄──────────────►│  Transport        │
│         │         │   控制消息      │         │         │
│ StructuredIO      │                │    Query           │
└───────────────────┘                └───────────────────┘
```

CLI 端使用 `SdkControlClientTransport`：

```typescript
export class SdkControlClientTransport implements Transport {
  constructor(
    private serverName: string,
    private sendMcpMessage: SendMcpMessageCallback,
  ) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) throw new Error('Transport is closed')
    // 发送消息并等待响应
    const response = await this.sendMcpMessage(this.serverName, message)
    // 将响应回传给 MCP 客户端
    if (this.onmessage) {
      this.onmessage(response)
    }
  }
}
```

SDK 端使用 `SdkControlServerTransport`：

```typescript
export class SdkControlServerTransport implements Transport {
  constructor(private sendMcpMessage: (message: JSONRPCMessage) => void) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) throw new Error('Transport is closed')
    this.sendMcpMessage(message)
  }
}
```

这种设计允许多个 SDK MCP 服务器同时运行，通过 `serverName` 字段路由到正确的服务器实例。

## 8.9 连接生命周期管理

MCP 连接不是一次性的——它们需要处理断开、重连、认证过期等各种情况。连接状态用联合类型精确建模：

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer    // 已连接，可用
  | FailedMCPServer       // 连接失败
  | NeedsAuthMCPServer    // 需要认证
  | PendingMCPServer      // 连接中
  | DisabledMCPServer     // 已禁用
```

连接批次管理通过批量并发控制来避免同时启动过多连接：

```typescript
export function getMcpServerConnectionBatchSize(): number {
  return parseInt(process.env.MCP_SERVER_CONNECTION_BATCH_SIZE || '', 10) || 3
}

function getRemoteMcpServerConnectionBatchSize(): number {
  return parseInt(process.env.MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE || '', 10) || 20
}
```

本地服务器（stdio）默认批量 3，远程服务器默认批量 20。这是因为本地服务器需要启动子进程，资源消耗更大。

## 8.10 实际应用场景

### 场景一：添加GitHub MCP服务器

用户运行 `claude mcp add github` 命令，提供 stdio 配置后，系统会：

1. `addMcpConfig` 验证配置有效性，检查企业策略
2. 写入 `.mcp.json` 或 `~/.claude/settings.json`
3. 下次启动时 `getClaudeCodeMcpConfigs` 读取配置
4. `connectToServer` 通过 STDIO 传输建立连接
5. `fetchToolsForClient` 获取工具列表，注册为 `mcp__github__*` 工具
6. Claude 在对话中可以调用这些工具

### 场景二：IDE扩展集成

VS Code 等 IDE 通过 SSE-IDE 传输类型集成 MCP 服务器，提供代码诊断、执行等能力。仅特定工具（如 `mcp__ide__executeCode`、`mcp__ide__getDiagnostics`）会暴露给模型。

### 场景三：企业级MCP管控

企业管理员通过 `managed-mcp.json` 配置文件实现：
- 使用 `allowedMcpServers` 白名单限制可用服务器
- 使用 `deniedMcpServers` 黑名单封禁特定服务器
- 企业配置具有排他控制权，用户无法添加额外服务器

## 8.11 本章小结

Claude Code 的 MCP 实现是一个工业级的协议客户端系统，其设计体现了以下关键思想：

1. **Memoization驱动的连接池**：通过 `memoize` 和 LRU 缓存避免重复连接和重复获取，同时在状态变化时精确失效。
2. **多传输层适配**：统一的 `connectToServer` 接口背后，支持从本地子进程到远程 WebSocket 的七种传输方式。
3. **层次化配置合并**：从企业到本地的多级配置，通过签名去重和策略过滤确保一致性。
4. **健壮的认证流**：完整的 OAuth 2.0 支持，包括非标准错误码规范化、token 刷新重试、认证状态缓存。
5. **优雅的模板覆盖模式**：`MCPTool` 作为基础模板，由 `fetchToolsForClient` 动态填充，将 JSON Schema 转换为标准 Tool 接口。

