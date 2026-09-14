# 包管理协议调研：首轮一手规范

本轮问题边界：为 AKM 的 Package Manifest、Release Artifact、Project Manifest/Lock、Dependency Resolver 与软件依赖模型确定可复用的成熟语义，不讨论 CLI 交互。

## 1. Semantic Versioning 2.0.0

来源：[Semantic Versioning 2.0.0](https://semver.org/)

可直接采用的事实：

- 标准版本采用 `MAJOR.MINOR.PATCH`；
- 先行版本与 build metadata 是规范的一部分；
- 已发布的同一版本内容不得被修改，任何内容变化都必须发布新版本；
- SemVer 的前提是发行对象存在可描述的公共兼容性表面。

对 AKM 的影响：

- Package Release 必须不可变；
- Package version 采用完整 SemVer 2.0.0；
- Skill 的“公共兼容性表面”至少包括 Skill 行为契约、Package dependencies、Software Requirements 与被其他 Skill 引用的 bundled resources。

## 2. npm manifest 与 lock

来源：

- [package.json dependencies](https://docs.npmjs.com/files/package.json/)
- [package-lock.json](https://docs.npmjs.com/cli/v11/configuring-npm/package-lock-json/)

可直接采用的事实：

- manifest 中的直接依赖使用 package name → version range 表达；
- lock 描述实际解析出的精确依赖树；
- lock 记录 `resolved` source 与 `integrity`，用于重建相同内容；
- Git、tarball、本地路径等来源可以与 registry 包共存，但最终 lock 必须固定到精确内容。

对 AKM 的影响：

- Project Manifest 只保存顶层意图，Project Lock 保存完整 transitive graph；
- source 与 integrity 必须属于 Lock Record，而不是依赖用户记忆；
- Git ref 在 lock 中必须解析成 commit；Release source 必须解析成不可变 artifact + digest。

## 3. Cargo resolver 与 lock

来源：

- [Cargo Dependency Resolution](https://doc.rust-lang.org/cargo/reference/resolver.html)
- [Cargo Dependencies](https://doc.rust-lang.org/cargo/guide/dependencies.html)

可直接采用的事实：

- dependency resolution 的结果进入 `Cargo.lock`；
- 后续解析优先保留仍满足新约束的 lock 版本，以兼顾可重建性与增量更新；
- `--locked` / `--frozen` 可禁止解析过程改写 lock；
- selective update 应尽量避免无关依赖漂移。

对 AKM 的影响：

- 普通 `sync` 默认优先复用仍合法的 Lock Record；
- `update` 应具有显式解锁范围；
- `frozen` 模式必须拒绝 manifest 与 lock 不一致，而不是自动修复。

## 4. Agent Skills specification

来源：[Agent Skills Specification](https://agentskills.io/specification)

当前规范只定义 Skill directory、`SKILL.md` YAML frontmatter 以及 `name`、`description`、`license`、`compatibility`、`metadata`、`allowed-tools` 等字段。它没有定义结构化 package dependency graph、release artifact、lockfile 或软件依赖解析协议。

对 AKM 的影响：

- AKM 不应把新的依赖协议硬塞进标准 `SKILL.md`；
- Package Manifest 应作为 Skill 目录旁的独立协议文件；
- `SKILL.md` 继续作为 Agent Skills 兼容入口，Package Manifest 负责软件包管理语义。

## 5. GitHub Release Assets

来源：

- [GitHub REST API: Release assets](https://docs.github.com/en/rest/releases/assets)
- [GitHub REST API: Releases](https://docs.github.com/en/rest/releases/releases)

当前 GitHub Release Asset API 返回 asset 的下载定位信息，并提供 `digest` 字段，示例为 `sha256:<hex>`。Release 本身仍有 tag、asset URL、发布时间等元数据。

对 AKM 的影响：

- GitHub Release 可以作为第一阶段稳定 artifact transport；
- AKM 仍应自行下载后计算 SHA-256 并与受信 Package Index / Lock Record 比较，不能只相信下载 URL；
- GitHub tag 不是 Package Identity，也不应成为依赖解析主键；Package Index 负责把 `Package Identity + Version` 映射到具体 Release Asset。

## 6. 本轮形成的设计约束

1. Package version 使用 SemVer 2.0.0；version 一经发布内容不可变。
2. Project Manifest 表达顶层需求；Project Lock 表达精确 transitive graph。
3. 所有可重建 source 必须最终落到 immutable release artifact digest 或 Git commit。
4. Package 管理元数据独立于 `SKILL.md`。
5. Package 自己可以声明 provenance 信息，但不能自证为可信来源。
6. resolver 必须优先复用仍满足约束的 lock pin，并提供 frozen 与 selective update 语义。
7. GitHub Release 适合作为第一阶段 artifact transport，但解析器接口不能绑定 GitHub。

## 尚未由一手规范直接决定的问题

以下属于 AKM 自己的领域设计，不能伪装成外部规范结论：

- Package 命名空间格式；
- 一个 Package 是否允许暴露多个 Skill Entry；
- 一个项目是否允许同 Package 多版本共存；
- artifact 使用何种归档格式；
- 软件依赖 Provider 的信任和自动安装边界；
- registry / Package Index 的首阶段实现方式。
