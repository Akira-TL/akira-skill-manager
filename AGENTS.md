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
- Git repository、Package、Skill、Release、Project Environment 等概念必须分别建模；它们之间的基数关系仍需通过设计讨论确定。
- `SKILL.md` 应尽量保持对 Agent Skills 既有标准的兼容；AKM 扩展元数据放在哪里、Package 是否允许一个或多个 Skill Entry，当前仍是开放设计问题。
- 旧 `/home/Akira/Projects/akira-skills` 仅作为迁移与既有行为参考，不在其中实现 AKM。

## 工程约定

- 项目显示名称统一写作 `Akira sKill Manager`，缩写为 `AKM`。
- Python 工程遵循全局 `uv` 与 `.venv` 规则；当前协议未稳定前不要为了占位提前建立 CLI 或运行时骨架。
- 仓库存在 `.codegraph/` 后，理解或定位代码时优先使用 CodeGraph；索引数据库是机器本地状态，不进入 Git。
- 协议设计中的新决定只有在讨论充分且确实满足 ADR 条件时才标记为 `Accepted`；其余使用 `Proposed` 或 working draft。
