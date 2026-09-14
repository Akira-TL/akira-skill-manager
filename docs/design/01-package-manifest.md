# Package Manifest v0 设计

## 目标

Package Manifest 是 AKM 的发行与依赖解析元数据，不替代 Agent Skills 的 `SKILL.md`。`SKILL.md` 继续定义 Agent 如何发现和执行 Skill；`akm-package.toml` 只定义“这个 Skill 作为软件包如何被版本化、依赖和校验”。

## 文件位置

每个 Package 根目录必须同时存在：

```text
<package-root>/
├── akm-package.toml
├── SKILL.md
├── scripts/        # optional
├── references/     # optional
├── assets/         # optional
└── ...
```

Package 根目录本身必须是一个合法 Agent Skill directory，因此 Package Store 中的该目录可以直接成为项目软链接目标。

## 最小 Manifest

```toml
schema = 1

[package]
name = "akira/ask-matt"
version = "1.4.0"
repository = "https://github.com/Akira-TL/matt-skills"

[dependencies.skills]
"akira/domain-modeling" = "^1.2.0"
"akira/codebase-design" = "^1.3.0"

[dependencies.software]
git = ">=2.40"
```

## 字段语义

### `schema`

Manifest schema version。当前固定为整数 `1`。解析器遇到未知 schema 必须 fail closed，不做“尽量猜测”。

### `[package].name`

Package Identity，格式固定为：

```text
<namespace>/<skill-name>
```

v0 约束：

- `namespace`：小写 ASCII 字母、数字与 `-`；
- `skill-name`：必须满足 Agent Skills specification 的 `name` 约束；
- `skill-name` 必须与同目录 `SKILL.md` frontmatter 的 `name` 完全一致；
- Package Identity 与 Git repository 无绑定关系。

### `[package].version`

完整 Semantic Versioning 2.0.0 版本号。Release 后同一 `name + version` 的内容禁止变化。

### `[package].repository`

可选 provenance 字段，用于指向源码主页。它不产生 source trust，也不参与 Package Identity 判定。

未来可增加 `homepage`、`license`、`authors` 等描述性字段，但不应让 Package Manifest 逐渐复制 `SKILL.md` 的 Agent routing metadata。

## Skill 依赖

`[dependencies.skills]` 的 key 是 Package Identity，value 是 version requirement：

```toml
[dependencies.skills]
"akira/research" = "^2.1"
"akira/literature" = ">=1.4 <2"
```

v0 采用 SemVer 2.0.0 版本值，并采用 npm 风格的显式 range 语义：

- `1.2.3`：精确版本；
- `^1.2.3`：兼容范围；
- `~1.2.3`：patch 级兼容范围；
- `>=1.2 <2`：比较器交集；
- `>=1 <2 || >=3 <4`：范围并集；
- `*`：任意稳定版本。

部分版本如 `^2.1` 允许作为输入，并规范化为等价完整范围；lock 中只保存精确完整版本。

Package Manifest 不允许：

- `latest` 等动态 tag；
- 在 dependency value 中嵌入 Git URL、HTTP URL、path 或 registry URL；
- 使用 dependency alias 把另一个 Package Identity 冒充当前名字。

先行版本默认不进入普通 stable range，除非 range 显式包含先行版本。

## 软件依赖

`[dependencies.software]` 的 key 是 Software Catalog 中的稳定能力 ID，value 是该能力自己的版本约束表达式：

```toml
[dependencies.software]
git = ">=2.40"
github-cli = ">=2.50"
python = ">=3.11,<3.14"
samtools = ">=1.20"
```

软件版本不强制全部使用 SemVer；每个能力的版本 scheme、探测方式和 Provider 映射由 Software Catalog 定义。

Package Manifest 不携带平台安装命令。

## Compatibility

v0 仅保留最小、可机械判断的 compatibility：

```toml
[compatibility]
platforms = ["linux", "macos", "windows"]
```

未声明 `platforms` 表示 Package 自身不限制操作系统；最终是否可执行仍可能受 Software Requirements 约束。

暂不把具体 executor 名称写入基础协议。若某 Skill 只兼容特定 Agent 产品，应优先继续使用标准 `SKILL.md` 的 `compatibility` 文本表达；等出现稳定、可机械求解的 executor compatibility 需求后再扩展 Package Manifest。

## 校验不变量

打包、索引和安装前必须至少验证：

1. `akm-package.toml` schema 已知；
2. Package Identity 合法；
3. Package Identity 末段与 `SKILL.md.name` 一致；
4. version 是合法完整 SemVer；
5. Skill dependency 不允许引用自身；
6. 每条 Skill dependency 的 range 可解析；
7. 每条 Software Requirement 的 ID 必须能在当前 Software Catalog 中识别，或明确进入 unsupported 状态；
8. Package 内容中不存在第二个作为独立顶层 Skill 的 `SKILL.md`。

最后一条不是禁止 `references/` 中出现 Markdown，而是禁止一个 Package artifact 同时捆绑多个可独立激活的 Skill。

## 暂不进入 v0 的能力

- optional dependencies；
- feature flags；
- peer dependencies；
- dependency aliases；
- post-install hooks；
- Package 自定义安装脚本；
- Package 内多 Skill Entry。

这些能力会显著扩大 resolver 与授权模型，只有出现真实用例后再设计。
