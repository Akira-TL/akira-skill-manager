# Project Skill Activation v0

状态：Accepted

对应 ADR：[`0008-flat-agent-skill-activation.md`](../adr/0008-flat-agent-skill-activation.md)

## 1. 项目目录

```text
<project>/.agents/
├── skills/
│   ├── ask-matt/
│   ├── browser-access/
│   └── ask-matt-other/
└── .akm/
    ├── akm.toml
    ├── akm.lock
    ├── activation.lock
    └── dependencies.lock
```

`.agents/skills/` 是 executor-visible 的扁平 Skill 安装面；`.agents/.akm/` 是 AKM 私有项目状态目录。

## 2. 默认激活名

每个 resolved Package 的默认 activation name：

```text
activation-name = SKILL.md.name
```

因此未发生冲突时：

```text
owner-a/repo-a/foo -> .agents/skills/foo
```

source coordinate 不进入 executor-visible 目录层级。

## 3. Preflight 冲突检查

AKM 在任何项目写入前先构造完整 activation plan，并检查：

1. planned Package 之间是否产生相同 activation name；
2. `.agents/skills/<name>` 是否已存在；
3. 已存在项是否由当前 `.agents/.akm/activation.lock` 声明为同一个 AKM-managed activation；
4. rename 后的新名字是否仍然冲突。

如果目标名字被其他 Package 或未知既有内容占用，返回：

```text
ActivationNameConflict
```

交互模式提示用户：

```text
冲突：.agents/skills/ask-matt 已被占用

新 Package: someone/other-repo/ask-matt

选择：
1. 重命名新 Skill
2. 放弃本次操作
```

AKM 不提供“自动覆盖”作为冲突选项。

非交互模式没有预先保存的 rename 时直接失败。

## 4. Rename 配置

用户确认 rename 后写入 portable 项目配置：

```toml
# .agents/.akm/akm.toml

[renames]
"someone/other-repo/ask-matt" = "ask-matt-other"
```

`[renames]` 的 key 必须是 resolved Package coordinate；value 是项目内 activation name。

规则：

- activation name 必须符合标准 Skill name grammar；
- activation name 在 `.agents/skills/` 中必须唯一；
- rename 可以针对顶层 Package，也可以针对 transitive Package；
- rename 不改变 Package coordinate、source resolution、dependency graph 或 Store `content-digest`。

## 5. Rename materialization

标准 Agent Skill 要求目录 basename 与 `SKILL.md.name` 一致。因此：

```text
.agents/skills/foo-other/
└── SKILL.md  name: foo
```

是不合法的。

rename 后 AKM 创建项目本地 managed activation view：

```text
.agents/skills/foo-other/
└── SKILL.md  name: foo-other
```

该 view 从原始 immutable Package Store entry 派生；至少只改写顶层 `SKILL.md` frontmatter 的 `name`。原始 Package Snapshot 与 `content-digest` 不变化。

AKM 不自动重写 Skill 正文、scripts、references 中对旧名字的自然语言/业务引用。用户选择 rename 时必须看到这一风险提示。

## 6. 未 rename Package

未 rename 时，AKM 可以直接把：

```text
.agents/skills/<skill-name>
```

链接到：

```text
<machine-store>/sha256/<content-digest-hex>
```

只要 executor 从 `.agents/skills/<skill-name>` 观察到合法 Skill Root 即满足协议。具体 link/materialize 策略属于实现层；不能改变 Package Store 的 immutable 语义。

## 7. `activation.lock`

`.agents/.akm/activation.lock` 是本机可重建的 AKM activation ownership/state 文件，默认不提交。

示意：

```toml
lock-version = 1

[[skill]]
activation-name = "ask-matt"
coordinate = "Akira-TL/matt-skills/ask-matt"
content-digest = "sha256:3333..."
mode = "store-link"

[[skill]]
activation-name = "ask-matt-other"
coordinate = "someone/other-repo/ask-matt"
content-digest = "sha256:aaaa..."
mode = "renamed-view"
source-name = "ask-matt"
```

它用于：

- 区分 AKM-managed Skill 与用户/其他工具已有 Skill；
- update/remove 时只修改 AKM 自己管理的 entry；
- doctor 检查 activation 是否缺失、指向错误 Store entry 或被意外修改；
- 记录平台相关 activation materialization mode。

portable rename intent 不依赖 `activation.lock`，而在 `.agents/.akm/akm.toml [renames]` 中保存。

## 8. 删除与更新

AKM 删除 Package 时，只删除 `activation.lock` 明确归属于该 Package 的 `.agents/skills/<activation-name>`。

如果 managed activation 已被外部修改且不能证明仍是 AKM 生成内容，update/remove 必须 fail closed，不把未知用户内容当成可安全覆盖对象。

## 9. 与 Package Store 的边界

```text
Package Store
= 原始 immutable Package Snapshot
= source-independent content identity

.agents/skills
= 当前项目 executor-visible activation
= 可能使用原名，也可能使用用户批准的 rename

.agents/.akm
= Project requirements / locks / activation state / dependency observations
```

因此同名冲突是 activation 层问题，不回流到 Package Resolver，也不改变 Package Content Digest。
