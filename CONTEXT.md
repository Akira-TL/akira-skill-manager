# AKM 领域词汇

## Skill Package

AKM 的最小安装、缓存与依赖解析单位。一个 Skill Package 恰好对应一个标准 Agent Skill；Package Root 与 Skill Root 重合，最低条件只有合法 `SKILL.md`。

## Package Name

GitHub repository 内的局部 Package 名称，例如 `ask-matt`。它以 `SKILL.md.name` 为唯一 source of truth，并应等于 Package Root basename；不承担全局唯一身份。

## GitHub Package Coordinate

Skiloom v0 的 GitHub Package Coordinate 是 `<owner>/<repo>/<package>`；用户顶层 target 可以省略 `package` 表示 repository-wide requirement，并另外附加 Release requirement 或 explicit Git ref。GitHub `owner/repo` 在 Core semantic comparison、resolver grouping、dependency edge 与 canonical Lock 中统一使用 ASCII lowercase；case-only 差异不产生新 source identity。

## GitHub Release Version

Release source 中的 repository 级版本作用域。v0 只接受 SemVer；GitHub tag 可以使用可选前导 `v`，例如 `v1.4.0` 规范化为 `1.4.0`。Release version 最终锁定实际 tag 与 exact commit，再在该 repository snapshot 中 discovery Skill Package。

## Release Version Requirement

Class R 对 GitHub Release repository version 的约束语言。v0 使用 Cargo-style default/caret/tilde/wildcard/comparison/comma-intersection semantics；不支持 npm `||` / hyphen range / whitespace-as-AND。多个同 repository requirement 共同约束同一个 exact Release snapshot；explicit Git binding 时同 repository Release range 不参与 commit selection。

## Git Source

没有合适 Release 或明确需要源码版本时显式使用的 source。用户 ref 最终解析为 exact commit；Package path 在该 commit 的 tracked tree 中通过 `SKILL.md` discovery 获得。

## Git Source Cache

机器级、可丢弃的 Git repository object cache，用于复用 clone/fetch 成本和读取 exact commit。项目不直接链接它；删除 cache 不破坏已经写入 Package Store 的 Package snapshot。

## Repository Discovery Control

Repository root 中可选的 `akm-repo.toml`。它只通过 repository-relative Package Root `include` / `exclude` 过滤 `SKILL.md` discovery 结果；没有该文件时默认全仓发现。`exclude` 优先于 `include`；v0 pattern 只支持 literal、`*`、`**`、`?`。它不定义 Package name、version 或 dependency，不能把没有合法 `SKILL.md` 的目录变成 Package。不同名称的 nested Skill Roots 可以同时被发现；只有最终 Package Name 重复才构成歧义。

## Router Skill

负责指导 Agent 在一组能力之间路由的普通 Skill Package。AKM 不为 Router 定义特殊 Package 类型；若存在可选 Manifest，其 Skill dependencies 可以形成 Router 的自动安装闭包。

## Release Source Snapshot

GitHub SemVer Release 解析得到的 actual tag + exact commit 对应 repository snapshot。AKM v0 不定义 per-Skill Release Asset；Release 只负责稳定选择 source snapshot，Package discovery 与 Git source 共用同一管线。

## Package Manifest

Package Root 中可选的 `akm-package.toml`。Schema 1 的 `[dependencies]` 属于 Skiloom Core Skill dependency graph；`[software]` 是同一物理 Manifest 中已登记的 Host Observation Extension attachment。Manifest 不复制 `SKILL.md.name` 或 GitHub Release version；不支持 Host Observation Extension 不会使 otherwise-valid Package 失去 Core conformance。

## Dependency Check File

Package Root 中可选、不可变的 `DEPENDENCIES.md`。它是 Package author 提供给 Agent 的特殊软件/环境依赖检查说明，只描述 requirement、检查方法与处理边界；不保存当前宿主状态。

## Dependency State Lock

项目本地 `.agents/.akm/dependencies.lock`。它只保存当前机器的 dependency observations；Package `content-digest` 是唯一 freshness anchor，不复制 requirement、`DEPENDENCIES.md` digest、检查时间或授权信息。AKM core 维护 common software observations，Agent 维护 special observations；属于可删除重建的本机状态，默认不提交版本控制。

## Skill Dependency

可选 Package Manifest 中显式声明的对另一个 Skill Package 的依赖。GitHub 模式使用 `<owner>/<repo>/<package>` 定位目标并附加 Release version range；运行时共享能力必须通过 Skill Dependency 表达，而不是跨 Package 文件共享。

## Project Requirement

项目主动声明的顶层 GitHub install target，可以是 `<owner>/<repo>/<package>` 或 `<owner>/<repo>`；前者选择一个 Skill，后者选择 source snapshot 中全部发现的 Skill Package。

## Project Intent

项目在 `.agents/.akm/akm.toml [skills]` 中声明的允许范围与 source intent。它描述“项目允许什么”，不等于当前实际安装的 exact version/commit。

## Confirmed Resolution

项目已经明确接受并写入 `.agents/.akm/akm.lock` 的 exact repository snapshots、Package identities 与 dependency graph。普通 `sync` 只恢复已有 Confirmed Resolution；只有 initial resolution 或显式 update 在接受后才能替换它。

## Candidate Repository Set

一次 initial resolution 或显式 re-resolution 产生的完整 repository source binding 集合。Transitive dependency 可以把新的 GitHub repository 提名进 candidate graph，但只有完整 Candidate Repository Set 被 acceptance decision 覆盖后，它才可以进入 Confirmed Resolution；普通 replay 不扩张该集合。

## Source Authorization Delta

Candidate Repository Set 相对当前 Confirmed Resolution 的 repository/source 变化事实，包括 repository 新增/移除、source-kind 变化与 exact Release/Git binding 变化。Non-interactive acceptance policy 必须基于完整 previous/candidate sets 与该 delta 做整份 candidate 的 accept/reject，而不是把 resolver 成功本身当授权。

## Resolved Graph

AKM 根据 Project Requirement、exact repository source snapshots、`SKILL.md` discovery 与可选 Manifest 求出的完整已知 dependency graph。没有 Manifest 的 Skill 是合法 leaf node；Skill dependency cycle 本身合法，只要 repository source/version constraints 可以形成完整 deterministic resolution。

## Package Snapshot

从一个已选 Skill Root materialize 出来的不可变 Package 内容边界。若其中包含另一个已经进入最终 discovery set 的 nested Skill Root，则该 nested Root 从祖先 Package Snapshot 中裁掉并独立 snapshot；被 discovery filter 排除的 nested `SKILL.md` 仍作为普通内容保留。

## Package Content Digest

Package Snapshot 的 canonical 内容身份。v0 使用 `AKM-PACKAGE-V1`：只允许 regular files，按 relative path 的原始 UTF-8 bytes 排序，只编码 path、executable bool、file size 与 exact bytes 的 SHA-256；时间戳、owner/group、普通权限和 archive metadata 不参与。最终表达为 `sha256:<64 lowercase hex>`。

## Project Requirement Record

`.agents/.akm/akm.lock` 中对 `.agents/.akm/akm.toml [skills]` 顶层 requirement 的规范化语义记录。Release requirement 保存 coordinate + version requirement；Git requirement 保存 coordinate + requested ref。`frozen` 比较 Requirement Set 语义而不是 `akm.toml` 原始 bytes。

## Repository Lock Record

`.agents/.akm/akm.lock` 中 source provenance 的唯一记录。Repository coordinate 使用 canonical lowercase owner/repo；Release source 保存规范化 SemVer、actual tag、exact commit 与 immutable signal，Git source 保存 exact commit。Package Record 不重复这些字段。GitHub 后续 rename/transfer 不自动改写既有 Lock provenance。

## Package Lock Record

`.agents/.akm/akm.lock` 中一个已解析 Package 的精简记录，只保存完整 coordinate、actual package-root、Package Content Digest 与 exact manifest-declared dependency edges；依赖边只写 `owner/repo/package`，其 exact source 由对应 Repository Lock Record 唯一决定。

## Package Store

机器级共享的不可变 Skill Package Snapshot 存储，直接以 Package Content Digest 作为 key。不同 repository/source 只要 snapshot 内容完全相同就复用同一 Store entry；source provenance 保存在 Lock，不进入 Store key。宿主环境检查状态不写回 Store。

## Project Skill Activation

项目中 executor-visible 的 Skill 直接扁平位于 `.agents/skills/<activation-name>`。默认 activation name 等于 `SKILL.md.name`；不同 source 的同名 Skill 在这里形成真实冲突，AKM 必须提示用户为新安装项 rename 或放弃，不能自动覆盖。

## Activation Rename

项目本地对 resolved Package 的 runtime Skill identity 改名。用户批准后写入 `.agents/.akm/akm.toml [renames]`。Rename 不改变 Package coordinate、source resolution 或 Package Store `content-digest`；AKM 在 `.agents/skills/<new-name>` materialize 合法 activation view，并同步使顶层 `SKILL.md.name == <new-name>`。

## Activation State Lock

项目本地 `.agents/.akm/activation.lock`。每个 `[[skill]]` 只保存 `activation-name`、Package coordinate、原始 Package Content Digest 与本机 materialization mode（`symlink` / `junction` / `copy`）。POSIX 未 rename 优先 symlink，Windows 未 rename 优先 junction，rename 一律 copy。它用于安全 update/remove/doctor；missing activation 可自动重建，modified/replaced activation 默认 fail closed；属于可重建本机状态，默认不提交版本控制。

## Package Store GC Boundary

AKM v0 不执行 destructive automatic Package Store GC。项目 remove 只移除 activation/state，不删除共享 Store entry；Git Source Cache 可以独立做 LRU/size/age pruning。未来只有引入 machine-wide project/reference registry、能够证明 digest 没有任何活跃项目引用后，才重新定义 destructive Store GC。

## Common Software Requirement

可选 `akm-package.toml [software]` 中声明的、AKM 内建只读 probe 能基础发现的常见软件要求。AKM 只检查并记录状态，不负责安装、升级或修复。

## Special Dependency

可选 `DEPENDENCIES.md` 中描述的复杂软件、硬件、服务、数据、驱动、授权或其他环境要求。由 Agent 检查；需要修改环境时必须先向用户说明并取得明确批准。

## Install Plan

在安装/同步前形成的 source 获取、Skill discovery、Package snapshot、Store 变化、`.agents/skills/` 扁平 activation preflight/rename 以及依赖检查计划。AKM 不把缺失宿主软件自动转换为系统安装动作。

## Skiloom Public Namespace

Skiloom v0 的公开 token 是 `skiloom`：CLI 为 `skiloom`，project state 位于 `.agents/.skiloom/`，Project Intent/Lock 为 `skiloom.toml` / `skiloom.lock`，Package/Repository optional metadata 为 `skiloom-package.toml` / `skiloom-repo.toml`，公开 Package Snapshot format identifier 为 `SKILOOM-PACKAGE-V1`。旧 `AKM / akm` 只属于 pre-standard working draft，不形成 v0 compatibility alias。

## Skiloom Core

Skiloom 面向独立实现的最小互操作协议面，只标准化会改变 Package discovery、dependency graph、source resolution、Confirmed Resolution、Package Content Digest 或 activation ownership 的可观察语义；CLI UX、缓存/Store 物理布局、平台 materialization 优化与宿主探测实现不因 reference manager 采用而自动成为 Core。

## Core Conformance Class

Skiloom Core 的可测试能力声明。Class P 负责 deterministic Package model，Class R 在 P 之上负责 source/resolution/Lock，Class A 在 R 之上负责 project activation safety；只有同一 Core version 同时通过 P、R、A 的实现才是 Full Core Manager。Host Observation 是独立 extension，不属于 Full Core conformance。
