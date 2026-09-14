# AKM 领域词汇

## Skill Package

AKM 的最小安装、缓存与依赖解析单位。一个 Skill Package 恰好对应一个标准 Agent Skill；Package Root 与 Skill Root 重合，最低条件只有合法 `SKILL.md`。

## Package Name

GitHub repository 内的局部 Package 名称，例如 `ask-matt`。它以 `SKILL.md.name` 为唯一 source of truth，并应等于 Package Root basename；不承担全局唯一身份。

## GitHub Package Coordinate

AKM v0 的 GitHub 分发坐标，形式为 `<owner>/<repo>/<package>@<version-or-ref>`。`package` 可在用户顶层安装目标中省略以表示安装该 source snapshot 中全部发现的 Skill Package。

## GitHub Release Version

Release source 中的版本作用域。`@version` 先选择 repository Release/tag，再在对应 exact source snapshot 中 discovery 一个或多个 Skill Package。

## Git Source

没有合适 Release 或明确需要源码版本时显式使用的 source。用户 ref 最终解析为 exact commit；Package path 在该 commit 的 tracked tree 中通过 `SKILL.md` discovery 获得。

## Git Source Cache

机器级、可丢弃的 Git repository object cache，用于复用 clone/fetch 成本和读取 exact commit。项目不直接链接它；删除 cache 不破坏已经写入 Package Store 的 Package snapshot。

## Repository Discovery Control

Repository root 中可选的 `akm-repo.toml`。它只通过 repository-relative Package Root `include` / `exclude` 过滤 `SKILL.md` discovery 结果；没有该文件时默认全仓发现。`exclude` 优先于 `include`；v0 pattern 只支持 literal、`*`、`**`、`?`。它不定义 Package name、version 或 dependency，不能把没有合法 `SKILL.md` 的目录变成 Package。不同名称的 nested Skill Roots 可以同时被发现；只有最终 Package Name 重复才构成歧义。

## Router Skill

负责指导 Agent 在一组能力之间路由的普通 Skill Package。AKM 不为 Router 定义特殊 Package 类型；若存在可选 Manifest，其 Skill dependencies 可以形成 Router 的自动安装闭包。

## Release Artifact

GitHub Release 对应的稳定 source snapshot，或作者额外提供的 AKM package-specific Asset。AKM 专用 Asset 是优化而非准入条件。

## Package Manifest

Package Root 中可选的 `akm-package.toml`。它声明结构化 Skill dependencies 与 AKM 能基础探测的常见软件 requirements；不复制 `SKILL.md.name` 或 GitHub Release version。

## Dependency Check File

Package Root 中可选、不可变的 `DEPENDENCIES.md`。它是 Package author 提供给 Agent 的特殊软件/环境依赖检查说明，只描述 requirement、检查方法与处理边界；不保存当前宿主状态。

## Dependency State Lock

项目本地 `.akm/dependencies.lock`。它保存 common software probe 与 Agent 对特殊依赖的当前观察结果，并关联 Package content/`DEPENDENCIES.md` digest；属于可重建本机状态，默认不提交版本控制。

## Skill Dependency

可选 Package Manifest 中显式声明的对另一个 Skill Package 的依赖。GitHub 模式使用 `<owner>/<repo>/<package>` 定位目标并附加 Release version range；运行时共享能力必须通过 Skill Dependency 表达，而不是跨 Package 文件共享。

## Project Requirement

项目主动声明的顶层 GitHub install target，可以是 `<owner>/<repo>/<package>` 或 `<owner>/<repo>`；前者选择一个 Skill，后者选择 source snapshot 中全部发现的 Skill Package。

## Resolved Graph

AKM 根据 Project Requirement、exact repository source snapshots、`SKILL.md` discovery 与可选 Manifest 求出的完整已知 dependency graph。没有 Manifest 的 Skill 是合法 leaf node。

## Lock Record

一个已解析 Package 的可重建记录，保存 exact repository source、actual package-root、`SKILL.md.name`、content integrity 和可选 manifest-declared dependency edges。

## Package Store

机器级共享的不可变 Skill Package snapshot 存储。多个项目可以复用同一 content；宿主环境检查状态不写回 Store。

## Project Skill Library

项目侧按 `<owner>/<repo>/<package>` 分层组织的 Skill 库，例如 `.akm/skills/Akira-TL/matt-skills/ask-matt`。leaf 链接到 Package Store；最终执行器发现由 executor adapter 或执行器自身负责。

## Common Software Requirement

可选 `akm-package.toml [software]` 中声明的、AKM 内建只读 probe 能基础发现的常见软件要求。AKM 只检查并记录状态，不负责安装、升级或修复。

## Special Dependency

可选 `DEPENDENCIES.md` 中描述的复杂软件、硬件、服务、数据、驱动、授权或其他环境要求。由 Agent 检查；需要修改环境时必须先向用户说明并取得明确批准。

## Install Plan

在安装/同步前形成的 source 获取、Skill discovery、Package snapshot、Store/Project Skill Library 变化及依赖检查计划。AKM 不把缺失宿主软件自动转换为系统安装动作。