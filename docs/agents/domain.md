# Domain Docs

Skiloom 当前采用 single-context domain model。

## 开始设计或实现前

- 先读取根目录 `CONTEXT.md`，使用其中已经定义且仍有效的领域词汇。
- 涉及架构决定时读取相关 `docs/adr/`；`Proposed` 仅代表当前提案，只有 `Accepted` 才是已确认决定。
- 若当前讨论暴露出新的概念、基数关系或术语冲突，使用 `domain-modeling` 更新领域语言，而不是把临时实现细节塞进 `CONTEXT.md`。

## 当前特别注意

Package 与 Skill 的基数、Package Snapshot、Project Intent / Confirmed Resolution、Core conformance 与公开 `skiloom` namespace 已经由 Accepted ADR 固定。设计或实现必须以当前 Accepted design/ADR 为准；Rejected ADR 与 pre-standard `AKM / akm` namespace 只提供历史上下文，不能重新成为实现 authority。
