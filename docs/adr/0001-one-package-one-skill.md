# ADR 0001：一个 Skill Package 只包含一个 Skill

- 状态：Accepted
- 日期：2026-09-14

## 背景

AKM 需要支持一个 GitHub repository 同时维护很多 Agent Skill，但安装 Router 或某个具体 Skill 时不能因为“同仓”把无关 Skill 隐式打进同一个 Package。

Multi-Skill Package 会重新引入 package 内部 Skill selector、部分激活、bundle 版本耦合和第二层 dependency graph。AKM 已经有 Skill dependency resolver，不需要再用 bundle 解决能力组合。

同时 AKM 应能安装普通 Agent Skill repository，不能要求第三方先增加 AKM 专用元数据。

## 决定

一个 AKM Skill Package 恰好包含一个标准 Agent Skill：

```text
Package Root == Skill Root
one Package == one Skill
```

Package Root 的最低要求只有：

```text
SKILL.md
```

可选增强文件：

```text
akm-package.toml    # structured Skill/software dependencies
DEPENDENCIES.md     # Agent-readable special dependency instructions
```

Package 局部名称以 `SKILL.md.name` 为唯一 source of truth，并满足：

```text
basename(Package Root) == SKILL.md.name
```

GitHub v0 不使用独立的 `namespace/package` Registry identity。完整安装坐标由外部 source context 组成：

```text
<owner>/<repo>/<package>@<release-version-or-git-ref>
```

一个 repository 可以维护多个独立 Skill Package；Router Skill 可以通过可选 Manifest 中的 Skill dependencies 组织其他 Package，形成产品能力族。

## 结果

- Multi-Skill Package 不进入 v0；
- 普通只有 `SKILL.md` 的第三方 Skill 是第一等可安装 Package；
- `akm-package.toml` 与 `DEPENDENCIES.md` 都不是准入条件；
- Git repository 可维护任意数量 Skill Package；
- 安装一个 Package 只选择该 Skill Root，并在存在结构化 dependency metadata 时安装其 dependency closure；
- 产品/Skill suite 由 Router + dependency closure 表达，不由 bundle Artifact 表达；
- Package 内不允许通过 sibling/shared runtime 文件形成隐藏依赖；
- GitHub Release 直接选择 repository source snapshot；v0 不定义 AKM 专用 per-Skill Release Asset。