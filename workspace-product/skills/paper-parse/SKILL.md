---
name: paper-parse
description: 对用户提供的任何学术论文（PDF附件或URL）进行双模式深度研读。当用户请求分析、研读、解读或总结一篇学术论文时，使用此技能。一次性生成两份报告：Part A 面向研究者的深度专业解析，Part B 面向快速理解的核心逻辑与价值提炼。
---

# Paper Parse

对任何学术论文进行专业深度解析，一次性产出两种不同深度的报告。

## 核心原则

- **学术严谨性**: 对研究设计、数据结果、论证逻辑的转述必须绝对准确，符合该领域学术规范。
- **理论深度**: 清晰揭示论文的理论基础、核心假设，以及它对现有理论体系的补充、修正或颠覆。
- **完整复现**: 完整呈现从提出问题到得出结论的全过程，特别是方法论和关键数据，做到关键信息零遗漏。
- **超越翻译**: 产出物应比线性翻译稿更能清晰地揭示论文的内在逻辑和创新点。
- **双模输出**: 始终在一个最终交付文件中同时提供 Part A 和 Part B。
- **图表优先**: 论文中的关键图表（原图）必须提取并插入报告对应位置，让报告图文并茂、可读性更强。图表是洞察的载体，不是装饰。

## 工作流程

论文研读分四步执行：

### Step 1: 通读论文全文 & 提取图表

#### 1.1 下载并提取全文

对于 URL 来源的论文，先下载 PDF，再使用 PyMuPDF 提取全文（`pdftotext` 命令通常不可用）：

```python
import fitz
doc = fitz.open('paper.pdf')
text = ''
for page in doc:
    text += page.get_text()
with open('paper.txt', 'w') as f:
    f.write(text)
print(f'Extracted {len(text)} chars, {len(doc)} pages')
doc.close()
```

#### 1.2 提取论文原生图表（关键！）

**必须提取 PDF 中的原生图片**，而不是整页截图。使用 PyMuPDF 提取嵌入图片：

```python
import fitz, os

os.makedirs('charts', exist_ok=True)
doc = fitz.open('paper.pdf')

for page_num in range(len(doc)):
    page = doc[page_num]
    images = page.get_images(full=True)
    for img in images:
        xref = img[0]
        base = doc.extract_image(xref)
        ext = base['ext']        # 如 'png'
        w, h = base['width'], base['height']
        with open(f'charts/fig_p{page_num+1}_xref{xref}.{ext}', 'wb') as f:
            f.write(base['image'])
        print(f'Saved: fig_p{page_num+1}_xref{xref}.{ext} ({w}x{h})')
doc.close()
```

**图表提取标准**（必须提取的图表类型）：
- 论文架构图 / 系统框图 / 框架图
- 方法流程图 / 算法示意图
- 主要实验结果图（主实验、对比实验）
- 数据集构成图 / 样本分布图

### Step 2: 综合分析

创建临时分析文件 `temp_analysis.md`，提取并组织以下要素：
- 研究问题、假设、方法论、数据来源
- 核心发现与关键数据
- 理论贡献与实践意义
- 论文的根本矛盾点、切入视角、方法创新
- 已提取的图表清单：

```markdown
[[CHARTS]]
- fig_p3_xref128.png: 图1，论文整体架构/系统框图
- fig_p4_xref182.png: 图2左，缩放点积注意力示意
- fig_p4_xref183.png: 图2右，多头注意力示意
[[/CHARTS]]
```

### Step 3: 图片处理与上传

#### 3.1 图片大小检查与缩放

如果图片宽度超过 900px，需要缩放以避免过大。使用 macOS 内置的 `sips` 命令：

```bash
# 检查图片宽度
sips -g pixelWidth charts/xxx.png | grep pixelWidth

# 缩放宽度至 900px（保持比例）
sips -Z 900 charts/xxx.png --out charts/xxx_resized.png
```

#### 3.2 上传图床获取 URL

**⚠️ 微信公众号发布时图片不能用外部图床**：微信富文本不支持 img402.dev 等外部链接，必须先上传到微信素材库获取 mmbiz URL。

**非公众号场景**（如飞书、GitHub）：上传到 img402.dev（无需认证，免费使用，7天有效期）：

```bash
curl -s -X POST https://img402.dev/api/free \
  -F "image=@charts/xxx_resized.png" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['url'])"
```

成功后会返回类似 `https://i.img402.dev/abc123.png` 的 URL。

**公众号发布场景**：必须调用微信素材库 API 上传（详见 wechat-mp-publisher skill），每张正文图片都要上传获取 mmbiz URL，封面单独上传获取 thumb_media_id。

### Step 4: 撰写双模报告

创建最终交付文件 `[论文简称]_研读报告.md`。

**撰写 Part A 前**，先读取模板：
- `~/.openclaw/workspace-product/skills/paper-parse/references/part-a-template.md`
- `~/.openclaw/workspace-product/skills/paper-parse/references/part-b-template.md`

### Step 5: 图表嵌入规范（重要！）

#### 图片 Markdown 格式

在 Markdown 中嵌入居中且带宽度控制的图片，**必须使用 HTML 格式**（不能用纯 Markdown 的 `![]()` 语法，否则无法控制居中和尺寸）：

```html
<p align="center">
<img src="https://i.img402.dev/abc123.png" width="900"/>
</p>
<p align="center">图1：图片标题说明（来源：原论文 [图1号]）</p>
```

#### 图片标题格式要求

- 标题文本**不加粗、不加星号**，使用纯文本
- 不需要写"已缩放至 XXXpx"等字样，缩放信息只在处理时用
- 标题格式统一为：`图X：描述（来源：原论文 [图X号]）`

#### LaTeX 公式说明

报告中插入的 LaTeX 公式（如 `$$公式$$`）需要目标 Markdown 渲染器支持 MathJax 或 KaTeX 才能正常显示。常见环境支持情况：

| 查看环境 | LaTeX 公式支持 |
|---|---|
| Typora / VS Code (安装 LaTeX 插件) | ✅ 支持 |
| 飞书 / Notion | ✅ 支持 |
| GitHub / GitLab README | ❌ 不支持（会显示原始 LaTeX 代码） |
| 普通 Markdown 编辑器 | ❌ 通常不支持 |

如果目标平台不支持 LaTeX，可将公式也做成图片嵌入，但这不是本次技能的标准处理方式——仅在用户明确要求时执行。

### Step 6: 交付成果

使用 `message` 工具发送报告文件，消息文本中简要概括论文的核心创新、关键发现和理论价值。

---

## 写作质量标准

- 使用完整段落而非大量列表，交替使用段落和表格组织信息
- 关键术语首次出现时提供中英文对照
- 引用论文中的具体数据和实验结果来支撑每一项论述
- Part A 追求专业性和完整性，Part B 追求洞察力和凝练度
- 最终文件中 Part A 和 Part B 之间用 `---` 分隔

---

## 附录：完整图表处理流程示例

```bash
# 1. 提取 PDF 原生图片
python3 << 'EOF'
import fitz, os
doc = fitz.open('paper.pdf')
os.makedirs('charts', exist_ok=True)
for page_num in range(len(doc)):
    for img in doc[page_num].get_images(full=True):
        xref = img[0]
        base = doc.extract_image(xref)
        with open(f'charts/fig_p{page_num+1}_xref{xref}.{base["ext"]}', 'wb') as f:
            f.write(base['image'])
doc.close()
EOF

# 2. 缩放过大图片（宽度 > 900px）
for f in charts/*.png; do
  w=$(sips -g pixelWidth "$f" | grep pixelWidth | awk '{print $2}')
  if [ "$w" -gt 900 ]; then
    sips -Z 900 "$f" --out "${f%.png}_resized.png"
  fi
done

# 3. 上传图床
for f in charts/*_resized.png charts/*.png; do
  url=$(curl -s -X POST https://img402.dev/api/free -F "image=@$f" | \
        python3 -c "import sys,json; d=json.load(sys.stdin); print(d['url'])")
  echo "$f: $url"
done
```

## 图床服务

默认使用 **img402.dev**（GitHub Image Hosting Skill）上传图片：
- 地址：https://img402.dev/api/free
- 限制：单文件 < 1MB，免费无需认证，保留 7 天
- 上传后返回 `https://i.img402.dev/xxx.png` 格式 URL

## 操作备忘（实战经验）

- **产出路径**：`workspace-product/research-papers/{论文简称}_研读报告.md`
  - 例：DeepSeek-R1、YOLO
  - **禁止**放入 `memory/` 目录（那是每日笔记和长期记忆的专属位置）
- **img402.dev 上传**：每张图单独一次 POST；若并发上传多张图，超时风险增加，建议逐张上传；整个上传流程（含缩放）约 60s，优先处理需要缩放的图
- **会话管理**：论文研读通常需要多轮工具调用，建议在 exec 中合并相邻的同类操作（如先提取全文再提取图表），减少往返延迟
- **今日产出记录**：每次研读完成后，在 `memory/YYYY-MM-DD.md` 末尾追加产出路径，形成可追溯的日志
