# Repository Discovery Control 工作草案

状态：Working Draft

> 本文只提出 repository-level discovery control 的候选协议；在用户明确确认前不标记为 Accepted。

## 1. 问题

AKM 已确定：合法 `SKILL.md` 是 Skill Package 的唯一最低准入条件。

默认情况下，AKM 会在一个 exact repository snapshot 中发现所有合法 `SKILL.md`。这对普通仓库零配置可用，但下列仓库可能包含“形式上合法、实际上不希望被发布/安装”的 Skill：

```text
repo/
├── skills/
│   ├── ask-matt/SKILL.md
│   └── tdd/SKILL.md
├── examples/
│   └── demo-skill/SKILL.md
├── tests/
│   └── fixtures/fake-skill/SKILL.md
└── docs/
    └── example/SKILL.md
```

仅靠 `SKILL.md.name` 无法可靠区分正式 Skill 与 example/fixture。AKM 不应通过硬编码 `examples/`、`tests/` 等目录名猜测，因为任意目录名都可能是作者真实 Package 布局。

因此需要一个 **可选的 repository-level discovery control**。

## 2. 候选文件：`akm-repo.toml`

文件只在 repository root 识别：

```text
repo/
├── akm-repo.toml       # optional
├── skills/
└── ...
```

不用 `akm.toml`，因为该名称已经用于 Project Manifest；不用把它放进每个 Skill Root，因为 Package 的最低门槛仍然只有 `SKILL.md`。

候选最小格式：

```toml
schema = 1

[discovery]
include = ["skills/**"]
exclude = [
  "skills/**/examples/**",
  "tests/**",
  "fixtures/**",
]
```

`akm-repo.toml` 只控制 discovery scope，不定义 Package name、version、dependencies 或 Project requirements。

## 3. 零配置仍是默认主路径

没有 `akm-repo.toml` 时：

```text
include = ["**"]
exclude = []
```

语义是扫描整个 exact source snapshot 中的合法 `SKILL.md`。

因此普通第三方 Skill repository 完全不需要为了 AKM 增加任何文件。

存在 `akm-repo.toml` 时，它只缩小/明确 discovery 候选集合；不能让一个没有合法 `SKILL.md` 的目录变成 Package。

## 4. Pattern 匹配对象

`include` / `exclude` 匹配的是 **repository-relative Package Root path**，不是 `SKILL.md` 文件路径。

例如：

```text
skills/engineering/ask-matt/SKILL.md
```

Package Root path 是：

```text
skills/engineering/ask-matt
```

所以：

```toml
[discovery]
include = ["skills/**"]
```

可以选中该 Package。

这样配置与 Package 的目录语义一致，不需要作者写：

```text
skills/**/SKILL.md
```

## 5. Include / Exclude 语义

候选规则：

1. `include` 省略或为空时等价于 `["**"]`；
2. 非空 `include`：candidate root 至少匹配一条才进入候选集合；
3. `exclude` 默认空；
4. `exclude` 命中始终优先于 `include`；
5. pattern 统一相对 repository root；
6. pattern 使用 `/` 作为路径分隔符，与宿主操作系统无关；
7. v0 至少支持字面路径、`*`、`**`、`?`；字符类等更复杂语法是否进入 v0 可后定；
8. 不允许绝对路径或 `..` 逃出 repository root。

`exclude` 优先与 Cargo/uv 的 workspace 成员过滤惯例一致：一个路径即使被成员/include pattern 命中，也可以被显式排除。

## 6. Discovery 顺序

统一流程：

```text
exact repository snapshot
  -> read root akm-repo.toml if present
  -> enumerate versioned SKILL.md files
  -> derive Package Root paths
  -> apply include/exclude to Package Root paths
  -> parse/validate SKILL.md
  -> detect duplicate Package Name / nested selected roots
  -> select requested package(s)
```

注意过滤发生在 Package validation 之前，但 pattern 不能“修复”非法 Skill：最终被选中的 root 仍必须满足标准 Agent Skill 约束。

## 7. 重名 Package

过滤后如果仍存在：

```text
a/foo/SKILL.md     name: foo
b/foo/SKILL.md     name: foo
```

则 `owner/repo/foo` 仍然无法唯一解析，返回 `AmbiguousPackageDiscovery`。

`akm-repo.toml` 可以通过排除其中一个 root 消除歧义：

```toml
[discovery]
exclude = ["examples/foo"]
```

不增加 `package name -> path` 第二份映射；Package Name 始终以 `SKILL.md.name` 为唯一 source of truth。

## 8. Nested Skill Roots

候选：**过滤后不允许两个最终 Package Root 互相嵌套。**

例如：

```text
skills/foo/SKILL.md
skills/foo/examples/bar/SKILL.md
```

如果两者都进入最终 discovery set，则 repository-wide install 与 `foo` payload 边界都会产生歧义。

作者可以通过 repository control 明确排除 example：

```toml
[discovery]
exclude = ["skills/foo/examples/**"]
```

如果没有配置而 snapshot 中出现嵌套合法 Skill Root，当前候选行为是返回明确的 `NestedPackageDiscovery`，而不是静默猜测哪个才是正式 Package。

这条仍是需要用户确认的 frontier 决策。

## 9. Release 与 Git 使用完全相同的 Discovery Control

`akm-repo.toml` 属于 repository source snapshot，因此：

- GitHub Release source archive：读取该 Release snapshot 根目录中的 `akm-repo.toml`；
- Git source：读取 exact commit 根目录中的 `akm-repo.toml`；
- Git Source Cache 只负责取得 exact tree，不持有独立 discovery policy；
- Lock 保存最终选中的 `package-root` 与 source snapshot，不复制整个 repository discovery config。

这样同一 commit/tag 无论通过 Release archive 还是 Git source materialize，discovery 结果应一致。

## 10. Package-specific AKM Asset

Package-specific Asset 已经明确知道目标 Package，因此 `akm-repo.toml` 不参与 Asset 内部 discovery。

但 Release 的“有哪些正式 Package”若将来需要完全依赖 repository control，则 package-specific Asset 的枚举与 repository-wide install 仍需单独定义发布索引/metadata；v0 不应因为优化 Asset 路径而改变 source snapshot discovery 语义。

## 11. 不在这个文件里放什么

`akm-repo.toml` v0 不承担：

- Package name/path 映射；
- Package dependency；
- software dependency；
- Release version；
- Project requirements；
- Git source override；
- executor discovery 配置；
- build/release script；
- registry publisher identity。

这些属于其他已有协议层或未来独立设计。

## 12. 当前推荐

当前推荐候选是：

```text
文件：akm-repo.toml
位置：repository root
默认：不存在 = 全仓 SKILL.md 自动发现
作用：只过滤 Package Root discovery
字段：schema + [discovery].include/exclude
优先级：exclude > include
Package name：始终来自 SKILL.md.name
Release/Git：共享同一 discovery 语义
```

该方案保持“零配置兼容普通 Skill repository”，同时为大型 monorepo 提供明确的发布边界。

参考的成熟 workspace 配置先例：

- Cargo Workspaces：`members` + `exclude`，支持 glob，`exclude` 用于从成员集合中排除路径：https://doc.rust-lang.org/cargo/reference/workspaces.html
- uv Workspaces：`members` + `exclude`，二者都支持 glob，命中 `exclude` 的 Package 不进入 workspace：https://docs.astral.sh/uv/concepts/projects/workspaces/
- npm Workspaces：root `package.json` 的 `workspaces` 使用路径/glob 声明 workspace locations：https://docs.npmjs.com/cli/v11/configuring-npm/package-json/

AKM 只借鉴“仓库根控制成员范围”的机制，不继承这些生态的 package identity、lock 或依赖语义。

## 13. 尚待确认

1. repository control 文件最终是否命名为 `akm-repo.toml`；
2. nested selected Skill Roots 是 hard error，还是定义自动优先规则；
3. v0 glob grammar 是否只支持 `*` / `**` / `?`；
4. 是否需要 `default` / `publish` 一类第二组选集，还是 v0 坚持只有一个 discovery set。
