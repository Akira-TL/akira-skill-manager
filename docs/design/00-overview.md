# AKM v0 协议工作草案

## 当前阶段

当前仍处于协议收敛阶段，不进入 CLI/runtime 实现。以下文档是当前 working draft；已经明确拒绝的旧方案不再作为实现候选。

设计文件：

1. [`skill-package-layout.md`](skill-package-layout.md)
2. [`repository-discovery.md`](repository-discovery.md)
3. [`01-package-manifest.md`](01-package-manifest.md)
4. [`02-release-artifact.md`](02-release-artifact.md)
5. [`03-project-manifest-lock.md`](03-project-manifest-lock.md)
6. [`04-resolver-and-install-plan.md`](04-resolver-and-install-plan.md)
7. [`05-software-dependencies.md`](05-software-dependencies.md)
8. [`06-module-boundaries.md`](06-module-boundaries.md)

## 已明确的 v0 方向

- GitHub 是首个直接分发坐标系，不先建设独立 Registry；
- 用户安装目标使用 `<owner>/<repo>[/<package>]@<version-or-ref>`；
- 默认优先使用 GitHub Release；没有 Release 或明确要源码版本时显式进入 Git source；
- Release 不要求仓库作者提供 AKM 专用 Asset，源码归档即可 discovery；AKM Asset 只是可选优化；
- 一个 Skill Package 恰好包含一个 Skill；
- Package Root 与 Skill Root 重合；
- **唯一最低准入条件是合法 `SKILL.md`**；
- `akm-package.toml` 是可选结构化依赖元数据；
- `DEPENDENCIES.md` 是可选 Agent-readable 特殊依赖说明；
- Multi-Skill Package 不进入 v0；
- Router 是普通 Skill，能力族由 Router + 可见 dependency closure 形成；
- 运行时共享能力必须写成 dependency，不允许跨 Package hidden shared files；
- GitHub Release `@version` 对应 repository Release version；
- Git source `@ref` 最终锁定 exact commit；
- Git source 进入机器级 disposable source cache，再从 exact commit discovery/snapshot Skill Root；
- repository 默认零配置扫描合法 `SKILL.md`；可选 root-level `akm-repo.toml` 作为 discovery include/exclude 过滤提案，不改变 `SKILL.md` 的准入地位；
- Project Skill Library 按 `<owner>/<repo>/<package>` 分层，不由 AKM core 扁平化；
- 同名 Skill 在 AKM library 层可以共存；最终 discovery 交给 executor adapter/执行器；
- Package payload 机器级共享且 immutable；
- 当前宿主依赖状态单独写入 `.akm/dependencies.lock`，永不改 Package 文件；
- AKM 只基础探测少量常见软件，不负责自动安装/修复宿主依赖。

## 核心关系

```text
GitHub repository snapshot
    ↓
SKILL.md discovery
    ├── Package A == Skill A
    ├── Package B == Skill B
    └── Package C == Skill C

optional akm-package.toml
    └── structured dependencies/software

optional DEPENDENCIES.md
    └── Agent-readable special requirements

Git Source Cache
    └── repository Git objects / exact commits

Package Store
    └── immutable selected Skill Root snapshots

Project Skill Library
    └── <owner>/<repo>/<package>/ -> Store
```

## 当前没有的东西

v0 不提前引入：

- 必填 `akm-package.toml`；
- 独立 Registry namespace/package identity；
- Multi-Skill bundle；
- Package Index 作为 GitHub 的强制中间层；
- 扁平 Skill name 全局唯一约束；
- Software Provider 自动安装体系；
- Package 自定义系统安装脚本；
- repository runtime shared directory；
- 项目直接引用 mutable Git checkout。

## 下一步仍需收敛

- GitHub Release tag/version 的严格命名规则；
- Release source archive 与可选 AKM Asset 的优先级/完整性契约；
- Git source 与 Release source 同 repo 混用是否完全禁止；
- `akm-repo.toml` 的最终命名、glob grammar 与 nested selected Skill Root 行为；
- `.akm/dependencies.lock` 的最终字段与状态失效规则；
- Project Skill Library 到不同 executor 的发现适配；
- version range 的最终 grammar；
- Lock 的 canonical TOML 结构；
- Git Source Cache GC 与 Package Store GC 的策略；
- optional dependencies / feature flags 是否需要进入后续版本。

## 协议稳定后的候选实现顺序

1. GitHub coordinate + `SKILL.md` metadata parser；
2. Release/Git source cache + Package discovery；
3. optional AKM metadata parser + dependency resolver；
4. immutable Package Store；
5. hierarchical Project Skill Library activation；
6. common dependency probes + `.akm/dependencies.lock`；
7. executor adapters；
8. CLI/MCP surface。