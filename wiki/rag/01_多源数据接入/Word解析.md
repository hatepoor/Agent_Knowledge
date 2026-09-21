---
tags: [rag, 数据接入, word, docx, docling, ooxml, 解析]
created: 2026-09-19
updated: 2026-09-19
type: concept
---

# Word 解析（DOCX 接入）

> `.docx` 是「ZIP 包 + OOXML」的数字文档，有完整文本层、不需要 OCR；用 docling 的 `MsWordDocumentBackend` 解析**不依赖任何机器学习模型**，图片在解析期就直接进元素层，导出 Markdown 时只剩占位符只是`image_mode` 的导出期开关在作怪。

## 核心结论

- **DOCX 本质是压缩包 + XML**：`word/document.xml` 是正文、`word/_rels/document.xml.rels` 是关系表、`word/media/` 下就是图片真实二进制。这决定了 Word 解析是「读 XML」，不是「认图片」，天然带文本层。
- **docling 的 Word 管线零模型**：`WordFormatOption` 走 `SimplePipeline` + `MsWordDocumentBackend`（基于 python-docx），不用 layout 模型、不下载权重。和 PDF 管线有本质区别。
- **图片提取没有任何开关**：网上广为流传的 `generate_picture_images=True` 只存在于 PDF 管线，`msword_backend.py` 里一次都没出现。DOCX 下 `pic.get_image(doc)` 必定返回 PIL 对象。
- **导出 Markdown 时图片「消失」是导出期行为，不是解析期丢图**：`export_to_markdown()` / `save_as_markdown()` 的 `image_mode` 默认是 `ImageRefMode.PLACEHOLDER`，改成 `REFERENCED` 或 `EMBEDDED` 即可落盘或内嵌。
- **装饰性图片已被自动过滤**：小面积、全透明、纯白的 spacer 落入 `ContentLayer.INVISIBLE`，`iterate_items()` 默认不遍历，不需要你自己写过滤。
- **DOCX 没有页码、没有坐标**：`PictureItem.prov == []`，「第 N 页」式溯源在 DOCX 场景做不到。可用定位标识只有 `self_ref`、标题链、文档流序号。
- **纯 DOCX 场景能跑在 ~50 MB 依赖上，无需 torch**：但要小心 `DocumentConverter` 的导入链会连带触发 PDF 后端，所以 `docling-parse` 与 `pypdfium2` 必须装。

## 机制与原理

### 1. DOCX 的本质：ZIP + OOXML

`.docx` 文件把一篇文档的所有部件打包进一个 ZIP。实测一个含 1 张图的 `.docx` 解包后可见：

| 路径 | 作用 |
|---|---|
| `word/document.xml` | 正文（段落、表格、图片引用的 XML） |
| `word/_rels/document.xml.rels` | 关系表（把 `document.xml` 里的 `r:embed="rId7"` 映射到具体媒体文件） |
| `word/media/image1.png` | 图片的**真实二进制**，文件头魔数为 `89504e47`（PNG 的 `‰PNG`） |

这个结构的关键含义：**文本和图片都是「原生数据」，不是从像素里认出来的**。所以 Word 解析不需要 OCR，只需要正确地从 XML 里读结构、从 `media/` 里读字节。

### 2. docling 的 Word backend：SimplePipeline + python-docx，无模型

docling 对每种格式用一个 `FormatOption` 描述「用哪条管线、哪个后端」。`WordFormatOption` 的关键事实：

- `pipeline_cls = SimplePipeline`——一条**不调用任何视觉/布局模型**的轻管线。
- `backend_cls = MsWordDocumentBackend`——底层基于 **python-docx** 读 OOXML。
- 与 PDF 管线（要渲染页面、跑 layout 模型、做 OCR）有**本质区别**：Word 直接读 XML，PDF 要先「把页面当图」。

> 正因为无模型，`MsWordDocumentBackend` 解析是确定性的、可重复的、零权重下载——这对个人知识库（数千 chunk、无 GPU 集群）是最友好的形态。

### 3. 图片提取：不需要任何开关

这是最常见的误解来源。很多博客写「要让 docling 提取 DOCX 图片，需设 `generate_picture_images=True`」——**这是错的**。

- `generate_picture_images=True` 这个参数**只出现在 PDF 管线**（`NativePdfFormatOption` 及相关的 PDF pipelines）中；在 `msword_backend.py` 里**全文零次出现**。
- Word backend 走的是 `_get_image_from_relationship()`：直接从关系表取出 `rel.target_part.blob`（图片二进制），用 **Pillow** 解码，最后以 `ImageRef.from_pil(pil_image, dpi=72)` 挂到 `PictureItem.image` 上。

结论：**DOCX 场景下 `pic.get_image(doc)` 必定返回 PIL 对象**——根本不需要、也没有任何开关来「开启图片提取」。

### 4. 为什么导出 Markdown 时图片只剩占位符

解析期其实已经把图片对象挂到了元素层，但 `DoclingDocument.export_to_markdown()` / `save_as_markdown()` 有个导出期参数 `image_mode`：

| `image_mode` 取值 | 行为 |
|---|---|
| `ImageRefMode.PLACEHOLDER`（**默认**） | 图片只留一个注释占位，正文里看不到图 |
| `ImageRefMode.REFERENCED` | 图片落盘到文件，Markdown 里写路径引用 |
| `ImageRefMode.EMBEDDED` | 图片以 base64 直接内嵌进 Markdown |

这是**导出期参数，不是解析期参数**——理解这件事是整个 Word 图片处理的关键。你在元素层始终能拿到完整的 PIL 图片；「Markdown 里看不到图」只是导出环节没选对模式，不代表解析阶段丢了图。

### 5. 装饰性图片已被自动过滤

Word backend 有 `_is_invisible_spacer()`，满足以下任一条件即判定为 spacer，放入 `ContentLayer.INVISIBLE`：

- 面积 ≤ **25 平方像素**；
- RGBA **全透明**（alpha 通道最大值为 0）；
- RGB **全白**（255, 255, 255）。

而 `iterate_items()` 默认只遍历 `{ContentLayer.BODY}`（即 `DEFAULT_CONTENT_LAYERS`）。所以 **logo、分隔线、占位图默认不会出现在遍历结果里**——不需要你再写一遍过滤逻辑。注意这只是「自动过滤了 spacer」，大尺寸装饰图（见下文）它过滤不掉。

### 6. `prov` 是空的——DOCX 没有页码和坐标

实测：`PictureItem.prov == []`（空列表）。

原因很直接：**DOCX 没有固定分页概念**，第几页、画在哪里是由渲染器（Word / WPS / 浏览器）在打开时才算出来的，docling 没有、也不提供定位坐标。

影响：

- **「第 N 页」这类溯源在 DOCX 场景做不到**。
- DOCX 下可用的定位标识只有三个：
  1. `self_ref`——形如 `#/pictures/0` 的元素内引用；
  2. **标题链**——遍历时你自己维护的「当前处于哪个章节下」；
  3. **文档流序号**——元素在遍历序列中的顺序位置。

对比 PDF：`PictureItem.prov` 里带有 `page_no`（页码）和 `bbox`（坐标框），溯源要强得多。设计 chunk 的溯源字段时，必须按「DOCX 无 page/bbox」来兜底（详见 [[rag/01_多源数据接入/元数据与溯源锚点|元数据与溯源锚点]]）。

### 7. 图注不会自动关联

`_add_picture_to_doc()` 在写入时**写死传 `caption=None`**，于是：

- `PictureItem.captions` 是空列表；
- `caption_text(doc)` 返回空串。

数据结构本身**支持**图注（PDF 管线会填 `captions`），只是 Word backend 不填。想要图注，只能自己从相邻段落提取（具体方法见 [[rag/01_多源数据接入/图片提取与描述化|图片提取与描述化]] 的「双向探测」）。

### 8. 其他边界情况

| 情况 | 行为 | 应对 |
|---|---|---|
| **外部链接的图片** | 判断 `rel.is_external`，只发 warning 并返回 `None` | 这类图取不到，需确认是否内嵌 |
| **EMF / WMF 格式**（Word 里粘贴的 Office 图表常见） | Pillow 读不了；docling 走兜底 `_convert_elements_via_docx`：临时生成 docx → 用 **LibreOffice** 转 PDF → 渲染成 PNG | **没装 LibreOffice 时这类图会得到 `None`** |
| **VML 图片**（Visio 等旧格式） | 走单独的处理路径 `_handle_vml_pictures` | 依赖该路径实现，结果质量「待核实」 |
| **同一位置多张图** | 会被包进一个 `GroupLabel.PICTURE_AREA` 组 | 遍历时注意组与单图的区别 |

> EMF/WMF 这条是纯 DOCX 轻依赖方案的最大暗坑：看似无模型、无需重量级依赖，但一旦文档里混了粘贴进来的 Office 图表（本质是 EMF），在无 LibreOffice 的环境里这些图会**静默变成 `None`**。

### 9. 最小依赖链（实测得出）

直接 `pip install docling` 会拉 `docling-slim[standard]`，里面含 `torch` / `torchvision` / `docling-ibm-models`（数百 MB）。但**纯 DOCX 场景不需要 torch**。实测最小可运行组合：

```
docling-slim[format-docx]    # 核心 + python-docx
docling-slim[convert-core]   # numpy / pillow / rtree / scipy
docling-parse                # 必须
pypdfium2                    # 必须
```

约 **50 MB**，无深度学习框架、无模型下载。

**但有个坑**：`from docling.document_converter import DocumentConverter` 这一句的导入链会连带触发 PDF 后端，所以即便你只处理 DOCX，`docling-parse` 和 `pypdfium2` 也必须装，缺一个就 `ModuleNotFoundError`。实测是按 `docling_parse → pypdfium2 → scipy` 的顺序逐个补齐才通过。

版本提醒：

- docling **2.129.0 起**包形态变为**元包 + `docling-slim` 实现包**，extras 拆分为 `format-docx` / `format-pdf` / `models-vlm-inline` / `feat-ocr-*` 等，安装写法以你实际版本为准。
- 旧版本（2.x 早期）的 `format_options` 可能是 **dict 形式**（`{"word": WordFormatOption(...)}`），与较新版本的 `FormatOption` 对象写法不同，需按版本核对。

## 实操

下面这段可运行代码展示：用 `DocumentConverter` + `WordFormatOption` 解析 DOCX，并**按元素类型分流**（正文/表格/图片分别处理），同时用 `REFERENCED` 模式把图片落盘。

```python
from docling.document_converter import DocumentConverter, ConversionResult
from docling.datamodel.base_models import InputFormat
from docling.document_converter import WordFormatOption
from docling_core.types.doc import DocItemLabel

conv = DocumentConverter(
    format_options={InputFormat.DOCX: WordFormatOption()}
)
res: ConversionResult = conv.convert("report.docx")
doc = res.document

for item, _ in doc.iterate_items():          # 默认只遍历 BODY 层
    if item.label == DocItemLabel.PICTURE:    # 图片：元素层直接取 PIL
        pil = item.get_image(doc)             # DOCX 下必返回 PIL 对象
        cap = item.caption_text(doc)          # DOCX 下恒为空串
        print("图片自_ref:", item.self_ref, "图注:", repr(cap))
    elif item.label == DocItemLabel.TABLE:
        df = item.export_to_dataframe()       # 表格转 DataFrame
    else:
        pass  # 正文段落等

md = doc.export_to_markdown(image_mode="REFERENCED")  # 图片落盘 + 路径引用
```

要点：

- `get_image(doc)` 在 DOCX 下永远返回 PIL，可直接喂视觉模型或落盘。
- `caption_text(doc)` 在 DOCX 下恒为空，别指望它；图注自己凑（见 [[rag/01_多源数据接入/图片提取与描述化|图片提取与描述化]]）。
- 想要图片进 Markdown，记得显式传 `image_mode="REFERENCED"` / `"EMBEDDED"`。

## 其他 Word 处理工具对比

docling 不是唯一选择，不同场景下各有取舍：

| 工具 | 定位 | 何时选它 | 局限 |
|---|---|---|---|
| **docling** | 推荐首选（自托管、MIT 许可） | 要一次性拿到结构化元素层（正文/表格/图片）、后续做结构感知分块与混合检索 | 纯 DOCX 也要带 `docling-parse`/`pypdfium2`；依赖比轻量库重 |
| **python-docx** | 底层 OOXML 直读 | 需要 **批注、修订记录、自定义 XML** 等 docling 覆盖不到的元数据；做精细字段级控制 | 自己写遍历逻辑，表格/图片结构化程度低，无 layout 概念 |
| **mammoth** | 轻量 DOCX→Markdown | 结构简单、只要「干净的 Markdown 正文」 | 不保留复杂结构、不擅长图片与精细表格，无 layout |
| **pandoc** | 通用格式转换 | 要**原样保留标题层级、列表、表格、图片引用**地转 Markdown/其他格式；老 `.doc` 也走这条 | 本地依赖（需装 pandoc 二进制），图片定位/描述需另接 |
| **MarkItDown**（微软） | 极轻量转 Markdown | 快速把 docx 丢进文本管线，结构要求极低 | **无 layout 能力**、图片/复杂表格弱（与 docling 对比见 [[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格\|解析总览]]） |

选型建议：

- 中文 + 英文混排、含表格与图片的 `.docx`，且目标是**混合检索（稠密 embedding + jieba/BM25 稀疏）** → 选 **docling**，因为它直接给你结构化的元素层，正文/表格/图片分流处理最顺。
- 只想要干净的 Markdown 正文、文档结构很简单 → mammoth / MarkItDown 更快。
- 需要读修订痕迹、批注这类元数据 → python-docx 直读。
- `.doc` 老格式 → 先 pandoc/LibreOffice 转 docx 再处理。

## 陷阱与边界

1. **以为要设 `generate_picture_images=True` 才能提取 DOCX 图片** —— 该参数只存在于 PDF 管线，Word backend 根本不认；DOCX 下图片默认就在元素层。
2. **从导出的 Markdown 里用正则抽图片** —— 默认 `image_mode=PLACEHOLDER`，Markdown 里只有占位注释，正则抽不到；要么改用 `REFERENCED`/`EMBEDDED`，要么直接回元素层 `get_image(doc)`。
3. **以为有页码、能溯源到「第 N 页」** —— DOCX 的 `prov` 为空，没有 page_no / bbox；定位只能用 `self_ref`、标题链、文档流序号。
4. **以为图注会自动带在 `PictureItem` 上** —— `_add_picture_to_doc()` 写死 `caption=None`，`captions` 恒空；图注必须自己从相邻段落识别。
5. **EMF/WMF 图在无 LibreOffice 时静默失败** —— 这类 Office 图表 Pillow 读不了，docling 兜底靠 LibreOffice 转 PDF 渲染；没装时得到 `None`，且往往不报错，容易被当成「图片本就缺失」。
6. **页眉页脚未剔除** —— 每页重复的页眉页脚会让 IDF 权重异常、污染 BM25（详见 [[rag/01_多源数据接入/多源数据接入\|多源数据接入]] 的六个坑）；解析阶段就该剥离或打标。
7. **装了 `docling-slim[format-docx]` 却报 `ModuleNotFoundError`** —— `DocumentConverter` 导入链会触发 PDF 后端，必须补齐 `docling-parse` 与 `pypdfium2`，否则即使只处理 DOCX 也跑不起来。
8. **依赖 docling VLM 能力处理 DOCX 图片描述** —— Word backend 只负责「取到图」，描述化要靠你接视觉大模型（见 [[rag/01_多源数据接入/图片提取与描述化|图片提取与描述化]]），docling 本身不给图片写描述。

## 相关导航

- 总览：[[rag/01_多源数据接入/多源数据接入|多源数据接入]] · 解析总览：[[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格|解析总览]]
- 兄弟页：[[rag/01_多源数据接入/PDF解析|PDF 解析]]（PDF 有 page_no/bbox，与 DOCX 的空 `prov` 不同）
- 下游必读：[[rag/01_多源数据接入/图片提取与描述化|图片提取与描述化]]（图注识别、上下文组装、描述生成）
- 表格处理：[[rag/01_多源数据接入/表格提取与处理|表格提取与处理]]（表格可进正文流、图片不能照搬）
- 溯源兜底：[[rag/01_多源数据接入/元数据与溯源锚点|元数据与溯源锚点]]（DOCX 无页码的替代定位方案）
- 切片衔接：[[rag/03_智能分块/混合索引的分块策略|混合索引的分块策略]]
