---
tags: [concept, RAG, 评估, 优化, RAGAS, 指标]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 评估与优化

> RAG 系统的评估是持续优化的基础。通过量化指标识别瓶颈，针对性调优，才能从"能用"走向"好用"。

## 评估维度

RAG 系统的评估分为两个层面：**检索质量**和**生成质量**。

### 检索质量指标

| 指标 | 定义 | 计算方式 | 目标 |
|------|------|---------|------|
| **Context Precision** | 检索结果中相关文档的排名 | 相关文档在 Top-K 中的平均排名位置 | 越高越好（相关文档应排在前面） |
| **Context Recall** | 所有相关文档被检索到的比例 | 检索到的相关文档数 / 总相关文档数 | 越高越好（不遗漏关键信息） |
| **Context Relevancy** | 检索结果与问题的相关程度 | LLM 评分或人工标注 | 越高越好（减少噪声） |

### 生成质量指标

| 指标 | 定义 | 计算方式 | 目标 |
|------|------|---------|------|
| **Faithfulness** | 回答是否忠实于检索到的上下文 | 检查回答中的每个声明是否能在上下文中找到依据 | 越高越好（减少幻觉） |
| **Answer Relevancy** | 回答与问题的相关程度 | LLM 评分或人工标注 | 越高越好（不答非所问） |
| **Answer Correctness** | 回答与标准答案的一致性 | 与 ground truth 对比（语义相似度） | 越高越好 |

### 综合指标

| 指标 | 定义 | 说明 |
|------|------|------|
| **Hallucination Rate** | 幻觉率 | 回答中包含上下文中不存在信息的比例 |
| **Answer Similarity** | 答案相似度 | 与标准答案的语义相似度（0-1） |
| **Latency** | 延迟 | 端到端响应时间（检索 + 生成） |
| **Cost** | 成本 | Token 消耗、API 调用费用 |

## RAGAS 评估框架

[RAGAS](https://github.com/explodinggradients/ragas) 是最流行的 RAG 评估框架，提供自动化评估能力。

### 安装与基础用法

```bash
pip install ragas
```

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 准备评估数据
eval_data = {
    "question": [
        "什么是 RAG？",
        "RAG 有哪些局限？",
    ],
    "answer": [
        "RAG 是检索增强生成，通过检索外部知识来增强 LLM 的回答能力。",
        "RAG 的局限包括无累积性、无综合能力、检索精度瓶颈。",
    ],
    "contexts": [
        ["RAG（Retrieval-Augmented Generation）是一种将外部知识源与 LLM 结合的技术。"],
        ["RAG 的核心局限：1. 无累积性 2. 无综合能力 3. 检索精度瓶颈"],
    ],
    "ground_truth": [
        "RAG 是检索增强生成，通过检索外部知识来增强 LLM 的回答能力。",
        "RAG 的局限包括无累积性、无综合能力和检索精度瓶颈。",
    ],
}

dataset = Dataset.from_dict(eval_data)

# 执行评估
results = evaluate(
    dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall,
    ],
)

print(results)
# {'faithfulness': 0.95, 'answer_relevancy': 0.88, ...}
```

### 指标详解

#### Faithfulness（忠实度）

衡量回答中的每个声明是否能在检索到的上下文中找到依据。

```python
from ragas.metrics import faithfulness

# 评估逻辑：
# 1. 从回答中提取所有声明（claims）
# 2. 检查每个声明是否能在上下文中找到支持
# 3. faithfulness = 有依据的声明数 / 总声明数

# 示例：
# 回答："Python 是 1991 年发布的，作者是 Guido van Rossum。"
# 上下文："Python 由 Guido van Rossum 于 1991 年首次发布。"
# faithfulness = 2/2 = 1.0
```

#### Answer Relevancy（回答相关性）

衡量回答与问题的相关程度。

```python
from ragas.metrics import answer_relevancy

# 评估逻辑：
# 1. 从回答生成多个问题（反向生成）
# 2. 计算原始问题与生成问题的语义相似度
# 3. 相似度越高，回答越相关
```

#### Context Precision（上下文精确度）

衡量检索结果中相关文档的排名质量。

```python
from ragas.metrics import context_precision

# 评估逻辑：
# 1. 对检索结果中的每个文档判断是否相关
# 2. 计算 Precision@K（前 K 个结果中相关文档的比例）
# 3. 加权平均（排名靠前的相关文档权重更高）
```

#### Context Recall（上下文召回率）

衡量所有相关文档被检索到的比例。

```python
from ragas.metrics import context_recall

# 评估逻辑：
# 1. 从 ground_truth 中提取关键信息
# 2. 检查这些信息是否能在检索到的上下文中找到
# 3. recall = 找到的关键信息数 / 总关键信息数
```

## 评估数据集构建

### 方法一：人工标注

```python
eval_data = [
    {
        "question": "LangChain 的 Agent 如何工作？",
        "ground_truth": "LangChain Agent 通过 ReAct 循环工作：推理→行动→观察，直到任务完成。",
        "contexts": [
            "LangChain Agent 使用 ReAct 框架，在循环中调用工具直到任务完成。",
            "Agent = 模型 + 框架，框架负责工具调用和状态管理。",
        ],
    },
    # ... 更多样本
]
```

### 方法二：LLM 生成

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate

llm = ChatOpenAI(model="gpt-4o-mini")

# 从文档生成问答对
prompt = PromptTemplate.from_template("""
根据以下文档生成一个问答对：

文档：{document}

输出格式：
问题：...
答案：...
""")

def generate_eval_pair(document):
    response = llm.invoke(prompt.format(document=document))
    # 解析 response 提取 question 和 answer
    return {"question": "...", "ground_truth": "..."}
```

### 方法三：从生产日志提取

```python
# 从实际用户查询中提取评估样本
def extract_from_logs(user_queries, retrieved_docs, generated_answers):
    """从生产日志提取评估数据"""
    eval_data = []
    for query, docs, answer in zip(user_queries, retrieved_docs, generated_answers):
        eval_data.append({
            "question": query,
            "contexts": [doc.page_content for doc in docs],
            "answer": answer,
            # ground_truth 需要人工标注或 LLM 生成
        })
    return eval_data
```

## 优化策略

### 检索优化

| 问题 | 优化策略 | 预期效果 |
|------|---------|---------|
| 召回率低 | 混合检索（BM25 + 向量）、Multi-Query | 召回率提升 20-40% |
| 精确度低 | 重排序（Cross-Encoder）、元数据过滤 | 精确度提升 15-30% |
| 上下文不完整 | Parent Document Retriever、增大 chunk_size | 上下文完整性提升 |
| 查询表述不佳 | 查询重写、HyDE（假设性文档嵌入） | 命中率提升 10-25% |

### 生成优化

| 问题 | 优化策略 | 预期效果 |
|------|---------|---------|
| 幻觉 | 强化 Prompt 约束、降低 temperature | 幻觉率降低 30-50% |
| 答非所问 | 改进 Prompt 模板、Few-shot 示例 | 相关性提升 |
| 回答冗长 | 限制 max_tokens、Prompt 指示简洁 | 回答质量提升 |
| 缺少引用 | Prompt 要求标注来源、后处理映射 | 可追溯性提升 |

### 端到端优化

```python
from langchain_openai import ChatOpenAI
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

# 优化后的 RAG 链
def create_optimized_rag_chain(vectorstore, llm):
    # 1. 混合检索 + 重排序
    retriever = vectorstore.as_retriever(
        search_type="mmr",
        search_kwargs={"k": 10, "fetch_k": 20, "lambda_mult": 0.7}
    )
    
    # 2. 优化的 Prompt 模板
    system_prompt = """你是一个专业的问答助手。请严格遵循以下规则：

规则：
1. 仅基于提供的上下文回答问题
2. 如果上下文中没有相关信息，明确说"根据现有资料无法回答"
3. 回答时标注信息来源，格式：[文档N]
4. 保持回答简洁，不超过 200 字

上下文：
{context}"""

    prompt = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{input}"),
    ])
    
    # 3. 组装链
    question_answer_chain = create_stuff_documents_chain(llm, prompt)
    return create_retrieval_chain(retriever, question_answer_chain)
```

## 持续监控

### 关键指标监控

```python
import time
from dataclasses import dataclass

@dataclass
class RAGMetrics:
    latency_ms: float
    tokens_used: int
    faithfulness: float
    relevancy: float
    hallucination_rate: float

def monitor_rag_system(rag_chain, eval_dataset):
    """定期评估 RAG 系统质量"""
    metrics = []
    
    for sample in eval_dataset:
        start = time.time()
        result = rag_chain.invoke({"input": sample["question"][[})
        latency = (time.time() - start) * 1000
        
        # 评估质量指标
        # ... 使用 RAGAS 或其他评估工具
        
        metrics.append(RAGMetrics(
            latency_ms=latency,
            tokens_used=result.get("usage", {}).get("total_tokens", 0),
            faithfulness=0.9,  # 实际评估值
            relevancy=0.85,
            hallucination_rate=0.05,
        ))
    
    return metrics
```

### 告警阈值

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| Faithfulness | < 0.8 | 幻觉率过高 |
| Context Recall | < 0.7 | 检索遗漏关键信息 |
| Latency | > 3000ms | 响应过慢 |
| Cost per Query | > $0.05 | 成本过高 |

## 相关页面

- [[agent/RAG/RAG核心概念与演进]] — RAG 定义与三代演进
- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[agent/RAG/RAG混合检索与重排序]] — 混合检索详解

## 来源

- RAGAS 官方文档：https://docs.ragas.io/
- 2025-2026 RAG 评估最佳实践
- LangSmith 评估指南
