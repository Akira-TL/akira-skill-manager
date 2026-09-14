# ADR 0007：Project Manifest 与 Lock 三层模型

- 状态：Accepted
- 日期：2026-09-14

## 背景

早期 `akm.toml` 为 Git source 使用 `"*"` version placeholder + 独立 `[sources]` table；`akm.lock` 又在 Package Record 中重复 source kind、version、commit，并保存 Manifest/Dependency Check File digest 与 software requirements。随着 repository source、Package Snapshot 和本机 dependency state 的职责已经分离，这些重复字段不再必要。

## 决定

Project state 分为三层：

```text
akm.toml
→ top-level requirement intent

akm.lock
→ normalized requirements + exact repository source + exact Package Snapshot graph

.akm/dependencies.lock
→ current host dependency observations
```

具体规则：

- `akm.toml [skills]` 的字符串值表示 GitHub Release version requirement；Git source 使用 `{ git = "<ref>" }` inline table；
- source binding 始终是 repository-scoped，同一 resolution 的一个 `owner/repo` 只能绑定一个 exact snapshot；不同 Git ref 或 Release/Git 混装返回 `RepositorySourceConflict`；
- `akm.lock` 只包含 `[[requirement]]`、`[[repository]]`、`[[package]]` 三类 Record；
- source provenance 只写在 Repository Record；Package Record 不重复 source kind/version/commit；
- dependency edge 只写 `owner/repo/package`，不重复 exact version/commit；
- Project Lock 不保存 `manifest-digest`、`dependencies-doc-digest` 或 `[[software]]`；Package `content-digest` 已覆盖 immutable payload；
- `frozen` 比较解析后的 Requirement Set，不 hash `akm.toml` 原始 bytes；
- Lock writer 使用 UTF-8、LF、无注释，并按 coordinate canonical 排序。

## 结果

Project Manifest 只描述用户意图，Repository Record 只描述 exact source，Package Record 只描述 exact content 与 dependency graph。本机环境状态完全留在 `.akm/dependencies.lock`，三层之间不再重复同一事实。
