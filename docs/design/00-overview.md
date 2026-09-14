# AKM v0 协议工作草案

## 当前阶段

当前仍处于协议收敛阶段，不进入 CLI/runtime 实现。以下文档是当前 working draft；已经明确拒绝的旧方案不再作为实现候选。

设计文件：

1. [`skill-package-layout.md`](skill-package-layout.md)
2. [`01-package-manifest.md`](01-package-manifest.md)
3. [`02-release-artifact.md`](02-release-artifact.md)
4. [`03-project-manifest-lock.md`](03-project-manifest-lock.md)
5. [`04-resolver-and-install-plan.md`](04-resolver-and-install-plan.md)
6. [`05-software-dependencies.md`](05-software-dependencies.md)
7. [`06-module-boundaries.md`](06-module-boundaries.md)

## 已明确的 v0 方向

- GitHub 是首个直接分发坐标系，不先建设独立 Registry；
- 用户安装目标使用 `<owner>/<repo>[/<package>]@<version>`；
- 默认从 GitHub Release 获取；Git clone 必须显式启用；
- 一个 Skill Package 恰好包含一个 Skill；
- Package Root 与 Skill Root 重合；
- Multi-Skill Package 不进入 v0；
- Router 是普通 Skill Package，产品能力族由 Router + transitive dependencies 形成；
- 运行时共享能力必须写成 dependency，不允许跨 Package hidden shared files；
- GitHub dependency 直接使用 `<owner>/<repo>/<package>` + version range；
- GitHub `@version` 对应 repository Release version；
- Project Skill Library 按 `<owner>/<repo>/<package>` 分层，不由 AKM core 扁平化；
- 同名 Skill 在 AKM library 层可以共存；最终 discovery 交给 executor adapter/执行器；
- 每个原生 Package 包含 `DEPENDENCIES.md`；
- AKM 只基础探测少量常见软件并写状态，不负责自动安装/修复宿主依赖；
- 特殊软件、硬件、服务、数据、授权等依赖由 Agent 根据 `DEPENDENCIES.md` 检查，涉及环境修改时先与用户确认；
- Package payload 机器级共享，项目侧 Package leaf 可以作为 activation overlay 保存可写 dependency status。

## 核心关系

```text
GitHub repository
    ├── Package A == Skill A
    ├── Package B == Skill B
    └── Package C == Skill C

Router Skill
    └── dependencies -> other Skill Packages

GitHub Release version
    └── multiple independent Package Artifacts

Project Skill Library
    └── <owner>/<repo>/<package>/
```

## 当前没有的东西

v0 不提前引入：

- 独立 Registry namespace/package identity；
- Multi-Skill bundle；
- Package Index 作为 GitHub 的强制中间层；
- 扁平 Skill name 全局唯一约束；
- Software Provider 自动安装体系；
- Package 自定义系统安装脚本；
- repository runtime shared directory。

## 下一步仍需收敛

- GitHub Release tag/version 的严格命名规则；
- Release asset integrity/attestation 的最小契约；
- Git source 与 Release source 同 repo 混用是否完全禁止；
- `DEPENDENCIES.md` 的精确可编辑区域/升级合并算法；
- Project Skill Library 到不同 executor 的发现适配；
- version range 的最终 grammar；
- Lock 的 canonical TOML 结构；
- Store GC 与跨项目引用发现；
- optional dependencies / feature flags 是否需要进入后续版本。

## 协议稳定后的候选实现顺序

1. GitHub coordinate + Package/Project metadata parser；
2. GitHub Release/Git source resolver；
3. Artifact verification + immutable Store；
4. hierarchical Project Skill Library activation；
5. common dependency probes + `DEPENDENCIES.md` status；
6. executor adapters；
7. CLI/MCP surface。
