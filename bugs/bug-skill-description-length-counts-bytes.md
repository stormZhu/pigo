# Bug: skill description 长度校验按字节计数，中文技能被误判超限

- 状态：待修复
- 发现时间：2026-09-18
- 影响范围：所有 `description` 含非 ASCII（中文等）的 skill，即 `~/.agents/skills` 下的中文技能
- 严重程度：中（静默丢弃技能，用户只看到一条 info 日志，技能直接不可用）

## 现象

启动 pigo 时 stderr 输出：

```
pigo: skills: skill /Users/yuqing/.agents/skills/lark-apps/SKILL.md: description exceeds 1024 characters (1287)
```

该 skill 既不出现在系统提示的可用技能列表里，也没有对应的 `/lark-apps` 斜杠命令——即**整个技能被丢弃**。

## 根因

`internal/runtime/skills.go` 中的校验用 `len()` 统计长度，而 Go 的 `len(string)` 返回的是**字节数**，不是字符数：

```go
// internal/runtime/skills.go:80-88
func validateSkillDescription(description string) error {
	if strings.TrimSpace(description) == "" {
		return errors.New("frontmatter missing required 'description'")
	}
	if len(description) > maxSkillDescriptionLength {   // <- 字节数
		return fmt.Errorf("description exceeds %d characters (%d)", maxSkillDescriptionLength, len(description))
	}
	return nil
}
```

上限定义在 `internal/runtime/skills.go:50-53`：

```go
const (
	maxSkillNameLength        = 64
	maxSkillDescriptionLength = 1024
)
```

失败链路：

1. `validateSkillDescription` 返回 error（`skills.go:159-161`，`ParseSkill` 内）；
2. `ParseSkill` 整体返回 error；
3. `LoadSkillsDir` 把该文件跳过，仅把错误 `errors.Join` 进返回值（`skills.go:236-240`）；
4. `internal/cli/run/run.go:175-178` 把 error 打到 stderr，然后**用剩下的技能继续运行**。

第 3 步是设计意图（"一个坏技能不能遮蔽其余技能"），但对本 bug 而言结果是：**一个本应合法的中文技能被静默剔除**。

## 证据

对报错文件实测（`~/.agents/skills/lark-apps/SKILL.md` 的 `description`）：

| 度量 | 值 |
| --- | --- |
| 字节数 `len(s)` | 1287 |
| 字符数 `utf8.RuneCountInString(s)` | **580** |

中文在 UTF-8 下每字 3 字节，因此 pigo 对纯中文 description 的实际容忍度约为规范的 **1/3**。该 description 为 580 字符，**远低于规范的 1024 字符上限**。

## 与规范及同类实现的对比

Agent Skills 规范（agentskills.io/specification）定义的是**字符**：

> `description` | Yes | Max 1024 characters. Non-empty.

| 实现 | 超限后的行为 |
| --- | --- |
| agentskills.io 规范 | 仅定义约束，不规定行为 |
| pi | 只警告，**技能照常加载**；仅缺 `description` 才不加载 |
| Claude Code | 遵循同一标准，`description + when_to_use` 合并后按 1536 字符截断 |
| pigo（本仓库） | 校验失败 → 跳过该技能 → **整个技能不被加载** |

因此存在两处偏离，本 bug 记录聚焦第 1 处：

1. **计数单位错误**（本文档主题，明确的 bug）；
2. **策略比 pi 严格**（设计取舍，非 bug）：pi 是"警告但保留"，pigo 是"警告且丢弃"。

## 修复建议

### 必做：改用字符计数

`internal/runtime/skills.go:84` 改为 rune 计数，与规范一致：

```go
if utf8.RuneCountInString(description) > maxSkillDescriptionLength {
	return fmt.Errorf("description exceeds %d characters (%d)",
		maxSkillDescriptionLength, utf8.RuneCountInString(description))
}
```

注意 `validateSkillName`（`skills.go:62-76`）同样用了 `len(name)`，但名称已被 `skillNamePattern` 限制为 `[a-z0-9-]`，字节数恒等于字符数，**无需改动**。

### 待定：是否对齐 pi 的宽松策略

若要连策略一起对齐 pi（超长只警告、技能仍加载），需要改动 `ParseSkill` 的返回值约定，让校验失败能携带"警告"而非"丢弃"语义，改动面明显更大。建议作为独立议题决策，不要与本 bug 修复混在一起。

## 测试影响

`internal/runtime/orchestration_test.go:308-322` 的 `TestValidateSkillDescription` 用 `strings.Repeat("x", 1025/1024)` 断言边界，因 ASCII 字符下字节数等于字符数，该测试**在修复后依然通过**，不会暴露此 bug。

修复时应补充一条非 ASCII 用例，例如：

```go
// 1024 个中文字符 = 3072 字节，按字符计数应合法。
if err := validateSkillDescription(strings.Repeat("中", 1024)); err != nil {
	t.Errorf("1024-char CJK description rejected: %v", err)
}
```

## 相关文件

- `internal/runtime/skills.go` — 校验逻辑与上限常量
- `internal/runtime/orchestration_test.go` — 现有边界测试
- `internal/cli/run/run.go` — 错误上报位置
- 触发文件：`~/.agents/skills/lark-apps/SKILL.md`
