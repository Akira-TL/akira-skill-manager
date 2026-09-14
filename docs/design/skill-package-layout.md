# Skill 包与 GitHub 分发布局

状态：Working Draft

对应 Wayfinder：`Choose the AKM Package Root and Skill Entry layout`

## 1. 当前已经明确的基础模型

AKM v0 不设计独立 Registry，也不先发明类似 npm 的全局 Package Identity。当前分发坐标直接建立在 GitHub repository 上。

一个可安装目标使用：

```text
<github-owner>/<repository>[/<package>]@<version>
```

例如：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
Akira-TL/matt-skills@1.4.0
```

语义：

- `github-owner`：GitHub owner/user/organization；
- `repository`：GitHub repository；
- `package`：repository 内的 Skill Package 名称，可省略；
- `version`：GitHub Release 版本；
- 指定 `package` 时只安装该 Package 及其依赖闭包；
- 省略 `package` 时安装该 Release 中的全部 Package。

未来如果 AKM 建立独立 Registry，再增加独立的 registry package identity。GitHub 安装坐标与未来 Registry identity 必须作为两种不同 source model 明确区分，而不是提前把 Registry 命名模型强塞进 GitHub 模式。

## 2. Release-first，Git clone 必须显式选择

默认安装路径：

```text
owner/repo[/package]@version
        ↓
GitHub Release
        ↓
下载对应 Package Artifact
```

AKM 不因为找不到 Release 就静默切换到 Git clone。

如果 repository 没有可用 Release，用户显式选择 Git 模式，例如：

```text
akm install Akira-TL/matt-skills/ask-matt@main --git
akm install Akira-TL/matt-skills@<commit> --git
```

Git 模式下：

1. clone/fetch repository；
2. 将用户 ref 解析为 exact commit；
3. 不假设 Package 所在目录，先对该 commit 的 tracked Git tree 做 Package discovery；
4. 以 `akm-package.toml` 为原生 Package discovery anchor，其父目录必须同时含 `SKILL.md`；
5. 校验 `basename(root) == package.name == SKILL.md.name`；
6. 指定 package 时按 manifest 中的 `package.name` 选择，而不是按路径猜测；
7. 未指定 package 时选择全部合法 Package Root；
8. Lock 保存实际发现的 repository-relative Package Root；
9. 同 repository 的 sibling dependency 复用同一个 exact commit；
10. 仍按 Package dependency 递归解析其他 repository 的 Skill Package。

例如用户只知道：

```text
owner/repo/ask-matt@main --git
```

而真实路径可能是：

```text
agent-tools/routers/ask-matt/
```

这不要求用户提前知道路径。AKM 在 checkout 后通过 `akm-package.toml` 找到 `package.name = "ask-matt"` 即可。

Repository 内 `package.name` 必须唯一，Package Root 也不得互相嵌套；否则 `owner/repo/package` 无法稳定定位唯一 payload，应直接报 discovery error。

只含 `SKILL.md`、没有 `akm-package.toml` 的 GitHub Skill 属于兼容发现问题，后续单独设计；原生 Package discovery 不通过目录猜测静默伪造 Manifest。

Release 是稳定分发路径；Git 是开发、兼容和“尚未发布 Release”的显式路径。

## 3. 一个 Package 只包含一个 Skill

Multi-Skill Package 不进入 AKM v0。

一个 Package Root 就是一个标准 Agent Skill Root：

```text
ask-matt/
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
one Package == one Skill
```

因此 Package 不需要 `skill-path`、`entry` 或 multi-entry 描述。

如果一个 repository 维护多个 Skill，它维护的是多个独立 Package：

```text
matt-skills/
└── skills/
    ├── ask-matt/
    │   ├── SKILL.md
    │   ├── akm-package.toml
    │   └── DEPENDENCIES.md
    ├── implement/
    │   ├── SKILL.md
    │   ├── akm-package.toml
    │   └── DEPENDENCIES.md
    └── tdd/
        ├── SKILL.md
        ├── akm-package.toml
        └── DEPENDENCIES.md
```

Package discovery 的机械条件是一个目录同时存在 `SKILL.md` 与 `akm-package.toml`。`DEPENDENCIES.md` 是原生 AKM Package 的依赖检查文件，具体语义见 `05-software-dependencies.md`。

## 4. Router 通过 Skill dependency 组织能力族

AKM 不使用 Multi-Skill bundle 表达 Matt、Research 等能力族。

能力族由一个 Router Skill 加依赖闭包形成。例如：

```text
ask-matt
├── implement
│   ├── tdd
│   └── code-review
├── wayfinder
│   ├── research
│   └── grilling
└── triage
```

Router 本身仍是一个普通 Skill Package。它通过 `SKILL.md` 指导 Agent 何时使用哪些能力；`akm-package.toml` 只声明必须安装的 Skill dependencies。

因此：

```text
Skill Suite / Product
= Router Package + transitive dependency closure
```

而不是一个包含多个 Skill 的 Artifact。

## 5. 禁止跨 Package Runtime 共享

Git repository 可以共享开发期工具，例如 repository-level lint、test、release builder；但 Skill runtime 不允许通过 repository 相对路径依赖 sibling Package 或公共目录。

不允许：

```text
repo/
├── shared/runtime-helper.py
└── skills/
    └── a/
        └── scripts/run.py  # 运行时依赖 ../../../shared/runtime-helper.py
```

如果两个 Skill 都需要某项 runtime 能力：

- 它是另一个 Skill 能力：拆成 Skill dependency；
- 它是外部软件/环境：写入 `DEPENDENCIES.md`，常见部分可同时进入结构化 software requirements；
- 它只是少量 Package 私有资源：各 Package 自己包含。

原则是：**运行时共享必须显式变成依赖，不允许依赖“恰好在同一个 Git checkout”。**

## 6. 项目 Skill Library 不做扁平化

AKM 不把所有 Skill 直接放成：

```text
skills/
├── foo
└── bar
```

项目 Skill Library 按安装坐标保留 GitHub owner/repository/package：

```text
<project>/.akm/skills/
├── Akira-TL/
│   └── matt-skills/
│       ├── ask-matt/
│       ├── implement/
│       └── tdd/
└── someone-else/
    └── another-repo/
        └── ask-matt/
```

因此两个 repository 都存在 `ask-matt` 时，AKM 自身没有名称冲突：

```text
Akira-TL/matt-skills/ask-matt
someone-else/another-repo/ask-matt
```

AKM 只维护有来源层级的 Skill Library。某个执行器是否支持递归 Skill discovery、需要额外索引、还是需要建立自己的执行器视图，由 executor adapter 负责；AKM 核心协议不为了某个执行器而强制扁平化。

## 7. Package Root 名称

Package 名称是 repository 内局部名称，不承担全局唯一身份。

建议机械约束：

```text
basename(Package Root)
== SKILL.md.name
== akm-package.toml package.name
```

例如：

```text
skills/ask-matt/
SKILL.md.name = "ask-matt"
akm-package.toml package.name = "ask-matt"
```

完整安装坐标由外部 source context 组成：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

而不是在 Package 内再次声明 `akira/ask-matt` 这种全局 ID。

## 8. Release Artifact 当前建议

GitHub Release 是 repository 级版本发布。一个 Release 可以附带多个 Package Artifact；每个 Package 仍是独立 Artifact。

例如 Release `1.4.0`：

```text
ask-matt.akm.tar.gz
implement.akm.tar.gz
tdd.akm.tar.gz
code-review.akm.tar.gz
```

指定：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

只下载 `ask-matt` Artifact，再根据它的 Skill dependencies 继续解析。

指定：

```text
Akira-TL/matt-skills@1.4.0
```

安装该 Release 中全部 AKM Package Artifact。

Artifact 内部直接是 Package Root 内容，不再套一层 package-name wrapper：

```text
ask-matt.akm.tar.gz
├── SKILL.md
├── akm-package.toml
├── DEPENDENCIES.md
├── references/
└── scripts/
```

Package name 由 Artifact 名、`akm-package.toml` 与 `SKILL.md.name` 三方校验。

## 9. 未来 Registry 的分界

当前 GitHub 模式：

```text
github:owner/repo/package@version
```

版本由 GitHub Release 提供，source provenance 天然属于 GitHub repository。

未来如果建立 AKM Registry，可增加另一种坐标，例如：

```text
registry:scope/package@version
```

Registry 模式才需要独立 namespace ownership、package index、publisher identity 等模型。两者共享 Package Manifest、dependency graph、Store 与 Project Lock 的大部分结构，但 source identity 不混为一谈。
