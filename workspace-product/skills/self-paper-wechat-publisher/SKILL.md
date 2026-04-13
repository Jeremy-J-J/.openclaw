---
name: self-paper-wechat-publisher
description: "将学术论文研读并发布到微信公众号草稿箱的完整流水线。当用户提供论文URL并要求发布到公众号时使用。"
metadata: {"openclaw":{"emoji":"📤"}}
---

# Skill: 论文研读与公众号发布流水线

将学术论文研读报告一键发布到微信公众号草稿箱的完整解决方案。

## 何时使用

- "帮我解读这篇论文并发布到公众号"
- "研读论文后上传到微信公众号草稿箱"
- "论文研读 + 公众号发布"
- "把 xxx 论文发布到公众号"
- 用户提供论文 arXiv/PDF 链接并要求上传到公众号

## 前置条件

- IP 白名单：当前设备公网 IP 已在微信公众号后台添加
- 凭证就绪：`skills/wechat-mp-publisher/wechat.env` 已配置 APP_ID 和 APP_SECRET
- 工具可用：Python 3、curl、PyMuPDF

## 执行清单

### Step 1: 论文研读（调用 paper-parse 技能）

1. 读取 `skills/paper-parse/SKILL.md`
2. 按流程执行：PDF 下载 → PyMuPDF 提取文字 + 图片 → 双模报告撰写
3. 产出：`research-papers/{论文简称}_研读报告.md`
4. **修改文章标题**：不用"xxx研读报告"等直白字样，起一个吸引人的标题
5. 研读完成后进入 Step 2

### Step 2: 准备微信公众号上传

#### 2.1 获取 access_token

```bash
source skills/wechat-mp-publisher/wechat.env
curl -s "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=${WECHAT_APP_ID}&secret=${WECHAT_APP_SECRET}"
```

#### 2.2 上传封面图（获取 thumb_media_id）

封面图使用论文的第一张图：
```bash
TOKEN="<上面获取的token>"
curl -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
  -F "media=@charts/fig_p1_xref155.png;type=image/png"
```
→ 响应中的 `media_id` 即为 `thumb_media_id`

#### 2.3 上传正文中所有图片（获取 mmbiz URL）

**重要**：正文中的图片必须先上传到微信素材库，获取 `mmbiz.qpic.cn` 地址后才能正确显示。

**★★★ 上传后必须验证 URL 是否正确 ★★★**：如果某张图片在公众号中不显示（空白或微信默认图标），很可能是该图片对应的 URL 有问题。将显示异常的图重新上传（用同一张原文件），获取新 URL 替换研读报告中对应图片的 URL。

```bash
for f in charts/fig_*.png charts/fig_*.jpeg; do
  TYPE=$(echo $f | grep -q png && echo "image/png" || echo "image/jpeg")
  RESP=$(curl -s -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
    -F "media=@$f;type=$TYPE")
  URL=$(echo $RESP | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
  echo "$f: $URL"
done
```

### Step 3: 核对PDF图片与论文图号的对应关系

**★★★ 关键警告：PDF提取的图片顺序 ≠ 论文中的Figure编号 ★★★**
PyMuPDF 按页码顺序提取图片，但论文中引用的 Figure 编号不一定对应文件的页码位置。**必须对照原论文确认每个位置对应的真实图号**，避免图号与图片内容不符。核对方法：阅读论文原文找到每个图号所在页面，对照 PyMuPDF 提取出来的图片确认实际内容。

### Step 4: 在研读报告中插入图片

**关键**：PDF 提取只能得到图片文件，不会自动插入正文。需要在适当位置手动插入。

在 `research-papers/{论文简称}_研读报告.md` 中，在章节标题后、表格前后等适当位置插入 Markdown 图片语法：

```
![图1：图片说明（来源：原论文 Figure X）](charts_resized/fig_xxx.png)
```

**注意**：在 Markdown 文件中写 `![alt](path)` 格式，**不要**写 HTML 标签。转换脚本会在 Step 5 中自动将 Markdown 图片语法替换为微信兼容的 HTML img 标签。

**图片标题格式**：`图X：描述（来源：原论文 [图X号]）`，纯文本、不加粗、不加星号。

### Step 5: 转换研读报告（Markdown → HTML）

#### 5.1 预处理

1. **替换图片 URL**：将研读报告中所有 `img402.dev` URL 替换为微信 mmbiz URL
2. **去除 Part B**：从 `\n## Part B:` 位置截断
3. **去除结构化摘要表格**：从 `### 结构化摘要` 到 `### 1. 引言` 之间的表格全部跳过
4. **跳过元标题**：H1（文章主标题）、"## 核心信息"、"## Part A"、"## Part B" 均不输出到正文
5. **研读报告 markdown 中不要写 HTML 标签**：所有内容用纯 Markdown 格式书写
6. **正文开头人工补充基本信息**：从结构化摘要表格中提取 "背景/目标"、"方法"、"结果"、"结论" 四个字段，转换为纯文本摘要段落，格式如下：

```html
<p style="margin-top:16px"><strong>论文标题：</strong>MinerU2.5: ...</p>
<p style="margin-top:16px"><strong>作者团队：</strong>Junbo Niu*, ...</p>
<p style="margin-top:16px"><strong>发布时间：</strong>arXiv:2509.22186v2（2025年9月29日）</p>
<p style="margin-top:16px"><strong>论文地址：</strong>https://arxiv.org/abs/2509.22186</p>
<p style="margin-top:16px"><strong>开源地址：</strong>https://github.com/opendatalab/MinerU</p>
<p style="margin-top:20px"><strong>背景/目标：</strong>从摘要提取的文本</p>
<p style="margin-top:16px"><strong>方法：</strong>从摘要提取的文本</p>
<p style="margin-top:16px"><strong>结果：</strong>从摘要提取的文本</p>
<p style="margin-top:16px"><strong>结论：</strong>从摘要提取的文本</p>
<hr/>
```

#### 5.2 Markdown → HTML 核心规则

| 元素 | 转换规则 |
|---|---|
| **Markdown 图片语法** `![alt](path)` | **先用 re.sub 替换**为 `<p align="center"><img src="mmbiz_url" width="600"/></p>\n<p align="center">alt文字</p>`，**不经过 escape** |
| **HTML img 标签行** | 直接保留，**不经过 escape 函数** |
| **LaTeX 公式** | 替换为 1-2 句话文字描述，跳过 `$$` 行 |
| **H1 标题** | 不输出（标题已在 article.title 设置） |
| **H2 标题** | 输出 `<p style="margin-top:20px"><strong>xxx</strong></p>` |
| **H3/H4 标题** | **全部**加 `margin-top:20px`（上下均有间距，不区分顶级/子章节） |
| **Markdown 表格行** | **全部跳过**（摘要信息已在 5.1 预处理中提取为纯文本） |
| **Markdown 列表项**（`- ` 开头） | 转 `<p>text</p>`，不用 `<ul><li>` |
| **普通段落** | strip_md_format → inline_format → escape → `<p style="margin-top:16px">text</p>` |
| **分隔线** | 转 `<hr/>` |
| **代码块** | 转 `<pre><code>code</code></pre>` |

#### 5.3 段落间距核心经验（最终调试结论）

WeChat 编辑器会忽略纯空行（`''`）和裸 `<br/>` 标签。

**最终方案（调试后最优）**：
- 所有段落统一加 `margin-top:16px`
- **所有 H2/H3/H4 标题统一加 `margin-top:20px`**（上下均有间距，不再区分顶级/子章节）
- 分隔线 `<hr/>` 上下均自然产生间距

**注意**：`text-align:left` 等 CSS 属性在 WeChat 中可能被过滤，**不要加**，直接用裸 `<p>` 标签。

#### 4.4 转换顺序

**正确顺序**：
1. `strip_md_format` — 去掉原始 `**粗体**` 标记（避免双重加粗）
2. `inline_format` — 应用粗体斜体转换
3. `escape` — 转义 HTML 特殊字符

#### 4.5 完整 Python 转换核心逻辑

```python
def escape(s):
    return s.replace('&','&amp;').replace('<','&lt;').replace('>','&gt;')

def inline_format(s):
    s = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', s)
    s = re.sub(r'\*(.+?)\*', r'<em>\1</em>', s)
    s = re.sub(r'\[([^\]]+)\]\(([^)]+)\)', r'<a href="\2">\1</a>', s)
    return s

def strip_md_format(s):
    s = re.sub(r'\*\*(.+?)\*\*', r'\1', s)
    s = re.sub(r'\*(.+?)\*', r'\1', s)
    return s

def is_html_line(line):
    s = line.strip()
    if s.startswith('<p align="center"><img'):
        return True
    if re.match(r'<(p|div|span|h[1-6]|table|thead|tbody|tr|th|td|ul|ol|li|pre|code|hr|br|blockquote)\b', s):
        return True
    if s.startswith('</'):
        return True
    return False

lines = md_content.split('\n')
result = []
table_rows = []
first_block = True
prev_was_hr = False

for line in lines:
    stripped = line.strip()

    if stripped.startswith('|'):                     # 表格行收集
        table_rows.append(stripped)
        continue

    if table_rows:                                    # 输出累积的表格
        result.extend(process_table(table_rows))
        table_rows = []
        first_block = False

    if stripped == '':
        continue

    if re.match(r'^-{3,}$', stripped):             # 分隔线
        result.append('<hr/>')
        result.append('')
        prev_was_hr = True
        first_block = False
        continue

    hm = re.match(r'^(#{1,4})\s+(.+)$', stripped)
    if hm:
        level = len(hm.group(1))
        text = hm.group(2)
        if level == 1:
            continue                                  # H1 跳过
        elif level == 2:
            if text in ['核心信息', 'Part A:', 'Part B:']:
                continue                              # 元标题跳过
            result.append(f'<p style="margin-top:20px"><strong>{inline_format(escape(text))}</strong></p>')
        else:                                          # H3/H4
            is_top = bool(re.match(r'^\d+\.\s+', text))
            needs_margin = (not first_block and is_top) or (prev_was_hr and is_top)
            tag = f'<p style="margin-top:20px"><strong>{inline_format(escape(text))}</strong></p>' if needs_margin else f'<p><strong>{inline_format(escape(text))}</strong></p>'
            result.append(tag)
        prev_was_hr = False
        first_block = False
        continue

    if is_html_line(stripped):                        # 图片等 HTML 行直接保留
        result.append(stripped)
        prev_was_hr = False
        continue

    if stripped.startswith('- '):                     # Markdown 列表项
        text = inline_format(escape(strip_md_format(stripped[2:])))
        result.append(f'<p style="margin-top:16px">{text}</p>')
        prev_was_hr = False
        first_block = False
        continue

    if stripped.startswith('```'):                  # 代码块
        code_lines = []
        while next_line and not next_line.strip().startswith('```'):
            code_lines.append(next_line)
        result.append(f'<pre><code>{escape(chr(10).join(code_lines))}</code></pre>')
        prev_was_hr = False
        first_block = False
        continue

    # 普通段落：全部加 margin-top:16px
    text = inline_format(escape(strip_md_format(stripped)))
    result.append(f'<p style="margin-top:16px">{text}</p>')
    prev_was_hr = False
    first_block = False

html = '\n'.join(result)
```

### Step 6: 上传草稿箱

**★★★ thumb_media_id 必须使用最新获取的 media_id ★★★**：每次重新上传封面图会得到新的 media_id，旧的可能失效。

```python
article = {
    "articles": [{
        "title": "吸引人的文章标题（不用研读报告字样）",
        "author": "极智视界",
        "digest": "简短摘要（不超过54字）",
        "content": html_content,
        "thumb_media_id": "<Step 2.2 获取的 media_id>",
        "need_open_comment": 0,
        "only_fans_can_comment": 0
    }]
}
```

注意 JSON 结构是 `articles` 数组，不是顶层 JSON 对象。

**curl 命令**：
```bash
curl -X POST "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=${TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @/tmp/wechat_article.json
```

成功响应：`{"media_id":"...", "item":[...]}`

**★★★ access_token 过期处理 ★★★**：如果 API 返回 `{"errcode":42001, "errmsg":"access_token expired"}`，立即重新获取 token。

### Step 7: 验证与记录

1. 登录微信公众号后台 → 内容管理 → 草稿箱，确认格式正确
2. 记录 Media ID 到 `memory/YYYY-MM-DD.md`

## 常见问题排查

### 图片显示为纯文本（如 `<p align="center"> <img src="...">`）
- **原因**：HTML 标签被错误转义
- **解决**：HTML 标签行直接 append 到结果列表，**不经过 escape 函数**

### 图片不显示
- **原因**：使用了 img402.dev 等外部图床链接，微信会过滤；或正文图片未上传到微信素材库
- **解决**：所有图片必须先通过 `material/add_material` 上传到微信素材库，用 `mmbiz.qpic.cn` 地址；在研读报告中手动插入

### 图片 URL 被截断导致无法显示
- **原因**：在 shell 中用 `echo $RESP | python3 ...` 提取 URL 时，URL 被 shell 截断（curl 响应的 JSON 中 URL 约 200 字符，超出终端显示宽度被截断），导致微信收到不完整 URL
- **解决**：**必须全程在 Python 中提取 URL**，不用 shell echo：
```python
import json
RESP = subprocess.run(["curl", "-s", "-X", "POST", ...], capture_output=True).stdout
data = json.loads(RESP)
url = data.get("url", "")  # Python 完整提取，不会截断
```
- **验证方法**：JSON 中每个 URL 长度必须 > 150 字符才是完整 URL

### 论文图片与 Figure 编号不匹配
- **原因**：论文 PDF 中提取出来的图片顺序与论文正文的 Figure 编号不一定对应；可能是 Logo、照片示例、supplementary material
- **解决**：对照论文原文 caption 确认每张图的实际内容和位置；用 AI 视觉验证渲染图内容与 caption 描述是否匹配

### 公众号中出现 "豆包AI生成" 等水印图片
- **原因**：这是论文中的实验照片（电商场景真实图片），并非文件损坏；论文用来展示 RefineAnything 的文字/Logo 修复效果
- **解决**：这是正常内容，无需处理

### 矢量图形用 get_images() 提取为空白
- **原因**：会议论文（ACM/SOSP/CVPR 等）的 Figure 几乎全是矢量路径（PDF paths），`get_images()` 返回空或只有 1 张 Logo
- **解决**：矢量图必须用 `get_pixmap()` 渲染（见 paper-parse skill Step 1.2）

### 正文没有图片
- **原因**：PDF 提取只会得到图片文件，不会自动插入正文
- **解决**：在 Step 3 中手动在适当位置插入图片 HTML

### 段落上方没有空行（间距）
- **原因**：WeChat 编辑器会忽略纯空行（`''`）和裸 `<br/>` 标签
- **解决**：所有段落统一加 `margin-top:16px`，所有 H2/H3/H4 标题统一加 `margin-top:20px`

### 子章节上方没有被空行（间距不足）
- **原因**：之前错误地只给顶级章节加间距，子章节（1.1、2.1）不加
- **解决**：**所有** H2/H3/H4 标题统一加 `margin-top:20px`，不再区分顶级/子章节

### 信息类表格内容双重加粗
- **原因**：单元格内容已有 `**粗体**` 标记，先 escape 再 inline_format 会导致重复
- **解决**：先 `strip_md_format`（去掉原始 `**`）再处理

### LaTeX 公式显示为乱码
- **原因**：公众号不支持 LaTeX 渲染
- **解决**：将 `$$...$$` 公式块整体替换为 1-2 句话文字描述

### 正文开头段落没有左对齐
- **原因**：加了 `text-align:left` CSS 被 WeChat 过滤反而出问题
- **解决**：直接用裸 `<p>` 标签，WeChat 默认 `<p>` 即为左对齐，**不要加任何 CSS**

### 研读报告 Markdown 中写了 HTML 标签导致转义问题
- **原因**：PDF 文本提取可能带入 HTML 片段，或手动写了 `<p>` 标签
- **解决**：统一用纯 Markdown 格式书写，转换脚本处理所有 HTML 生成

## 输出产物

- `research-papers/{论文简称}_研读报告.md` — 研读报告（纯 Markdown）
- 草稿箱新文章 — Media ID 记录在 `memory/YYYY-MM-DD.md`

## 完美实践检查清单（每次发布前逐项确认）

- [ ] 正文开头有论文基本信息（标题/作者/时间/链接），纯文本格式，不用表格
- [ ] 正文开头有核心摘要（背景/目标、方法、结果、结论），纯文本格式，不用表格
- [ ] 所有标题（H2/H3/H4）上下均有空行（统一加 `margin-top:20px`）
- [ ] 所有段落统一加 `margin-top:16px`
- [ ] 图片使用 Markdown `![alt](path)` 语法写在 .md 文件中，转换脚本自动替换为 HTML
- [ ] HTML img 标签行直接 append，不经过 escape 函数
- [ ] 封面图 thumb_media_id 使用最新获取的 media_id
- [ ] access_token 如已过期（2小时）先重新获取
- [ ] JSON 中每个图片 URL 长度 > 150 字符（完整 URL）
- [ ] 矢量图形论文（ACM/SOSP/CVPR 等）已用 get_pixmap() 渲染，而非 get_images()

## 相关技能

- `paper-parse`：论文研读（PDF → 双模报告）
- `wechat-mp-publisher`：微信公众号 API 凭证配置
