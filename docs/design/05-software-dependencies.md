# 软件依赖与 Provider v0 设计

## 目标

Agent Skill 经常依赖宿主机上的外部软件，例如 Git、GitHub CLI、Python、R、Node.js、samtools 或 bcftools。AKM 必须把这些要求作为一等依赖检查，但不能让 Package 自己决定如何修改宿主系统。

v0 的职责是：**声明、聚合、探测、解释、计划**。自动安装接口可以设计，但第一阶段默认不执行软件安装。

## Software Requirement

Package Manifest 示例：

```toml
[dependencies.software]
git = ">=2.40"
github-cli = ">=2.50"
python = ">=3.11,<3.14"
samtools = ">=1.20"
```

key 是 AKM Software Catalog 的稳定能力 ID，不必等于 executable 名或某个平台的 package-manager 名。

例如：

```text
github-cli
  executable: gh
  apt package: gh
  brew formula: gh
  winget package: GitHub.cli
```

这些映射属于 AKM Catalog/Provider，不属于 Skill Package。

## Software Catalog

Software Catalog 为每个能力定义：

```text
id
human name
version scheme
probe adapter
supported platforms
provider adapters
capabilities exposed
```

示意：

```text
id: python
version scheme: pep440
probe: PythonRuntimeProbe
providers:
  linux/apt: AptPythonProvider
  macos/homebrew: BrewPythonProvider
  windows/winget: WingetPythonProvider
```

v0 不要求 Catalog 一开始覆盖所有软件；未知 ID 必须进入显式 unsupported 状态，而不是忽略。

## 为什么不统一强制 SemVer

Package Release 使用 SemVer，但宿主软件生态并不统一：

- Python 版本与 Python package 通常更接近 PEP 440 语义；
- Node.js 使用 SemVer；
- Git、samtools、R 等有自己的发布习惯；
- 某些工具版本字符串包含 vendor suffix。

因此 Software Catalog 决定某个软件 ID 的 version parser 与 comparator。Package Manifest 只提供 requirement string；Package author 必须遵循对应 Catalog entry 的 scheme。

## Probe

Probe 的职责是纯读取：

```text
probe() -> SoftwareObservation
```

Observation 至少包含：

```text
status: present | missing | broken | unknown
version?: normalized version
raw_version?: original version text
location?: executable/runtime path
provider_hint?: detected installation origin
```

Probe 不修改 PATH、不自动安装、不自动升级。

一个 Software Requirement 的状态：

- `satisfied`；
- `missing`；
- `version-too-low` / `version-too-high`；
- `unparseable-version`；
- `unsupported-platform`；
- `unknown-software-id`。

## Requirement 聚合

多个 Package 可以要求同一软件能力：

```text
A -> python >=3.11
B -> python <3.14
C -> python >=3.12
```

Planner 聚合为：

```text
python >=3.12,<3.14
required-by: A, B, C
```

若 requirement 本身无交集，例如：

```text
A -> python <3.11
B -> python >=3.12
```

这不是 Package version solver 的 package conflict，而是 `SoftwareRequirementConflict`。Install Plan 必须显示冲突来源，不能尝试任选一个。

## Provider

Provider 是满足 Software Requirement 的平台适配器。逻辑接口：

```text
inspect(requirement) -> SoftwareObservation
plan(requirement, observation) -> SoftwareActionPlan
apply(approved_plan) -> SoftwareActionResult
```

关键约束：

- Package 不能提供 Provider implementation；
- Package 不能覆盖 Software Catalog 的 probe/provider 映射；
- Provider 生成的计划必须具体到平台、目标版本/范围和预期变更；
- `apply` 只能接收已经被用户批准、且与当前 observation 仍匹配的 plan；
- observation 发生变化时必须重新计划，避免执行过期计划。

## v0.1 执行策略

第一实现阶段建议：

- 实现 Software Catalog；
- 实现常见软件 Probe；
- 实现 requirement aggregation；
- 在 Install Plan 中报告缺失与不兼容；
- **不自动执行 Provider 安装**。

此时计划可以输出：

```text
BLOCKED SOFTWARE
- samtools >=1.20: missing
  required by akira/ngs@2.0.1
  provider candidates: apt, conda, brew ...
```

但不会在一次 `sync` 中自行修改系统。

自动 Provider 安装作为后续独立 milestone 加入；它不改变 Package Manifest、Resolver 或 Project Lock 结构。

## Runtime 与普通 executable

v0 不在 Package Manifest 语法上分裂 `[runtimes]` 与 `[software]`。

原因：从 Package 的角度，`python`、`R`、`node` 与 `samtools` 都是宿主能力要求；它们的差异由 Software Catalog entry 表达。

Catalog 可以为 runtime 附加：

```text
kind = runtime
can_host_libraries = true
```

以后如果需要管理 Python package、R package、Node package 等“运行时内部依赖”，应引入独立的 Runtime Environment 模型，不把它们与宿主 executable requirement 混成同一个字符串表。

## Python/R/Node 内部库依赖

v0 暂不允许 Package 直接写：

```text
pip install ...
npm install ...
R install.packages(...)
```

如果 Skill bundled script 依赖第三方语言库，第一阶段有两种合法方式：

1. Skill 自己采用可重建的项目环境并把启动逻辑封装在 script 内，Package Manifest 只声明所需 runtime；
2. 等 AKM Runtime Environment 模型稳定后，声明结构化 runtime package dependencies。

这避免 v0 同时重造 pip/npm/renv。

## 系统能力而非软件名称

未来 Catalog 可以支持抽象能力，例如：

```text
container-runtime
c-compiler
cuda-runtime
```

由不同 Provider 实现 Docker/Podman、gcc/clang、不同 CUDA 安装来源。

但 v0 优先覆盖有明确 identity 与 probe 的软件，不提前抽象尚未出现的需求。

## 软件状态不进入 Project Lock

Project Lock 保存 Requirement closure 和 required-by provenance，不保存某台机器探测到的安装路径或当前版本。

host observation 属于本地 runtime state，可以缓存于 `.akm/state` 或机器 cache，但它必须可随时重新 probe，不得成为跨机器可重建性的权威来源。
