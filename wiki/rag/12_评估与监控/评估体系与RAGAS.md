---
tags: [rag, 评估, rag_metrics, ragas, llm-as-judge]
created: 2026-08-28
updated: 2026-09-17
type: concept
source_url: https://docs.ragas.io/en/stable/
---

# RAG 评估体系与 RAGAS

## 概述

RAG 评估分两个层次：**组件级**（检索质量：Recall@k、MRR 等，不需要 LLM）与**端到端级**（生成质量：答案是否忠于证据、是否答非所问——需要 LLM-as-judge 或标注）。RAGAS 是 2024–2026 最主流的开源端到端评估框架，核心思路：**用生成式 LLM 当裁判，自动算一组可解释指标**，把"感觉变好了"变成可回归的数字。

> 相关既有笔记：[[agent/RAG/RAG评估与优化|RAG 评估与优化]] · [[rag/12_评估与监控/离线评估集|离线评估集]]。本文给出完整指标体系 + 指标组合诊断法 + RAGAS v0.4 最小闭环 + 生产评估工程方法。
>
> ⚠️ **注意版本**：RAGAS 在 v0.4 做了破坏性重构，网上（及本库旧版）大量示例仍停留在 v0.1–v0.3 的 `evaluate()` 写法，已进入弃用通道。见第四节。

## 一、指标全景：三层金字塔

### 第 1 层：检索组件指标（离线、无需 LLM，最便宜）
| 指标 | 含义 | 何时看 |
|------|------|--------|
| Recall@k / Hit Rate | Top-k 里有多少相关文档 | 分块/检索器/混合检索改动 |
| Precision@k | Top-k 里相关占比 | 上下文污染排查 |
| MRR / nDCG | 首个命中位置 / 排序质量 | 重排器改动 |
> 注意这些指标需要"相关文档标注"或弱监督代理（如答案中包含的片段）。**只改检索时优先看这层**，快且无 LLM 抖动。

### 第 2 层：端到端生成指标（RAGAS，LLM-as-judge）
| 缩写 | 指标 | 衡量 | 通俗解释 |
|------|------|------|----------|
| **F** | **Faithfulness**（忠实度） | 答案中的每个论断是否被检索上下文支持 | "有没有照着证据说，还是自己编" |
| **AR**（旧称 Answer Relevancy） | **Response Relevancy**（回答相关性） | 答案是否切题、无废话 | "是不是答非所问" |
| **CP** | **Context Precision**（上下文精确率） | 相关片段是否排在前面 | "喂给模型的资料干不干净" |
| **CR** | **Context Recall**（上下文召回率） | 参考答案所需信息是否都被检索到 | "该找的都找齐没有" |
> v0.4 起 `Answer Relevancy` 正式更名为 `Response Relevancy`；老代码/老教程里的 `answer_relevancy` 指同一个指标。
> 生产建议经验区间（非硬性标准，需按场景标定）：Faithfulness > 0.90、Response Relevancy > 0.85；Context Precision/Recall 结合第 1 层一起看。低于该水平先查对应环节。

### 第 3 层：业务/系统指标
- 人工黄金标注的端到端准确率、引用正确率（citation accuracy，逐条核答案标注的来源编号）。
- 系统：延迟 P95、成本/请求、缓存命中率、失败率（结合 [[rag/12_评估与监控/在线监控与可观测性|生产可观测性]]）。

## 二、四个指标的缩写与计算机制

理解机制才能理解"分数为什么低"。四个指标全部依赖 LLM 做分解与判断，**裁判模型的推理能力直接决定分数质量**。

| 缩写 | 全称 | 计算机制 | 需 reference |
|------|------|----------|--------------|
| **F** | Faithfulness | 把回答拆成原子声明（claims）→ 逐条判断能否被 context 支撑 → `支撑数 / 总声明数`。机制上等价于对幻觉率取补 | 否 |
| **AR** | Response Relevancy | 用 LLM 从回答**反向生成 3 个问题** → 计算这 3 个问题与原始问题的 embedding 余弦相似度 → 取均值。逻辑是"若答案切题，原问题就能从答案反推出来" | 否 |
| **CP** | Context Precision | 逐条判断每个检索到的 chunk 对回答是否有用 → 按**排序位置加权**计算平均精度，排在前面的相关 chunk 权重更高 | 是 |
| **CR** | Context Recall | 把 reference 拆成若干句子 → 逐句判断能否在 context 中找到支撑 → 覆盖率 | 是 |

**关键依赖**：`AR` 需要 embedding（算反向问题的相似度），其余三个只需 LLM。这是接入自研/国产模型时最容易踩的坑——见第七节 Q5。

> ⚠️ `AR` 的取值**不保证落在 0–1**，因为余弦相似度的数学范围是 −1~1。这是官方明确说明的边界情况。

## 三、核心方法论：指标组合诊断矩阵

**单个分数没有意义，指标组合才有诊断力。** 这是把 RAGAS 从"打分工具"用成"排障工具"的关键：

| 症状组合 | 根因定位 | 改进方向 |
|----------|----------|----------|
| F 高 · CR 低 | 检索不全。回答忠实于已有上下文，但上下文本身缺信息 | 调 chunking 粒度、提高 top-k、混合检索、查询改写 |
| **F 低 · CR 高** | **最危险**。检索是对的，模型在正确文档上编造，且回答常引用来源、极具说服力 | 强化"仅依据上下文作答"约束、换更强模型、降 temperature、加引用校验 |
| CP 低 · CR 高 | 召回全但噪声多，信噪比差 | 加 reranker（[[rag/08_交叉编码器重排/重排序与交叉编码器|Cross-Encoder]]）、提高相似度阈值、元数据过滤 |
| CP 高 · CR 低 | 检索精准但覆盖不足 | 补库、多路召回（[[rag/05_查询转换与改写/查询改写与扩展|查询改写]]） |
| F 高 · AR 低 | 回答不跑偏但答非所问 | 检查 prompt 结构、query 意图理解、多轮上下文是否串味 |
| 四项均高 · 人工仍不满 | 评估集不具代表性，或质量问题落在 RAGAS 测不到的维度（延迟、格式、口吻） | 重建评估集，引入自定义 criteria 指标 |

**指标与环节的绑定关系**（决定"改了什么该看哪项"）：

| 你改了什么 | 该看的指标 |
|------------|-----------|
| 分块策略 / 解析器 | CR、CP（配第 1 层 Recall@k） |
| Embedding 模型 / 向量库 | CR、CP（**注意：换 embedding = 全量重建索引**） |
| 混合检索权重 / 加 BM25 | CP、CR |
| 加 reranker | CP、MRR/nDCG |
| Prompt / 生成模型 | F、AR |
| 上下文组装 / 压缩 | F、AR |

## 四、RAGAS 版本演进与 API 变更（时效性关键）

**当前状态（核实于 2026-09-17）**：最新稳定版 `ragas==0.4.3`（2026-01-13 发布，此后无新版）。仓库已从 `explodinggradients/ragas` 迁移至 `vibrantlabsai/ragas`（旧地址重定向）。项目处于低活跃状态，但并未废弃。

**v0.3 → v0.4 破坏性变更**：

| 维度 | v0.3（大量旧教程） | v0.4（当前） |
|------|--------------------|--------------|
| 评估入口 | `evaluate(dataset, metrics=[...])` | `@experiment` 装饰器（`evaluate` 已弃用） |
| 指标导入 | `from ragas.metrics import Faithfulness` | `from ragas.metrics.collections import Faithfulness` |
| LLM 初始化 | `LangchainLLMWrapper(...)` | `llm_factory(model, client=...)` 统一入口 |
| 打分调用 | `single_turn_ascore(SingleTurnSample(...))` | `ascore(**kwargs)` 关键字直接传参 |
| 返回类型 | `float` | `MetricResult(.value, .reason)` |
| 标准答案字段 | `ground_truths`（列表） | `reference`（单个字符串） |

**`.reason` 字段是升级的充分理由**：`0.62` 这个分数本身毫无信息量，裁判给出的扣分理由才是定位问题的线索。

**依赖冲突（实测确认，2026-09）**：
- `ragas 0.4.3` 会 import 一个在 `langchain-community 0.4.2` 中被移除的模块 → 需锁 `langchain-community==0.4.1`
- `instructor 1.16.0` 声明 `openai>=2,<3`，与 `openai 3.x` 不兼容 → openai 需锁 2.x

## 五、最小可运行示例（v0.4 collections API）

```bash
# 版本组合互相制约，必须锁定，不要盲目升级
pip install "ragas==0.4.3" "openai>=2,<3" "instructor==1.16.0" "langchain-community==0.4.1"
```

```python
import asyncio, os
from openai import AsyncOpenAI
from ragas.llms import llm_factory
from ragas.embeddings.base import embedding_factory
from ragas.metrics.collections import (
    Faithfulness, AnswerRelevancy, ContextPrecision, ContextRecall,
)

# ---- 1. 裁判模型（以 OpenAI 兼容端点为例，可换任意兼容服务） ----
judge_client = AsyncOpenAI(
    api_key=os.environ["JUDGE_API_KEY"],
    base_url=os.environ.get("JUDGE_BASE_URL"),   # 自建/国产模型填兼容端点
    timeout=60.0,        # 评估长上下文时不设超时容易挂起
    max_retries=2,
)
judge_llm = llm_factory("gpt-4o-mini", client=judge_client)

# AR 指标需要 embedding，必须单独提供（裁判模型的供应商未必有 embedding 端点）
emb_client = AsyncOpenAI(api_key=os.environ["EMB_API_KEY"], base_url=os.environ["EMB_BASE_URL"])
embeddings = embedding_factory("openai", model="text-embedding-3-small", client=emb_client)

f  = Faithfulness(llm=judge_llm)                          # 生成侧，只需 llm
ar = AnswerRelevancy(llm=judge_llm, embeddings=embeddings) # 生成侧，llm + embeddings
cp = ContextPrecision(llm=judge_llm)                      # 检索侧
cr = ContextRecall(llm=judge_llm)                         # 检索侧

# ---- 2. 逐样本打分（0.4 直接传关键字参数，不再包 SingleTurnSample） ----
async def score_one(row):
    r_f  = await f.ascore(response=row["response"],
                          retrieved_contexts=row["retrieved_contexts"])
    r_ar = await ar.ascore(user_input=row["user_input"],
                           response=row["response"])
    r_cr = await cr.ascore(user_input=row["user_input"],
                           retrieved_contexts=row["retrieved_contexts"],
                           reference=row["reference"])   # 需 reference
    r_cp = await cp.ascore(user_input=row["user_input"],
                           retrieved_contexts=row["retrieved_contexts"],
                           reference=row["reference"])
    return {
        **row,
        "faithfulness": r_f.value, "answer_relevancy": r_ar.value,
        "context_recall": r_cr.value, "context_precision": r_cp.value,
        "faithfulness_reason": r_f.reason,   # 裁判给的理由，比分数有用
    }

# ---- 3. 批量执行（并发上限约 5，再高会触发裁判侧限流） ----
async def evaluate_all(rows, concurrency=5):
    sem = asyncio.Semaphore(concurrency)
    async def guarded(r):
        async with sem:
            return await score_one(r)
    return await asyncio.gather(*(guarded(r) for r in rows))
```

要点：
- **逐样本看"哪个 case 哪项低"**，比只看均值有用得多；配合 `.reason` 直接定位。
- 测试集 50–200 条代表性问题起步，覆盖各文档类型与查询难度；**质量问题优先于数量**。
- 裁判 LLM 需与答案语言一致；中文场景避免用对中文不敏感的裁判模型。
- 中英混排语料应**按语言组合分片统计**（中文问→中文答 / 中文问→英文答 / 英文问→英文答），整体均值会掩盖跨语言短板。
- 若需官方推荐的 `@experiment` 装饰器入口（附带 git 版本跟踪），见官方迁移指南；`evaluate()` 写法仍可运行但会发出弃用警告。

## 六、评估工程化（生产必做）

1. **Golden set 来源**：真实用户 query 采样 + 失败 case 回流 + 领域专家补充。每季度审计一次过时样本。
2. **LLM-as-judge 的可信度**：抽样 30–50 条与人工标注对拍（一致率、Cohen's kappa）；定期复查裁判漂移（裁判模型升级后要回归）。
3. **回归门槛（CI）**：把评估写成 CI 任务，合并代码/改索引前跑：`Faithfulness 下降 > 0.03 → 阻断`。数据与 Prompt 改动都要触发。
4. **双维度决策**：端到端指标（是否变好）+ 组件指标（哪里变好），避免"总分升了但检索更差（靠生成硬撑）"的误判。
5. **RAG Triad（TruLens 视角）可作对照**：Context Relevance / Groundedness / Answer Relevance，概念与 RAGAS 互补，多框架交叉验证更稳。

## 七、常见问题

**Q1：LLM-as-judge 本身会错，评估结果可信吗？**
会有系统偏差（如偏好更长答案、裁判与模型同源偏见）。缓解：① 裁判用不同供应商/更强模型；② 关键变更先人工抽样复核；③ 指标只做**相对回归**（同一评估配置下 A/B），别当绝对真理。

**Q2：分数是绝对的还是相对的？**
**相对的。** 0.78 单独看没有意义，只有与"同一指标配置下的历史基线"比较才有含义。**换裁判模型后所有基线分数会整体漂移，历史对比立即失效，必须重新建基线。** 同时 LLM 有随机性，同输入重跑分数会小幅波动，应多次取均值而非单跑定论。

**Q3：Context Recall 需要 reference，人工标不动怎么办？**
妥协方案：reference 用"正确答案在哪些文档里"的文档级标注替代块级；或用 LLM 生成答案后人工抽检。块级精确标注只在关键子集上做。

**Q4：评估成本高（每样本多次 LLM 调用）怎么办？**
① 先跑组件级指标（免费）快速筛；② 端到端评估用更小测试集 + 更小裁判模型 + 批处理；③ 只对变更影响面相关的子集回归。

**Q5：用国产模型/自建模型当裁判，要注意什么？**
① **确认是否有 embedding 端点**——`Response Relevancy` 依赖 embedding，若裁判模型供应商不提供 embedding，必须另配一个来源；② 确认结构化输出（JSON mode / function calling）可用，0.4 的 collections 指标依赖 `instructor` 式结构化输出；③ 中文场景验证裁判的中文判断力，必要时覆写指标 prompt（官方默认 prompt 以英文为主）；④ `max_tokens` 要设够，评估长文档时不足会导致打分静默截断。

**Q6：线上和离线指标对不上？**
正常且常见。离线测"标准问题集"，线上真实查询分布不同。解法：线上采样真实流量回流到 golden set（见 [[rag/12_评估与监控/在线监控与可观测性|可观测性]] 的反馈回路），让离线集持续逼近真实分布。

## 八、最佳实践

1. **评估先行**：任何 RAG 改动（分块、检索、Prompt、模型）都应有指标背书，否则视为未完成。
2. **指标与环节绑定**：改检索→看 CR/CP；改 Prompt→看 F/AR；别拿端到端总分定位一切（绑定关系见第三节）。
3. **保存每次实验的完整快照**（代码版本、索引版本、评估集 hash、裁判模型与版本、分数），保证可回溯可比。
4. 定期把"线上翻车 case"加入 golden set 并回归——这是 RAG 持续变好的核心飞轮。

## 相关资料

- RAGAS 官方文档（指标定义与 API）— https://docs.ragas.io/en/stable/
- RAGAS v0.3→v0.4 迁移指南 — https://docs.ragas.io/en/latest/howtos/migrations/migrate_from_v03_to_v04/
- TruLens RAG Triad — https://www.trulens.org/getting_started/core_concepts/rag_triad/
- LangSmith 评估（生产级 LLM 评估平台）— https://docs.smith.langchain.com/evaluation
- 检索指标背景（Recall/MRR/nDCG）— https://python.langchain.com/docs/concepts/retrieval/
- 本库既有：[[agent/RAG/RAG评估与优化|RAG 评估与优化]] · [[agent/RAG/RAG参数调优手册]]

## 相关页面

- 上一篇：[[rag/14_范式与前沿/Agentic_RAG|Agentic RAG]]
- 同层：[[rag/12_评估与监控/离线评估集|离线评估集]] · [[rag/12_评估与监控/在线监控与可观测性|生产可观测性]]
- 相关：[[rag/10_LLM生成/生成校验与防幻觉|生成校验与防幻觉]] · [[rag/09_上下文构建与溯源/溯源与引用|溯源与引用]]
