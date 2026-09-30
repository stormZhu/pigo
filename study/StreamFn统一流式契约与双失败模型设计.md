# StreamFn 统一流式契约与双失败模型设计

本文深入剖析 `pigo` 中 Agent 核心循环与模型 Provider 之间的核心交互枢纽：[`StreamFn`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L118) 函数类型，以及其返回值 [`*AssistantMessageEventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L76) 的选型原因与双失败模型机制。

---

## 目录

- [一、核心签名与设计定位](#一核心签名与设计定位)
- [二、StreamFn 的完整使用链路](#二streamfn-的完整使用链路)
  - [1. 生产端：Provider 接口适配为函数值](#1-生产端provider-接口适配为函数值)
  - [2. 装配端：注入 LoopConfig 配置](#2-装配端注入-loopconfig-配置)
  - [3. 调用端：streamAssistantResponse 发起请求](#3-调用端streamassistantresponse-发起请求)
  - [4. 消费端：双轨消费（事件流与终态快照）](#4-消费端双轨消费事件流与终态快照)
- [三、深度剖析：为什么返回值类型是 AssistantMessageEventStream](#三深度剖析为什么返回值类型是-assistantmessageeventstream)
  - [1. 泛型流的 Provider 语义具化：过程事件 (T) 与终态结果 (R)](#1-泛型流的-provider-语义具化过程事件-t-与终态结果-r)
  - [2. 闭环「双失败模型」：保障断线上下文不丢失](#2-闭环双失败模型保障断线上下文不丢失)
  - [3. 防 Goroutine 泄漏与逐事件同步背压](#3-防-goroutine-泄漏与逐事件同步背压)
  - [4. 双轨消费模式：UI 实时渲染 vs 后台同步等待](#4-双轨消费模式ui-实时渲染-vs-后台同步等待)
- [四、架构全景接线图](#四架构全景接线图)
- [五、源码索引](#五源码索引)

---

## 一、核心签名与设计定位

在 [`internal/provider/provider.go`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L118) 中，`StreamFn` 被定义为一个极简的纯函数类型：

```go
type StreamFn func(ctx context.Context, model string, llm LlmContext, cfg StreamConfig) (*AssistantMessageEventStream, error)
```

### 1. 为什么是函数（Functional）而不是接口（Interface）？
- **最窄依赖原则**：核心 Agent 循环（`internal/runtime`）只需要执行“发起流式补全”这一件事，它不需要关心 Provider 的注册机制、可用模型列表（`Models()`）或者厂商名称（`Name()`）。函数类型让调用方依赖最小化。
- **极简适配与 Mock 测试**：在测试或特殊控制场景下（如子代理 RPC、测试打桩 `fauxProvider`），无需实现完整的 Provider 接口，只需手写一个闭包函数即可注入。

---

## 二、StreamFn 的完整使用链路

```
[具体 Provider (OpenAI/Anthropic/Ollama)]
         │
         ▼ StreamFnFromProvider()
   [StreamFn 闭包]
         │
         ▼ 注入 LoopConfig.Stream
[streamAssistantResponse (运行时循环)]
         │
         ▼ cfg.Stream(...) 触发建流
[*AssistantMessageEventStream]
    ├──> stream.Events() ──> 实时推送到 UI/打字机，更新 agentCtx.Messages
    └──> stream.Result() ──> 获取最终 AssistantMessage 写入历史会话
```

### 1. 生产端：Provider 接口适配为函数值

面向对象的 [`Provider`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider_interface.go#L61) 接口通过 [`StreamFnFromProvider`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider_interface.go#L75) 转换为 `StreamFn`：

```go
// internal/provider/provider_interface.go
func StreamFnFromProvider(p Provider) StreamFn {
    return func(ctx context.Context, model string, llm LlmContext, cfg StreamConfig) (*AssistantMessageEventStream, error) {
        return p.StreamCompletion(ctx, CompletionRequest{Model: model, Context: llm, Config: cfg})
    }
}
```

### 2. 装配端：注入 LoopConfig 配置

在 CLI 启动、REPL、TUI 或 SubAgent 派生时（如 [`internal/cli/repl/repl.go`](file:///Users/yuqing/Documents/workspace/pigo/internal/cli/repl/repl.go#L598)），通过适配器装配进循环参数：

```go
cfg := runtime.LoopConfig{
    Model:    "claude-3-7-sonnet",
    Provider: "anthropic",
    Stream:   provider.StreamFnFromProvider(prov),
    // ...
}
```

### 3. 调用端：streamAssistantResponse 发起请求

在单轮交互内部，[`streamAssistantResponse`](file:///Users/yuqing/Documents/workspace/pigo/internal/runtime/stream_response.go#L67) 组织上下文并调用 `StreamFn`：

```go
// 准备系统提示词、过滤后的历史消息与工具定义
llm := provider.LlmContext{
    SystemPrompt: agentCtx.SystemPrompt,
    Messages:     msgs,
    Tools:        agentCtx.Tools,
}

// 调用 StreamFn
stream, err := cfg.Stream(ctx, cfg.Model, llm, provider.StreamConfig{
    APIKey:        key,
    ThinkingLevel: cfg.ThinkingLevel,
    Extra:         cfg.Extra,
})
if err != nil {
    // 仅用于捕获“连流都建不起来”的早期参数/配置错误
    return newErrorAssistantMessage(cfg, err), nil
}
```

### 4. 消费端：双轨消费（事件流与终态快照）

```go
for ev := range stream.Events() {
    switch e := ev.(type) {
    case provider.StreamStartEvent:
        // 收到消息首个增量，开始在上下文占位
    case provider.StreamTextEvent:
        // 收到正文流（Delta 增量与 Partial 快照）
    case provider.StreamThinkingEvent:
        // 收到思考链（Reasoning）增量
    case provider.StreamToolCallEvent:
        // 收到工具调用参数拼接
    case provider.StreamDoneEvent:
        // 正常结束，获得终态消息
        return e.Message, nil
    case provider.StreamErrorEvent:
        // 异常终止，携带中断前累积的局部消息及 StopReason
        return e.Message, nil
    }
}
```

---

## 三、深度剖析：为什么返回值类型是 AssistantMessageEventStream

许多流式库直接返回 Go 原生只读通道 `<-chan Event`、回调函数 `func(event)` 或者 `io.Reader`。`pigo` 选择专门封装并具化为 `*AssistantMessageEventStream`，有深层的工程权衡：

### 1. 泛型流的 Provider 语义具化：过程事件 (T) 与终态结果 (R)

[`AssistantMessageEventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L76) 是底层通用泛型流 [`agentcore.EventStream[T, R]`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L25) 的特化类型别名：

```go
type AssistantMessageEventStream = agentcore.EventStream[AssistantMessageEvent, agentcore.AssistantMessage]
```

它将抽象的 `T` 与 `R` 绑定为：
- **`T: AssistantMessageEvent`（过程增量）**：包含 `start`、`text`、`thinking`、`toolcall` 等密封事件，承载增量流，供实时渲染与打字机输出。
- **`R: agentcore.AssistantMessage`（终态结果）**：代表整段流结束后的聚合实体，直接对应持久化存储的消息结构。

### 2. 闭环「双失败模型」：保障断线上下文不丢失

在大模型 Agent 场景下，**错误不全是瞬间发生的，很多发生在流式传输中途**。例如：
- 模型已经生成了思考过程（Thinking），正文输出了前 200 字，随后上游网络超时或服务端 500 断开。
- 模型发起了工具调用（ToolCall），参数生成到一半时连接断开。

#### 传统返回 `(..., error)` 的缺陷：
若将运行期断线直接作为 Go `error` 返回，调用栈会立即展开，**此前已经接收到的思考链、部分文本或部分工具调用将全部丢弃**。这在 Agent 记忆与调试中是不可接受的。

#### 双失败模型的设计：
1. **函数返回的 `error`**：契约写明仅用于**流尚未建立时的早期硬错误**（如缺少必需参数、鉴权头无法构造）。
2. **运行期故障随流而下**：一旦流启动，一切网络抖动、协议损坏、上游 HTTP 500、双看门狗超时，**绝不作为函数 error 返回**，而是转化为密封事件 [`StreamErrorEvent`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L60)。

在 [`NewAssistantMessageEventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L80) 构造函数中，通过注入 Provider 语义闭环这一模型：
```go
s.IsComplete = func(e AssistantMessageEvent) bool {
    k := e.EventKind()
    return k == StreamEventDone || k == StreamEventError // 无论成功 Done 还是失败 Error，皆为流的完结
}
s.ExtractResult = func(e AssistantMessageEvent) agentcore.AssistantMessage {
    switch ev := e.(type) {
    case StreamDoneEvent:  return ev.Message
    case StreamErrorEvent: return ev.Message // 即使是 Error，仍提取包含截至断线前已累积消息与 StopReasonError 的终态消息
    }
}
```
这样，即使发生异常，下游依然能通过 `IsComplete -> ExtractResult -> Result()` 拿到一条携带 `StopReason: "error"` 与错误信息的有效 `AssistantMessage`，保证了消息历史的完整性与因果链。

### 3. 防 Goroutine 泄漏与逐事件同步背压

如果单纯使用 Go 原生通道 `<-chan Event`：
- **背压不可控**：无缓冲通道发送若无接收者会死锁；带缓冲通道容易掩盖下游消费过慢的问题。
- **Goroutine 泄漏风险**：若消费端发生 panic 或因为逻辑提前 `break` 退出循环，底层的网络读取泵（pump goroutine）向 channel 发送数据将永远阻塞无法退出。

[`EventStream.Emit`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L67) 强制通过 `select` 监听 `ctx.Done()`：
```go
select {
case s.ch <- event:
    return nil
case <-ctx.Done():
    return ctx.Err()
}
```
配合 `buffer: 0`，既实现了端到端的逐事件同步背压（模拟类似 TypeScript 中的 `await emit(...)`），又确保消费者取消时底层 goroutine 安全回收。

### 4. 双轨消费模式：UI 实时渲染 vs 后台同步等待

同一个流可以在不同消费场景中自然适配：
1. **交互式主循环（逐事件消费）**：
   通过 `for ev := range stream.Events()` 实时推送到终端显示或前端 UI，实现打字机效果。
2. **非交互式后台任务（只要结果）**：
   在上下文压缩（`summary.go`）或记忆沉淀（`apply.go` 的 `drainToMessage`）等场景中，调用方无需打字机效果。此时调用方既可以排空通道，也可以直接调用：
   ```go
   finalMsg, err := stream.Result(ctx)
   ```
   直接同步等待流结算后的终态 `AssistantMessage`，免去消费端自己编写状态机拼接字符串和解析工具参数的冗余开销。

---

## 四、架构全景接线图

```
┌────────────────────────────────────────────────────────┐
│                      Agent Loop                        │
│                (streamAssistantResponse)               │
└──────────────────────────┬─────────────────────────────┘
                           │ 1. StreamFn(ctx, model, llm, cfg)
                           ▼
┌────────────────────────────────────────────────────────┐
│                   StreamFn Adapter                     │
│               (StreamFnFromProvider)                   │
└──────────────────────────┬─────────────────────────────┘
                           │ 2. StreamCompletion(ctx, req)
                           ▼
┌────────────────────────────────────────────────────────┐
│             Concrete Provider Driver                   │
│   (openAICompatDriver / anthropicCompatDriver / etc.)  │
└─────────────┬────────────────────────────┬─────────────┘
              │ 3. 创建特化流                │ 4. 异步 pump
              ▼                            ▼
┌──────────────────────────┐    ┌────────────────────────┐
│AssistantMessageEventStream│<───│  HTTP SSE / SDK Stream │
│  - Events() <-chan       │    │  (Decoder 状态机解析)   │
│  - Result() AssistantMsg │    └────────────────────────┘
└─────────────┬────────────┘
              │ 5. 双轨道分发
              ├──> range Events(): StreamStart/Text/Tool/Error
              └──> Result(): 终态聚合结果 (即使失败亦有快照)
```

---

## 五、源码索引

| 概念 / 类型 | 定义位置 | 核心职责 |
| :--- | :--- | :--- |
| [`StreamFn`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L118) | `internal/provider/provider.go` | Agent 循环与 Provider 间的函数式统一契约 |
| [`Provider`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider_interface.go#L61) | `internal/provider/provider_interface.go` | 提供商面向对象核心接口 |
| [`StreamFnFromProvider`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider_interface.go#L75) | `internal/provider/provider_interface.go` | `Provider` 到 `StreamFn` 的直连适配器 |
| [`AssistantMessageEventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L76) | `internal/provider/provider.go` | 具化泛型事件流（Provider 专属类型别名） |
| [`NewAssistantMessageEventStream`](file:///Users/yuqing/Documents/workspace/pigo/internal/provider/provider.go#L80) | `internal/provider/provider.go` | 注入 `IsComplete` 与 `ExtractResult` 的流工厂 |
| [`EventStream[T, R]`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/event_stream.go#L25) | `internal/agentcore/event_stream.go` | 底层带背压、防泄漏、通道与信箱分离的泛型流 |
| [`streamAssistantResponse`](file:///Users/yuqing/Documents/workspace/pigo/internal/runtime/stream_response.go#L67) | `internal/runtime/stream_response.go` | Agent 运行时消费 `StreamFn` 的主驱动循环 |
