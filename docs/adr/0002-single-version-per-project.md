# ADR 0002：扁平 Skill name 冲突与 Package Identity 单版本模型

- 状态：Rejected
- 日期：2026-09-14

## 原提案

原提案假设 Project Skill View 是扁平的：

```text
skills/<skill-name>
```

因此不同来源只要导出同名 `SKILL.md.name` 就会冲突，并进一步提出“一个 Package Identity 在项目中只能有一个版本”的全局 Package Identity 模型。

## 拒绝原因

AKM v0 已改为按 GitHub 安装坐标分层保存项目 Skill Library：

```text
skills/<owner>/<repo>/<package>/
```

例如：

```text
skills/Akira-TL/matt-skills/ask-matt/
skills/someone/other-repo/ask-matt/
```

因此同名 Skill 不在 AKM core library 层构成冲突。最终执行器是否能同时发现、如何映射这些 Skill Root，由 executor adapter/执行器本身处理。

同时 v0 不建立独立 Registry Package Identity；版本首先属于 GitHub repository Release。来自同一个 repository 的依赖约束会共同选择一个满足条件的 Release version，再从该 Release 中选 Package Artifact。

## 当前方向

- 不维护全局扁平 Skill name 唯一性约束；
- 不以 `namespace/package` 作为 v0 Package Identity；
- GitHub resolver 以 repository Release version + package selector 为核心；
- 将来如果增加 Registry source model，再单独设计 Registry package identity 与多版本规则。
