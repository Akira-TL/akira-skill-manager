# ADR 0002：扁平 Skill name 冲突与 Package Identity 单版本模型

- 状态：Rejected
- 日期：2026-09-14
- 后续修正：ADR 0008 接受“项目 activation 层扁平且同名会冲突”，但仍不接受本 ADR 绑定提出的全局 Package Identity 模型。

## 原提案

原提案把两个问题绑定在一起：

1. 项目 Skill discovery 面是扁平的，因此不同来源导出同名 `SKILL.md.name` 时会冲突；
2. 为解决这种冲突，引入独立的全局 Package Identity，并进一步要求“一个 Package Identity 在项目中只能有一个版本”。

## 最终拆分结果

ADR 0008 已确认项目 executor-visible Skill 直接位于：

```text
.agents/skills/<activation-name>
```

因此扁平 runtime name 冲突确实存在。例如：

```text
A/repo/foo
B/repo/foo
```

两个 Package 在 source/resolver/Store 层可以独立共存，但默认都要激活为：

```text
.agents/skills/foo
```

此时返回 `ActivationNameConflict`，由用户选择为新安装项 rename 或放弃；AKM 不自动覆盖或自动改名。

这只是**项目 activation name 冲突**，不需要引入全局 Package Identity。

## 仍然拒绝的部分

v0 仍不建立独立 Registry-style `namespace/package` Package Identity，也不因为 activation name 冲突而改变 GitHub source model。

版本首先属于 GitHub repository Release；来自同一个 repository 的依赖约束共同选择一个满足条件的 SemVer Release，并从该 exact repository snapshot 中 discovery 所需 Skill Package。显式 Git source 则绑定同一 repository exact commit。

## 当前方向

- Package coordinate 继续使用 `owner/repo/package`；
- 不建立全局 Package Identity registry；
- source/resolver/Store 层允许不同 repository 中同名 Skill 独立存在；
- 单个项目 `.agents/skills/` activation name 必须唯一；
- 同名 activation conflict 通过用户明确 rename 或 abort 解决；
- 将来如果增加 Registry source model，再单独设计 Registry package identity 与多版本规则。
