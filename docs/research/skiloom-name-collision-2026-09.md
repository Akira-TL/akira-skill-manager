# Skiloom 命名碰撞核查（2026-09）

状态：Research

对应 Issue：#14 `Choose a collision-free product and CLI name`

目的：核查 `Skiloom` / `skiloom` 是否存在足以继续阻塞 Agent Skill 包管理协议项目的直接软件命名碰撞，并为 product / CLI / repository / distribution namespace 决策提供事实基础。

## 1. GitHub

2026-09-14 使用 GitHub CLI / GitHub repository search 核查：

```text
Akira-TL/skiloom
-> repository not found

search repositories: skiloom in:name
-> []
```

因此本轮核查时没有发现 GitHub 上名为 `skiloom` 的 repository，`Akira-TL/skiloom` 也尚未被占用。

这只是当前 GitHub namespace 状态，不是未来保留承诺。

## 2. Package registries

对 exact package name `skiloom` 查询：

```text
PyPI      -> 404
npm       -> 404
crates.io -> 404
```

因此在本轮核查时，三个常见软件发行 registry 都没有 exact `skiloom` package。

项目尚未选择 reference implementation language，因此本事实只用于命名冲突判断；不在协议设计阶段注册或发布空占位 package。

## 3. Broad web search

Broad web search 没有发现当前 Agent tooling / developer tooling 产品使用 `Skiloom`。搜索结果中出现过 2015 年 CrowdWorks 的 WEB marketing 屋号命名提案，其中包含 `SKILOOM` / `skilloom` 作为候选名称：

- <https://crowdworks.jp/public/jobs/239242/proposal_products/users/309030>

这不是本轮发现的当前软件产品或 package-manager identity，但说明 `Skiloom` 不是语言学上从未出现过的字符串。

## 4. 边界

本核查回答的是：

> 是否存在足以像 `AKM` / `akm`、`ASKM` 那样直接撞上当前 Agent/software tooling 的工程命名冲突？

当前答案：**未发现。**

本核查不构成：

- 商标法律意见；
- 全球公司名称清查；
- 域名所有权保证；
- package/repository 名称未来仍可注册的承诺。

## 5. 结论

`Skiloom` / `skiloom` 足以通过本项目当前的 collision-free engineering name bar，可以进入 #14 的正式 namespace 决策。

由于 reference implementation language 尚未决定，协议只应固定产品/CLI/protocol/repository 的 canonical public identity；未来具体 ecosystem distribution 使用 `skiloom` 作为首选 exact package name，并在实际发布前再次做 registry availability / ownership 校验。
