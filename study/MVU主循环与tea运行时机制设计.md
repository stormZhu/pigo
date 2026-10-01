# MVU 主循环与 tea 运行时机制设计

本文把 Bubble Tea v2 的 **MVU 驱动细节**单独拆出来讲：`Update` 与 `View` 的职责边界、`Cmd` 的真实语义（返回的是函数、并不在 `Update` 里执行）、`cmds` / `msgs` 两个 channel 为什么都是无缓冲，以及「一条消息进来之后到底按什么顺序发生哪些事」。

它是 [全屏 TUI 事件桥与 MVU 生命周期设计](./全屏TUI事件桥与MVU生命周期设计.md) 的补充：那篇以 [`tui.Run`](../internal/cli/tui/run.go#L13-L24) 为主线讲这条表现层整体怎么跑，本文只聚焦 MVU 主循环与 tea 运行时这一层机制。文中 `tea.go`、`renderer.go` 等指 Bubble Tea **v2.0.8** 模块缓存里的源码，行号对应该版本。

---

## 目录

- [一、MVU 三元与职责边界](#一mvu-三元与职责边界)
- [二、一条消息的固定处理顺序](#二一条消息的固定处理顺序)
- [三、`Cmd` 是「被返回的函数」](#三cmd-是被返回的函数)
- [四、两个 channel 都是无缓冲](#四两个-channel-都是无缓冲)
- [五、绘制链路](#五绘制链路)
- [六、对本项目的三条硬约束](#六对本项目的三条硬约束)
- [七、源码索引](#七源码索引)

---

## 一、MVU 三元与职责边界

Bubble Tea v2 的 `Model` 接口只有三个方法（`tea.go:53-65`）：

```go
type Model interface {
	Init() Cmd
	Update(Msg) (Model, Cmd)
	View() View
}
```

职责边界可以压成一张表：

| 方法              | 职责                                       | 不做的事                       |
| :---------------- | :----------------------------------------- | :----------------------------- |
| `Init`            | 返回首个 `Cmd`                             | 业务逻辑                       |
| `Update`          | 纯状态迁移：读 `Msg`、返回新 `Model` + 可选 `Cmd` | 绘制、阻塞、直接做副作用       |
| `View`            | 纯投影：`Model` → 这一帧的完整画面         | 副作用、阻塞                   |

两处容易踩的地方：

- **`Update` 不绘制。** 它的返回值里没有任何「画」的动作，绘制发生在主循环里紧随其后的一次 `render` 调用（见第二节）。
- **`View` 在 v2 返回的是 `tea.View` 结构体，不是字符串。** 它同时携带 `Content`，以及 `AltScreen` / `MouseMode` 这类**终端声明**。本项目的 `View` 正是靠它声明备用屏与 cell-motion 鼠标模式（[§四](./全屏TUI事件桥与MVU生命周期设计.md#四mvu-主循环)）。

## 二、一条消息的固定处理顺序

主循环 `eventLoop`（`tea.go:743-883`）里，每收到一条消息都走同一段固定序列：

```go
case msg := <-p.msgs:                  // 752 取一条消息
	msg = p.translateInputEvent(msg)   // 753 归一化输入事件
	...
	if msg == nil { continue }         // 759-761 filter 丢弃
	switch msg := msg.(type) { ... }   // 764 内置消息：QuitMsg / InterruptMsg / printLineMessage …

	var cmd Cmd
	model, cmd = model.Update(msg)     // 872 只更新状态

	select {
	case <-p.ctx.Done():
		return model, nil
	case cmds <- cmd:                  // 877 同步交接 Cmd
	}

	p.render(model)                    // 880 才绘制
}
```

三点值得记住：

- **`Update` 与绘制是两步。** 872 只产出新 Model，880 才把 Model 交出去画。
- **Cmd 的交接排在绘制之前。** 所以你看到的每一帧，都是「这条消息的状态迁移已完成、下一条 Cmd 已挂上」之后才画的。
- **每条消息都会调一次 `View`。** 这里没有「状态没变就跳过」的判断——那是 renderer 内部的事，不属于主循环语义。

## 三、`Cmd` 是「被返回的函数」

```go
type Cmd func() Msg   // tea.go:390
```

`Update` 返回的 `cmd` 是一个**函数值**，此刻并没有执行。真正执行它的是命令分发器 `handleCommands`：

```go
// tea.go:700-739
case cmd := <-cmds:
	if cmd == nil {
		continue
	}
	// Don't wait on these goroutines, otherwise the shutdown latency would get
	// too large as a Cmd can run for some time (e.g. tick commands that sleep
	// for half a second)...
	go func() {
		...
		msg := cmd()   // 731 这里才执行
		p.Send(msg)
	}()
```

由此得到一条关键结论：**`Update` 不会被「阻塞式读 channel」卡住**——那个阻塞发生在 tea 运行时替这个 Cmd 派生出的 goroutine 里，`Update` 早已返回。这正是本项目的 [`waitForEvent`](../internal/cli/tui/bridge.go#L123-L127) 必须写成 `tea.Cmd`、而不能在 `Update` 里直接 `<-ch` 的原因。

同一段注释也点明了代价：这些 goroutine **不会被等待**，会一直存在到 `Cmd` 自己返回为止。所以 `Cmd` 应当短小、且能自行结束。

## 四、两个 channel 都是无缓冲

```go
msgs: make(chan Msg),     // tea.go:598
cmds := make(chan Cmd)    // tea.go:998
```

两个都是 cap 0，于是两处都是 rendezvous（同步交接）：

- **`cmds <- cmd`（877）**：`eventLoop` 要等到 `handleCommands` 把它接住才继续。因为对端除了 `select` 就是 spawn 一条 goroutine，常年处于 ready 状态，实际等待可忽略。
- **`p.Send(msg)`（1183-1188）往 `msgs` 投递**：所有 Cmd goroutine 的投递都要在 `msgs` 上逐一握手，因此**同一时刻只可能有一条消息正在被交给 eventLoop**——tea 的消息投递天生是串行的。这是[事件桥](./全屏TUI事件桥与MVU生命周期设计.md#五事件桥两条-goroutine一个-channel)「简单消费者唯一」那半边的底层依据。

**cap 0 是必须的吗？** 不是。缓冲区大小只是个旋钮，换成带缓冲功能上照样成立；真正不可让步的是「`Cmd` 的执行必须离开主循环」（第三节那条 `go func()`）。选 0 换来的是三点边际好处：

1. 交接即同步点，不存在「命令排着队、还没被执行」的中间态；
2. 关停时不会出现「Cmd 已塞进缓冲区，下一轮却先被 `ctx.Done` 抢走」而留下没人执行的悬空命令；
3. 收（`msgs`）发（`cmds`）一个口径，不必再解释第二套排队语义。

代价则是 `eventLoop` 每次都要等一个握手。这个等待「可以忽略」是对当前实现的经验判断，不是 cap 0 本身给出的保证。

## 五、绘制链路

```go
// tea.go:886-890
func (p *Program) render(model Model) {
	if p.renderer != nil {
		p.renderer.render(model.View())   // View() 交出这一帧
	}
}
```

`View` 和 `Update` 一样跑在 **tea 主循环 goroutine** 上，并且每收到一条消息就被调用一次。因此它必须是**便宜且无副作用**的：同样的 Model 应投影出同样的画面；写文件、发 channel、跑外部命令这类事都不能放进去，它们是 `Cmd` 的活儿。

renderer 拿到 `tea.View` 之后如何写终端（是否差分、如何节流）属于渲染层，不在本文范围。

## 六、对本项目的三条硬约束

把上面几条折成本项目 `internal/cli/tui` 的约束：

1. **`Update` 里不能有阻塞读。** 读 channel 必须包成 `tea.Cmd`（[`waitForEvent`](../internal/cli/tui/bridge.go#L123-L127)），否则卡死的是主循环。
2. **所有状态迁移都在 tea goroutine 上。** `Update` 是 `m.*` 的唯一写入口，pump goroutine 只碰 `agentCtx` 与 channel——`m.running` / `m.runCh` 因此不需要任何锁。
3. **副作用只从 `Cmd` 出去。** 落盘、git 探测、远程监听都做成 `Cmd`，`View` 保持纯投影。

## 七、源码索引

- `Model` 接口：`tea.go:53-65`
- `Cmd` 定义：`tea.go:390`
- 命令执行：`handleCommands`，`tea.go:700-739`
- 主循环：`eventLoop`，`tea.go:743-883`；`Update` 调用 872、Cmd 交接 877、绘制 880
- 渲染入口：`render`，`tea.go:886-890`
- 通道创建：`msgs` `tea.go:598`、`cmds` `tea.go:998`
- 消息投递：`Send`，`tea.go:1183-1188`
- 本项目落点：[`Model.Update`](../internal/cli/tui/model.go#L238-L547)、[`Model.View`](../internal/cli/tui/model.go#L1186-L1198)、[`waitForEvent`](../internal/cli/tui/bridge.go#L123-L127)、[`pumpNext`](../internal/cli/tui/model.go#L1173-L1178)
