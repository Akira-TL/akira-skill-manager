# Akira sKill Manager 项目规则

Akira sKill Manager（AKM）是独立的 Agent Skill 包管理器项目。当前阶段仍处于协议与领域模型设计期；`docs/design/` 与 `docs/adr/` 中未明确标记为 `Accepted` 的内容都属于可修改提案，不得当作最终协议或直接据此进入实现。

## Agent skills

### Issue tracker

项目使用 GitHub Issues 记录 Spec、Ticket、Wayfinder 决策项与外部请求。具体操作见 `docs/agents/issue-tracker.md`。

### Workflow roles

Matt Engineering 使用 canonical workflow role 名称作为 GitHub label。映射见 `docs/agents/triage-labels.md`。

### Domain docs

本项目采用 single-context：先读取根目录 `CONTEXT.md`，涉及架构决定时再读取相关 `docs/adr/`。消费规则见 `docs/agents/domain.md`。

## 当前设计边界

- 先完成 Skill 本体目录/文件结构、Package Manifest、Release Artifact、Project Manifest/Lock、Dependency Resolver 与软件依赖模型的协议设计，再进入实现。
- v0 使用 GitHub source model：安装目标采用 `<owner>/<repo>[/<package>]@<version-or-ref>`；默认优先从 GitHub Release 获取，没有合适 Release 或明确需要源码版本时才显式进入 Git source。
- 一个 Skill Package 恰好对应一个标准 Agent Skill；Package Root 与 Skill Root 重合，Multi-Skill Package 不进入 v0。
- 合法 `SKILL.md` 是唯一最低准入条件；`akm-package.toml` 与 `DEPENDENCIES.md` 都是可选增强文件，不得把普通第三方 Skill 降级成兼容特例。
- Git source 使用机器级 disposable Source Cache；Package discovery 基于 exact repository snapshot 中的 `SKILL.md`，最终项目只链接 immutable Package Store，不直接引用 mutable checkout/cache。
- 产品能力族通过 Router Skill + 可见 transitive Skill dependencies 形成，不通过 bundle Artifact 表达。
- Project Skill Library 按 `<owner>/<repo>/<package>` 分层，不在 AKM 核心中扁平化同名 Skill；Skill discovery 交给 executor adapter/执行器。
- 运行时共享能力必须显式写成 Skill dependency；不允许 Package 依赖 Package Root 外的 runtime 文件。
- 可选 `DEPENDENCIES.md` 用于 Agent 检查复杂软件/环境依赖；AKM 只对可选 Manifest 中少量常见软件要求做基础只读探测，不负责自动修复宿主环境。
- 旧 `/home/Akira/Projects/akira-skills` 仅作为迁移与既有行为参考，不在其中实现 AKM。

## 工程约定

- 项目显示名称统一写作 `Akira sKill Manager`，缩写为 `AKM`。
- Python 工程遵循全局 `uv` 与 `.venv` 规则；当前协议未稳定前不要为了占位提前建立 CLI 或运行时骨架。
- 仓库存在 `.codegraph/` 后，理解或定位代码时优先使用 CodeGraph；索引数据库是机器本地状态，不进入 Git。
- 协议设计中的新决定只有在讨论充分且确实满足 ADR 条件时才标记为 `Accepted`；其余使用 `Proposed` 或 working draft。
