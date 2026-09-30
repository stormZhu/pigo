# Pigo 学习笔记

本目录用于记录和归纳 `pigo` 源码研读过程中的核心技术点、设计模式与架构考量。

## 笔记索引

1. [运行时并发模型与 CLI 接缝架构设计](./runtime-and-cli-design.md)
   - Producer Goroutine 生命周期管理与防泄漏机制
   - 为什么为每次 Run 启动独立的 Goroutine（替代 async generator、UI 解耦、背压）
   - 为什么不采用常驻 Worker 协程池（轻量哲学、嵌套 SubAgent 死锁防范、Channel 自然闭环）
   - CLI 入口架构设计：薄 `main()` 与可脱离全局 Flag 单独测试的 `dispatch()` 接缝

2. [工具策略与安全边界架构设计](./tool-policy-and-security.md)
   - 为什么校验必须在插件与记忆工具就位之后（运行期动态名字发现与防御性校验）
   - 为什么过滤发生在注册层（结构性物理阻断：不广播、不可分派）
   - 为什么 `--approve` 无法放宽策略边界（能力层 vs 审计确认层权限隔离）
   - 策略冲突处理：Fail-Closed 原则（Deny 优先于 Allow）

3. [会话持久化时序、SIGPIPE 与信封模式架构设计](./session-persistence-and-sigpipe.md)
   - Unix 管道机制与 `SIGPIPE` 信号陷阱（为什么 `| head` 会杀死进程导致落盘失败）
   - 会话生命周期：会话诞生（`agent_start`）与会话持久化（`hs.persist`）的时序差异
   - 信封模式（Envelope Pattern）：`eventEnvelope` 的统一类型抹平与数据脱敏职责
   - 流式消费最佳实践：`OUT=$(...)` 等待机制与安全续跑

4. [环境装配与运行配置解耦设计](./环境装配与运行配置解耦设计.md)
   - 谁依赖谁：第 1 次创建的 `RunConfig` 是如何完全由 `Env` 哺育初始化的
   - 以谁为准：为什么核心执行永远以 `RunConfig` 为唯一真理（`runLoop` 物理隔离 `Env`）
   - 变与不变：交互中 Model 与 Provider 均可热切换，底层重型资产（Tools/Memory/Cwd）持续锚定 `Env`
   - 为什么不合并为一个大配置（生命周期、资源所有权、架构分层与复用模式对比）
   - 资源释放职责（`defer Close()` 闭环）

5. [大模型工具传递机制与上下文开销优化](./大模型工具传递机制与上下文开销优化.md)
   - 大模型 API 的无状态（Stateless）特性与全量自包含 Payload
   - 工具定义（Schema）对输入 Token 与计费的侵占
   - 现代 Agent 工程优化：Prompt Caching 前缀缓存、渐进式披露（Skills 按需加载）、结构性策略裁剪

6. [行式 REPL 与全屏 TUI 架构对比与设计](./行式REPL与全屏TUI架构对比.md)
   - 概念解析：什么是 REPL、什么是行式流式输出
   - 全屏 TUI（默认） vs 行式 REPL（`--no-tui`）核心特性全景对比
   - 为什么必须保留 `--no-tui` 开关（兼容性、长文本框选复制、极客习惯、非 TTY 降级）
   - 架构本质：TUI 仅作为行式 REPL 的全屏皮肤，底层核心生命周期 100% 复用

7. [密封接口设计与 JSON 多态反序列化](./密封接口设计与JSON多态反序列化.md)
   - 架构基石：为什么 `agentcore` 作为地基包要 "imports nothing from them"（DAG 单向依赖防循环）
   - 变体闭包：密封接口（Sealed Interface 与未导出方法）在 Go 中模拟判别联合
   - 序列化核心痛点：为什么 `encoding/json` 无法反序列化到接口类型（误区澄清：并非私有方法的锅）
   - 两步解码设计（Two-Pass Peek & Dispatch）：先探测 `type` 再二次反序列化到具体 Struct
   - 防御性序列化：针对大模型流式输出非法 JSON 参数的容错与安全落盘保障

8. [上下文压缩与增量检查点架构设计](./上下文压缩与增量检查点架构设计.md)
   - 核心矛盾：长对话上下文爆炸与主流 LLM API 缺失 `role: "compaction"` 的冲突
   - 双重形态伪装：本地以 `CompactionMessage` 一等公民持久化，发送时通过 `AsUserMessage()` 包装 `<summary>` 降阶为用户消息重新入场
   - 职责边界：面向 LLM 认知的非结构化 `Summary` 与面向系统精准审计的强类型 `Details`（`json.RawMessage` 依赖解耦）
   - 级联增量压缩：滑动切片（只总结新增消息）与增量 Diff 提示词（旧摘要 + 新进展），将 Token 消耗严格锁在 $O(1)$ 稳态
   - 零成本文件足迹：基于工具调用的本地静态提取与 Go Map 集合运算，0 磁盘 I/O、0 LLM 总结开销

9. [json.RawMessage 机制与切片零分配复用深度剖析](./json-rawmessage与切片零分配复用.md)
   - 数组切分本质：为什么 `[]byte` 可以通过 `json.Unmarshal` 反序列化为 `[]json.RawMessage`（逐元素延迟拷贝，避免提前 AST 解析）
   - 内存复用黑魔法：`(*m)[0:0]` 长度清零、保留容量、指针不动的切片三元组机制
   - 0 分配性能考量：结合 `append` 在容量充足时达成 0 Heap Allocations，消除 GC 标记压力
   - 防御性安全隔离：为什么坚决不用浅拷贝 `*m = data`（解码器复用内部缓冲区导致的数据污染隐患）
   - 工业级两步解码流水线：在 `pigo` 中作为多态消息与事件反序列化的高性能分拣传送带

10. [多模态 ImageContent 类型选型设计](./多模态ImageContent类型选型设计.md)

- 核心疑问：为什么多模态图片块中的 `Data` 字段选用 `string` 而非 Go 惯用的 `[]byte`
- 协议天然对齐：主流 LLM（OpenAI Data URI、Anthropic Base64 Source）多模态协议的文本化契约与零二次编解码成本
- 序列化暗坑规避：Go `encoding/json` 默认对 `[]byte` 进行 Base64 编码导致的“二次 Base64（Double Base64）”持久化损坏陷阱
- 状态不可变性（Immutability）：`string` 只读特性消除多协程流转（Producer 循环/UI 渲染/网络 发送）中的并发数据竞争（Data Race）与防御性拷贝
- 数据全生命周期：从磁盘边界读入一次性编码，到只读值对象跨协程分发的极简链路

11. [流式增量与累积快照双轨事件架构设计](./流式增量与累积快照双轨事件架构设计.md)

- 核心命题：为什么 `MessageUpdateEvent` 同时承载局部累积快照（`Message`）与底层原始增量（`AssistantMessageEvent`）
- 单轨困境剖析：纯增量（Delta-only）导致端侧被迫维护复杂状态机；纯快照（Snapshot-only）导致打字机流式输出面临无谓的 $O(N)$ 前缀 diff 开销
- 双轨优雅解耦：面向声明式重绘的直接快照渲染，与面向通道分流（Thinking/Text/ToolCall）的精细增量派发
- SDK 双游标前缀差量算法：在 `agent/events.go` 中将内部复合事件平滑转换为外部友好的 `MessageDeltaEvent`
- 完整生命周期时序：从底层 HTTP Chunk 到运行时原子回填（backfill）再到事件广播的因果一致性保障

12. [AgentTool 工具契约与实现全景体系](./AgentTool工具契约与实现全景体系.md)
    - 开放接口哲学：与封闭的密封接口（Content/Message）相反，以首字母全部大写的开放契约达成“万物皆工具”
    - 五大契约方法职责：标识、描述、参数 JSON Schema、执行模式与流式进度回调执行体
    - 读写分离调度：`ToolExecutionMode`（`parallel` vs `sequential`）在批处理执行器中的屏障等待（Barrier Sync）机制
    - 内置工具全景矩阵：文件（read/write/edit）、检索（grep/find/ls）、Shell（bash/output/kill）、网络、记忆与目标控制
    - 高级元工具（Meta Tools）：`SubAgentTool`（子 Agent 循环分身与技能）及 `pluginTool`（外部多语言独立进程 RPC 代理）的透明适配

13. [EventStream 事件通道与结果信箱分离设计](./EventStream事件通道与结果信箱分离设计.md)
    - 思考题溯源：为什么最终结果 `R` 刻意不走事件通道 `ch`，而是用独立的 `resultCh` + `sync.Once` 暴露
    - 核心误区澄清：背压机制下 `ch` 必须被持续消费，否则依然会导致生产者卡死；防泄漏契约（No-leak Contract）的成立前提
    - 单轨合流灾难：若将结果塞入 `ch` 尾端，将引发死锁（直觉等待结果导致起点阻塞）、状态机污染（被迫充当人肉抽水机）与取消脱困失败
    - 关注点分离正解：`DrainStream` 作为通用底层水泵负责静默排空，业务端一行 `stream.Result(ctx)` 优雅一键结算
    - 并发基石：`sync.Once` 先到先得决胜与 `close(resultCh)` 零锁广播唤醒机制

14. [Provider 流式抽象与传输层双驱动设计](./Provider流式抽象与传输层双驱动设计.md)
    - `AssistantMessageEventStream` 如何由泛型 `EventStream` 特化而来（`IsComplete` / `ExtractResult` 注入 Provider 语义）
    - `Emit` 自动捕获结果机制：生产者为何从不需要手写 `SetResult`，终态事件为何仍进通道
    - 生产者统一骨架：建流 → `go pump` → 立即返回，以及 `buffer=0` 同步背压的取舍
    - 消费者两种范式：主循环逐事件驱动 UI（`stream_response.go`）vs 静默排空只取结果（`summary.go` / `apply.go`）
    - 核心对比：`transport.go`（provider-agnostic 传输框架，provider 退化为 `Decoder`）vs `responses.go`（SDK 直连的具体驱动）
    - 为什么 responses 不复用 transport、以及「双失败模型」下二者对上层可观察行为为何一致

15. [SDK 联合类型实现机制与 As 访问器模式](./SDK联合类型实现机制与As访问器模式.md)
    - 问题背景：Go 没有原生 sum type，接口路线 vs 扁平结构体路线各自适用什么场景
    - 五层机制：扁平联合体（字段并集 + `Type` tag + `JSON.raw`）→ `UnmarshalJSON` 留原始字节 → `AsXxx()` 二次解析还原变体 → `AsAny()` 按 tag 分派 → 密封 marker 接口锁定返回类型
    - 关键实现选择：`As` 访问器用 raw JSON 重新反序列化（而非字段搬运），未识别类型 `return nil` 而非 panic
    - 向前兼容设计：嵌套子联合、`ExtraFields` 存未知字段、调用方白名单 switch 与 `nil` 的咬合
    - 与 pigo 自有「接口 + 未导出 marker」方案的逐维度对比，以及选型权衡

16. [StreamFn 统一流式契约与双失败模型设计](./StreamFn统一流式契约与双失败模型设计.md)
    - 函数式契约定位：Agent 循环与 Provider 解耦的极简咽喉要道（窄依赖、易 Mock、闭包适配）
    - 完整使用时序：从 `Provider` 适配到 `LoopConfig.Stream` 挂载，再到 `streamAssistantResponse` 消费
    - 为什么返回值类型是 `AssistantMessageEventStream`：泛型流的 Provider 语义具化（`T` 过程增量 vs `R` 终态结果）
    - 闭环「双失败模型」：建流早期错误返回 `error`，运行期断线化为 `StreamErrorEvent` 随流而下，保障上下文不丢失
    - 安全性与灵活性：防 Goroutine 泄漏的逐事件同步背压，以及 UI 打字机渲染 vs 后台同步等待的双轨消费支持

17. [ToolRegistry 注册表与 JSON Schema 参数校验闭环设计](./ToolRegistry注册表与JSON-Schema参数校验闭环设计.md)
    - 核心结构与并发安全：`tools` 与 `compiled` 双 map 架构，`RWMutex` 保障高频并发读取
    - 预编译机制：为什么“注册即编译”（Fail-Fast 阻断在启动期、消除重复编译、`mem:///` 内存命名空间隔离）
    - 一图两用哲学：对外作为 Function Calling 规格说明书，对内作为参数准入防御网
    - 深入校验引擎：RFC 6901 JSON 指针、为什么缺少 `required` 约束对应空切片并渲染为 `(root)`
    - 模型自我纠错闭环（Self-Correction）：字段级错误包装为 `AgentToolResult`（`Terminate: nil`）回喂上下文

18. [executeToolCall 生命周期与全局 AOP 切面架构设计](./executeToolCall生命周期与全局AOP切面架构设计.md)
    - 四大参数职责分工：`ctx` 存活与前置短路、`cfg` 环境与切面门禁、`call` 调用意图原料、`emit` 外部流式事件总线
    - 返回值语义：不抛 Go Error 的错误实体化原则（保证 LLM 会话因果链不中断）与终止信号
    - 三阶段执行流水线：`prepareToolCall` 五道闸门短路 → `runToolWithRetry` 瞬态重试与增量更新 → `finalizeToolCall` 预算裁剪与消息封装
    - 切面共享哲学：架构上“全局统一管道，绝无漏网之鱼”，业务上入参显式注入工具标识实现“策略按名分流”

19. [会话消息格式演进与单元素数组解码技巧](./会话消息格式演进与单元素数组解码技巧.md)
    - 物理布局：JSONL 一行一对象，第 1 行 header、其后消息行，行格式由 header 的 `version` 决定
    - 两代行格式：v1/v2 裸消息 vs v3+ 包装条目（`Entry`：id/parentId/timestamp/message），演进动机是支撑 `/fork` 与 `/clone` 的树结构
    - 解码难点：`Message` 密封接口无法被 `encoding/json` 直接反序列化，故借 `MessageList` 的 `role` 判别解码器
    - 单元素数组技巧：`"["+raw+"]"` 把单条消息伪装成数组交给 `MessageList`（显式用于 v1/v2 迁移，隐藏于 `Entry.UnmarshalJSON`）
    - 误区澄清：v1/v2 恰恰**不能**直接解码，格式差异只在外壳，内层解码两种格式完全相同
    - 配套细节：Scanner 缓冲放大（64KiB→16MiB）、`sc.Bytes()` 别名陷阱、schema 版本校验、原子写入、`AppendBranch` 分支增长

20. [会话恢复 SystemPrompt 保留策略与落盘时序](./会话恢复SystemPrompt保留策略与落盘时序.md)
    - 冲突点：resume 时旧会话 header 自带的 `SystemPrompt` 与本次调用解析的 `env.SysPrompt` 该以谁为准
    - 保留策略：旧值优先（忠于原会话、避免历史消息与系统提示自相矛盾），仅当旧值为空时回落到本次 `sysPrompt`
    - 落盘链路：回落后的 `hs.header` 经 `persist → AppendBranch → SaveEntries → writeSessionEntries` 重写文件第 1 行，**回落值会固化**（无新消息则不写；`omitempty` 边界）
    - 对照判断标准：`SystemPrompt` 属对话语义 → 保留旧值；`Model` / `Provider` 属描述性元数据 → 强制刷新为本次实际值

21. [headless 驱动与 Hook 装配生命周期设计](./headless驱动与Hook装配生命周期设计.md)
    - 生命周期全景：prompt 解析 → 会话 → 思考强度 → 凭据/RunConfig → Hook 装配 → 触发 → `RunHeadless` → 落盘，以及两步配置错误（exit 2）与运行失败（exit 1）的分野
    - Prompt 三段式：`p.Prompt` → `headlessPrompt`（插件斜杠命令展开）→ `promptContent`（`ContentList` 拆出图片块），Hook 看到的是展开后的最终 prompt
    - 斜杠命令两种兜底语义：未命中/出错 → 原样透传整行；命中且成功但无 prompt → 只留 `args`（命令名属宿主内部语法，不应泄漏给模型；headless 单轮必须有话可跑）
    - 分层配置链：`default < global < project < env < CLI`，指针字段区分「未设置」、字段级替换、缺失文件非错误、CLI 空白视作未设置
    - Hook 三阶段而非一步：解析（信任门控 `ResolveHookSet`）→ 装配（`InstallDriverHooks` 装 seam + SessionStart + 链式通知器，空集零成本）→ 触发（`UserPromptSubmit`）
    - `UserPromptSubmit` 双效果：block 优先（exit 2 / `decision:block` / `continue:false`），`additionalContext` 沦为一次性 reminder 仅注入本轮且不落盘；执行失败与超时一律 fail-open
    - 目录可信（Trust）：三态与「最近祖先」继承、`trust.json` 持久化、`--approve` 会话级；它既 gate 副作用工具确认，也 gate 项目层 hooks 合并；headless 无对话框，等价于要求已持久化的 `true` 记录

22. [SessionID 与 EntryID 架构区分与树状会话模型](./SessionID与EntryID架构区分与树状会话模型.md)
    - 抽象分工：宏观文件级会话生命周期（`SessionID`） vs 微观节点级消息拓扑（`EntryID`）
    - 物理布局：首行 `SessionHeader` 独立检索与持久化 vs 后续行 `Entry` 包装体携带 `ParentID`
    - 生成算法：时间序字典排序（`NewID`） vs 文件内防碰撞 4 字节短哈希（`newEntryID`）
    - 因果树重构：`PathToLeaf` 沿 `ParentID` 链倒查回溯线性上下文，容错截断保障
    - 双轨流转与分叉：运行时 `RunConfig`/首个事件广播 vs 节点追加，以及会话 Fork（`ParentSession`）与分支切片机制

23. [会话中途分叉重新生成与 Fork 物理隔离设计](./会话中途分叉重新生成与Fork物理隔离设计.md)
    - 交互形态：REPL 中的 `/clone`（整树复刻存档）与 `/fork`（列出提问编号中途分叉）
    - 截断机制：以目标提问的 `ParentID` 为截断叶子，`PathToLeaf` 提取前缀切片
    - 物理隔离：分配全新 `SessionID` 并建立独立文件，`ParentSession` 记录父血缘，内存指针无缝换轨
    - 架构决断：对比单文件树状共存，以微小磁盘冗余换取 `--resume` 极简恢复、误删隔离与开箱即用导出

24. [全屏 TUI 事件桥与 MVU 生命周期设计](./全屏TUI事件桥与MVU生命周期设计.md)
    - 整体形态：Bubble Tea v2 的 MVU 状态机 + 把 `AgentEvent` 翻译成 `tea.Msg` 的事件桥，三条硬约束（单写者、事件不丢、表现层可替换）
    - 入口分派：dispatch 的 `shouldUseTUI(isTTY && !--no-tui)` 门控，以及 TUI / REPL 共用同一份 `Options` 与同一个 runtime 内核
    - 启动装配：`newRunSession`（store / live / slash / hooks / trust）→ `withSession` 接线 `startRunFn` / `interruptFn` → `tea.NewProgram.Run()` 进入 alt-screen
    - 事件桥：`pump` goroutine 独占 `agentCtx.Messages`，tea 循环独占 Model，二者只经容量 64 的 channel 通信；`pumpNext` 逐条拉取形成背压且不丢事件
    - Update 分派表：`textDeltaMsg` / `toolStartMsg` / `subagentProgressMsg` / `telemetryMsg` / `runEndMsg` 等全部桥消息的落点，以及 `turnEndMsg` 兜底与 `runEndMsg` 无竞态的时序依据
    - 提交生命周期：斜杠命令先行拦截 → 占位符还原（`expandPastes` / `expandImages`）→ Hook 在落 context 前触发 → `context.WithCancel` + 两段式中断 → 落盘
    - 落盘双模式：常规走 `AppendBranch` 分支追加保留会话树；compaction 后退化线性重写并重置分支游标
    - 布局与渲染：`relayout` 先扣固定开销再给 transcript，`follow` 语义、`applySelection` 后置覆盖与自适应滚动条


