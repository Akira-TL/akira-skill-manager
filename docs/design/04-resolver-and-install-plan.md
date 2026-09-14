# Dependency Resolver 与 Install Plan v0 工作草案

## 1. Resolver 要解决什么

AKM v0 不是对一个抽象 Registry Package 世界求解，而是对 GitHub repositories 的 Release versions 与其中的 Package selectors 求解。

顶层输入例如：

```text
Akira-TL/matt-skills/ask-matt ^1.4
Akira-TL/skills/browser-access ^2.0
```

Package Manifest 再引入：

```toml
[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
"Akira-TL/matt-skills/wayfinder" = "^1.4"
```

Resolver 最终确定：

- 每个 GitHub repository 使用哪个精确 Release version；
- 每个 Release 中真正需要哪些 Package Artifact；
- 完整 transitive dependency graph；
- 哪些 source 显式使用 Git 模式。

## 2. GitHub 模式的版本求解单位

GitHub v0 中 version 属于 repository Release。

例如同一 dependency graph 出现：

```text
Akira-TL/matt-skills/ask-matt    ^1.4
Akira-TL/matt-skills/implement   >=1.4 <2
Akira-TL/matt-skills/tdd         ^1.5
```

这些不是三个彼此独立的 Package version choice，而是共同约束：

```text
repository = Akira-TL/matt-skills
release version must satisfy all relevant ranges
```

一旦选中 Release `1.6.2`，需要的 Package 分别取：

```text
ask-matt.akm.tar.gz
implement.akm.tar.gz
tdd.akm.tar.gz
```

这保持 `owner/repo/package@version` 的语义简单。

## 3. 同名 Skill 不属于 AKM resolver 冲突

AKM Project Skill Library 是：

```text
skills/<owner>/<repo>/<package>/
```

所以：

```text
A/repo/foo
B/repo/foo
```

可以同时存在。

AKM core 不再定义 `SkillNameCollision`。

如果某个执行器只能接受扁平 Skill namespace，冲突由 executor adapter 在生成该执行器视图时处理或报告；这不影响 Package graph 本身是否合法。

## 4. Dependency cycle

Skill dependency graph 仍不允许 cycle：

```text
A -> B -> C -> A
```

原因不是文件名冲突，而是安装/能力依赖关系无法形成清晰的 dependency closure。

Resolver 必须报告完整 cycle path。

## 5. Source 模式

### GitHub Release

默认模式。候选版本来自 repository 的 Releases。

### Git

只有顶层 CLI/Project Manifest 显式指定后才进入。

Git 模式至少锁定：

```text
repository
requested ref
exact commit
selected Package Root(s)
content digest
```

Release 不存在时不能自动改用 Git。

Git 与 Release 混用同一个 repository 的详细约束仍需单独设计；v0 应优先避免同一 project resolution 同时从同一 repo 的 Release 与 Git checkout 取不同 Package。

## 6. Resolver 输入

概念接口：

```text
resolve(
  project_requirements,
  release_source,
  explicit_git_sources,
  previous_lock?,
  mode,
) -> Resolution | ResolutionError
```

`release_source` 对 GitHub 至少提供：

```text
available_releases(owner, repo)
package_manifest(owner, repo, release, package)
package_asset(owner, repo, release, package)
```

未来 Registry adapter 可以实现另一套 source interface，但不要求 GitHub dependency 先映射到 Registry ID。

## 7. 版本选择

普通 `sync`：

1. previous lock 的 repository Release 仍满足当前全部 range 时优先保留；
2. 需要更新时选择满足全部约束的候选 Release；
3. 默认选择最高 compatible stable Release；
4. prerelease 只有显式允许时参与；
5. Package Artifact 在选中的 Release 中不存在时，该 Release 对相应 requirement 不可用。

冲突示例：

```text
A requires owner/repo/x <2
B requires owner/repo/y >=3
```

因为 x/y 属于同一 repository Release version 空间，没有一个 Release 同时满足，必须报告版本冲突。

## 8. Repository-wide top-level install

用户可以显式安装：

```text
owner/repo@1.4.0
```

这表示：

1. 选择 Release `1.4.0`；
2. 枚举该 Release 中全部 `*.akm.tar.gz`；
3. 每个 Artifact 校验为合法 Package；
4. 将这些 Package 全部视为 top-level selected Packages；
5. 继续解析它们的 dependency closure。

Repository-wide target 只允许作为用户顶层意图，不允许 Package dependency 写成 `owner/repo`。

## 9. Resolution 输出

至少包含：

```text
repositories:
  exact GitHub Release or Git commit records

packages:
  exact owner/repo/package selections

edges:
  exact dependency edges

common_software_requirements:
  package -> [software] requirements

warnings:
  dependency checks that still require Agent inspection
```

Resolver 不安装软件，也不决定特殊依赖如何解决。

## 10. Install Plan

Planner 把 Resolution 与本地状态组合成：

### Fetch

- 哪些 Release Artifact 需要下载；
- 哪些显式 Git source 需要 checkout；
- 哪些 Package 已在 machine Store。

### Verify

- Asset digest；
- safe extraction；
- `package.name/version`；
- `SKILL.md.name`；
- `DEPENDENCIES.md` 存在。

### Project Skill Library

目标路径始终按：

```text
<owner>/<repo>/<package>/
```

创建/更新/移除 activation leaf。

### Dependency Check

- 对 `[software]` 中 AKM 已知常见软件运行只读 probe；
- 更新项目侧 `DEPENDENCIES.md` Current status；
- 标出仍需 Agent 检查的特殊依赖。

AKM 不生成 `apt/brew/winget/...` 修复动作。

## 11. 执行顺序

```text
parse project requirements
  -> resolve repository releases / git commits
  -> resolve package closure
  -> build fetch plan
  -> fetch & verify Package Artifacts
  -> put immutable payload in machine Store
  -> materialize hierarchical Project Skill Library
  -> run common software probes
  -> update project-local DEPENDENCIES.md
  -> write Lock atomically
```

需要修改宿主软件环境的工作发生在 AKM 之后：Agent 读取 `DEPENDENCIES.md`，说明缺口，获得用户明确同意后再使用当前环境真实可用的方式处理。

## 12. Frozen 与 offline

### `frozen`

只接受 Lock 已确定的 exact Release/Git commit 和 Package graph；Manifest/Project requirement 不一致就失败，不重新求解。

### `offline`

不访问 GitHub，也不 clone/fetch Git；只能使用 Lock 与本地 machine Store/cache 已存在内容。

二者相互独立。

## 13. remove / orphan / why

删除顶层 target 后重新计算 dependency closure。

新的 graph 中不可达 Package 从 Project Skill Library 移除；machine Store 内容进入 GC candidate。

`why owner/repo/package` 从 Lock graph 反向构造依赖路径，不维护第二套 reverse-dependency 状态。

## 14. 结构化错误

至少区分：

- `NoReleaseSolution`；
- `UnavailableRelease`；
- `UnavailablePackageAsset`；
- `DependencyCycle`；
- `InvalidPackageManifest`；
- `GitSourceNotExplicit`；
- `ArtifactIntegrityMismatch`；
- `UnsafeArtifact`。

执行器层的扁平 Skill name collision 不属于 core resolver error。
