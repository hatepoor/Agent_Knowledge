---
tags: [rag, 框架实战, agentic, langgraph, self-rag, crag]
created: 2026-08-28
updated: 2026-08-28
type: concept
source_url: https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_agentic_rag/
---

# Agentic RAG

## 概述

Agentic RAG 把检索从"固定流水线的一步"升级为 **LLM（Agent）自主编排的行为**：何时检索、检索几次、改写与否、结果够不够、要不要换路（本地库/联网/直接答）都由 Agent 决策。2024–2026 的代表方案：Adaptive-RAG、Self-RAG、Corrective RAG（CRAG），以及 LangGraph 官方的 agentic-rag 参考图。

> LangGraph 图式实现见 [[langgraph/tutorial/RAG_Agent|langgraph RAG_Agent]]；`create_agent + retrieve_context` 的轻量 Agent 化见 [[rag/00_导览与基础/LangChain_RAG端到端实现|LangChain RAG 实现]]。区别：后者是"单工具自主调用"，本文是可自定义条件循环的完整图。

## 一、为什么需要 Agentic RAG（固定流水线的局限）

| 固定流水线的失败模式 | Agentic RAG 的应对 |
|---------------------|---------------------|
| 与领域无关的问题也强制检索 | 路由：直接答 / 查库 / 联网 |
| 查询表述差导致 0 命中 | 检测到低分 → 改写查询重试（rewrite 节点） |
| 检索结果鱼龙混杂 | 文档相关性评分节点过滤后再生成（grade） |
| 多跳问题一次检索不够 | 循环：生成→自查缺啥→再检索 |
| 无法判断"资料够不够答" | 生成后校验或反思（reflection） |

**何时值得上 Agentic**：查询分布复杂（多跳/多意图/需判断时效）、可接受一定延迟与成本。单跳固定问答反而用确定性流水线更优——Agentic 是"用推理成本换检索精准"，要算账。

## 二、三大代表论文速览

| 方案 | 核心机制 | 出处 |
|------|----------|------|
| **Adaptive-RAG** | 路由分类器按查询复杂度选：直接答 / 单次检索 / 多轮检索 | arXiv:2403.14403 |
| **Self-RAG** | 检索+反思 token：是否需要检索、片段是否相关、答案是否被支持，自主决定检索次数 | arXiv:2310.11511 |
| **CRAG (Corrective RAG)** | 检索结果评分：高→直接生成；低→触发 Web 搜索修正；中→按需精炼 | arXiv:2401.15884 |

共同点：把"质量判断"交给模型（LLM-as-judge 内嵌），把"是否检索"从硬编码变成条件边。

## 三、LangGraph 参考实现（官方 Agentic RAG 图）

```python
# 节点：agent(决定动作) / retrieve / grade_docs(相关性把关) / rewrite / generate
from typing import Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing import Annotated, TypedDict

class GraphState(TypedDict):
    question: str
    generation: str
    documents: list[str]
    messages: Annotated[list, add_messages]

def agent(state):      # 1. 决定动作（检索 or 直接生成）
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

def retrieve(state):   # 2. 执行检索
    docs = retriever.invoke(state["question"])
    return {"documents": docs, "question": state["question"]}

def grade_docs(state): # 3. 对每个片段做相关性评分（LLM-as-judge）
    filtered = [d for d in state["documents"] if is_relevant(state["question"], d)]
    return {"documents": filtered}

def rewrite(state):    # 4. 查询改写后重回检索
    better = rewrite_llm.invoke(f"改写查询以更好检索：{state['question']}")
    return {"question": better.content}

def generate(state):   # 5. 生成
    return {"generation": gen_llm.invoke(build_prompt(state["question"], state["documents"]))}

def route(state) -> Literal["retrieve", "generate"]:
    last = state["messages"][-1]
    return "retrieve" if last.tool_calls else "generate"

def decide_after_retrieve(state) -> Literal["generate", "rewrite"]:
    return "generate" if state["documents"] else "rewrite"   # 空召回 → 改写重试

graph = StateGraph(GraphState)
graph.add_node(agent); graph.add_node(retrieve); graph.add_node(grade_docs)
graph.add_node(rewrite); graph.add_node(generate)
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", route, {"retrieve": "retrieve", "generate": END})
graph.add_edge("retrieve", "grade_docs")
graph.add_conditional_edges("grade_docs", decide_after_retrieve,
                            {"generate": "generate", "rewrite": "rewrite"})
graph.add_edge("rewrite", "retrieve"); graph.add_edge("generate", END)
app = graph.compile()

answer = app.invoke({"question": "LangGraph 的 Send API 和动态扇出有什么关系？"})
print(answer["generation"])
```

要点：
- **grade/decide 用结构化输出**（如 `{"relevant": true/false, "confidence": 0.9}`），比自由文本更稳。
- **给循环设上限**（max_rewrites=2），防止"改写→检索→再改写"死循环烧钱。
- 官方完整版（含联网 fallback 与答案生成校验）见 LangGraph agentic-rag tutorial。
- 需要多轮对话/多 Agent 协同编排见 [[langgraph/tutorial/多轮对话Agent|多轮对话 Agent]]；多租户隔离见 05 层。

## 四、生产化的关键考量

| 维度 | 建议 |
|------|------|
| 成本 | 每多一次工具调用=多一次 LLM 推理；用路由前置拦截简单问题 |
| 延迟 | 设定 max steps；评分可用小模型（如 gpt-4o-mini/本地 7B） |
| 可观测 | 记录"路由决策、检索次数、改写历史、被过滤片段"供复盘（04 层） |
| 稳定性 | 给条件边返回类型加 Literal 约束；错误走兜底生成而非崩溃 |
| 评估 | 对"该检索却直接答/该直接答却检索"两类错误分别统计（决策质量也是指标） |

## 五、常见问题

**Q1：Agentic RAG 和普通 Agent 的区别？**
普通 Agent 工具集宽（搜索/代码/API），RAG Agent 聚焦"知识检索"这一核心动作并围绕它做质量控制循环（改写/评分/兜底）。实践中 Agentic RAG 常作为大 Agent 的一个子技能存在。

**Q2：Self-RAG/CRAG 的"反思"会不会反而更不稳？**
会，判断模型不准时越绕越偏。缓解：评分任务用强模型+结构化输出；把"检索后直接生成"作为对照基线做消融，收益不显著就别上。

**Q3：需要人工介入（human-in-the-loop）的场景怎么接？**
LangGraph 的 interrupt/checkpoint 可在 grade_docs 或生成前暂停等待审核，适合高风险内容审核（配合 [[rag/11_安全审核与日志/安全与合规|安全与合规]]）。

**Q4：多轮对话中 Agent 何时用历史、何时重新检索？**
把"历史摘要"注入 agent 节点消息，让模型判断"老话题追问=直接答/局部检索"或"新话题=全新检索"。这也是长短记忆设计的交界（见 [[langchain/memory/短期记忆|langchain 短期记忆]] 与 [[langchain/memory/长期记忆|长期记忆]]）。

## 六、最佳实践

1. **默认图骨架：agent→(route)→retrieve→grade→(rewrite 循环)→generate**，先跑通再叠加 Web 兜底/人工节点。
2. **检索失败路径必须显式设计**：0 命中、全被过滤、改写两次仍空——每种都要有明确行为（联网/承认不知/降级摘要）。
3. **把 Agentic RAG 的"决策"也纳入评估**：用决策正确率 + 端到端指标双维度回归。
4. 全部可复现：固化随机种子、模型版本、检索 k；用 LangSmith 回放失败 case 到 golden set。

## 相关资料

- LangGraph 官方 Agentic RAG 教程（完整图+可运行代码）— https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_agentic_rag/
- Self-RAG 论文 — https://arxiv.org/abs/2310.11511 · CRAG — https://arxiv.org/abs/2401.15884 · Adaptive-RAG — https://arxiv.org/abs/2403.14403
- LangChain Agent 概念（create_agent）— https://python.langchain.com/docs/concepts/agents/
- 本库既有：[[langgraph/tutorial/RAG_Agent]] · [[agent/RAG/RAG关键模块解析]]

## 相关页面

- 上一篇：[[rag/00_导览与基础/LangChain_RAG端到端实现|LangChain RAG 实现]]
- 下一篇：[[rag/12_评估与监控/评估体系与RAGAS|RAG 评估体系与 RAGAS]]
