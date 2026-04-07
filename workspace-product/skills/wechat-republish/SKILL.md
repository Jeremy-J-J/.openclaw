---
name: wechat-republish
description: "微信公众号文章检索、筛选与转载。当需要从微信公众号搜索文章、筛选高质量内容并转载时使用此技能。"
metadata: { "openclaw": { "emoji": "📋" } }
---

# 微信公众号文章转载

通过 Chrome MCP 操控已登录的微信公众号后台，搜索原创文章、筛选高质量内容并完成转载。

## 何时使用

- "帮我转载公众号文章"
- "搜索 xxx 相关文章并转载"
- "找一篇高质量的公众号文章转载"
- "公众号内容转载"

## 前置准备

1. 确认 Chrome MCP 可用：`browser(action=status, profile="user")`
2. 确保 Chrome 已安装 OpenClaw 扩展并开启远程调试

## 标准流程（2026-04-07 优化版）

### 第一步：打开转载页

```
URL: https://mp.weixin.qq.com/cgi-bin/appmsg?t=media/appmsg_edit_v2&action=edit&isNew=1&type=77&share=1&token=<TOKEN>&lang=zh_CN
```

token 从已登录的公众号后台 URL 中获取。**用同一个 tab 完成全流程，不要开新 tab。**

### 第二步：搜索关键词

1. `snapshot` 获取搜索框 ref（通常为 `*_39`）
2. `browser(act, type, ref="*_39", text="关键词")`
3. `browser(act, press, ref="*_39", key="Enter")`

### 第三步：识别快捷转载文章

搜索结果判断标准：
- **快捷转载**：正文直接展示，不可修改原文 → ✅ 可直接转载
- **分享**：需跳转原文阅读 → ❌ 不适合

快捷转载特征：有"快捷转载"+"不可修改，显示转载来源"字样

### 第四步：评估文章质量（可选）

建议直接选快捷转载文章，质量有保障。如需进一步评估，用 JS 提取链接后打开查看：

```javascript
browser(act, evaluate, fn: "() => { const links = document.querySelectorAll('a'); let target = null; links.forEach(a => { if (a.textContent.trim() === '文章标题') target = a; }); return target ? target.href : 'not found'; }")
```

质量标准：篇幅长、无广告、无二维码、无商品售卖

### 第五步：执行转载（关键！）

**所有操作在当前 tab 完成，不要切换 tab。**

**5.1 点击单选框选中文章**
```javascript
browser(act, evaluate, fn: "() => { const links = document.querySelectorAll('a'); let targetLink = null; links.forEach(a => { if (a.textContent.trim() === '目标文章标题') targetLink = a; }); if (!targetLink) return 'link not found'; let row = targetLink.closest('ul') || targetLink.parentElement.parentElement; let radio = row ? row.querySelector('input[type=\"radio\"]') : null; if (!radio) { const parent = targetLink.parentElement; const radios = parent.querySelectorAll('input[type=\"radio\"]'); if (radios.length > 0) radio = radios[0]; } if (radio) { radio.click(); return 'clicked'; } return 'radio not found'; }")
```

**5.2 勾选同意条款**
```javascript
browser(act, evaluate, fn: "() => { const cb = document.querySelector('input[type=\"checkbox\"]'); if (cb) { cb.click(); return 'checked'; } return 'not found'; }")
```

**5.3 点击「转载」按钮**
```javascript
browser(act, evaluate, fn: "() => { const btns = document.querySelectorAll('button'); for (const b of btns) { if (b.textContent.trim() === '转载') { b.click(); return 'clicked'; } } return 'not found'; }")
```

**5.4 ⭐ 等待页面跳转到编辑预览态（关键！）**

点击转载后，页面会跳转到编辑预览页，显示文章内容。此时**不要做任何操作**，等 snapshot 显示以下内容后再继续：
- 标题出现在页面中
- 看到「当前文章通过快捷转载方式添加，不支持修改」
- 看到「保存为草稿」和「发表」按钮

**5.5 点击「保存为草稿」（用 JS，不要用 ref）**
```javascript
browser(act, evaluate, fn: "() => { const btns = document.querySelectorAll('button'); for (const b of btns) { if (b.textContent.includes('保存为草稿')) { b.click(); return 'clicked'; } } return 'not found'; }")
```

**5.6 确认保存成功**
- snapshot 中出现「已保存」或「操作时间 + 手动保存」→ 成功
- 页面 state 仍为当前 tab，**不要关闭或切换 tab**

## 常见问题排查

### Q: 点击「保存为草稿」后提示 not found？
**A**: 页面还没完全加载完，等待一下再执行 JS。如果持续找不到，用 `snapshot` 确认页面状态。

### Q: 如何确认草稿保存成功了？
**A**: 页面顶部出现「04-07 xx:xx / JeremyJ / 网页版 / 手动保存」记录，且页面显示「已保存」，即成功。

### Q: 为什么文章没有出现在草稿箱？
**A**: 可能是点击「转载」后页面状态切换了，但保存操作没有在正确的页面 context 下执行。确保在编辑预览页（能看到正文内容）再点击保存。

### Q: tab 找不到了怎么办？
**A**: 用 `browser(action=tabs)` 列出所有 tab，找到公众号编辑页的 targetId。如果全丢了，重新 open 转载页，从搜索结果页重新走流程。

## 核心原则

1. **一个 tab 全流程**：从搜索到保存，全在同一个 tab 完成，不要开新 tab
2. **JS 优先**：所有按钮点击都用 JS evaluate，避免 ref 失效问题
3. **等页面稳定**：转载后等页面完全跳转到编辑预览态再保存
4. **保存后验证**：确认「已保存」或操作记录出现才算成功

## 注意事项

- 仅转载**原创**文章
- 快捷转载**不可修改**正文
- 转载前确认无广告/二维码/商品
- 建议先存草稿，确认后再发表
