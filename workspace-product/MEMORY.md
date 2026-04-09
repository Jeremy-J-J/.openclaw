# MEMORY.md - 长期记忆

## 最佳实践

### Chrome MCP 微信公众号后台账号切换
- **难点**：左下角账号名称（"希望之雪"等）在 accessibility tree 中显示为纯文本 statictext，没有可点击的 ref，普通 click 无法触发
- **解决方案**：用 JS evaluate 遍历所有元素，找到 textContent 匹配目标账号名的元素，调用 `.click()`
- **完整流程**：
  1. 打开公众号后台 `https://mp.weixin.qq.com/cgi-bin/home?t=home/index&lang=zh_CN&token=691930933`
  2. `browser(action=act, targetId=tabId, request={kind:"evaluate", fn:"() => { const el = Array.from(document.querySelectorAll('*')).find(e => e.textContent.trim() === '当前显示的账号名' && e.children.length === 0); if (el) el.click(); }"})` 点击账号文本 → 弹出账号菜单（显示账号详情、切换账号等）
  3. 找到「切换账号」link ref（如 `ref=22_10`），`browser(action=act, targetId=tabId, ref="22_10", kind="click")` → 弹出账号列表
  4. `browser(action=act, targetId=tabId, request={kind:"evaluate", fn:"() => { const el = Array.from(document.querySelectorAll('*')).find(e => e.textContent.trim() === '目标账号名' && e.children.length === 0); if (el) el.click(); }"})` 点击目标账号 → 完成切换
- **关键技巧**：
  - 用 `request={kind:"evaluate", fn:"() => { ... }"}` 传参（不是直接传 `fn`）
  - 每次 snapshot 后 ref 会变化（递增），需重新获取最新 ref
  - 公众号后台 token 固定：`token=691930933`

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

### Chrome MCP 微信公众号转载（2026-04-07 优化版）
- **场景**：在公众号后台搜索原创文章、筛选高质量内容并转载
- **转载搜索页 URL**：`https://mp.weixin.qq.com/cgi-bin/appmsg?t=media/appmsg_edit_v2&action=edit&isNew=1&type=77&share=1&token=1702686186&lang=zh_CN`
- **★★★ 核心原则：一个 tab 全流程★★★**
  - 从搜索到保存，**全在同一个 tab 完成**，绝对不要开新 tab
  - 点击「转载」后页面跳转到编辑预览态，tab targetId 不变，但旧 tab context 会话丢失
  - 保存操作必须在当前 tab 执行，不能切换 tab
- **搜索流程**：输入关键词 → 回车 → 等待结果
- **筛选标准**：
  - **快捷转载**（可直接转载）vs **分享**（需跳转，不适合）
  - 快捷转载文章特征：有"快捷转载"和"不可修改，显示转载来源"字样
- **选中文章**：通过 JS evaluate 找到文章对应的 radio 单选框并点击
- **勾选条款**：JS `document.querySelector('input[type="checkbox"]').click()`
- **提交转载**：JS 查找并点击「转载」按钮 → 页面跳转到编辑预览页
- **★★★ 保存草稿的正确方式 ★★★**：
  1. 转载后**等待**页面完全跳转到编辑预览态（能看到正文内容 + 「保存为草稿」按钮）
  2. 用 JS evaluate 查找「保存为草稿」按钮：`Array.from(document.querySelectorAll('button')).find(b => b.textContent.includes('保存为草稿'))`
  3. **不要用 ref 点击**，ref 在页面跳转后会失效
  4. 点击后页面顶部出现「04-07 xx:xx / 操作人 / 网页版 / 手动保存」即为成功
- **★★★ tab 会话丢失的处理 ★★★**：
  - 如果 tab 丢了（targetId invalid），用 `browser(action=tabs)` 找到编辑页 tab
  - 如果编辑页不在 tabs 里，文章可能没保存成功，需要重新走完整流程
- **已创建自定义 Skill**：`skills/wechat-republish/SKILL.md`（2026-04-07 已更新优化版）

### 工具选择
- **Chrome MCP（browser 工具）**：需要操作用户已登录状态、保留 cookies/session 的场景
- **agent-browser CLI**：独立 session、多 session 隔离、自动化流程

### Paper-Parse 论文解析最佳实践（2026-04-09 总结）
- **★★★ 图表必须提取原生图片，不可用整页截图 ★★★**：用 PyMuPDF(fitz) 从 PDF 中提取 `page.get_images()` 原生图片，而非用 `get_pixmap()` 截整页
- **★★★ 图片缩放：宽度 > 900px 必须缩放 ★★★**：用 macOS 内置 `sips -Z 900` 命令；可一次处理多张：
  ```bash
  for f in charts/*.png; do
    w=$(sips -g pixelWidth "$f" | grep pixelWidth | awk '{print $2}')
    [ "$w" -gt 900 ] && sips -Z 900 "$f" --out "${f%.png}_resized.png"
  done
  ```
- **★★★ 图床：使用 img402.dev ★★★**：curl 上传，免费无需认证，返回 `https://i.img402.dev/xxx.png` URL；保留7天
  ```bash
  curl -s -X POST https://img402.dev/api/free -F "image=@charts/xxx.png" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d['url'])"
  ```
- **★★★ Markdown 图片居中显示写法 ★★★**：必须用 HTML 格式，不能用纯 Markdown：
  ```html
  <p align="center">
  <img src="https://i.img402.dev/xxx.png" width="900"/>
  </p>
  <p align="center">图1：图片标题（来源：原论文 [图1号]）</p>
  ```
- **★★★ 图片标题格式 ★★★**：不加粗、不加星号，纯文本；不写"已缩放至XXXpx"等处理细节
- **LaTeX 公式**：使用 `$$公式$$` 格式；需目标渲染器支持 MathJax/KaTeX；GitHub/GitLab 不支持
- **全文提取**：优先 `pdftotext`；若不可用，用 PyMuPDF：`for page in doc: text += page.get_text()`
- **相关 Skill**：`skills/paper-parse/SKILL.md`（2026-04-09 已同步更新）
