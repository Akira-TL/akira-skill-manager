# AKM 领域词汇

## Skill Package

AKM 的最小安装、版本校验与依赖解析单位。一个 Skill Package 恰好对应一个标准 Agent Skill；Package Root 与 Skill Root 重合。

## Package Name

GitHub repository 内的局部 Package 名称，例如 `ask-matt`。它同时等于 Package Root basename、`SKILL.md.name` 与 `akm-package.toml` 中的 `package.name`，但不承担全局唯一身份。

## GitHub Package Coordinate

AKM v0 的 GitHub 分发坐标，形式为 `<owner>/<repo>/<package>@<version>`。`package` 可在用户顶层安装目标中省略以表示安装该 Release 中全部 Package；Skill dependency 必须包含 package。

## GitHub Release Version

GitHub 分发模式中的版本作用域。`@version` 先选择 repository 的 GitHub Release，再在该 Release 中选择一个或多个 Package Artifact；同一 Release 中的原生 AKM Package 使用该 Release version。

## Router Skill

负责指导 Agent 在一组能力之间路由的普通 Skill Package。AKM 不为 Router 定义特殊 Package 类型；一个 Skill Suite 由 Router Package 加其 transitive Skill dependency closure 形成。

## Release Artifact

GitHub Release 中一个 Skill Package 的不可变归档。一个 Release 可以包含多个独立 Package Artifact，每个 Artifact 只包含一个 Skill Package。

## Package Manifest

Package Root 中的 `akm-package.toml`。它声明局部 Package name、version、Skill dependencies 与 AKM 能基础探测的常见软件 requirement，不复制 `SKILL.md` 的 Agent 行为信息。

## Dependency Check File

Package Root 中的不可变 `DEPENDENCIES.md`。它是 Package author 提供给 Agent 的软件/环境依赖检查说明，只描述特殊依赖、检查方法与处理边界；它不是可执行安装脚本，也不保存当前宿主状态。

## Dependency State Lock

项目本地 `.akm/dependencies.lock`。它保存 common software probe 与 Agent 对特殊依赖的当前观察结果，并关联 Package content/`DEPENDENCIES.md` digest；属于可重建的本机状态，默认不提交版本控制。

## Skill Dependency

一个 Skill Package 对另一个 Skill Package 的显式依赖。GitHub 模式使用 `<owner>/<repo>/<package>` 定位目标并附加版本范围；运行时共享能力必须通过 Skill Dependency 表达，而不是跨 Package 文件共享。

## Project Requirement

项目直接声明的顶层 GitHub install target，可以是 `<owner>/<repo>/<package>` 或 `<owner>/<repo>`；前者安装一个 Package 及其依赖闭包，后者安装该 Release 中全部 Package。

## Resolved Graph

AKM 根据 Project Requirement、GitHub Release/Git source 与 Package Manifest 求出的完整 Skill dependency graph。每个节点具有精确 source、release/commit 与 Package name。

## Lock Record

Resolved Graph 中一个 Package 的可重建记录，保存精确 GitHub repository、Release version 或 Git commit、Artifact/content integrity 与 dependency edges。

## Package Store

机器级共享的不可变 Package 内容存储。多个项目可以复用同一 Package payload；宿主环境检查状态不写回共享 Store。

## Project Skill Library

项目侧按 `<owner>/<repo>/<package>` 分层组织的 Skill 库，例如 `.akm/skills/Akira-TL/matt-skills/ask-matt/`。AKM 不把不同来源的同名 Skill 扁平化；最终 Skill discovery 由 executor adapter 或执行器自身负责。

## Common Software Requirement

`akm-package.toml` 中 `[software]` 声明的、AKM 内建只读 probe 能做基础发现的常见软件要求。AKM 只检查并记录状态，不负责安装、升级或修复。

## Special Dependency

无法由 AKM 常见 probe 可靠处理的软件、硬件、服务、数据、驱动、授权或其他环境要求。它写在 `DEPENDENCIES.md` 中，由 Agent 检查；若需要修改环境，Agent 必须先向用户说明并获得明确批准。

## Install Plan

在项目安装/同步前形成的 Package 获取、依赖解析、Store/Project Skill Library 变化及依赖检查结果。AKM 不把缺失宿主软件自动转换为系统安装动作。
