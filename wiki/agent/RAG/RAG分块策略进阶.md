---
tags: [concept, RAG, 分块, chunking, 策略, 进阶]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 分块策略进阶

> 分块（Chunking）是 RAG 系统中最影响检索质量的环节。合理的分块策略能让检索准确率提升 50% 以上。本文深入讲解各种高级分块技术。

## 分块的核心挑战

| 挑战 | 说明 | 影响 |
|------|------|------|
| 语义断裂 | 在句子/段落中间切分 | 丢失上下文，检索质量下降 |
| 粒度权衡 | 块太大稀释语义，块太小丢失上下文 | 检索精度与完整性的平衡 |
| 结构破坏 | 破坏表格、列表、代码块的完整性 | 结构化信息无法正确检索 |
| 重叠控制 | 重叠太多增加冗余，太少丢失连贯性 | 存储成本与检索质量的权衡 |

## 基础分块策略

### 1. 固定大小分块

```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n",
    chunk_size=1000,
    chunk_overlap=200,
)
```

**优点**：简单快速，适合结构化文本（日志、CSV）
**缺点**：可能在句子中间切断，破坏语义

### 2. 递归字符分块（推荐首选）

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", "。", "！", "？", ".", " ", ""],  # 中文优化
)
```

**原理**：按优先级尝试多种分隔符，优先在段落边界切分，其次句子边界，最后字符边界。

**优点**：保持语义完整性，通用性强
**缺点**：对复杂结构（表格、代码）支持有限

### 3. 语义分块

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()
splitter = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)
```

**原理**：计算相邻句子的语义相似度，在相似度骤降处切分。

**优点**：语义连贯性最好
**缺点**：需要嵌入模型，计算成本高，速度慢

## 高级分块策略

### 4. Parent-Child 分块（父子分块）

**核心思想**：小块用于精确检索，大块用于提供完整上下文。

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma

# 父分块器：大块，提供完整上下文
parent_splitter = RecursiveCharacterTextSplitter(
    chunk_size=2000,
    chunk_overlap=200,
)

# 子分块器：小块，用于精确检索
child_splitter = RecursiveCharacterTextSplitter(
    chunk_size=400,
    chunk_overlap=50,
)

# 向量库存储子块
vectorstore = Chroma(
    collection_name="parent_child",
    embedding_function=embeddings,
)

# 文档存储存储父块
store = InMemoryStore()

# 创建检索器
retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 添加文档
retriever.add_documents(documents)

# 检索时：用子块匹配，返回父块
results = retriever.invoke("什么是 RAG？")
# 返回的是 2000 字符的父块，而非 400 字符的子块
```

**适用场景**：
- 长文档需要完整上下文
- 检索精度要求高
- 文档结构层次分明

### 5. 句子窗口分块（Sentence Window）

**核心思想**：检索单个句子，但返回包含前后 N 个句子的窗口。

```python
from llama_index.core.node_parser import SentenceWindowNodeParser

# 创建句子窗口解析器
parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,           # 前后各 3 个句子
    window_metadata_key="window",
    original_text_metadata_key="original_text",
)

# 解析文档
nodes = parser.get_nodes_from_documents(documents)

# 检索时返回窗口
from llama_index.core.indices.postprocessor import MetadataReplacementPostProcessor

postprocessor = MetadataReplacementPostProcessor(
    target_metadata_key="window"
)

query_engine = index.as_query_engine(
    node_postprocessors=[postprocessor]
)
```

**优点**：检索粒度细（句子级），上下文完整（窗口级）
**缺点**：需要 LlamaIndex，存储开销增加

### 6. Markdown 感知分块

**核心思想**：按 Markdown 结构（标题层级）分块，保持文档结构。

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

# 定义标题层级
headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on,
    strip_headers=False,
)

# 分块
chunks = splitter.split_text(markdown_text)

# 每个 chunk 包含：
# - page_content: 该章节的文本
# - metadata: {"Header 1": "...", "Header 2": "...", ...}
```

**适用场景**：Markdown 文档、技术文档、知识库文章

### 7. 代码感知分块

**核心思想**：按代码结构（函数、类、模块）分块。

```python
from langchain_text_splitters import (
    PythonCodeTextSplitter,
    RecursiveCharacterTextSplitter,
)

# Python 代码分块
python_splitter = PythonCodeTextSplitter(
    chunk_size=1500,
    chunk_overlap=200,
)

# 按函数/类边界切分
chunks = python_splitter.split_text(python_code)

# 通用代码分块（支持多种语言）
code_splitter = RecursiveCharacterTextSplitter.from_language(
    language="python",  # "javascript", "java", "go", etc.
    chunk_size=1500,
    chunk_overlap=200,
)
```

### 8. 表格感知分块

**核心思想**：保持表格完整性，将表格作为独立块。

```python
from unstructured.partition.html import partition_html

# 使用 unstructured 库解析 HTML 表格
elements = partition_html(filename="document.html")

# 分离表格和文本
from unstructured.chunking.title import chunk_by_title

chunks = chunk_by_title(
    elements,
    max_characters=2000,
    combine_text_under_n_chars=1000,
)

# 表格元素保持完整
for chunk in chunks:
    if chunk.category == "Table":
        # 表格作为独立块
        pass
```

## 分块策略选择指南

| 文档类型 | 推荐策略 | 理由 |
|---------|---------|------|
| 普通文本/文章 | RecursiveCharacterTextSplitter | 通用性强，语义保持好 |
| Markdown 文档 | MarkdownHeaderTextSplitter | 保持标题层级结构 |
| 长文档/书籍 | ParentDocumentRetriever | 小块检索 + 大块上下文 |
| 代码文件 | PythonCodeTextSplitter | 按函数/类边界切分 |
| 技术文档（含表格） | 表格感知分块 | 保持表格完整性 |
| 高精度要求 | SemanticChunker + Parent-Child | 语义分块 + 父子检索 |
| 快速原型 | CharacterTextSplitter | 简单快速 |

## 分块参数调优

### chunk_size 选择

| 场景 | 推荐 chunk_size | 理由 |
|------|----------------|------|
| FAQ 问答 | 200-500 | 问题通常对应短答案 |
| 知识检索 | 500-1000 | 平衡精度与上下文 |
| 文档摘要 | 1000-2000 | 需要完整段落 |
| 代码检索 | 1000-1500 | 保持函数完整性 |

### chunk_overlap 选择

**经验法则**：chunk_overlap = chunk_size × 10%-20%

| chunk_size | 推荐 overlap | 说明 |
|-----------|-------------|------|
| 500 | 50-100 | 轻量重叠 |
| 1000 | 100-200 | 标准重叠 |
| 2000 | 200-400 | 重度重叠 |

### 中文分块特殊考虑

```python
# 中文分隔符优化
chinese_separators = [
    "\n\n",      # 段落
    "\n",        # 换行
    "。",        # 句号
    "！",        # 感叹号
    "？",        # 问号
    "；",        # 分号
    "，",        # 逗号（谨慎使用）
    " ",         # 空格
    "",          # 字符级（最后手段）
]

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=chinese_separators,
)
```

## 分块质量评估

### 评估指标

```python
def evaluate_chunking_quality(chunks):
    """评估分块质量"""
    metrics = {
        "avg_length": sum(len(c.page_content) for c in chunks) / len(chunks),
        "min_length": min(len(c.page_content) for c in chunks),
        "max_length": max(len(c.page_content) for c in chunks),
        "completeness": check_sentence_completeness(chunks),  # 句子完整性
        "coherence": check_semantic_coherence(chunks),        # 语义连贯性
    }
    return metrics

def check_sentence_completeness(chunks):
    """检查句子是否完整（不以标点结尾的比例）"""
    incomplete = 0
    for chunk in chunks:
        text = chunk.page_content.strip()
        if text and text[-1][[ not in "。！？.!?":
            incomplete += 1
    return 1 - (incomplete / len(chunks))
```

### 常见问题诊断

| 问题 | 症状 | 解决方案 |
|------|------|---------|
| 语义断裂 | 检索结果句子不完整 | 调整 separators，增加 overlap |
| 块过大 | 检索结果包含大量无关信息 | 减小 chunk_size |
| 块过小 | 检索结果缺乏上下文 | 增大 chunk_size，使用 Parent-Child |
| 表格破碎 | 表格数据无法正确检索 | 使用表格感知分块 |
| 代码断裂 | 函数/类被切断 | 使用代码感知分块 |

## 相关页面

- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[agent/RAG/RAG混合检索与重排序]] — 混合检索详解

## 来源

- LangChain 官方文档：Text Splitters
- LlamaIndex 文档：Node Parsers
- 2025 RAG 分块最佳实践
