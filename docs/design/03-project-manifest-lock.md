# Project Manifest 与 Lock v0 设计

## 目标

Project Manifest 只表达项目的顶层 Skill 意图；Project Lock 保存 resolver 实际选择出的完整、精确、可校验 dependency graph。

二者职责必须分开：

- Manifest 适合人类编辑；
- Lock 由 AKM 生成并提交到版本控制；
- transitive dependency 不回写 Manifest；
- 新机器优先按 Lock 重建相同 Skill 环境。

## 文件名

v0 采用：

```text
akm.toml
akm.lock
```

两者都使用 TOML。`akm.toml` 是用户文件；`akm.lock` 是生成文件，不接受手工维护。

运行时项目状态放在 `.akm/`，不与这两个 committed protocol files 混用。

## Project Manifest

最小示例：

```toml
schema = 1

[skills]
"akira/ask-matt" = "^1.4"
"akira/ngs" = "^2.0"
```

`[skills]` 中只出现用户主动选择的顶层 Package Requirement。

### 显式 source override

非默认 Package Index 来源必须由项目显式声明：

```toml
[sources."thirdparty/special-skill"]
kind = "git"
url = "https://github.com/example/special-skill.git"
ref = "main"
subdir = "skill"
```

或开发中的本地包：

```toml
[sources."akira/ask-matt"]
kind = "path"
path = "../matt-skills/skills/engineering/ask-matt"
```

规则：

- source override 只能由 Project Manifest 或更高层受信配置引入；
- Package transitive dependency 不能创建新的 source override；
- `git.ref` 是用户意图，Lock 必须把它固定为 commit；
- `path` 是开发来源，Lock 必须标记其非 portable 性质并保存当前 content digest；
- Project Requirement 的 version range 仍然约束 override source 中 Package Manifest 的 version。

## Project Lock

建议结构：

```toml
lock-version = 1
manifest-digest = "sha256:..."

[[package]]
name = "akira/ask-matt"
version = "1.4.3"
skill = "ask-matt"
source-kind = "release"
source = "github-release:https://github.com/Akira-TL/matt-skills/releases/tag/ask-matt-v1.4.3#akira--ask-matt-1.4.3.akm.tar.gz"
integrity = "sha256:..."
size = 18432
dependencies = [
  "akira/domain-modeling@1.2.1",
  "akira/codebase-design@1.3.4",
]

[[package]]
name = "akira/domain-modeling"
version = "1.2.1"
skill = "domain-modeling"
source-kind = "release"
source = "github-release:..."
integrity = "sha256:..."
size = 9216
dependencies = []

[[software]]
id = "git"
requirement = ">=2.40"
required-by = ["akira/ask-matt@1.4.3"]
```

上例是可读性示意；实现时 writer 必须使用稳定字段顺序和稳定 package 排序，保证 lock diff 可审阅。

## Lock Record 必须保存的内容

每个 Package 至少保存：

- Package Identity；
- exact version；
- exported Skill name；
- source kind；
- immutable source locator；
- artifact/content integrity；
- artifact size（若来源具备）；
- exact dependency edges。

不同 source kind 的附加 provenance：

### Release

- immutable artifact locator；
- SHA-256；
- 可选 release provider metadata。

### Git

- canonical repository URL；
- exact commit；
- subdir；
- normalized Package snapshot digest。

`branch` / `tag` / 用户输入的 `ref` 可以作为 provenance 保存，但不能替代 exact commit。

### Path

- project-relative canonical path；
- current Package snapshot digest；
- `portable = false`。

`frozen` 模式使用 path source 时，当前内容必须与 lock digest 一致。

## `manifest-digest`

`manifest-digest` 是对解析后的 Project Manifest canonical representation 计算的 SHA-256，不直接对原始 TOML 字节计算。

因此注释或无意义空白变化不会让 Lock 失效；真正改变 Project Requirement / source override / resolution policy 的变化才会改变 digest。

## Software Requirements 在 Lock 中的角色

Lock 保存完整 Package Graph 聚合后的 Software Requirements，但不保存“这台机器现在安装了什么”。

原因：

- 软件环境是 host state，不是 Package Graph；
- Linux、macOS、Windows 可能由不同 Provider 满足同一能力；
- 同一 Lock 应能在不同 host 上执行 doctor；
- Provider 实际执行结果属于本机状态，不应污染项目可移植 lock。

未来如果 AKM 管理 hermetic runtime，可以增加独立 environment lock，而不是让 v0 Project Lock 同时承担两个职责。

## 项目激活视图

逻辑布局：

```text
<project>/
├── akm.toml
├── akm.lock
└── .akm/
    └── skills/
        ├── ask-matt -> <machine-store>/.../akira/ask-matt@1.4.3
        └── ngs      -> <machine-store>/.../akira/ngs@2.0.2
```

`.akm/skills/` 整个目录由 AKM 拥有。AKM 可以根据 Lock 原子重建它，而不必维护第二份 Skill 内容。

v0 不规定 Claude Code、Codex 或其他 executor 最终从哪里读取 Skill；executor adapter 只能引用 Project Skill View，不再创建自己的 Package Store。

## 操作语义

### `sync`

- 读取 Manifest 与 Lock；
- 已锁版本仍满足当前约束时优先保留；
- 仅对被修改要求影响到的图重新求解；
- 必要时更新 Lock；
- 按新 Lock 重建 Package Store 缺失内容与 Project Skill View。

### `update`

- 显式放宽指定 Package 的 lock pin；
- resolver 可以选择该 Package 的新兼容版本；
- 其受影响 transitive closure 可以随之变化；
- 无关 Package 尽量维持原 lock。

### `frozen`

- `manifest-digest` 必须匹配；
- 不允许运行 resolver 产生新 Lock；
- 不允许修改 Lock；
- 可以按 Lock 中精确 locator 获取缺失 artifact，除非同时启用 offline policy。

### `offline`

这是与 `frozen` 正交的网络策略：

- 不访问 Package Index；
- 不下载新的 artifact；
- 只能使用当前 Lock 与本地 Store/cache 已存在内容。

## remove 与 orphan

删除一个顶层 Package 的正确操作不是“直接删软链接”，而是：

1. 从 `akm.toml` 移除该 Project Requirement；
2. 重新解析完整 graph；
3. 新 graph 中不可达的 transitive Package 成为项目 orphan；
4. 更新 Lock；
5. Project Skill View 移除 orphan 的激活链接。

机器级 Store 不立即删除 orphan 内容。Store garbage collection 是独立操作，应参考所有已知 Project Lock / pin 或采用保守缓存策略。

## reverse dependency

Reverse Dependency 不另存一套可漂移状态，直接从 Lock 的 dependency edges 反向构造。

它用于：

- `why <package>`；
- remove 影响解释；
- selective update 影响范围；
- orphan 判定；
- 冲突报告。
