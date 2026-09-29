---
tags: [docling, 结构, 核心]
created: 2026-09-21
updated: 2026-09-21
type: concept
---

> `DoclingDocument` 是 Docling 全库的核心数据模型：一份文档经过解析后，所有内容都落进这棵带 `self_ref`、带 `prov`、分层 `content_layer` 的树里。理解它是理解后续一切处理的前提。

## 核心结论

1. 文档是一棵树：`body` 是根，子节点是各类 `DocItem`；`GroupItem` 用于把若干元素聚合成区域（如 `PICTURE_AREA`、`LIST`）。
2. 每个元素靠 `self_ref`（形如 `#/pictures/0`）唯一定位，靠 `parent`/`children` 的 `RefItem` 双向串联，可向上回溯父级、向下遍历子级。
3. **DOCX 的 `prov` 是空列表**——没有页码、没有坐标；PDF 才有 `page_no` 与 `bbox`。这是 DOCX 与 PDF 在结构层最本质的差异之一。
4. `iterate_items()` 默认只遍历 `ContentLayer.BODY` 层，装饰图等被归入 `INVISIBLE` 层的元素**不会**被返回，必须先知晓这一点。

## 机制与原理

### 文档树模型

`DoclingDocument` 顶层持有 `body`（一个 `NodeItem`），它是所有内容的根容器。树的节点分为两类：

- **`DocItem`**（内容节点，可叶子）：携带 `label`、`prov`、`source`、`comments` 等。再细分为 `TextItem` 系、`PictureItem`、`TableItem`、`CodeItem`、`KeyValueItem`、`FormItem`、各类 `Field*` 项。
- **`GroupItem`**（容器节点，非叶子）：只负责聚合，自身不含正文，靠 `name` + `label`（GroupLabel）描述区域语义，如 `LIST`、`PICTURE_AREA`、`CHAPTER`、`SECTION`、`SLIDE`、`FORM_AREA`、`KEY_VALUE_AREA`。

`FloatingItem`（`PictureItem`/`TableItem`/`CodeItem`/`KeyValueItem`/`FormItem` 的父类）额外带 `captions`、`references`、`footnotes`、`image`，用于表示「浮动元素」及其图注/引用关系。

### 元素类型完整清单（源码核实）

下表为 `docling_core` 中实际定义的 `DocItem` 子类（已对照源码逐项确认）：

| 类型 | 典型 label | 含义 | 典型来源 |
|---|---|---|---|
| `TextItem` | `PARAGRAPH` / `TEXT` / `CAPTION` / `FOOTNOTE` / `PAGE_HEADER` / `PAGE_FOOTER` / `REFERENCE` / `MARKER` / `CHECKBOX_SELECTED` / `CHECKBOX_UNSELECTED` / `FIELD_KEY` / `FIELD_HINT` / `HANDWRITTEN_TEXT` / `EMPTY_VALUE` | 纯文本单元（label 由判别字段区分） | 段落、脚注、页眉页脚、复选框等 |
| `TitleItem` | `TITLE` | 文档/章节标题 | 标题行 |
| `SectionHeaderItem` | `SECTION_HEADER` | 小节标题，带 `level` | 各级标题 |
| `ListItem` | `LIST_ITEM` | 列表项，带 `enumerated`/`marker` | 有序/无序列表 |
| `FormulaItem` | `FORMULA` | 公式 | 公式块 |
| `CodeItem` | `CODE` | 代码块，带 `code_language` | 代码段 |
| `PictureItem` | `PICTURE` / `CHART` | 图片/图表（浮动） | 插图、照片、图表 |
| `TableItem` | `TABLE` / `DOCUMENT_INDEX` | 表格（浮动），含 `data` | 表格 |
| `KeyValueItem` | `KEY_VALUE_REGION` | 键值区域（浮动），含 `graph` | 表单键值对 |
| `FormItem` | `FORM` | 表单（浮动），含 `graph` | 填报表单 |
| `FieldRegionItem` | `FIELD_REGION` | 表单字段区域 | 表单 |
| `FieldHeadingItem` | `FIELD_HEADING` | 字段标题（TextItem 子类） | 表单标签 |
| `FieldItem` | `FIELD_ITEM` | 表单字段 | 表单字段 |
| `FieldValueItem` | `FIELD_VALUE` | 字段值（TextItem 子类） | 表单取值 |
| `GroupItem` 及其子类 | `GroupLabel`（如 `LIST`/`PICTURE_AREA`/`CHAPTER`） | 聚合容器，非叶子 | 逻辑区域 |

> 注：`label` 是判别字段（discriminator），`ContentItem` 联合类型正是由 `label` 决定具体子类，所以同一 `TextItem` 类承载了十几种语义不同的文本。

### 层级与顺序

`doc.iterate_items()` 返回一个生成器，每项是一个元组 `(item, level)`：

- `item`：树中的节点（`NodeItem` 子类）。
- `level`：该节点在树中的深度（`len(stack)`，`body` 为 0，其直接子为 1，依此类推）。

**文档顺序即阅读顺序**：遍历按树的前序（parent→children）展开，输出的先后就是文档的自然阅读顺序。这让「按顺序切片」「定位第 N 个元素」变得直接。

### prov（位置信息）

`prov` 是 `ProvenanceItem` 列表，单个 `ProvenanceItem` 的字段（源码核实）：

- `page_no: int` —— 页码。
- `bbox: BoundingBox` —— 该元素在页面上的包围盒。
- `charspan: CharSpan` —— 在原文档文本中的字符区间。

**实测结论：DOCX 的 `prov` 是空列表**。原因是 Word 没有固定分页概念，`MsWordDocumentBackend` 不会填入页码与坐标。因此：
- 对 DOCX，`DocItem.get_image(doc)`、`get_location_tokens()` 这类依赖 `prov` 的方法无效（返回 `None`/空串）。
- DOCX 的溯源只能靠 `self_ref`、标题链（`parent` 回溯到 `TitleItem`/`SectionHeaderItem`）或文档流中的序号定位，无法精确到页与坐标。PDF 则三者齐备。

### content_layer（内容层）

`ContentLayer` 枚举（源码核实）取值：

| 层 | 含义 |
|---|---|
| `BODY` | 文档正文（主内容） |
| `FURNITURE` | 页眉/页脚等版式家具 |
| `BACKGROUND` | 水印等背景 |
| `INVISIBLE` | 隐藏/不可见文本（如装饰图） |
| `NOTES` | 作者/演讲者批注、修订 |

`DEFAULT_CONTENT_LAYERS = {ContentLayer.BODY}`。装饰图（面积极小 ≤25 平方像素、全透明、纯白）会被自动过滤进 `ContentLayer.INVISIBLE`。由于 `iterate_items()` 默认只遍历 `BODY` 层，**INVISIBLE 的元素默认不会出现在遍历结果里**——这对「为什么我的装饰图标找不到」是最常见解释。

### 引用机制

- **`self_ref: str`**：节点自身的 JSON Pointer，如 `#/pictures/0`、`#/texts/3`，是文档内唯一地址。
- **`RefItem`**（别名 `$ref`）：用 `cref` 指向另一个节点；`resolve(doc)` 把它解析成真实对象（按 `#/pictures/0` 三段切分定位到 `doc.pictures[0]`）。
- **`parent` / `children`**：`NodeItem` 持有 `parent: RefItem | None` 和 `children: list[RefItem]`，构成双向链接。向上回溯父级就是沿 `parent.resolve(doc)` 一直走到 `body`；`FineRef`（带 `range`）还能表达细粒度 span。

这套机制是「父级回溯」「图注关联」「表格单元格反查」的基础。

### 遍历 API 全解

`iterate_items()` 签名（源码核实）：

```python
def iterate_items(
    self,
    root: NodeItem | None = None,          # 从哪个节点开始；默认 doc.body
    with_groups: bool = False,             # 是否把 GroupItem 也 yield 出来（默认只 yield DocItem）
    traverse_pictures: bool = False,       # 是否深入 PictureItem 的子节点（默认只返回其 caption，跳过其余子节点）
    page_no: int | None = None,            # 只返回 prov.page_no 匹配该页的元素（DOCX prov 为空，过滤后为空）
    included_content_layers: set[ContentLayer] | None = None,  # 指定遍历哪些层；默认 DEFAULT_CONTENT_LAYERS={BODY}
    _level: int = 0,                       # 已废弃的内部参数
) -> Iterable[tuple[NodeItem, int]]
```

要点：
- 默认 `with_groups=False`：只返回内容节点，跳过 `GroupItem`。
- 默认 `traverse_pictures=False`：遇到 `PictureItem` 只保留其 `captions`，其余子节点（如分类/描述注释）不展开。
- `page_no` 过滤依赖 `prov`；DOCX 因 `prov` 为空会被全部过滤掉，所以 DOCX 不要用 `page_no` 过滤。
- 想拿到 INVISIBLE 层（如装饰图），必须显式传 `included_content_layers={ContentLayer.BODY, ContentLayer.INVISIBLE}`。

### 导出 API

文档级（`DoclingDocument`）序列化方法（源码核实）：

| 方法 | 作用 |
|---|---|
| `export_to_dict()` | 导出为 Python dict（结构化全集） |
| `save_as_json(filename)` | 落地为 JSON 文件 |
| `save_as_yaml(filename)` | 落地为 YAML 文件 |
| `export_to_markdown(...)` | 导出 Markdown 字符串（默认 `image_mode=PLACEHOLDER`） |
| `save_as_markdown(filename, ...)` | 落地为 Markdown（`image_mode`、`artifacts_dir` 等） |
| `export_to_html(...)` | 导出 HTML 字符串 |
| `save_as_html(filename)` | 落地为 HTML |
| `export_to_doctags(...)` | 导出 DocTags 字符串（旧名 `export_to_document_tokens` 已废弃） |
| `save_as_doctags(filename)` | 落地为 DocTags |

元素级另有：`TableItem.export_to_dataframe()` / `export_to_otsl()`，`PictureItem.export_to_markdown/html`，`TextItem`/`SectionHeaderItem` 等的 `export_to_doctags()`。

## 实操

下面这段遍历代码直观展示结构：打印每个元素的「层级 + 类型 + 标题链 + 页码」，覆盖阅读顺序与溯源两件事。

```python
from docling_core.types.doc.items.text import TitleItem, SectionHeaderItem

doc = result.document

def heading_chain(doc, item):
    """沿 parent 向上收集标题，得到该元素的章节路径。"""
    chain, parent = [], item.parent
    while parent is not None:
        p = parent.resolve(doc)
        if isinstance(p, (TitleItem, SectionHeaderItem)):
            chain.append(p.text)
        parent = p.parent
    return " > ".join(reversed(chain))

for item, level in doc.iterate_items():
    label = item.label.value if hasattr(item, "label") else type(item).__name__
    page = item.prov[0].page_no if getattr(item, "prov", None) else "-"
    print(f"L{level} | {label:14} | 页码:{page} | 标题链:{heading_chain(doc, item)}")
```

若想连装饰图也看到，把遍历改成：

```python
from docling_core.types.doc.common.content_layer import ContentLayer
for item, level in doc.iterate_items(
    included_content_layers={ContentLayer.BODY, ContentLayer.INVISIBLE}
):
    ...
```

## 陷阱与边界

- **以为 DOCX 有页码**：DOCX 的 `prov` 实测为空列表，所有依赖 `prov.page_no`/`bbox` 的能力（取页图、定位坐标）对 DOCX 图片/文本无效；PDF 才有。
- **以为 `iterate_items()` 返回所有元素**：默认 `DEFAULT_CONTENT_LAYERS={BODY}`，`INVISIBLE` 层（装饰图等）被过滤；要拿到它们必须显式传 `included_content_layers`。
- **`with_groups=False` 看不到容器**：区域聚合（`PICTURE_AREA`、`LIST`）默认不出现在遍历里，做区域分析时要开 `with_groups=True`。
- **`page_no` 过滤对 DOCX 失效**：因 `prov` 为空，传 `page_no` 会过滤掉全部 DOCX 元素。
- **图注不会自动关联 DOCX 图片**：Word backend 构造图片时传 `caption=None`，故 `PictureItem.captions` 为空、`caption_text(doc)` 返回空串；别指望 DOCX 图片自带图注。

## 相关导航

- 先看怎么得到这棵树：[[docling/03_快速上手_最小可用示例|快速上手：最小可用示例]]
- 再学怎么配置解析出这棵树：[[docling/04_进阶用法_配置与选项|进阶用法：配置与选项]]
