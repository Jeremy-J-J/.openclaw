# MEMORY.md - 长期记忆

## 最佳实践

### Chrome MCP 微信公众号后台账号切换
- **难点**：左下角账号名称（"希望之雪"等）在 accessibility tree 中显示为纯文本 statictext，没有可点击的 ref，普通 click 无法触发
- **解决方案**：用 JS evaluate 遍历所有元素，找到 textContent 匹配目标账号名的元素，调用 `.click()`
- **完整流程**：
  1. `browser(action=act, targetId=tabId, kind="evaluate", fn="...textContent 希望之雪...")` 点击账号文本 → 弹出账号菜单
  2. 找到「切换账号」link ref → click → 弹出账号列表
  3. JS evaluate 点击目标账号（如"极智视界"）→ 完成切换
- **注意**：每次 snapshot 后 ref 会变化（递增），操作前需重新 snapshot 获取最新 ref

### Chrome MCP 浏览器操作
- **使用场景**：需要操作用户已登录的 Chrome 浏览器（如抓取微博热搜等需要登录态的操作）
- **操作流程**：
  1. 先调用 `browser(action=status, profile="user")` 确认连接状态
  2. 确认 `transport: "chrome-mcp"` 且 `running: true` 后再执行后续操作
  3. 使用 `browser(action=open, profile="user", url="<url>")` 打开目标页面
  4. 用 `browser(action=snapshot, profile="user", targetId="<tabId>")` 获取页面快照
- **关键**：必须用 `profile="user"` 才能操作用户已登录的 Chrome session
- **重要经验**：
  - `status` 超时不代表浏览器不可用，**直接尝试 open/snapshot 往往能成功**
  - 微博热搜地址：`https://weibo.com/hot/search`（需登录态，Chrome MCP 是最稳定方案）
  - 若 status 持续超时，检查 Chrome 扩展是否安装并启用、远程调试端口是否开启

### Chrome MCP 微博操作
- **热搜榜入口**：`https://weibo.com/hot/search`
- **点击热搜词**：从列表页点击后会打开新标签页，通过 `tabs` 接口找到新标签页 targetId 再 snapshot
- **转发微博流程**：
  1. 进入话题页 → 点击「分享」按钮 → 出现分享弹窗
  2. 弹窗中 `textbox` 输入转发语 → 点击「发布」
  3. 发布后弹窗关闭，回到话题页（注意确认发布是否成功）
- **关键 refs**：`发微博` 按钮 → 打开发布框；`分享` 按钮（话题页）→ 打开分享弹窗
- **注意**：微博页面交互尽量用 `act` 而非 `click`，部分按钮点击需用 `act` 触发

### 工具选择
- **Chrome MCP（browser 工具）**：需要操作用户已登录状态、保留 cookies/session 的场景
- **agent-browser CLI**：独立 session、多 session 隔离、自动化流程
