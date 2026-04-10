---
name: wechat-mp-publisher
version: 2.0.2
description: 远程微信公众号发布技能 (合规优化版)。通过 HTTP MCP 解决家用宽带 IP 变动问题，支持安全凭证隔离与依赖检查。
homepage: https://github.com/caol64/wenyan-mcp
metadata:
  openclaw:
    emoji: "🚀"
    category: publishing
  clawdbot:
    emoji: "🚀"
    requires:
      bins: ["mcporter", "curl", "jq"]
    install:
      - id: "node"
        kind: "node"
        package: "mcporter"
        bins: ["mcporter"]
        label: "安装 MCP 客户端 (mcporter)"
---

# 微信公众号远程发布 (Remote Publisher - Compliance Optimized)

**核心痛点解决**：家用宽带 IP 频繁变动，无法固定添加到公众号白名单？
本技能通过远程 `wenyan-mcp` 服务中转，让你的本地 OpenClaw 也能稳定发布文章，无需本地 IP 权限！

## 🌟 架构优势

- **IP 漫游无忧**：仅需将远程 MCP 服务器 IP 加入白名单，无论你在家里、咖啡厅还是 4G 热点，都能随时发布。
- **合规隔离**：凭证与系统配置分离，避免污染全局 `TOOLS.md`。
- **依赖自检**：脚本运行时自动检查 `jq`、`mcporter` 和 `wenyan-cli`。
- **灵活配置**：支持自定义 MCP 配置文件路径。

## ⚙️ 快速配置

### 1. 准备凭证 (wechat.env)

在技能根目录下复制 `wechat.env.example` 为 `wechat.env` 并填入公众号凭证：

```bash
cp wechat.env.example wechat.env
nano wechat.env
```

内容示例：
```bash
export WECHAT_APP_ID="wx..."
export WECHAT_APP_SECRET="cx..."
# Optional: 指定 MCP 配置文件 (默认 $HOME/.openclaw/mcp.json)
# export MCP_CONFIG_FILE="/path/to/your/mcp.json"
```

### 2. 连接远程服务 (mcp.json)

确保你的 `mcp.json` 指向远程 MCP 实例：

```json
{
  "mcpServers": {
    "wenyan-mcp": {
      "name": "公众号远程助手",
      "transport": "sse",
      "url": "http://<your-remote-server-ip>:3000/sse",
      "headers": {
        "X-API-Key": "<optional-api-key>"
      }
    }
  }
}
```

## 🚀 使用指南

### 方式 A: 智能助手 (推荐)

直接对我说：
> "帮我把 `path/to/article.md` 发布到公众号，使用默认主题。"

我会自动：
1. 读取 `wechat.env` 获取凭证
2. 检查本地环境 (`mcporter`, `jq`)
3. 调用远程 MCP 完成发布

### 方式 B: 命令行脚本 (高级)

我们提供了封装好的脚本 `scripts/publish-remote.sh`，体验与本地 CLI 一致：

```bash
# 赋予执行权限
chmod +x scripts/publish-remote.sh

# 发布文章 (自动加载 wechat.env)
./scripts/publish-remote.sh ./my-post.md

# 指定主题 (lapis)
./scripts/publish-remote.sh ./my-post.md lapis
```

## 📝 Markdown 规范

与标准 wenyan-cli 一致，头部必须包含元数据：

```markdown
---
title: 我的精彩文章
cover: https://example.com/cover.jpg
---

# 正文开始
...
```

*提示：`cover` 推荐使用图床链接，以确保远程服务器能正确下载封面。*

## 🛠️ 故障排查

| 现象 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| **Dependencies Missing** | 缺少 `jq` 或 `mcporter` | 请确保系统已安装这些工具 |
| **Config Not Found** | 未找到 `wechat.env` | 请按照步骤 1 创建并配置 |
| **IP not in whitelist** | 远程服务器 IP 未加白 | 登录公众号后台 -> 基本配置 -> IP 白名单，添加 **MCP 服务器的公网 IP** |
| **wenyan-cli 静默退出（EXIT:0无输出）** | Node v23.x 与 wenyan-cli v2.x 不兼容 | 参见下方「Node 版本要求」章节 |
| **css.replace is not a function** | Node v23.x 与 wenyan-cli v1.x CSS 模块不兼容 | 参见下方「Node 版本要求」章节 |
| **invalid appsecret (40125)** | appSecret 错误或 Node 环境导致加解密异常 | 检查 wechat.env 凭证；或使用「直接 API 方案」绕过 |

## 📌 Node 版本要求

wenyan-cli 对 Node 版本有严格要求：

| wenyan-cli 版本 | 要求的 Node 版本 | 当前环境兼容情况 |
| :--- | :--- | :--- |
| **v2.0.x** | `^20.19.0 \|\| ^22.12.0 \|\| >=24.0.0` | ❌ Node v23.x 不支持 |
| **v1.0.x** | 较宽松，但存在 CSS 模块兼容问题 | ⚠️ Node v23.x 下报 `css.replace` 错误 |

**当前环境 Node v23.6.0**：建议使用「直接 API 方案」（见下），或升级/降级 Node 至 v20/v22。

## 🔧 直接 API 方案（wenyan-cli 不可用时的备选）

当 wenyan-cli 因 Node 版本问题无法工作时，或需要更精细控制时，可直接调微信 API：

```bash
# 1. 获取 access_token（IP 需在白名单）
TOKEN=$(curl -s "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 2. 上传封面图 → 获取 thumb_media_id
COVER_RESP=$(curl -s -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
  -F "media=@cover.png;type=image/png")
THUMB_MEDIA_ID=$(echo $COVER_RESP | python3 -c "import sys,json; print(json.load(sys.stdin)['media_id'])")

# 3. 上传正文图片（每张图都要！）→ 获取 mmbiz URL
IMG_RESP=$(curl -s -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
  -F "media=@fig_xxx.png;type=image/png")
MMBIZ_URL=$(echo $IMG_RESP | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

# 4. 调用 draft/add 上传草稿
curl -s -X POST "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"articles":[{"title":"标题","author":"作者","digest":"摘要","content":"<p><img src=\"MMBIZ_URL\"/></p>","thumb_media_id":"THUMB_MEDIA_ID","need_open_comment":0,"only_fans_can_comment":0}]}'
```

**⚠️ 重要规范（踩坑总结）**：
1. **正文图片必须用微信素材库地址（mmbiz）**：调用 `material/add_material` 上传后返回的 `url` 字段就是 `mmbiz.qpic.cn` 地址，**不能用** img402.dev 等外部图床链接（微信富文本不展示外部图片）
2. **封面 thumb_media_id 必须来自本次上传**：每次发布都要重新上传封面图获取新 media_id，不能用之前其他论文的封面 ID
3. **所有 markdown 格式必须转 HTML**：`**xxx**` → `<strong>xxx</strong>`，`---` → `<hr/>`，不要裸传 markdown 语法

## 📋 凭证文件格式（重要！）

wenyan-cli 使用的凭证文件位于 `~/.config/wenyan-md/credential.json`，**必须**包含 `wechat` 外层包装：

```json
{
  "wechat": {
    "wx1767b589cc50950f": {
      "appId": "wx1767b589cc50950f",
      "appSecret": "your_app_secret_here"
    }
  }
}
```

❌ 错误格式（缺少 `wechat` 外层）：
```json
{
  "wx1767b589cc50950f": { ... }
}
```
