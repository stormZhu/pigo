# executeToolCall 生命周期与全局 AOP 切面架构设计

本文深入剖析 `pigo` 工具执行层的心脏函数：[`executeToolCall`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L54)。详细拆解其四大参数职责、三阶段（Prepare → Execute → Finalize）执行流水线，以及“管道统一共用，策略按名分流”的全局 AOP 切面机制。

---

## 目录

- [一、核心定位与设计原则](#一核心定位与设计原则)
- [二、函数签名与四大参数职责剖析](#二函数签名与四大参数职责剖析)
  - [1. ctx context.Context：生命周期控制与前置短路保护](#1-ctx-contextcontext生命周期控制与前置短路保护)
  - [2. cfg ToolExecutorConfig：环境注入与 AOP 切面门禁](#2-cfg-toolexecutorconfig环境注入与-aop-切面门禁)
  - [3. call agentcore.AgentToolCall：模型调用意图原料](#3-call-agentcoreagenttoolcall模型调用意图原料)
  - [4. emit agentcore.EmitFunc：外部流式事件总线](#4-emit-agentcoreemitfunc外部流式事件总线)
  - [5. 返回值语义：不抛 Go Error 的消息实体化原则](#5-返回值语义不抛-go-error-的消息实体化原则)
- [三、工具执行三阶段流水线](#三工具执行三阶段流水线)
  - [阶段 1：准备阶段（prepareToolCall）](#阶段-1准备阶段preparetoolcall)
  - [阶段 2：执行阶段（runToolWithRetry）](#阶段-2执行阶段runtoolwithretry)
  - [阶段 3：收尾阶段（finalizeToolCall）](#阶段-3收尾阶段finalizetoolcall)
- [四、深度探讨：切面钩子是所有工具共用的吗？](#四深度探讨切面钩子是所有工具共用的吗)
  - [1. 架构统一性：全局一道门，绝无漏网之鱼](#1-架构统一性全局一道门绝无漏网之鱼)
  - [2. 业务灵活性：管道统一共用，策略按名分流](#2-业务灵活性管道统一共用策略按名分流)
- [五、架构全景执行时序图](#五架构全景执行时序图)
- [六、相关源码索引](#六相关源码索引)

---

## 一、核心定位与设计原则

在 Agent 循环中，大模型生成的“工具调用指令”只是一段未经检验的 JSON 字符串。将这段原始指令转化为真正对系统产生安全影响的操作，并把结果可靠写回模型上下文，这一重任完全由 [`executeToolCall`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L54) 统驭。

其核心设计遵循两大黄金原则：
1. **统一安全门禁**：无论是内置工具、Shell 命令还是跨进程插件，必须统一走通预检、参数校验与安全审批管道。
2. **错误消息实体化（绝不返回 Go `error`）**：执行器内的任意失败（未知工具、参数非法、用户审批拒绝、执行 panic、命令超时），全部编码为携带 `IsError=true` 的 [`ToolResultMessage`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/message.go)，确保大模型的因果推理链条不发生物理断裂。

---

## 二、函数签名与四大参数职责剖析

```go
// internal/agenttool/tool_executor.go
func executeToolCall(
    ctx context.Context,
    cfg ToolExecutorConfig,
    call agentcore.AgentToolCall,
    emit agentcore.EmitFunc,
) (agentcore.ToolResultMessage, bool)
```

### 1. ctx context.Context：生命周期控制与前置短路保护
- **前置短路（Fail-Fast）**：在准备阶段最开端检查 `ctx.Err() != nil`。如果用户按下 `Ctrl+C` 或全局已超时，直接短路生成 `"tool aborted before execution"`，**物理阻断任何本地磁盘或网络操作的发起**。
- **逐级透传**：将 Context 传递到底层工具的 `Execute(ctx, ...)`，以便随时中断耗时的子进程（如 `bash` 命令）或网络抓取。

### 2. cfg ToolExecutorConfig：环境注入与 AOP 切面门禁
封装了工具执行所需的全部依赖与切面：
- **`Registry *ToolRegistry`**：工具查询与预编译 JSON Schema 校验中心。
- **`PrepareArguments` 钩子**：在 Schema 校验前改写参数（如补充工作区默认根目录、修剪空格）。
- **`BeforeToolCall` 钩子**：校验后、执行前的安全门禁（交互式 REPL 弹出审批确认、高危操作阻断）。
- **`AfterToolCall` 钩子**：执行后结果改写（对敏感信息脱敏、字段覆写）。
- **`MaxResultBytes`**：输出文本的最大字节预算（默认 100KB），防止单次工具输出打爆模型上下文。
- **`MaxToolRetries`**：瞬态错误重试次数。

### 3. call agentcore.AgentToolCall：模型调用意图原料
包含大模型在上一轮对话中发出的完整元数据：
- **`ID string`**：调用唯一标识（如 `call_xyz`），必须原样回填给结果消息的 `ToolCallID`，供 LLM 关联调用与结果。
- **`Name string`**：工具标识名（如 `"read"`、`"bash"`），用于注册表路由与策略分流。
- **`Arguments json.RawMessage`**：未经解析的原始 JSON 入参。

### 4. emit agentcore.EmitFunc：外部流式事件总线
签名简练的回调函数（`func(ctx, event) error`），用于将工具内部的微观状态实时广播到外部系统（TUI 终端、Web 前端或日志信箱）：
- **时序通知**：开始前发送 `ToolExecutionStartEvent`，结束时发送 `ToolExecutionEndEvent`。
- **增量进度流**：长耗时工具（如 `bash` 实时输出、文件下载）通过 `ToolUpdateFunc` 触发发射 `ToolExecutionUpdateEvent`。
- **反向中断**：当外部消费者已断开（`emit` 返回 error）时，执行器能立即获知并中止执行。

### 5. 返回值语义：不抛 Go Error 的消息实体化原则
- **`agentcore.ToolResultMessage`**：构造完成的标准化消息，包含 `ToolCallID`、文本内容与 `IsError` 标记。
- **`bool`**：标识该次工具调用是否请求**提前终止整个 Agent 轮次**（例如执行了 `goal_complete` 工具）。

---

## 三、工具执行三阶段流水线

```
┌────────────────────────────────────────────────────────┐
│               1. 准备阶段 prepareToolCall              │
│  Context检查 ──> Registry寻址 ──> PrepareArguments改写 │
│  ──> JSON Schema校验 ──> BeforeToolCall安全审批        │
└──────────────────────────┬─────────────────────────────┘
                           │ 任何一步失败即短路返回
                           ▼
┌────────────────────────────────────────────────────────┐
│               2. 执行阶段 runToolWithRetry             │
│  发射 StartEvent ──> 带重试执行工具 (支持增量 Update)   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│              3. 收尾阶段 finalizeToolCall              │
│  AfterToolCall结果重写 ──> MaxResultBytes文本预算裁剪  │
│  ──> 组装 ToolResultMessage ──> 发射 EndEvent          │
└────────────────────────────────────────────────────────┘
```

### 阶段 1：准备阶段（prepareToolCall）
严格经过 5 道关卡，任何一道不通过立即短路返回 `*AgentToolResult`，跳过后续所有执行：
1. **Context 存活检查**：确认上游未取消；
2. **注册表检索**：确认 `Name` 存在对应 `AgentTool`；
3. **参数预处理**：触发可选的 `PrepareArguments`；
4. **JSON Schema 校验**：使用预编译 Schema 校验参数合法性；
5. **前置门禁审批**：触发 `BeforeToolCall`，由交互策略决定 `Block` 还是改写 `UpdatedInput`。

### 阶段 2：执行阶段（runToolWithRetry）
- 发射 `ToolExecutionStartEvent`；
- 包装在重试循环中（针对被判决为瞬态网络抖动或可恢复的错误重试，由 `MaxToolRetries` 限制上限）；
- 向工具的 `Execute` 方法注入 `ToolUpdateFunc` 闭包，让工具内部能实时推流。

### 阶段 3：收尾阶段（finalizeToolCall）
- 触发 `AfterToolCall` 钩子对结果进行字段级重写或脱敏；
- **执行器外层预算约束**：检查结果总文本量，若超过 `MaxResultBytes`（默认 100,000 字节）强制截断并打上标记；
- 构造终态 `ToolResultMessage` 并发射 `ToolExecutionEndEvent`。

---

## 四、深度探讨：切面钩子是所有工具共用的吗？

### 1. 架构统一性：全局一道门，绝无漏网之鱼

在批量调度器 [`ExecuteToolCalls`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/batch_executor.go#L33) 中：

```go
// internal/agenttool/batch_executor.go
results[i], terminates[i] = executeToolCall(ctx, cfg.ToolExecutorConfig, call, emit)
```

系统内所有的工具调用请求，无论是并行的只读工具（`read`, `grep`）还是串行的写操作（`edit`, `bash`），**全部使用同一套 `ToolExecutorConfig` 并经过相同的生命周期切面**。
- **避免工具自我负责安全**：杜绝了在每个工具中散落重复写权限校验、确认弹窗的混乱代码；
- **集中管控**：安全策略（Trust Manager）、远程控制（Remote Control）或第三方插件钩子（Hooks Dispatcher）只需注入在这一处，即可覆盖系统内的全量工具。

### 2. 业务灵活性：管道统一共用，策略按名分流

虽然管道是全局共用的，但每个钩子的签名中都注入了工具的上下文标识：
- `PrepareArguments(ctx, toolName, args)`
- `BeforeToolCall(ctx, call)`（内含 `call.Name` 和 `call.Arguments`）
- `AfterToolCall(ctx, call, result, isError)`

钩子内部依靠传入的 `toolName / call.Name` 实行精准的白名单或黑名单分流：

```go
func beforeToolCall(ctx context.Context, call agentcore.AgentToolCall) *agentcore.BeforeToolCallDecision {
    // 1. 安全只读工具无需审批，直接放行
    if call.Name == "read" || call.Name == "grep" {
        return nil
    }

    // 2. 高危工具（如 bash、kill）实施安全确认
    if call.Name == "bash" {
        if !userApproved(call.Arguments) {
            return &agentcore.BeforeToolCallDecision{Block: true}
        }
    }
    return nil
}
```

这种模式确保了**“全局安全防御没有漏网之鱼”**与**“细粒度业务策略随心定制”**的完美统一。

---

## 五、架构全景执行时序图

```
LLM Output (ToolCall)
         │
         ▼
[ExecuteToolCalls 批调度器]
         │
         ▼ 统一传入 ToolExecutorConfig
[executeToolCall 核心函数]
         │
         ├──> 1. prepareToolCall()
         │      ├── ctx.Err() 存活校验
         │      ├── cfg.Registry.Get() 寻址
         │      ├── cfg.PrepareArguments() 参数改写
         │      ├── cfg.Registry.Validate() Schema 校验
         │      └── cfg.BeforeToolCall() 安全审批 (如 bash 确认)
         │           └── 若拦截: 直接短路跳至第 3 步 (返回错误消息)
         │
         ├──> 2. runToolWithRetry()
         │      ├── emit(ToolExecutionStartEvent)
         │      ├── tool.Execute(ctx, id, args, onUpdate)
         │      │      └── onUpdate() ──> emit(ToolExecutionUpdateEvent)
         │      └── 瞬态故障重试决策
         │
         └──> 3. finalizeToolCall()
                ├── cfg.AfterToolCall() 结果修改
                ├── MaxResultBytes 文本预算截断
                ├── emit(ToolExecutionEndEvent)
                └── 返回标准 ToolResultMessage (IsError 标记)
```

---

## 六、相关源码索引

| 模块 / 结构 | 文件路径 | 核心职责 |
| :--- | :--- | :--- |
| [`executeToolCall`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L54) | `internal/agenttool/tool_executor.go` | 单工具完整执行引擎入口 |
| [`ToolExecutorConfig`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L34) | `internal/agenttool/tool_executor.go` | 统一执行配置与三阶段 AOP 切面结构体 |
| [`prepareToolCall`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L78) | `internal/agenttool/tool_executor.go` | 准备阶段（短路保护、Schema校验、门禁审批） |
| [`runToolWithRetry`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L146) | `internal/agenttool/tool_executor.go` | 执行阶段（重试机制、进度广播） |
| [`finalizeToolCall`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L208) | `internal/agenttool/tool_executor.go` | 收尾阶段（切面重写、预算裁剪、消息构造） |
| [`ExecuteToolCalls`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/batch_executor.go#L33) | `internal/agenttool/batch_executor.go` | 批量工具执行调度器（并行/串行流转） |
| [`BeforeToolCallFunc`](file:///Users/yuqing/Documents/workspace/pigo/internal/agentcore/helpers.go#L56) | `internal/agentcore/helpers.go` | 执行前拦截门禁函数契约 |
