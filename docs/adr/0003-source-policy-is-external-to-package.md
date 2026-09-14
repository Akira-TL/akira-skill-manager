# ADR 0003：Skill dependency 与 source 分离

- 状态：Rejected
- 日期：2026-09-14

## 原提案

原提案让 Skill dependency 只声明抽象 Package Identity 与 version range，再由独立 Package Index 或 source policy 决定实际获取位置。

## 拒绝原因

AKM v0 不先建设独立 Registry，也不提前引入抽象 Package Identity。

GitHub 是当前直接使用的分发坐标，因此 dependency 写成：

```toml
[dependencies]
"Akira-TL/matt-skills/implement" = "^1.4"
```

`owner/repo/package` 已经同时表达 repository 与 Package selector；version range 用于选择对应 GitHub Release。

## 当前方向

- 默认从 GitHub Release 获取；
- 找不到 Release 时不自动切换获取方式；
- Git clone 只有在显式 Git 模式下使用；
- transitive dependency 必须精确到 `owner/repo/package`；
- 未来如果建立 AKM Registry，再增加独立的 Registry source model；
- 软件依赖不由 AKM 的 Provider 自动处理：AKM只做常见软件的基础探测，复杂依赖由 `DEPENDENCIES.md` 交给 Agent 检查并与用户确认后处理。
