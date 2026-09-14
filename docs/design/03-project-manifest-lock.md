# Project Manifest 与 Lock v0

状态：Accepted

对应 Wayfinder：#6 `Define Project Manifest and Lock semantics`

## 1. 三层职责

AKM 项目侧有三个不同职责的文件：

```text
akm.toml                   # 用户声明：我想要什么
akm.lock                   # 可提交：最终解析成什么
.akm/dependencies.lock     # 本机状态：当前环境是否满足依赖
```

固定边界：

- `akm.toml` 只声明顶层 Skill/Repository requirement；
- `akm.lock` 保存 requirement 的规范化语义、exact repository source、Package Snapshot 身份和 resolved dependency edges；
- `.akm/dependencies.lock` 保存当前机器上的软件/特殊依赖观察结果；
- Package 自身的 `akm-package.toml` / `DEPENDENCIES.md` 属于 immutable Package Snapshot，不复制到 Project Lock。

## 2. `akm.toml`

最小格式：

```toml
schema = 1

[skills]
"Akira-TL/matt-skills/ask-matt" = "^1.4"
"Akira-TL/skills/browser-access" = "^2.0"
"example/special-skills/special-skill" = { git = "main" }
```

字符串值表示 GitHub Release version requirement。coordinate 可以是 `owner/repo/package` 或 `owner/repo`；前者选择一个 Skill Package，后者表示 repository-wide top-level requirement。

Git source 使用 inline table，必须显式：

```toml
[skills]
"example/special-skills/special-skill" = { git = "main" }
"example/another-skills" = { git = "0123456789abcdef" }
```

不再使用 Git source 的 `"*"` version placeholder 和独立 `[sources]` table。

## 3. Repository-scoped source binding

Project Requirement 可以精确到一个 Package，但 source binding 始终作用于 `owner/repo`。

例如：

```toml
[skills]
"owner/repo/foo" = { git = "main" }
```

解析到：

```text
owner/repo -> exact commit abc123...
```

同 repository 的 `bar`、`baz` dependency 都复用同一个 exact commit。

同一项目若要求同 repository 的不同 Git ref，或 Release/Git 混装，返回：

```text
RepositorySourceConflict
```

v0 中一个 project resolution 的每个 repository 只允许一个 exact source snapshot。

## 4. `akm.lock` 三类 Record

Canonical Lock 只使用：

```text
requirement
repository
package
```

关系：

```text
requirement
    ↓ 用户顶层意图
repository
    ↓ exact source provenance
package
    ↓ exact Package Snapshot + resolved edges
```

### 4.1 Requirement Record

Release：

```toml
[[requirement]]
coordinate = "Akira-TL/matt-skills/ask-matt"
source-kind = "github-release"
version = "^1.4"
```

Git：

```toml
[[requirement]]
coordinate = "example/special-skills/special-skill"
source-kind = "git"
ref = "main"
```

Requirement Record 保存 `akm.toml` 解析后的语义，不保存原始 TOML bytes。因此只改注释、空白、表项顺序或等价排版不会让 Lock 失效。

### 4.2 Repository Record

Release：

```toml
[[repository]]
coordinate = "Akira-TL/matt-skills"
source-kind = "github-release"
version = "1.4.3"
tag = "v1.4.3"
commit = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
immutable = true
```

Git：

```toml
[[repository]]
coordinate = "example/special-skills"
source-kind = "git"
commit = "0123456789abcdef0123456789abcdef01234567"
```

Repository Record 是 source provenance 的唯一位置。Package Record 不重复 `source-kind`、version/tag/commit/requested-ref；Git `ref` 属于 top-level Requirement 语义，Repository Record 只保存最终 exact commit。

### 4.3 Package Record

```toml
[[package]]
coordinate = "Akira-TL/matt-skills/ask-matt"
package-root = "skills/engineering/ask-matt"
content-digest = "sha256:3333333333333333333333333333333333333333333333333333333333333333"
dependencies = [
  "Akira-TL/matt-skills/implement",
  "Akira-TL/matt-skills/wayfinder",
]
```

Package Record 只保存完整 Package coordinate、实际 repository-relative `package-root`、`AKM-PACKAGE-V1` `content-digest` 和已解析的 exact Skill dependency edges。

不再保存 `name`：它已经是 coordinate 最后一段，并且 discovery 已验证它来自 `SKILL.md.name`。

dependency edge 不带 `@version` / commit；目标 Package 所属 `[[repository]]` 已唯一决定 exact source snapshot。

## 5. Project Lock 不复制 Package 内部状态

`akm.lock` 不保存：

```text
manifest-digest
dependencies-doc-digest
[[software]]
```

`akm-package.toml` / `DEPENDENCIES.md` 已包含在 Package Snapshot 中，任意 byte 变化都会改变 Package `content-digest`。common software requirement 可从 immutable Package Snapshot 读取；当前机器 observation 属于 `.akm/dependencies.lock`。

因此 `content-digest` 是 Package payload 的唯一项目级完整性身份。

## 6. Canonical serialization

AKM 自己生成 `akm.lock`；v0 writer 固定：

1. UTF-8；
2. LF line endings；
3. 不生成注释；
4. `lock-version` 位于最前；
5. `[[requirement]]` 按 `coordinate` 的 UTF-8 bytes 升序；
6. `[[repository]]` 按 `coordinate` 的 UTF-8 bytes 升序；
7. `[[package]]` 按 `coordinate` 的 UTF-8 bytes 升序；
8. `dependencies` 按 coordinate 的 UTF-8 bytes 升序；
9. 同一种 Record 的字段使用固定顺序。

字段顺序：

```text
requirement:
  coordinate
  source-kind
  version | ref

repository / github-release:
  coordinate
  source-kind
  version
  tag
  commit
  immutable

repository / git:
  coordinate
  source-kind
  commit

package:
  coordinate
  package-root
  content-digest
  dependencies
```

Canonical ordering 用于稳定 Git diff 和 deterministic generation；Lock 语义仍由解析后的 TOML 数据决定。

## 7. `frozen`

`frozen` 先把当前 `akm.toml` 解析成 canonical Requirement Set，再与 Lock 中 `[[requirement]]` 比较。

语义不同返回：

```text
FrozenRequirementMismatch
```

语义相同则不重新选择版本/ref，只验证 Lock 中 exact repository source、Package Root、Package Content Digest 和 dependency graph。

因此 `frozen` 不因 `akm.toml` 注释或排版变化失败。

## 8. `sync` / `update`

普通 `sync`：

```text
parse akm.toml
  -> compare previous Requirement Set
  -> reuse still-valid previous repository resolution when possible
  -> resolve changed/new requirements
  -> build exact repository graph
  -> discover/resolve Packages
  -> materialize/verify Package Snapshots
  -> rebuild Project Skill Library
  -> atomically rewrite canonical akm.lock
```

Release retargeting仍按已有规则处理：previous Lock 中同一个 Release tag 的 exact commit 若与当前 GitHub 解析不同，普通 `sync` 返回 `ReleaseRetargeted`，不静默漂移。

显式 `update` 才允许主动重新选择满足 requirement 的较新 Release 或接受用户明确要求的新 source snapshot。

## 9. Project Skill Library

Lock 最终激活：

```text
<project>/.akm/skills/<owner>/<repo>/<package>
    -> <machine-store>/sha256/<content-digest-hex>
```

Package leaf 直接链接 immutable Package Store entry；AKM 不在项目内复制或修改 Package payload。

## 10. 不属于 `akm.lock` 的内容

以下内容明确不属于 Project Lock：

- Git Source Cache 机器绝对路径；
- Release source archive bytes/digest；
- Package 内文件逐项 digest；
- `akm-package.toml` 单独 digest；
- `DEPENDENCIES.md` 单独 digest；
- 当前宿主软件版本/路径/状态；
- Agent 对特殊依赖的检查 note；
- executor-specific activation/discovery view。

这些分别属于 source cache、Package Snapshot、`.akm/dependencies.lock` 或 executor adapter。
