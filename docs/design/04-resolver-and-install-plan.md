# Dependency Resolver 与 Install Plan v0 设计

## 目标

resolver 只回答一个问题：给定项目顶层需求、source policy、Package Index 和已有 Lock，应选择哪一组精确 Package Release 才能满足全部约束。

resolver 不直接下载 Package、不创建 symlink、不安装软件。所有持久化动作必须先被转换成 Install Plan。

## 核心不变量

1. 同一 Project Environment 中一个 Package Identity 最多一个版本；
2. 所有选中 Package 必须从顶层 Project Requirement 可达；
3. 所有 Skill dependency version range 必须被满足；
4. 一个 Package Identity 在一次解析中只能绑定一个 source policy；
5. 两个不同 Package Identity 不得导出同一个 Skill name；
6. 最终 Package dependency graph 不允许有 cycle；
7. unsupported platform / software requirement 不得被伪装成成功安装；
8. resolver 输出必须是纯数据，可在不修改文件系统的测试中完整验证。

## Resolver 输入

逻辑接口：

```text
resolve(
  project_requirements,
  source_policy,
  package_index,
  previous_lock?,
  resolution_mode,
) -> Resolution | ResolutionError
```

### `project_requirements`

来自 `akm.toml [skills]` 的顶层 Package Identity + version range。

### `source_policy`

决定每个 Package Identity 从哪个 Package Index / source override 获取候选版本。Package Manifest 自身不能修改它。

### `package_index`

resolver 只依赖一个小接口：

```text
available_versions(package) -> ordered version metadata
manifest(package, exact_version) -> Package Manifest
artifact_metadata(package, exact_version) -> locator + integrity
```

GitHub Releases、未来 registry、本地测试 index 都是这个 seam 上的 Adapter。

resolver 获取 manifest metadata 不等于下载 Release Artifact。

### `previous_lock`

可选。用于普通 `sync` 中优先复用仍满足全部约束的 exact version，降低无关依赖漂移。

### `resolution_mode`

至少包括：

- `sync`；
- `update-all`；
- `update-selected(package set)`；
- `frozen`。

`offline` 是 source access policy，不与 resolution mode 混为一个枚举。

## 版本选择顺序

对同一 Package Identity：

1. 如果 previous lock exact version 仍满足全部约束、source policy 和 compatibility，则优先选择；
2. 如果该 Package 被显式 update，则不使用其 lock preference；
3. 在剩余候选中优先选择最高的 compatible stable version；
4. prerelease 只有在顶层或 transitive range 显式允许时才参与；
5. yanked Release 不作为新选择，但 previous lock 已锁定的 yanked version 可以继续使用并产生 warning，除非 Package Index 标记为 revoked/security-blocked。

“最高版本”只是 candidate preference，不改变约束正确性。

## 算法选择

v0 设计采用 PubGrub 风格的 incompatibility solver 作为首选实现方向。

原因：

- 它的标准问题定义就是“每个 Package 选至多一个版本”；
- 支持 transitive version constraints；
- 冲突时能生成 derivation tree，而不是只返回“没有解”；
- package metadata 可以 lazy 获取，不需要先枚举整个生态；
- 与 AKM 的 Package Index seam 相容。

参考：[Dart pub solver design](https://github.com/dart-lang/pub/blob/master/doc/solver.md)。

算法本身必须封装在 `resolution` Module 内；未来换算法不能改变 Manifest/Lock 协议。

## Cycle

PubGrub 的“版本有解”不代表 AKM 的 Skill 依赖图符合领域规则。

因此 version solve 成功后必须对 exact dependency graph 运行 cycle detection。发现：

```text
A -> B -> C -> A
```

必须失败并报告完整 cycle path。

v0 不区分 build-time/runtime cycle；所有 Skill dependency cycle 都拒绝。

## Skill name collision

解析后读取每个 Package 的 exported Skill name。若：

```text
namespace-a/foo -> skill name = foo
namespace-b/foo -> skill name = foo
```

且两者同时进入 graph，则失败。

这与 Package Identity 冲突不同：即使 package 名不同，项目激活视图仍只能拥有一个 `foo`。

## Source conflict

同一 Package Identity 不能在一部分依赖链使用 registry Release、另一部分使用 Git override。

Project source override 一旦存在，就对该 Package Identity 的全部解析生效；若 override 的 Manifest identity/version 不满足依赖要求，解析失败。

这避免 dependency confusion 和 provenance 漂移。

## Frozen

`frozen` 不执行普通求解：

1. 校验 `manifest-digest`；
2. 校验 Lock 的 Package Graph 自洽；
3. 校验每条 top-level / transitive constraint 均被 Lock exact version 满足；
4. 校验 source policy 与 Lock provenance 没有冲突；
5. 直接返回 Lock 对应的 Resolution；
6. 任一条件不满足即报错，不生成新版本选择。

## Resolution 输出

Resolution 至少包含：

```text
packages: exact package records
edges: exact dependency edges
software_requirements: aggregated requirements + provenance
warnings: yanked/deprecated/non-portable source 等
```

Resolution 不包含“应该下载哪些文件”，因为当前 Store/cache 状态尚未参与。

## Planner

Planner 接收纯 Resolution 与当前机器/项目状态：

```text
plan(
  resolution,
  machine_store_state,
  project_activation_state,
  software_state,
  trust_state,
) -> InstallPlan
```

Install Plan 至少分为五类动作：

### Package fetch

哪些 exact Package content 当前 Store 缺失，需要从哪个 immutable source 获取和验证。

### Package keep

哪些 content 已存在且 integrity 匹配，不需要重复获取。

### Activation changes

哪些 Project Skill View link 需要新增、切换或移除。

### Software status

哪些 Software Requirement 已满足、缺失、版本过低或当前平台无 Provider。

### Approval requirements

哪些动作需要用户授权，例如：

- 首次使用一个项目 source override；
- Provider 计划对宿主软件环境做修改；
- 非 portable path source；
- 其他策略层要求显式批准的来源。

## “先计划，再执行”

执行顺序固定为：

```text
parse
  -> resolve
  -> validate graph
  -> inspect machine state
  -> build complete InstallPlan
  -> present / authorize
  -> fetch & verify packages
  -> optional software provider actions
  -> activate project links
  -> write lock/state atomically
```

在授权前允许的操作仅限读取本地状态、读取已批准 Package Index metadata 以及计算计划所必需的只读 source metadata。

任何会改变 host 软件、Project Skill View、Package Store 最终状态或 Lock 的动作都属于执行阶段。

## 执行原子性

Package fetch 可以先进入临时 cache，但只有校验成功的 content 才能原子进入 Store。

Project Skill View 采用 staging directory：

1. 在 `.akm/.staging-<id>/skills` 创建全部目标 symlink；
2. 验证每个 target 已存在且匹配 Lock；
3. 原子替换 `.akm/skills`；
4. 失败则保留旧 view。

Lock 写入使用 temp file + atomic replace，避免产生半写文件。

## remove

remove 是“修改 Project Requirement + 重新求解”，不是对 Store 的破坏性操作。

若用户删除顶层 `A`：

- 仍被其他顶层包需要的 transitive dependency 保留；
- 完全不可达的 Package 从新 Lock 与 Project Skill View 消失；
- Store content 进入可 GC 候选，而不是立即删除。

## reverse dependency 与 `why`

Reverse Dependency 从 exact graph 反向索引，不单独持久化。

`why C` 应能显示至少一条从顶层 requirement 到 `C` 的路径；remove/update plan 可以利用同一索引解释影响范围。

## 冲突错误模型

ResolutionError 不应只有字符串。至少区分：

- `NoVersionSolution`：版本约束无解，并带 derivation；
- `DependencyCycle`：带 cycle path；
- `SkillNameCollision`：带冲突 Package；
- `SourceConflict`：带 source policy 与 dependency provenance；
- `UnavailablePackage` / `UnavailableVersion`；
- `IncompatiblePlatform`；
- `InvalidManifest`。

CLI 最终只是这些结构化错误的一个 renderer。
