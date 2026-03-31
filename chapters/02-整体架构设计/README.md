# 第二章：整体架构设计

## 2.1 分层架构总览

Claude Code采用了清晰的分层架构设计。从用户输入到最终输出，数据流经五个核心层次：

```
┌──────────────────────────────────────────────────────────────────┐
│                        入口层 (Entry Layer)                       │
│  cli.tsx → main.tsx → Commander.js命令解析                        │
│  职责：快速路径分发、参数解析、模式判定                               │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     编排层 (Orchestration Layer)                   │
│  setup.ts → replLauncher.tsx → REPL.tsx / print.ts               │
│  职责：环境初始化、会话建立、UI渲染或headless执行                    │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                  查询引擎层 (Query Engine Layer)                   │
│  QueryEngine.ts → query.ts → claude.ts (API client)              │
│  职责：对话状态管理、System Prompt构建、API调用、流式响应处理         │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     工具层 (Tool Layer)                            │
│  Tool.ts → tools.ts → 具体工具实现 (BashTool, FileEditTool, ...)  │
│  职责：工具注册与发现、权限检查、输入验证、工具执行、结果格式化       │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                     服务层 (Service Layer)                         │
│  services/api/ → services/mcp/ → services/compact/ → utils/      │
│  职责：API通信、MCP协议、上下文压缩、文件操作、进程管理              │
└──────────────────────────────────────────────────────────────────┘
```

这种分层有两个显著特点：

**1. 层与层之间的解耦通过接口和类型完成。** 例如，工具层的核心是 `Tool` 接口（定义在 `Tool.ts`），所有工具实现都遵循这个统一接口。查询引擎层不需要知道具体有哪些工具，只需要通过 `Tools` 类型（`readonly Tool[]`）与工具池交互。

**2. 数据向下流动，事件向上冒泡。** 用户输入从入口层逐级传递到服务层，而工具执行结果、流式事件则通过回调函数和状态更新机制向上传播。

## 2.2 入口层：多模式启动

Claude Code的入口层设计体现了一个重要原则：**快速路径优先（fast-path first）**。

在 `cli.tsx` 中，程序并不是一股脑地加载所有模块，而是首先检查命令行参数，对简单请求采用最短路径响应：

```typescript
// cli.tsx — 快速路径示例
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // 快速路径：--version 完全不需要加载任何模块
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // 快速路径：bridge模式
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
    const { bridgeMain } = await import('../bridge/bridgeMain.js');
    await bridgeMain(args.slice(1));
    return;
  }

  // 快速路径：daemon worker
  if (feature('DAEMON') && args[0] === '--daemon-worker') {
    const { runDaemonWorker } = await import('../daemon/workerRegistry.js');
    await runDaemonWorker(args[1]);
    return;
  }

  // 默认路径：加载完整CLI
  // ...进入 main.tsx 的完整初始化流程
}
```

这种设计确保了：
- `claude --version` 的响应时间几乎为零（没有任何 `import` 开销）
- 子命令（如 `claude mcp serve`）只加载必要的模块
- 只有完整的交互/print模式才触发全量初始化

## 2.3 编排层：Commander.js命令体系

进入完整路径后，`main.tsx` 使用 Commander.js 构建了一个多层命令体系：

```
claude [prompt]                    # 默认命令：启动交互或print模式
claude mcp <subcommand>            # MCP服务器管理
claude auth <subcommand>           # 认证管理
claude plugin <subcommand>         # 插件管理
claude open <cc-url>               # 直连服务器
claude ssh <host> [dir]            # SSH远程模式
```

`main.tsx` 中的 `run()` 函数展示了Commander.js的配置方式：

```typescript
async function run(): Promise<CommanderCommand> {
  const program = new CommanderCommand()
    .configureHelp(createSortedHelpConfig())
    .enablePositionalOptions();

  // preAction hook：命令执行前的初始化
  program.hook('preAction', async (thisCommand) => {
    await Promise.all([
      ensureMdmSettingsLoaded(),
      ensureKeychainPrefetchCompleted()
    ]);
    await init();
    runMigrations();
    void loadRemoteManagedSettings();
    void loadPolicyLimits();
  });

  // 主命令定义（带有丰富的选项）
  program
    .name('claude')
    .argument('[prompt]', 'Your prompt')
    .option('-p, --print', 'Print response and exit')
    .option('--model <model>', 'Model for the session')
    .option('-c, --continue', 'Continue most recent conversation')
    .option('-r, --resume [value]', 'Resume conversation by ID')
    .option('--permission-mode <mode>', 'Permission mode')
    .option('--mcp-config <configs...>', 'Load MCP servers')
    // ... 更多选项（共计50+个CLI选项）
    .action(async (prompt, options) => {
      // 主命令的处理逻辑
    });

  return program.parseAsync();
}
```

一个值得注意的设计是 **preAction hook**。Commander.js的hook机制让初始化逻辑只在实际执行命令时运行，而不是在显示帮助信息时运行。这避免了 `claude --help` 触发不必要的网络请求和文件系统操作。

## 2.4 查询引擎层：对话循环的心脏

`QueryEngine` 是整个系统最核心的组件。它封装了一个完整的对话生命周期：

```typescript
// QueryEngine.ts — 核心结构
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]       // 对话历史
  private abortController: AbortController // 中断控制
  private totalUsage: NonNullableUsage     // Token用量追踪
  private readFileState: FileStateCache    // 文件状态缓存

  // 提交新消息，启动一轮对话
  async *submitMessage(userInput: string): AsyncGenerator<SDKMessage> {
    // 1. 处理用户输入（斜杠命令解析、附件处理等）
    // 2. 构建System Prompt
    // 3. 调用API（流式）
    // 4. 处理工具调用
    // 5. 如果有工具调用，将结果加入历史，回到步骤3
    // 6. 输出最终响应
  }
}
```

`QueryEngineConfig` 体现了该层需要的所有依赖注入：

```typescript
export type QueryEngineConfig = {
  cwd: string                        // 工作目录
  tools: Tools                       // 可用工具列表
  commands: Command[]                // 斜杠命令列表
  mcpClients: MCPServerConnection[]  // MCP客户端连接
  agents: AgentDefinition[]          // Agent定义
  canUseTool: CanUseToolFn           // 权限检查函数
  getAppState: () => AppState        // 状态读取
  setAppState: (f) => void           // 状态更新
  customSystemPrompt?: string        // 自定义系统提示
  maxTurns?: number                  // 最大轮次限制
  maxBudgetUsd?: number              // 预算限制
  // ...更多配置
}
```

## 2.5 工具层：统一的Tool接口

Claude Code的工具系统建立在一个精心设计的 `Tool` 接口之上（定义在 `Tool.ts`）。这个接口覆盖了工具生命周期的每一个阶段：

```typescript
export type Tool<Input, Output, P> = {
  // === 身份标识 ===
  readonly name: string
  aliases?: string[]
  searchHint?: string           // ToolSearch关键词匹配

  // === Schema定义 ===
  readonly inputSchema: Input   // Zod输入Schema
  outputSchema?: z.ZodType      // 输出Schema

  // === 生命周期方法 ===
  isEnabled(): boolean                              // 是否启用
  validateInput?(input, context): Promise<Result>   // 输入验证
  checkPermissions(input, context): Promise<Result> // 权限检查
  call(input, context, canUseTool): Promise<Result> // 执行调用

  // === 行为声明 ===
  isConcurrencySafe(input): boolean    // 是否支持并发
  isReadOnly(input): boolean           // 是否只读
  isDestructive?(input): boolean       // 是否有破坏性
  interruptBehavior?(): 'cancel' | 'block' // 中断策略

  // === 渲染方法 ===
  description(input, options): Promise<string>   // 动态描述
  prompt(options): Promise<string>               // 系统提示词
  renderToolUseMessage(input, options): ReactNode // 渲染使用UI
  renderToolResultMessage?(output, ...): ReactNode // 渲染结果UI

  // === 结果处理 ===
  maxResultSizeChars: number                     // 结果大小限制
  mapToolResultToToolResultBlockParam(content, id): ToolResultBlockParam
}
```

工具通过 `buildTool()` 工厂函数创建，它提供了合理的默认值：

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    // 安全的默认值
    isEnabled: () => true,
    isConcurrencySafe: () => false,  // 默认不安全（保守策略）
    isReadOnly: () => false,          // 默认假设有写操作
    isDestructive: () => false,
    checkPermissions: (input) =>
      Promise.resolve({ behavior: 'allow', updatedInput: input }),
    userFacingName: () => def.name,
    // 用户提供的定义覆盖默认值
    ...def,
  }
}
```

这里的设计哲学是 **"fail-closed where it matters"**——安全相关的默认值总是选择更保守的选项。

## 2.6 工具注册与组装

`tools.ts` 负责将所有工具组装为一个统一的工具池。其中最重要的函数是 `getAllBaseTools()`：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    // ... 条件加载的工具
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    // ... 更多工具
  ]
}
```

然后通过 `assembleToolPool()` 将内置工具与MCP工具合并：

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 内置工具排在前面作为连续前缀（优化prompt cache命中率）
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',  // 内置工具在名字冲突时优先
  )
}
```

注意排序策略的注释：内置工具和MCP工具分别排序后再拼接，是为了保持 **prompt cache的稳定性**。Anthropic API的服务端会在工具描述上设置缓存断点，如果工具顺序变化，所有下游的缓存键都会失效。

## 2.7 命令系统：斜杠命令

与工具系统平行的是命令系统（`commands.ts`）。命令是用户通过 `/command-name` 在对话中触发的交互操作：

```
┌─────────────────────────────────────────────────────────┐
│                    命令来源层次                           │
├─────────────────────────────────────────────────────────┤
│  1. 内置命令 (COMMANDS)        /help, /compact, /model  │
│  2. 内置技能 (bundledSkills)   /commit, /review         │
│  3. 内置插件技能              来自enabled插件的技能        │
│  4. 用户技能 (.claude/skills/) 用户自定义的skill文件      │
│  5. Workflow命令              workflow脚本              │
│  6. 插件命令                  第三方插件提供的命令         │
│  7. MCP技能                   MCP服务器提供的prompt      │
└─────────────────────────────────────────────────────────┘
```

命令系统的设计允许多个来源的命令共存，并通过优先级和去重机制确保一致性：

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,          // 内置技能优先
    ...builtinPluginSkills,
    ...skillDirCommands,       // 用户技能
    ...workflowCommands,
    ...pluginCommands,         // 插件命令
    ...pluginSkills,
    ...COMMANDS(),             // 内置命令最后（但最终以name去重时第一个胜出）
  ]
})
```

## 2.8 核心数据流

理解了各层的职责后，让我们追踪一个完整的用户请求生命周期：

```
用户输入 "请修复 src/app.ts 中的类型错误"
        │
        ▼
┌─ 1. 输入处理 ──────────────────────────────────────────┐
│  - 斜杠命令检测（非命令，直接作为prompt）                  │
│  - 创建 UserMessage                                     │
│  - 加载附件（CLAUDE.md记忆文件等）                        │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ 2. System Prompt 构建 ────────────────────────────────┐
│  - 基础系统提示                                          │
│  - 工具描述注入                                          │
│  - CLAUDE.md记忆文件内容                                 │
│  - 用户上下文（OS、git状态、目录结构等）                   │
│  - System Context（项目信息、环境信息）                   │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ 3. API调用（流式） ──────────────────────────────────┐
│  POST /v1/messages (stream=true)                       │
│  - model: claude-sonnet-4-6                            │
│  - system: [构建的系统提示词]                            │
│  - messages: [对话历史 + 新消息]                         │
│  - tools: [工具定义列表]                                │
│  - thinking: { type: "enabled", budget_tokens: ... }   │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ 4. 流式响应处理 ─────────────────────────────────────┐
│  逐token接收、渲染到终端                                 │
│  如果收到 tool_use block:                               │
│    - 提取工具名称和参数                                  │
│    - 跳到步骤5                                         │
│  如果收到 stop_reason="end_turn":                       │
│    - 对话结束，返回最终响应                               │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ 5. 工具执行循环 ─────────────────────────────────────┐
│  for each tool_use in response:                        │
│    a. 查找工具定义                                      │
│    b. 验证输入 (validateInput)                          │
│    c. 检查权限 (checkPermissions + hooks)               │
│    d. 如果被拒绝 → 返回拒绝消息给模型                    │
│    e. 如果通过 → 执行工具 (tool.call)                    │
│    f. 格式化结果 (mapToolResultToToolResultBlockParam)  │
│  将所有工具结果作为新的user消息                           │
│  回到步骤3，继续对话                                     │
└────────────────────────────────────────────────────────┘
```

这个循环会持续运行，直到：
- 模型返回 `stop_reason="end_turn"`（模型认为任务完成）
- 达到最大轮次限制（`--max-turns`）
- 达到预算限制（`--max-budget-usd`）
- 用户中断（Ctrl+C）
- 发生不可恢复的错误

## 2.9 关键设计决策与权衡

**决策1：权限检查在工具层而非查询引擎层。**

每个Tool自己实现 `checkPermissions()` 方法，而不是在查询引擎中统一处理。这是因为权限语义与工具语义紧密耦合——BashTool需要检查命令是否安全，FileEditTool需要检查路径是否在允许范围内。将权限逻辑下放到工具层，保持了关注点的分离。

**决策2：工具描述是动态函数而非静态字符串。**

`description()` 方法接收输入参数和选项，返回 `Promise<string>`。这允许工具描述根据当前的权限模式、工具可用性等上下文动态变化。例如，BashTool的描述会包含沙箱限制信息。

**决策3：Bootstrap State与AppState的分离。**

`bootstrap/state.ts` 维护的是进程级别的全局状态（session ID、model配置、遥测计数器等），而 `AppStateStore.ts` 维护的是会话级别的UI状态（消息列表、MCP连接、权限上下文等）。这种分离使得同一进程可以托管多个会话上下文。

**决策4：Memoization大量使用。**

`commands.ts` 中的 `loadAllCommands` 和 `getSkillToolCommands` 都使用了 lodash 的 `memoize`。命令加载涉及磁盘I/O和动态导入，memoization避免了重复加载的开销。但同时提供了 `clearCommandsCache()` 方法来在必要时刷新缓存。

---

