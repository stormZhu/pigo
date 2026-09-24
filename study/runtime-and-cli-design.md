# Pigo 架构与并发设计学习要点

本目录用于沉淀对 `pigo` 核心运行时并发模型与命令行架构的学习笔记。

---

## 目录
- [一、Producer Goroutine 生命周期与退出机制](#一producer-goroutine-生命周期与退出机制)
- [二、为什么为每次 Run 启动独立的 Goroutine](#二为什么为每次-run-启动独立的-goroutine)
- [三、为什么不采用常驻 Worker 协程池](#三为什么不采用常驻-worker-协程池)
- [四、CLI 入口架构：薄 main() 与可测试的 dispatch() 接缝](#四cli-入口架构薄-main-与可测试的-dispatch-接缝)

---

## 一、Producer Goroutine 生命周期与退出机制

### 核心结论
[`StartRun`](file:///Users/yuqing/Documents/workspace/pigo/internal/runtime/loop.go#L135-L137) 启动的 producer goroutine 在**单次 Run 结束或取消后会立即正常退出，不会残留或泄漏**。

### 1. 正常执行完的退出闭环
- **退出条件**：内层循环在当前轮次无工具调用（`len(calls) == 0`）且无后续跟进消息（`GetFollowUpMessages` 为空）且未被 `OnStop` 钩子拦截时跳出。
- **出口收尾（`finish()`）**：
  ```go
  finish := func() {
      _ = emit(tel.summary())
      msgs := newMessages()
      _ = emit(agentcore.AgentEndEvent{Messages: msgs})
      stream.SetResult(msgs)
      stream.Close()
  }
  ```
  1. 发送最终遥测指标事件（[`TelemetryEvent`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/events.go)）；
  2. 发送终止事件（[`AgentEndEvent`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/events.go)）；
  3. [`stream.SetResult(msgs)`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L81-L86) 落地最终结果；
  4. [`stream.Close()`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L100-L108) 关闭底层 channel；
  5. 函数执行完毕返回，goroutine 被 Go 运行时回收。

### 2. 异常与取消保障（防泄漏机制）
- **Channel 背压与取消感知**：
  在 [`EventStream.Emit`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L67-L77) 发送事件时，同时监听了 `ctx.Done()`：
  ```go
  select {
  case s.ch <- event:
      return nil
  case <-ctx.Done():
      return ctx.Err()
  }
  ```
  如果下游消费者提前停止读取或 context 被取消（如用户按下 `Ctrl+C`），`Emit` 不会永久阻塞在 channel 发送端，而是立即返回错误并触发 `finish()` 退出。
- **统一错误收敛**：
  无论是网络异常还是 context 取消，[`runLoop`](file:///Users/yuqing/Documents/workspace/pigo/internal/runtime/loop.go#L141-L293) 在任一环节捕获错误都会直接进入 `finish()` 并 `return`。

---

## 二、为什么为每次 Run 启动独立的 Goroutine

### 1. 替代语言级的 `async generator`
- 在 TypeScript 参考实现（`pi`）中，采用了异步生成器：
  `async *runLoop(...) -> yield event`，外部通过 `for await (const ev of stream)` 迭代。
- Go 语言没有原生生成器机制（`yield`），Go 的惯用替代方案就是 **独立 Goroutine 生产 + Channel（[`EventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L25-L39)）传输**。

### 2. 避免阻塞驱动宿主的主事件循环
一次 Run 耗时极长（多轮 LLM 调用、网络 I/O、本地工具/Shell 执行）：
- **全屏 TUI（Bubble Tea）**：UI 线程必须维持高帧率响应键盘按键与窗口缩放，不能被模型网络 I/O 阻塞。
- **并发子 Agent（SubAgent）**：父 Agent 可以在工具执行中并行拉起多个子任务，每个子任务独立异步推进。

### 3. 支持精确的时序与同步背压（Back-pressure）
- 默认 buffer 为 0 时，生产者发出的每个事件必须等消费者读走才能进入下一步，精确还原了 `await emit(...)` 的时序。

### 4. 事件管道与最终结果解耦
- 过程事件通过 `stream.Events()` 实时推流。
- 最终结果通过 [`stream.Result(ctx)`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L113-L121) 获取。如果消费者仅关心最终输出（如 Headless 纯文本模式），无需遍历全部事件即可直接等待结果。

---

## 三、为什么不采用常驻 Worker 协程池

很多有 Java/C++ 背景的开发者习惯使用“常驻线程池/Worker”来处理任务，但在 Go 中为每次 Run 启动单独协程是更优的设计：

| 考量维度 | 常驻 Worker 方案 | Per-Run Goroutine 方案（当前方案） |
| :--- | :--- | :--- |
| **创建成本** | 传统线程重，必须复用 | Goroutine 极轻量（~2KB 栈，纳秒级拉起），无需复用 |
| **并发与嵌套（SubAgent）** | ⚠️ **极易死锁**：父 Agent 占住 Worker 等待子 Agent，子 Agent 排队等待可用 Worker |  **无死锁风险**：支持任意深度的递归与并发调度 |
| **Channel 语义** | 必须维持长连接，需自定义帧协议与结束标志（EOS） |  **自然闭环**：任务完成直接 `close(ch)`，消费端 `for range` 自然终止 |
| **状态污染与清理** | 每次任务需严密重置各种闭包状态与钩子，易残留脏状态 |  **物理隔离**：每次 Run 拥有独立的局部变量、Context、Telemetry |
| **GC 内存回收** | 长生命周期对象可能延长大对象存活时间 |  **及时回收**：Goroutine 结束栈帧立即销毁，中间对象即刻进入垃圾回收 |

---

## 四、CLI 入口架构：薄 main() 与可测试的 dispatch() 接缝

### 1. `main()` 的极简职责（四步流水线）
[`cmd/pigo/main.go`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main.go) 中的 `main()` 故意设计成不包含任何业务逻辑的“贫血”入口：
1. **解析（Parse）**：使用 `pflag` 将命令行参数解析进纯数据结构 [`cliOptions`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main.go#L59-L130)；
2. **叠加配置（Overlay）**：调用 [`applyFileConfig`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main.go#L303-L364) 合并文件配置（CLI 显式参数 > 配置文件 > 默认值）；
3. **分派（Dispatch）**：把解析好的纯数据对象与标准 I/O 传入 [`dispatch()`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main.go#L368-L450)；
4. **映射退出码（Exit Code）**：
   ```go
   os.Exit(dispatch(context.Background(), opts, os.Stdout, os.Stderr))
   ```

### 2. 为什么 dispatch() 脱离全局 Flag 单独可测
传统 CLI 单元测试的最大痛点是：
- 业务逻辑直接读取全局 flag，测试并发时互相污染；
- 遇到错误直接 `os.Exit()` 导致 `go test` 进程直接崩溃，无法捕获返回值；
- 输出写死 `os.Stdout`，无法方便地在内存中捕获断言。

而在 `pigo` 中，[`dispatch()`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main.go#L368-L450) 签名如下：
```go
func dispatch(ctx context.Context, opts cliOptions, out, errOut io.Writer) int
```
- **输入为纯结构体**：直接在测试里通过 `cliOptions{prompt: "...", outputFmt: "yaml"}` 传入场景，无需重复初始化或重置命令行解析器；
- **输出为通用接口**：测试传入 `&bytes.Buffer` 即可断言标准输出与错误输出；
- **返回值为普通数字**：直接校验退出码 `code == 2`，不会中断测试运行器。
详细示例可参考 [`cmd/pigo/main_test.go`](file:///Users/yuqing/Documents/workspace/pigo/cmd/pigo/main_test.go)。
