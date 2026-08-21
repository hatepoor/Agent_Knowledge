---
tags: [chroma, langchain, RAG, integration]
type: howto
created: 2026-07-26
updated: 2026-07-26
---

# Chroma 与 LangChain 集成

## 安装

```bash
pip install langchain-chroma langchain-openai
```

`langchain-chroma` 是 LangChain 官方维护的 Chroma 集成包，提供 `Chroma` 类作为 VectorStore 接口的实现。

## 基本用法

### 创建 VectorStore

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 内存模式
vectorstore = Chroma(
    collection_name="docs",
    embedding_function=embeddings,
)

# 持久化模式
vectorstore = Chroma(
    collection_name="docs",
    embedding_function=embeddings,
    persist_directory="./chroma_data",
)
```

### 添加文档

```python
from langchain_core.documents import Document

docs = [
    Document(
        page_content="LangChain 是开发 LLM 应用的框架",
        metadata={"source": "wiki", "topic": "langchain"},
    ),
    Document(
        page_content="Agent 能自主决策并调用工具",
        metadata={"source": "wiki", "topic": "agent"},
    ),
]

# 添加文档（LangChain 自动调用 embedding_function 向量化）
ids = vectorstore.add_documents(docs)

# 也可以指定 ID
vectorstore.add_documents(docs, ids=["custom_1", "custom_2"])
```

### 检索

```python
# 相似度搜索 —— 返回 Document 对象列表
docs = vectorstore.similarity_search("什么是 Agent？", k=3)

# 相似度搜索 + 分数
docs_with_scores = vectorstore.similarity_search_with_score("什么是 RAG？", k=3)
for doc, score in docs_with_scores:
    print(f"[{score:.4f}] {doc.page_content}")

# 元数据过滤
docs = vectorstore.similarity_search(
    "框架介绍",
    k=2,
    filter={"topic": "langchain"},
)

# MMR 检索（最大边际相关性，减少结果间的重复）
docs = vectorstore.max_marginal_relevance_search(
    "AI 应用开发",
    k=5,
    fetch_k=20,  # 先检索 20 条，再用 MMR 选出 5 条
)
```

### 转为 Retriever

```python
# 基本 Retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",  # 或 "mmr"
    search_kwargs={"k": 5},
)

# 使用 Retriever
docs = retriever.invoke("什么是向量数据库？")
```

## 完整 RAG 实现

### 简单 RAG Chain

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 1. 组件
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(
    collection_name="knowledge",
    embedding_function=embeddings,
    persist_directory="./chroma_data",
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
llm = ChatOpenAI(model="gpt-4o")

# 2. Prompt
prompt = ChatPromptTemplate.from_template("""
根据以下参考资料回答问题。如果资料中没有相关信息，请说明。

参考资料：
{context}

问题：{question}
""")

# 3. RAG Chain
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 4. 使用
answer = rag_chain.invoke("LangChain 是什么？")
print(answer)
```

### Agent + RAG + Chroma

将 Chroma 检索封装成 Agent 工具：

```python
from langchain.agents import create_agent, tool
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

vectorstore = Chroma(
    collection_name="knowledge",
    embedding_function=OpenAIEmbeddings(),
    persist_directory="./chroma_data",
)

@tool
def search_knowledge(query: str) -> str:
    """搜索知识库，返回相关的文档片段"""
    docs = vectorstore.similarity_search(query, k=3)
    return "\n\n".join(f"[{d.metadata.get('source', '未知')}] {d.page_content}" for d in docs)

@tool
def search_by_topic(query: str, topic: str) -> str:
    """按主题过滤搜索知识库"""
    docs = vectorstore.similarity_search(query, k=3, filter={"topic": topic})
    return "\n\n".join(d.page_content for d in docs)

agent = create_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[search_knowledge, search_by_topic],
)

result = await agent.ainvoke({
    "messages": [{"role": "user", "content": "帮我查一下关于 Agent 记忆机制的资料"}]
})
```

## 文档加载与入库

### 从文件加载

```python
from langchain_community.document_loaders import (
    PyPDFLoader,
    TextLoader,
    DirectoryLoader,
)
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 加载文档
loader = DirectoryLoader(
    "./documents",
    glob="**/*.md",
    loader_cls=TextLoader,
)
raw_docs = loader.load()

# 2. 切分
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "，", " "],
)
chunks = splitter.split_documents(raw_docs)

# 3. 入库
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings(),
    collection_name="my_wiki",
    persist_directory="./chroma_data",
)
```

### 从已有 Chroma 加载

```python
# 连接已有的 Chroma 数据库
vectorstore = Chroma(
    collection_name="my_wiki",
    embedding_function=OpenAIEmbeddings(),
    persist_directory="./chroma_data",
)

# 查看已有数据量
print(f"文档数量：{vectorstore._collection.count()}")
```

## 连接已有 Chroma 服务

```python
# 连接 Client-Server 模式的 Chroma
import chromadb

chroma_client = chromadb.HttpClient(host="localhost", port=8000)

vectorstore = Chroma(
    client=chroma_client,
    collection_name="production_docs",
    embedding_function=OpenAIEmbeddings(),
)
```

## 高级用法

### 多 Collection 管理

```python
# 不同主题的文档存在不同 collection
wiki_store = Chroma(collection_name="wiki", embedding_function=embeddings, persist_directory="./db")
code_store = Chroma(collection_name="code", embedding_function=embeddings, persist_directory="./db")
memo_store = Chroma(collection_name="memo", embedding_function=embeddings, persist_directory="./db")

# 统一检索
def unified_search(query: str, k: int = 3):
    results = []
    for store in [wiki_store, code_store, memo_store]:
        results.extend(store.similarity_search(query, k=k))
    # 按相关性排序
    results_with_score = [(doc, store.similarity_search_with_score(query, k=k))
                          for doc in results]
    return results[:k]
```

### 增量更新

```python
from hashlib import md5

def add_if_new(vectorstore, doc: Document):
    """只在内容变化时更新（通过内容哈希去重）"""
    content_hash = md5(doc.page_content.encode()).hexdigest()
    doc.metadata["content_hash"] = content_hash

    # 检查是否已存在相同内容
    existing = vectorstore.similarity_search(doc.page_content, k=1)
    if existing and existing[0].metadata.get("content_hash") == content_hash:
        return  # 内容未变，跳过

    # 新增或更新
    doc_id = md5(doc.metadata.get("source", "").encode()).hexdigest()
    vectorstore.add_documents([doc], ids=[doc_id])
```

### 删除和重建

```python
# 删除特定文档
vectorstore.delete(ids=["doc_1", "doc_2"])

# 按元数据删除（需要先 get 出 ID）
# Chroma 不直接支持按条件删除，需要手动查询
collection = vectorstore._collection
results = collection.get(where={"source": "old_file.md"})
if results["ids"]:
    collection.delete(ids=results["ids"])

# 删除整个 collection
client = chromadb.PersistentClient(path="./chroma_data")
client.delete_collection("old_collection")
```

## 注意事项

**Embedding 一致性**：同一个 collection 的 add 和 query 必须使用相同的 embedding 函数。更换模型后需要重新向量化所有文档。

**批量操作**：大量文档入库时用 `Chroma.from_documents()` 或分批 `add_documents()`，不要逐条添加。

**ID 唯一性**：同一个 collection 中 ID 不能重复，重复会报错或覆盖。建议用内容哈希或来源路径生成 ID。

## 相关页面

- [[tools/向量数据库/chroma/概述|← 返回 Chroma 概述]]
- [[tools/向量数据库/chroma/持久化与部署|持久化与部署]]
- [[agent/RAG/RAG_LangChain完整实现|RAG LangChain 完整实现]]
- [[agent/RAG/RAG关键模块解析|RAG 关键模块解析]]
