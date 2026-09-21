---
tags: [rag, 数据接入, PDF解析, ocr, 表格重建, 多栏, 溯源]
created: 2026-09-19
updated: 2026-09-19
type: concept
---

# PDF 解析

> PDF 解析是 RAG 数字文档接入里最容易「看起来成功、实则翻车」的一环：文本抽出来了、索引建好了，但多栏串行、表格被拍平、页码页眉混入正文——这些缺陷下游完全救不回，却往往到用户提问时才暴露。

## 核心结论

- **先判断文本层是否存在**，再决定走文本层解析还是 OCR/视觉层。这一步错了，后面全错。
- **多栏阅读顺序是 PDF 最典型的失败模式**：按坐标顺序抽，会把左右栏文字串行，语义彻底断裂。
- **纯 OCR 只产出无布局文字**，必须配 layout 分析才能还原段落与标题；扫描件优先用「转 Markdown」的管线（MinerU / Marker / VLM），不是裸 OCR。
- **PDF 的「页」是真实对象**，`page_no` 和 `bbox` 坐标天然可做溯源锚点——这是它相对 DOCX 的独特优势。

## 机制与原理

### 1. 数字版 vs 扫描版：文本层分流

PDF 分两类，处理方式根本不同：

| 类型 | 特征 | 解析路线 |
|------|------|----------|
| 数字版（有文本层） | 内部存有可提取字符 | 直接读文本+结构（PyMuPDF、PDFPlumber） |
| 扫描版（无文本层） | 本质是图片 | OCR / 转 Markdown 管线 |

**如何检测文本层**——不要靠肉眼看，用程序判定：

- **抽取字符数**：用 PyMuPDF 提取整页文本，字符数过少（如 < 50 字/页）基本是扫描件。
- **字符密度**：字符数 / 页面面积（如 72 DPI 下的像素面积）。密度低于经验阈值（例如约 0.5 字符/平方英寸，具体阈值随文档类型而定，待核实）倾向扫描件。
- **字体与编码检查**：数字版通常含 `/Font` 资源与 ToUnicode 映射；扫描版往往没有。

```python
import fitz  # PyMuPDF
doc = fitz.open("doc.pdf")
for i, page in enumerate(doc):
    text = page.get_text("text")
    chars = len(text.strip())
    area = page.rect.width * page.rect.height
    density = chars / area * 72 * 72  # 字符/平方英寸(72dpi)
    print(i, "chars=", chars, "density=", round(density, 3))
    # density 过低 -> 当作扫描件走 OCR / 转 Markdown 管线
```

### 2. 多栏版式的阅读顺序问题

两栏（或多栏）PDF 是重灾区。PDF 内部字符只带坐标 `(x, y)`，解析器若**按坐标顺序**依次抽取，会把左栏下半部分和右栏上半部分连成一句，语义完全断裂。

正确的阅读顺序需要：

- **按版式分栏**：先用 layout 模型（或基于列边界的启发式）把页面切成栏区块；
- **栏内按阅读序**：每栏内部再从上到下、各栏从左到右拼接；
- **跨栏标题处理**：跨栏的大标题要能被识别为「不属于任一栏」，避免插进正文中间。

> 这是为什么「先试文本层」之后，遇到多栏 PDF 仍可能需要带 layout 能力的解析器（docling / Marker / MinerU），而不是裸 PyMuPDF。

### 3. OCR 选型

纯 OCR 把图片变成一串文字，**没有段落、没有标题、没有表格结构**。必须配合 layout 分析才能还原。

| 引擎 | 中文 | 英文 | 特点 |
|------|------|------|------|
| **PaddleOCR** | 强 | 良好 | 中文场景首选，模型丰富 |
| **RapidOCR** | 良好 | 良好 | 轻量 ONNX 版，依赖少、易部署 |
| **Tesseract** | 弱 | 强 | 老牌，英文/拉丁语系好，中文需额外模型且弱 |

> 关键认知：**OCR ≠ 文档解析**。纯 OCR 管线（尤其扫描件里的表格、多栏）产出的是无结构文本流，需要再接 layout 模型或转 Markdown 管线才能用。个人知识库里，扫描件直接走 MinerU/Marker 这类「一步出 Markdown」的方案通常更省事。

### 4. 表格重建：TableFormer 类模型

表格最难的不是「看到格子」，而是**重建行列关系与多级表头**——尤其合并单元格（跨行/跨列）、嵌套表头。

TableFormer（docling 采用的模型）的思路是把表格解析建模为**预测表格分隔符的排列（permutation）问题**：不是简单地按视觉线切格子，而是联合建模行/列的逻辑关系，从而正确处理跨单元格合并与层次表头（具体实现细节以官方论文为准，待核实）。

`table_mode` 的取舍：

| 模式 | 精度 | 速度 | 适用 |
|------|------|------|------|
| `accurate` | 高 | 慢 | 复杂合并单元格、财务/学术表 |
| `fast` | 较低 | 快 | 结构规整、批量预处理 |

```python
from docling.document_converter import DocumentConverter
from docling.datamodel.pipeline import TableModel
conv = DocumentConverter(
    table_model=TableModel(mode="accurate")  # 复杂中文表用 accurate
)
res = conv.convert("report.pdf")
for t in res.document.tables:
    df = t.export_to_dataframe()  # 保留行列与表头的 DataFrame
```

### 5. 数学公式与图注（caption）

- **公式**：数字 PDF 里的公式常是矢量路径或内嵌图片，直接抽文本会变成乱码/点阵。需要公式识别（nougat、Mathpix 类）转成 LaTeX 文本才能被检索到语义。个人知识库若公式不多，可先放过低优先级处理。
- **图注（caption）**：图片下方的 "Figure 3 ..." 文字是图片语义的关键，解析时应把 caption 与图片**绑定**作为描述来源，而不是当成正文碎片丢掉。详见 [[rag/01_多源数据接入/图片提取与描述化|图片提取与描述化]]。

### 6. PDF 与 DOCX 的一个关键差异

> **PDF 的「页」是真实存在的对象；DOCX 没有。**

这意味着：

- PDF 每个 `page` 有独立的 `page_no`，字符/表格/图片都有 `bbox` 坐标（x1,y1,x2,y2）。这些坐标可直接做**溯源锚点**——回答时引用「第 3 页第 2 张表」，用户能精确回查。
- DOCX 是流式文档，没有「页」的概念（页码是渲染时才算出来的）。所以 DOCX 的溯源只能到「章节/段落」，到不了「页」。这一点在 [[rag/01_多源数据接入/Word解析|Word 解析]] 与 [[rag/01_多源数据接入/元数据与溯源锚点|元数据与溯源锚点]] 中展开。

对混合检索（稠密向量 + jieba/BM25 稀疏）来说，PDF 的坐标元数据还能用来做「同页 chunk 归并」「按 bbox 过滤无关区域」，是结构红利的一部分。

## 实操

一个最小可用的 PDF 接入判断流程：

```python
def route_pdf(path):
    doc = fitz.open(path)
    total_chars = sum(len(p.get_text("text").strip()) for p in doc)
    # 粗略分流：平均字符数过低 -> 扫描件
    if total_chars / max(len(doc), 1) < 50:
        return "ocr_or_markdown_pipeline"   # MinerU / Marker
    return "text_layer"                      # PyMuPDF / docling
```

- 文本层路线：用 docling 一次性拿 Markdown + 结构化 JSON（含表格、坐标）。
- 扫描件路线：先用 MinerU/Marker 转 Markdown，再对其中的表格用 docling 的 `accurate` 模式补精度（可选）。

## 陷阱与边界

1. **多栏串行**：两栏 PDF 按坐标顺序抽，左右栏文字被串成一句，语义断裂。→ 必须带 layout 分栏。
2. **表格被拍平**：表格转成纯文本行，行列关系永久丢失，LLM 会自信答错数字。→ 走 docling 的 TableFormer 或保留 Markdown 表结构。
3. **页码/页眉/页脚混入正文**："第 3 页""公司机密"等被当成正文进索引，污染检索。→ 解析阶段就剥离，或建索引时按 bbox 区域过滤。
4. **扫描件无文本层却硬抽**：对扫描件用 `get_text()` 拿到空字符串还当成功，建出空索引。→ 先检测文本层再分支。
5. **公式变乱码**：公式被抽成点阵或路径，进索引后完全不可检索。→ 公式识别或标记跳过。
6. **坐标漂移 / 旋转页**：扫描时页面倾斜，bbox 偏差导致溯源锚点不准。→ 必要时先做去旋转（deskew）预处理。

## 相关导航

- 总览与选型：[[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格|文档解析总览]]
- 表格专题：[[rag/01_多源数据接入/表格提取与处理|表格提取与处理]]
- 溯源设计：[[rag/01_多源数据接入/元数据与溯源锚点|元数据与溯源锚点]]
- Word 对比：[[rag/01_多源数据接入/Word解析|Word 解析]]
- 下游：[[rag/03_智能分块/混合索引的分块策略|混合索引的分块策略]]
