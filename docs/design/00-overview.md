# AKM v0 协议工作草案

## 当前阶段

当前仍处于协议收敛阶段，不进入 CLI/runtime 实现。以下文档是当前 working draft；已经明确拒绝的旧方案不再作为实现候选。

设计文件：

1. [`skill-package-layout.md`](skill-package-layout.md)
2. [`repository-discovery.md`](repository-discovery.md)
3. [`01-package-manifest.md`](01-package-manifest.md)
4. [`02-release-artifact.md`](02-release-artifact.md)
5. [`03-project-manifest-lock.md`](03-project-manifest-lock.md)
6. [`04-resolver-and-install-plan.md`](04-resolver-and-install-plan.md)
7. [`05-software-dependencies.md`](05-software-dependencies.md)
8. [`06-module-boundaries.md`](06-module-boundaries.md)
9. [`package-snapshot-digest.md`](package-snapshot-digest.md)
10. [`project-activation.md`](project-activation.md)
11. [`activation-runtime-state.md`](activation-runtime-state.md)
12. [`dependency-runtime-state.md`](dependency-runtime-state.md)

## 已明确的 v0 方向

- GitHub 是首个直接分发坐标系，不先建设独立 Registry；
- 用户安装目标使用 `<owner>/<repo>[/<package>]@<version-or-ref>`；
- 默认优先使用 GitHub Release；没有 Release 或明确要源码版本时显式进入 Git source；
- Release 只负责通过规范化 SemVer 选择 repository snapshot；v0 不定义 AKM 专用 per-Skill Release Asset；
- 一个 Skill Package 恰好包含一个 Skill；
- Package Root 与 Skill Root 重合；
- **唯一最低准入条件是合法 `SKILL.md`**；
- `akm-package.toml` 是可选结构化依赖元数据；
- `DEPENDENCIES.md` 是可选 Agent-readable 特殊依赖说明；
- Multi-Skill Package 不进入 v0；
- Router 是普通 Skill，能力族由 Router + 可见 dependency closure 形成；
- 运行时共享能力必须写成 dependency，不允许跨 Package hidden shared files；
- GitHub Release `@version` 只接受 SemVer，tag 允许可选前导 `v`，最终锁定实际 tag + exact commit；
- Git source `@ref` 最终锁定 exact commit；
- Git source 进入机器级 disposable source cache，再从 exact commit discovery/snapshot Skill Root；
- repository 默认零配置扫描合法 `SKILL.md`；可选 root-level `akm-repo.toml` 只过滤 discovery 范围，不改变 `SKILL.md` 的准入地位；`exclude` 优先于 `include`，v0 glob 只支持 literal / `*` / `**` / `?`；
- resolved Skill 直接扁平激活到项目 `.agents/skills/<activation-name>`；默认 activation name 等于 `SKILL.md.name`；
- 不同 source 的同名 Skill 会在 activation 层真实冲突；AKM 必须在写入前提示用户为新安装项 rename 或放弃，不能自动覆盖/自动改名；
- Package snapshot 以 `AKM-PACKAGE-V1` canonical tree hash 计算 `content-digest`：独立 nested Skill 从祖先 snapshot 裁掉，v0 禁止 symlink/特殊文件，只保留 relative path、executable bit 与 exact bytes；
- Package Store 直接以 `content-digest` 寻址并跨来源去重；Package payload 机器级共享且 immutable；
- AKM 项目状态统一位于 `.agents/.akm/`：`akm.toml`、`akm.lock`、`activation.lock`、`dependencies.lock`；
- `.agents/.akm/akm.toml` 保存 top-level requirements，Release 用 version string，Git 用 `{ git = "<ref>" }`；用户批准的本地 Skill rename 写入 `[renames]`；同一 repository 不允许混用多个 Git ref 或 Release/Git source；
- `.agents/.akm/akm.lock` 采用 canonical `requirement -> repository -> package` 三层结构，source provenance 只写一次，Package Record 只保存 `package-root`、`content-digest` 与 exact dependency edges；
- `.agents/.akm/activation.lock` 只保存 `activation-name`、Package coordinate、`content-digest` 与 `mode`（`symlink` / `junction` / `copy`）；POSIX 未 rename 优先 symlink，Windows 未 rename 优先 junction，rename 一律 copy；managed drift 默认 fail closed；
- AKM 只基础探测少量常见软件，不负责自动安装/修复宿主依赖。

## 核心关系

```text
GitHub repository snapshot
    ↓
SKILL.md discovery
    ├── Package A == Skill A
    ├── Package B == Skill B
    └── Package C == Skill C

optional akm-package.toml
    └── structured dependencies/software

optional DEPENDENCIES.md
    └── Agent-readable special requirements

Git Source Cache
    └── repository Git objects / exact commits

Package Store
    └── immutable selected Skill Root snapshots

Project Skill Activation
    └── .agents/skills/<activation-name>

AKM Project State
    └── .agents/.akm/{akm.toml, akm.lock, activation.lock, dependencies.lock}
```

## 当前没有的东西

v0 不提前引入：

- 必填 `akm-package.toml`；
- 独立 Registry namespace/package identity；
- Multi-Skill bundle；
- Package Index 作为 GitHub 的强制中间层；
- 全局 Skill name registry（只在单个项目 `.agents/skills/` activation 层要求名字唯一）；
- Software Provider 自动安装体系；
- Package 自定义系统安装脚本；
- repository runtime shared directory；
- 项目直接引用 mutable Git checkout。

## 下一步仍需收敛

- `.agents/.akm/dependencies.lock` 的最终字段与状态失效规则（当前见 Proposed [`dependency-runtime-state.md`](dependency-runtime-state.md)）；
- version range 的最终 grammar；
- Git Source Cache GC 的具体 LRU/size/age 策略；
- optional dependencies / feature flags 是否需要进入后续版本。

## 协议稳定后的候选实现顺序

1. GitHub coordinate + `SKILL.md` metadata parser；
2. Release/Git source cache + Package discovery；
3. optional AKM metadata parser + dependency resolver；
4. immutable Package Store；
5. flat `.agents/skills/` activation + collision/rename handling；
6. common dependency probes + `.agents/.akm/dependencies.lock`；
7. activation ownership/doctor/remove；
8. CLI/MCP surface。