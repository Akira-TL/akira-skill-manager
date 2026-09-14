# Dependencies

## Special requirements

### GitHub authentication
- Requirement: 如果当前工作流需要访问私有 repository，`gh` 或 Git credential 必须已经具备相应访问权限。
- Check: 使用只读命令确认当前身份和目标 repository 可访问。
- Resolution: 如果权限不足，先向用户说明缺少什么访问能力；任何登录、凭据写入或权限变更都必须先取得用户明确同意。

### Executor discovery
- Requirement: 当前执行器必须能够发现 AKM Project Skill Library 中的 `ask-matt` Skill Root。
- Check: 根据当前执行器的 Skill discovery 机制验证项目级可见性。
- Resolution: 如果执行器需要额外链接或索引，由 executor adapter 或 Agent 提出具体变更并先说明影响。

## Agent procedure

1. 在依赖状态未知或已失效时读取本文件。
2. 若存在 `.akm/dependencies.lock`，读取当前 Package 的检查状态。
3. 检查每一项 Special requirement。
4. 将检查结果写入 `.akm/dependencies.lock`，不要修改本文件。
5. 若有缺失或不兼容，明确说明当前状态、缺口与影响。
6. 提出具体解决方案及其副作用。
7. 在安装、升级、登录、下载、修改配置、启停服务或其他环境变更前取得用户明确同意。
8. 处理后重新检查，并更新 `.akm/dependencies.lock`。
