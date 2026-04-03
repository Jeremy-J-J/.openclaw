# MEMORY.md - 长期记忆

## 最佳实践

### Chrome MCP 浏览器操作
- **使用场景**：需要操作用户已登录的 Chrome 浏览器（如抓取微博热搜等需要登录态的操作）
- **操作流程**：
  1. 先调用 `browser(action=status, profile="user")` 确认连接状态
  2. 确认 `transport: "chrome-mcp"` 且 `running: true` 后再执行后续操作
  3. 使用 `browser(action=open, profile="user", url="<url>")` 打开目标页面
  4. 用 `browser(action=snapshot, profile="user", targetId="<tabId>")` 获取页面快照
- **关键**：必须用 `profile="user"` 才能操作用户已登录的 Chrome session
- **常见问题**：若 status 超时需重启 OpenClaw gateway

### 工具选择
- **Chrome MCP（browser 工具）**：需要操作用户已登录状态、保留 cookies/session 的场景
- **agent-browser CLI**：独立 session、多 session 隔离、自动化流程
