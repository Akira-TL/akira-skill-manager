# Package Manifest v0 工作草案

## 1. 目标

`akm-package.toml` 只描述一个 Skill Package 在 AKM 中如何版本化、依赖其他 Skill Package，以及哪些常见软件可以由 AKM 做基础探测。

它不复制 `SKILL.md` 的 Agent routing metadata，也不声明 GitHub owner/repository；GitHub source context 来自安装坐标：

```text
<owner>/<repo>/<package>@<version>
```

## 2. 文件位置

一个原生 AKM Package Root：

```text
<package-name>/
├── SKILL.md
├── akm-package.toml
├── DEPENDENCIES.md
├── scripts/                 # optional
├── references/              # optional
├── assets/                  # optional
└── ...
```

固定关系：

```text
Package Root == Skill Root
basename(Package Root) == SKILL.md.name == package.name
```

## 3. 最小 Manifest

```toml
schema = 1

[package]
name = "ask-matt"
version = "1.4.0"

[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
"Akira-TL/matt-skills/wayfinder" = "^1.4"
"Akira-TL/matt-skills/triage" = "^1.4"

[software]
git = ">=2.40"
gh = ">=2.45"
```

Package 没有 Skill dependency 或常见软件依赖时，对应 table 可以省略。

## 4. `schema`

```toml
schema = 1
```

表示 Manifest schema version。未知 schema 必须 fail closed。

## 5. `[package]`

### `name`

只是在当前 GitHub repository 内定位 Package 的局部名称：

```toml
[package]
name = "ask-matt"
```

它不是全局 Package ID。

完整 GitHub 安装坐标由 source context 补齐：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

v0 不提前引入 `namespace/package` 形式的独立 Registry identity。

必须满足：

```text
package.name
== basename(Package Root)
== SKILL.md.name
```

### `version`

```toml
version = "1.4.0"
```

当前 GitHub Release 模式中，Package version 必须与它所属的 Release version 一致。这样：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

可以直接解析 GitHub Release `1.4.0` 中的 `ask-matt` Artifact。

如果以后建立独立 Registry，可以再允许 Package version 与 GitHub repository Release lifecycle 解耦；v0 不提前引入这层复杂度。

Git `--git` 模式下，Lock 以 exact commit 为可重建依据；Manifest version 仍用于 dependency compatibility 判断，但 source ref 不必等于一个 GitHub Release。

## 6. `[dependencies]`

Skill dependency 使用与用户安装相同的 GitHub 坐标模型，只是版本范围单独作为 value：

```toml
[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
"Akira-TL/skills/browser-access" = ">=2.0 <3"
```

规则：

- dependency key 必须是 `<owner>/<repo>/<package>`，必须包含 package；
- dependency 不允许写成 repository-wide target，因为依赖必须精确到一个 Skill Package；
- dependency value 是允许的 Release version range；
- 安装 dependency 时默认仍走 GitHub Release；
- Git fallback 不静默发生，只有项目/source policy 明确允许 Git 模式时才使用；
- transitive dependency 使用同样的语法递归解析。

### Router

Router 不需要特殊 Package type。

例如 `ask-matt` 的 `SKILL.md` 负责告诉 Agent 什么时候使用 `implement`、`wayfinder`、`triage`；Package Manifest 只保证这些 Skill 被安装：

```toml
[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
"Akira-TL/matt-skills/wayfinder" = "^1.4"
"Akira-TL/matt-skills/triage" = "^1.4"
```

因此一个产品能力族由 Router + dependency closure 形成，不需要 bundle package。

## 7. 版本范围

v0 保留简单、常见的范围语义：

- `1.4.0`：精确版本；
- `^1.4` / `^1.4.0`：兼容范围；
- `~1.4` / `~1.4.0`：patch 级兼容范围；
- `>=1.4 <2`：比较器交集；
- `>=1 <2 || >=3 <4`：并集；
- `*`：任意稳定 Release。

动态标签如 `latest` 不进入 Package Manifest；用户交互层以后可把 `latest` 解析成某个具体 Release，再写入 Lock。

## 8. `[software]`

这里只声明 **AKM 内建探测器能够基础检查的常见软件**：

```toml
[software]
git = ">=2.40"
gh = ">=2.45"
python = ">=3.11"
node = ">=22"
```

这不是“AKM 负责安装的软件清单”。

AKM 对这些条目只负责：

1. 使用内建只读 probe 查找；
2. 尽可能读取版本；
3. 判断 `present / missing / incompatible / unknown`；
4. 把结果写入项目侧 `DEPENDENCIES.md` 状态区。

AKM **不负责安装、升级、卸载或选择系统 package manager**。

不常用、环境相关或无法机械表达的软件/硬件/服务要求写在 `DEPENDENCIES.md`，由 Agent 处理。

## 9. 不进入 Manifest 的内容

以下内容不写入 `akm-package.toml`：

- GitHub owner/repository：来自 source/install coordinate；
- description：来自 `SKILL.md`；
- Skill routing/invocation：来自 `SKILL.md`；
- `router = true`：Router 是 Skill 行为，不改变 Package 安装语义；
- release URL / asset URL：来自 GitHub Release 与 Lock；
- Artifact SHA-256：由 Release/Lock 记录，不由 Package 自己为自己背书；
- 特殊软件安装方案：写入 `DEPENDENCIES.md` 并由 Agent 判断；
- multi-Skill entry：v0 不支持。

## 10. 校验不变量

Package discovery、Release build 和安装前至少验证：

1. `schema` 已知；
2. `package.name` 合法；
3. `package.name == Package Root basename == SKILL.md.name`；
4. `package.version` 合法；
5. Release 模式下 `package.version` 与目标 GitHub Release version 一致；
6. dependency key 都是完整 `<owner>/<repo>/<package>`；
7. dependency version range 可解析；
8. Package 不依赖 Package Root 外的 runtime 文件；
9. Package 内没有第二个可独立激活的 Skill Root；
10. `DEPENDENCIES.md` 存在并符合 Agent dependency-check 文件格式。
