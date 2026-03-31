# 第三章：启动与初始化流程

## 3.1 总体启动序列

Claude Code的启动过程是一个精心编排的异步流程，从用户敲下 `claude` 命令到REPL界面出现，经历了多个阶段。以下是完整的启动序列概览：

```
用户执行 `claude`
      │
      ▼
┌─ cli.tsx ──────────────────────────────────────────────┐
│  1. 环境预处理（COREPACK、CCR heap、ablation baseline）  │
│  2. 快速路径检测与分发                                    │
│  3. 加载 main.tsx 模块                                   │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ main.tsx 模块加载 ────────────────────────────────────┐
│  4. 并行启动副作用（profileCheckpoint, MDM读取,          │
│     Keychain预取）                                      │
│  5. 135ms+ 的模块导入                                   │
│  6. 调试检测（禁止调试运行）                               │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ main() 函数 ──────────────────────────────────────────┐
│  7. 安全预设（NoDefaultCurrentDirectoryInExePath）       │
│  8. 信号处理注册（SIGINT, exit cursor reset）            │
│  9. 特殊URL/子命令预处理                                 │
│  10. 交互性判定（isInteractive）                         │
│  11. 客户端类型确定（clientType）                         │
│  12. Settings early loading                             │
│  13. 调用 run()                                         │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ run() → Commander.js解析 ─────────────────────────────┐
│  14. Commander.js程序配置                                │
│  15. preAction hook（MDM等待、init()、迁移、远程设置）    │
│  16. 命令参数解析与验证                                   │
│  17. 进入主命令 action                                   │
└────────┬───────────────────────────────────────────────┘
         ▼
┌─ 主命令action ─────────────────────────────────────────┐
│  18. setup() 调用（环境检查、worktree、hooks快照）       │
│  19. MCP服务器启动                                      │
│  20. 工具集和命令集构建                                   │
│  21. 会话恢复或新建                                      │
│  22. AppState初始化                                     │
│  23. launchRepl() 或 runHeadless()                      │
└────────────────────────────────────────────────────────┘
```

## 3.2 CLI入口点：cli.tsx

`cli.tsx` 是整个应用的真正入口点（由package.json的 `bin` 字段或Bun打包的入口指定）。它的设计原则是**最小化默认路径的模块加载**。

### 环境预处理

在任何函数定义之前，`cli.tsx` 就执行了几个顶层副作用：

```typescript
// 禁止corepack自动修改package.json
process.env.COREPACK_ENABLE_AUTO_PIN = '0';

// CCR容器环境：设置子进程最大堆大小
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  process.env.NODE_OPTIONS = existing
    ? `${existing} --max-old-space-size=8192`
    : '--max-old-space-size=8192';
}
```

### 快速路径分发

`cli.tsx` 的 `main()` 函数实现了一个精妙的快速路径分发机制。这些路径按照"加载开销从低到高"的顺序排列：

```
快速路径优先级（从快到慢）：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. --version         零模块加载，直接输出版本号
2. --dump-system-prompt  只加载config和prompt模块
3. --claude-in-chrome-mcp  加载Chrome MCP服务器
4. --daemon-worker   加载daemon worker
5. remote-control    加载bridge模块
6. daemon            加载daemon主模块
7. ps/logs/attach/kill  加载背景会话管理
8. mcp serve         加载MCP服务器
9. 默认路径          加载完整的main.tsx
```

最快路径——`--version`——的实现尤其值得注意：

```typescript
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;
}
```

`MACRO.VERSION` 是构建时内联的常量，无需任何运行时计算。

### SSH和Direct Connect的预处理

对于特殊的连接模式，`cli.tsx` 需要在Commander.js解析之前重写 `process.argv`：

```typescript
// `claude ssh <host> [dir]` 的处理
if (feature('SSH_REMOTE') && _pendingSSH) {
  const rawCliArgs = process.argv.slice(2);
  if (rawCliArgs[0] === 'ssh' && rawCliArgs[1] && !rawCliArgs[1].startsWith('-')) {
    _pendingSSH.host = rawCliArgs[1];
    // 从argv中移除 'ssh' 和参数
    // 让剩余参数通过Commander正常解析
    process.argv = [process.argv[0]!, process.argv[1]!, ...rest];
  }
}
```

这种"先提取再传递"的模式允许SSH功能复用主命令的完整选项集（`--model`, `--resume` 等），同时又避免了Commander.js解析 `ssh` 子命令的复杂性。

## 3.3 main.tsx模块加载：并行预取的艺术

当执行路径进入 `main.tsx` 时，文件头部的副作用立即开始执行。这是一段经过精心调优的并行化代码：

```typescript
// 第1步：标记入口时间点
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

// 第2步：启动MDM设备管理数据读取（plutil/reg query子进程）
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();

// 第3步：启动Keychain预取（macOS OAuth + API key读取）
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();

// 第4步：后续的135ms+模块导入...
import { Command as CommanderCommand } from '@commander-js/extra-typings';
import chalk from 'chalk';
// ... 大量import语句
```

关键洞察在注释中有明确说明：

> `startMdmRawRead` fires MDM subprocesses (plutil/reg query) so they run in parallel with the remaining ~135ms of imports below.

> `startKeychainPrefetch` fires both macOS keychain reads (OAuth + legacy API key) in parallel — `isRemoteManagedSettingsEligible()` otherwise reads them sequentially via sync spawn inside `applySafeConfigEnvironmentVariables()` (~65ms on every macOS startup)

这是一个精妙的时间线优化：

```
时间线：
0ms     ─────────────────────────────────────────────────→
        │ profileCheckpoint
        │ startMdmRawRead() ─── 子进程运行 ──── 完成
        │ startKeychainPrefetch() ─── 子进程运行 ──── 完成
        │ 模块导入开始 ─────── 135ms+ ─────── 导入完成
        │                                      │
        │                                      ▼
        │                              preAction中等待结果
```

子进程的启动和模块导入在时间上完全重叠，节省了约65ms的启动时间。

## 3.4 init()：初始化函数

`entrypoints/init.ts` 的 `init()` 函数使用 lodash 的 `memoize` 包装，确保全局只执行一次：

```typescript
export const init = memoize(async (): Promise<void> => {
  // 1. 验证并启用配置系统
  enableConfigs();

  // 2. 应用安全的环境变量（信任对话框之前只加载安全部分）
  applySafeConfigEnvironmentVariables();

  // 3. 应用自定义CA证书（必须在首次TLS握手前）
  applyExtraCACertsFromConfig();

  // 4. 设置优雅关闭处理
  setupGracefulShutdown();

  // 5. 初始化1P事件日志（异步，不阻塞）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging();
    gb.onGrowthBookRefresh(() => {
      void fp.reinitialize1PEventLoggingIfConfigChanged();
    });
  });

  // 6. OAuth信息填充（异步）
  void populateOAuthAccountInfoIfNeeded();

  // 7. 远程管理设置的加载承诺初始化
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise();
  }
  if (isPolicyLimitsEligible()) {
    initializePolicyLimitsLoadingPromise();
  }

  // 8. 配置mTLS和全局代理
  configureGlobalMTLS();
  configureGlobalAgents();

  // 9. API预连接（TCP+TLS握手与后续工作重叠）
  preconnectAnthropicApi();

  // 10. Windows shell设置
  setShellIfWindows();

  // 11. 清理注册
  registerCleanup(shutdownLspServerManager);

  // 12. Scratchpad目录初始化
  if (isScratchpadEnabled()) {
    await ensureScratchpadDir();
  }
});
```

这个初始化序列有几个重要的设计考量：

**信任边界的划分。** 在信任对话框（Trust Dialog）被接受之前，只能应用"安全的"配置环境变量。完整的环境变量应用（`applyConfigEnvironmentVariables()`）必须等到信任建立之后。这是因为 git hooks 等机制可以在未受信任的仓库中执行任意代码。

**API预连接。** `preconnectAnthropicApi()` 在init阶段就开始TCP+TLS握手，这样当第一个API请求发出时，连接已经就绪。注释中说这可以节省100-200ms。

**异步与同步的混合。** 非关键路径的操作（OAuth、事件日志、JetBrains检测等）使用 `void` 前缀的异步调用，让它们在后台执行而不阻塞init的返回。

## 3.5 setup()：会话级设置

`setup.ts` 中的 `setup()` 函数处理与具体会话相关的初始化：

```typescript
export async function setup(
  cwd: string,
  permissionMode: PermissionMode,
  allowDangerouslySkipPermissions: boolean,
  worktreeEnabled: boolean,
  worktreeName: string | undefined,
  tmuxEnabled: boolean,
  customSessionId?: string | null,
  worktreePRNumber?: number,
  messagingSocketPath?: string,
): Promise<void> {
```

setup的主要职责：

### 环境验证

```typescript
// Node.js版本检查
const nodeVersion = process.version.match(/^v(\d+)\./)?.[1]
if (!nodeVersion || parseInt(nodeVersion) < 18) {
  console.error(chalk.bold.red(
    'Error: Claude Code requires Node.js version 18 or higher.'
  ));
  process.exit(1);
}
```

### Worktree创建

当用户通过 `--worktree` 标志请求git worktree时，setup会处理完整的worktree生命周期：

```typescript
if (worktreeEnabled) {
  const inGit = await getIsGit();
  if (!hasHook && !inGit) {
    // 错误：非git仓库且无hook
    process.exit(1);
  }

  const worktreeSession = await createWorktreeForSession(
    getSessionId(), slug, tmuxSessionName, ...
  );

  // 切换工作目录到worktree
  process.chdir(worktreeSession.worktreePath);
  setCwd(worktreeSession.worktreePath);
  setOriginalCwd(getCwd());
  setProjectRoot(getCwd());

  // 清除并重建缓存
  clearMemoryFileCaches();
  updateHooksConfigSnapshot();
}
```

### Hooks快照

```typescript
// 在setCwd之后捕获hooks配置快照
// 用于检测运行期间的hooks篡改
captureHooksConfigSnapshot();

// 初始化FileChanged hook监听器
initializeFileChangedWatcher(cwd);
```

### 权限模式验证

当使用 `--dangerously-skip-permissions` 时，setup会验证环境安全性：

```typescript
if (permissionMode === 'bypassPermissions') {
  // 禁止在root/sudo下使用
  if (process.getuid() === 0 && process.env.IS_SANDBOX !== '1') {
    console.error('Cannot use with root/sudo privileges');
    process.exit(1);
  }

  // 对内部用户，要求Docker/沙箱环境且无网络访问
  const [isDocker, hasInternet] = await Promise.all([
    envDynamic.getIsDocker(),
    env.hasInternetAccess(),
  ]);
  if (!isSandboxed || hasInternet) {
    console.error('Can only be used in sandboxed containers without internet');
    process.exit(1);
  }
}
```

### 延迟预取

setup的最后阶段启动了一系列后台预取任务：

```typescript
// 预取命令列表（memoized，只加载一次）
void getCommands(getProjectRoot());

// 预取插件hooks
void import('./utils/plugins/loadPluginHooks.js').then(m => {
  void m.loadPluginHooks();
  m.setupPluginHookHotReload();
});

// 锁定当前版本（防止其他进程删除）
void lockCurrentVersion();
```

## 3.6 Bootstrap State的作用

`bootstrap/state.ts` 维护了整个进程的全局状态。它被特意放在 `bootstrap/` 目录下，与应用层代码隔离：

```typescript
// bootstrap/state.ts 的注释
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE

type State = {
  originalCwd: string           // 原始工作目录
  projectRoot: string           // 项目根目录（稳定不变）
  totalCostUSD: number          // 累计API费用
  totalAPIDuration: number      // 累计API耗时
  cwd: string                   // 当前工作目录
  modelUsage: { [modelName: string]: ModelUsage }  // 模型用量
  mainLoopModelOverride: ModelSetting | undefined   // 模型覆盖
  initialMainLoopModel: ModelSetting                // 初始模型
  isInteractive: boolean        // 是否交互模式
  sessionId: SessionId          // 会话ID
  parentSessionId: SessionId | undefined  // 父会话ID
  clientType: string            // 客户端类型
  sessionSource: string | undefined       // 会话来源

  // 遥测相关
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  // ... 更多计数器

  // 安全相关
  sessionBypassPermissionsMode: boolean
  sessionTrustAccepted: boolean

  // 会话管理
  sessionPersistenceDisabled: boolean
  invokedSkills: Map<string, SkillInfo>
  planSlugCache: Map<string, string>

  // Agent相关
  agentColorMap: Map<string, AgentColorName>
  mainThreadAgentType: string | undefined
  isRemoteMode: boolean
}
```

Bootstrap State的设计理念是：

1. **进程级单例** ——通过模块系统保证全局唯一
2. **读写分离** ——提供 `getXxx()` 和 `setXxx()` 函数，不直接暴露State对象
3. **最小状态原则** ——注释明确要求"不要轻易增加状态"
4. **与AppState互补** ——Bootstrap State处理进程级信息，AppState处理会话级信息

## 3.7 REPL启动流程

REPL的启动是整个初始化流程的最后一步。`replLauncher.tsx` 采用了延迟加载策略：

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void> {
  // 延迟加载React组件——App和REPL都是动态导入
  const { App } = await import('./components/App.js');
  const { REPL } = await import('./screens/REPL.js');

  await renderAndRun(
    root,
    <App {...appProps}>
      <REPL {...replProps} />
    </App>,
  );
}
```

这里的延迟加载意味着React组件树（包括REPL的完整UI逻辑）只在真正需要渲染时才被加载。对于非交互模式（`--print`），这些模块永远不会被导入，节省了大量的模块解析时间。

`App` 组件是整个React组件树的根，它提供了：
- Context Provider层（状态、主题、FPS追踪等）
- 初始AppState注入

`REPL` 组件是主要的交互界面，它负责：
- 消息列表渲染
- 用户输入框
- 工具执行进度显示
- 权限审批弹窗
- 状态栏

## 3.8 延迟预取：首次渲染后的后台工作

在REPL首次渲染完成后，`startDeferredPrefetches()` 会启动一系列不影响首屏渲染的后台任务：

```typescript
export function startDeferredPrefetches(): void {
  // 如果是性能测量模式或bare模式，跳过所有预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER) ||
      isBareMode()) {
    return;
  }

  // 用户信息初始化
  void initUser();
  void getUserContext();

  // 系统上下文预取（git状态等，仅在信任已建立时）
  prefetchSystemContextIfSafe();

  // 提示信息预取
  void getRelevantTips();

  // 云提供商凭证预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    void prefetchAwsCredentialsAndBedRockInfoIfSafe();
  }

  // 文件计数（用于UI显示）
  void countFilesRoundedRg(getCwd(), AbortSignal.timeout(3000), []);

  // 分析和Feature Flag
  void initializeAnalyticsGates();
  void prefetchOfficialMcpUrls();
  void refreshModelCapabilities();

  // 设置变更检测器
  void settingsChangeDetector.initialize();
  void skillChangeDetector.initialize();
}
```

这些预取任务被设计为在"用户正在打字"的窗口期内完成——当用户开始输入第一个问题时，这些后台数据已经准备就绪，首次API调用不会因为等待这些数据而延迟。

## 3.9 性能优化总结

回顾整个启动流程，Claude Code采用了多种性能优化策略：

### 并行化时间线

```
0ms    50ms   100ms  150ms  200ms  250ms  300ms  350ms
|------|------|------|------|------|------|------|------|
[MDM子进程读取                    ]
[Keychain预取                     ]
[模块导入 (135ms+)                ]
                                   [init()         ]
                                   [API预连接       ]
                                                    [setup()       ]
                                                    [REPL渲染      ]
                                                                   [延迟预取...]
```

### 快速路径（Fast Path）

| 请求类型 | 加载模块数 | 响应时间 |
|---------|-----------|---------|
| `--version` | 0 | <10ms |
| `--daemon-worker` | ~5 | <50ms |
| `mcp serve` | ~20 | <100ms |
| 完整REPL | 全量 | ~350ms+ |

### 延迟加载（Lazy Loading）

- React组件（App, REPL）在 `launchRepl()` 中动态导入
- OpenTelemetry模块在 `doInitializeTelemetry()` 中延迟加载（~400KB）
- Coordinator模式、Kairos模式等通过 `feature()` 条件加载
- 插件hooks通过 `void import(...)` 异步加载

### Memoization

- `init()` 使用 `memoize()` 确保单次执行
- `loadAllCommands()` 使用 `memoize()` 缓存命令列表
- `getSkillToolCommands()` 使用 `memoize()` 缓存技能列表
- 提供 `clearCommandsCache()` 等方法在需要时刷新

### 预连接与预取

- `preconnectAnthropicApi()` ——在init阶段预建TCP+TLS连接
- `prefetchSystemContextIfSafe()` ——提前获取git状态
- `prefetchFastModeStatus()` ——提前查询Fast Mode可用性
- `prefetchOfficialMcpUrls()` ——提前获取官方MCP注册信息
- `countFilesRoundedRg()` ——提前计算文件数量

这些优化策略共同确保了Claude Code的启动体验——从用户敲下命令到看到可交互的REPL界面，整个过程流畅而迅速，同时为首次API调用的快速响应做好了充分准备。

## 3.10 迁移系统

值得一提的是 `main.tsx` 中的迁移系统。每次启动时都会检查并运行必要的数据迁移：

```typescript
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateEnableAllProjectMcpServersToSettings();
    resetProToOpusDefault();
    migrateSonnet1mToSonnet45();
    migrateLegacyOpusToCurrent();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    migrateReplBridgeEnabledToRemoteControlAtStartup();

    // 保存新版本号
    saveGlobalConfig(prev =>
      prev.migrationVersion === CURRENT_MIGRATION_VERSION
        ? prev
        : { ...prev, migrationVersion: CURRENT_MIGRATION_VERSION }
    );
  }
}
```

迁移系统的设计特点：
- 版本号驱动（`CURRENT_MIGRATION_VERSION`），避免重复执行
- 每个迁移函数是幂等的
- 同步执行，确保在命令执行前完成
- 模型名称迁移（如 `sonnet-4-5` → `sonnet-4-6`）是常见的迁移类型，因为Anthropic频繁发布新模型

---
