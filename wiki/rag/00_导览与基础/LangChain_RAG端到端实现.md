---
tags: [rag, 框架实战, langchain, create_agent]
created: 2026-08-28
updated: 2026-08-28
type: concept
source_url: https://python.langchain.com/docs/tutorials/rag/
---

# LangChain RAG 实现

## 概述

LangChain 提供两套 RAG 写法：**确定性流水线**（显式 Index/Retrieve/Generate 编排）与 **Agent 化写法**（把"检索"封装成工具交给 `create_agent` 自主调用，即 Retrieval Agent）。2025 起官方 RAG 教程默认走 `create_agent` + `retrieve_context` 工具路线。本文覆盖两条路线的最小可运行实现、取舍与常见坑。

> 前置：02 层各环节原理；API 级组件见 [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]] 等页；LangGraph 图式 Agentic RAG 见 [[rag/14_范式与前沿/Agentic_RAG|Agentic_RAG]]。本库另有完整实现笔记：[[agent/RAG/RAG_LangChain完整实现|RAG LangChain 完整实现]]。

## 一、路线对比

| 维度 | 确定性流水线（Chain） | Agent 化（create_agent + retrieve_context） |
|------|----------------------|----------------------------------------------|
| 流程 | 固定：改写→检索→组装→生成 | LLM 自主决定何时检索/是否追问/多次检索 |
| 可预测性 | 高、延迟稳定 | 低（多轮工具调用） |
| 适用 | 知识库问答等单跳场景、强 SLA | 需判断"要不要查、查完够不够"的复杂场景 |
| 可观测 | 各步显式 | 追踪工具调用链（LangSmith） |
| 推荐起点 | 学习原理、固定流水线需求 | 官方教程默认、生产 Agent 化演进 |

## 二、路线 A：确定性 RAG 流水线（最小闭环）

```python
# pip install -qU langchain langchain-openai langchain-chroma langchain-text-splitters
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 加载 → 2. 分块 → 3. 向量化入库
from langchain_core.documents import Document
raw = Document(page_content="...你的企业知识文档...", metadata={"source": "doc_a.pdf"})
chunks = RecursiveCharacterTextSplitter(chunk_size=600, chunk_overlap=100).split_documents([raw])
vectorstore = Chroma.from_documents(chunks, OpenAIEmbeddings(model="text-embedding-3-small"))

# 4. 检索器（可加 k / 过滤）
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 5. Prompt 组装 + LLM 生成
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", "仅依据资料回答，标注引用编号 [i]，资料不足就明说。\n\n资料：\n{context}"),
    ("human", "{question}"),
])
chain = (
    {"context": retriever | format_docs, "question": lambda x: x["question"]}
    | prompt
    | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    | StrOutputParser()
)
answer = chain.invoke({"question": "公司年假政策是什么？"})
```

> 升级路径：把第 5 步换成 02 层的「混合检索 + 重排 + 引用编号组装」即得生产骨架。完整官方版见 LangChain RAG tutorial。

## 三、路线 B：Agent 化（create_agent + retrieve_context）

```python
# pip install -qU langchain "langchain[openai]" langchain-chroma
from langchain.agents import create_agent
from langchain.agents.tool_node import retrieve_context  # 内置检索工具

agent = create_agent(
    model="openai:gpt-4o",                    # 或 ChatDeepSeek 等任意受支持 provider
    tools=[retrieve_context(retriever)],      # 把检索器暴露成工具：agent 需要时才调用
    system_prompt=(
        "你是企业知识助手。先用检索工具查证，再回答；"
        "引用来源；检索不到就承认不知道。"
    ),
)
result = agent.invoke({"messages": [{"role": "user", "content": "我们 2026 年调休怎么安排？"}]})
# agent 自主：可能 1 次检索直接答；或改写查询后多次检索再综合
```

工作原理：`retrieve_context` 把"查询→向量检索→返回带来源文本"封装为工具；ReAct 循环中 LLM 生成 ToolCall → 执行检索 → 结果回填消息历史 → 继续推理直到给出最终答案。多轮消息自动携带，天然支持追问式对话。

## 四、工程化注意点

1. **Embedding 必须全链路一致**（入库/检索同一模型同一版本）；升级模型=重建集合。
2. **检索器与生成解耦**：把 retriever 抽象为接口，便于替换（BM25/混合/重排）而不动生成逻辑。
3. **流式输出**：`agent.stream(...)` / chain astream 逐 token；生产对话体验必需。
4. **持久化与断点**：Agent 化场景用 Checkpointer（SqliteSaver/PostgresSaver）恢复多轮会话，见 [[langchain/memory/Checkpointer存储后端|Checkpointer 存储后端]] 与 [[langgraph/tutorial/多轮对话Agent|多轮对话 Agent]]。
5. **别在 system prompt 里硬塞大段资料**——检索上下文应经工具结果进入消息，才能被追溯与审计。
6. 参考 LangChain v2 全家桶：追踪用 LangSmith（[[rag/12_评估与监控/在线监控与可观测性|生产可观测性]]），确定性流程追求可控时退回路线 A。

## 五、常见问题

**Q1：什么时候别用 Agent 化 RAG？**
固定单跳问答、强 SLA、或需要逐次可控输出时——Agent 多轮检索带来延迟与不可控性。可"默认流水线，检测到低置信/复杂查询才升级 Agent"混合架构。

**Q2：create_agent 与手写 LangGraph StateGraph 怎么选？**
create_agent 快、约定优于配置；需要自定义循环/条件边/人工审核节点时下沉到 LangGraph（LangChain v2 中 create_agent 本身基于 LangGraph，可无缝升级）。见 [[langgraph/tutorial/RAG_Agent|langgraph RAG_Agent]]。

**Q3：官方教程为何默认 retrieve_context 而非老式 RetrievalQA？**
LangChain v1 起官方路线收敛到 create_agent：检索即工具，行为可被追踪、可组合中间件（Checkpointer/Store），与 Agent 生态一致；RetrievalQA 等旧链逐渐淡出教程。

**Q4：检索工具结果太长撑爆上下文？**
给 retrieve_context 配小 k；或工具结果过 Summarization 类中间件压缩；生产用 [[rag/12_评估与监控/在线监控与可观测性|可观测性]] 盯每轮 token。

## 六、最佳实践

1. **先写 golden-queries 评估集再调实现**：任何框架/参数改动跑 [[rag/12_评估与监控/评估体系与RAGAS|RAGAS]] 回归，杜绝"感觉变好了"。
2. **检索与生成分离部署**：检索服务（向量库+重排）与生成（LLM 网关）可独立扩缩容（05 层架构）。
3. **为每个 answer 保留证据链**（检索块 id + 来源 url + 重排分数），这是审计与后续优化的原始数据。
4. 代码复用优先官方教程基线再按域改造，避免从零堆组件。

## 相关资料

- LangChain 官方 RAG 教程（create_agent + retrieve_context，持续更新）— https://python.langchain.com/docs/tutorials/rag/
- LangChain Retrieval 概念 — https://python.langchain.com/docs/concepts/retrieval/ · VectorStore — https://python.langchain.com/docs/concepts/vectorstores/
- 本库既有：[[agent/RAG/RAG_LangChain完整实现|RAG LangChain 完整实现]] · [[agent/RAG/RAG关键模块解析]] · [[langchain/modules/langchain_retrievers/自定义Retriever]]

## 相关页面

- 上一层导航：[[rag/RAG_学习路径导航|RAG 学习路径导航]]
- 下一篇：[[rag/14_范式与前沿/Agentic_RAG|Agentic RAG]]
