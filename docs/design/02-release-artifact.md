# GitHub Release 获取与 Artifact v0 工作草案

## 1. 目标

AKM v0 优先使用 GitHub Release 作为稳定版本入口，但 **不要求 repository 为 AKM 专门发布自定义 Asset**。

用户请求：

```text
owner/repo[/package]@1.4.0
```

AKM 首先解析对应 GitHub Release/tag，再取得该版本对应的不可变源码快照，随后按 `SKILL.md` 发现一个或多个 Skill Package。

因此普通只有 `SKILL.md` 的 GitHub Skill repository，只要有 Release，也可以直接安装。

## 2. Release source 的两种获取路径

### 基础路径：Release 源码归档

所有 GitHub repository 都可以通过 Release/tag 对应的源码 archive 取得版本快照。AKM 可以下载该版本的 tarball/zip，安全解包后执行 Package discovery。

流程：

```text
resolve release/version
  -> resolve exact tag/commit
  -> download release source archive
  -> verify provider metadata / local digest
  -> safe extract
  -> discover SKILL.md Package Roots
  -> select requested package(s)
  -> snapshot into machine Store
```

这是 v0 的最低兼容路径，不要求仓库作者了解 AKM。

### 可选优化：AKM Package Asset

作者可以额外在 GitHub Release 上传：

```text
ask-matt.akm.tar.gz
implement.akm.tar.gz
```

如果请求明确 package，并且 Release 中存在可验证的同名 AKM Asset，AKM 可以直接下载该 Package Artifact，避免下载整个 repository snapshot。

但 Asset 是优化，不是 Package 准入条件。

## 3. Package discovery

无论 Release 源码归档还是 Git source，核心发现规则一致：

```text
tracked/versioned tree 中的 SKILL.md
        ↓
其父目录 = Package Root candidate
        ↓
解析 SKILL.md.name
        ↓
basename(root) == SKILL.md.name
```

可选文件：

```text
akm-package.toml
DEPENDENCIES.md
```

存在就读取增强信息，不存在不影响安装。

指定：

```text
owner/repo/ask-matt@1.4.0
```

则按 `SKILL.md.name == "ask-matt"` 选择唯一 candidate。

省略 package：

```text
owner/repo@1.4.0
```

则选择该版本快照中全部合法 Skill Package candidate。

## 4. Release version 的作用域

GitHub 模式的 `@1.4.0` 是 repository Release version，而不是 Package 内隐藏的第二套 version。

因此没有 `akm-package.toml` 时也不存在版本缺失：

```text
owner/repo/foo@1.4.0
              ^^^^^
          repository Release
```

Lock 最终保存 Release identity/tag、exact commit 与选中 Package Root。

如果可选 Manifest 存在，其中也不需要重复声明 version。

## 5. AKM 专用 Asset

如果作者提供：

```text
<package-name>.akm.tar.gz
```

Archive root 直接是 Skill Root：

```text
SKILL.md
akm-package.toml       # optional
DEPENDENCIES.md        # optional
scripts/
references/
assets/
...
```

校验至少包括：

```text
asset basename package name == SKILL.md.name
```

如果 Manifest 存在，再验证其 schema/dependencies/software 语法。

AKM Asset 不得包含多个独立 Skill Root。

## 6. 安全解包

AKM 管理的 Package Artifact v0 只允许普通文件和目录，并拒绝：

- absolute path；
- `..` traversal；
- 规范化后重复路径；
- device node；
- FIFO；
- 会逃出 Package Root 的链接或路径。

对于 GitHub provider 生成的 repository source archive，解包同样必须经过 path traversal 防护；完成 discovery 后只把选中的 Skill Root snapshot 放入 Package Store，不把整仓直接激活。

## 7. 完整性

AKM 对实际下载 bytes 计算 SHA-256，并把 source provenance 写入 `akm.lock`。

Release source archive 至少记录：

```text
repository
release/tag
exact commit
archive digest
selected package-root
package content digest
```

AKM Package Asset 额外记录：

```text
asset name/id
asset digest
```

GitHub API 若提供 provider digest，应与本地计算结果比较；hash integrity 与 publisher trust 仍是两件不同的事情。

## 8. Release 安装流程

```text
owner/repo/package@version
  -> resolve GitHub Release
  -> pin exact commit
  -> if valid package-specific AKM asset exists:
       download/verify/extract that asset
     else:
       download/verify/extract release source archive
       discover all SKILL.md roots
       select SKILL.md.name == package
  -> read optional akm-package.toml
  -> read optional DEPENDENCIES.md
  -> snapshot selected Skill Root
  -> put immutable snapshot into machine Store
  -> link Project Skill Library
  -> run any structured common software probes
  -> update .akm/dependencies.lock
```

Repository-wide install省略 selector 时选择全部发现的合法 Skill Root。

## 9. 没有 Release 时使用 Git Source Cache

如果没有合适 Release，AKM 不静默改变 source。用户显式选择 Git source 后：

```text
owner/repo/package@main --git
```

AKM 使用机器级 Git source cache：

```text
~/.cache/akm/git/github.com/<owner>/<repo>.git/
```

第一次获取 repository；以后安装同 repo 的 branch/tag/commit 复用 cache，只 fetch 缺失 objects/ref。

随后：

```text
resolve ref -> exact commit
  -> inspect/materialize exact commit from cache
  -> discover SKILL.md roots
  -> select package(s)
  -> snapshot selected Skill Root(s)
  -> Package Store
```

项目不直接引用 mutable Git cache。

## 10. Source Cache 与 Package Store

二者职责必须分开：

```text
Git Source Cache
- 缓存 repository Git objects
- 为 discovery 与 snapshot 提供源码
- 可删除并重新下载
- 不直接暴露给 executor

Package Store
- 保存已选择 Skill Root 的 immutable snapshot
- content-addressed 去重
- Project Skill Library 的真实链接目标
```

删除 Source Cache 不破坏已经存在于 Store 的项目环境；未来需要新的 Git commit 时再重新 fetch。

## 11. Cache GC

v0 至少保留简单策略：

- Git source cache 是 disposable cache；
- 可以按最近使用时间/容量清理；
- 清理前不需要检查 Project Skill Library，因为项目引用 Store，不引用 source cache；
- Package Store GC 则必须单独考虑项目引用和 Lock，不能与 source cache GC 混为一谈。