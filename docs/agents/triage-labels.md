# Workflow Role Mapping

本项目使用 Matt Engineering 的 canonical workflow role 名称作为 GitHub label。

| Workflow role | GitHub label | Used by |
| --- | --- | --- |
| `ready-for-agent` | `ready-for-agent` | `to-tickets`, `triage` |
| `bug` | `bug` | `triage` |
| `enhancement` | `enhancement` | `triage` |
| `needs-triage` | `needs-triage` | `triage` |
| `needs-info` | `needs-info` | `triage` |
| `ready-for-human` | `ready-for-human` | `triage` |
| `wontfix` | `wontfix` | `triage` |
| `wayfinder:map` | `wayfinder:map` | `wayfinder` |
| `wayfinder:research` | `wayfinder:research` | `wayfinder` |
| `wayfinder:prototype` | `wayfinder:prototype` | `wayfinder` |
| `wayfinder:grilling` | `wayfinder:grilling` | `wayfinder` |
| `wayfinder:task` | `wayfinder:task` | `wayfinder` |

下游 Skill 应读取这里的映射，不自行发明替代 label。项目初始化只创建缺失 label，不覆盖已有 label 的颜色或描述。
