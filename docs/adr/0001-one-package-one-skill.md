# ADR 0001：一个 Package 只发行一个 Skill

- 状态：Proposed
- 日期：2026-09-14

> 当前协议尚未定稿。该提案仍需结合 Skill 文件结构、Package 粒度与真实发布场景继续验证；不得作为实现前提。

## 背景

一个 Git repository 可以同时维护大量 Skill，但项目安装某个 Router Skill 时不能因为“同仓”把不相关 Skill 一并安装。若 Package 继续以 repository 为单位，依赖解析、版本发布和最小安装集合都会重新耦合到源码仓布局。

## 决定

AKM 的一个 Package 恰好发行一个 Agent Skill。

Package Identity 使用 `namespace/skill-name` 形式，例如 `akira/ask-matt`；其中末段 `skill-name` 必须与 Package 根目录 `SKILL.md` 的 `name` 一致。

Git repository 只是源码与发布 artifact 的一种承载位置，不是安装单位。一个 repository 可以独立发布任意数量的 Package。

## 结果

- 安装 `akira/ask-matt` 只会带入它的 dependency closure；
- 同仓其他 Skill 不会因 repository 粒度被隐式安装；
- 每个 Package 可以独立版本化和发布；
- Package Store 中每个实体本身就是可直接激活的标准 Skill directory；
- 如果未来确实出现“必须原子发行多个 Skill”的需求，应通过一个显式 Router/Meta Package 依赖多个 Package 表达，而不是扩大 Package 粒度。
