# 软件依赖说明与本地状态模型

状态：Working Draft

`.agents/.akm/dependencies.lock` 的进一步精简候选见 [`dependency-runtime-state.md`](dependency-runtime-state.md)；在该候选正式 Accepted 前，本文件现有 state schema 仅作旧工作草案参考。

## 1. Package 最低条件仍只有 `SKILL.md`

软件依赖能力是可选增强，不影响普通 Skill 安装。

一个 Package 可以只有：

```text
foo/
└── SKILL.md
```

也可以增加：

```text
foo/
├── SKILL.md
├── akm-package.toml       # optional structured dependencies/software
└── DEPENDENCIES.md        # optional Agent-readable special requirements
```

两种可选文件职责不同。

## 2. 三层依赖模型

```text
Skill dependency
→ optional akm-package.toml [dependencies]
→ AKM 自动解析/安装

常见可机械探测软件
→ optional akm-package.toml [software]
→ AKM 只读 probe
→ .agents/.akm/dependencies.lock

复杂软件/硬件/服务/数据/授权条件
→ optional immutable DEPENDENCIES.md
→ Agent 检查、解释
→ .agents/.akm/dependencies.lock
```

没有这些文件时，AKM 不猜测。

## 3. `DEPENDENCIES.md` 永不写状态

`DEPENDENCIES.md` 存在时，它属于 immutable Package payload，只写：

- requirement；
- check 方法；
- resolution guidance；
- 哪些环境修改必须先征求用户批准。

例如：

```markdown
# Dependencies

## Special requirements

### Blender
- Requirement: Blender 4.3+ with Example Add-on enabled.
- Check: verify version and Add-on state.
- Resolution: explain the gap and ask before modifying Blender.

## Agent procedure

1. Read `.agents/.akm/dependencies.lock` when available.
2. Check unresolved requirements.
3. Never modify this file.
4. Record observations in `.agents/.akm/dependencies.lock`.
5. Ask before environment-changing actions.
```

Package 升级不需要三方合并该文件。

## 4. `[software]` 是可选结构化 probe 输入

如果 Package 有 Manifest，可以声明：

```toml
[software]
git = ">=2.40"
gh = ">=2.45"
python = ">=3.11"
node = ">=22"
```

AKM 对已支持的软件最多：

- 找 executable/runtime；
- 尝试读取版本；
- 判断 requirement；
- 写本地 observation。

AKM 不执行安装、升级、PATH 修改、系统配置、驱动/服务管理。

没有 Manifest 就没有结构化 common software requirement；这不影响 Skill 安装。

## 5. `.agents/.akm/dependencies.lock`

当前宿主状态统一保存：

```text
<project>/.agents/.akm/dependencies.lock
```

它是可重建的本机状态，默认不提交。

示意：

```toml
lock-version = 1

[[package]]
coordinate = "Akira-TL/matt-skills/ask-matt"
content-digest = "sha256:..."
dependencies-doc-digest = "sha256:..."

[[package.software]]
name = "git"
requirement = ">=2.40"
status = "satisfied"
detected-version = "2.45.2"
location = "/usr/bin/git"

[[package.special]]
name = "GitHub authentication"
status = "unknown"
checked-by = "agent"
note = "Private repository access has not been checked."
```

状态先保持：

```text
unknown
satisfied
missing
incompatible
blocked
```

## 6. 状态失效

以下变化使相关 observation stale/unknown：

- Package content digest 改变；
- `DEPENDENCIES.md` digest 改变；
- `[software]` requirements 改变；
- executable/location 消失；
- 用户/Agent 要求重新检查；
- Agent 判断外部环境发生变化。

如果 Package 根本没有 `DEPENDENCIES.md`，就没有特殊依赖说明需要检查；如果没有 `[software]`，就没有 common probe 输入。

## 7. Package Store 与项目激活分离

原始 Package Store 始终 immutable。项目 executor-visible Skill 直接位于：

```text
<project>/.agents/skills/<activation-name>
```

未 rename Package 可以直接链接 Store；发生用户批准的 rename 时，项目 activation view 可以 materialize 并只对激活身份做必要改写。AKM 的本机 dependency observation 仍只写 `.agents/.akm/dependencies.lock`，不回写 Package Store。

## 8. Agent 边界

特殊依赖存在时：

1. Agent 读取 immutable `DEPENDENCIES.md`；
2. 读取 `.agents/.akm/dependencies.lock`；
3. 做只读检查；
4. 解释缺口；
5. 给出解决方案；
6. 涉及安装、升级、登录、下载、配置、服务或其他宿主修改时先取得用户批准；
7. 完成后重新检查，只更新 `.agents/.akm/dependencies.lock`。

## 9. 可复用 Agent 能力仍应成为 Skill dependency

如果某项依赖本质是另一个 Agent Skill 能力，而作者希望 AKM 自动安装它，就在可选 Manifest 中声明：

```toml
[dependencies]
"owner/repo/helper-skill" = "^1.0"
```

不要通过 repository shared runtime directory 隐式共享。