# Package Manifest v0 Schema 候选协议

状态：Proposed

对应 Wayfinder：#4 `Define the Package Manifest schema and ownership boundaries`

> 当前公开文件名仍暂写作 `akm-package.toml`。产品/CLI/协议命名空间存在直接外部碰撞，已由 #14 单独决策；本文件先固定 Manifest 的结构语义，最终文件名随命名决策一次性迁移。

## 1. 定位

Package Manifest 是可选的结构化增强文件。Skill Package 的最低准入条件仍只有合法 `SKILL.md`。

若 Manifest 不存在：

- Package 仍然合法；
- AKM 不推断 Skill dependencies；
- AKM 不推断 common software requirements。

若 Manifest 存在，则必须完整通过当前 schema 校验。AKM 不允许“Manifest 写坏了但静默忽略”，因为这样可能漏装作者明确声明的依赖。

## 2. v0 完整结构

Schema 1 只定义三个顶层成员：

```toml
schema = 1

[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
"Akira-TL/matt-skills/wayfinder" = "^1.4"

[software]
git = ">=2.40"
gh = ">=2.45"
```

其中：

- `schema`：必填 integer，schema 1 固定为 `1`；
- `[dependencies]`：可选 table；
- `[software]`：可选 table。

`schema = 1` 单独存在也是合法 Manifest，只是没有实际增强语义。

Schema 1 不定义 `[package]` table，也不接受任何其他顶层 key/table。

## 3. Strict schema

当 `schema = 1` 时，未知字段或未知 table 都是错误：

```text
UnknownManifestField
```

例如以下内容在 v0 都不合法：

```toml
schema = 1
name = "foo"
version = "1.0.0"

[package]
description = "..."

[features]
...
```

原因不是这些概念永远不能存在，而是 schema 1 必须能发现拼写错误，并避免旧实现对未来字段进行部分解释后漏掉重要语义。

## 4. `schema` 演进

`schema` 使用正整数，不使用 `1.0` / `1.1` 形式。

规则：

- 缺失 `schema`：`MissingManifestSchema`；
- 非 integer：`InvalidManifestSchema`；
- 当前实现不支持的值：`UnsupportedManifestSchema`；
- 不对未知 schema 做 best-effort 部分解析；
- 新增会影响解析/依赖/运行语义的字段时升级 schema；
- 同一个 schema 内只允许不改变数据模型的 parser bugfix / 校验澄清。

未来实现可以同时支持多个已知 schema，但每个文件只声明一个 schema。

## 5. 不属于 Manifest 的字段

Schema 1 明确不保存：

```text
Package name
Package version
Skill description
Router / entrypoint flag
GitHub owner/repository
Release tag / commit
source kind / source URL
activation rename
project-local settings
platform/provider install commands
arbitrary scripts/hooks
license / author / homepage 等重复展示元数据
```

归属：

- Package name / description / Agent 行为 → `SKILL.md`；
- Release version / tag / commit / source provenance → source resolution + project Lock；
- activation rename → project Manifest；
- 特殊环境/硬件/服务/授权条件 → `DEPENDENCIES.md`；
- 当前宿主 observation → project-local `dependencies.lock`。

## 6. `[dependencies]`

每个 key 表示一个 Skill Package dependency：

```toml
[dependencies]
"owner/repo/package" = "<release-version-requirement>"
```

结构规则：

- key 必须恰好包含三个非空 `/` 分隔段：`<owner>/<repo>/<package>`；
- `<package>` 必须符合标准 Agent Skill `name` grammar；
- dependency 不允许省略 package，因此不能表达 repository-wide install；
- value 必须是非空 string；
- value 的具体 version requirement grammar 由 #7 Resolver 协议定义，Manifest schema 不复制一套 grammar；
- Manifest dependency 不提供 `{ git = ... }`、URL、path 或 source override 语法；
- 如果 project 已把同 repository 显式绑定到 Git source，则 Resolver 按已接受规则复用该 repository exact commit。

owner/repo 的 GitHub canonicalization、大小写与 source trust 规则属于 #9，不由 Manifest schema 自行重新定义。

## 7. `[software]`

每个 entry 表示一个内建 common software probe：

```toml
[software]
git = ">=2.40"
python = ">=3.11"
```

结构规则：

- key 必须匹配 `[a-z][a-z0-9-]*`；
- value 必须是非空 string；
- key 必须是当前实现支持的 canonical software probe ID，否则返回 `UnsupportedSoftwareProbe`；
- version requirement 的具体 grammar 与 common resolver grammar 由 #7 统一定义；
- Manifest 不允许声明 probe command、install command、package manager/provider、PATH 修改或其他环境写操作。

复杂或无法由内建只读 probe 表达的 requirement 放入 `DEPENDENCIES.md`，而不是扩展任意 executable command。

具体“哪些 software probe ID 被实现支持”属于实现 capability table，不进入 Package identity；schema 只固定 key/value 形状和 unknown-ID fail-fast 行为。

## 8. Optional / features

v0 不定义：

```text
optional dependencies
peer dependencies
feature flags
platform-conditioned dependencies
extras / groups
```

所有 `[dependencies]` edge 都是 required edge。若未来确有需求，再通过新 schema 明确加入，避免 schema 1 中出现多套隐式依赖语义。

## 9. TOML 与语义校验

Manifest 必须：

1. 是合法 TOML；
2. 顶层是 table；
3. 通过 `schema` 校验；
4. 只包含 schema 1 已知成员；
5. `[dependencies]` / `[software]` 若存在必须是 table；
6. 所有 table value 类型必须为 string；
7. dependency coordinate、Skill name、software probe ID 必须通过各自结构校验；
8. requirement string 必须由 #7 定义的 requirement parser 接受。

任何失败都会使“该 Manifest 增强后的 Package”对当前包管理器无法安装，而不是降级成忽略 Manifest 的裸 Skill。

## 10. Canonical semantics，不 canonicalize source bytes

TOML key 顺序、空白和注释不改变 Manifest 的解析语义；resolver 使用解析后的 map。

但 Manifest 原始文件 bytes 属于 immutable Package Snapshot，因此纯格式变化仍会改变 Package `content-digest`。snapshot 前不自动格式化、重排或重写作者 Manifest。

## 11. Schema 1 摘要

```text
Package Manifest optional
  └─ present -> schema required and strict
       ├─ [dependencies] owner/repo/package -> release requirement
       └─ [software]     canonical probe id -> version requirement

No identity
No package version
No source override
No install commands
No optional/features
No hidden unknown fields
```
