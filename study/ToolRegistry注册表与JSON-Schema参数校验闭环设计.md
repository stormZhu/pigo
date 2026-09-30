# ToolRegistry 注册表与 JSON Schema 参数校验闭环设计

本文系统性解析 `pigo` 中的工具注册表 [`ToolRegistry`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L28)、[`compiled map[string]*jsonschema.Schema`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L31) 缓存机制、JSON Schema 的规范形态，以及执行前参数拦截与字段级自我纠错（Self-Correction）的完整闭环。

---

## 目录

- [一、背景：大模型工具调用的脆弱性与防御原则](#一背景大模型工具调用的脆弱性与防御原则)
- [二、ToolRegistry 核心架构与“注册即编译”](#二toolregistry-核心架构与注册即编译)
  - [1. 双 Map 并发安全结构](#1-双-map-并发安全结构)
  - [2. compiled 字段的核心职责](#2-compiled-字段的核心职责)
  - [3. 命名空间隔离：内存 URI (mem:///)](#3-命名空间隔离内存-uri-mem)
- [三、JSON Schema 的规范形态与“一图两用”](#三json-schema-的规范形态与一图两用)
  - [1. 真实 Schema 形态剖析（以 ReadTool 为例）](#1-真实-schema-形态剖析以-readtool-为例)
  - [2. 一图两用：对外是说明书，对内是防火墙](#2-一图两用对外是说明书对内是防火墙)
- [四、校验引擎与路径提取机制：深入底层细节](#四校验引擎与路径提取机制深入底层细节)
  - [1. 实例树解码（UseNumber 与 Instance）](#1-实例树解码usenumber-与-instance)
  - [2. 为什么缺失 required 约束对应空路径？](#2-为什么缺失-required-约束对应空路径)
  - [3. RFC 6901 JSON 指针与 (root) 具象化渲染](#3-rfc-6901-json-指针与-root-具象化渲染)
- [五、运行时拦截与模型自我纠错闭环（Self-Correction）](#五运行时拦截与模型自我纠错闭环self-correction)
- [六、相关源码索引](#六相关源码索引)

---

## 一、背景：大模型工具调用的脆弱性与防御原则

大模型在生成工具调用（Tool Call / Function Calling）的 JSON 参数时，天然存在不可预测性：
* **缺少必填项**：如调用 `read` 时漏传 `"path"`；
* **类型幻觉**：如将数值型的起始行号传成字符串 `"offset": "first_line"`；
* **多余参数幻觉**：生成不存在的虚假字段（如 `"encoding": "utf-8"`）。

如果直接将未经验证的 JSON 反序列化到具体 Go 结构体：
1. Go 原生 `json.Unmarshal` 遇到缺少必填字段时只会静默填入零值（导致业务层把空字符串作为路径去操作文件，引发灾难）；
2. 若业务层强行抛出 Go `error`，会导致整个 Agent 循环中断，前功尽弃。

因此，系统必须在**工具真正执行之前**，建立一道严格、细粒度且具备指导意义的**安全闸门**。

---

## 二、ToolRegistry 核心架构与“注册即编译”

在 [`internal/agenttool/registry.go`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L28) 中，注册表定义如下：

```go
type ToolRegistry struct {
    mu       sync.RWMutex
    tools    map[string]agentcore.AgentTool
    compiled map[string]*jsonschema.Schema
}
```

### 1. 双 Map 并发安全结构
- **`tools`**：保存工具实例本身，供 [`Get(name)`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L67) 查询工具元数据与执行体。
- **`compiled`**：按工具名索引**预编译好的 JSON Schema 校验器指针**。
- **`mu sync.RWMutex`**：保护并发读写（工具批量并发执行时会高频读）。

### 2. compiled 字段的核心职责
- **把坏 Schema 阻断在启动期（Fail-Fast）**：
  在 [`Register`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L45) 阶段即调用 `compileSchema`。如果某个工具的 Schema JSON 存在语法错误，程序在初始化装配阶段立即抛出错误，杜绝“错误潜伏到运行时模型首次调用才爆炸”。
- **消除运行时重复编译 CPU 开销**：
  JSON Schema 语法树构建与类型校验规则编译相对繁重。编译一次并缓存在 `compiled` map 中后，后续所有工具调用只需读锁复用该指针。

### 3. 命名空间隔离：内存 URI (mem:///)

```go
func compileSchema(name string, raw json.RawMessage) (*jsonschema.Schema, error) {
    // ...
    c := jsonschema.NewCompiler()
    loc := "mem:///" + name + ".json"
    if err := c.AddResource(loc, doc); err != nil {
        return nil, err
    }
    return c.Compile(loc)
}
```
为每个工具在内存中生成合成 URI `mem:///<name>.json`，保证不同工具间的 `$ref` 引用与局部定义在内存中严格隔离，互不干扰。

---

## 三、JSON Schema 的规范形态与“一图两用”

### 1. 真实 Schema 形态剖析（以 ReadTool 为例）

内置工具 [`ReadTool`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/read_tool.go#L76) 的 Schema 遵循标准 JSON Schema 规范：

```json
{
  "type": "object",
  "properties": {
    "path":   {"type": "string", "description": "File path to read, relative to the workspace root."},
    "offset": {"type": "integer", "description": "1-based line number to start from.", "minimum": 0},
    "limit":  {"type": "integer", "description": "Maximum number of lines to return.", "minimum": 0}
  },
  "required": ["path"],
  "additionalProperties": false
}
```

* **`"type": "object"`**：大模型调用参数的根结构必须为键值对象；
* **`"properties"`**：声明字段类型与人类可读的 `description`（作为模型的行为指引）；
* **`"required": ["path"]`**：显式声明必填约束；
* **`"additionalProperties": false`**：严格封门，拒绝一切未声明字段。

### 2. 一图两用：对外是说明书，对内是防火墙

```
                     ┌────────────────────────┐
                     │     tool.Schema()      │ (原始 JSON Schema)
                     └───────────┬────────────┘
                                 │
         ┌───────────────────────┴────────────────────────┐
         ▼                                                ▼
 【对外：提供给模型】                            【对内：本地安全校验】
 encodeOpenAITools()                             ToolRegistry.Register()
         │                                                │
         ▼                                                ▼
 拼入发往 LLM 的 Tool API Payload            compileSchema() ──> 缓存至 compiled map
 (指导大模型按格式填参)                                      │
                                                          ▼
                                            执行前: ToolRegistry.Validate(name, args)
```

---

## 四、校验引擎与路径提取机制：深入底层细节

### 1. 实例树解码（UseNumber 与 Instance）

在 [`r.Validate`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L98) 中：

```go
var inst any
dec := json.NewDecoder(bytes.NewReader(nonEmptyJSON(args)))
dec.UseNumber() // 避免 float64 精度损失与整数误判
dec.Decode(&inst)
```
输入的 JSON 字节流被解析为内存实例对象 `inst`（即 JSON Schema 规范中的 Instance），交由 `sch.Validate(inst)` 匹配。

### 2. 为什么缺失 required 约束对应空路径？

这是理解校验器机制最关键的一点：
* 当传入 `{"path": 123}` 时，校验器深入到了字段 `path`，发现值类型与 `string` 冲突，此时发生违规的实体是 **子字段 `path` 本身**；
* 当传入 `{"offset": 10}` 时，缺少必需的 `"path"`。由于 **`path` 字段在入参中根本不存在**，错误无法挂载在一个不存在的节点上；
* `required: ["path"]` 是声明在外层根对象（Object）上的规则，它断言的是 *“当前 Object 必须包含某些 key”*。因此，**违规的责任实体是持有这些属性的最外层根对象**。
* 校验器生成的 `ValidationError.InstanceLocation` 自然是一个空切片 `[]string{}`（即未向下深入任何层级）。

### 3. RFC 6901 JSON 指针与 (root) 具象化渲染

`pigo` 通过以下两条逻辑将底层错误转为易读信息：

1. **[`jsonPointer`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L191)**：遵循 [RFC 6901](https://datatracker.ietf.org/doc/html/rfc6901) 标准，空切片被渲染为 `""`（空字符串）；子字段路径被渲染为 `"/path"`、`"/offset"`。
2. **[`ValidationErrorResult`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L138)**：将空字符串优雅替换为 `(root)`：
   ```go
   field := e.Field
   if field == "" {
       field = "(root)"
   }
   fmt.Fprintf(&b, "  - %s: %s\n", field, e.Message)
   ```

常见错误渲染效果对比：

| 错误类型 | 传入参数样例 | 提取 Field | 最终回传模型的提示 |
| :--- | :--- | :--- | :--- |
| **遗漏必填** | `{"offset": 0}` | `""` → `(root)` | `(root): missing properties: 'path'` |
| **类型错误** | `{"path": 123}` | `"/path"` | `/path: expected string, but got number` |
| **负数越界** | `{"path": "a.go", "offset": -1}` | `"/offset"` | `/offset: -1 must be >= 0` |
| **多余字段** | `{"path": "a.go", "extra": 1}` | `""` → `(root)` | `(root): additional properties 'extra' not allowed` |

---

## 五、运行时拦截与模型自我纠错闭环（Self-Correction）

在 [`internal/agenttool/tool_executor.go:103`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L103) 中，校验失败不会导致 Agent 奔溃，而是形成如下闭环：

```
                    ┌────────────────────────────┐
                    │  大模型发起 ToolCall 调用   │
                    └─────────────┬──────────────┘
                                  │ args: {"offset": 10}
                                  ▼
                    ┌────────────────────────────┐
                    │   tool_executor: 执行拦截  │
                    └─────────────┬──────────────┘
                                  │ 触发 Registry.Validate(name, args)
                                  ▼
                    ┌────────────────────────────┐
                    │    参数校验失败 (len > 0)   │
                    └─────────────┬──────────────┘
                                  │ 生成 ValidationErrorResult
                                  ▼
                    ┌────────────────────────────┐
                    │ AgentToolResult:           │
                    │ - Content: 字段级报错提示  │
                    │ - Terminate: nil (不终止)  │
                    └─────────────┬──────────────┘
                                  │ 作为 tool_result 写入会话历史
                                  ▼
                    ┌────────────────────────────┐
                    │      进入下一轮交互        │
                    │ 大模型读取报错，修正必填项 │
                    │ 重新发起合法 ToolCall 调用  │
                    └────────────────────────────┘
```

通过将参数错误转化为对话上下文中的“建设性反馈（Actionable Feedback）”，系统赋予了大模型无需人工介入即可自主修正参数的能力。

---

## 六、相关源码索引

| 模块 / 结构 | 文件路径 | 核心功能 |
| :--- | :--- | :--- |
| [`ToolRegistry`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L28) | `internal/agenttool/registry.go` | 工具注册与 Schema 校验并发安全管理者 |
| [`compiled`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L31) | `internal/agenttool/registry.go` | 预编译 Schema 缓存 map |
| [`flattenValidationError`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L168) | `internal/agenttool/registry.go` | 递归遍历 AST 提取叶子节点错误与路径 |
| [`ValidationErrorResult`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/registry.go#L134) | `internal/agenttool/registry.go` | 组装用于模型自我纠错的字段级自然语言消息 |
| [`tool_executor.go`](file:///Users/yuqing/Documents/workspace/pigo/internal/agenttool/tool_executor.go#L103) | `internal/agenttool/tool_executor.go` | 工具执行入口处的前置 Schema 校验拦截与门禁 |
