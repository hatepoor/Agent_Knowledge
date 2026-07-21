---
tags: [concept, LangChain, API, 方法, 调用]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# LangChain 核心方法对比

> LangChain 中有多个核心方法，分别用于不同场景。理解它们的区别是正确使用 LangChain 的关键。

## 方法分类

LangChain 的方法分为两大类：

### 1. 嵌入方法（Embeddings 类）

用于将文本转换为向量，属于 `Embeddings` 抽象类的方法。

| 方法 | 用途 | 输入 | 输出 | 使用场景 |
|------|------|------|------|---------|
| `embed_query()` | 嵌入单个查询 | 单个字符串 | 单个向量 | 用户查询向量化 |
| `embed_documents()` | 批量嵌入文档 | 字符串列表 | 向量列表 | 构建向量数据库 |

### 2. 执行方法（Runnable 接口）

用于执行链、LLM 或检索器，属于 `Runnable` 接口的方法。

| 方法 | 用途 | 输入 | 输出 | 使用场景 |
|------|------|------|------|---------|
| `invoke()` | 同步执行 | 输入数据 | 完整结果 | 一次性获取完整回答 |
| `stream()` | 流式执行 | 输入数据 | 迭代器 | 实时显示生成过程 |
| `ainvoke()` | 异步同步执行 | 输入数据 | 完整结果 | 异步环境的一次性调用 |
| `astream()` | 异步流式执行 | 输入数据 | 异步迭代器 | 异步环境的流式输出 |

## embed_query vs embed_documents

### embed_query()

嵌入**单个查询文本**，通常用于用户输入。

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# 嵌入单个查询
query_vector = embeddings.embed_query("什么是 LangChain？")
print(f"维度: {len(query_vector)}")  # 1536

# 用于相似度搜索
results = vectorstore.similarity_search_by_vector(query_vector, k=5)
```

**特点**：
- 输入：单个字符串
- 输出：单个向量（list[float]）
- 某些模型会对查询做特殊处理（如添加 "query: " 前缀）

### embed_documents()

批量嵌入**多个文档**，用于构建向量数据库。

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# 批量嵌入文档
documents = [
    "LangChain 是一个框架",
    "Python 是编程语言",
    "RAG 是检索增强生成"
]

doc_vectors = embeddings.embed_documents(documents)
print(f"文档数: {len(doc_vectors)}")  # 3
print(f"每个向量维度: {len(doc_vectors[0][[)}")  # 1536

# 存入向量数据库
vectorstore.add_embeddings(
    texts=documents,
    embeddings=doc_vectors
)
```

**特点**：
- 输入：字符串列表
- 输出：向量列表（list[list[float]]）
- 批量处理更高效（减少 API 调用次数）
- 某些模型会对文档做特殊处理（如添加 "document: " 前缀）

### 为什么要分开？

某些嵌入模型（如 E5、BGE）对查询和文档使用不同的前缀：

```python
# 查询前缀
query_vector = embeddings.embed_query("query: 什么是 LangChain？")

# 文档前缀
doc_vectors = embeddings.embed_documents([
    "document: LangChain 是一个框架",
    "document: Python 是编程语言"
])
```

LangChain 自动处理这个差异，你只需调用正确的方法。

## invoke vs stream

### invoke()

**同步执行**，等待完整结果后返回。

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# 同步调用，等待完整回答
response = llm.invoke("写一首关于 Python 的诗")
print(response.content)
# 输出完整的诗（一次性）
```

**特点**：
- 阻塞等待完整结果
- 返回完整的 `AIMessage` 对象
- 适合后台任务或不需要实时显示的场景

### stream()

**流式执行**，逐步返回生成的内容。

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# 流式调用，逐步输出
for chunk in llm.stream("写一首关于 Python 的诗"):
    print(chunk.content, end="", flush=True)
    # 实时输出每个 token
```

**特点**：
- 非阻塞，逐步返回
- 返回 `Iterator[AIMessageChunk]`
- 适合实时显示生成过程（如聊天界面）
- 用户体验更好（不用等待完整生成）

### 对比示例

```python
from langchain_openai import ChatOpenAI
import time

llm = ChatOpenAI(model="gpt-4o-mini")

# invoke：等待完整结果
print("=== invoke ===")
start = time.time()
response = llm.invoke("写一个长故事")
print(f"耗时: {time.time() - start:.2f}s")
print(response.content)  # 一次性输出全部内容

# stream：实时输出
print("\n=== stream ===")
start = time.time()
for chunk in llm.stream("写一个长故事"):
    print(chunk.content, end="", flush=True)
print(f"\n耗时: {time.time() - start:.2f}s")
```

## 在 Chain 中的使用

### RAG Chain 示例

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

# 初始化组件
llm = ChatOpenAI(model="gpt-4o-mini")
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings)
retriever = vectorstore.as_retriever()

# 创建 Chain
prompt = ChatPromptTemplate.from_template("""
根据上下文回答问题：
{context}
问题：{input}
""")

question_answer_chain = create_stuff_documents_chain(llm, prompt)
rag_chain = create_retrieval_chain(retriever, question_answer_chain)

# 方式 1：invoke（同步，完整结果）
result = rag_chain.invoke({"input": "什么是 RAG？"})
print(result["answer"])  # 完整回答
print(result["context"])  # 检索到的文档

# 方式 2：stream（流式，逐步输出）
for chunk in rag_chain.stream({"input": "什么是 RAG？"}):
    if "answer" in chunk:
        print(chunk["answer"], end="", flush=True)
```

### 异步版本

```python
import asyncio

async def main():
    # ainvoke：异步同步
    result = await rag_chain.ainvoke({"input": "什么是 RAG？"})
    print(result["answer"])
    
    # astream：异步流式
    async for chunk in rag_chain.astream({"input": "什么是 RAG？"}):
        if "answer" in chunk:
            print(chunk["answer"], end="", flush=True)

asyncio.run(main())
```

## 方法选择指南

| 场景 | 推荐方法 | 原因 |
|------|---------|------|
| 构建向量数据库 | `embed_documents()` | 批量处理更高效 |
| 用户查询向量化 | `embed_query()` | 自动处理查询前缀 |
| 后台任务 | `invoke()` | 简单直接 |
| 聊天界面 | `stream()` | 实时显示，用户体验好 |
| 异步环境 | `ainvoke()` / `astream()` | 不阻塞事件循环 |
| 高并发服务 | `ainvoke()` / `astream()` | 更好的并发性能 |

## 常见错误

### 错误 1：混淆嵌入方法和执行方法

```python
# ❌ 错误：embed_query 不是 Runnable 方法
result = embeddings.embed_query("问题")  # 返回向量，不是回答

# ✅ 正确：使用 Chain 的 invoke
result = rag_chain.invoke({"input": "问题"})  # 返回回答
```

### 错误 2：在流式输出中访问完整内容

```python
# ❌ 错误：chunk 是部分结果
for chunk in llm.stream("问题"):
    print(chunk.content)  # 每次只有一部分文本

# ✅ 正确：累积内容
full_response = ""
for chunk in llm.stream("问题"):
    full_response += chunk.content
print(full_response)  # 完整内容
```

### 错误 3：忘记 flush

```python
# ❌ 错误：输出缓冲
for chunk in llm.stream("问题"):
    print(chunk.content, end="")  # 可能不会立即显示

# ✅ 正确：强制刷新
for chunk in llm.stream("问题"):
    print(chunk.content, end="", flush=True)  # 立即显示
```

## format vs invoke

`format()` 和 `invoke()` 属于不同的抽象层级，作用完全不同。

### 核心区别

| 维度 | `format()` | `invoke()` |
|------|-----------|------------|
| 所属 | `PromptTemplate` 专属 | 所有 `Runnable` 都有 |
| 作用 | 只做变量替换 | 执行组件的完整逻辑 |
| 返回类型 | `str`（纯字符串） | 取决于组件（PromptValue / AIMessage / dict 等） |
| 能否调 LLM | 不能 | 能（在 model/chain 上时） |
| 适用范围 | 只有 PromptTemplate | Prompt、Model、Chain、Parser、Retriever、Tool… |

### format()：仅做变量替换

`format()` 是 `PromptTemplate` 专属方法，只做字符串级别的变量替换，不调用 LLM，不执行任何链。

```python
from langchain_core.prompts import PromptTemplate

template = PromptTemplate.from_template("请解释什么是{concept}")

# format() 只做字符串替换
result = template.format(concept="量子计算")
print(result)          # "请解释什么是量子计算"
print(type(result))    # <class 'str'>
```

对于 `ChatPromptTemplate`，对应的方法是 `format_messages()`，返回消息列表：

```python
from langchain_core.prompts import ChatPromptTemplate

chat_template = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}"),
    ("human", "{question}"),
][[)

# format_messages() 返回消息列表
messages = chat_template.format_messages(role="专家", question="什么是AI？")
print(type(messages))    # <class 'list'>
for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个专家
# human: 什么是AI？
```

### invoke()：执行完整逻辑

`invoke()` 是 `Runnable` 接口的通用方法，所有 LangChain 组件都实现了它。它执行组件的完整逻辑——对 PromptTemplate 来说是格式化，对 ChatModel 来说是调 LLM API，对 chain 来说是跑完整个流水线。

```python
# 在 PromptTemplate 上 —— 返回 PromptValue（不是 str）
prompt_value = template.invoke({"concept": "量子计算"})
print(type(prompt_value))  # StringPromptValue
print(prompt_value.to_string())  # "请解释什么是量子计算"

# 在 ChatModel 上 —— 调用 LLM API
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4o-mini")
response = model.invoke("你好")  # 发 API 请求，返回 AIMessage
print(type(response))    # AIMessage
print(response.content)  # "你好！有什么可以帮你的？"

# 在 Chain 上 —— 跑完整个流水线
chain = template | model | StrOutputParser()
result = chain.invoke({"concept": "量子计算"})
# 内部依次执行：格式化模板 → 调 LLM → 解析输出
print(type(result))      # str（StrOutputParser 的输出）
```

### format_prompt()：介于两者之间

`format_prompt()` 是 `PromptTemplate` 的另一个方法，返回 `PromptValue` 对象（可以转为字符串或消息列表），与 `invoke()` 在 PromptTemplate 上的行为一致：

```python
# format() → str
text = template.format(concept="量子计算")

# format_prompt() → PromptValue
prompt_value = template.format_prompt(concept="量子计算")
print(prompt_value.to_string())   # 转为字符串
print(prompt_value.to_messages()) # 转为消息列表

# invoke() → PromptValue（等价于 format_prompt）
prompt_value2 = template.invoke({"concept": "量子计算"})
# 结果与 format_prompt() 相同
```

### 何时用哪个

| 场景 | 推荐方法 | 原因 |
|------|---------|------|
| 单独格式化模板，只需要字符串 | `format()` / `format_messages()` | 直观，返回类型简单 |
| 在 LCEL 链中调用 | `invoke()` | 链中自动传递 PromptValue |
| 需要调 LLM | `invoke()` | `format()` 不能调 LLM |
| 需要 PromptValue 对象 | `format_prompt()` 或 `invoke()` | 可灵活转 str 或 messages |

### 常见错误

```python
# ❌ 错误：期望 format() 调用 LLM
result = template.format(concept="量子计算")
# result 只是字符串 "请解释什么是量子计算"，没有调用 LLM！

# ✅ 正确：用 invoke() 或链式调用
chain = template | model | StrOutputParser()
result = chain.invoke({"concept": "量子计算"})

# ❌ 错误：在 ChatModel 上调用 format()
model = ChatOpenAI(model="gpt-4o-mini")
model.format(text="你好")  # AttributeError! ChatModel 没有 format()

# ✅ 正确：用 invoke()
response = model.invoke("你好")
```

## 相关页面

- [[langchain/basics/Agent_invoke_vs_ChatModel_invoke]] — create_agent().invoke() 与 ChatModel.invoke() 返回内容完整对比
- [[langchain/modules/langchain_embeddings/概述]] — embeddings 模块详解
- [[langchain/streaming/流式传输]] — 流式传输机制
- [[agent/RAG/RAG_LangChain完整实现]] — RAG 完整实现代码
