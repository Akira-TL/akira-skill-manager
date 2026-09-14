# Project Manifest 与 Lock v0 工作草案

## 1. 目标

Project Manifest 记录项目主动安装的 GitHub Skill targets；Project Lock 记录解析后的精确 GitHub Release/Git commit、Package 集合与依赖图。

当前 v0 不依赖独立 Package Registry。

## 2. 文件名

```text
akm.toml
akm.lock
```

`akm.toml` 由用户/Agent 编辑；`akm.lock` 由 AKM 生成。

## 3. Project Manifest

指定单个 Package：

```toml
schema = 1

[skills]
"Akira-TL/matt-skills/ask-matt" = "^1.4"
"Akira-TL/skills/browser-access" = "^2.0"
```

允许显式安装某个 repository 的全部 Package：

```toml
[skills]
"Akira-TL/matt-skills" = "1.4.0"
```

两种 key 与 CLI 安装目标一致：

```text
owner/repo/package@version
owner/repo@version
```

区别：

- `owner/repo/package`：顶层 requirement 是一个 Package；
- `owner/repo`：顶层 requirement 是该 Release 中发现的全部 Package；
- transitive `[dependencies]` 必须始终精确到 `owner/repo/package`，不能依赖整个 repository。

## 4. Git 模式必须显式记录

默认 requirement 解析 GitHub Release。

开发或没有 Release 时，项目可以显式允许 Git source：

```toml
[sources."Akira-TL/matt-skills/ask-matt"]
kind = "git"
ref = "main"
```

或对 repository-wide target：

```toml
[sources."Akira-TL/matt-skills"]
kind = "git"
ref = "9b2c7f..."
```

Git source 规则：

- 必须是显式配置/显式 CLI 参数；
- 不因 Release 不存在自动 fallback；
- Lock 必须固定 exact commit；
- clone 后机械扫描 `SKILL.md + akm-package.toml` Package Roots；
- package selector 存在时只选择对应 Package；
- selector 省略时选择全部发现的 Package。

## 5. GitHub Release version 的作用域

当前 GitHub 模式中，`@version` 指 repository 的 GitHub Release version。

例如：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

含义是：

1. 选择 `Akira-TL/matt-skills` Release `1.4.0`；
2. 在该 Release 中选择 `ask-matt` Package Artifact；
3. 校验其中 `package.name = "ask-matt"`；
4. 校验 `package.version = "1.4.0"`。

因此同一个 repository Release 中的原生 AKM Package 使用同一 Release version。

如果多个 dependency 都来自同一 repository：

```text
Akira-TL/matt-skills/implement ^1.4
Akira-TL/matt-skills/tdd       >=1.4 <2
```

resolver 实际选择一个满足这些约束的 `matt-skills` Release version，再从该 Release 取所需 Package Artifacts。

未来 Registry 模式可以有独立 package version；那是另一种 source model，不改变 GitHub v0 的简单规则。

## 6. Lock Record

示意：

```toml
lock-version = 1

[[github-release]]
repository = "Akira-TL/matt-skills"
version = "1.4.3"
release-url = "https://github.com/Akira-TL/matt-skills/releases/tag/1.4.3"

[[package]]
coordinate = "Akira-TL/matt-skills/ask-matt"
name = "ask-matt"
version = "1.4.3"
source-kind = "github-release"
artifact = "ask-matt.akm.tar.gz"
integrity = "sha256:..."
dependencies = [
  "Akira-TL/matt-skills/implement@1.4.3",
  "Akira-TL/matt-skills/wayfinder@1.4.3",
]

[[package]]
coordinate = "Akira-TL/matt-skills/implement"
name = "implement"
version = "1.4.3"
source-kind = "github-release"
artifact = "implement.akm.tar.gz"
integrity = "sha256:..."
dependencies = [
  "Akira-TL/matt-skills/tdd@1.4.3",
]
```

Git source：

```toml
[[package]]
coordinate = "example/tools/foo"
name = "foo"
version = "0.4.0"
source-kind = "git"
repository = "https://github.com/example/tools.git"
commit = "0123456789abcdef..."
package-root = "skills/foo"
content-digest = "sha256:..."
```

## 7. 项目 Skill Library

Project Lock 的 Package 不扁平激活。

逻辑布局：

```text
<project>/
├── akm.toml
├── akm.lock
└── .akm/
    └── skills/
        ├── Akira-TL/
        │   └── matt-skills/
        │       ├── ask-matt/
        │       ├── implement/
        │       └── tdd/
        └── someone/
            └── other-repo/
                └── ask-matt/
```

因此同名 Skill 不在 AKM library 层冲突。

`Project Skill Library` 保存来源层级；某个执行器如何发现这些 leaf Skill Roots，由 executor adapter/执行器自身完成。AKM 核心不维护一个全局扁平 `<skill-name> -> package` 名称表。

每个 Package leaf 可以是项目侧 activation overlay：

```text
ask-matt/
├── SKILL.md            -> machine store
├── akm-package.toml    -> machine store
├── references          -> machine store
├── scripts             -> machine store
└── DEPENDENCIES.md     # project-local writable dependency status
```

## 8. Lock 中的软件依赖

Lock 可以保存 Package 的结构化 `[software]` requirement，方便知道某个 Package 声明了什么：

```toml
[[software]]
package = "Akira-TL/matt-skills/ask-matt"
name = "git"
requirement = ">=2.40"
```

但 Lock 不保存这台机器当前是否安装、路径在哪里或特殊依赖是否满足。

实际状态写入项目侧 Package leaf 的 `DEPENDENCIES.md`，并允许重新检查。

## 9. `sync`

`sync`：

1. 读取 `akm.toml`；
2. 根据已有 Lock 尽量保留仍合法的 GitHub Release/Git commit；
3. 解析 dependency closure；
4. 下载缺失 Package Artifact 或显式 Git source；
5. 校验并写入机器 Store；
6. 重建 `.akm/skills/<owner>/<repo>/<package>/`；
7. 对 `[software]` 做基础探测并更新每个 Package 的 `DEPENDENCIES.md` 状态区。

## 10. remove 与 orphan

删除一个顶层 target 后重新计算 dependency closure。

不再可达的 transitive Package 从 Project Skill Library 中移除，但机器 Store 可继续缓存，交由单独 GC 策略处理。

Reverse Dependency 直接由 Lock 中的 dependency edges 推导，不单独维护第二份状态。
