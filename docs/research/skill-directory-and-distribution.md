# Agent Skill 目录与分发约束调研

日期：2026-09-14

## 调研问题

AKM 在设计 Package Root、Release Artifact 与安装后 Skill View 前，必须先区分：哪些目录/文件规则属于当前 Agent Skills 规范，哪些只是 Claude/OpenAI 等客户端或生态实现的分发约定。

## 结论

### 1. 可移植 Skill 的规范单位是“一个目录”，目录根至少包含 `SKILL.md`

Agent Skills 官方规范把 Skill 定义为一个目录，唯一必需文件是根目录的 `SKILL.md`。规范推荐但不强制 `scripts/`、`references/`、`assets/`；同时明确允许 Skill 根目录包含其他文件与目录。因此 AKM 若在 Skill 根加入自己的工具元数据文件，本身不会破坏 Agent Skills 的目录结构规范。

`SKILL.md` 必须带 YAML frontmatter；`name` 与 `description` 必需。`name` 必须与父目录名一致，使用小写字母、数字和连字符。文件引用应相对 Skill root。

来源：

- Agent Skills Specification: https://agentskills.io/specification
- Canonical specification source: https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx

### 2. `scripts/`、`references/`、`assets/` 是推荐约定，不是封闭目录白名单

规范称这些目录为 optional conventions，并允许任意其他文件或目录。因此 AKM 不应把 Skill 内容建模成固定三目录 schema；Package builder 应把 Skill directory 视为一个有少量可验证入口规则、其余内容基本透明的目录树。

这也意味着 AKM 不应该重写 Skill 内部相对路径。Release Artifact 解包后的 Skill root 应保持作者发布时的内部相对结构。

来源：

- https://agentskills.io/specification

### 3. `SKILL.md` frontmatter 不是成熟的包依赖协议

当前规范的结构化字段只有 `name`、`description`、可选 `license`、`compatibility`、`metadata` 和实验性的 `allowed-tools`。`metadata` 只是 string-to-string map；规范没有 Skill dependency、Package identity、Lock、Release Artifact 或 transitive dependency schema。

因此复杂依赖直接塞入 `SKILL.md` frontmatter 会把分发层协议绑定到 Agent 消费层，而且无法自然表达完整依赖图。AKM 使用独立 manifest 仍然是合理方向，但 manifest 的位置与 Package/Skill 基数关系需要继续设计。

来源：

- https://agentskills.io/specification

### 4. Anthropic 的实现也把 Skill directory 本身当作能力载体

Anthropic 对 Agent Skills 的介绍同样把 Skill 描述为包含 `SKILL.md`、脚本和其他资源的目录，并强调 progressive disclosure：启动只加载 name/description，激活后读取 `SKILL.md`，需要时再读取同一 Skill directory 中的其他文件。

这支持一个重要兼容目标：AKM 安装后暴露给执行器的对象应该仍然是标准 Skill directory，而不是要求执行器理解 AKM Package wrapper。

来源：

- Anthropic Engineering, “Equipping agents for the real world with Agent Skills”: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

### 5. OpenAI 当前 Skills API 进一步表明“Skill bundle/version”可以是目录或 ZIP 的不可变版本

OpenAI 当前 Skills API 接受目录上传或单个 ZIP 创建 Skill，并提供独立的 Skill Version 资源；官方接口把版本创建描述为 immutable skill version，也支持下载对应版本的 ZIP bundle。

这不是 Agent Skills 规范本身，但说明至少一个主流客户端/平台已经把“一个 Skill 的目录/ZIP bundle + 不可变版本”当作分发接口。AKM 若让普通 Package 能直接对应一个标准 Skill directory，会更容易与这类平台互操作。

来源：

- OpenAI Skills API: https://developers.openai.com/api/reference/go/resources/skills
- Create Skill: https://developers.openai.com/api/reference/python/resources/skills/methods/create
- Create immutable Skill Version: https://developers.openai.com/api/reference/cli/resources/skills/subresources/versions/methods/create

### 6. Agent Skills 官方仓库里已经有人提出独立 package manifest，但它目前只是讨论稿

官方仓库 Discussion #210（2026-03-05）提出把分发元数据放在独立 `skills.json`，让 `SKILL.md` 保持不变；其草案允许一个 package 声明多个 Skill path，并配 lockfile。这与 AKM 的问题高度重合，但它不是已合并规范，不能直接当作标准。

它对 AKM 的价值主要在于证明两种模型都真实存在：

- 单 Skill package：Package Root 与 Skill Root 可以重合；
- multi-Skill package：Package Root 包含 manifest，再引用多个 Skill Root。

因此 AKM 当前应该先比较这两种布局的语义成本，而不是因为现有生态里出现某一种草案就直接继承。

来源：

- Agent Skills Discussion #210: https://github.com/agentskills/agentskills/discussions/210

## 对 AKM 的直接约束

基于上述一手资料，当前可以先固定以下“兼容性约束”，但还不能固定 Package cardinality：

1. **Skill Root 是执行器兼容边界。** 激活给 Agent 的目录必须保持有效 Agent Skills 结构，根有 `SKILL.md`，Skill 内相对路径稳定。
2. **Skill 内容默认 opaque。** 除 `SKILL.md` 与少量规范校验外，AKM 不限制 Skill 根只能有哪些目录。
3. **AKM Package metadata 与 Agent instructions 分层。** 依赖、Release provenance、integrity 等分发信息优先放独立 Package Manifest，不要求 Agent 读取。
4. **源码布局不应强迫等于安装布局。** 多 Skill repository 可以有自己的开发目录；AKM 构建产物只需生成自包含 Package Artifact。
5. **Release Artifact 必须能定位一个或多个完整 Skill Root。** 最终是否允许 1:N，应由 Package Root 设计票决定。
6. **不要把客户端规则升级成通用规范。** 例如 Anthropic 的作者指南可能推荐 Skill 内不放 README，但 Agent Skills 规范本身允许其他文件；AKM validator 应以规范为 portable baseline，再由可选客户端 profile 做更严格检查。

## 仍未解决

- `akm-package.toml` 应位于 Skill Root 内，还是位于包裹 Skill Root 的 Package Root；
- 普通 Package 是否强制 1:1 Skill，是否另外引入 Bundle/Meta Package；
- multi-Skill repository 的 Package source root 如何声明，是否需要 workspace-level manifest；
- Package 内多个 Skill 若共享 scripts/references，应该复制到各 Skill、允许 package-level shared payload，还是通过另一个机制表达；
- Release Artifact 解包根是否就是 Package Root，以及 manifest 是否随 payload 一起进入 Store。
