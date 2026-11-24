---
name: bili-course-checklist
description: 把 B 站课程（BV 号或视频链接）整理成学习待办清单，可输出黑白打印版 PDF，也可输出可交互的单文件 HTML（点选打勾、自动记录完成时间、自动算进度）。当用户给出 B 站课程链接并要求生成待办表、学习计划、进度清单，或要求修改已有清单的列与格式时使用。
---

# B 站网课待办清单（PDF / HTML）

把一门 B 站课程变成一张能打印、能打勾的进度表。核心是三件事：**数据只来自 B 站接口**、**分章合理**、**排版规整（不跨页错位）**。

## 输入与输出

**输入**

| 项 | 说明 |
|---|---|
| 必须 | 一个 B 站视频链接或 BV 号（合集里的单集也可以，会自动取整个合集） |
| 可选 | 要 PDF、要 HTML，或两者都要（默认两者都出） |
| 可选 | 章节划分偏好、是否略去某些分P（如推广、插曲、结语） |
| 可选 | 输出文件名（默认 `科目-老师-网课清单`） |

**输出**

| 文件 | 形态 | 说明 |
|---|---|---|
| `科目-老师-网课清单.pdf` | A4 打印版 | 六列表格，纯黑白，「完成」「完成时间」留空手写 |
| `科目-老师-网课清单.html` | 单文件网页 | 同样六列，可点选打勾，勾上自动写入完成时间，自动统计总进度和每章进度，数据存本地浏览器 |

两者共用同一份课程数据（`checklist_data.py` 里的 `CONFIG` 与 `CHAPTERS`），改一处即可同时更新。

## 输出规格（最新版，务必照此）

表格六列，顺序固定：

| 编号 | 课程内容 | 时长 | 本章累计（到本节上完） | 完成 | 完成时间 |
|---|---|---|---|---|---|
| 3 | 本征半导体、杂质半导体 | 38:26 | 38 min |  |  |

- **编号**：全课连续 1…N
- **课程内容**：原视频分P标题（可去【】序号、可修错别字，需在回复中说明）
- **时长**：`mm:ss`，超过 60 分钟按分钟累计（如 `84:52`）
- **本章累计**：本章第一节到本节上完的累计时长，**英文单位、只到分钟**：`38 min`、`1 h 34 min`，不显示秒
- **完成**：PDF 留空白格子手写打勾；HTML 是可点的方框
- **完成时间**：PDF 留空手写；HTML 勾选时自动写入 `YYYY-MM-DD HH:MM`，取消勾选即清空

样式（PDF）：

- 纯黑白：白底、纯黑字、内网格 0.5pt `#333333`、外框 1pt 黑
- 章标题行：跨列合并、加粗、字号 13.5、下方 1pt 黑线，文案 `第X章　名称　共 N 讲 · X 小时 Y 分`
- 页头：标题（19–20pt 加粗居中）+ 副标题（`UP主：xxx｜副标题（共 N 讲，总时长 X 小时 Y 分）`）+ 使用说明一行
- 每页重复表头；章标题不允许单独落在页面最底部
- A4，左右 0.7in，上 0.65in，下 0.6in

样式（HTML）：

- 正文黑白，只用一个深墨绿 `#2a6249` 做强调：进度数字、进度条、勾选框对勾、完成时间文字、有进度的章计数
- 打印时强调色自动转黑，保持纸面纯黑白
- 顶部一条工具条：`已学 X / N 讲 · 已学时长 / 总时长`、进度条、`打印 / 存为 PDF`、`清空记录`；打印时整条隐藏

## 执行流程

### 1. 取数据

从链接里取 BV 号，然后：

```
GET https://api.bilibili.com/x/web-interface/view?bvid={BV}
Header: User-Agent: Mozilla/5.0 ... ; Referer: https://www.bilibili.com/
```

- `data.pages[]` 给 `part`（标题）与 `duration`（秒）
- `data.duration` 是整段总秒数，用于校验
- 若 `data.ugc_season` 存在：说明这是合集里的单集，**必须改用** `data.ugc_season.sections[].episodes[]`（字段 `title` / `arc.duration` / `bvid`），否则只能拿到 1 集

### 2. 校验时长

待收录的每节秒数求和，与 `data.duration` 对比：

- 全收：两者必须相等
- 略去非教学分P：`总和 + 被略去分P秒数 = data.duration`

对不上说明有漏或重复，不要直接出文件。

### 3. 分章

B 站课程基本没有官方章节，按分P标题归纳到教材/知识结构：

- 章节名用教材口径：`第1章　常用半导体器件`（章号与名称之间一个全角空格）
- 原视频标题里的 `§1.1` 这类小节号照原样保留，不重新编号
- 原视频跳号（如只有 §4 直接到 §6）就照抄，并在页脚备注说明
- 一门课里套两门课（如傅里叶 + 拉普拉斯，各自有第1章）时只分两个大章，用 `第一章 / 第二章`
- 开场说明、结束语可归一个「课程说明（预备）」或直接略去

### 4. 生成文件

数据写进 `checklist_data.py`（`CONFIG` + `CHAPTERS`），再分别跑两个模板：

- `python make_todo_pdf.py` → PDF
- `python make_todo_html.py` → HTML

### 5. 自检（必做）

不要只看脚本没报错，生成后重新读文件核验：

- PDF 页数是否合理（约 25 行/页）；每页条目数相加 = N
- **每页最后一行不是章标题**（用 PyMuPDF 取每页文本最后一行判断）
- 累计列最后一节 = 该章总时长（分钟口径一致）
- HTML 用 Playwright 打开：勾 3 讲看进度是否联动、取消 1 讲看时间是否清空、`page.pdf()` 看打印页数与表头重复
- 目视抽查首页、跨页页、末页

## 数据规则

- 标题、时长、UP 主全部来自接口，**不编造、不估算**
- 略去非教学分P（推广、插曲、结语）时，页脚写明略去了什么、多少秒
- 原视频标题有错别字可以修正（如「开发电路」→「运算电路」），但要说明改了什么
- 章节划分是归纳结果，页脚注明「原视频未提供官方章节」

## 生成 PDF（代码模板）

```python
# -*- coding: utf-8 -*-
"""课程数据 -> A4 黑白待办清单 PDF"""
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

from checklist_data import CONFIG, CHAPTERS


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
    m, s = divmod(int(sec), 60)
    return "%d:%02d" % (m, s)


def fmt_sum(sec):
    total_min = int(sec) // 60
    h, m = divmod(total_min, 60)
    return ("%d 小时 %d 分" % (h, m)) if h else ("%d 分钟" % m)


def fmt_cum(sec):
    total_min = int(sec) // 60
    h, m = divmod(total_min, 60)
    return ("%d h %d min" % (h, m)) if h else ("%d min" % m)


col_widths = [0.5 * inch, 2.95 * inch, 0.6 * inch, 1.32 * inch,
              0.5 * inch, 0.93 * inch]          # 合计 6.80in <= 版心 6.87in

TOTAL_N = sum(len(items) for _, items in CHAPTERS)
TOTAL_SEC = sum(sec for _, items in CHAPTERS for _, sec in items)

HEADER = ["编号", "课程内容", "时长", "本章累计<br/>（到本节上完）", "完成", "完成时间"]

prefix = [
    Spacer(1, 8),
    Paragraph(CONFIG["title"], title_s),
    Paragraph("UP主：%s%s（共 %d 讲，总时长 %s）"
              % (CONFIG["up"],
                 ("｜" + CONFIG["subtitle"]) if CONFIG.get("subtitle") else "",
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

prefix_h = sum((f.height if isinstance(f, Spacer) else f.wrap(TABLE_W, PAGE_H)[1])
               + getattr(f, 'getSpaceAfter', lambda: 0)() for f in prefix)


def make_page_table(page_rows):
    cells = [[Paragraph(h, head_s) for h in HEADER]] + [r[0] for r in page_rows]
    spans = [i + 1 for i, r in enumerate(page_rows) if r[1]]
    t = Table(cells, colWidths=col_widths, repeatRows=0)
    t.setStyle(row_style(spans))
    return t


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
story.append(Paragraph(CONFIG["note"], foot_s))

out = os.path.join(os.path.dirname(os.path.abspath(__file__)),
                   CONFIG["stem"] + ".pdf")
doc = SimpleDocTemplate(out, pagesize=A4,
                        leftMargin=0.7 * inch, rightMargin=0.7 * inch,
                        topMargin=0.65 * inch, bottomMargin=0.6 * inch,
                        title=CONFIG["title"])
doc.build(story)
print("PDF_OK:", out)
print("TOTAL:", TOTAL_N, "讲", TOTAL_SEC, "秒 =", fmt_sum(TOTAL_SEC))
```

## 生成 HTML（代码模板）

```python
# -*- coding: utf-8 -*-
"""课程数据 -> 单文件 HTML 待办清单（可勾选、自动记完成时间、可打印）"""
import os
import html as h

from checklist_data import CONFIG, CHAPTERS

TOTAL_N = sum(len(items) for _, items in CHAPTERS)
TOTAL_SEC = sum(sec for _, items in CHAPTERS for _, sec in items)


def fd(sec):
    m, s = divmod(int(sec), 60)
    return "%d:%02d" % (m, s)


def fs(sec):
    t = int(sec) // 60
    a, b = divmod(t, 60)
    return ("%d 小时 %d 分" % (a, b)) if a else ("%d 分钟" % b)


def fc(sec):
    t = int(sec) // 60
    a, b = divmod(t, 60)
    return ("%d h %d min" % (a, b)) if a else ("%d min" % b)


CSS = """
:root{--ink:#111;--ink2:#555;--line:#b9b9b9;--strong:#111;--accent:#2a6249}
*{box-sizing:border-box}
body{margin:0;padding:28px 14px 60px;background:#efefed;color:var(--ink);
  font:13px/1.5 "Microsoft YaHei","PingFang SC",system-ui,sans-serif;-webkit-font-smoothing:antialiased}
.sheet{width:860px;max-width:100%;margin:0 auto;background:#fff;padding:44px 52px 40px;
  box-shadow:0 1px 2px rgba(0,0,0,.08),0 12px 32px rgba(0,0,0,.06)}
.head{text-align:center;margin-bottom:22px}
h1{margin:0 0 8px;font-size:22px;line-height:1.35}
.meta{margin:0;font-size:12.5px;color:var(--ink2)}
.hint{margin:8px 0 0;font-size:12px;color:#888}
.rule{height:2px;background:var(--ink);margin:18px 0 0}
.bar{display:flex;align-items:center;gap:14px;flex-wrap:wrap;margin:16px 0 14px;
  padding:10px 12px;border:1px solid #e2e2e2;background:#fafafa}
.stat{font-size:12.5px;white-space:nowrap}
.stat b{color:var(--accent)}
.track{flex:1 1 160px;height:6px;background:#e3e3e3;position:relative;min-width:120px}
.fill{position:absolute;left:0;top:0;bottom:0;width:0;background:var(--accent);transition:width .18s}
button{font:inherit;font-size:12px;padding:5px 12px;border:1px solid var(--ink);
  background:var(--ink);color:#fff;cursor:pointer;border-radius:2px}
button.ghost{background:#fff;color:var(--ink)}
table{width:100%;border-collapse:collapse;margin-top:4px;font-variant-numeric:tabular-nums}
col.c1{width:6%}col.c2{width:43%}col.c3{width:9%}col.c4{width:17%}col.c5{width:8%}col.c6{width:17%}
th{font-size:12px;padding:9px 5px;border:1px solid var(--strong);line-height:1.35}
td{border:1px solid var(--line);padding:7px 8px;vertical-align:middle}
td.c{text-align:center}
tbody tr.item:hover{background:#f7f7f7}
tr.ch td{border-top:1.6px solid var(--strong);border-bottom:1px solid var(--strong);
  padding:10px 8px;font-weight:700;font-size:13.5px}
.ch-meta{font-weight:400;color:var(--ink2);font-size:12px;margin-left:10px}
.ch-prog{float:right;font-weight:400;font-size:11.5px;color:#8a8a8a}
.ch-prog.has{color:var(--accent)}
tr.done td{color:#adadad}
.done-at{font-size:12px;color:var(--accent);font-variant-numeric:tabular-nums}
input[type=checkbox]{appearance:none;-webkit-appearance:none;width:15px;height:15px;margin:0;
  border:1.4px solid var(--ink);background:#fff;cursor:pointer;position:relative}
input[type=checkbox]:hover,input[type=checkbox]:checked{border-color:var(--accent)}
input[type=checkbox]:checked::after{content:"";position:absolute;left:4px;top:1px;width:4px;height:8px;
  border:solid var(--accent);border-width:0 1.6px 1.6px 0;transform:rotate(42deg)}
.foot{margin:16px 0 0;font-size:11px;line-height:1.7;color:#777}
@media(max-width:720px){body{padding:12px 0 40px}.sheet{padding:24px 16px}h1{font-size:18px}
  table{font-size:12px}td{padding:6px 5px}.ch-meta{display:block;margin:2px 0 0}.ch-prog{float:none;display:block}}
@media print{
  @page{size:A4 portrait;margin:14mm 12mm}
  :root{--accent:#111}
  body{background:#fff;padding:0;font-size:10.5pt}
  .sheet{width:auto;max-width:none;margin:0;padding:0;box-shadow:none}
  .no-print{display:none!important}
  h1{font-size:16pt}table{font-size:9.5pt}
  th{padding:6px 4px}td{padding:4.5px 5px}
  tbody tr{break-inside:avoid;page-break-inside:avoid}
  tr.ch{break-after:avoid;page-break-after:avoid}
  thead{display:table-header-group}
}
"""

JS = """
(function(){
  var KEY='checklist:'+CONFIG_ID;
  var rows=[].slice.call(document.querySelectorAll('tr.item'));
  var totalN=rows.length;
  var totalSec=rows.reduce(function(a,r){return a+parseInt(r.dataset.sec,10)},0);
  var statEl=document.getElementById('stat'),fillEl=document.getElementById('fill'),chEls={};
  function fs(s){var m=Math.floor(s/60),h=Math.floor(m/60),mm=m%60;
    return h?(h+' 小时 '+mm+' 分'):(mm+' 分钟')}
  function pad(n){return n<10?'0'+n:''+n}
  function now(){var d=new Date();return d.getFullYear()+'-'+pad(d.getMonth()+1)+'-'+
    pad(d.getDate())+' '+pad(d.getHours())+':'+pad(d.getMinutes())}
  function save(s){try{localStorage.setItem(KEY,JSON.stringify(s))}catch(e){}}
  var state={};try{state=JSON.parse(localStorage.getItem(KEY)||'{}')||{}}catch(e){state={}}
  function apply(){
    var dn=0,ds=0,pc={};
    rows.forEach(function(r,i){
      var rec=state[i]||{},box=r.querySelector('input[type=checkbox]'),when=r.querySelector('.done-at');
      box.checked=!!rec.done;
      when.textContent=rec.done?(rec.doneAt||''):'';
      r.classList.toggle('done',box.checked);
      var ch=r.dataset.ch;pc[ch]=pc[ch]||{n:0,d:0};pc[ch].n++;
      if(box.checked){dn++;ds+=parseInt(r.dataset.sec,10);pc[ch].d++}
    });
    statEl.innerHTML='已学 <b>'+dn+'</b> / '+totalN+' 讲 · <b>'+fs(ds)+'</b> / '+fs(totalSec);
    fillEl.style.width=(totalN?dn/totalN*100:0).toFixed(1)+'%';
    Object.keys(chEls).forEach(function(k){
      var p=pc[k]||{n:0,d:0};
      chEls[k].textContent=p.d+' / '+p.n;
      chEls[k].classList.toggle('has',p.d>0);
    });
  }
  rows.forEach(function(r,i){
    r.querySelector('input[type=checkbox]').addEventListener('change',function(){
      if(this.checked){state[i]={done:true,doneAt:now()}}else{delete state[i]}
      save(state);apply();
    });
  });
  [].forEach.call(document.querySelectorAll('[data-chprog]'),function(el){chEls[el.dataset.chprog]=el});
  document.getElementById('printBtn').addEventListener('click',function(){window.print()});
  document.getElementById('resetBtn').addEventListener('click',function(){
    if(!confirm('清空所有打勾记录和完成时间？'))return;
    state={};save(state);
    rows.forEach(function(r){r.querySelector('input[type=checkbox]').checked=false});
    apply();
  });
  apply();
})();
"""


def build_rows():
    out, seq, ci = [], 1, 0
    for title, items in CHAPTERS:
        cs = sum(x for _, x in items)
        out.append('<tr class="ch"><td colspan="6">%s'
                   '<span class="ch-meta">共 %d 讲 · %s</span>'
                   '<span class="ch-prog" data-chprog="%d">0 / %d</span></td></tr>'
                   % (h.escape(title), len(items), fs(cs), ci, len(items)))
        cum = 0
        for name, sec in items:
            cum += sec
            out.append('<tr class="item" data-ch="%d" data-sec="%d">'
                       '<td class="c">%d</td><td>%s</td><td class="c">%s</td>'
                       '<td class="c">%s</td>'
                       '<td class="c"><input type="checkbox" aria-label="第 %d 讲 完成"></td>'
                       '<td class="c"><span class="done-at"></span></td></tr>'
                       % (ci, sec, seq, h.escape(name), fd(sec), fc(cum), seq))
            seq += 1
        ci += 1
    return "\n".join(out)


meta = "UP主：%s%s（共 %d 讲，总时长 %s）" % (
    CONFIG["up"], ("｜" + CONFIG["subtitle"]) if CONFIG.get("subtitle") else "",
    TOTAL_N, fs(TOTAL_SEC))

doc = """<!DOCTYPE html>
<html lang="zh-CN"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{title}</title><style>{css}</style></head><body>
<div class="sheet">
<header class="head"><h1>{title}</h1>
<p class="meta">{meta}</p>
<p class="hint no-print">点“完成”列打勾，右侧会自动记下打勾时间；取消勾选则清除。进度保存在本机浏览器，可直接打印或另存为 PDF。</p>
<div class="rule"></div></header>
<div class="bar no-print">
<span class="stat" id="stat">已学 <b>0</b> / {n} 讲 · <b>0 分钟</b> / {total}</span>
<span class="track"><span class="fill" id="fill"></span></span>
<span><button id="printBtn">打印 / 存为 PDF</button>
<button id="resetBtn" class="ghost">清空记录</button></span>
</div>
<table><colgroup><col class="c1"><col class="c2"><col class="c3">
<col class="c4"><col class="c5"><col class="c6"></colgroup>
<thead><tr><th>编号</th><th>课程内容</th><th>时长</th>
<th>本章累计<br>（到本节上完）</th><th>完成</th><th>完成时间</th></tr></thead>
<tbody>
{rows}
</tbody></table>
<p class="foot">{note}</p>
</div>
<script>var CONFIG_ID={cid};{js}</script>
</body></html>""".format(title=h.escape(CONFIG["title"]), css=CSS,
                         meta=h.escape(meta), n=TOTAL_N, total=fs(TOTAL_SEC),
                         rows=build_rows(), note=h.escape(CONFIG["note"]),
                         cid="'%s'" % CONFIG["id"], js=JS)

out = os.path.join(os.path.dirname(os.path.abspath(__file__)),
                   CONFIG["stem"] + ".html")
with open(out, "w", encoding="utf-8") as f:
    f.write(doc)
print("HTML_OK:", out)
print("TOTAL:", TOTAL_N, "讲", TOTAL_SEC, "秒 =", fs(TOTAL_SEC))
```

## 踩过的坑

**PDF 分页**

1. `Spacer.wrap()` 返回高度 0 —— 取 `f.height`
2. `Paragraph` 还要计入 `getSpaceAfter()`（少算 10pt 就会超页）
3. 表格总宽必须 **≤ 版心宽**（A4 8.27in − 0.7in×2 = 6.87in），超一点就会被 reportlab 二次拆分，出现「下一页只剩一行」
4. 不要用「单行高度累加」估算分页，会低估；正确做法是把候选行拼成整表 `wrap()` 实测
5. 章标题行做高度测量时必须带 `SPAN`，否则文字被挤进窄列，高度虚高到 300pt
6. 断页时若该页最后一行是章标题，把它挪到下一页

**HTML**

7. `input[type=date]` 空值会显示 `dd/mm/yyyy` 占位符，要么隐藏占位符，要么干脆不用 date 控件（现在版本用纯文本 span，勾选自动写时间）
8. 打印时 `:root{--accent:#111}` 把墨绿压成黑，避免彩色打印出现杂色
9. `<thead>` 要 `display:table-header-group` 才会每页重复；`tr.ch{break-after:avoid}` 防章标题落在页脚
10. `localStorage` 的 key 用 `checklist:<BV号>`，不同课程互不覆盖

## 自检清单

- [ ] 秒数总和与接口 `data.duration` 一致（或差额正好等于被略去分P）
- [ ] 章节数、讲数、每章总时长三者自洽
- [ ] PDF 页数合理，每页最后一行不是章标题
- [ ] HTML 勾选 → 进度联动、时间写入；取消 → 时间清空
- [ ] 打印预览：无工具条、表头重复、无空白占位符
- [ ] 文件里没有本机路径、真实姓名、邮箱等隐私信息

## 回复用户时

- 附 `computer://` 链接给出 PDF / HTML
- 报核心数字：N 讲、总时长、页数，以及与接口总秒数的校验结果
- 说明章节是归纳的、哪些非教学分P被略去
- 若修过原始标题的错别字，明说改了哪个
