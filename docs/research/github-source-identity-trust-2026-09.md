# GitHub Source Identity / Trust 事实核查（2026-09）

状态：Research

对应 Issue：#9 `Define package sources, indexes and trust policy`

问题边界：只核查 Skiloom Core v0 GitHub source profile 所依赖的外部平台事实，包括 coordinate 大小写、repository rename/transfer redirect、Release candidate visibility、tag/commit authority、immutable Release signal 与认证失败。本文不决定 Skiloom 协议；协议结论见后续 Source / Trust conformance 文档与 ADR。

## 1. `owner` / `repo` API path 参数大小写不敏感

GitHub REST `List releases` 等 repository-scoped endpoint 明确写明：

- `owner` name is not case sensitive；
- `repo` name is not case sensitive。

来源：

- GitHub REST Releases：<https://docs.github.com/en/rest/releases/releases>
- GitHub REST Repository Contents：<https://docs.github.com/en/rest/repos/contents>

### 对 Skiloom 的含义

GitHub source identity 不能把 `Owner/Repo` 与 `owner/repo` 当成不同 repository。Core 需要定义 case-insensitive semantic comparison，并给 canonical Lock / resolver ordering 一个唯一表示。

## 2. Rename / transfer 会建立 redirect，但旧 namespace 之后可能失去 redirect

GitHub 对 repository rename：

- web traffic 会 redirect；
- `git clone` / `git fetch` / `git push` 使用旧 URL 也会继续工作；
- 但如果后来在旧名称创建新的 repository，旧 redirect 不再工作。

来源：

- <https://docs.github.com/en/repositories/creating-and-managing-repositories/renaming-a-repository>

GitHub 对 repository transfer 同样会 redirect 旧 location，并警告：如果在旧 location 创建新的 repository 或 fork，redirect 会被永久删除。

来源：

- <https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository>

GraphQL `repository(owner:, name:, followRenames:)` 还明确提供 `followRenames` 开关；关闭后，以旧名字引用已 rename repository 会返回错误。默认值是 `true`。

来源：

- <https://docs.github.com/en/graphql/reference/repos>

Organization rename 也存在同类风险：旧 organization name 可被其他人重新取得，并通过新 repository 覆盖原 redirect。

来源：

- <https://docs.github.com/en/organizations/managing-organization-settings/renaming-an-organization>

### 对 Skiloom 的含义

Redirect 是 GitHub 当前路由状态，不是永久 repository identity 证明。Skiloom 若在 initial resolution / explicit re-resolution 中静默跟随 owner/repo redirect，会让同一个 dependency coordinate 在不同时间解析成不同 canonical source identity。

因此 protocol 应把真正的 owner/repo rename/transfer 视为显式 source-coordinate transition，而不是普通 case canonicalization。

## 3. Release listing 对不同权限可能返回不同记录

GitHub REST `List releases`：

- public/published releases 对所有可访问者可见；
- 只有具有 push access 的用户会在 listing 中收到 draft releases。

来源：

- <https://docs.github.com/en/rest/releases/releases>

### 对 Skiloom 的含义

如果 resolver 直接把 API 返回数组当 candidate set，不同凭据的独立实现可能产生不同结果。Skiloom Release source profile 必须显式排除 `draft=true` records。

## 4. GitHub `prerelease` flag 不是稳定的 SemVer prerelease authority

GitHub immutable releases 锁定 associated Git tag 与 release assets，但仍允许修改：

- release title；
- release notes；
- whether the release is marked as a pre-release；
- whether it is marked latest。

来源：

- <https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases>

### 对 Skiloom 的含义

即使 GitHub Release 是 immutable，`prerelease` boolean 仍是可变 presentation metadata。因此 Class R 已定义的 prerelease eligibility 应继续只由 normalized SemVer tag / requirement semantics决定，而不能用 GitHub `prerelease` flag 改写 candidate eligibility。

## 5. Existing tag 的 source authority 是 tag 本身，不是 `target_commitish`

GitHub REST Create Release 对 `target_commitish` 的定义明确说明：如果 Git tag 已经存在，该字段不使用（unused / ignored）。

来源：

- <https://docs.github.com/en/rest/releases/releases>

### 对 Skiloom 的含义

Release provenance 不能把 `target_commitish` 当 exact source identity。Core 必须从 actual Release tag 解析 Git object，并最终得到 exact commit；若 tag 最终不能 peel 到 commit，则该 Release 不能形成 repository source snapshot。

## 6. Immutable Release 是强 provenance signal，但不是 Package content identity

GitHub immutable release：

- published 后 associated Git tag 锁定到 specific commit；
- tag 不能移动；
- assets 不能被修改或删除；
- GitHub 自动生成包含 release tag、commit SHA 与 assets 的 release attestation。

来源：

- <https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases>
- <https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity>

GitHub 也说明自动 source zip/tarball 不能通过 release-asset verification command 校验，因为这些 archive 是请求时生成的。

### 对 Skiloom 的含义

ADR 0005 的现有边界合理：`immutable` 是 provenance/trust signal，不是 v0 admission requirement；Skiloom Package identity 仍应由 exact repository snapshot 内选中 Package 的 canonical Package Content Digest 决定，而不是 release archive bytes。

Release attestation verification 可以成为额外 trust capability，但没有必要成为 P/R/A Full Core 的前置条件。

## 7. `404` 不能可靠解释成 repository 不存在

GitHub REST 认证文档明确说明：无 token 或 token 权限不足时可能返回 `404 Not Found` 或 `403 Forbidden`；对 private resource，GitHub 也会用 `404` 避免确认资源存在。

来源：

- <https://docs.github.com/en/rest/authentication/authenticating-to-the-rest-api>
- <https://docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api>

### 对 Skiloom 的含义

Core source error 不应把一个普通 `404` machine-readably断言为“repository definitely does not exist”。Repository-level acquisition failure 至少需要能表达 `not-found-or-not-authorized` / unavailable boundary；只有已经成功访问 repository metadata 后，才适合进一步精确断言某个 Release、tag 或 ref 确实不存在。

Credentials、token storage、OAuth flow、GitHub App installation 等属于 implementation / product secret-management，不进入 Lock，也不属于 Core source identity。

## 8. GitHub Repository API exposes numeric IDs，但本轮没有发现 rename/transfer 稳定性规范承诺

GitHub REST repository response包含 `id` / `node_id` / `full_name` 等字段；但本轮官方文档核查没有找到“numeric repository ID 是 Skiloom 可以依赖的跨 rename/transfer 永久 identity contract”这一明确规范承诺。

来源：

- <https://docs.github.com/en/rest/repos/repos>

### 对 Skiloom 的含义

v0 不应为了 rename/transfer 自动迁移而把 undocumented stability assumption 引入 canonical Lock schema。继续以显式 GitHub owner/repo source coordinate + exact commit + Package Content Digest 建模更符合当前已接受三层 Lock。

如果未来 GitHub 明确提供适合作为协议依赖的 stable repository identity contract，可在新 source profile / Lock generation 中另行考虑。

## 9. 结论

对 #9 最有约束力的事实是：

```text
case difference
  -> GitHub semantic identity 相同，可 canonicalize

owner/repo rename / transfer
  -> GitHub 可能 redirect，但 redirect 可被旧 namespace reuse 破坏
  -> 不应作为静默 source identity migration

Release listing
  -> credentials 可能改变是否看到 drafts
  -> candidate set 必须明确过滤 drafts

GitHub prerelease flag
  -> immutable release 上仍可变
  -> 不属于 SemVer candidate authority

Release target_commitish
  -> existing tag 时 ignored
  -> exact tag -> exact commit 才是 source provenance

immutable release
  -> strong advisory provenance signal
  -> 不替代 canonical Package Content Digest

404/403
  -> 可能表示 authorization gap
  -> 不应泄露成不可靠的 existence assertion
```

这些事实足以支持一个严格但较小的 GitHub Source / Trust Core profile，不需要在 v0 引入 Registry、publisher PKI、repository numeric identity 或通用 provider framework。
