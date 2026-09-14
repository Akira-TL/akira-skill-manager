# v0 Module 边界工作草案

本文件只描述当前协议对应的候选代码 seam；协议未稳定前不进入实现。

## 总体数据流

```text
Project requirements
    ↓
source resolution
    ↓
Release repository snapshot / Git source cache
    ↓
SKILL.md discovery
    ↓
optional metadata parsing
    ↓
dependency resolution
    ↓
Package Store
    ↓
Project Skill Library
    ↓
optional dependency probes -> .akm/dependencies.lock
```

## `metadata`

职责：

- 解析 `SKILL.md` frontmatter；
- 解析可选 `akm-package.toml`；
- 解析 `akm.toml`；
- 读写 `akm.lock`；
- 解析 GitHub install coordinate；
- 解析 Release version requirement。

核心 domain values：

```text
GitHubRepository
SkillName
GitHubPackageCoordinate
ReleaseVersion
VersionRequirement
OptionalPackageManifest
ProjectRequirement
LockRecord
```

Package 名称来自 `SKILL.md.name`，不依赖 Manifest。

## `sources`

### GitHub Release source

```text
available_releases(owner, repo)
resolve_release(owner, repo, version) -> exact tag/commit
resolve_release(owner, repo, normalized_semver) -> actual tag + exact commit + immutable signal
fetch_repository_snapshot(owner, repo, exact_commit)
```

### Git source cache

```text
ensure_cached(owner, repo)
fetch_refs(cache_entry)
resolve_ref(cache_entry, ref) -> exact commit
read_tree(cache_entry, commit)
materialize_tree(cache_entry, commit, temp_path)
```

Git source cache 是 disposable acceleration layer，不是 Store。

## `discovery`

独立负责从一个 exact repository snapshot 发现 Skill Package：

```text
discover_skills(tree) -> SkillPackageCandidates
```

规则以 `SKILL.md` 为唯一 anchor：

- 父目录 = Package Root；
- basename == `SKILL.md.name`；
- repository 内 Skill name 唯一；
- nested Skill Roots 允许，只要最终 `SKILL.md.name` 不重复；
- optional Manifest / `DEPENDENCIES.md` 只作为附加 metadata。

把 discovery 从 source adapter 和 resolver 中独立出来，可以让 Release archive、Git cache、未来 Registry payload 共用同一套规则。

## `resolution`

职责：

- 选择 exact repository Release / Git commit；
- 选择用户指定或 repository-wide 的 discovered Skills；
- 读取可选 Manifest 的 `[dependencies]`；
- 合并同 repository Release constraints；
- 保证同 repository source snapshot 一致；
- cycle detection；
- previous Lock preference / frozen validation。

没有 Manifest 的 Skill 是合法 leaf node。

## `planning`

返回纯数据计划：

```text
release repository snapshots to obtain
git cache entries to create/fetch
package roots to snapshot
store entries to reuse
project library links to add/remove
common software probes to run
special dependency docs requiring Agent inspection
```

不包含宿主软件安装动作。

## `snapshots`

职责：把 selected Package Root 变成 verified immutable Package snapshot。

```text
snapshot_and_verify(source_root) -> VerifiedPackageSnapshot
```

至少负责：

- safe source materialization；
- `SKILL.md` 校验；
- optional Manifest 校验；
- optional `DEPENDENCIES.md` digest；
- content digest。

## `store`

只保存 immutable Skill Package snapshot：

```text
contains(digest)
put(snapshot)
get(digest)
```

Source Cache 被删不会影响已经进入 Store 的 Package。

## `activation`

根据 Lock 构建：

```text
.akm/skills/<owner>/<repo>/<skill-name>
    -> <machine-store>/<content-digest>
```

executor adapter 在其后负责最终 Skill discovery 适配。

## `dependency_checks`

只处理可选依赖增强：

- Manifest `[software]` 的 common probes；
- `DEPENDENCIES.md` digest / Agent inspection state；
- `.akm/dependencies.lock` 读写与失效。

不提供 install/upgrade/remove/configure。

## `application`

上层用例：

```text
install_target()
sync_project()
update_project()
remove_target()
doctor_project()
cache_gc()
store_gc()
```

Source Cache GC 和 Package Store GC 必须分开。

## 明确避免

v0 不建立：

- 必填 `akm-package.toml`；
- 抽象 Registry Package Identity 作为 GitHub 前置层；
- Multi-Skill Package；
- 扁平 global Skill name registry；
- Software Provider 自动安装体系；
- Package 自定义系统安装脚本；
- repository runtime shared directory；
- 项目直接链接 mutable Git checkout。