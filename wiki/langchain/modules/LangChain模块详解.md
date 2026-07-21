---
tags: [langchain, modules, overview, reference]
created: 2026-07-07
updated: 2026-07-11
type: overview
---

# LangChain 模块详解

> 本目录系统讲解 LangChain 的核心模块：每个模块**是什么、有什么用、怎么用**，并附上完整可运行的代码示例。每个模块以文件夹形式组织，概述页 + 原子化子页面。

---

## 模块地图

```
wiki/langchain/modules/
├── LangChain模块详解.md              # 本页（导航入口）
├── langchain_core/                  # 核心抽象（消息、提示词、输出解析器）
│   └── 概述.md
├── langchain_agents/                # Agent 框架（create_agent、AgentState）
│   ├── 概述.md
│   ├── create_agent参数.md
│   └── Agent与LangGraph关系.md
├── langchain_tools/                 # 工具系统（@tool、ToolRuntime）
│   ├── 概述.md
│   ├── ToolRuntime.md
│   └── 工具返回值与状态更新.md
├── langchain_models/                # 模型抽象（init_chat_model、ChatModel）
│   ├── 概述.md
│   ├── bind_tools工具绑定.md
│   ├── 结构化输出.md
│   └── 多提供商配置.md
├── langchain_messages/              # 消息系统（HumanMessage、AIMessage、ToolMessage）
│   ├── 概述.md
│   ├── 消息类型详解.md
│   ├── 多模态消息.md
│   └── RemoveMessage历史裁剪.md
├── langchain_prompts/               # 提示词模板（ChatPromptTemplate、MessagesPlaceholder）
│   ├── 概述.md
│   ├── from_template方法.md
│   ├── PromptTemplate.md
│   ├── ChatPromptTemplate.md
│   ├── SystemMessagePromptTemplate.md
│   ├── FewShotPromptTemplate.md
│   ├── FewShotChatMessagePromptTemplate.md
│   ├── MessagesPlaceholder.md
│   ├── Prompt应用场景.md
│   └── Prompt高级技巧.md
├── langchain_output_parsers/        # 输出解析器（StrOutputParser、PydanticOutputParser）
│   ├── 概述.md
│   ├── StrOutputParser.md
│   ├── JsonOutputParser.md
│   ├── PydanticOutputParser.md
│   ├── ListOutputParser.md
│   ├── EnumOutputParser.md
│   ├── OutputFixingParser.md
│   ├── with_structured_output.md
│   └── 错误处理.md
├── langchain_runnables/             # Runnable 协议（LCEL、pipe、batch、stream）
│   ├── 概述.md
│   ├── LCEL链式组合.md
│   ├── RunnableLambda.md
│   ├── RunnablePassthrough.md
│   ├── RunnableParallel.md
│   ├── RunnableBranch.md
│   ├── RunnableWithFallbacks.md
│   ├── RunnableSerializable.md
│   └── RunnableWithMessageHistory.md
├── langchain_documents/             # 文档加载与分割（Document Loaders、Splitters）
│   ├── 概述.md
│   ├── Document数据模型.md
│   ├── 文档加载器.md
│   └── 文本分割器.md
├── langchain_embeddings/            # Embeddings 接口
│   ├── 概述.md
│   ├── embed_query_vs_embed_documents.md
│   ├── OpenAIEmbeddings.md
│   ├── HuggingFaceEmbeddings.md
│   └── CacheBackedEmbeddings.md
├── langchain_vectorstores/          # 向量存储（FAISS、Chroma、PGVector）
│   ├── 概述.md
│   ├── InMemoryVectorStore.md
│   ├── FAISS.md
│   ├── Chroma.md
│   ├── PGVector.md
│   └── Pinecone.md
├── langchain_retrievers/            # 检索器（VectorStoreRetriever、MultiQueryRetriever）
│   ├── 概述.md
│   ├── VectorStoreRetriever.md
│   ├── MultiQueryRetriever.md
│   ├── SelfQueryRetriever.md
│   ├── ContextualCompressionRetriever.md
│   ├── EnsembleRetriever.md
│   ├── BM25Retriever.md
│   └── 自定义Retriever.md
├── langchain_callbacks/             # 回调系统（日志、追踪、Token 统计）
│   ├── 概述.md
│   ├── 回调事件类型.md
│   ├── 自定义CallbackHandler.md
│   ├── LangSmith追踪.md
│   └── Token统计.md
├── langchain_stores/                # 键值存储（BaseStore、PostgresStore）
│   ├── 概述.md
│   ├── Store核心API.md
│   ├── 命名空间设计.md
│   └── Store与Checkpointer对比.md
├── langchain_checkpoints/           # 检查点（BaseCheckpointSaver）
│   ├── 概述.md
│   ├── Checkpoint元数据.md
│   ├── 多轮对话与thread_id.md
│   ├── 时间旅行.md
│   ├── 人机交互中断与恢复.md
│   └── Checkpointer创建链示例.md
└── langchain-mcp/                   # MCP 集成（Model Context Protocol）
    ├── 概述.md
    ├── MCP协议概述.md
    ├── MultiServerMCPClient.md
    ├── 传输方式对比.md
    ├── MCP服务器示例.md
    ├── MCP服务器开发.md
    ├── MCP安全模型.md
    └── MCP与Agent中间件集成.md
```

---

## 模块分类速览

### 核心抽象层（langchain-core）

| 模块 | 作用 | 关键类/函数 |
|------|------|------------|
| **messages** | 消息数据模型 | `HumanMessage`、`AIMessage`、`SystemMessage`、`ToolMessage` |
| **prompts** | 提示词模板 | `ChatPromptTemplate`、`MessagesPlaceholder` |
| **output_parsers** | 输出解析 | `StrOutputParser`、`PydanticOutputParser` |
| **runnables** | 可执行协议 | `Runnable`、`RunnableLambda`、`RunnablePassthrough` |
| **embeddings** | 嵌入接口 | `Embeddings` 抽象基类 |
| **documents** | 文档抽象 | `Document`（page_content + metadata） |
| **vectorstores** | 向量存储接口 | `VectorStore` 抽象基类 |
| **retrievers** | 检索器接口 | `BaseRetriever` 抽象基类 |
| **stores** | 键值存储 | `BaseStore` 抽象基类 |
| **callbacks** | 回调钩子 | `BaseCallbackHandler` |
| **tools** | 工具抽象 | `BaseTool`、`@tool` 装饰器 |
| **agents** | Agent 工厂 | `create_agent` |
| **chat_models** | 聊天模型接口 | `BaseChatModel` |

### 集成层（langchain-{provider}）

| 模块 | 作用 |
|------|------|
| `langchain-openai` | OpenAI / Azure OpenAI 集成 |
| `langchain-anthropic` | Anthropic Claude 集成 |
| `langchain-google-genai` | Google Gemini 集成 |
| `langchain-aws` | AWS Bedrock 集成 |
| `langchain-ollama` | 本地 Ollama 模型 |
| `langchain-huggingface` | HuggingFace 模型 |

### 扩展包

| 模块 | 作用 |
|------|------|
| `langgraph` | 底层编排（StateGraph、Checkpointer） |
| `langgraph-checkpoint-postgres` | Postgres 持久化 |
| `langchain-mcp-adapters` | MCP 协议客户端 |
| `deepagents` | 高级 Harness（文件系统、子 Agent） |
| `langchain-community` | 社区贡献集成 |

---

## 模块使用层次

```
┌──────────────────────────────────────────────────────┐
│  高级封装：deepagents.create_deep_agent               │
├──────────────────────────────────────────────────────┤
│  标准工厂：langchain.agents.create_agent             │
├──────────────────────────────────────────────────────┤
│  协议层：Runnable / LCEL                              │
├──────────────────────────────────────────────────────┤
│  组件：Tools / Prompts / Parsers / Retrievers         │
├──────────────────────────────────────────────────────┤
│  数据：Messages / Documents                          │
├──────────────────────────────────────────────────────┤
│  基础设施：ChatModel / Embeddings / VectorStore      │
└──────────────────────────────────────────────────────┘
```

**学习建议**：从底层（Messages、Prompts）→ 中层（Runnable、Tools）→ 顶层（Agents）。

---

## 文档索引

### 核心抽象（langchain-core）

- [[langchain/modules/langchain_core/概述|langchain-core 概述]] — 包结构与设计理念
- [[langchain/modules/langchain_agents/概述|agents 模块]] — `create_agent` 工厂
- [[langchain/modules/langchain_tools/概述|tools 模块]] — 工具定义与运行时
- [[langchain/modules/langchain_models/概述|chat_models 模块]] — 模型统一接口
- [[langchain/modules/langchain_messages/概述|messages 模块]] — 消息数据模型
- [[langchain/modules/langchain_prompts/概述|prompts 模块]] — 提示词模板
- [[langchain/modules/langchain_output_parsers/概述|output_parsers 模块]] — 输出解析
- [[langchain/modules/langchain_runnables/概述|runnables 模块]] — Runnable 协议与 LCEL

### RAG 三件套

- [[langchain/modules/langchain_documents/概述|documents 模块]] — 文档加载与分割
- [[langchain/modules/langchain_embeddings/概述|embeddings 模块]] — 向量化接口
- [[langchain/modules/langchain_vectorstores/概述|vectorstores 模块]] — 向量存储
- [[langchain/modules/langchain_retrievers/概述|retrievers 模块]] — 检索器

### 状态与回调

- [[langchain/modules/langchain_callbacks/概述|callbacks 模块]] — 回调钩子
- [[langchain/modules/langchain_stores/概述|stores 模块]] — 长期记忆键值存储
- [[langchain/modules/langchain_checkpoints/概述|checkpoints 模块]] — 检查点持久化

### 集成与扩展

- [[langchain/modules/langchain-mcp/概述|mcp-adapters 模块]] — MCP 协议集成

---

## 与已有知识库的关联

- [[langchain/LangChain使用手册]] — 按章节组织的方法论手册
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]] — 4 大模块综合实战
- [[langchain/agents/Agent代理]] — Agent 章节
- [[langchain/tools/工具]] — 工具章节
- [[agent/orchestration/LangChain]] — 框架实体
