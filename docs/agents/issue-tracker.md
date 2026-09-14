# Issue tracker: GitHub

Akira sKill Manager 的 Spec、Ticket、Wayfinder 决策项与外部请求使用 GitHub Issues。仓库内执行 `gh` 时由当前 `origin` 自动解析目标仓库。

## 基本操作

- 创建 Issue：`gh issue create --title "..." --body "..."`
- 读取 Issue：`gh issue view <number> --comments`
- 列出 Issue：`gh issue list --state open`
- 评论：`gh issue comment <number> --body "..."`
- 修改标签：`gh issue edit <number> --add-label "..."`
- 关闭：`gh issue close <number> --comment "..."`

PR 不作为默认 triage request surface。

## Matt 工作流

当 Matt Skill 要求发布 Spec、Ticket 或决策项到 issue tracker 时，创建 GitHub Issue。工作项之间优先使用 GitHub 原生 sub-issue 与 dependency 关系；workflow role 与具体 label 的映射见 `docs/agents/triage-labels.md`。
