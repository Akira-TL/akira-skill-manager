# ADR 0002：同一项目中一个 Package Identity 只解析一个版本

- 状态：Accepted
- 日期：2026-09-14

## 背景

传统语言包管理器有时允许同一依赖的多个版本同时存在，例如通过嵌套模块目录隔离。但 Agent Skill 在项目中最终以 Skill 名称进入一个项目级激活视图；两个版本如果暴露同一个 `SKILL.md` name，无法在该视图中同时拥有无歧义的名字。

## 决定

一个 Project Environment 的 Resolved Graph 中，同一 Package Identity 最多存在一个精确版本。

若两个依赖链要求互不相交的版本范围，resolver 必须报告版本冲突，而不是偷偷安装两个版本。

机器级 Package Store 仍允许同时保存任意数量的版本和内容；不同项目可以锁定同一 Package 的不同版本。

此外，不同 Package Identity 如果最终导出相同 Skill 名称，也视为激活冲突并 fail closed。

## 结果

- 项目激活结果确定且无名称歧义；
- resolver 可以采用天然支持“每包单版本”的 PubGrub 类模型；
- 冲突必须通过升级、降级或修改依赖约束解决，不能靠隐藏的重复版本绕过；
- 跨项目版本并存不受影响。
