---
tags: [rag, 智能分块, docling, chunker, 父子分块, 组件选型]
created: 2026-09-21
updated: 2026-09-21
type: concept
---

# docling 原生分块器

> 解析已经用 docling 时，**不要绕道 Markdown + LangChain 去切分**。docling 自带完整的 chunker 族与父级回溯机制，把 `DoclingDocument` 对象直接交给它即可——"结构感知 + token 限制 + 父子回溯"三件事都是官方组件。

## 核心结论

1. **`HybridChunker` 就是「结构感知 + token 限制」的官方实现**，不需要自己拼两阶段切分器。
2. 它的超长兜底用的是 **semchunk 语义边界切分**，不是递归分隔符切分。
3. **父子分块的正规做法**：`HybridChunker` 切子块 + `TreeChunkExpander` 回溯父级，两者都是内置组件。
4. **标题链前置是内置的**：`chunker.contextualize(chunk)` 直接产出"带标题链、给 embedding 用"的文本。
5. 绕道 Markdown 会丢三样东西：**元素类型**（表格/图片的 label）、**元素引用**（可回溯到原文）、**阅读顺序**。

## 一、组件族清单

| 组件 | 作用 |
|---|---|
| `HierarchicalChunker` | 按文档结构（元素/标题）切块，**不做 token 限制** |
| **`HybridChunker`** | 在 Hierarchical 之上加 token 限制与碎块合并 —— **首选** |
| `LineChunker` | 按行切分 |
| `PageChunker` | 按页切分（依赖 `prov.page_no`，**DOCX 不可用**） |
| `CodeChunker` | 按代码语法切（tree-sitter 支持 Python / JS / TS / C） |
| `TreeChunkExpander` | 把块扩展回**顶层项**（父级回溯） |
| `PageChunkExpander` | 把块扩展回整页（**DOCX 不可用**） |

## 二、HybridChunker 的四步流程（源码级）

`chunk()` 主方法是四步，不是两步：

```
① _inner_chunker.chunk()        → HierarchicalChunker，按文档结构切
② _split_by_doc_items()         → 按 doc_items 滑动窗口聚合，累积到不超 max_tokens
③ _split_using_plain_text()     → 单个元素仍超长时，交给 semchunk 按语义边界切
④ _merge_chunks_with_matching_metadata() → 合并同一标题链下的相邻小块
```

`_inner_chunker` 是一个 `cached_property`，返回 `HierarchicalChunker` 实例——**所以 HybridChunker 确实是"结构感知打底 + token 约束"的两层结构**。

### 一处必须纠正的说法

网上（以及本库早期版本）常把 HybridChunker 描述为「结构感知 + 递归切分」。**第三刀实际是 `semchunk`**，依据是统计句子边界，不是分隔符优先级。

这个区别有实际影响：装 `docling-core[chunking]` 时就带着 semchunk，**你不需要再自己配中英混排的递归分隔符列表**。

### 两个默认开启的行为

| 参数 | 默认 | 作用 |
|---|---|---|
| `merge_peers` | `True` | 合并**同一标题链**下、合计仍不超 max_tokens 的相邻小块——**碎块瘟疫的自动解药** |
| `repeat_table_header` | `True` | 表格被切分时重复表头到每个分段 |

## 三、父子分块怎么落地

- **子块**：`HybridChunker.chunk(doc)`
- **父块**：`TreeChunkExpander.expand(chunk, dl_doc, serializer)` —— 从块内元素的 `self_ref` 沿文档树**向上遍历到 body 的直接子节点**，再按原始阅读顺序序列化

```python
from docling.document_converter import DocumentConverter, WordFormatOption
from docling.datamodel.base_models import InputFormat
from docling_core.transforms.chunker.hybrid_chunker import HybridChunker
from docling_core.transforms.chunker.chunk_expander import TreeChunkExpander

doc = DocumentConverter(
    format_options={InputFormat.DOCX: WordFormatOption()}
).convert("report.docx").document

# ① 子块：结构感知 + token 限制 + 碎块合并，一次拿到
chunker = HybridChunker(max_tokens=512)
children = list(chunker.chunk(doc))

# ② 检索用文本（自动带标题链）
index_texts = [chunker.contextualize(chunk=c) for c in children]

# ③ 父级回溯：命中子块后扩回它所属的顶层项
expander = TreeChunkExpander()
serializer = chunker.serializer_provider.get_serializer(doc=doc)
parent_chunk = expander.expand(chunk=children[0], dl_doc=doc, serializer=serializer)
```

### 两个必须知道的边界

1. **`TreeChunkExpander` 扩到的是「顶层项」**（body 的直接子节点），不是"任意上一级父块"。章节嵌套很深时，扩出来可能是整个大章节，需检查是否超 LLM 窗口。
2. **`PageChunkExpander` 依赖 `prov.page_no`**。DOCX 的位置信息是空列表（见 [[rag/01_多源数据接入/Word解析|Word 解析]] 的实测结论），所以 **DOCX 场景只有 Tree 这一条回溯路径**。

## 四、标题链前置是内置能力

`BaseChunker.contextualize(chunk)` 把 `meta` 中未被排除的字段与 `chunk.text` 序列化拼接，产出**专门给 embedding 用的文本**。

`DocMeta.excluded_embed` 排除了 `schema_name` / `version` / `doc_items` / `origin`，因此 **`headings` 会被保留并拼进文本**。

结论：**"把标题链拼进 chunk 文本"这件事不需要自己写**——这正是 [[rag/03_智能分块/结构感知分块|结构感知分块]] 里手工做的那一步，docling 已内置。

## 五、chunk 里到底有什么

| 字段 | 内容 |
|---|---|
| `chunk.text` | 块正文 |
| `chunk.meta.headings` | 标题链（如 `["RAG 检索系统设计", "性能对比"]`） |
| `chunk.meta.doc_items` | 块对应的原始元素列表（可回溯到 `self_ref`） |
| `chunk.meta.origin` | 文档来源信息 |

**没有显式 `parent_id`** —— 父子关系通过 `headings` 与 `doc_items` 表达，这也是 `TreeChunkExpander` 能工作的前提（它靠 `doc_items` 的 `self_ref` 向上遍历）。

## 六、表格的特殊处理

`repeat_table_header=True`（默认开）：当块内**只有一个 `TableItem`** 且超长时，表格按行切，**表头行作为前缀重复到每个分段**；`omit_header_on_overflow` 控制"表头本身导致溢出"时是否降级丢弃。

这比"表格整块不切"更实用——超长表不会撑爆 chunk，且每个分段都自带表头语义。取舍见 [[rag/03_智能分块/特殊内容分块|特殊内容分块]]。

## 七、依赖与 tokenizer

```bash
pip install "docling-core[chunking]"          # semchunk + transformers + tree-sitter
pip install "docling-core[chunking-openai]"   # 若要换 OpenAITokenizer（tiktoken）
```

**默认 tokenizer 是 `sentence-transformers/all-MiniLM-L6-v2`，需要访问 HuggingFace。** 国内网络的三个应对：

| 方案 | 做法 |
|---|---|
| 用镜像 | 设环境变量 `HF_ENDPOINT=https://hf-mirror.com` |
| 换 tokenizer | `OpenAITokenizer(tokenizer=tiktoken.get_encoding("cl100k_base"), max_tokens=512)` |
| 自定义 | 实现 `BaseTokenizer` 的三个方法（`count_tokens` / `get_max_tokens` / `get_tokenizer`），可用近似估算 |

> 注意：`HybridChunker(max_tokens=512)` 这种写法**仍会触发默认 HF tokenizer 加载**（`max_tokens` 被转发给 tokenizer 构造）。要完全避开 HuggingFace，必须显式传 `tokenizer=`。

## 八、陷阱

1. **`max_tokens` 是「含标题链」的总预算**。源码里 `available_length = max_tokens - other_len`，`other_len` 是标题与 caption 占用的 token。所以设 512 时，正文实际可用不到 512。
2. **忘了装 `docling-core[chunking]`** 会在运行时抛 `ImportError`（semchunk 缺失），而不是导入期。
3. **绕道 Markdown 再切**会丢掉元素类型与元素引用——那两个正是父级回溯与按类型分流的基础。
4. **`TreeChunkExpander` 可能扩得过大**：它扩到顶层项，深嵌套文档里单次扩展可能产出几千 token。
5. **`PageChunkExpander` 在 DOCX 上静默失败**（只发 warning 返回原块），因为 DOCX 没有页码。

## 九、若坚持用「结构感知 + 递归字符切分」组合

`HybridChunker` 的第二刀是 semchunk（语义边界），**不是递归字符切分**。如果需要的是**分隔符优先级完全可控**的递归切分，那就该走 LangChain 那一套。两者的差异如下：

| 维度 | 结构感知 + 递归字符（LangChain 组合） | 结构感知 + semchunk（docling HybridChunker） |
|---|---|---|
| 第一刀依据 | 标题标记（`#` / HTML 标签 / Word 样式） | 文档元素树（HierarchicalChunker） |
| 第二刀依据 | **分隔符优先级逐级降级** | 统计句子边界 |
| 分隔符可控性 | **高**（可针对中文标点定制优先级） | 低（黑盒） |
| 中英混排适配 | 需自己配中英文标点列表 | semchunk 自行处理 |
| 与 docling 集成 | 需先转 Markdown（**丢元素引用**） | 原生（保留 `doc_items`） |
| 碎块合并 | 需自己写后处理 | `merge_peers` 内置 |

**选择依据**：需要精确控制分隔符（尤其中文标点的切分优先级）→ 选 LangChain 组合；想省事、不介意黑盒 → 选 HybridChunker。

### 组件

| 环节 | 组件 |
|---|---|
| 第一刀（结构感知，切父块） | `MarkdownHeaderTextSplitter`（`langchain-text-splitters`） |
| 第二刀（递归字符，切子块） | `RecursiveCharacterTextSplitter` |
| 合并碎块 | 自己写（按 metadata 相同判断同父块） |

**关键 API**：第二刀必须用 `splitter.split_documents(sections)` 而非 `split_text(text)` —— 前者**继承 metadata**，后者会让第一刀提取的标题层级全部丢失。

### 与 docling 的衔接（关键取舍）

这套组合需要 **Markdown 文本**，而 docling 给 Markdown 有代价：

- `doc.export_to_markdown()` → 得到 Markdown，但**丢掉元素引用**（无 `self_ref`、无元素 label）
- 直接用元素树 → 保留引用，但不是 Markdown，喂不进 `MarkdownHeaderTextSplitter`

**实用方案是按内容类型分工**：

| 内容类型 | 处理方式 |
|---|---|
| **正文** | `export_to_markdown()` → Markdown 标题切分 → 递归字符切分（正文本来就是纯文本，丢引用影响最小） |
| **表格** | 直接走元素树取（见 [[rag/01_多源数据接入/表格提取与处理\|表格提取与处理]]） |
| **图片** | 直接走元素树取（见 [[rag/01_多源数据接入/图片提取与描述化\|图片提取与描述化]]） |

最后把三路结果合并为统一 chunk 列表，**共用一套 `chunk_id`**（否则混合检索时 RRF 融合无法对齐）。

**不要做的事**：把表格塞进 Markdown 再交给递归切分器——它会把表格切断，行列对应关系就废了。表格必须在切分前就摘出来。

### 父子关系怎么表达

第一刀切出的 `section` 就是父块，第二刀切出的 `chunk` 就是子块。父子的关联靠 **metadata 继承**：

- 子块继承父块的 `h1` / `h2` / `h3` 字段 → 标题链即"父标识"
- 若要显式的父块实体（检索子块后返回完整父块），需要在第一刀产出时给每个 section 分配 `parent_id`，并单独存一份父块表

> 对比：docling 路线用 `TreeChunkExpander` 做这一步，扩到的是**顶层项**；LangChain 路线需要自己维护父块表，但可以精确控制"扩到哪一级"。

## 相关导航

- 上游：[[rag/03_智能分块/文档分块策略|分块总览]] · [[rag/03_智能分块/结构感知分块|结构感知分块]]
- 同级：[[rag/03_智能分块/分块工具选型|分块工具选型]] · [[rag/03_智能分块/特殊内容分块|特殊内容分块]] · [[rag/03_智能分块/层级父子与句子窗口检索|层级父子与句子窗口检索]]
- 依赖的解析层：[[rag/01_多源数据接入/Word解析|Word 解析]]
