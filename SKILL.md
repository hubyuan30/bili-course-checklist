---
name: bili-course-checklist
description: 把 B 站课程（BV 号/视频链接）整理成黑白打印版《学习待办清单》PDF：自动抓取分P标题与时长、按教材结构分章、统计章内累计时长并按页精确分页。当用户给出 B 站课程链接并要求生成待办表、学习计划、进度清单 PDF，或要求修改已有清单的列与格式时使用。
---

# B 站网课待办清单 PDF

把一门 B 站课程变成一张可以打印、可以打勾的进度表。核心是三件事：**数据只来自 B 站 API**、**分章合理**、**排版规整（黑白、不跨页错位）**。

## 何时使用

- 用户给 B 站课程链接，要「待办清单 / 学习计划 / 进度表」PDF
- 用户要改已有清单的列结构、单位、命名
- 用户要批量把多门课做成同一套格式

## 输出规格（最新版，务必照此）

文件名：`科目-老师-网课清单.pdf`（例：`模电-姬清影-网课清单.pdf`、`概率论-宋浩2.0-网课清单.pdf`）

表格六列，顺序固定：

| 编号 | 课程内容 | 时长 | 本章累计（到本节上完） | 完成 | 计划时间 |
|---|---|---|---|---|---|
| 1 | 引言 | 10:45 | 10 min |  |  |

- **编号**：全课连续 1…N，不重复章节内编号
- **课程内容**：原视频分P标题（可去【】序号、可修错别字，需在回复中说明）
- **时长**：`mm:ss`，超过 60 分钟按分钟累计（如 `84:52`）
- **本章累计**：本章第一节到本节上完的累计时长，**英文单位、只到分钟**：`38 min`、`1 h 34 min`，**不显示秒**
- **完成**：空白单元格，**不要放 □ 或任何符号**（表格本身已有格子，手写打勾）
- **计划时间**：空白

样式：

- 纯黑白：页白底、文字纯黑、内网格 0.5pt `#333333`、外框 1pt 黑
- 章标题行：五/六列跨列合并、加粗、字号 13.5、下方 1pt 黑线，文案 `第X章　名称　共 N 讲 · X 小时 Y 分`
- 页头：标题（20pt 加粗居中）+ 副标题（`UP主：xxx｜副标题（共 N 讲，总时长 X 小时 Y 分）`）+ 使用说明一行
- 每页重复表头；**章标题不允许单独落在页面最底部**
- A4，左右 0.7in，上 0.65in，下 0.6in

## 执行流程

### 1. 取数据

从链接里取 BV 号，然后：

```
GET https://api.bilibili.com/x/web-interface/view?bvid={BV}
Header: User-Agent: Mozilla/5.0 ... ; Referer: https://www.bilibili.com/
```

- `data.pages[]` 给 `part`（标题）与 `duration`（秒）
- `data.duration` 是整段总秒数，用于校验
- 若 `data.ugc_season` 存在：说明这是合集里的单集，**必须改用** `data.ugc_season.sections[].episodes[]`，字段是 `title` / `arc.duration` / `bvid`，否则只能拿到 1 集
- 分P数 = `data.videos`，与 pages 长度或 episodes 数核对

### 2. 校验时长

先把待收录的每节秒数求和，与 `data.duration` 对比：

- 全收：两者必须相等
- 略去非教学分P：`总和 + 被略去分P秒数 = data.duration`

对不上就说明有分P漏了或重复，不要直接出 PDF。

### 3. 分章

B 站课程基本没有官方章节，需要按分P标题归纳到教材/知识结构。规则：

- 章节名用教材口径：`第1章　常用半导体器件`（章号与名称之间用一个全角空格）
- 原视频标题里有 `§1.1` 这类小节号的，**照原样保留在课时标题里**，不重新编号
- 原视频跳号（例如只有 §4 直接到 §6）就照抄，并在页脚备注里说明，不要自作主张改号
- 原视频把两门课拼在一起（如傅里叶 + 拉普拉斯，各自都有第1章）时，只分两个大章，用 `第一章 / 第二章` 区分
- 开场说明、结束语可归一个「课程说明（预备）」或直接略去

### 4. 生成 PDF

把下面模板复制到项目目录，改 `CONFIG` 与 `CHAPTERS`，`python make_todo_pdf.py`。

### 5. 自检（必做）

生成后重新读 PDF 核验，不要只看脚本没报错：

- 页数是否合理（约 25 行/页）
- 每页编号条目数相加 = N
- **每页最后一行不是章标题**（用 PyMuPDF 取每页文本最后一行判断）
- 累计列最后一节 = 该章总时长（分钟口径一致）
- 目视抽查首页、跨页页、末页

## 数据规则

- 标题、时长、UP 主全部来自 API，**不编造、不估算**
- 略去非教学分P（推广、插曲、结语）时，页脚写明略去了什么、多少秒
- 原视频标题有错别字可以修正（如「开发电路」→「运算电路」），但要在回复里说明改了什么
- 章节划分是归纳结果，页脚注明「原视频未提供官方章节」

## 分页坑（都踩过，直接照做）

1. `Spacer.wrap()` 返回高度 0 —— 取 `f.height`
2. `Paragraph` 还要计入 `getSpaceAfter()`（标题样式常带 spaceAfter，少算 10pt 就会超页）
3. 表格总宽必须 **≤ 版心宽**（A4 8.27in − 0.7in×2 = 6.87in）；超一点点就会被 reportlab 二次拆分，出现「下一页只剩一行」
4. 不要用「单行高度累加」估算分页——会低估。正确做法：把候选行拼成整表 `wrap()` 实测，超了就断页
5. 章标题行做高度测量时必须带 `SPAN`，否则文字被挤进窄列，高度虚高到 300pt
6. 断页时若发现该页最后一行是章标题，把它挪到下一页

## 代码模板

```python
# -*- coding: utf-8 -*-
"""B站网课待办清单 PDF（黑白打印版）
用法：改 CONFIG 与 CHAPTERS 后直接运行；依赖 pip install reportlab
"""
import os
from reportlab.lib.pagesizes import A4
from reportlab.lib.units import inch
from reportlab.lib.colors import HexColor
from reportlab.lib.styles import ParagraphStyle
from reportlab.lib.enums import TA_CENTER, TA_LEFT
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.platypus import (SimpleDocTemplate, Paragraph, Spacer, Table,
                                TableStyle, PageBreak)

# ============ 只改这里 ============
CONFIG = {
    "title": "【老师】《课程》学习待办清单",
    "up": "UP主",
    "subtitle": "版本说明（可留空）",
    "out": "科目-老师-网课清单.pdf",
}

# 章节数据: (章节标题, [(课时标题, 时长秒数), ...])
CHAPTERS = [
    ("第1章　示例章节", [
        ("1.1 示例课时", 1800),
        ("1.2 示例课时", 2400),
    ]),
]
# =================================


def _reg(name, paths):
    for p in paths:
        if os.path.exists(p):
            try:
                pdfmetrics.registerFont(TTFont(name, p, subfontIndex=0))
                return True
            except Exception:
                try:
                    pdfmetrics.registerFont(TTFont(name, p))
                    return True
                except Exception:
                    continue
    return False


def register_cjk_font():
    ok = _reg("CJKFont", ["C:/Windows/Fonts/msyh.ttc",
                          "C:/Windows/Fonts/simsun.ttc",
                          "C:/Windows/Fonts/msyh.ttf"])
    bold_ok = _reg("CJKFont-Bold", ["C:/Windows/Fonts/msyhbd.ttc",
                                    "C:/Windows/Fonts/simhei.ttf"])
    if ok and bold_ok:
        pdfmetrics.registerFontFamily("CJKFont", normal="CJKFont",
                                      bold="CJKFont-Bold",
                                      italic="CJKFont",
                                      boldItalic="CJKFont-Bold")
    elif ok:
        pdfmetrics.registerFontFamily("CJKFont", normal="CJKFont",
                                      bold="CJKFont", italic="CJKFont",
                                      boldItalic="CJKFont")
    return "CJKFont" if ok else None


CJK = register_cjk_font()
CJK_BOLD = "CJKFont-Bold" if "CJKFont-Bold" in pdfmetrics.getRegisteredFontNames() else CJK

title_s = ParagraphStyle('title', fontName=CJK_BOLD, fontSize=19, leading=26,
                         textColor=HexColor('#000000'), alignment=TA_CENTER,
                         spaceAfter=6, wordWrap='CJK')
sub_s = ParagraphStyle('sub', fontName=CJK, fontSize=11, leading=16,
                       textColor=HexColor('#000000'), alignment=TA_CENTER,
                       spaceAfter=2, wordWrap='CJK')
foot_s = ParagraphStyle('foot', fontName=CJK, fontSize=9, leading=14,
                        textColor=HexColor('#000000'), alignment=TA_LEFT,
                        wordWrap='CJK')
cell_s = ParagraphStyle('cell', fontName=CJK, fontSize=10.5, leading=15,
                        textColor=HexColor('#000000'), alignment=TA_LEFT,
                        wordWrap='CJK')
num_s = ParagraphStyle('num', fontName=CJK, fontSize=10.5, leading=15,
                       textColor=HexColor('#000000'), alignment=TA_CENTER,
                       wordWrap='CJK')
ch_s = ParagraphStyle('ch', fontName=CJK_BOLD, fontSize=13.5, leading=19,
                      textColor=HexColor('#000000'), alignment=TA_LEFT,
                      wordWrap='CJK')
meta_s = ParagraphStyle('meta', fontName=CJK, fontSize=10, leading=16,
                        textColor=HexColor('#000000'), alignment=TA_CENTER,
                        spaceAfter=2, wordWrap='CJK')
head_s = ParagraphStyle('head', fontName=CJK_BOLD, fontSize=9.5, leading=13,
                        textColor=HexColor('#000000'), alignment=TA_CENTER,
                        wordWrap='CJK')


def fmt_dur(sec):
    """秒 -> mm:ss（>60 分钟按分钟累计）"""
    m, s = divmod(int(sec), 60)
    return "%d:%02d" % (m, s)


def fmt_sum(sec):
    """秒 -> 中文总时长，用于章标题与页头"""
    total_min = int(sec) // 60
    h, m = divmod(total_min, 60)
    return ("%d 小时 %d 分" % (h, m)) if h else ("%d 分钟" % m)


def fmt_cum(sec):
    """秒 -> 章内累计（英文单位，只到分钟）"""
    total_min = int(sec) // 60
    h, m = divmod(total_min, 60)
    return ("%d h %d min" % (h, m)) if h else ("%d min" % m)


col_widths = [0.5 * inch, 2.95 * inch, 0.6 * inch, 1.32 * inch,
              0.5 * inch, 0.93 * inch]          # 合计 6.80in ≤ 版心 6.87in

TOTAL_N = sum(len(items) for _, items in CHAPTERS)
TOTAL_SEC = sum(sec for _, items in CHAPTERS for _, sec in items)

HEADER = ["编号", "课程内容", "时长", "本章累计<br/>（到本节上完）", "完成", "计划时间"]

prefix = [
    Spacer(1, 8),
    Paragraph(CONFIG["title"], title_s),
    Paragraph("UP主：%s%s（共 %d 讲，总时长 %s）"
              % (CONFIG["up"],
                 ("｜" + CONFIG["subtitle"]) if CONFIG["subtitle"] else "",
                 TOTAL_N, fmt_sum(TOTAL_SEC)), meta_s),
    Paragraph("使用说明：“本章累计”为该节上完后本章累计已学时长，便于估算章节进度。", sub_s),
    Spacer(1, 10),
]

PAD_TB, PAD_LR = 6, 7


def row_style(span_rows):
    cs = [
        ('FONTNAME', (0, 0), (-1, -1), CJK),
        ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'),
        ('ALIGN', (0, 0), (0, -1), 'CENTER'),
        ('ALIGN', (2, 0), (2, -1), 'CENTER'),
        ('ALIGN', (3, 0), (3, -1), 'CENTER'),
        ('ALIGN', (4, 0), (4, -1), 'CENTER'),
        ('ALIGN', (5, 0), (5, -1), 'CENTER'),
        ('TOPPADDING', (0, 0), (-1, -1), PAD_TB),
        ('BOTTOMPADDING', (0, 0), (-1, -1), PAD_TB),
        ('LEFTPADDING', (0, 0), (-1, -1), PAD_LR),
        ('RIGHTPADDING', (0, 0), (-1, -1), PAD_LR),
        ('BACKGROUND', (0, 0), (-1, -1), HexColor('#ffffff')),
        ('GRID', (0, 0), (-1, -1), 0.5, HexColor('#333333')),
        ('BOX', (0, 0), (-1, -1), 1, HexColor('#000000')),
        ('TOPPADDING', (0, 0), (-1, 0), 7),
        ('BOTTOMPADDING', (0, 0), (-1, 0), 7),
    ]
    for r in span_rows:
        cs += [('SPAN', (0, r), (-1, r)),
               ('TOPPADDING', (0, r), (-1, r), 9),
               ('BOTTOMPADDING', (0, r), (-1, r), 9),
               ('LINEBELOW', (0, r), (-1, r), 1, HexColor('#000000'))]
    return TableStyle(cs)


# 组织行：(单元格列表, 是否章标题行)
all_rows = []
seq = 1
for ch_title, items in CHAPTERS:
    ch_sec = sum(sec for _, sec in items)
    all_rows.append(([Paragraph("<b>%s</b>　共 %d 讲 · %s"
                                % (ch_title, len(items), fmt_sum(ch_sec)), ch_s),
                      "", "", "", "", ""], True))
    cum = 0
    for name, sec in items:
        cum += sec
        all_rows.append(([Paragraph(str(seq), num_s),
                          Paragraph(name, cell_s),
                          Paragraph(fmt_dur(sec), num_s),
                          Paragraph(fmt_cum(cum), num_s),
                          Paragraph("", cell_s),
                          Paragraph("", cell_s)], False))
        seq += 1

TABLE_W = sum(col_widths)
PAGE_H = A4[1] - 0.65 * inch - 0.6 * inch

# 前缀高度：Spacer 取 height，段落取 wrap 高度 + spaceAfter
prefix_h = sum((f.height if isinstance(f, Spacer) else f.wrap(TABLE_W, PAGE_H)[1])
               + getattr(f, 'getSpaceAfter', lambda: 0)() for f in prefix)


def make_page_table(page_rows):
    cells = [[Paragraph(h, head_s) for h in HEADER]] + [r[0] for r in page_rows]
    spans = [i + 1 for i, r in enumerate(page_rows) if r[1]]
    t = Table(cells, colWidths=col_widths, repeatRows=0)
    t.setStyle(row_style(spans))
    return t


# 整表实测分页：逐行试加，超页即断；章标题不留页脚
pages, idx, first_page = [], 0, True
while idx < len(all_rows):
    limit = PAGE_H - (prefix_h if first_page else 0) - 2
    take = 0
    while idx + take < len(all_rows):
        if make_page_table(all_rows[idx:idx + take + 1]).wrap(TABLE_W, 10 ** 6)[1] <= limit:
            take += 1
        else:
            break
    if take == 0:
        take = 1
    if idx + take < len(all_rows) and all_rows[idx + take - 1][1] and take > 1:
        take -= 1
    pages.append(all_rows[idx:idx + take])
    idx += take
    first_page = False

story = list(prefix)
for pi, page_rows in enumerate(pages):
    story.append(make_page_table(page_rows))
    if pi < len(pages) - 1:
        story.append(PageBreak())

story.append(Spacer(1, 10))
story.append(Paragraph(
    "备注：本清单依据 B 站视频 %s（UP主：%s）整理，标题与单节时长取自 B 站接口，"
    "章节按知识结构归纳；已略去非教学分P（如有）。“本章累计”为该节上完后本章累计已学时长。"
    % (CONFIG.get("bvid", ""), CONFIG["up"]), foot_s))

out = os.path.join(os.path.dirname(os.path.abspath(__file__)), CONFIG["out"])
doc = SimpleDocTemplate(out, pagesize=A4,
                        leftMargin=0.7 * inch, rightMargin=0.7 * inch,
                        topMargin=0.65 * inch, bottomMargin=0.6 * inch,
                        title=CONFIG["title"])
doc.build(story)
print("PDF_OK:", out)
print("TOTAL:", TOTAL_N, "讲", TOTAL_SEC, "秒 =", fmt_sum(TOTAL_SEC))
```

## 回复用户时

- 附 `computer://` 链接给出 PDF
- 报一句核心数字：N 讲、总时长，以及和 API 总秒数的校验结果
- 说明章节是归纳的、哪些非教学分P被略去
- 若改过原始标题的错别字，明说改了哪个
