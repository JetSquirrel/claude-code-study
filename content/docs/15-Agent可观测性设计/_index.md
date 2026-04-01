---
title: "第十五章：Agent 可观测性设计"
weight: 150
bookCollapseSection: false
---

# 第十五章：Agent 可观测性设计

> 从 Claude Code 源码学习如何系统性地设计 Agent 可观测性架构

## 写在前面

在所有主流 Agent Runtime 的源码中，Claude Code 在可观测性架构上的设计最为系统和深入。它不仅仅是"打日志 + 统计接口耗时"，而是对 Agent 运行时的语义做了完整的建模。本章将深入剖析 Claude Code 可观测性设计的四个核心特征：

1. **三层可观测性架构** - 产品事件、标准化 Telemetry、会话级 Tracing 各司其职
2. **统一关联键系统** - session、agent、parent 的完整关联链
3. **Agent 语义级 Span** - 把 Agent Runtime 的语义映射成 Span 类型
4. **信号类型的清晰边界** - Event、Metric、Trace 各归各位，不互相越界

## 15.1 边界说明

### 15.1.1 分析性分组 vs 源码类型

本章中的一些分组（如"产品事件"和"运行时事件"）是为了讲清观测目标而做的**分析性划分**，并非 Claude Code 源码中的正式类型名。源码中真正可以直接看到的是：

- `services/analytics/*` - 事件日志管线
- `utils/telemetry/*` - OpenTelemetry / Tracing 管线
- 大量以 `tengu_` 开头的事件名
- `interaction`、`llm_request`、`tool` 等 Span 名称

> **术语说明：** `tengu` (天狗) 是日本民间信仰中的神秘生物，属于神道中的神灵（kami）或妖怪（yōkai）。Claude Code 使用此名称作为事件命名前缀。

### 15.1.2 Tracing 能力的条件开启

需要注意的是，Tracing 能力不是无条件常开的：

- `sessionTracing.ts` 受 enhanced telemetry / beta tracing 条件控制
- Hook span 只在 beta tracing 打开时创建
- Perfetto 是独立的本地 opt-in tracing

因此，本章讨论"运行时事件"时，核心是在讲 **Claude Code 如何观测 Agent Runtime**，而不是说源码里存在一个叫 `RuntimeEvent` 的统一类型系统。

## 15.2 三层可观测性架构

Claude Code 的可观测性不是单一的日志系统，而是被精心设计成三个独立的层次。

### 15.2.1 第一层：产品分析事件

**入口：** `services/analytics/index.ts`

这个模块刻意保持无依赖，提供统一的入口：

```typescript
// 统一入口
logEvent() / logEventAsync()
```

关键设计特点：

1. **Sink 延迟装配** - 在 `sink.ts` 中，真正的后端路由会分流到 Datadog 和 1P event logging
2. **启动期队列** - Sink 尚未挂载时，事件会先入队，避免丢失
3. **无依赖模块** - 调用方不知道后端是 Datadog 还是 1P，完全解耦

**典型事件：**
- `tengu_oauth_flow_start` - OAuth 流程开始
- `tengu_plugin_install_command` - 插件安装命令
- `tengu_input_prompt` - 用户输入提示
- `tengu_session_resumed` - 会话恢复

### 15.2.2 第二层：标准化 Telemetry

**入口：** `utils/telemetry/instrumentation.ts`

使用 OpenTelemetry 的标准化体系：
- **Metrics** - 指标数据
- **Logs** - 日志记录
- **Traces** - 追踪数据

**Exporter 配置：**
- OTLP (OpenTelemetry Protocol)
- Prometheus
- Console
- 内部指标导出器

这是"基础设施观测层"，强调标准生态兼容性。通过标准化接口，可以轻松集成到现有的可观测性平台。

### 15.2.3 第三层：会话级 Tracing

**核心模块：** `utils/telemetry/sessionTracing.ts`

这一层不追踪网络请求，而是**直接对 Claude Code 的业务语义建模**：

- `interaction` - 用户交互回合
- `llm_request` - LLM 请求
- `tool` - 工具调用
- `hook` - 钩子执行
- `blocked_on_user` - 等待用户批准的时间

把"Agent 在做什么"表达成 Trace 结构，这是理解 Agent 行为的关键。

### 15.2.4 三层架构的职责划分

```
┌──────────────────────────────────────────────────────────────┐
│                  analytics/*                                  │
│              回答"发生了什么产品事件"                           │
│         产品经理和运营团队关注的指标                            │
└──────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────────────────────────────────────────────┐
│            telemetry/instrumentation.ts                       │
│         回答"指标和标准 traces 怎么接入后端"                   │
│              标准化的可观测性基础设施                           │
└──────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────────────────────────────────────────────┐
│               sessionTracing.ts                               │
│         回答"一个 agent turn 在 runtime 里怎么展开"            │
│           工程师和 SRE 关注的执行细节                          │
└──────────────────────────────────────────────────────────────┘
```

## 15.3 为什么不能混在一起？

Agent 可观测性不能追求"全都记下来"，必须先做目标分层。否则会踩三个坑：

### 15.3.1 命名体系失控

`user_clicked_button` 和 `model_fallback_triggered` 出现在同一个 event stream 里，没人分得清这条 event 该归产品分析还是归工程排查。

### 15.3.2 后端 Schema 和 Cardinality 爆炸

Agent 可观测数据的特征：
- **半结构化** - 字段不固定
- **高维度** - 几十上百个字段
- **高基数** - 大量唯一值

一个典型 Agent 执行事件可能有几十上百个字段。不分层管理的话，后端被维度组合淹没只是时间问题。

### 15.3.3 看到日志但看不清因果链

一个 user turn 可能拆成：
- 多次 LLM request
- 多个 tool call
- 多个 subagent

数据平铺在一起，排查时只能靠 grep 和人肉拼凑。

### 15.3.4 产品事件 vs 运行时事件

在 Claude Code 里，这两类事件的边界不是绝对的，但目标很不同：

**产品事件：** 回答"用户做了什么"
- `tengu_oauth_flow_start`
- `tengu_plugin_install_command`
- `tengu_input_prompt`
- `tengu_session_resumed`

**运行时事件：** 回答"一个 turn 怎么跑起来的"
- Tracing Span: `claude_code.interaction`, `claude_code.llm_request`, `claude_code.tool`
- Runtime Events: `tengu_model_fallback_triggered`, `tengu_auto_compact_succeeded`, `tengu_tool_use_can_use_tool_rejected`

更准确的理解：**两种不同的观测目标用三种不同的信号形态来承载**。这种"观测目标"和"信号形态"的双维度设计，是 Claude Code 可观测性架构里第一个可以直接借鉴的决策。

### 15.3.5 工程细节：入口去耦

`analytics/index.ts` 的工程化设计：

```typescript
// 公共入口 - 无依赖模块 + 启动期队列
logEvent(eventName, metadata);

// 调用方只关心接口，不知道后端是什么
// Sink 可以延迟装配
// 启动早期的事件不会丢
// 不引入 import cycle
```

对大项目来说，这种**入口去耦**往往比"换了什么 observability 平台"更重要。

## 15.4 统一关联键：串起同一个实体

Agent 系统里，真正困难的不是"生成一条事件"，而是**"让不同事件能被关联起来"**。

### 15.4.1 跨系统关联的痛点

典型场景：
- `trace_id` 在 Jaeger 里
- Token 用量在 Prometheus 里
- 对话内容在 Elasticsearch 里

定位一个问题要跨三个界面来回跳。把三类信号存在同一个库里、用一条 SQL 关联查询，是 LLM 可观测性最该优先解决的问题。

Agent 系统比单次 LLM 调用复杂得多，对关联能力的需求只会更强。

### 15.4.2 Claude Code 的关联键设计

Claude Code 在关联键这件事上做得克制，集中在两处：

1. **`telemetryAttributes.ts`** - 偏 OTel attributes
2. **`metadata.ts`** - 偏 analytics / 1P event metadata

#### OTel 侧的基础关联键

`getTelemetryAttributes()` 统一挂一批跨信号共享的基础属性：

```typescript
// 基础属性
- user.id
- session.id
- organization.id
- terminal.type
// ... 等等
```

**关键设计：主动管理 Cardinality**

不是一股脑全开。`session.id`、版本号、account UUID 这些字段是否带上，受 `OTEL_METRICS_INCLUDE_*` 环境变量控制。

高基数字段不是默认全塞进指标系统的。这是在主动控制数据规模和查询性能。

#### Analytics 侧的核心上下文

`metadata.ts` 的 `getEventMetadata()` 补了另一层上下文：

```typescript
// 事件元数据
sessionId           // 会话ID
model              // 使用的模型
userType           // 用户类型
betas              // Beta 特性列表
entrypoint         // 入口点
clientType         // 客户端类型
isInteractive      // 是否交互式
envContext         // 环境上下文
processMetrics     // 进程指标
subscriptionType   // 订阅类型
repo remote hash   // 仓库远程哈希
```

`envContext` 和 `processMetrics` 特别值得注意。Claude Code 把 analytics 当**轻量运行时事实采集**用，终端类型、平台、CI 状态、CPU/内存等信息跟着事件一起出去，不只是埋点计数。

### 15.4.3 Agent / Parent / Team 的关联链

对 Agent 系统更关键的一层是"**谁是谁的子节点**"。

`metadata.ts` 补了这组字段：

```typescript
agentId            // 当前 Agent ID
parentSessionId    // 父会话 ID
agentType          // Agent 类型
teamName           // 团队名称
```

**取值优先级：**
1. AsyncLocalStorage 中的 agent context
2. Swarm / teammate 环境
3. Bootstrap state 中的 parent session

#### 为什么这么复杂？

同一个系统里，agent 既可能是：
- 同进程 subagent
- Swarm teammate
- Standalone agent

可观测性层不能假定只有一种 agent 身份来源，必须自己做**统一归并**。

归并之后，后端拿到的不是一堆孤立事件，而是一个**最小可关联图**：
- 当前 session 是谁
- 当前 agent 是谁
- 上游 parent session 是谁
- 它是 teammate / subagent / standalone
- 属于哪个 team

排查多代理问题时这很关键。不然你只看到"调了工具"和"发了请求"，不知道动作属于：
- 主会话
- 哪个子代理
- 还是某个团队 agent

这也是 Multi-Agent 可观测性的核心痛点：**跨边界的 Context 传播**。

Claude Code 的解法不算万能，但至少在单 runtime 内做到了一致的关联模型。

### 15.4.4 工程细节：Schema 漂移前置到类型层

`metadata.ts` 在 `to1PEventFormat()` 里有一段关键注释：

> env 使用 proto-generated EnvironmentMetadata 类型，不是手写并行类型。

**原因：**

手写平行类型的风险是：
- 新增字段在本地对象里存在
- 但被生成的 `toJSON()` 静默丢掉
- 你以为打点成功，后端根本没收到

**解决方案：**

Event schema 和 proto schema 绑定之后，**schema drift 在编译期暴露**，不用上线后靠数仓排查。

## 15.5 Agent 语义级 Span：Trace 一个 Turn，不是 Trace 一个函数

如果只看 OpenTelemetry 接入，很容易以为 tracing 的重点是 exporter 和 provider 怎么配。但 `sessionTracing.ts` 才是真正有意思的部分。

### 15.5.1 Span 类型的语义建模

这个模块的关键在于它把 Claude Code 的业务语义直接编码成了 Span 类型：

```
interaction              - 一次用户交互回合 (root span)
  ├─ llm_request        - 一次模型调用
  ├─ tool               - 一次工具调用
  │   ├─ tool.blocked_on_user   - 等待用户批准
  │   └─ tool.execution         - 工具实际执行
  └─ hook               - 钩子执行 (仅 beta tracing)
```

### 15.5.2 Interaction：Root Span 是"用户的一次交互"

`startInteractionSpan()` 把从用户输入到 Claude 响应这一整轮当成 root span，创建 `claude_code.interaction`。

**为什么选择 interaction 作为根节点？**

可观测性的基本单位是"用户的一次交互回合"，不是某次网络请求。

对 Agent 尤其重要。一次 turn 通常包含：
- Prompt 处理
- 多次模型调用
- 多个工具执行
- 可能的人工确认
- Hook 和恢复逻辑

没有 interaction 作为根节点，后面所有 request / tool span 碎成平铺事件，读 trace 时完全看不出这轮到底发生了什么。

**反面案例：**

想象在 Grafana 里看到 100 个独立 span，不知道哪些属于同一轮对话。很多 Agent 系统做 tracing 就是这个状态。

### 15.5.3 LLM Request：显式区分模型调用

`startLLMRequestSpan()` 创建 `claude_code.llm_request`，带上：
- 模型名称
- 速度模式
- 上下文来源
- Token 统计
- Cache 信息
- TTFT (Time To First Token)

**两个设计点：**

1. **记录 `llm_request.context`** - 区分请求是挂在 interaction 下面还是独立的 standalone request
   - 有的模型请求属于一个用户 turn
   - 有的是系统内部附属请求
   - 两者不混淆

2. **并行请求问题** - `endLLMRequestSpan()` 的注释提到：
   ```
   Warmup、topic classifier、主线程请求可能同时在飞
   结束时必须传回准确的 span 实例
   否则响应会被错误归因
   ```

这不是纸面 tracing，是在处理 Agent Runtime 真实的**并发归因**。

### 15.5.4 Tool 与 Tool.Execution：把两个阶段拆开

Tool span 和 tool.execution span 并存。工具作为 Agent 决策的一部分被调用，和工具内部真正进入执行阶段，被拆成了两个 span。

在 `toolExecution.ts` 中：
```
startToolSpan()
  → startToolBlockedOnUserSpan()
  → startToolExecutionSpan()
```

按实际运行顺序串起来。执行链上显式切分了**"被选中"、"等待批准"、"真正执行"**三个阶段。

不用事后从日志去猜"大概在等权限"。

这让 trace 能回答更细的问题：
- 工具在等权限时花了多少时间？
- 从被模型选中到开始执行，中间卡在哪？

### 15.5.5 Tool.Blocked_on_User：把人在回路的时间独立出来

这个 span 单独拎出来说。

很多 Agent 平台做 tracing 只追踪模型延迟和工具延迟，但 Claude Code 还单独追踪**"等用户批准的时间"**。

**这等于承认了一个事实：**

Agent 的真实耗时不全是机器耗时。**权限审批和人工中断本身就是主链路的一部分**。

把这类时间独立出来之后，你才能回答：
- 慢是因为模型慢，还是审批慢？
- 某类工具成功率低，是执行问题还是审批被拒绝太多？
- 一个 turn 很长，时间主要花在 Agent 计算上还是等人上?

**实际案例：**

在 TMA (Trace-based Multi-Agent Analysis) 系统中做 session 分析时，经常碰到这种情况：

一个 session 看起来跑了很长，实际大量时间花在等用户点"允许"。Trace 里没这个维度的话，延迟分析永远是错的。

### 15.5.6 AsyncLocalStorage：上下文传播的实用选择

实现上，`sessionTracing.ts` 用 AsyncLocalStorage 保存：
- `interactionContext` - 回合根上下文
- `toolContext` - 工具链局部上下文

后续的 `llm_request`、`tool.execution`、`hook` 自动挂到正确父节点下。

**为什么选择 AsyncLocalStorage？**

Agent 系统里这是很实用的选择。一旦引入：
- Streaming
- Hook
- 后台任务
- 工具嵌套
- 多代理

靠参数手传 parent trace 很快就控制不住。

### 15.5.7 工程细节

#### 1. 并发归因

`endLLMRequestSpan()` 明确提醒调用方：

```typescript
/**
 * IMPORTANT: When multiple LLM requests run in parallel
 * (e.g., warmup requests, topic classifier, file path extractor, main thread),
 * you MUST pass the specific span to ensure responses are attached
 * to the correct request.
 *
 * Without it, responses may be incorrectly attached to whichever span
 * happens to be "last" in the activeSpans map.
 *
 * If not provided, falls back to finding the most recent llm_request span
 * (legacy behavior).
 */
```

多请求并发时必须传回原始 span 实例，否则响应挂到错误请求上。

#### 2. 悬挂 Span 清理

`activeSpans` 结合：
- WeakRef
- AsyncLocalStorage
- 30 分钟 TTL cleanup

避免 tracing 自身变成长会话的内存负担。

Agent 执行链长、异常路径多、用户中断频繁，不处理 orphaned span 的话，trace 本身就是内存泄漏源。

源码里 `ensureCleanupInterval()` 的注释写得很直接：

> This is a safety net for spans that were never ended.

- Aborted streams
- Uncaught exceptions
- Mid-query 中断

这些在 Agent Runtime 里不是边缘情况，是**常态**。

能不能正确处理它们，是"demo 级 tracing"和"生产级 tracing"的分水岭。

## 15.6 Event / Metric / Trace 各归各位

Claude Code 没有把所有信号都压成"事件"，这一点要说一下。

### 15.6.1 两个维度的分组

先理清概念：

- **观测目标维度：** 产品事件 / 运行时事件
- **信号形态维度：** Event / Metric / Trace

前者问"你想观测什么"，后者问"你用什么信号承载"。

Claude Code 的做法是**把这两个维度分开处理**，不搅在一起。

### 15.6.2 Event：发生了什么

**负责模块：**
- `analytics/index.ts`
- `sink.ts`
- `firstPartyEventLogger.ts`

关注某类行为是否发生、发生时的上下文、送往哪个后端。

**最重要的设计不是路由，是字段约束：**

```typescript
// logEvent() 的 metadata 被限制成数值和布尔值
// 字符串要进来，必须显式通过一个名字极长的标记类型
AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
```

这名字看着像玩笑，但意图很认真：**在类型层就限制误报敏感信息**，不靠事后代码评审。

**`_PROTO_*` 机制：**

把"允许进入 1P 特权字段的值"和"绝不能进入通用后端的值"分成两条路径。

`sink.ts` 在 Datadog fanout 前统一调 `stripProtoFields()`，Datadog 永远看不到 `_PROTO_*` 字段。

**一个过滤点管住所有 sink**，比各写一份脱敏逻辑靠谱。

### 15.6.3 Trace：怎么串起来的

Trace 由 `sessionTracing.ts` 建模、`instrumentation.ts` 装配。

在 Claude Code 语义层上：
- `interaction` 是根
- `llm_request` 和 `tool` 是子动作
- `tool.execution` / `blocked_on_user` / `hook` 挂更细的上下文

**Trace 在这里是因果结构图，不是换了个名字的事件日志。**

### 15.6.4 Metric：整体状态

Metric 在 `instrumentation.ts` 和 `bootstrap/state.ts` 侧，承载更聚合的状态量：

- Session 数量
- 代码修改量
- 成本统计
- Token 统计

### 15.6.5 三者的区别

在 Agent 系统里尤其明显：
- **Event** - 看离散行为
- **Trace** - 看因果链
- **Metric** - 看趋势

不分层的话：
- 要么什么都塞 trace 导致 trace 爆炸
- 要么什么都打 event 之后只能写 SQL 拼执行过程

### 15.6.6 Taxonomy 也是隐私治理

Claude Code 的信号分类带着明显的隐私意识。

`metadata.ts` 里有一整套 MCP 工具名处理逻辑：

```typescript
// 默认把 mcp__ 前缀的工具名进行处理
// 避免泄露用户的 MCP 服务器配置信息
```

可观测性不只是技术问题，也是隐私和安全问题。

## 15.7 可借鉴的设计原则

从 Claude Code 的可观测性设计中，我们可以提取出以下可直接应用的原则：

### 15.7.1 分层先于分析

在开始收集数据之前，先明确：
1. 产品事件 vs 运行时事件
2. Event vs Metric vs Trace
3. 哪些数据给产品看，哪些数据给工程看

### 15.7.2 关联键是一等公民

设计可观测性系统时，关联键的设计应该和事件本身一样重要：
- sessionId 串联会话
- agentId 标识代理身份
- parentSessionId 构建调用链
- teamName 标识协作关系

### 15.7.3 语义优先于技术

不要 trace 函数调用，要 trace 业务语义：
- 不是 `executeFunction()`，而是 `interaction`
- 不是 `apiCall()`，而是 `llm_request`
- 不是 `runTool()`，而是 `tool.execution`

### 15.7.4 Cardinality 是成本问题

高基数字段不是默认全塞进指标系统的：
- 用环境变量控制包含哪些高基数字段
- 在类型层限制敏感字段
- 用 proto schema 避免 schema drift

### 15.7.5 人在回路是一等延迟

Agent 系统的延迟不只包括：
- 模型推理时间
- 工具执行时间

还包括：
- 等待用户批准
- 人工中断
- 权限审批

这些必须被显式建模和追踪。

### 15.7.6 异常路径是常态

生产级 tracing 必须处理：
- Aborted streams
- Uncaught exceptions
- Mid-query 中断
- Orphaned spans
- Memory cleanup

这些不是边缘情况，是 Agent Runtime 的日常。

## 15.8 小结

Claude Code 的可观测性设计给我们最大的启示是：**可观测性不是事后补充的监控，而是架构设计的一部分**。

从项目一开始就要明确：
1. **观测什么** - 产品行为 vs 运行时行为
2. **用什么承载** - Event vs Metric vs Trace
3. **怎么关联** - session / agent / parent 的统一关联键
4. **怎么建模** - 业务语义级的 span 设计

只有系统性地思考这些问题，才能构建出真正可用的 Agent 可观测性系统，而不是一堆看不懂的日志文件。

---

**延伸阅读：**
- 第七章：Agent 与多智能体系统 - 了解 agent 层次结构
- 第十二章：记忆、持久化与会话管理 - 了解会话管理机制
- 第十四章：设计模式与 Agent 开发实践 - 了解工程实践
