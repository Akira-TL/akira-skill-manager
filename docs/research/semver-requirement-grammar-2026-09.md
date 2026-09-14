# Skiloom Release Requirement Grammar 调研（2026-09）

状态：Research

问题边界：为 #7 Class R 选择一套**已有、可精确定义、实现中立**的 SemVer version requirement 语义。只研究 Release requirement grammar / prerelease / candidate preference；不研究 Host Observation `[software]` grammar，也不照搬其他包管理器的 source/lock/feature 模型。

## 1. SemVer 2.0.0 本身不定义 dependency range language

Semantic Versioning 2.0.0 规范固定的是：

- `MAJOR.MINOR.PATCH` 与 prerelease/build metadata 的合法版本格式；
- version precedence；
- build metadata 不参与 precedence。

其规范 BNF 只描述 valid SemVer **version**，没有定义 caret、tilde、wildcard、OR 或 dependency range grammar。规范正文只是举例说明依赖可以表达 `>=3.1.0` 且 `<4.0.0` 的意图。

来源：Semantic Versioning 2.0.0，<https://semver.org/spec/v2.0.0.html>

因此 Skiloom 若只写“使用 SemVer range”，仍然是不完整协议；#7 必须绑定一套 range semantics。

## 2. Cargo Version Requirement

Cargo Book 官方文档定义了一套较小的 version requirement language：

- default requirement：裸 `1.2.3` 等价于 caret-compatible requirement；
- caret：`^1.2.3`；
- tilde：`~1.2.3`；
- wildcard：`1.*`、`1.2.*`、`*`；
- comparisons：`>=`、`>`、`<`、`<=`、`=`；
- 多个 requirement 用逗号连接并取**交集**；
- 没有 npm-style `||` union；
- prerelease 默认不匹配，除非 requirement 明确指定 prerelease；
- 一个 prerelease requirement 可以前进到同一 release tuple 的更新 prerelease，并可以最终前进到 SemVer-compatible stable release；
- requirement 中 build metadata 被忽略；
- resolver 一般偏好当前可用的最高满足版本。

官方来源：

- Cargo Book / Specifying Dependencies：<https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html>
- Cargo Book / Dependency Resolution：<https://doc.rust-lang.org/cargo/reference/resolver.html>

Cargo caret compatibility 对 `0.y.z` 使用“最左非零分量保持不变”语义，例如：

```text
^1.2.3 -> >=1.2.3, <2.0.0
^0.2.3 -> >=0.2.3, <0.3.0
^0.0.3 -> >=0.0.3, <0.0.4
```

这与 Skiloom 现有文档已经使用的 `^1.4` 表达习惯兼容。

## 3. npm / node-semver range language

npm 官方 `node-semver` 提供更丰富的 range language：

- primitive comparators；
- whitespace comparator intersection；
- `||` union；
- hyphen ranges；
- X-ranges / partial ranges；
- tilde；
- caret；
- prerelease opt-in rule。

官方来源：

- npm/node-semver：<https://github.com/npm/node-semver>
- npm semver docs：<https://docs.npmjs.com/cli/v6/using-npm/semver/>

它的 prerelease 默认规则是：只有 comparator set 中存在同一 `major.minor.patch` tuple 的 prerelease comparator 时，该 tuple 的 prerelease 才可匹配；否则 prerelease 默认排除。

node-semver 功能覆盖更广，但作为 Skiloom v0 normative grammar 有两个成本：

1. `||`、hyphen range、X-range 与多种 partial desugaring显著扩大 parser/test vector 面；
2. 官方仓库长期存在“README/BNF 与实际 parser 对空白等输入接受范围不完全一致”的公开 issue（例如 npm/node-semver #392：<https://github.com/npm/node-semver/issues/392>）。如果规范写成“行为等同当前 node-semver library”，独立实现会被迫追逐 library edge cases，而不是实现稳定的 Skiloom protocol。

因此不建议把某个 node-semver library version 本身作为 Skiloom 的 conformance oracle。

## 4. 对 Skiloom v0 的适配判断

Skiloom 与 Cargo/npm 都有一个关键差异：Skiloom version 首先属于 **GitHub repository Release**，同 repository 的多个 Skill Package constraint 共同选择一个 exact repository snapshot；并且 ADR 0011 已让普通 replay 完全 lock-preserving。

这意味着 v0 range language 最重要的不是“表达所有可能集合”，而是：

- 能表达常见 compatible update；
- 能表达 exact / lower / upper bounds；
- 多个 dependency constraint 能做确定性交集；
- prerelease opt-in 明确；
- explicit update 时能从候选 Release 中产生唯一、可测试结果。

Cargo requirement language已经覆盖这些需求，而且比 npm range grammar 少一整层 union/hyphen/X-range 组合复杂度。

## 5. 推荐

#7 v0 建议采用：

```text
Skiloom Release Requirement v1
= Cargo Version Requirement syntax/semantics profile
+ SemVer 2.0.0 version precedence
```

具体建议：

1. 支持 Cargo documented 的 default/caret/tilde/wildcard/comparison/comma-intersection forms；
2. v0 不支持 `||` disjunction、npm hyphen ranges 或 npm whitespace comparator-set grammar；
3. prerelease 使用 Cargo documented opt-in/upgrade semantics；
4. build metadata 不用于 requirement matching；
5. Project/Manifest 文档 SHOULD 推荐显式 caret（例如 `^1.4`）而不是依赖裸版本的 Cargo-default shorthand，以减少读者误把裸版本理解成 exact；
6. resolver 在 fixed candidate set 上以最高满足 SemVer precedence 为默认 preference；
7. Skiloom 自己发布 conformance fixtures，不把 Cargo executable 或某个 Rust library 当运行时依赖/oracle。

这一选择是“绑定既有 range semantics”，不是把 Cargo 的 registry、features、git/path dependency 或 lock resolver 模型引入 Skiloom。

## 6. 仍需由 #7 自己定义的部分

外部 range grammar 无法替 Skiloom回答：

- 多 repository、version-dependent dependency graph 下的 deterministic global search/backtracking order；
- equal SemVer precedence（例如只差 build metadata）的 GitHub Releases 如何 fail closed；
- Project Intent 中 requirement 的 canonical serialization/equality；
- explicit Git repository binding 时 Release requirement 如何处理；
- cycle 是否是 graph validity error；
- unsatisfiable constraint provenance 的 canonical machine-readable facts；
- previous Confirmed Resolution 是否影响 explicit update candidate ordering。

这些必须由 Skiloom #7 独立固定，才能满足 Class R conformance。
