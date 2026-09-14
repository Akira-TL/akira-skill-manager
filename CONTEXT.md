# AKM 领域词汇

## Package

AKM 的版本化发行与依赖解析单位，不等同于承载源码的 Git repository。一个 Package 暴露一个还是多个 Skill Entry 当前仍是开放设计问题。

## Package Identity

跨版本稳定的包身份。由命名空间与包名组成，例如 `akira/research`。它与 GitHub owner/repository、Release tag、下载 URL 相互独立。

## Skill Entry

Package 对 Agent 暴露的一个标准 Agent Skill。Skill Entry 以 `SKILL.md` 为核心；它在 Package 内的目录位置以及一个 Package 可包含多少个 Skill Entry 当前仍待设计。

## Release

某个 Package Identity 的一个版本化、不可变发行快照，并指向一个可校验的 Release Artifact。具体版本约束语法仍由协议设计决定。

## Release Artifact

Release 的可下载、不可变归档。它包含 Package Manifest 与完整 Skill 内容，并具有内容完整性摘要。

## Package Manifest

随 Package 源码与 Release Artifact 一起存在的结构化元数据。它声明 Package Identity、版本、Skill Entry 信息、Skill 依赖、软件依赖与兼容性，但不自行决定自己是否可信。

## Package Index

向解析器提供 Package 可用版本、Package Manifest 与 Release Artifact 定位信息的受信元数据来源。它可以由 registry、GitHub Release 索引或其他适配器实现。

## Source Locator

指向某个精确 Package 内容的来源描述，例如 Release Artifact、Git commit 或本地 path。Source Locator 描述“从哪里取”，不等于 Package Identity。

## Project Requirement

项目直接声明的顶层 Package 需求。它表达 Package Identity 与允许版本范围，必要时可附带项目显式批准的 source override。

## Resolved Graph

Dependency Resolver 根据 Project Requirement、可用版本与锁定状态求出的完整 Package 依赖图。图中的每个 Package 都具有精确版本和精确来源。

## Lock Record

Resolved Graph 中一个已解析 Package 的可重建记录。它保存精确版本、来源、完整性摘要及依赖边，不承担用户手写需求的职责。

## Package Store

机器级共享的不可变 Package 内容存储。多个项目可以复用同一内容，不通过复制形成各自副本。

## Project Skill View

某个项目实际启用的 Skill 名称集合。每个条目由 AKM 管理的软链接指向 Package Store 中的精确 Package 内容。

## Software Requirement

Package 对宿主环境中的外部软件或运行时能力提出的结构化要求，例如 Git、GitHub CLI、samtools、Python、R 或 Node.js 的版本范围。

## Software Catalog

AKM 维护的“软件能力身份 → 探测方法 → 平台 Provider 映射”。Package 只声明需要什么能力，不嵌入任意系统安装命令。

## Provider

在特定平台上检查或安装某个 Software Requirement 的适配器。Provider 的安装动作只有在用户批准 Install Plan 后才允许执行。

## Install Plan

在任何持久化修改发生前生成的完整执行计划。它同时描述 Package 下载、校验、Package Store 变更、Project Skill View 变更、信任决策和缺失软件依赖。

## Trust Boundary

决定某个 source、Package Index 或软件 Provider 是否允许进入解析和执行流程的边界。信任不能由 Package 自身元数据单方面声明。

## Reverse Dependency

在 Resolved Graph 中直接依赖某 Package 的其他 Package。

## Orphan Package

曾因依赖关系进入项目 Resolved Graph，但在当前顶层 Project Requirement 的依赖闭包中已经不可达的 Package。
