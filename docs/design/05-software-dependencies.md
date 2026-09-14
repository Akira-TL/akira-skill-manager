# 软件依赖检查文件与 Agent 处理模型

状态：Working Draft

## 1. 目标

AKM 负责 Skill Package 的安装与 Skill dependency 解析，但不试图成为通用系统包管理器。

软件、运行时、硬件、服务、数据库、驱动、外部应用和其他环境依赖可能非常复杂。v0 的职责边界是：

```text
AKM：常见依赖的基础发现 + 状态记录
Agent：特殊依赖的检查 + 解释 + 经用户批准后的处理
```

AKM 不负责自动修复缺失软件环境。

## 2. 每个原生 Package 都带 `DEPENDENCIES.md`

Package Root：

```text
<package>/
├── SKILL.md
├── akm-package.toml
├── DEPENDENCIES.md
└── ...
```

`DEPENDENCIES.md` 不是可执行脚本，也不是系统安装脚本。它是给 Agent 阅读的依赖检查与处理说明。

它承担三类信息：

1. Package author 声明的特殊依赖；
2. Agent 应如何检查这些特殊依赖；
3. 当前项目/宿主环境的依赖检查状态。

## 3. 为什么不能只靠结构化 TOML

例如下列依赖都很难由一个统一的系统包管理器处理：

- 特定版本 Blender，并要求某个 Add-on 已启用；
- CUDA、GPU driver、显存或特定 GPU capability；
- 某个 R/Bioconductor 环境；
- 数据库、参考基因组、模型权重或几十 GB 数据资源；
- Docker daemon 已运行并拥有某项权限；
- Chrome 已登录特定站点；
- 某个 MCP server 已配置；
- 专有软件 license 已激活；
- 一台远程服务器可以 SSH 登录；
- 特殊硬件已经接入。

因此 AKM 不能把所有依赖硬编码成 Provider，也不能替 Package author 编写通用安装脚本。

这些依赖更适合由 Agent 根据自然语言说明检查当前环境、解释缺口，再征求用户授权处理。

## 4. TOML 只声明 AKM 能基础探测的常见软件

`akm-package.toml` 可以有：

```toml
[software]
git = ">=2.40"
gh = ">=2.45"
python = ">=3.11"
node = ">=22"
```

AKM 内建少量稳定的只读探测器，例如：

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

具体支持集合是 AKM 自身实现细节和版本能力，不要求 Package author 为未知软件伪造结构化 ID。

对这些常见软件，AKM最多做：

```text
发现 executable/runtime
读取版本（能够可靠读取时）
比较最低/最高版本（能够可靠比较时）
记录状态
```

不会：

```text
apt install
brew install
winget install
pip install
npm install
修改 PATH
修改系统配置
启停服务
```

## 5. `DEPENDENCIES.md` 建议格式

```markdown
# Dependency Check

## Special requirements

### Blender
- Requirement: Blender 4.3+ with Example Add-on enabled.
- Check: verify `blender --version`, then inspect whether the Add-on is enabled.
- Resolution: explain the missing requirement and ask the user for approval before changing Blender or its Add-ons.

### Model weights
- Requirement: FooModel v2 weights, approximately 18 GB.
- Check: verify the configured model path and expected files.
- Resolution: ask the user before downloading or changing the configured model path.

## Current status

<!-- akm-status:start -->
- git >=2.40: satisfied — 2.45.2 at /usr/bin/git
- python >=3.11: satisfied — 3.13.7
- gh >=2.45: incompatible — detected 2.45.0 if package requires >2.45
<!-- akm-status:end -->

## Agent procedure

1. Read the AKM-generated status above.
2. Check every Special requirement that AKM cannot verify.
3. If all requirements are satisfied, continue using the Skill.
4. If something is missing or incompatible, explain the exact gap to the user.
5. Propose a concrete remedy, including side effects and scope.
6. Obtain explicit user approval before installing, upgrading, configuring, downloading, enabling services, or otherwise modifying the environment.
7. After the approved change, verify again and update Current status.
```

Package author 维护 `Special requirements` 和检查说明。

AKM 维护 `<!-- akm-status:start -->` 到 `<!-- akm-status:end -->` 之间的基础探测状态。

Agent 可以在完成特殊依赖检查后补充状态，但不能删除 author 的要求或伪造已满足结果。

## 6. 共享 Store 与可写状态的冲突

Release/Store 中的 Package 内容应该不可变，但 `Current status` 明显是当前项目和当前机器的状态，不能写回共享 Store。

因此 Project Skill Library 中的每个 Package leaf 是一个 **项目侧激活目录**，不是直接把整个 leaf 软链接到 Store。

示意：

```text
<project>/.akm/skills/Akira-TL/matt-skills/ask-matt/
├── SKILL.md            -> <store>/.../SKILL.md
├── akm-package.toml    -> <store>/.../akm-package.toml
├── references          -> <store>/.../references
├── scripts             -> <store>/.../scripts
└── DEPENDENCIES.md     # 项目侧可写副本
```

安装时：

1. Store 保存 Release Artifact 中原始、不可变的 `DEPENDENCIES.md` 模板；
2. Project Skill Library 首次激活时复制该模板为项目侧 `DEPENDENCIES.md`；
3. AKM 把常见软件 probe 结果写入状态区；
4. Agent 在项目侧文件中记录特殊依赖的当前检查情况；
5. Package 升级时，AKM 需要保留/合并项目状态区，而不能无条件覆盖用户或 Agent 已记录的状态。

这让 Package payload 仍可机器级共享，同时允许每个项目记录自己真实的软件环境状态。

## 7. 缺失依赖不会触发 AKM 自动修复

AKM 安装 Package 后，可以给出：

```text
DEPENDENCY CHECK

ask-matt:
  git >=2.40       satisfied
  gh >=2.45        satisfied

special requirements:
  1 unchecked item — see DEPENDENCIES.md
```

如果 common probe 发现缺失：

```text
DEPENDENCY CHECK

foo:
  docker >=27      missing
  2 special requirements unchecked

Package installed, but runtime requirements are not fully satisfied.
```

然后由 Agent 读取 Package 的 `DEPENDENCIES.md`。

Agent 可以建议用户：

```text
缺少 Docker >=27。
我可以帮助你检查当前平台适合的安装/升级方式；这会修改宿主环境，是否继续？
```

只有用户明确批准，Agent 才使用当前 harness 真实拥有的工具去处理。

AKM 自身不内置“缺 docker 就 apt install docker”这种策略。

## 8. 特殊依赖之间也可以引用 Skill dependency

如果所谓“特殊能力”其实可以作为 Agent Skill 独立提供，就不应该塞进 `DEPENDENCIES.md` 作为共享文件或人工步骤，而应拆成正常 Skill dependency。

例如多个 Skill 都需要一个专门处理 NCBI 数据库检查的 Agent 能力，可以依赖：

```toml
[dependencies]
"owner/repo/ncbi-database-helper" = "^1.0"
```

原则：

```text
可复用 Agent 能力 -> Skill dependency
常见可机械探测软件 -> [software]
复杂环境/软件/硬件/服务条件 -> DEPENDENCIES.md
```

## 9. Project Lock 不保存宿主软件现状

`akm.lock` 可以记录某 Package 声明了哪些结构化 `[software]` requirement，便于重建依赖需求；但不保存：

- `/usr/bin/git`；
- 当前 Python 安装路径；
- Docker daemon 当前是否运行；
- Blender Add-on 当前是否启用；
- GPU driver 当前状态。

这些都是宿主状态，记录在项目侧 `DEPENDENCIES.md` 或其他可重新检查的 local state 中。
