# Agent Skill 包管理器竞争架构审计（2026-09）

状态：Research

目的：判断 Skiloom 当前协议是否只是已有工具的功能重组，识别真正可形成独立协议/行业标准价值的边界。

## 1. 结论先行

当前生态已经存在多种与 Skiloom 部分甚至高度重叠的工具。仅凭“SKILL.md + lockfile + content hash + machine-level store + symlink + frozen install”不足以构成新颖性；GSKILL 已经覆盖这组能力的大部分，sksync/skillspm/sklock 等也覆盖其中若干关键点。

Skiloom 若要形成独立标准价值，应把核心从“功能更多”收窄为“一个 Skill Package 的精确定义与跨实现可复现协议”，即：

1. `SKILL.md`-first：合法标准 Skill 无需额外 Manifest 即是 first-class Package；
2. one Skill = one Package：拒绝 bundle/multi-primitive Package 作为核心对象；
3. GitHub repository-scoped version source：Release version 选择 exact repository snapshot，再 discovery Package；Git source 显式分离；
4. zero-metadata discovery + optional strict dependency Manifest；
5. canonical Package Snapshot + source-independent content identity；
6. source / immutable package / project activation / host observation 四层状态严格分离；
7. `.agents/skills/` 扁平 activation 的 ownership 与冲突语义是协议的一部分，而不是“复制文件”实现细节；
8. 不把任意 installer、MCP、prompt、agent、hook、系统包管理等纳入 v0 核心。

这组边界的价值不是单项功能首次出现，而是把 Agent Skill 做成一个可由多个独立实现共同消费的最小、严格、可组合软件包模型。

## 2. 高重叠项目

### 2.1 GSKILL

来源：https://github.com/glapsfun/gskill

已具备：

- `SKILL.md`-based Skill package management；
- reproducible lockfile；
- content hash / verify；
- frozen install；
- machine/user-level shared store；
- cross-project reuse；
- symlink/copy activation；
- Git/local sources；
- offline restore；
- store GC。

对 Skiloom 的压力：很高。Skiloom 不能再把“全局 Store + lock + hash + symlink”作为主要差异化。

目前未观察到其公开模型明确提供 Skiloom 已设计的：repository-scoped GitHub Release version model、Skill-to-Skill transitive dependency Manifest、canonical tree identity 规范、flat activation rename/ownership state、host dependency observation sidecar 等完整组合。

### 2.2 skillspm

来源：https://github.com/sheng-gou/skillspm

已具备：

- `skills.yaml` intent；
- `skills.lock` exact accepted version + digest + provenance；
- machine-local library；
- provider/public GitHub recovery；
- lock-first reproduction；
- fail-closed digest mismatch；
- target sync/adopt/doctor；
- portable packs。

对 Skiloom 的压力：高。尤其“project intent + lock + machine-local library + digest”并非新概念。

### 2.3 sksync

来源：https://github.com/takemo101/sksync

已具备：

- GitHub/local/skills.sh sources；
- repo-root SKILL.md discovery；
- lockfile + resolved commits/hashes；
- symlink activation；
- drift/conflict detection；
- safe non-overwrite semantics；
- doctor/outdated/update/install；
- bundle membership；
- managed link ownership。

对 Skiloom 的压力：高。activation safety / drift 也不是空白领域。

Skiloom 仍可区别于它：不把“多 Agent target mapping”作为核心；强调 canonical Skill Package identity、dependency resolution、Release semantics 与严格状态层次。

### 2.4 sklock

来源：https://github.com/artieax/sklock

已具备：

- Agent Skills dependency graph；
- nested skills；
- `contentHash`（自身内容）与 `closureHash`（完整子树）；
- reproducible lock；
- dependency visualization。

对 Skiloom 的压力：中高。证明“Skill dependency graph + content identity”已有实现。

但其 dependency model 基于 recursive/nested Skill workspace，而 Skiloom 的核心是跨 GitHub repository/package coordinate、explicit dependency edge、repository-scoped source resolution。

### 2.5 Agent Skills Discussion #210

来源：https://github.com/agentskills/agentskills/discussions/210

社区已提出：

- Skill package manifest；
- git-addressed dependencies；
- `skills.lock`；
- exact commit；
- content integrity digest；
- transitive dependency resolution。

因此 Skiloom 不可声称“首次提出 Agent Skill dependency manifest/lockfile”。若要成为标准，必须在协议精度、兼容性和实现中立性上超过该 proposal。

## 3. 邻近但产品边界不同

### Microsoft APM / OpenAPM

来源：
- https://github.com/microsoft/apm
- https://microsoft.github.io/apm/specs/openapm-v01/

APM 已具备成熟的 manifest、lock、transitive resolution、content hashes、frozen replay、policy、multi-source、`.agents/skills` deploy，并已发布 OpenAPM normative Working Draft。

但 OpenAPM 的 Package/primitive type system覆盖 instructions、prompts、agents、skills、commands、hooks、MCP 等多个类型，并支持 APM package、Skill bundle、Skill collection、Plugin collection。其核心目标是“agent context package manager”。

Skiloom 不应与其比功能覆盖面，而应保持：

```text
OpenAPM = broad agent context packaging/deployment standard
Skiloom = narrow Agent Skill package/dependency/content identity standard
```

如果 Skiloom 后续开始吸收 prompts/MCP/hooks/agents/plugin marketplace，就会直接丧失这一差异化。

## 4. 非新颖能力清单

以下能力不能单独作为 Skiloom 的标准化卖点：

- 从 GitHub 安装 `SKILL.md`；
- repository 内 Skill discovery；
- manifest；
- lockfile；
- exact commit pin；
- content hash；
- frozen install；
- machine/global library；
- symlink/copy activation；
- drift detection；
- transitive dependency graph；
- doctor/update/remove；
- provider/source abstraction。

生态中都已有一个或多个实现。

## 5. Skiloom 值得守住的标准化楔子

### 5.1 Package admission 是标准 Skill 本身

合法 `SKILL.md` 即 first-class Package；作者无需先迁移到某个新 ecosystem manifest。

可选 Manifest 只增强 dependency/software metadata，不重新声明 identity/version/description。

### 5.2 Repository version 与 Package identity 分离

GitHub v0 中版本首先属于 repository Release；一个 exact repository snapshot 被 discovery 成若干 one-Skill Packages。Package 不自报版本。

这避免一个 repository 中多个 Skill 各自伪造 release/version lifecycle，也自然支持同仓依赖统一 snapshot。

### 5.3 Canonical Package Snapshot 是协议对象

Package content identity 不等价于：

- Git commit；
- GitHub tree SHA；
- archive SHA；
- `SKILL.md` hash；
- host materialized directory hash。

Skiloom 已定义独立的 canonical snapshot boundary和 `AKM-PACKAGE-V1` 树序列化模型；后续重命名时应迁移协议 domain tag，但保持该思想。

这允许不同 source/commit/repository 在 Package 内容相同时得到同一 content identity。

### 5.4 四层状态不能混

```text
Source Cache
  acquisition acceleration

Immutable Package Store
  canonical content identity

Project Resolution Lock
  source provenance + dependency graph

Project Activation / Host Observation
  local materialization + local environment state
```

多个现有工具把其中两到三层合并进一个 lock/cache/library。Skiloom 应把分层作为规范性要求。

### 5.5 Activation ownership 是 Package Manager 安全语义

`.agents/skills/` 是扁平公共命名空间：

- 未管理内容永不接管；
- 同名冲突 rename/abort；
- rename 必须保持目录 basename 与 `SKILL.md.name` 一致；
- managed drift fail closed；
- activation state 与 package resolution 分离。

这是 Skiloom 可形成清晰跨实现互操作语义的一块。

### 5.6 不执行任意 host installers

Skill dependency、common software observation、special human/Agent-readable dependency instructions 分层；包内容不能通过 Package Manifest 获得任意 host install hook 权限。

这一边界使 Skill package manager 保持数据/协议工具，而不是第二个 Agent runtime/plugin system。

## 6. 要成为“行业标准”还缺的东西

新颖架构不等于标准。要成为可被其他实现采用的行业标准，Skiloom 后续至少需要：

1. 协议与 CLI 实现分离：发布独立、版本化、实现中立的规范；
2. RFC 2119/8174 规范词；
3. machine-readable schemas；
4. conformance fixtures/test suite；
5. independent implementation test vectors，特别是 canonical digest；
6. version-range grammar 精确绑定，不自创模糊 SemVer dialect；
7. security/trust model；
8. migration/evolution policy；
9. 至少验证第二个独立 consumer 能实现同样 lock/digest/discovery 结果。

在没有独立实现之前，只能称“reference protocol / proposed standard”，不能声称 industry standard。

## 7. 设计红线

为了维持标准化楔子，v0 应明确拒绝：

- prompts/agents/hooks/MCP/plugin package 类型；
- arbitrary install/build/postinstall hooks；
- 强制 registry；
- 强制 package manifest 才能安装普通 `SKILL.md`；
- multi-Skill Package；
- source-specific content identity；
- project lock 与 local activation/environment state 混写；
- 为支持某个具体 Agent harness 把核心协议变成 target adapter framework。

这些能力可以由上层工具、未来 extension spec 或其他产品承担，但不进入 Skiloom Core。
