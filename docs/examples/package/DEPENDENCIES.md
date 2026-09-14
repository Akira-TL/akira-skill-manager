# Dependency Check

## Special requirements

### GitHub authentication
- Requirement: 如果当前工作流需要访问私有 repository，`gh` 或 Git credential 必须已经具备相应访问权限。
- Check: 使用只读命令确认当前身份和目标 repository 可访问。
- Resolution: 如果权限不足，先向用户说明缺少什么访问能力；任何登录、凭据写入或权限变更都必须先取得用户明确同意。

### Executor discovery
- Requirement: 当前执行器必须能够发现 AKM Project Skill Library 中的 `ask-matt` Skill Root。
- Check: 根据当前执行器的 Skill discovery 机制验证项目级可见性。
- Resolution: 如果执行器需要额外链接或索引，由 executor adapter 或 Agent 提出具体变更并先说明影响。

## Current status

<!-- akm-status:start -->
- git >=2.40: unchecked
- gh >=2.45: unchecked
<!-- akm-status:end -->

## Agent procedure

1. 先读取 AKM 生成的 Current status。
2. 检查每一项 Special requirement。
3. 若全部满足，继续使用 Skill。
4. 若有缺失或不兼容，明确说明当前状态、缺口与影响。
5. 提出具体解决方案及其副作用。
6. 在安装、升级、登录、下载、修改配置、启停服务或其他环境变更前取得用户明确同意。
7. 处理后重新检查，并更新 Current status。
