# Dependencies

## Special requirements

### GitHub authentication
- Requirement: 如果当前工作流需要访问私有 repository，`gh` 或 Git credential 必须已经具备相应访问权限。
- Check: 使用只读命令确认当前身份和目标 repository 可访问。
- Resolution: 如果权限不足，先向用户说明缺少什么访问能力；任何登录、凭据写入或权限变更都必须先取得用户明确同意。

### Executor discovery
- Requirement: 当前执行器必须能够从项目 `.agents/skills/` 发现已激活的 `ask-matt` Skill Root。
- Check: 验证 `.agents/skills/<activation-name>/SKILL.md` 对当前执行器可见且 Skill metadata 合法。
- Resolution: 如果目标 activation name 冲突，AKM 应提示用户为新安装项 rename 或放弃；不得静默覆盖已有 Skill。

## Agent procedure

1. 在依赖状态未知或已失效时读取本文件。
2. 若存在 `.agents/.akm/dependencies.lock`，读取当前 Package 的检查状态。
3. 检查每一项 Special requirement。
4. 将检查结果写入 `.agents/.akm/dependencies.lock`，不要修改本文件。
5. 若有缺失或不兼容，明确说明当前状态、缺口与影响。
6. 提出具体解决方案及其副作用。
7. 在安装、升级、登录、下载、修改配置、启停服务或其他环境变更前取得用户明确同意。
8. 处理后重新检查，并更新 `.agents/.akm/dependencies.lock`。
