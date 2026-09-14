# v0 Module 边界工作草案

本文件只描述当前协议对应的候选代码 seam；协议未稳定前不进入实现。

## 总体数据流

```text
Project requirements
    ↓
metadata
    ↓
resolution ← GitHub Releases / explicit Git source
    ↓
planning ← store / project skill library / common dependency observations
    ↓
artifact fetch & verify
    ↓
store
    ↓
activation
    ↓
common dependency probes -> .akm/dependencies.lock
```

AKM 流程到本地依赖状态记录为止。特殊依赖后续由 Agent 读取不可变 `DEPENDENCIES.md` 与 `.akm/dependencies.lock` 处理。

## `metadata`

职责：

- 解析 `akm-package.toml`；
- 解析 `akm.toml`；
- 读写 `akm.lock`；
- 解析 GitHub install coordinate；
- 解析 version requirement；
- 校验 `package.name == Package Root basename == SKILL.md.name`。

候选 domain values：

```text
GitHubRepository
PackageName
GitHubPackageCoordinate
ReleaseVersion
VersionRequirement
PackageManifest
ProjectRequirement
LockRecord
CommonSoftwareRequirement
```

不再以 `PackageIdentity(namespace/package)` 作为 v0 核心类型。

## `sources`

GitHub v0 需要两个 source adapter：

### GitHub Release source

```text
available_releases(owner, repo)
list_package_assets(owner, repo, release)
fetch_package_manifest(owner, repo, release, package)
fetch_asset(owner, repo, release, package)
```

### Git source

只服务显式 Git 模式：

```text
checkout(owner, repo, ref) -> exact commit
discover_packages(exact_commit) -> validated package roots
snapshot(package_root)
```

未来 Registry 是新的 source adapter，不要求改造 GitHub coordinate 语义。

## `resolution`

职责：

- 合并同一 GitHub repository 的 Release version constraints；
- 选择 exact Release version；
- 展开 `owner/repo/package` dependencies；
- 处理 repository-wide top-level selection；
- cycle detection；
- previous Lock preference；
- frozen validation。

Interface：

```text
resolve(request) -> Resolution
validate_frozen(request, lock) -> Resolution
```

AKM core 不检查扁平 Skill name collision，因为 Project Skill Library 保留 `owner/repo/package` 层级。

## `planning`

职责：比较 Resolution 与本地状态并返回：

```text
artifacts to fetch
store entries to reuse
project library leaves to add/update/remove
common software probes to run
special dependency files that still require Agent inspection
```

Planning 不包含宿主软件安装动作。

## `artifacts`

职责：把 GitHub Release asset 或显式 Git Package Root 变成已验证 Package snapshot。

```text
fetch_and_verify(source) -> VerifiedPackageSnapshot
```

内部负责：

- Release asset download；
- Git exact commit snapshot；
- SHA-256；
- safe tar extraction；
- Package Manifest 校验；
- `SKILL.md` 校验；
- `DEPENDENCIES.md` 存在性校验。

## `store`

机器级 Store 只保存不可变 Package payload。

```text
contains(digest)
put(snapshot)
get(digest)
```

Store 内部可以 content-addressed；它不承担项目可见的 `owner/repo/package` 目录结构。

## `activation`

职责：根据 Lock 构建分层 Project Skill Library：

```text
.akm/skills/<owner>/<repo>/<package>/
```

Package leaf 直接链接 machine Store 中的完整不可变 Package Root：

```text
.akm/skills/<owner>/<repo>/<package>
    -> <machine-store>/<exact-package-root>
```

依赖检查状态与 Package payload 分离，统一保存在 `.akm/dependencies.lock`。

executor adapter 如果需要把 Project Skill Library 转成某个执行器的 Skill discovery 结构，在 activation 之后工作；它不改变 core Package graph。

## `dependency_checks`

替代原来的 Software Catalog/Provider 安装体系。

职责只有：

- 对 AKM 内建支持的常见 `[software]` requirement 做只读 probe；
- 产出 `satisfied / missing / incompatible / unknown / blocked`；
- 读写项目本地 `.akm/dependencies.lock`；
- 根据 Package content/`DEPENDENCIES.md` digest 判断旧状态是否失效；
- 报告还有多少特殊依赖需要 Agent 检查。

Interface：

```text
probe_common(requirements) -> DependencyObservations
load_dependency_state(project) -> DependencyState
write_dependency_state(project, state)
```

该 Module 不提供：

```text
install
upgrade
remove
configure
```

## `application`

上层用例编排：

```text
install_target()
sync_project()
update_project()
remove_target()
doctor_project()
```

CLI/MCP/Python surface 以后都调用同一 application layer。

## 明确避免

v0 不建立：

- 抽象 Registry Package Identity 作为 GitHub 模式前置层；
- Multi-Skill Package；
- 扁平 global Skill name registry；
- Software Provider 自动安装体系；
- Package 自定义系统安装脚本；
- repository runtime shared directory；
- CLI command 内自行实现 dependency resolution。
