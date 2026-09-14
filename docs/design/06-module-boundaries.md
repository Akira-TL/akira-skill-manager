# v0 Module 边界

本文件只定义首阶段代码应形成的深 Module 与 seam，不提前设计 CLI。

## 总体数据流

```text
Project files
    ↓
metadata
    ↓
resolution ← package_index
    ↓
planning ← store / activation / software observations
    ↓
InstallPlan
    ↓ approval
artifacts → store → activation
             ↘ software providers（后续）
```

## `metadata`

职责：

- 解析、规范化和验证 `akm-package.toml`；
- 解析、规范化和验证 `akm.toml`；
- 读写 `akm.lock`；
- 提供强类型 domain values：PackageIdentity、Version、VersionRequirement、SourceSpec、LockRecord、SoftwareRequirement。

外部 Interface 应尽量小：

```text
load_package_manifest(path)
load_project_manifest(path)
load_lock(path)
write_lock(path, lock)
```

TOML 细节不泄漏到 resolver。

## `package_index`

职责：为 resolver 提供“某 Package 有哪些版本、这些版本依赖什么、最终 artifact 在哪里”的只读 metadata。

Interface：

```text
available_versions(package)
manifest(package, version)
artifact(package, version)
```

计划 Adapter：

- InMemoryPackageIndex：测试；
- StaticIndex：首阶段本地/HTTP index；
- GitHubReleaseIndex：如果首阶段需要直接消费 GitHub Release；
- future registry adapter。

resolver 不知道 HTTP、GitHub API 或 cache 细节。

## `resolution`

职责：把 Project Requirement 解析成唯一 exact Package Graph。

Interface：

```text
resolve(request) -> Resolution
validate_frozen(request, lock) -> Resolution
```

Implementation 内部可以使用 PubGrub 或其他 solver；调用者只看到结构化 Resolution / ResolutionError。

cycle detection、Skill name collision、source conflict 属于这个 Module，因为它们决定“这个 dependency graph 是否合法”。

## `planning`

职责：比较 Resolution 与当前机器/项目状态，生成完整 InstallPlan。

Interface：

```text
plan(resolution, observations) -> InstallPlan
```

它不执行动作。

这样测试可以在纯数据上断言：

- 应下载什么；
- 应保留什么；
- 应增删哪些 activation link；
- 缺哪些软件；
- 哪些动作需要 approval。

## `artifacts`

职责：把一个已批准的 exact source 变成已验证 Package snapshot。

Interface：

```text
fetch_and_verify(package_source) -> VerifiedPackageSnapshot
```

内部负责：

- Release download；
- Git exact commit checkout/snapshot；
- path snapshot；
- SHA-256；
- safe archive extraction；
- Manifest / `SKILL.md` identity 校验。

不同 source kind 是内部 Adapter，不把 source-specific 流程散到 planner/store。

## `store`

职责：机器级不可变 Package content 的去重与查询。

Interface：

```text
contains(digest)
put(verified_snapshot) -> StoreEntry
get(digest) -> StoreEntry
```

逻辑 key 是 content digest，而不是 repository checkout path。

Package Identity/version 到 digest 的便利索引可以存在，但不是内容真相来源。

## `activation`

职责：把 Lock/Resolution 映射成项目级 Skill symlink view。

Interface：

```text
inspect(project)
reconcile(project, desired_links)
doctor(project, lock)
```

只管理 AKM 自己拥有的 `.akm/skills`，不直接管理具体 executor 的私有 Skill 目录。

executor adapter 如果未来需要，应依赖 `Project Skill View`，不复制 Package。

## `software`

职责：

- Software Catalog；
- Probe；
- requirement aggregation；
- provider selection；
- software action planning。

Interface：

```text
doctor(requirements) -> SoftwareReport
plan(requirements, report) -> SoftwarePlan
```

v0.1 不需要暴露 `apply` 给普通 sync 流程。

## `application`

这是唯一允许编排多个深 Module 的上层用例层，例如：

```text
sync_project()
update_project()
remove_requirement()
doctor_project()
```

它负责 transaction ordering 和 approval handoff，但不重新实现 resolver/store/artifact 规则。

未来 CLI、MCP、Python API 都应该调用同一 application use case，而不是各自重写业务逻辑。

## 明确避免的浅 Module

v0 不建立：

- `GitHubService` 贯穿全项目；
- 每个 TOML table 一个 manager class；
- `DependencyManager` / `PackageManager` 这种只转发调用的总管类；
- resolver 内部直接调用 subprocess/network/filesystem；
- CLI command 内自行计算 dependency graph；
- Provider 内重新解析 Package Manifest。

## 第一阶段建议源码布局

```text
src/akm/
├── metadata.py
├── package_index.py
├── resolution.py
├── planning.py
├── artifacts.py
├── store.py
├── activation.py
├── software.py
└── application.py
```

只有当某个文件真的形成多个独立变化原因时再拆成 package。首阶段先追求深 Module，而不是目录数量。
