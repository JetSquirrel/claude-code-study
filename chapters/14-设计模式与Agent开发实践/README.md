# 第十四章：设计模式与 Agent 开发实践

## 14.1 引言

经过前十三章对 Claude Code 源码的深入分析，我们已经对一个工业级 AI Agent 系统的方方面面有了全面的理解。本章将从更高的抽象层次出发，提炼 Claude Code 中蕴含的核心设计模式，并将其转化为可复用的 Agent 开发实践指南。

本章分为四个部分：首先总结 8 个核心设计模式，然后给出构建高效 Agent 的实战指南，接着讨论性能优化经验，最后列出安全最佳实践清单。

## 14.2 核心设计模式

### 14.2.1 工具插件架构模式（Tool Plugin Pattern）

**问题**：AI Agent 需要与外部世界交互，但不同的交互方式（文件读写、命令执行、代码搜索）有截然不同的安全需求、参数格式和执行语义。如何设计一个统一的工具系统，既能容纳这些差异，又能保持一致的注册、发现和执行机制？

**解决方案**：定义统一的 Tool 接口，每个工具实现自己的参数 schema、权限需求和执行逻辑，通过注册中心统一管理：

```typescript
// 统一的 Tool 接口
interface Tool {
  name: string
  description: string
  inputSchema: Record<string, unknown>  // JSON Schema
  isEnabled(): boolean
  isReadOnly(): boolean
  needsPermission(input: unknown): PermissionDecision
  execute(input: unknown, context: ToolUseContext): AsyncGenerator<ToolResult>
}

// 注册与组装
function assembleToolPool(config: ToolConfig): Tool[] {
  const tools: Tool[] = []
  // 核心工具
  tools.push(new BashTool(), new ReadTool(), new WriteTool(), ...)
  // MCP 工具
  for (const server of mcpServers) {
    tools.push(...server.tools.map(wrapMCPTool))
  }
  // 按配置过滤
  return tools.filter(t => t.isEnabled())
}
```

**Claude Code 中的体现**：每个工具（BashTool、FileReadTool、FileEditTool、GrepTool 等）都是独立的模块，有自己的 `prompt.ts`（为 LLM 生成的描述）、`constants.ts`（配置常量）和主文件（执行逻辑）。工具通过 `assembleToolPool` 统一注册，通过 `inputSchema` 生成 API 参数，通过 `needsPermission` 接入权限系统。

**关键洞察**：
- 工具的描述（`prompt.ts`）和实现是分离的 —— 模型看到的是 Prompt 工程优化过的描述，不是代码注释
- `isReadOnly()` 标记使得权限系统可以自动区分安全操作和危险操作
- `AsyncGenerator` 返回类型天然支持流式输出和进度报告

### 14.2.2 权限分层决策模式（Layered Permission Decision）

**问题**：Agent 需要执行各种操作，从安全的文件读取到危险的 `rm -rf`。如何在不频繁打扰用户的前提下，确保每个操作都经过适当的安全审查？

**解决方案**：实现多层权限决策链，每层可以独立决定 allow/deny/escalate：

```typescript
type PermissionDecision =
  | { action: 'allow', reason: string }
  | { action: 'deny', reason: string }
  | { action: 'ask', message: string }  // 升级到下一层

// 决策链
async function checkPermission(tool: Tool, input: unknown): Promise<PermissionDecision> {
  // Layer 1: 权限模式基线（如 "plan" 模式禁止所有写入）
  const modeDecision = checkPermissionMode(tool, input)
  if (modeDecision.action !== 'ask') return modeDecision

  // Layer 2: 静态规则（settings.json 中的 allow/deny 列表）
  const ruleDecision = checkStaticRules(tool, input)
  if (ruleDecision.action !== 'ask') return ruleDecision

  // Layer 3: 会话缓存（用户之前对类似操作的决定）
  const cacheDecision = checkSessionCache(tool, input)
  if (cacheDecision.action !== 'ask') return cacheDecision

  // Layer 4: 用户确认
  return { action: 'ask', message: formatPermissionPrompt(tool, input) }
}
```

**Claude Code 中的体现**：权限系统从 `permissionMode`（plan/auto-edit/full-auto 等）到 `settings.json` 中的 `allowedTools` 和 `deniedTools`，到会话级别的缓存，形成了完整的决策链。`PathMatcher` 支持 glob 语法的规则匹配，`yoloClassifier` 使用 LLM 进行更智能的安全分类。

**关键洞察**：
- 每层决策都是独立的、可测试的
- 缓存层避免了对同类操作的重复确认
- `deny` 可以在任何层面终止链条，`allow` 也可以在任何层面放行
- 用户确认是最后的兜底，而非第一道防线

### 14.2.3 渐进式上下文压缩模式（Progressive Context Compaction）

**问题**：LLM 的上下文窗口是有限的，但长对话中会累积大量的工具调用结果、代码片段和中间推理。如何在不丢失关键信息的前提下管理上下文长度？

**解决方案**：实现多级压缩策略，从温和到激进渐进式地释放空间：

```
Level 0: 原始对话（无压缩）
  ↓ 当上下文占用达到 X%
Level 1: Snip（截断旧的工具结果为占位符）
  ↓ 当仍然不足
Level 2: Microcompact（合并相邻的小工具结果）
  ↓ 当仍然不足
Level 3: Autocompact（生成结构化摘要替换全部/部分对话）
  ↓ 当 API 返回 413
Level 4: Reactive Compact（紧急压缩 + 重试）
```

**Claude Code 中的体现**：
- 9 段式摘要模板确保压缩后保留关键信息（用户意图、文件变更、错误修复、待办事项）
- Partial Compact 支持只压缩旧消息而保留最近的上下文
- 压缩后提供原始 transcript 路径，允许回读精确细节
- `<analysis>` 标签作为思考草稿，提高摘要质量后被剥离

**关键洞察**：
- 压缩是渐进式的，不是一步到位的
- 每级压缩都保留了"安全网"（如 transcript 文件路径）
- 压缩提示词本身就是一个精心设计的 Prompt 工程产物

### 14.2.4 流式生成器管线模式（Streaming Generator Pipeline）

**问题**：Agent 的响应过程涉及 API 调用、工具执行、权限检查等多个异步步骤。如何实现实时的流式输出，同时保持对整个过程的控制？

**解决方案**：使用 `AsyncGenerator` 构建数据处理管线，每个阶段产出结构化的事件流：

```typescript
// 查询层产出 MessageStream 事件
async function* query(messages: Message[]): AsyncGenerator<MessageEvent> {
  // 调用 API，流式返回
  for await (const chunk of apiStream) {
    yield { type: 'text_delta', text: chunk.text }
    yield { type: 'tool_use', tool: chunk.tool }
  }
}

// 对话循环消费事件流
async function conversationLoop() {
  while (true) {
    for await (const event of query(messages)) {
      switch (event.type) {
        case 'text_delta':
          render(event.text)        // 实时渲染
          break
        case 'tool_use':
          const result = await executeTool(event.tool)
          messages.push(result)      // 累积上下文
          break
      }
    }
    if (noMoreToolCalls) break
  }
}
```

**Claude Code 中的体现**：`query()` 函数是 AsyncGenerator，`processAPIResponse()` 在其上构建了工具执行循环，`StreamingMessageRenderer` 消费生成器的输出进行 UI 渲染。整条管线都是惰性求值的，只在消费端拉取时才产生数据。

**关键洞察**：
- Generator 的暂停/恢复语义天然适合"生成一些 -> 执行工具 -> 继续生成"的模式
- `yield` 点是天然的取消检查点（检查 AbortSignal）
- 管线的每一级都可以独立测试

### 14.2.5 多 Agent 邮箱通信模式（Mailbox Communication）

**问题**：复杂任务需要多个 Agent 协作（主 Agent、代码搜索 Agent、验证 Agent 等），但 Agent 之间的直接通信会引入复杂的同步和上下文管理问题。如何让多个 Agent 协作而不互相干扰？

**解决方案**：每个 Agent 有独立的上下文窗口，通过"邮箱"（结构化的输入/输出）通信：

```typescript
// 主 Agent 通过工具调用启动子 Agent
const result = await agentTool.execute({
  prompt: "Search the codebase for all usages of X",
  subagent_type: "explore",
})

// 子 Agent 完成后，结果以工具返回值的形式传回主 Agent
// 主 Agent 的上下文中只有摘要结果，不包含子 Agent 的完整工具调用历史
```

**Claude Code 中的体现**：`AgentTool` 为每个子 Agent 创建独立的消息历史和工具集。子 Agent 的输出经过格式化后作为工具结果返回给主 Agent。Fork 模式的子 Agent 甚至在后台运行，不阻塞主 Agent 的交互。

**关键洞察**：
- 隔离保护了主 Agent 的上下文窗口免受子 Agent 输出的污染
- 邮箱语义使得通信是异步的、可序列化的
- 同一个 Agent 框架支持多种子 Agent 类型（explore、verify、fork）

### 14.2.6 延迟加载与特性门控模式（Deferred Loading & Feature Gates）

**问题**：大型系统有大量可选功能，每个功能都有自己的依赖链。如何避免在启动时加载所有功能，同时确保未发布的功能不会泄露？

**解决方案**：使用编译时特性门控（Feature Gates）进行死代码消除，使用运行时条件进行延迟加载：

```typescript
// 编译时门控：未启用的特性在打包时完全删除
import { feature } from 'bun:bundle'

const proactiveModule = feature('PROACTIVE')
  ? require('../proactive/index.js')
  : null

// 运行时门控：通过 GrowthBook 进行 A/B 测试
if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
  // 启用验证 Agent
}

// Lazy Schema：Zod schema 只在首次访问时创建
export const HookCommandSchema = lazySchema(() =>
  z.discriminatedUnion('type', [...])
)
```

**Claude Code 中的体现**：`feature()` 宏在 Bun 打包时解析为布尔常量，使得未启用特性的代码被完全消除。`require()` 在 `feature()` 守卫下使用，确保未启用特性的模块不被加载。`lazySchema()` 延迟 Zod schema 的创建，避免启动时的 CPU 开销。

**关键洞察**：
- 编译时门控 > 运行时门控 > 配置门控：越早消除，性能越好
- `feature()` 必须直接用在 `if` 条件中，不能赋值给变量后再判断，否则打包器无法常量折叠
- 运行时特性值应该缓存（`_CACHED_MAY_BE_STALE`），避免每次访问都查询远程配置

### 14.2.7 工作树隔离模式（Worktree Isolation）

**问题**：Agent 在修改代码时可能影响用户正在开发的分支。如何让 Agent 在不干扰用户工作的前提下进行代码修改？

**解决方案**：利用 git worktree 创建隔离的工作环境：

```typescript
// 创建 worktree
async function createWorktree(branch: string): Promise<WorktreeSession> {
  const worktreePath = computeWorktreePath(branch)
  await exec(`git worktree add ${worktreePath} -b ${branch}`)
  return {
    path: worktreePath,
    branch,
    originalCwd: getCwd(),
  }
}

// Agent 在 worktree 中工作
async function runInWorktree(session: WorktreeSession, task: Task) {
  setCwd(session.path)
  try {
    await executeTask(task)
  } finally {
    setCwd(session.originalCwd)
  }
}
```

**Claude Code 中的体现**：`EnterWorktree` / `ExitWorktree` 工具允许 Agent 创建和管理 worktree。Worktree 的路径被注入到 System Prompt 中（`This is a git worktree — an isolated copy of the repository`），确保模型知道自己在隔离环境中。同一仓库的所有 worktree 共享同一个记忆目录（通过 `findCanonicalGitRoot`）。

**关键洞察**：
- 隔离不仅是技术需求，也需要在 Prompt 层面让模型理解
- 共享记忆确保了跨 worktree 的知识连续性
- Worktree 清理需要与进程生命周期绑定

### 14.2.8 依赖注入测试模式（Dependency Injection for Testing）

**问题**：Agent 系统的核心逻辑依赖文件系统、API 调用、进程执行等外部资源，如何编写可靠的单元测试？

**解决方案**：通过抽象层注入外部依赖：

```typescript
// 文件系统抽象
interface FsOperations {
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  mkdir(path: string): Promise<void>
  readdir(path: string): Promise<Dirent[]>
  readFileSync(path: string, options: { encoding: string }): string
}

// 运行时获取实现
export function getFsImplementation(): FsOperations {
  // 在测试中可以注入 mock 实现
  return currentImplementation
}

// 全局状态集中管理
export function resetStateForTests(): void {
  resetCostState()
  clearSystemPromptSections()
  clearBundledSkills()
  // ...
}
```

**Claude Code 中的体现**：`getFsImplementation()` 抽象了文件系统操作，`getSettings_DEPRECATED()` / `getInitialSettings()` 抽象了配置读取。全局状态通过 `bootstrap/state.ts` 集中管理，提供了 `resetStateForTests()` 等测试辅助函数。Memoized 函数（如 `getAutoMemPath`）在测试间通过 `cache.clear()` 重置。

**关键洞察**：
- 全局状态应该有显式的 reset 机制，不能依赖模块重新加载
- Memoized 函数需要暴露缓存清除接口
- 测试辅助函数（`clearBundledSkills`、`clearBuiltinPlugins`）是公共 API 的一部分

## 14.3 构建高效 Agent 的实战指南

### 14.3.1 如何设计工具系统

1. **工具描述是 Prompt 工程**：为每个工具编写独立的描述文件（`prompt.ts`），针对 LLM 的理解方式优化措辞，而非简单的 API 文档
2. **区分只读和写入工具**：通过 `isReadOnly()` 标记，使权限系统可以自动区分安全操作
3. **提供专用工具而非万能工具**：Claude Code 有 Read、Edit、Write 三个独立的文件操作工具，而非一个通用的 FileOp 工具。这让权限控制更精细，也让模型更容易选择正确的操作
4. **工具结果应该结构化**：返回带有元数据（如 `isError`、`isAutoTruncated`）的结构化结果，而非纯文本

### 14.3.2 如何管理上下文窗口

1. **实现多级压缩**：从无损截断（snip）到有损摘要（compact），根据紧急程度选择不同策略
2. **设计结构化的摘要模板**：Claude Code 的 9 段式模板经过了大量 eval 迭代，每个段落都有明确的信息保留目标
3. **保留原始 transcript**：即使压缩了对话，也要提供回读原始记录的路径
4. **防止压缩器调用工具**：在压缩 Prompt 的开头和结尾都加上 "NO TOOLS" 指令
5. **利用 Prompt Cache**：将静态内容和动态内容分离，最大化缓存命中率

### 14.3.3 如何实现安全的命令执行

1. **分层权限**：模式 -> 规则 -> 缓存 -> 用户确认，每层独立、可测试
2. **默认安全**：未知操作应该要求确认，而非默认允许
3. **路径验证**：对所有文件路径进行规范化（normalize）和遍历检测（`..` 检查）
4. **命令注入防护**：使用 `execFile` 而非 `exec`，避免 shell 注入
5. **超时控制**：所有外部操作都有超时，防止 Agent 被挂起的进程阻塞

### 14.3.4 如何构建多 Agent 协作

1. **上下文隔离**：每个子 Agent 有独立的消息历史和工具集
2. **结果摘要**：子 Agent 的输出应该被摘要后返回给主 Agent，而非原始转发
3. **并行 vs 串行**：独立的子任务并行执行，有依赖关系的串行执行
4. **递归深度限制**：防止 Agent 无限嵌套创建子 Agent
5. **共享记忆**：子 Agent 应该能访问主 Agent 的记忆，但不应该直接修改

### 14.3.5 如何处理流式响应

1. **使用 AsyncGenerator**：天然支持暂停/恢复和取消
2. **事件类型化**：用 discriminated union 定义事件类型，而非字符串标记
3. **背压处理**：如果消费端处理速度跟不上生产端，需要缓冲或丢弃策略
4. **优雅取消**：通过 AbortSignal 传播取消请求，在每个 yield 点检查

### 14.3.6 如何实现会话持久化

1. **JSONL 格式**：每行一个 JSON 对象，支持追加写入和反向读取
2. **文件锁**：并发会话写入同一文件时使用文件锁防止损坏
3. **大内容外部化**：大的粘贴内容通过哈希引用存储，避免历史文件膨胀
4. **会话 ID**：每个会话有唯一 ID，用于隔离和恢复

## 14.4 性能优化经验

### 14.4.1 Prompt Cache 利用策略

**原则**：System Prompt 的前缀越稳定，缓存命中率越高。

**实践**：
- 将所有运行时条件判断集中到 DYNAMIC_BOUNDARY 之后
- 使用 section 缓存确保每轮生成的 Prompt 文本稳定
- MCP 指令通过 attachment 增量传递，避免服务器连接/断开导致的缓存失效
- `DANGEROUS_uncachedSystemPromptSection` 命名约定强制审查缓存破坏行为

**量化影响**：每个运行时条件如果放在缓存前缀中，会使缓存变体数翻倍（2^N）。将 5 个条件从前缀移到后缀，可以将变体数从 32 降到 1。

### 14.4.2 Memoization 最佳实践

**原则**：计算一次，复用多次，显式失效。

**实践**：
```typescript
// Good: 有清除接口的 memoize
export const getAutoMemPath = memoize(
  (): string => { ... },
  () => getProjectRoot(),  // 缓存键函数
)
// 测试时：getAutoMemPath.cache.clear()

// Good: Section 缓存有显式的清除时机
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()
  clearBetaHeaderLatches()
}

// Bad: 无法清除的闭包缓存（Claude Code 中没有这种）
```

### 14.4.3 并发工具执行

**原则**：独立的工具调用并行执行，有依赖的串行执行。

**实践**：
```typescript
// System Prompt 中的指令
`You can call multiple tools in a single response. If you intend to call
multiple tools and there are no dependencies between them, make all independent
tool calls in parallel.`

// Git 状态获取中的并行
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['status', '--short'], ...),
  execFileNoThrow(gitExe(), ['log', '--oneline', '-n', '5'], ...),
  execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
])
```

### 14.4.4 Token 预算管理

**原则**：充分利用预算但不空转。

**实践**：
- `COMPLETION_THRESHOLD = 0.9`：90% 使用量时判定为完成
- `DIMINISHING_THRESHOLD = 500`：连续 3 次增量不足 500 tokens 时强制停止
- 延续消息告知模型当前进度：`"You've used X% of your budget. Keep working..."`

## 14.5 安全最佳实践清单

基于 Claude Code 的安全设计，以下是构建 AI Agent 时的安全检查清单：

### 命令执行安全
- [ ] 使用 `execFile` 而非 `exec`，避免 shell 注入
- [ ] 所有外部命令有超时限制
- [ ] 危险目录（`/etc`、`/usr`、`~/.ssh`）在默认写入白名单之外
- [ ] 递归操作（`rm -rf`）需要显式用户确认
- [ ] 环境变量在传递给子进程前经过过滤

### 文件系统安全
- [ ] 所有路径在使用前经过 `normalize()` 规范化
- [ ] 检测并拒绝路径遍历（`..` 段）
- [ ] 验证符号链接的目标（防止 TOCTOU 攻击）
- [ ] 敏感文件（`.env`、`credentials.json`）默认不可读取
- [ ] 写入操作验证目标路径不在危险目录中

### 信息安全
- [ ] System Prompt 中不暴露内部模型 ID（当处于 undercover 模式时）
- [ ] Hook 的环境变量插值需要显式白名单
- [ ] 项目配置不能覆盖记忆路径到敏感目录
- [ ] 工具结果中检测并标记可能的 Prompt Injection
- [ ] 大的粘贴内容通过哈希引用而非内联存储

### 权限安全
- [ ] 默认拒绝未知操作（deny by default）
- [ ] 权限缓存有明确的作用域和失效条件
- [ ] 用户的单次授权不会被推广到更大范围
- [ ] 权限规则支持 glob 语法的精确匹配
- [ ] 子 Agent 继承但不能扩展父 Agent 的权限

### 网络安全
- [ ] URL 验证：不允许模型自行生成 URL（除非用于编程）
- [ ] HTTP Hook 的 URL 必须通过 schema 验证
- [ ] MCP 服务器连接需要用户确认
- [ ] 外部服务调用有超时和重试限制
- [ ] 上传到第三方服务前考虑内容敏感性

## 14.6 从 Claude Code 学到的架构经验

### 14.6.1 "对模型透明"原则

Claude Code 在 System Prompt 中对模型极度透明：

- 告知模型权限模式的存在和工作方式
- 告知模型上下文自动压缩的事实
- 告知模型 `<system-reminder>` 标签的含义
- 告知模型 Hook 反馈应视为用户输入
- 告知模型 worktree 环境的特殊性

这种透明性不是信息泄露，而是让模型在完整的上下文中做出更好的决策。

### 14.6.2 "约定优于配置，配置优于代码"原则

扩展体系从简单到复杂排列：

1. **约定**：`.claude/skills/*.md` 放在指定位置就能工作
2. **配置**：`settings.json` 中的 `hooks` 和 `permissions`
3. **代码**：`BundledSkillDefinition` 注册和 `registerBuiltinPlugin`

大多数用户只需要前两层就能满足需求。

### 14.6.3 "缓存是架构决策"原则

Claude Code 中的缓存不是事后优化，而是架构的一部分：

- SYSTEM_PROMPT_DYNAMIC_BOUNDARY 在设计时就决定了
- Section 缓存策略在创建时就通过 `systemPromptSection` vs `DANGEROUS_uncached` 声明
- MCP 指令的 attachment 增量传递是专门为缓存优化设计的
- `DANGEROUS_` 命名约定使缓存破坏成为需要解释的有意行为

### 14.6.4 "安全是深度防御"原则

Claude Code 的安全设计从不依赖单一防线：

- 路径安全：normalize + 遍历检测 + 符号链接验证 + 目录白名单
- 命令安全：权限模式 + 静态规则 + 会话缓存 + 用户确认
- 记忆安全：路径验证 + 来源限制 + 写入权限隔离
- 文件写入安全：O_EXCL + O_NOFOLLOW + 0o600 权限 + 进程 nonce

每一层都假设其他层可能失败。

## 14.7 未来展望

### 14.7.1 Agent 系统的演进方向

1. **更智能的上下文管理**：从基于规则的压缩到基于语义的信息检索，让 Agent 能在无限的对话历史中按需获取精确信息
2. **更丰富的多 Agent 协作**：从当前的主从模式到对等的 Agent 网络，支持更复杂的协作拓扑
3. **更深度的 IDE 集成**：从当前的终端 CLI 到与 VS Code、JetBrains 等 IDE 的原生集成，获取更丰富的编辑上下文
4. **更强的自主性**：从当前的"人在回路"（Human-in-the-loop）到有限范围内的完全自主操作，通过更精确的风险评估减少不必要的确认

### 14.7.2 对开发者的建议

如果你正在构建自己的 AI Agent 系统，Claude Code 的源码提供了一份无价的参考。以下是优先级建议：

1. **首先建立工具系统**：一个好的工具抽象层是整个系统的基础
2. **然后建立权限系统**：安全问题越早解决越好，事后补救的成本远高于提前设计
3. **接着建立流式管线**：用户体验的核心是实时反馈
4. **再建立持久化**：记忆和历史是 Agent 从"一次性对话"进化为"持久助手"的关键
5. **最后优化性能**：Prompt Cache、Memoization、并发执行都是在功能稳定后的优化

Claude Code 的源码证明了一个道理：工业级的 AI Agent 不是一个简单的"LLM + 工具"组合，而是一个需要精心设计的软件系统。从 System Prompt 的每一个段落到安全写入的每一个文件标志，每个细节都体现了工程实践的积累和对用户体验的深思。

希望本书能够帮助读者理解这些设计决策背后的思考，并将其应用到自己的 Agent 开发实践中。
