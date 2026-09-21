---
tags: [rag, 前沿扩展, multimodal, 多模态, long-context, 长上下文]
created: 2026-08-28
updated: 2026-08-28
type: concept
source_url: https://docs.llamaindex.ai/en/stable/
---

# 多模态与长上下文 RAG

## 概述

2025–2026 RAG 的两条演进主线：**多模态**（语料从纯文本扩展到图片/表格/PDF 版式/音视频）与**长上下文**（上下文窗口从 128k 走向 1M+，但"塞得下"不等于"找得到"）。二者分别回答"知识形态变宽了怎么接"与"窗口变大了 RAG 还有没有用"。本文给出多模态 RAG 的四种架构路线与长上下文时代 RAG 的正确打开方式。

> 相关既有内容：[[agent/RAG/多模态RAG|agent 域多模态 RAG]]。成本/上下文预算联动见 [[rag/13_工程横切/性能与成本优化|性能与成本优化]]。

## 一、多模态 RAG：四种架构路线

| 路线 | 做法 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| A. 转文本（captioning） | 用 VLM 为图片生成描述/OCR，走**标准文本 RAG** | 零新基建、复用全部 02–05 层 | 信息损失（图表细节/空间关系） | 通用文档、成本敏感 |
| B. 统一多模态向量 | CLIP 类多模态 embedding，图与文同空间检索 | 原生跨模态检索（文找图/图找图） | 模型生态有限、领域精度待验证 | 图库检索、图文配对 |
| C. 分模态索引+路由 | 各模态独立 embedding 与索引，路由按查询类型选 | 各模态用最优模型 | 工程复杂、需路由 | 混合语料生产方案 |
| D. 视觉文档检索（Late Interaction） | ColPali 类：整页视觉编码 + 多向量交互排序，**跳过解析直接检索 PDF 版面** | 版面/表格/图表信息零丢失、省去解析管道 | 需专用检索模型与算力 | **扫描件/复杂版式 PDF 首选趋势** |

生产务实组合：**默认 A**（文本化最省），版面信息是核心价值时升级 **D**，检索"图与文互查"再加 **B**。

```python
# 路线 A 示例：图片 → VLM 描述 → 进入标准文本分块/向量流水线
from langchain_openai import ChatOpenAI
vlm = ChatOpenAI(model="gpt-4o", temperature=0)   # 或本地 Qwen2.5-VL 等

def describe_image(path: str) -> str:
    # 简化示意：真实实现需 base64/多模态消息格式
    resp = vlm.invoke([{"type": "text", "text": "详细描述此图内容与关键数据"},
                       {"type": "image_url", "image_url": {"url": path}}])
    return resp.content

# 把"图片描述文本"与邻近正文一起分块入索引，检索命中图片内容时返回原图引用
```

## 二、长上下文与 RAG：互补而非替代

**关键事实（Lost in the Middle，Liu et al. 2023/2024）**：LLM 对超长上下文中**中部内容**的利用显著差于头尾——"窗口够大"不等于"检索无用"。长上下文解决的是**单篇深读**，RAG 解决的是**海量选对**：

| 场景 | 方案 |
|------|------|
| 单篇 500 页 PDF 深读问答 | 长上下文直读（或 长上下文+分段精读） |
| 10 万文档中找对的 5 段 | RAG（长上下文装不下，也不需要装） |
| 多跳+需全库归纳 | RAG/GraphRAG 综合（[[rag/14_范式与前沿/GraphRAG与知识图谱|GraphRAG]]） |
| 混合 | "长上下文 + 精筛"：RAG 先精筛，命中后把整篇/整章喂长窗口深度作答 |

### 长上下文时代 RAG 的四个新打法
1. **块更大、更"文档级"**：窗口变大使父子分块的父块可放大到整节/整章，检索子块、精读全文。
2. **Contextual Retrieval（Anthropic）**：块嵌入前用 LLM 生成上下文前缀（"这块在全文中的位置与作用"），检索失败率 -49%，加重排 -67%；配合 **prompt caching** 摊薄上下文化成本。
3. **Late Chunking**：长上下文 embedding 一次编码全文、句边界池化，块向量携带全文语境（详见 [[rag/03_智能分块/文档分块策略|分块策略]]）。
4. **多轮/记忆型 RAG**：长窗口容纳更完整的历史与中间推理，Agentic RAG 的重写/反思轨迹不必频繁截断（03 层）。

## 三、常见问题

**Q1：有了 1M 上下文，把全部文档塞进 prompt 不就行了？**
不行，三点：① 中部遗忘（Lost in the Middle）让超长直读不可靠；② 成本随 token 线性暴涨（1M token/请求 的单价远超多次检索）；③ 无索引=无引用溯源与权限过滤，安全与合规（05 层）全丢。正确姿势是"检索做粗筛、长窗口做精读"。

**Q2：多模态用 VLM 描述图片，描述错了怎么办？**
描述质量=检索质量。缓解：描述要求结构化（关键对象/数据/文字）；描述与原文一起存储供溯源；对关键图表抽检；必要时保留原图，命中后把原图直接给多模态 LLM 二次确认。

**Q3：ColPali 这类视觉检索贵不贵、值不值？**
贵（每页视觉编码算力开销大），但省掉了解析管道与解析错误，复杂版式/扫描文档场景整体 ROI 常为正。建议先用小样本对比"文本化 vs 视觉检索"的 Recall 再定。

**Q4：多模态检索和纯文本检索的评估指标通用吗？**
部分通用（Recall@k、Faithfulness），但多模态要加"描述保真度、版面还原度、图文对应正确率"等专用指标，评估集需含图文样本（04 层扩展）。

## 四、最佳实践

1. **多模态先问"信息在哪"**：答案在表格数值/版面结构里 → 别用纯文本化；答案在语义描述里 → 文本化足够。按文档类型分治。
2. **长上下文不是 RAG 的敌人而是工具**：把"长窗口精读"作为检索后的第二阶段（retrieve→read full→answer），兼顾召回与深度。
3. **上下文预算显式管理**：检索预算 + 全文精读预算分开计，P95 与成本才可控。
4. 跟踪模型能力变化（窗口、视觉、长上下文 embedding），每季度评估一次"哪些场景可以切到更省/更强的新路线"。

## 相关资料

- LlamaIndex 多模态 RAG 文档 — https://docs.llamaindex.ai/en/stable/ · LangChain 多模态 — https://python.langchain.com/docs/how_to/multimodal_inputs/
- ColPali: Efficient Document Retrieval with Vision Language Models — https://arxiv.org/abs/2407.01449
- Liu et al., *Lost in the Middle: How Language Models Use Long Contexts* — https://arxiv.org/abs/2307.03172
- Anthropic Contextual Retrieval（-49%/-67% 数据出处）— https://www.anthropic.com/news/contextual-retrieval
- 本库既有：[[agent/RAG/多模态RAG|多模态 RAG 笔记]] · [[rag/03_智能分块/文档分块策略|文档分块策略]]

## 相关页面

- 上一篇：[[rag/14_范式与前沿/GraphRAG与知识图谱|GraphRAG 与知识图谱]]
- 回到入口：[[rag/RAG_学习路径导航|RAG 学习路径导航]]
