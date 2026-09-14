# Akira sKill Manager

Akira sKill Manager（AKM）是一个面向 Agent Skill 的独立包管理器项目。它把 Skill 作为真正的软件包处理，而不是把 Git repository 当作安装单位。

当前阶段优先固定领域模型和协议，不先实现 CLI。首批设计覆盖：

- Package Manifest；
- Release Artifact；
- Project Manifest 与 Lock；
- Dependency Resolver 与 Install Plan；
- 外部软件依赖与 Provider 模型。

核心目标：一个项目只声明真正需要的顶层 Skill；解析器计算完整依赖闭包；包内容机器级共享；项目通过软链接形成自己的 Skill 视图；所有外部来源和软件安装在执行前形成可审计计划并由用户授权。

当前设计入口：

- [`CONTEXT.md`](CONTEXT.md)：领域词汇；
- [`docs/research/package-management-prior-art.md`](docs/research/package-management-prior-art.md)：一手规范调研；
- `docs/design/`：协议设计；
- `docs/adr/`：少量难以逆转的架构决定。
