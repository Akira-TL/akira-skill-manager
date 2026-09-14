# Project Manifest 与 Lock v0 工作草案

## 1. 两个文件、两个职责

项目协议文件：

```text
akm.toml
akm.lock
```

- `akm.toml`：用户/Agent 声明顶层安装目标；
- `akm.lock`：AKM 生成，锁定 exact source snapshot、实际发现的 Package Root、content digest 与结构化 dependency graph。

本机依赖检查状态另放：

```text
.akm/dependencies.lock
```

## 2. Project Manifest

Release source 示例：

```toml
schema = 1

[skills]
"Akira-TL/matt-skills/ask-matt" = "^1.4"
"Akira-TL/skills/browser-access" = "^2.0"
```

Repository-wide：

```toml
[skills]
"Akira-TL/matt-skills" = "1.4.0"
```

Git source 必须显式：

```toml
[skills]
"example/special-skills/special-skill" = "*"

[sources."example/special-skills/special-skill"]
kind = "git"
ref = "main"
```

Git 模式下 `ref` 决定 source；`[skills]` 中的 version range 不参与 Git commit 选择。实现时可以进一步收敛 Git target 的 manifest 语法，避免 `"*"` 这种占位表达。

## 3. Package 不要求 Manifest

Lock 中每个 Package 都至少来自一个合法 `SKILL.md`。

如果 Package 没有 `akm-package.toml`：

- 仍可安装；
- `dependencies = []`；
- 没有结构化 `[software]` requirements；
- 如果有 `DEPENDENCIES.md`，仍记录其 digest 供 Agent dependency checking 使用。

如果 Manifest 存在，则 Lock 固化解析出的 dependency edges 与 software requirements。

## 4. Release Lock Record

示意：

```toml
lock-version = 1
manifest-digest = "sha256:..."

[[repository]]
coordinate = "Akira-TL/matt-skills"
source-kind = "github-release"
version = "1.4.3"
tag = "v1.4.3"
commit = "abcdef0123456789..."
immutable = true

[[package]]
coordinate = "Akira-TL/matt-skills/ask-matt"
name = "ask-matt"
source-kind = "github-release"
version = "1.4.3"
package-root = "skills/engineering/ask-matt"
content-digest = "sha256:..."
manifest-digest = "sha256:..."
dependencies-doc-digest = "sha256:..."
dependencies = [
  "Akira-TL/matt-skills/implement@1.4.3",
  "Akira-TL/matt-skills/wayfinder@1.4.3",
]
```

没有可选文件时对应 digest 字段省略。

## 5. Git Lock Record

```toml
[[repository]]
coordinate = "example/special-skills"
source-kind = "git"
requested-ref = "main"
commit = "0123456789abcdef0123456789abcdef01234567"

[[package]]
coordinate = "example/special-skills/special-skill"
name = "special-skill"
source-kind = "git"
commit = "0123456789abcdef0123456789abcdef01234567"
package-root = "weird/path/special-skill"
content-digest = "sha256:..."
dependencies = []
```

Lock **不保存机器 Git cache 的绝对路径**。Cache 是 disposable 本机实现细节；真正可重建的是 repository + exact commit + package-root + content digest。

## 6. Repository snapshot 级一致性

同一个 project resolution 内：

```text
owner/repo
```

只能绑定一个 exact source snapshot：

- 一个 GitHub Release 对应的 exact commit；或
- 一个 Git exact commit。

来自同一 repository 的多个 Package Lock Record 必须引用同一个 repository snapshot。

## 7. Project Skill Library

```text
<project>/
├── akm.toml
├── akm.lock
└── .akm/
    ├── dependencies.lock
    └── skills/
        ├── Akira-TL/
        │   └── matt-skills/
        │       ├── ask-matt -> <machine-store>/<digest>
        │       └── tdd      -> <machine-store>/<digest>
        └── other/
            └── repo/
                └── ask-matt -> <machine-store>/<digest>
```

Package leaf 直接链接完整 immutable Skill Root。AKM 不修改 Package 文件。

## 8. `akm.lock` 与 `.akm/dependencies.lock`

### `akm.lock`

可提交、可移植，保存：

- exact repository source snapshot；
- Release source 的规范化 SemVer、实际 tag、exact commit 与 immutable signal；
- actual package-root；
- `SKILL.md.name`；
- content digest；
- optional Manifest/Dependency Check File digest；
- exact manifest-declared Skill dependency edges；
- optional structured common software requirements。

### `.akm/dependencies.lock`

默认不提交，保存当前机器检查状态：

- common software probe observations；
- Agent 对 `DEPENDENCIES.md` 中特殊条件的检查结果；
- 对应 Package/content/dependency-doc digest，用于失效判断。

## 9. `sync`

```text
read akm.toml + previous lock
  -> keep still-valid exact repository snapshots when possible
  -> resolve SemVer Release -> actual tag -> exact commit，或解析 explicit Git source ref -> exact commit
  -> discover SKILL.md Package Roots
  -> read optional manifests
  -> resolve dependency closure
  -> snapshot missing Package Roots into Store
  -> rebuild Project Skill Library
  -> refresh common dependency observations
  -> invalidate stale special-dependency observations
  -> write locks atomically
```

## 10. remove / orphan

删除一个顶层 target 后重新计算当前可见的 manifest dependency closure。

不再可达的 Package 从 Project Skill Library 移除；machine Store 内容由单独 GC 策略处理。Git source cache 也独立 GC，因为项目不直接引用 cache。

Reverse Dependency 直接由 `akm.lock` 中 dependency edges 推导。

## 11. Release retargeting

如果 previous Lock 已记录某个 Release tag 对应 exact commit，而当前 GitHub 上同一 tag 已指向不同 commit，普通 `sync` 返回 `ReleaseRetargeted`，不得自动重写 Lock。

`frozen` 只使用 Lock 中的 exact commit；显式 update/重新解析才允许接受新的 source snapshot。