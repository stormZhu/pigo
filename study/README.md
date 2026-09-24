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
