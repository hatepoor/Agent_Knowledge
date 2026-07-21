---
tags: [langgraph, rag, 检索增强, 向量数据库, 教程, 条件路由]
type: tutorial
created: 2026-07-11
updated: 2026-07-11
---

# RAG Agent

## 概述

RAG（Retrieval-Augmented Generation，检索增强生成）是最常见的 LLM 应用模式之一。通过 LangGraph，我们可以构建一个**智能 RAG Agent**——它不仅能检索文档回答问题，还能自主判断是否需要检索，或者直接回答。

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   用户提问                                               │
│     │                                                   │
│     ▼                                                   │
│  ┌──────────┐                                           │
│  │ 路由判断  │ ─── 可以直接回答 ──→ 直接回答 → END        │
│  └──────────┘                                           │
│     │ 需要检索                                           │
│     ▼                                                   │
│  ┌──────────┐    相关文档    ┌──────────┐               │
│  │ 检索节点  │ ───────────→ │ 生成节点  │ ──→ END       │
│  └──────────┘               └──────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 前置准备

```bash
pip install langgraph langchain-openai langchain-chroma chromadb
```

## 第一步：定义 RAG 状态

RAG Agent 的状态需要包含消息列表和检索到的上下文：

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class RAGState(TypedDict):
    messages: Annotated[list, add_messages]  # 对话消息
    context: str                              # 检索到的文档上下文
    need_retrieval: bool                      # 是否需要检索
```

> **为什么需要 `context` 字段？** 将检索结果和消息分开存储，便于在节点之间传递结构化数据，避免把所有信息都塞进消息列表。

## 第二步：准备向量数据库

使用 Chroma 创建本地向量数据库：

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document

# 准备示例文档
documents = [
    Document(
        page_content="LangGraph 是 LangChain 团队开发的图计算框架，专门用于构建有状态的多步骤 AI 应用。"
                    "它基于有向图模型，支持循环、条件分支等复杂流程。"
                    "LangGraph 的核心概念包括：状态（State）、节点（Node）、边（Edge）和条件边（Conditional Edge）。",
        metadata={"source": "langgraph_docs"},
    ),
    Document(
        page_content="LangChain 是一个用于开发大语言模型应用的框架。"
                    "它提供了模型接口、提示词模板、输出解析器、文档加载器、文本分割器等组件。"
                    "LangChain 的核心理念是将 LLM 与其他数据源连接起来，形成强大的应用链。",
        metadata={"source": "langchain_docs"},
    ),
    Document(
        page_content="RAG（检索增强生成）是一种结合信息检索和文本生成的技术。"
                    "它首先从知识库中检索相关文档，然后将检索结果作为上下文提供给 LLM，"
                    "让模型基于真实数据生成更准确的回答，减少幻觉。",
        metadata={"source": "rag_tutorial"},
    ),
    Document(
        page_content="向量数据库是 RAG 系统的核心组件。常见的向量数据库包括 Chroma、FAISS、Pinecone、Milvus 等。"
                    "它们将文本转换为高维向量，通过余弦相似度等算法进行语义检索。",
        metadata={"source": "vector_db_guide"},
    ),
    Document(
        page_content="Checkpointer 是 LangGraph 的持久化机制。它在每个节点执行后保存状态快照，"
                    "支持多轮对话记忆、时间旅行、故障恢复等功能。"
                    "常用的 Checkpointer 包括 MemorySaver（内存）、SqliteSaver、PostgresSaver。",
        metadata={"source": "langgraph_checkpoint"},
    ),
]

# 创建向量数据库
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embeddings,
    collection_name="langgraph_knowledge",
)

# 创建检索器
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
```

## 第三步：定义各节点

### 路由判断节点

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def route_question(state: RAGState) -> dict:
    """判断用户问题是否需要检索知识库
    
    简单问题（如问候语）直接回答，
    知识性问题则先检索再生成。
    """
    user_message = state["messages"][-1].content
    
    # 使用 LLM 判断是否需要检索
    response = llm.invoke([
        SystemMessage(content=(
            "判断用户的问题是否需要从知识库检索信息来回答。\n"
            "如果是简单的问候、闲聊，或者问题与知识库主题无关，回答'不需要检索'。\n"
            "如果问题涉及 LangGraph、LangChain、RAG、向量数据库等技术话题，回答'需要检索'。\n"
            "只回答'需要检索'或'不需要检索'，不要有其他内容。"
        )),
        state["messages"][-1],
    ])
    
    need_retrieval = "需要检索" in response.content
    return {"need_retrieval": need_retrieval}
```

### 检索节点

```python
def retrieve(state: RAGState) -> dict:
    """从向量数据库中检索相关文档"""
    query = state["messages"][-1].content
    # 调用检索器获取相关文档
    docs = retriever.invoke(query)
    # 将文档内容拼接为上下文
    context = "\n\n---\n\n".join([doc.page_content for doc in docs])
    return {"context": context}
```

### 生成节点

```python
from langchain_core.messages import AIMessage

def generate(state: RAGState) -> dict:
    """基于检索到的上下文生成回答"""
    query = state["messages"][-1].content
    context = state["context"]
    
    response = llm.invoke([
        SystemMessage(content=(
            "你是一个技术文档助手。请根据以下参考资料回答用户的问题。\n"
            "如果参考资料中没有相关信息，请如实告知。\n"
            "回答要准确、简洁，并引用来源。\n\n"
            f"参考资料：\n{context}"
        )),
        state["messages"][-1],
    ])
    return {"messages": [response]}
```

### 直接回答节点

```python
def direct_answer(state: RAGState) -> dict:
    """不需要检索时直接回答"""
    response = llm.invoke([
        SystemMessage(content="你是一个友好的 AI 助手。请简洁地回答用户的问题。"),
        state["messages"][-1],
    ])
    return {"messages": [response]}
```

## 第四步：条件路由

```python
def should_retrieve(state: RAGState) -> str:
    """根据路由判断结果决定下一步"""
    if state.get("need_retrieval", False):
        return "retrieve"    # 需要检索
    return "direct"          # 直接回答
```

## 第五步：构建完整的 RAG 图

```python
from langgraph.graph import StateGraph, START, END

# 创建状态图
workflow = StateGraph(RAGState)

# 添加节点
workflow.add_node("route", route_question)      # 路由判断
workflow.add_node("retrieve", retrieve)          # 检索
workflow.add_node("generate", generate)          # 生成
workflow.add_node("direct", direct_answer)       # 直接回答

# 添加边
workflow.add_edge(START, "route")               # 入口 → 路由判断
workflow.add_conditional_edges(                  # 路由 → 条件分支
    "route",
    should_retrieve,
    {
        "retrieve": "retrieve",     # 需要检索 → 检索节点
        "direct": "direct",         # 不需要 → 直接回答
    }
)
workflow.add_edge("retrieve", "generate")        # 检索 → 生成
workflow.add_edge("generate", END)               # 生成 → 结束
workflow.add_edge("direct", END)                 # 直接回答 → 结束

# 编译
app = workflow.compile()
```

完整图结构：

```
              ┌──────────────┐
              │              │
START ──→ route ──→ [条件判断]
              │              │
         need_retrieval   not needed
              │              │
              ▼              ▼
         retrieve       direct ──→ END
              │
              ▼
         generate ──→ END
```

## 完整可运行代码

```python
"""
RAG Agent 完整实现
基于 LangGraph + Chroma 的检索增强生成 Agent
"""
import os
from typing import TypedDict, Annotated

from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.documents import Document
from langchain_core.messages import SystemMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# ========== 1. 定义状态 ==========
class RAGState(TypedDict):
    messages: Annotated[list, add_messages]
    context: str
    need_retrieval: bool

# ========== 2. 准备向量数据库 ==========
documents = [
    Document(
        page_content="LangGraph 是 LangChain 团队开发的图计算框架，用于构建有状态的多步骤 AI 应用。"
                    "核心概念包括：状态（State）、节点（Node）、边（Edge）和条件边（Conditional Edge）。",
        metadata={"source": "langgraph_docs"},
    ),
    Document(
        page_content="LangChain 是一个用于开发大语言模型应用的框架。"
                    "提供了模型接口、提示词模板、输出解析器、文档加载器、文本分割器等组件。",
        metadata={"source": "langchain_docs"},
    ),
    Document(
        page_content="RAG（检索增强生成）是结合信息检索和文本生成的技术。"
                    "先从知识库检索相关文档，再将检索结果作为上下文提供给 LLM，减少幻觉。",
        metadata={"source": "rag_tutorial"},
    ),
    Document(
        page_content="向量数据库是 RAG 系统的核心组件。常见的有 Chroma、FAISS、Pinecone、Milvus。"
                    "它们将文本转换为高维向量，通过余弦相似度等算法进行语义检索。",
        metadata={"source": "vector_db_guide"},
    ),
    Document(
        page_content="Checkpointer 是 LangGraph 的持久化机制，在每个节点执行后保存状态快照。"
                    "支持多轮对话记忆、时间旅行、故障恢复等功能。",
        metadata={"source": "langgraph_checkpoint"},
    ),
]

embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embeddings,
    collection_name="langgraph_knowledge",
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ========== 3. 创建模型 ==========
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# ========== 4. 定义节点 ==========
def route_question(state: RAGState) -> dict:
    """判断是否需要检索"""
    response = llm.invoke([
        SystemMessage(content=(
            "判断用户的问题是否需要从知识库检索信息来回答。"
            "如果是简单问候或闲聊，回答'不需要检索'。"
            "如果涉及技术话题，回答'需要检索'。"
            "只回答'需要检索'或'不需要检索'。"
        )),
        state["messages"][-1],
    ])
    return {"need_retrieval": "需要检索" in response.content}

def retrieve(state: RAGState) -> dict:
    """检索相关文档"""
    query = state["messages"][-1].content
    docs = retriever.invoke(query)
    context = "\n\n---\n\n".join([doc.page_content for doc in docs])
    return {"context": context}

def generate(state: RAGState) -> dict:
    """基于上下文生成回答"""
    response = llm.invoke([
        SystemMessage(content=(
            "你是技术文档助手。请根据参考资料回答用户问题。\n"
            "如果参考资料中没有相关信息，请如实告知。\n\n"
            f"参考资料：\n{state['context']}"
        )),
        state["messages"][-1],
    ])
    return {"messages": [response]}

def direct_answer(state: RAGState) -> dict:
    """直接回答"""
    response = llm.invoke([
        SystemMessage(content="你是友好的 AI 助手。简洁回答用户问题。"),
        state["messages"][-1],
    ])
    return {"messages": [response]}

# ========== 5. 条件路由 ==========
def should_retrieve(state: RAGState) -> str:
    return "retrieve" if state.get("need_retrieval") else "direct"

# ========== 6. 构建图 ==========
workflow = StateGraph(RAGState)
workflow.add_node("route", route_question)
workflow.add_node("retrieve", retrieve)
workflow.add_node("generate", generate)
workflow.add_node("direct", direct_answer)

workflow.add_edge(START, "route")
workflow.add_conditional_edges("route", should_retrieve, {
    "retrieve": "retrieve",
    "direct": "direct",
})
workflow.add_edge("retrieve", "generate")
workflow.add_edge("generate", END)
workflow.add_edge("direct", END)

app = workflow.compile()

# ========== 7. 运行测试 ==========
if __name__ == "__main__":
    from langchain_core.messages import HumanMessage
    
    # 测试 1：需要检索的问题
    print("=== 测试 1：技术知识问题 ===")
    result = app.invoke({
        "messages": [HumanMessage(content="LangGraph 的核心概念有哪些？")]
    })
    print(f"回答: {result['messages'][-1].content}")
    
    # 测试 2：简单问候（不需要检索）
    print("\n=== 测试 2：简单问候 ===")
    result = app.invoke({
        "messages": [HumanMessage(content="你好！")]
    })
    print(f"回答: {result['messages'][-1].content}")
    
    # 测试 3：另一个技术问题
    print("\n=== 测试 3：RAG 相关问题 ===")
    result = app.invoke({
        "messages": [HumanMessage(content="什么是 RAG？向量数据库在其中起什么作用？")]
    })
    print(f"回答: {result['messages'][-1].content}")
```

## 执行流程演示

### 流程 1：需要检索的问题

用户问："LangGraph 的核心概念有哪些？"

```
步骤 1: START → route
┌─────────────────────────────────────────────┐
│ 节点: route_question                         │
│ 输入: "LangGraph 的核心概念有哪些？"           │
│ LLM 判断: 涉及技术话题 → "需要检索"           │
│ 输出: need_retrieval = True                   │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 2: route → should_retrieve → "retrieve"
                    │
                    ▼
步骤 3: retrieve
┌─────────────────────────────────────────────┐
│ 节点: retrieve                               │
│ 查询: "LangGraph 的核心概念有哪些？"           │
│ 检索结果:                                    │
│   - "LangGraph 是...核心概念包括：状态、      │
│      节点、边、条件边..."                      │
│   - "Checkpointer 是 LangGraph 的..."        │
│   - "RAG（检索增强生成）..."                   │
│ 输出: context = 拼接的文档内容                  │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 4: retrieve → generate
┌─────────────────────────────────────────────┐
│ 节点: generate                               │
│ 输入: 用户问题 + context                      │
│ 输出: "LangGraph 的核心概念包括：              │
│       1. 状态（State）...                     │
│       2. 节点（Node）...                      │
│       3. 边（Edge）...                        │
│       4. 条件边（Conditional Edge）..."        │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 5: generate → END
```

### 流程 2：不需要检索的问题

用户问："你好！"

```
步骤 1: START → route
┌─────────────────────────────────────────────┐
│ 节点: route_question                         │
│ LLM 判断: 简单问候 → "不需要检索"             │
│ 输出: need_retrieval = False                  │
└─────────────────────────────────────────────┘
                    │
                    ▼
步骤 2: route → should_retrieve → "direct"
                    │
                    ▼
步骤 3: direct → END
┌─────────────────────────────────────────────┐
│ 节点: direct_answer                          │
│ 输出: "你好！有什么可以帮助你的吗？"           │
└─────────────────────────────────────────────┘
```

## 进阶：带重排序的 RAG

检索后可以对文档进行重排序，提高生成质量：

```python
def rerank_and_retrieve(state: RAGState) -> dict:
    """检索 + 重排序"""
    query = state["messages"][-1].content
    
    # 第一步：检索更多文档（k=10）
    docs = retriever.invoke(query)
    
    # 第二步：使用 LLM 重排序（选出最相关的 3 个）
    response = llm.invoke([
        SystemMessage(content=(
            "请从以下文档中选出与问题最相关的 3 个，返回编号（如 1,3,5）：\n\n"
            + "\n\n".join([f"[{i+1}] {doc.page_content}" for i, doc in enumerate(docs)])
        )),
        state["messages"][-1],
    ])
    
    # 解析 LLM 返回的编号
    try:
        indices = [int(x.strip()) - 1 for x in response.content.split(",")]
        selected = [docs[i] for i in indices if i < len(docs)]
    except (ValueError, IndexError):
        selected = docs[:3]  # 解析失败则取前 3 个
    
    context = "\n\n---\n\n".join([doc.page_content for doc in selected])
    return {"context": context}
```

## 进阶：结合工具调用的 RAG Agent

将 RAG 与工具调用结合，让 Agent 可以自主选择检索策略：

```python
from langchain_core.tools import tool

@tool
def search_knowledge_base(query: str) -> str:
    """搜索内部知识库。当用户询问技术文档、产品信息等内部知识时使用。"""
    docs = retriever.invoke(query)
    return "\n\n".join([doc.page_content for doc in docs])

@tool  
def search_web(query: str) -> str:
    """搜索互联网。当用户询问最新信息或知识库中没有的内容时使用。"""
    # 模拟网络搜索
    return f"搜索结果：关于 '{query}' 的最新信息..."

tools = [search_knowledge_base, search_web]
llm_with_tools = llm.bind_tools(tools)

# Agent 可以自主决定是查知识库还是搜索网络
def smart_agent(state: RAGState) -> dict:
    """智能 Agent：自主选择检索策略"""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response][[}
```

## 相关页面

- [[langgraph/tutorial/ReAct_Agent实现|ReAct Agent 实现]] — 基础 Agent 构建
- [[langgraph/tutorial/多轮对话Agent|多轮对话 Agent]] — 添加对话记忆
- [[langgraph/concepts/图计算范式|图计算范式]] — 为什么需要图结构
- [[langgraph/concepts/LangGraph_vs_LangChain|LangGraph vs LangChain]] — 框架对比
- [[langchain/modules/langchain_vectorstores/Chroma|Chroma 向量数据库]] — 向量存储详解
- [[langchain/modules/langchain_vectorstores/概述|向量数据库概述]] — 向量数据库全景
- [[langchain/modules/langchain_retrievers/概述|Retriever 概述]] — 检索器详解
