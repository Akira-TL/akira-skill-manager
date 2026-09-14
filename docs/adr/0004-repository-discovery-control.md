# ADR 0004：Repository Discovery Control

- 状态：Accepted
- 日期：2026-09-14

## 背景

AKM 已确定：合法 `SKILL.md` 是 Skill Package 的唯一最低准入条件，Package Name 以 `SKILL.md.name` 为唯一 source of truth。

普通 repository 可以零配置自动发现全部 Skill，但大型 repository 可能同时包含正式 Skill、example、fixture 或文档示例。AKM 不能硬编码目录名猜测哪些 Skill 应参与安装，也不应重新要求每个 Skill 提供 `akm-package.toml` 才能被发现。

因此需要一个可选、repository-level 的 discovery scope control。

## 决定

Repository root 可以提供可选：

```text
akm-repo.toml
```

最小结构：

```toml
schema = 1

[discovery]
include = ["skills/**"]
exclude = ["tests/**", "fixtures/**"]
```

规则：

- 没有 `akm-repo.toml` 时默认发现整个 exact source snapshot 中全部版本化合法 `SKILL.md`；
- `include` / `exclude` 匹配 repository-relative Package Root path；
- `exclude` 优先于 `include`；
- v0 glob grammar 只支持字面路径、`*`、`**`、`?`；
- Package Name 仍只来自 `SKILL.md.name`；
- `akm-repo.toml` 不能创建一个没有合法 `SKILL.md` 的 Package；
- 不增加 Package Name 到路径的第二份映射；
- 不增加 default/publish 等第二套 discovery set；
- Release source 与 Git source 对同一个 exact repository snapshot 使用相同 discovery policy；
- 不同名称的 nested Skill Roots 可以同时存在；只有最终 Package Name 重复才返回 `AmbiguousPackageDiscovery`。

## 结果

- 普通第三方 Skill repository 保持零配置可安装；
- monorepo 可以显式排除 example、fixture 等不希望参与安装的 Skill Root；
- repository discovery control 不承担 Package identity、version、dependency 或 executor discovery 职责；
- nested Skill Root 不再是 AKM core discovery error；
- Repository-wide install `owner/repo@...` 始终表示安装该 snapshot 经唯一 discovery policy 过滤后的全部 Package。
