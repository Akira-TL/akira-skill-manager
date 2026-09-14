# Skill Package Layout 工作草案

状态：Proposed

对应 Wayfinder：`Choose the AKM Package Root and Skill Entry layout`

## 问题

AKM 必须同时处理四种不同位置：

1. 作者在 Git repository 中维护 Skill 的源码位置；
2. AKM 认定的 Package Root；
3. Release Artifact 解包后的目录；
4. 机器级 Package Store 与项目激活视图中的目录。

如果这四种位置没有明确关系，Package Manifest、Release 构建、完整性校验、软链接激活和多 Skill repository 都会产生隐式假设。

## 上游兼容底线

当前 Agent Skills 规范只要求 Skill 是一个目录，根目录含 `SKILL.md`；`scripts/`、`references/`、`assets/` 是推荐约定而不是封闭白名单，其他文件和目录允许存在。

因此 AKM 的 portable baseline 应满足：

- 激活给执行器的对象始终是合法 Skill Root；
- AKM 不重写 Skill 内部相对路径；
- AKM 不把 Skill 内容限制为固定目录 schema；
- Package metadata 可以作为额外文件存在，但 Agent 不需要读取它；
- 客户端自己的更严格规则只能作为 profile validator，而不是 AKM portable package 的基础规则。

事实依据见 `docs/research/skill-directory-and-distribution.md`。

## 候选 A：Inline Skill Package

一个普通 Package 恰好对应一个 Skill，Package Root 与 Skill Root 重合：

```text
ask-matt/
├── SKILL.md
├── akm-package.toml
├── scripts/                 # optional
├── references/              # optional
├── assets/                  # optional
└── ...                      # Agent Skills 允许的任意其他内容
```

这里：

```text
Package Root == Skill Root
```

`akm-package.toml` 是分发工具读取的文件；`SKILL.md` 仍然只服务 Agent discovery/activation。

### 多 Skill repository

Repository 不是 Package：

```text
matt-skills/
├── README.md
├── skills/
│   ├── engineering/
│   │   ├── ask-matt/
│   │   │   ├── SKILL.md
│   │   │   ├── akm-package.toml
│   │   │   └── ...
│   │   ├── tdd/
│   │   │   ├── SKILL.md
│   │   │   ├── akm-package.toml
│   │   │   └── ...
│   │   └── code-review/
│   │       ├── SKILL.md
│   │       ├── akm-package.toml
│   │       └── ...
│   └── productivity/
│       └── ...
└── repo-only tooling/
```

每个存在 `akm-package.toml` 的 Skill Root 是独立 Package source root。一次 repository checkout 可以承载很多 Package，但安装、版本、依赖和 Artifact 都按 Package 分开。

### Release Artifact

当前候选归档形状：

```text
akira-ask-matt-1.4.0.akm.tar.gz
└── ask-matt/
    ├── SKILL.md
    ├── akm-package.toml
    ├── scripts/
    ├── references/
    └── ...
```

Artifact 保留一个以 Skill name 命名的顶层目录。这样无论解包还是 Store 存储，`SKILL.md.name` 与物理父目录名都保持一致。

### Machine Store

候选 Store 对象形状：

```text
~/.local/share/akm/store/
└── sha256/
    └── <artifact-or-payload-digest>/
        └── ask-matt/
            ├── SKILL.md
            ├── akm-package.toml
            └── ...
```

项目激活只链接 Skill Root：

```text
<project-skill-view>/ask-matt
    -> ~/.local/share/akm/store/sha256/<digest>/ask-matt
```

因此执行器最终看到的仍然是标准 Skill directory，不需要理解 Package wrapper。

## 候选 B：Wrapper / Multi-Skill Package

Package Root 与 Skill Root 分离：

```text
engineering-bundle/
├── akm-package.toml
└── skills/
    ├── ask-matt/
    │   └── SKILL.md
    ├── tdd/
    │   └── SKILL.md
    └── code-review/
        └── SKILL.md
```

Package Manifest 需要声明多个 Skill Entry path；Project Skill View 再分别链接到 Package 内的各 Skill Root。

### 优点

- 能把一组 Skill 作为同版本、同 Artifact 的原子发行单元；
- package-level shared payload 更容易表达；
- 与 Agent Skills 官方仓库 Discussion #210 的 multi-Skill package 草案接近。

### 代价

- 一个 Skill 的升级会耦合整个 bundle 的版本；
- “安装一个 Skill”不再等价于“安装一个 Package”；
- dependency graph 的节点与最终激活 Skill 名称出现第二层关系；
- 同 Package 内部分 Skill 是否可独立依赖、禁用、覆盖会引出额外 resolver 语义；
- Store 对象不能直接作为一个 Skill Root 使用；
- 用户最初提出的“同仓 A/B/C…，安装 A 不能顺带安装 G/H/I…”问题更容易重新出现。

因此当前不建议把它作为 v0 普通 Package 模型。

## 候选 C：Repository-as-Package

```text
repo/
├── akm-package.toml
└── skills/
    ├── a/
    ├── b/
    ├── c/
    └── ...
```

这会重新把 Git repository 粒度带回安装和版本模型，与 AKM 的原始问题直接冲突。当前应视为 rejected candidate，而不是工作方向。

## 当前推荐

v0 优先采用 **Inline Skill Package**：

```text
Package Root == Skill Root
one ordinary Package == one Skill Entry
```

但这仍是 Proposed，而不是 Accepted。它目前的主要优势是：

1. 与 Agent Skills 标准目录天然兼容；
2. Git repository 与 Package 粒度彻底解耦；
3. Release Artifact、Store object 和 Project activation 都不需要目录转换层；
4. 一个多 Skill repository 可以独立发布其中任意 Skill；
5. Package dependency graph 与最终 Skill activation graph 基本同构；
6. OpenAI 当前的 directory/ZIP Skill bundle 与 immutable Skill Version 模型也更容易适配。

## 需要一起固定的文件规则

如果采用 Inline Skill Package，建议同时固定以下约束：

### Package Root

- 目录 basename 必须与 `SKILL.md.name` 一致；
- 必须包含合法 `SKILL.md`；
- 必须包含 `akm-package.toml` 才是原生 AKM Package；
- 其他 Skill 内容由 Agent Skills 规范决定，AKM 默认透明保留；
- Package 不允许依赖 Package Root 之外的相对文件才能运行。

### Symlink

源码 repository 内是否允许普通 symlink 可以另行讨论，但 Release Artifact v0 建议不包含 symlink/hardlink。Package builder 应把 Artifact 视为自包含普通文件目录树，避免解包逃逸与跨平台差异。

### README 等额外文件

Agent Skills portable validator 不应因为存在 `README.md` 或其他额外文件而拒绝 Package，因为当前规范允许任意附加内容。若 Claude 或其他客户端有更严格的作者建议，应由可选 client profile 给 warning/error。

### 多个 `SKILL.md`

普通 Inline Package Root 的根必须只有一个作为 Skill Entry 的 `SKILL.md`。`references/` 等内部资源中即使存在同名 Markdown 文件也不等于新的 Skill Entry；真正的嵌套 Skill Root 是否禁止，应在 validator 规则中明确定义。

## 多 Skill repository 的共享文件问题

一个 repository 可以有 repo-level 开发工具、测试 fixtures、文档和生成脚本，但一个可发布 Package Artifact 必须自包含。

v0 最简单的规则是：

```text
Package payload = Package Root 内的文件闭包
```

也就是说不能通过 `../shared/...` 让运行时依赖逃出 Package Root。若多个 Skill 需要相同 runtime resource，作者需要把它物化进各自 Package，或后续设计真正的共享 runtime dependency；不能依赖“它们刚好在同一个 Git checkout”。

这条规则对 Git source 尤其重要：Git checkout 只是构建/获取来源，不能成为运行时隐藏依赖。

## Source layout 与 Artifact layout 分离

AKM 不应要求 repository 根就是 Package Root。Package Locator 可以指向 repository 内任意明确 Package Root：

```text
repo = https://github.com/Akira-TL/matt-skills
ref = <commit/tag>
subdir = skills/engineering/ask-matt
```

构建或 Git compatibility install 只从该 `subdir` 形成 Package payload。其他 sibling Skill 不进入 Artifact，也不进入当前项目。

以后如需优化 monorepo discoverability，可增加 repository/workspace-level index，但它只负责列出 Package roots，不改变 Package 本身的 1:1 语义。

## 尚未决定

- 是否现在就把 ordinary Package = one Skill Entry 提升为 Accepted ADR；
- `akm-package.toml` 是否必须进入运行时 Store，还是 Store 另建 sidecar metadata；
- Artifact integrity 是对 canonical tar bytes、解压 payload tree，还是两者都记录；
- GitHub multi-package repository 如何命名 tag/release，才能允许各 Package 独立版本；
- 第三方只有 `SKILL.md`、没有 `akm-package.toml` 时，AKM 的 synthetic manifest 放在哪里；
- 是否需要可选 `akm-workspace.toml` 来显式列出 repository 内 Package roots。
