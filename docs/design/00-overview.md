# AKM v0 核心协议总览

## 本阶段范围

当前只固定包管理核心模型，不实现 CLI，也不迁移旧 `akira-skills` installer。

设计文件：

1. [`01-package-manifest.md`](01-package-manifest.md)
2. [`02-release-artifact.md`](02-release-artifact.md)
3. [`03-project-manifest-lock.md`](03-project-manifest-lock.md)
4. [`04-resolver-and-install-plan.md`](04-resolver-and-install-plan.md)
5. [`05-software-dependencies.md`](05-software-dependencies.md)
6. [`06-module-boundaries.md`](06-module-boundaries.md)

## 已固定的核心决定

- 一个 Package 恰好发行一个 Skill；
- Package Identity 使用 `namespace/skill-name`，与 repository 身份分离；
- Package Release 使用 Semantic Versioning 2.0.0；同一版本内容不可变；
- `SKILL.md` 不承载 AKM 结构化依赖协议；独立使用 `akm-package.toml`；
- 一个项目中同一 Package Identity 只允许一个版本；不同项目可锁不同版本；
- Package dependency 只写 identity + version range，不能自行引入新的 source；
- 正常稳定安装使用不可变 Release Artifact；Git/path 是显式开发/兼容 source；
- Project Manifest `akm.toml` 只写顶层需求；`akm.lock` 保存完整 exact graph；
- Lock 固定 exact version、exact source provenance 和 integrity；
- resolver 首选 PubGrub 风格实现，并额外拒绝 dependency cycle 和 Skill name collision；
- resolver 与执行分离；所有变化先形成完整 Install Plan；
- 软件依赖只声明 Software Catalog capability，不嵌入平台安装方法；
- v0.1 先实现 software doctor + plan，不自动修改宿主软件环境；
- Package content 机器级共享，项目激活视图只使用 symlink；
- remove 通过重新求解 dependency closure 完成，Store garbage collection 独立处理。

## 关键不变量

```text
Repository != Package
Package == one Skill
Release == immutable Package@Version
Manifest == intent/metadata
Lock == exact resolved graph
Store == immutable shared content
Project Skill View == symlinks only
Package dependency != source grant
Software requirement != installation method
Resolution != execution
```

## v0.1 推荐实现顺序

### Slice 1：typed metadata

- PackageIdentity；
- SemVer / VersionRequirement；
- PackageManifest；
- ProjectManifest；
- Lock model；
- TOML parser/validator/serializer；
- canonical manifest digest。

完成门槛：examples 可 round-trip，非法 identity/range/schema 有结构化错误。

### Slice 2：offline resolver

- InMemoryPackageIndex；
- transitive dependency；
- single-version constraint；
- lock preference；
- conflict derivation；
- cycle；
- Skill name collision；
- reverse dependency / orphan 计算。

完成门槛：完全不接网络/文件系统即可测试 A→B→C、diamond dependency、冲突、cycle、selective update。

### Slice 3：artifact + Store

- deterministic package builder；
- safe tar extraction；
- SHA-256 verification；
- Release/Git/path snapshot；
- content-addressed Store。

完成门槛：错误 digest、path traversal、identity drift 全部 fail closed。

### Slice 4：Project activation

- `.akm/skills` staging/reconcile；
- atomic switch；
- doctor；
- remove/orphan activation semantics。

### Slice 5：software doctor

- Software Catalog；
- Probe；
- requirement aggregation；
- SoftwareReport；
- InstallPlan 合并。

到这里才具备实现 CLI/MCP surface 的稳定核心。

## 仍明确留到后续的决策

- 首个远端 Package Index 的实际托管格式；
- GitHub Release publish automation；
- Package signature / attestation；
- Runtime 内部依赖（Python/R/Node libraries）；
- software Provider 自动 apply；
- executor-specific activation adapter；
- central registry namespace ownership；
- store GC 的跨项目发现机制；
- optional dependencies / feature flags。
