---
tags: [source, RAG, LangChain, 代码, 实现, Python]
type: source
created: 2026-07-08
updated: 2026-07-08
source_url: https://docs.langchain.com/oss/python/langchain/overview
---

# RAG LangChain 完整实现

> 📌 **历史遗留页面**：权威内容见 [[rag/00_导览与基础/LangChain_RAG端到端实现]]。对照见 [[agent/RAG/README|RAG 迁移对照表]]。

> 一套从文档加载到问答 Agent 的完整、可运行的 RAG 代码示例。基于 LangChain 最新 API（`langchain_core`、`langchain_community`、`langchain_openai`、`langchain_chroma`），包含两种检索策略的对比实现。

> ⚠️ 本页面部分代码使用 langchain_community 导入（已弃用）。新项目请参考 [[langchain/modules/langchain_documents/文档加载器|文档加载器]] 页面的推荐方式。

## 环境准备

```bash
# 安装依赖（Python 3.10+）
pip install langchain langchain-core langchain-community \
    langchain-openai langchain-chroma \
    langchain-text-splitters langchain-huggingface \
    chromadb faiss-cpu sentence-transformers \
    pypdf tiktoken
```

## 模块 ① 环境配置

```python:rag_pipeline.py
"""RAG Pipeline 环境配置"""
import os
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_huggingface import HuggingFaceEmbeddings

# ── OpenAI 配置（二选一）──
# 方式 A：使用 OpenAI API（需要 OPENAI_API_KEY 环境变量）
os.environ["OPENAI_API_KEY"] = "sk-xxx"  # 实际使用时从环境变量读取

# LLM 用于生成回答（temperature=0 保证回答确定性）
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Embedding 模型（text-embedding-3-small：1536维，$0.02/1M tokens，性价比最优）
embeddings_openai = OpenAIEmbeddings(model="text-embedding-3-small")

# ── 开源模型配置（替代方案，无需 API Key）──
# 方式 B：使用 BGE-M3 本地模型（中文最强，需 GPU 或较强 CPU）
embeddings_bge = HuggingFaceEmbeddings(
    model_name="BAAI/bge-large-zh-v1.5",  # 中文专用 BGE 模型
    model_kwargs={"device": "cpu"},        # 无 GPU 时用 cpu
    encode_kwargs={"normalize_embeddings": True},  # 归一化后可用点积
)

# 方式 C：使用 M3E 轻量模型（CPU 友好，512 token 上限）
embeddings_m3e = HuggingFaceEmbeddings(
    model_name="moka-ai/m3e-base",
    encode_kwargs={"normalize_embeddings": True},
)

# 选择当前使用的 Embedding 模型（切换方案时改这一行即可）
embeddings = embeddings_openai  # 实际使用时切换为 embeddings_bge 或 embeddings_m3e
```

## 模块 ② 文档加载与切分

```python
"""文档加载与文本分块"""
from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

# ── 单文件加载 ──
# PyPDFLoader：每页返回一个 Document，metadata 包含页码
pdf_loader = PyPDFLoader("./docs/llm_wiki.pdf")
pdf_docs = pdf_loader.load()
print(f"PDF 加载完成：{len(pdf_docs)} 页")

# ── 批量目录加载 ──
# DirectoryLoader：递归扫描目录，支持多线程
dir_loader = DirectoryLoader(
    path="./docs",
    glob="**/*.{pdf,md,txt}",      # 匹配 PDF/Markdown/文本
    loader_cls=TextLoader,          # 默认加载器（PDF 可换 PyPDFLoader）
    use_multithreading=True,        # 多线程加速
    silent_errors=True,             # 无法加载的文件静默跳过
)
all_docs = dir_loader.load()
print(f"目录加载完成：{len(all_docs)} 个文档")

# ── 文本分块 ──
# RecursiveCharacterTextSplitter：通用首选，按层级分隔符递归切分
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,                # 每块最大 500 字符（QA 场景偏小）
    chunk_overlap=50,              # 重叠 50 字符，防止语义断裂
    add_start_index=True,          # metadata 记录块起始位置，便于溯源
    separators=["\n## ", "\n### ", "\n\n", "\n", " ", ""],  # Markdown 感知分隔符
)
splits = text_splitter.split_documents(all_docs)
print(f"分块完成：{len(splits)} 个 chunk，平均 {sum(len(s.page_content) for s in splits)//len(splits)} 字符/chunk")
```

## 模块 ③ 向量库构建

```python
"""向量数据库构建（Chroma 持久化 + FAISS 内存）"""
from langchain_chroma import Chroma
from langchain_community.vectorstores import FAISS

# ── 方式 A：Chroma（推荐，支持持久化和元数据过滤）──
vectorstore_chroma = Chroma.from_documents(
    documents=splits,
    embedding=embeddings,
    persist_directory="./chroma_db",   # 持久化到磁盘，重启不丢失
    collection_name="rag_docs",        # 集合名称，可存多组文档
)
print(f"Chroma 向量库构建完成，集合: rag_docs")

# ── 方式 B：FAISS（纯内存，极速，适合原型验证）──
vectorstore_faiss = FAISS.from_documents(
    documents=splits,
    embedding=embeddings,
)
# FAISS 持久化（手动保存/加载）
vectorstore_faiss.save_local("faiss_index")
# 重新加载：new_store = FAISS.load_local("faiss_index", embeddings)

# 选择当前使用的向量库
vectorstore = vectorstore_chroma  # 生产用 Chroma，原型用 FAISS
```

## 模块 ④ 检索器配置

```python
"""检索器配置：基础 Similarity vs MMR + Rerank"""

# ═══════════════════════════════════════════
# 策略一：基础相似度检索（Simple & Fast）
# ═══════════════════════════════════════════
retriever_basic = vectorstore.as_retriever(
    search_type="similarity",       # 纯相似度搜索
    search_kwargs={"k": 4},         # 返回 Top-4
)

# 测试基础检索
test_query = "什么是检索增强生成？"
basic_results = retriever_basic.invoke(test_query)
print(f"[基础检索] 返回 {len(basic_results)} 个文档")
for i, doc in enumerate(basic_results):
    print(f"  [{i+1}] source={doc.metadata.get('source', 'N/A')}")
    print(f"      {doc.page_content[:100]}...")

# ═══════════════════════════════════════════
# 策略二：MMR + BGE Rerank（Diverse & Precise）
# ═══════════════════════════════════════════
retriever_mmr = vectorstore.as_retriever(
    search_type="mmr",              # 最大边际相关性：平衡相关性与多样性
    search_kwargs={
        "k": 4,                     # 最终返回 4 个
        "fetch_k": 20,              # MMR 过滤前先取 20 个候选
        "lambda_mult": 0.5,         # 0.5 = 相关性与多样性等权
    },
)

# ── 重排序：用 BGE-Reranker 对 MMR 结果精排 ──
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain.retrievers import ContextualCompressionRetriever

# Cross-Encoder 模型（比双塔 Embedding 更精准的相关性判断）
reranker_model = HuggingFaceCrossEncoder(
    model_name="BAAI/bge-reranker-base",  # 轻量级，CPU 可用
)
reranker = CrossEncoderReranker(
    model=reranker_model,
    top_n=3,  # 重排后只保留 Top-3
)

# 组装：MMR 检索 → Rerank 精排
retriever_rerank = ContextualCompressionRetriever(
    base_compressor=reranker,       # 重排序器
    base_retriever=retriever_mmr,   # 基础检索器（MMR）
)

# 测试高级检索
rerank_results = retriever_rerank.invoke(test_query)
print(f"\n[MMR+Rerank] 返回 {len(rerank_results)} 个文档（重排后）")
for i, doc in enumerate(rerank_results):
    print(f"  [{i+1}] source={doc.metadata.get('source', 'N/A')}")
    print(f"      {doc.page_content[:100]}...")

# ── 可选：多查询检索（Multi-Query，提升召回率）──
from langchain.retrievers.multi_query import MultiQueryRetriever
retriever_multi = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 4}),
    llm=llm,  # LLM 生成 3-5 个子查询，并行检索后合并去重
)
```

## 模块 ⑤ 链式组装

```python
"""RAG 链组装：检索 + 生成"""
from langchain.chains import create_retrieval_chain, create_history_aware_retriever
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage

# ── Prompt 模板设计 ──
# System Prompt 注入检索到的上下文，要求 LLM 仅基于上下文回答
system_prompt = (
    "你是一个专业的问答助手。请仅根据以下检索到的上下文回答问题。\n"
    "规则：\n"
    "1. 如果上下文中有相关信息，请基于上下文回答，并在句尾标注来源 [文档N]\n"
    "2. 如果上下文中没有相关信息，请回答'根据现有资料无法回答该问题'\n"
    "3. 不要编造信息，不要使用上下文之外的知识\n\n"
    "检索到的上下文：\n{context}"
)

qa_prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    MessagesPlaceholder("chat_history"),  # 多轮对话历史（可选）
    ("human", "{input}"),
])

# ── 历史感知检索器 ──
# 将后续问题（如"那个怎么样？"）结合历史重写为独立查询
contextualize_q_prompt = ChatPromptTemplate.from_messages([
    ("system", "根据聊天历史将用户最新问题重写为一个独立、可检索的查询。不要回答问题，只重写。"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(
    llm,                          # 用于查询重写的 LLM
    retriever_rerank,             # 使用 MMR+Rerank 检索器
    contextualize_q_prompt,       # 查询上下文化模板
)

# ── 文档填充链 ──
# 将检索到的多个 Document 合并为一个 context 字符串，注入 Prompt
question_answer_chain = create_stuff_documents_chain(
    llm,        # 生成回答的 LLM
    qa_prompt,  # 问答 Prompt 模板
)

# ── 完整 RAG 链 ──
# 检索器（历史感知） → 文档填充链（注入上下文）→ LLM 生成
rag_chain = create_retrieval_chain(
    history_aware_retriever,
    question_answer_chain,
)
```

## 模块 ⑥ 高级技巧

```python
"""高级技巧：多查询 + 父文档检索 + 结果溯源"""

# ── 父文档检索器：小块检索 + 大块返回 ──
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 父分割器：生成大块（2000字符），用于返回上下文
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
# 子分割器：生成小块（400字符），用于精确嵌入和检索
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=50)

# 创建父文档检索器
parent_retriever = ParentDocumentRetriever(
    vectorstore=Chroma.from_documents(  # 子块存向量库
        documents=[],
        embedding=embeddings,
        collection_name="child_chunks",
    ),
    docstore=InMemoryStore(),           # 父块存文档存储
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
parent_retriever.add_documents(all_docs)  # 添加原始文档

# ── 对比测试：基础 vs 高级 ──
def compare_strategies(query: str):
    """对比两种检索策略的效果"""
    print(f"\n{'='*60}")
    print(f"查询：{query}")
    print(f"{'='*60}")

    # 策略一：基础相似度检索
    print("\n── 策略一：Similarity Search ──")
    basic_chain = create_retrieval_chain(
        vectorstore.as_retriever(search_kwargs={"k": 4}),
        question_answer_chain,
    )
    result_basic = basic_chain.invoke({"input": query, "chat_history": []})
    print(f"回答：{result_basic['answer'][:200]}...")
    print(f"检索到 {len(result_basic['context'])} 个文档")

    # 策略二：MMR + Rerank
    print("\n── 策略二：MMR + BGE Rerank ──")
    result_rerank = rag_chain.invoke({"input": query, "chat_history": []})
    print(f"回答：{result_rerank['answer'][:200]}...")
    print(f"检索到 {len(result_rerank['context'])} 个文档")

    # 溯源：打印检索到的文档来源
    print("\n── 检索结果溯源 ──")
    for i, doc in enumerate(result_rerank["context"]):
        source = doc.metadata.get("source", "unknown")
        page = doc.metadata.get("page", "N/A")
        start = doc.metadata.get("start_index", "N/A")
        print(f"  [文档{i+1}] source={source}, page={page}, start={start}")

    return result_basic, result_rerank


# ── 多轮对话 ──
def chat_loop(rag_chain):
    """多轮对话式 RAG"""
    chat_history = []
    print("RAG 问答系统已启动（输入 'quit' 退出）")
    while True:
        question = input("\n用户: ")
        if question.lower() == "quit":
            break

        # 调用 RAG 链
        response = rag_chain.invoke({
            "input": question,
            "chat_history": chat_history,
        })

        print(f"助手: {response['answer']}")

        # 更新对话历史
        chat_history.extend([
            HumanMessage(content=question),
            AIMessage(content=response["answer"]),
        ][[)

# ── 主程序入口 ──
if __name__ == "__main__":
    # 对比测试
    compare_strategies("什么是检索增强生成（RAG）？")
    compare_strategies("RAG 有哪些局限性？")

    # 启动多轮对话
    # chat_loop(rag_chain)
```

## 策略对比说明

| 维度 | Similarity Search | MMR + Rerank |
|------|-------------------|-------------|
| 检索速度 | 快（单次向量搜索） | 较慢（MMR + Cross-Encoder 推理） |
| 结果多样性 | 可能返回近重复内容 | MMR 保证多样性 |
| 精度 | 依赖 Embedding 质量 | Rerank 精排后精度更高 |
| 成本 | 仅 Embedding API 调用 | 额外 Cross-Encoder 推理 |
| 适用场景 | 原型验证、快速问答 | 生产环境、精度要求高 |

## 相关页面

- [[agent/RAG/RAG核心概念与演进]] — RAG 定义与三代演进
- [[agent/RAG/RAG关键模块解析]] — 六大模块的参数选型详解
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[langchain/agents/Agent代理]] — Agent = 模型 + 框架
- [[langchain/tools/工具]] — Agent 工具系统
- [[langchain/memory/短期记忆]] — Agent 记忆管理

## 来源

- LangChain 官方文档：RAG Tutorial, Retrievers, Text Splitters
- LangChain API Reference：`create_retrieval_chain`, `create_history_aware_retriever`
- BGE-Reranker 模型文档：BAAI/bge-reranker-base
