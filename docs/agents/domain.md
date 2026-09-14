# Domain Docs

Akira sKill Manager 当前采用 single-context domain model。

## 开始设计或实现前

- 先读取根目录 `CONTEXT.md`，使用其中已经定义且仍有效的领域词汇。
- 涉及架构决定时读取相关 `docs/adr/`；`Proposed` 仅代表当前提案，只有 `Accepted` 才是已确认决定。
- 若当前讨论暴露出新的概念、基数关系或术语冲突，使用 `domain-modeling` 更新领域语言，而不是把临时实现细节塞进 `CONTEXT.md`。

## 当前特别注意

Package 与 Skill 的基数关系、Skill 包目录结构、AKM 扩展元数据位置仍未最终确定。任何设计或实现不得仅因为旧文档曾写过某个方案就把它视为既定事实。
