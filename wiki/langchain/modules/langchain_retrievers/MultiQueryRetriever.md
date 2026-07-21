---
tags: [langchain, retrievers, multi-query, llm, recall]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# MultiQueryRetriever

## 是什么

用户问的问题往往表述不够好。`MultiQueryRetriever` 让 LLM 把一个问题**改写为多个不同角度的变体**，分别检索后合并去重，**在 RAG 召回环节提效**。

```
原始 query: "怎么用 LangChain？"
    ↓ LLM 改写
变体1: "LangChain 的安装和基础使用方法"
变体2: "LangChain 构建 LLM 应用的典型流程"
变体3: "LangChain 各模块的功能和使用示例"
    ↓ 并行检索
合并去重 → 返回
```

## 导入

```python
from langchain.retrievers import MultiQueryRetriever
from langchain_openai import ChatOpenAI
```

---

## 基本用法

```python
from langchain_core.documents import Document
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

embeddings = OpenAIEmbeddings()
llm = ChatOpenAI(model="gpt-4o", temperature=0)  # temperature=0 保证改写稳定

docs = [
    Document(page_content="LangChain 是一个 LLM 应用开发框架，支持 chains 和 agents", metadata={"topic": "langchain"}),
    Document(page_content="LangChain 的安装：pip install langchain", metadata={"topic": "langchain"}),
    Document(page_content="Python 在数据科学和 Web 开发中广泛应用", metadata={"topic": "python"}),
    Document(page_content="AI Agent 可以自主调用工具完成复杂任务", metadata={"topic": "ai"}),
]
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)

# 基础版：默认生成 3 个变体
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=llm,
)

results = multi_retriever.invoke("怎么使用 LangChain？")
# LLM 自动生成类似变体：
#   "LangChain 的基础入门和使用方法"
#   "LangChain 的功能模块和典型用例"
#   "如何安装和配置 LangChain 开发环境"
```

---

## 自定义改写 Prompt

默认的改写 prompt 可能不够贴合你的场景。可以传自定义 prompt：

```python
from langchain_core.prompts import ChatPromptTemplate

# 自定义改写 prompt
custom_prompt = ChatPromptTemplate.from_template("""你是一个搜索查询优化专家。
将用户的原始问题改写为 {num_queries} 个不同的搜索查询，
每个查询从不同角度表达相同意图，帮助提高检索召回率。

原始问题: {question}

改写查询（一行一个）:
""")

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
    prompt=custom_prompt,
)
```

---

## 参数对比

| 参数 | 默认 | 作用 | 建议 |
|------|------|------|------|
| `llm` | 必传 | 用于生成改写查询的模型 | 用便宜的模型即可，不需要推理能力强的 |
| `retriever` | 必传 | 底层检索器 | 通常 VectorStoreRetriever |
| `include_original` | `False` | 是否把原始 query 也作为一路 | 建议 `True`，确保不丢失原始意图 |
| `parser_key` | `"lines"` | 从 LLM 输出解析查询的键 | 一般不需要改 |
| `num_queries` | 3 | 生成多少个变体 | 2-5 之间，越多成本越高 |

```python
# 包含原始查询 + 2 个变体，共 3 路检索
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
    include_original=True,
    num_queries=2,
)
```

---

## 成本分析

**每次 `invoke` 的成本**：

```
LLM 调用: 1 次（生成 N 个变体）
检索次数: N 次（N = 变体数 + 原始 query（如果 include_original=True））
总 Token: 改写 prompt token + N × 每个 query 的 embedding token
```

| num_queries | LLM 调用 | 检索次数 | 额外 token |
|-------------|----------|----------|------------|
| 2 | 1 | 2-3 | ~200 |
| 3 (默认) | 1 | 3-4 | ~300 |
| 5 | 1 | 5-6 | ~500 |

> **结论**：MultiQueryRetriever 的成本大头在检索次数（每次检索都要 embedding query），不在 LLM。如果你的 embedding API 按调用次数收费（非按 token），要注意。

---

## MultiQueryRetriever vs EnsembleRetriever

| | MultiQueryRetriever | EnsembleRetriever |
|---|---|---|
| **原理** | 改写 query，多角度搜同一个 VectorStore | 同一个 query，搜多个不同的检索器 |
| **解决的问题** | query 表述不佳 | 检索器覆盖面不够 |
| **何时用** | 用户问题模糊、多义词、中英混用 | 需要结合关键词 + 语义、多数据源 |
| **成本** | 1 次 LLM + N 次检索 | 0 次 LLM + N 次检索 |

> 如果问题出在"搜到的内容单一"→ MultiQuery；如果问题出在"搜的方式单一"→ Ensemble。

---

## 去重策略

MultiQueryRetriever 内部会对合并结果去重（基于 `Document.page_content`）。**这意味着如果多路返回了相同内容的文档，只保留一份。**

```python
# 如果 2 个变体搜到了同一个文档，结果中只出现 1 次
# 3 路结果: [A, B, C] + [A, D, E] + [B, F, G]
# 最终: [A, B, C, D, E, F, G]（去重后随机顺序）
```

---

## 故障排查

### 问题 1：改写质量差，变体和原问题完全无关

**原因**：LLM 太弱或 temperature 太高。

**修正**：`temperature=0` + 用指令遵循能力强的模型。

```python
# ❌ temperature 太高导致改写发散
llm = ChatOpenAI(model="gpt-4o", temperature=1.0)

# ✅ 改写需要稳定性
llm = ChatOpenAI(model="gpt-4o", temperature=0)
```

### 问题 2：结果没有比单个 query 更好

**原因**：知识库本身太简单或太小，单 query 已经能全覆盖。

**诊断**：对比 `len(multi_retriever.invoke(q))` vs `len(retriever.invoke(q))`。如果去重后数量相同或只多 1-2 个，说明知识库不需要多路召回。

### 问题 3：调用太慢

**原因**：LLM 生成变体本身需要 1-3 秒，加上 N 次检索。

**修正**：减少 `num_queries` 到 2，或用更快的 LLM（DeepSeek、本地模型）。

---

## 子页面

- [[langchain/modules/langchain_retrievers/EnsembleRetriever|EnsembleRetriever]] — 多路融合 vs 查询改写
- [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]] — 基础检索器

## 相关

- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
