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

```bash
for f in charts/fig_*.png charts/fig_*.jpeg; do
  TYPE=$(echo $f | grep -q png && echo "image/png" || echo "image/jpeg")
  RESP=$(curl -s -X POST "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=${TOKEN}&type=image" \
    -F "media=@$f;type=$TYPE")
  URL=$(echo $RESP | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
  echo "$f: $URL"
done
```

收集所有返回的 `mmbiz.qpic.cn` URL，建立 URL 映射表。

### Step 3: 转换研读报告（Markdown → HTML）

#### 3.1 预处理

1. **替换图片 URL**：将研读报告中所有 `img402.dev` URL 替换为微信 mmbiz URL
2. **去除 Part B**：从 `\n## Part B:` 位置截断
3. **跳过元标题**：H1（文章主标题）、"## 核心信息"、"## Part A"、"## Part B" 均不输出到正文
4. **研读报告 markdown 中不要写 HTML 标签**：所有内容用纯 Markdown 格式书写

#### 3.2 Markdown → HTML 核心规则

| 元素 | 转换规则 |
|---|---|
| **图片 URL** | 使用微信 mmbiz 地址（img402.dev 不可用） |
| **HTML 标签行** | 直接保留（如 `<p align="center"><img...>`），**不经过 escape 函数** |
| **LaTeX 公式** | 替换为 1-2 句话文字描述，跳过 `$$` 行 |
| **H1 标题** | 不输出（标题已在 article.title 设置） |
| **H2 标题（顶级章节）** | 第一个顶级章节直接输出 `<p><strong>xxx</strong></p>`，后续加 `margin-top:20px` |
| **H3 顶级章节标题**（如 `### 1. 引言`、`### 2. 模型架构`） | 非第一个时加 `margin-top:20px` |
| **H3/H4 子章节**（如 `#### 1.1`、`#### 2.1`） | 输出 `<p><strong>xxx</strong></p>`，**不加 margin-top** |
| **信息类表格**（两列，项目/维度/指标） | 转 `<p><strong>key</strong>：value</p>`，跳过表头行 |
| **复杂表格**（多列、含数据） | 转标准 HTML `<table>` |
| **Markdown 列表项**（`- ` 开头） | 转 `<p>text</p>`，不用 `<ul><li>` |
| **普通段落** | strip_md_format → inline_format → escape → `<p>text</p>` |
| **分隔线** | 转 `<hr/>` |
| **代码块** | 转 `<pre><code>code</code></pre>` |

**段落间距关键**：WeChat 编辑器会忽略纯空行（`''`）和裸 `<br/>` 标签。只在顶级章节标题前加 `margin-top:20px`；在 `<hr/>` 后面紧跟的顶级章节标题也要加 `margin-top:20px`（通过 `prev_was_hr` 标志）。

#### 3.3 转换顺序（重要）

**正确顺序**：
1. `strip_md_format` — 去掉原始 `**粗体**` 标记（避免双重加粗）
2. `inline_format` — 应用粗体斜体转换
3. `escape` — 转义 HTML 特殊字符

#### 3.4 完整 Python 转换关键逻辑

```python
import re

def strip_md_format(s):
    s = re.sub(r'\*\*(.+?)\*\*', r'\1', s)
    s = re.sub(r'\*(.+?)\*', r'\1', s)
    return s

def inline_format(s):
    s = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', s)
    s = re.sub(r'\*(.+?)\*', r'<em>\1</em>', s)
    return s

def escape(s):
    return s.replace('&','&amp;').replace('<','&lt;').replace('>','&gt;')

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

    # 表格行收集
    if stripped.startswith('|'):
        table_rows.append(stripped)
        continue

    # 输出累积的表格
    if table_rows:
        result.extend(process_table(table_rows))
        table_rows = []
        first_block = False

    if stripped == '':
        continue

    # 分隔线
    if re.match(r'^-{3,}$', stripped):
        result.append('<hr/>')
        result.append('')
        prev_was_hr = True
        first_block = False
        continue

    # H1：跳过（标题在外部）
    hm = re.match(r'^(#{1,4})\s+(.+)$', stripped)
    if hm:
        level = len(hm.group(1))
        text = hm.group(2)
        if level == 1:
            continue
        elif level == 2:
            if text in ['核心信息', 'Part A: 深度专业学术解析', 'Part B: 核心逻辑链与价值提炼']:
                continue
            styled = f'<p style="margin-top:20px"><strong>{inline_format(escape(text))}</strong></p>' if not first_block else f'<p><strong>{inline_format(escape(text))}</strong></p>'
            result.append(styled)
        else:  # H3/H4
            is_top = bool(re.match(r'^\d+\.\s+', text))
            needs_margin = (not first_block and is_top) or (prev_was_hr and is_top)
            styled = f'<p style="margin-top:20px"><strong>{inline_format(escape(text))}</strong></p>' if needs_margin else f'<p><strong>{inline_format(escape(text))}</strong></p>'
            result.append(styled)
        prev_was_hr = False
        first_block = False
        continue

    # HTML 标签行（图片等）：直接保留
    if is_html_line(stripped):
        result.append(stripped)
        prev_was_hr = False
        continue

    # Markdown 列表项
    if stripped.startswith('- '):
        text = inline_format(escape(strip_md_format(stripped[2:])))
        result.append(f'<p>{text}</p>')
        prev_was_hr = False
        first_block = False
        continue

    # 代码块
    if stripped.startswith('```'):
        code_lines = []
        while next_line and not next_line.strip().startswith('```'):
            code_lines.append(next_line)
            advance...
        result.append(f'<pre><code>{escape(chr(10).join(code_lines))}</code></pre>')
        prev_was_hr = False
        first_block = False
        continue

    # 普通段落
    text = inline_format(escape(strip_md_format(stripped)))
    result.append(f'<p>{text}</p>')
    prev_was_hr = False
    first_block = False

html = '\n'.join(result)
```

### Step 4: 上传草稿箱

```python
article = {
    "title": "吸引人的文章标题（不用研读报告字样）",
    "author": "极智视界",
    "digest": "简短摘要（不超过54字）",
    "content": html_content,
    "thumb_media_id": "<Step 2.2 获取的 media_id>",
    "need_open_comment": 0,
    "only_fans_can_comment": 0
}
# curl -X POST "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=${TOKEN}" \
#   -H "Content-Type: application/json; charset=utf-8" \
#   -d @/tmp/wechat_article.json
```

成功响应：`{"media_id":"...", "item":[...]}`

### Step 5: 验证与记录

1. 登录微信公众号后台 → 内容管理 → 草稿箱，确认格式正确
2. 记录 Media ID 到 `memory/YYYY-MM-DD.md`

## 常见问题排查

### 图片显示为纯文本（如 `<p align="center"> <img src="...">`）
- **原因**：HTML 标签被错误转义（如 `<` 变成 `&lt;`）
- **解决**：HTML 标签行直接 append 到结果列表，**不经过 escape 函数**

### 图片不显示
- **原因**：使用了 img402.dev 等外部图床链接，微信会过滤
- **解决**：所有图片必须先通过 `material/add_material` 上传到微信素材库，用 `mmbiz.qpic.cn` 地址

### 段落上方没有空行（间距）
- **原因**：WeChat 编辑器会忽略纯空行（`''`）和裸 `<br/>` 标签
- **解决**：在顶级章节标题（H2 全部、H3 中匹配 `^\d+\.\s+` 的）的 `<p>` 上加 `style="margin-top:20px"`

### 子章节（1.1、2.1）上方也被加了空行
- **原因**：误将 H4 子章节也加了 `margin-top`
- **解决**：只有 H2 全部、H3 中匹配 `^\d+\.\s+`（不含小数点）的才加 margin；H4 子章节（如 `#### 1.1`）不加

### 信息类表格内容双重加粗（如 `<strong><strong>作者团队</strong>：</strong>`）
- **原因**：单元格内容已有 `**粗体**` 标记，先 escape 再 inline_format 会导致重复
- **解决**：先 `strip_md_format`（去掉原始 `**`）再处理

### LaTeX 公式显示为乱码
- **原因**：公众号不支持 LaTeX 渲染
- **解决**：将 `$$...$$` 公式块整体替换为 1-2 句话文字描述

### 正文开头的段落没有左对齐
- **原因**：WeChat 默认 `<p>` 即为左对齐，无需额外 CSS
- **解决**：直接使用裸 `<p>` 标签，不要加 `style="text-align:left"` 等多余属性

### 研读报告 Markdown 中写了 HTML 标签导致转义问题
- **原因**：PDF 文本提取可能会带入 HTML 片段，或手动写了 `<p>` 标签
- **解决**：研读报告 markdown 中统一用纯 Markdown 格式，转换脚本处理所有 HTML 生成

## 输出产物

- `research-papers/{论文简称}_研读报告.md` — 研读报告（纯 Markdown）
- 草稿箱新文章 — Media ID 记录在 `memory/YYYY-MM-DD.md`

## 相关技能

- `paper-parse`：论文研读（PDF → 双模报告）
- `wechat-mp-publisher`：微信公众号 API 凭证配置
