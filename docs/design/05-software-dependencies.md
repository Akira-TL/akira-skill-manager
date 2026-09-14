# 软件依赖说明与本地状态模型

状态：Working Draft

## 1. 目标

AKM 负责 Skill Package 安装、Skill dependency 解析，以及少量常见软件的基础发现；它不成为通用系统包管理器。

依赖模型分成三层：

```text
Skill dependency
→ akm-package.toml [dependencies]
→ AKM 自动解析和安装

常见可机械探测软件
→ akm-package.toml [software]
→ AKM 只读 probe
→ 状态写入项目本地依赖状态文件

复杂软件/硬件/服务/数据/授权条件
→ Package 内不可变 DEPENDENCIES.md
→ Agent 阅读、检查、解释
→ 如需修改环境，先取得用户批准
```

## 2. `DEPENDENCIES.md` 是 Package 内容，下载后永不修改

每个原生 AKM Package Root 包含：

```text
<package>/
├── SKILL.md
├── akm-package.toml
├── DEPENDENCIES.md
└── ...
```

`DEPENDENCIES.md` 是 Package author 提供给 Agent 的依赖说明文件。它属于 Release Artifact 与 machine Store 中的不可变 Package payload。

AKM 和 Agent 都不得把当前机器状态写回该文件。

它只描述：

1. 哪些特殊依赖必须满足；
2. Agent 应如何检查；
3. 缺失时应该向用户解释什么；
4. 哪类修复动作需要先获得用户批准。

这样 Package 升级、校验、缓存和去重都不需要处理本地修改或三方合并。

## 3. 为什么还需要自然语言依赖说明

以下要求无法靠少量统一 probe 完整判断：

- Blender 某版本且指定 Add-on 已启用；
- CUDA、GPU driver、显存和特定计算能力；
- R/Bioconductor 组合环境；
- 模型权重、参考数据库、参考基因组或大型数据资源；
- Docker daemon 是否运行并具备特定权限；
- Chrome 是否已经登录某个站点；
- MCP server 是否配置且可用；
- 专有软件 license 是否激活；
- 远程服务器是否可 SSH 登录；
- 特殊硬件是否连接并工作正常。

因此 `DEPENDENCIES.md` 保持 Agent-readable，而不是演化成新的万能安装 DSL。

## 4. `[software]` 只声明 AKM 能基础探测的常见软件

`akm-package.toml` 示例：

```toml
[software]
git = ">=2.40"
gh = ">=2.45"
python = ">=3.11"
node = ">=22"
```

AKM 可以内建少量稳定的只读 probe，例如：

```text
git
ssh
gh
python
node
npm
uv
docker
java
R
```

具体支持集合属于 AKM 实现能力，不要求 Package author 为不常见软件伪造结构化 ID。

对已知软件，AKM最多：

- 发现 executable/runtime；
- 尝试读取版本；
- 在能够可靠比较时判断 requirement 是否满足；
- 写入本地状态。

AKM 不执行：

- `apt install` / `brew install` / `winget install`；
- `pip install` / `npm install`；
- PATH、系统配置、驱动、服务或登录状态修改。

## 5. `DEPENDENCIES.md` 建议格式

```markdown
# Dependencies

## Special requirements

### Blender
- Requirement: Blender 4.3+ with Example Add-on enabled.
- Check: verify Blender version, then inspect whether the Add-on is enabled.
- Resolution: explain the gap and ask the user before changing Blender or Add-ons.

### Model weights
- Requirement: FooModel v2 weights, approximately 18 GB.
- Check: verify the configured model path and expected files.
- Resolution: ask the user before downloading or changing the configured model path.

## Agent procedure

1. Read this file before using the Skill when dependency state is unknown or stale.
2. Read the current package entry in `.akm/dependencies.lock` when available.
3. Check every Special requirement that AKM cannot verify.
4. Record the observed result in `.akm/dependencies.lock`, never in this file.
5. If something is missing or incompatible, explain the exact gap and proposed remedy.
6. Obtain explicit user approval before any environment-changing action.
7. Re-check after an approved change and update `.akm/dependencies.lock`.
```

Package author 只维护依赖要求和检查方法；当前状态完全外置。

## 6. 项目本地 `.akm/dependencies.lock`

依赖检查结果写到项目本地、可重建的：

```text
<project>/.akm/dependencies.lock
```

它类似 package lock/state 文件，但与 `akm.lock` 职责不同：

- `akm.lock`：可提交、可移植，锁定 Skill Package graph 与精确 source；
- `.akm/dependencies.lock`：本机/本项目状态，默认不提交，可以随时重新检查生成。

建议结构：

```toml
lock-version = 1

[[package]]
coordinate = "Akira-TL/matt-skills/ask-matt"
version = "1.4.3"
content-digest = "sha256:..."
dependencies-doc-digest = "sha256:..."

[[package.software]]
name = "git"
requirement = ">=2.40"
status = "satisfied"
detected-version = "2.45.2"
location = "/usr/bin/git"
checked-at = "2026-09-14T17:20:00+08:00"

[[package.software]]
name = "gh"
requirement = ">=2.45"
status = "missing"
checked-at = "2026-09-14T17:20:00+08:00"

[[package.special]]
name = "GitHub authentication"
status = "unknown"
checked-by = "agent"
note = "Private repository access has not been checked yet."
```

状态值先保持小而明确：

```text
unknown
satisfied
missing
incompatible
blocked
```

`note` 是 Agent 可写的人类可读说明，不作为 resolver 输入。

## 7. 状态失效规则

`.akm/dependencies.lock` 只是观察缓存，不是真相来源。

下列情况必须重新检查或标记 stale/unknown：

- Package content digest 改变；
- `DEPENDENCIES.md` digest 改变；
- `[software]` requirement 改变；
- 用户/Agent 明确要求重新检查；
- probe 发现原记录的 executable/location 已不存在；
- Agent 判断特殊依赖环境发生变化。

Package 升级时不需要合并 `DEPENDENCIES.md`，只需要根据新 digest 使旧检查结果失效。

## 8. Project Skill Library 因此可以保持只读链接

因为所有可写状态都在 `.akm/dependencies.lock`，Project Skill Library 不再需要为 `DEPENDENCIES.md` 建 writable overlay。

可以直接：

```text
<project>/.akm/skills/Akira-TL/matt-skills/ask-matt
    -> <machine-store>/<exact-package-root>
```

整个 Package payload 保持不可变。

这比“复制一份 `DEPENDENCIES.md` 再合并状态”简单，也更利于升级、校验和垃圾回收。

## 9. Agent 的处理边界

当 common probe 或 `DEPENDENCIES.md` 检查发现缺失时：

1. AKM 只记录事实；
2. Agent 读取 Package 原始 `DEPENDENCIES.md` 与 `.akm/dependencies.lock`；
3. Agent 解释当前缺口；
4. Agent 给出当前平台可行的处理方案；
5. 涉及安装、升级、登录、下载、改配置、启停服务等动作时必须先获得用户批准；
6. 处理后重新检查，并只更新 `.akm/dependencies.lock`。

## 10. 共享能力仍然必须建模成 Skill dependency

如果某项“依赖”其实是可以复用的 Agent 能力，就不要通过 repository 共享文件或 `DEPENDENCIES.md` 隐式复用，而应该拆成正常 Skill dependency：

```toml
[dependencies]
"owner/repo/helper-skill" = "^1.0"
```

因此边界保持：

```text
可复用 Agent 能力 -> Skill dependency
常见可机械探测软件 -> [software]
复杂环境条件 -> DEPENDENCIES.md
当前环境观察 -> .akm/dependencies.lock
```
