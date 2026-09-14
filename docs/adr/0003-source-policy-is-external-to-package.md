# ADR 0003：Package 依赖声明与 Source Policy 分离

- 状态：Accepted
- 日期：2026-09-14

## 背景

如果 transitive dependency 可以自行指定新的外部来源，那么安装一个 Package 时，实际参与解析的来源集合会在依赖图展开过程中不断变化，用户也无法在执行前看到稳定、完整的计划。

## 决定

Package Manifest 中的 Skill dependency 只声明 `Package Identity -> version range`，不在依赖边中指定下载来源。

Package Identity 的可用版本与 Release Artifact 定位由 Package Index 提供；Git、path 或其他非默认来源只能由 Project Manifest 的显式 source override 或更高层项目策略引入。

Package Manifest 可以保存 repository 等 provenance 信息，但 provenance 只描述“这个包来自哪里”，不代表项目已经批准该来源。

软件依赖采用同样原则：Package 只声明 AKM Software Catalog 中的软件能力与版本要求；具体平台如何满足该要求由 AKM Provider 决定。

## 结果

- resolver 可以在执行前枚举完整 source 集合；
- transitive dependency 不会隐式改变项目来源策略；
- 第三方 Git Skill 仍然可用，但必须成为项目的显式 source override；
- 软件依赖声明保持跨平台，平台差异集中在 Provider。
