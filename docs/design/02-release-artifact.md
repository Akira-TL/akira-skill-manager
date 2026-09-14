# Release Artifact v0 设计

## 目标

Release Artifact 是一个 Package Release 的不可变交付物。正常稳定安装面向 Release Artifact，而不是直接 checkout 整个 Git repository。

一个 Git repository 可以发布多个 Package；每个 artifact 只对应一个 Package Identity 与一个精确版本。

## v0 归档格式

v0 选择 `tar.gz`，标准文件名：

```text
<namespace>--<skill-name>-<version>.akm.tar.gz
```

例如：

```text
akira--ask-matt-1.4.0.akm.tar.gz
```

选择原因：

- Linux/macOS/Windows 上均有成熟实现；
- 可保留必要的 executable bit；
- GitHub Release 对任意二进制 asset 都是自然承载；
- 不要求宿主提前具备 zstd；
- v0 的 Skill 内容以文本、小型脚本和资源为主，压缩率不是首要约束。

压缩算法不是 Package Identity 的组成部分。未来可以引入新的 artifact encoding，但同一 Package Release 在一个 Package Index 记录中必须指向明确的 artifact 与 digest。

## Artifact 内部布局

归档解包后根目录就是 Skill directory，不额外增加 repository 层：

```text
akm-package.toml
SKILL.md
scripts/
references/
assets/
...
```

因此 Package Store 中完成校验的解包目录可以直接作为 Project Skill View 的软链接目标。

## 安全与可移植性约束

v0 artifact 只允许：

- 普通文件；
- 目录。

v0 明确拒绝：

- 归档内 symlink；
- hardlink；
- device node；
- FIFO；
- absolute path；
- 含 `..` 的路径逃逸；
- 规范化后重复的路径；
- 解包后指向 Package 根目录之外的内容。

项目激活所需的 symlink 由 AKM 自己在 Package Store 与 Project Skill View 之间创建，不依赖 artifact 内预制 symlink。

## 完整性

每个 Release Artifact 使用 SHA-256 作为 v0 必需完整性摘要：

```text
sha256:<64 hex chars>
```

digest 对下载到的原始归档字节计算。

安装时至少执行：

1. 获取 Package Index 中声明的 artifact locator、size 与 SHA-256；
2. 下载到临时文件；
3. 本地计算 SHA-256；
4. digest 不一致则立即拒绝；
5. 在隔离临时目录中执行安全解包；
6. 读取并校验 `akm-package.toml`；
7. 校验 Manifest 的 `name` / `version` 与正在安装的 Package Release 一致；
8. 校验 `SKILL.md.name` 与 Package Identity 末段一致；
9. 成功后以原子方式进入 Package Store。

GitHub 当前 Release Asset API 自身提供 `digest` 字段，但 AKM 仍以 Package Index / Lock Record 中的 expected digest 为校验输入，并自行计算实际 digest。

## 可重建打包

发布工具应生成 deterministic archive，以减少同一内容在不同构建机产生不同 digest：

- 文件路径按 UTF-8 byte order 排序；
- uid/gid 归零；
- user/group name 清空；
- mtime 固定为发布输入的 `SOURCE_DATE_EPOCH`，缺失时使用 `0`；
- directory mode 规范化为 `0755`；
- executable regular file 规范化为 `0755`；
- 其他 regular file 规范化为 `0644`；
- gzip header 不写本地文件名，并固定时间戳。

Release 发布流程必须从干净 source tree 生成 artifact；Package Manager 安装端只依赖 artifact digest，不假设发布者真的使用了 deterministic builder。

## Package Index 映射

Package Index 中一个 Release 至少需要提供：

```text
Package Identity
Exact Version
Artifact Locator
SHA-256
Artifact Size
Manifest Metadata
Yanked State
```

GitHub Release adapter 可以把 locator 表示为：

```text
repository + release tag/id + asset id/name
```

但 resolver 的核心接口不能依赖 GitHub 对象模型。

## Release 不可变性

如果 Package Index 已经记录：

```text
akira/ask-matt@1.4.0 -> sha256:AAA...
```

之后观察到同一 `name + version` 指向 `sha256:BBB...`，AKM 必须视为 supply metadata conflict 并 fail closed，不能把它当作普通更新。

修复后的内容必须发布新版本。

## Git 与 path 来源的统一处理

Git/path 是开发与兼容 source，不直接改变 Package Store 的不可变语义。

AKM 在使用 Git commit/subdir 或本地 path 时，应先把选中的 Package directory 规范化为同样的 Package snapshot，计算 content digest，再写入共享 Package Store。Lock Record 额外保留 Git commit 或 path provenance。

这样“Release package”“Git package”“path package”在 Store 与 activation 以后共享同一模型；差异只保留在 source adapter 与 Lock provenance 中。

## 签名

v0 必须有 SHA-256 integrity，但不在本阶段强制设计公钥签名协议。

Package Index 的签名、Sigstore provenance、GitHub artifact attestation 等可以作为后续独立信任层加入；当前接口必须预留“artifact integrity”和“publisher trust”是两个不同概念，避免未来把 hash 误当身份认证。
