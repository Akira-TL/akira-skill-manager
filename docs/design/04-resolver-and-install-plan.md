# Dependency Resolver 与 Install Plan v0 工作草案

## 1. Resolver 的对象

AKM v0 直接解析 GitHub source：

```text
owner/repo[/package]@version-or-ref
```

它不要求 Skill 作者先注册到独立 Registry，也不要求存在 `akm-package.toml`。

每个被选择的 Skill Package 至少由以下信息确定：

```text
owner/repo
source kind: release | git
exact source snapshot: release+commit | exact commit
repository-relative package-root
SKILL.md.name
content digest
```

如果可选 `akm-package.toml` 存在，Resolver 再展开其结构化 Skill dependencies。

## 2. Release source

默认 source 是 GitHub Release。

Release version 属于 repository。例如：

```text
owner/repo/foo@1.4.0
owner/repo/bar@^1.4
```

如果 `foo` 与 `bar` 来自同一个 repository，则约束共同作用于 repository Release version。

选定 Release 后，AKM 可以：

1. 使用 package-specific AKM Asset（若存在）；或
2. 下载该 Release/tag 对应 repository source archive；
3. 从版本快照中按 `SKILL.md` discovery 找到 Package Root。

Package 没有 Manifest 时仍是合法 leaf Package，只是没有 AKM 可见的结构化 transitive dependency。

## 3. Git source

没有 Release 或明确需要源码版本时，用户显式选择 Git source：

```text
owner/repo/foo@main --git
```

AKM：

1. 从机器级 Git Source Cache 取得/fetch repository；
2. 把 requested ref 解析为 exact commit；
3. 基于 exact commit 的 tracked tree 做 Skill discovery；
4. snapshot 选中的 Skill Root 到 immutable Package Store；
5. Lock exact commit、package-root 与 content digest。

Git source 不直接把 mutable checkout 暴露给项目。

## 4. Git Source Cache

逻辑布局：

```text
~/.cache/akm/git/github.com/<owner>/<repo>.git/
```

它适合实现为 bare/mirror-style repository cache：

- 同一 repository 只缓存一份 Git objects；
- 不同项目、不同 branch/tag/commit 可以复用；
- ref 更新只 fetch 增量 objects；
- discovery 可以直接读 Git tree，真正需要 snapshot 时再临时 materialize；
- cache 可以删除并重新获取，不是项目状态真相。

Source Cache 不进入 `akm.lock` 的本机绝对路径。Lock 只保存可重建 provenance：repository、requested ref、exact commit、package-root、content digest。

## 5. Package discovery

Git checkout/package path **不能由安装坐标提前推出**。

统一 discovery anchor 是 `SKILL.md`。

推荐算法：

1. 取得 exact repository snapshot；
2. 枚举版本化 tree 中所有 `SKILL.md`；
3. 每个文件父目录成为 Package Root candidate；
4. 解析 `SKILL.md` frontmatter `name`；
5. 校验：

```text
basename(package-root) == SKILL.md.name
```

6. 如果 repository root 存在可选 `akm-repo.toml`，对 candidate 的 repository-relative Package Root path 应用 `include` / `exclude`；没有配置时等价于全量候选；
7. 过滤后如果同一个 `SKILL.md.name` 出现多个 candidate，报告 `AmbiguousPackageDiscovery`；
8. 过滤后 Package Root 若互相嵌套，当前草案报告 `NestedPackageDiscovery`，不自动选浅层或深层；
9. 如果 `akm-package.toml` 存在，解析依赖增强信息；
10. 如果 `DEPENDENCIES.md` 存在，记录其 digest；
11. selector 存在时按 `SKILL.md.name` 匹配；
12. selector 省略时选择全部最终合法 Package Roots；
13. Lock 保存实际 repository-relative path。

Repository-level discovery control 的完整候选语义见 [`repository-discovery.md`](repository-discovery.md)。

例如：

```text
repo/
├── agent-tools/routers/ask-matt/SKILL.md
└── engineering/tdd/SKILL.md
```

安装：

```text
owner/repo/ask-matt@main --git
```

不需要用户知道 `agent-tools/routers/ask-matt`。

## 6. 同 repository source 一致性

一个 project resolution 中，同一个 `owner/repo` 只绑定一个 source snapshot：

```text
Release X (+ exact commit)
或
Git exact commit Y
```

不允许：

```text
foo <- Release 1.4.0
bar <- Git main
```

来自同一个 repository 的 Package 必须来自同一 snapshot。

Git source 下，如果一个有 Manifest 的 Skill 声明同 repository sibling dependency：

```toml
[dependencies]
"owner/repo/helper" = "^1.4"
```

当前 explicit Git binding 优先：`helper` 从同一个 exact commit discovery/snapshot，不再切回 Release。该 dependency 的 Release range 在 Git override 下不作为版本选择条件；Lock 明确记录 source-kind=git，使这种开发态 override 可审计。

跨 repository dependency 没有显式 Git override 时仍按 Release source 解析。

## 7. Version resolution

普通 Release `sync`：

1. previous Lock 的 exact Release 仍满足全部 repository ranges 时优先保留；
2. 否则枚举符合约束的 Release；
3. 默认优先最高 compatible stable Release；
4. prerelease 只有显式允许时参与；
5. 选中的 Release snapshot 必须能 discovery 到所需 `SKILL.md.name`。

如果 Package 没有 Manifest，不产生新的 transitive version constraints。

## 8. Dependency graph

Skill dependency 只来自可选 Manifest：

```toml
[dependencies]
"owner/repo/helper" = "^1.0"
```

没有 Manifest：

```text
AKM graph node has no declared outgoing Skill edges
```

AKM 不从 `SKILL.md` 自然语言、目录名称或引用文件中猜测结构化 dependency。

Dependency graph v0 不允许 cycle：

```text
A -> B -> C -> A
```

发现时报告完整 cycle path。

## 9. 同名 Skill

Project Skill Library 保留：

```text
<owner>/<repo>/<package>
```

因此：

```text
A/repo/foo
B/repo/foo
```

在 AKM core 层可以共存。

执行器若需要扁平 namespace，由 executor adapter 解决冲突，不属于 Package Resolver 错误。

## 10. Repository-wide install

用户安装：

```text
owner/repo@1.4.0
```

Release 模式：

1. 取得 Release source snapshot；
2. discover 全部合法 `SKILL.md` Package Roots；
3. 全部作为顶层选择；
4. 对存在 Manifest 的 Package 展开 dependency closure。

Git 模式：

```text
owner/repo@main --git
```

同理，只是 source 来自 cached Git exact commit。

Repository-wide target 只作为用户顶层意图；Manifest dependency 必须精确到 `owner/repo/package`。

## 11. Resolution 输出

至少包含：

```text
repositories:
  exact release/tag/commit or exact git commit

packages:
  owner/repo/package
  SKILL.md.name
  package-root
  content digest
  optional manifest digest
  optional dependencies document digest

edges:
  manifest-declared exact dependency edges

common software requirements:
  optional manifest [software]

warnings:
  optional dependency checks still requiring Agent inspection
```

## 12. Install Plan

### Source fetch

- 哪些 Release archive/Asset 需要下载；
- 哪些 Git cache 需要 clone/fetch；
- 哪些 Store snapshot 已存在可复用。

### Discovery

- 每个 repository snapshot 发现哪些 `SKILL.md` roots；
- root `akm-repo.toml`（若存在）的 include/exclude 过滤结果；
- selector 最终匹配哪个 relative path；
- 是否有重名/嵌套/非法 frontmatter。

### Verify

- Release/Asset/Git source provenance；
- archive/path safety；
- `SKILL.md`；
- optional `akm-package.toml`；
- optional `DEPENDENCIES.md`；
- content digest。

### Activate

按：

```text
.akm/skills/<owner>/<repo>/<package>
    -> <machine-store>/<content-digest>
```

建立只读项目库。

### Dependency check

只有存在 `[software]` 才运行对应 common probes；只有存在 `DEPENDENCIES.md` 才提示 Agent 存在特殊依赖说明。检查结果写 `.akm/dependencies.lock`。

## 13. 执行顺序

```text
parse target/project manifest
  -> choose release or explicit git source
  -> obtain exact repository snapshot
  -> discover SKILL.md roots
  -> select requested package(s)
  -> read optional AKM metadata
  -> expand dependency closure
  -> build complete plan
  -> fetch/materialize selected roots
  -> verify + put immutable Store
  -> rebuild Project Skill Library
  -> run common probes
  -> update .akm/dependencies.lock
  -> write akm.lock atomically
```

## 14. Frozen / offline

`frozen`：只接受 Lock 的 exact source snapshot、package-root、content digest 和 graph，不重新选择。

`offline`：不访问 GitHub、不 fetch Git；只能使用本地 cache/Store 与 Lock。

当 Store 已有需要的 Package snapshot 时，即使 Git source cache 被 GC，也能离线激活；如果 Store 缺失而只剩 Lock，没有对应 source cache/archive，则 offline 失败。

## 15. remove / orphan / why

删除顶层 target 后重新计算 manifest-declared dependency closure。不可达 Package 从 Project Skill Library 移除；Store 进入独立 GC 候选。

`why owner/repo/package` 从 Lock graph 反向构造路径。

## 16. 结构化错误

至少区分：

- `UnavailableRelease`；
- `UnavailableGitRef`；
- `PackageNotFound`；
- `AmbiguousPackageDiscovery`；
- `NestedPackageRoot`；
- `InvalidSkillMetadata`；
- `InvalidOptionalManifest`；
- `DependencyCycle`；
- `ArtifactIntegrityMismatch`；
- `UnsafeArtifact`；
- `OfflineSourceUnavailable`。