# GitHub Release Artifact v0 工作草案

## 1. 目标

AKM v0 的稳定分发入口直接使用 GitHub Release。

一个 repository Release 可以发布多个 Skill Package，但每个 Skill Package 都是独立 Artifact：

```text
Release 1.4.0
├── ask-matt.akm.tar.gz
├── implement.akm.tar.gz
├── tdd.akm.tar.gz
└── code-review.akm.tar.gz
```

安装：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

只需要获取 `ask-matt.akm.tar.gz`，再按 Manifest dependency closure 获取其他 Artifact。

安装：

```text
Akira-TL/matt-skills@1.4.0
```

则安装该 Release 中全部 `*.akm.tar.gz` Package Artifact。

## 2. Artifact 文件名

v0 使用：

```text
<package-name>.akm.tar.gz
```

例如：

```text
ask-matt.akm.tar.gz
```

版本不重复写入 Artifact 文件名，因为 version 已由 GitHub Release 选定。

AKM 校验：

```text
asset package name
== akm-package.toml package.name
== SKILL.md.name
```

## 3. Artifact 内部布局

Archive root 直接就是 Package Root 内容，不额外套 wrapper：

```text
SKILL.md
akm-package.toml
DEPENDENCIES.md
scripts/
references/
assets/
...
```

因此解包后的 snapshot 本身就是一个合法 Skill Package。

## 4. Release version 校验

假设用户请求：

```text
Akira-TL/matt-skills/ask-matt@1.4.0
```

AKM：

1. 找到 repository Release `1.4.0`；
2. 找到 `ask-matt.akm.tar.gz`；
3. 解包读取 `akm-package.toml`；
4. 要求 `package.name = "ask-matt"`；
5. 要求 `package.version = "1.4.0"`；
6. 要求 `SKILL.md.name = "ask-matt"`。

任一不一致都拒绝安装。

因此 GitHub v0 模式没有第二套隐藏 package version。

## 5. Package discovery

### Release 模式

GitHub Release 中所有符合：

```text
<valid-package-name>.akm.tar.gz
```

的 asset 都是 AKM Package candidate。

指定 package 时只查对应 asset。

未指定 package 时枚举全部 candidate，逐个校验 Manifest 和 `SKILL.md` 后安装。

Repository 的普通 Release asset，例如：

```text
source.zip
manual.pdf
screenshots.zip
```

不属于 AKM Package，不参与 discovery。

### Git 模式

显式 `--git` 时，不依赖 Release asset 名，而是在 checkout 中扫描同时含：

```text
SKILL.md
akm-package.toml
```

的目录作为 Package Root。

## 6. 归档格式

v0 采用 `tar.gz`。

原因：

- 可保留 executable bit；
- 各主要平台有成熟实现；
- GitHub Release 可直接托管；
- 不要求 zstd 等额外解压工具。

未来可以增加其他 encoding，但同一 Release asset 必须有明确 encoding 与 integrity。

## 7. 安全解包

Artifact v0 只允许普通文件和目录。

拒绝：

- symlink；
- hardlink；
- device node；
- FIFO；
- absolute path；
- `..` path traversal；
- 规范化后重复路径；
- 任何解包后逃出 Package Root 的内容。

Package 运行时也不能依赖 repository 里的 sibling/shared 文件。

## 8. 完整性

Artifact 下载后至少计算 SHA-256：

```text
sha256:<64-hex>
```

Lock 记录：

```text
GitHub repository
Release version/id
asset name/id
asset digest
local SHA-256
```

如果 GitHub Release Asset metadata 提供 digest，AKM 应比较 GitHub metadata 与本地实际 digest。

若未来发布流程需要更强的独立校验，可以在 Release 中增加 checksum/attestation asset；这不改变 Package Artifact 布局。

## 9. 安装流程

指定 Package：

```text
owner/repo/package@version
```

流程：

```text
resolve GitHub Release
  -> find <package>.akm.tar.gz
  -> download temp file
  -> verify digest
  -> safe extract
  -> validate akm-package.toml
  -> validate SKILL.md
  -> validate DEPENDENCIES.md exists
  -> insert immutable snapshot into machine Store
  -> materialize project Skill Library leaf
  -> run common dependency probes
  -> update project-local DEPENDENCIES.md status
```

## 10. Git fallback 不是 Release fallback

找不到：

```text
owner/repo/package@version
```

对应 Release/asset 时，默认返回明确错误。

只有显式：

```text
--git
```

或等价项目配置，才 clone repository 并走 Git Package discovery。

这避免一次“稳定版安装”在用户不知道的情况下变成任意 branch checkout。

## 11. Store

机器 Store 保存已经验证的不可变 Package snapshot。

Store 的内部目录可以内容寻址；它不需要复刻用户安装坐标层级。来源层级由 Lock 和 Project Skill Library 保存。

例如：

```text
machine store:
  sha256/<digest>/...

project library:
  .akm/skills/Akira-TL/matt-skills/ask-matt/...
```

Project leaf 允许把不可变 payload 以 symlink 形式复用，同时保留一个项目侧可写的 `DEPENDENCIES.md` 状态文件。
