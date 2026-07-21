---
tags: [reference, RAG, 参数, 调优, 速查]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 参数调优手册

> RAG pipeline 全链路可调参数速查表。按模块分类，标注默认值、调优范围、对效果的影响和建议。

## 全链路参数总表

| 参数名                    | 所属模块          | 默认值                      | 调优范围                      | 对效果的影响                       | 调优建议                                            |
| ---------------------- | ------------- | ------------------------ | ------------------------- | ---------------------------- | ----------------------------------------------- |
| `chunk_size`           | 文本分块          | 1000                     | 200-2000                  | 小→检索精准但可能丢上下文；大→上下文完整但稀释嵌入质量 | QA 场景 500；摘要场景 1000-2000                        |
| `chunk_overlap`        | 文本分块          | 200                      | chunk_size 的 10%-20%      | 过小→语义在边界断裂；过大→冗余增加存储和检索成本    | 50-200，通常为 chunk_size 的 15%                     |
| `separators`           | 文本分块          | `["\n\n","\n"," ",""]`   | 自定义列表                     | 控制切分位置，影响块的语义完整性             | Markdown 用 `["\n## ","\n### ","\n\n","\n"," "]` |
| `add_start_index`      | 文本分块          | False                    | True/False                | 开启后 metadata 含块起始位置，便于溯源     | 建议开启                                            |
| `encoding_name`        | 文本分块(Token模式) | `cl100k_base`            | `cl100k_base`/`p50k_base` | 不同 tokenizer 对应不同模型          | OpenAI 用 `cl100k_base`；Claude 用 `p50k_base`     |
| `model` (Embedding)    | 嵌入模型          | `text-embedding-3-small` | 见模型选型表                    | 维度和质量直接决定检索精度                | 中文用 BGE-M3；英文用 text-embedding-3-small           |
| `dimensions`           | 嵌入模型          | 模型默认                     | 256-3072                  | 降维节省存储但损失精度                  | 512 是质量与成本的良好平衡点                                |
| `normalize_embeddings` | 嵅入模型          | False                    | True/False                | 归一化后余弦相似度=点积，数值更稳定           | 建议开启                                            |
| `persist_directory`    | 向量库           | None                     | 路径字符串                     | 持久化到磁盘，重启不丢失                 | 生产环境必须设置                                        |
| `collection_name`      | 向量库           | `chroma`                 | 自定义名称                     | 同一库存多组文档时区分                  | 按项目或文档集命名                                       |
| `k`                    | 检索器           | 4                        | 1-20                      | 大→上下文更多但可能引入噪声；小→精准但可能遗漏     | QA 用 3-5；研究用 8-10                               |
| `fetch_k`              | MMR 检索器       | 20                       | 10-50                     | MMR 过滤前的候选数，影响多样性选择空间        | `k` 的 3-5 倍                                     |
| `lambda_mult`          | MMR 检索器       | 0.5                      | 0.0-1.0                   | 0=最大多样性；1=最大相关性              | 避免重复用 0.3；精度优先用 0.7                             |
| `score_threshold`      | 分数阈值检索        | None                     | 0.0-1.0                   | 过滤低相关度结果                     | 0.3-0.5（视模型和归一化而定）                              |
| `top_n`                | 重排序器          | 3                        | 1-10                      | 重排后保留的文档数                    | 通常 3-5，与 `k` 一致                                 |
| `model` (Reranker)     | 重排序器          | `bge-reranker-base`      | 见重排器表                     | Cross-Encoder 的质量决定精排效果      | 中文用 `bge-reranker-large`；英文用 Cohere             |
| `temperature`          | LLM 生成        | 0.7                      | 0.0-1.0                   | RAG 场景需确定性回答                 | **固定为 0**                                       |
| `max_tokens`           | LLM 生成        | 模型默认                     | 256-4096                  | 控制回答长度                       | QA 用 512-1024                                   |

## 按模块分类详解

### 文本分块参数

| 参数 | 分块器 | 默认 | 范围 | 影响 | 建议 |
|------|--------|------|------|------|------|
| `chunk_size` | Recursive / Character | 1000 | 200-2000 | 检索粒度 | QA: 500, 摘要: 1000-2000 |
| `chunk_overlap` | Recursive / Character | 200 | 50-400 | 语义连续性 | chunk_size 的 10%-20% |
| `separators` | Recursive | `["\n\n","\n"," ",""]` | 自定义 | 切分位置 | Markdown 感知: `["\n## ","\n\n","\n"," "][[` |
| `add_start_index` | Recursive | False | True/False | 溯源能力 | 建议开启 |
| `is_separator_regex` | Recursive | False | True/False | 分隔符是否为正则 | 复杂切分规则时用 True |
| `breakpoint_threshold_type` | SemanticChunker | `percentile` | 3种 | 语义断点灵敏度 | 通用: `percentile`, 精准: `standard_deviation` |
| `breakpoint_threshold_amount` | SemanticChunker | 95 | 50-95 | 断点阈值 | 90-95 |
| `min_chunk_size` | SemanticChunker | 无 | 50-300 | 防过小块 | 150 |
| `max_chunk_size` | SemanticChunker | 无 | 500-2000 | 防过大块 | 800 |

### 嵌入模型参数

| 参数 | 默认值 | 范围 | 影响 | 建议 |
|------|--------|------|------|------|
| `model` | `text-embedding-3-small` | 见模型表 | 检索质量根基 | 中文: BGE-M3, 英文: 3-small |
| `dimensions` | 模型默认 | 256-3072 | 存储/精度权衡 | 512 是良好平衡 |
| `normalize_embeddings` | False | True/False | 数值稳定性 | 建议开启 |
| `device` | `cpu` | `cpu`/`cuda` | 推理速度 | 有 GPU 必须用 cuda |
| `encode_kwargs.prompt` | None | 前缀字符串 | E5/BGE 模型必需 | E5: `"passage: "` / `"query: "` |

### 向量库参数

| 参数 | 默认值 | 范围 | 影响 | 建议 |
|------|--------|------|------|------|
| `persist_directory` | None | 路径 | 持久化 | 生产环境必须设置 |
| `collection_name` | `chroma` | 字符串 | 多组文档隔离 | 按项目命名 |
| `search_type` | `similarity` | 3种 | 检索策略 | 通用: similarity, 去重: mmr |
| `filter` | None | dict | 元数据过滤 | 按来源/日期/标签过滤 |

### 检索器参数

| 参数 | 检索类型 | 默认 | 范围 | 影响 | 建议 |
|------|---------|------|------|------|------|
| `k` | 全部 | 4 | 1-20 | 返回数量 | QA: 3-5, 研究: 8-10 |
| `fetch_k` | MMR | 20 | 10-50 | 候选池大小 | k 的 3-5 倍 |
| `lambda_mult` | MMR | 0.5 | 0-1 | 多样性 vs 相关性 | 去重: 0.3, 精度: 0.7 |
| `score_threshold` | 阈值过滤 | None | 0-1 | 过滤低质量 | 0.3-0.5 |
| `top_n` | Reranker | 3 | 1-10 | 重排后保留数 | 与 k 一致 |

### 生成参数

| 参数 | 默认 | 范围 | 影响 | 建议 |
|------|------|------|------|------|
| `temperature` | 0.7 | 0-1 | 确定性 vs 创造性 | **RAG 固定为 0** |
| `max_tokens` | 模型默认 | 256-4096 | 回答长度 | QA: 512-1024 |
| `system_prompt` | 无 | 自定义 | 回答风格和规则 | 明确指示"仅基于上下文" |
| `context_format` | stuff | stuff/map_reduce/refine | 多文档合并方式 | stuff: 简单；refine: 迭代精炼 |

## 常见问题与调优方向

| 症状 | 可能原因 | 调优方向 |
|------|---------|---------|
| 检索结果不相关 | chunk_size 过大或 Embedding 质量差 | 减小 chunk_size；换更强的 Embedding 模型 |
| 检索结果重复 | 缺乏多样性控制 | 切换为 MMR 检索；降低 `lambda_mult` |
| 回答出现幻觉 | Prompt 未限制或检索结果为空 | 强化 system_prompt；增加 `score_threshold` |
| 回答缺少上下文 | chunk 过小或未用父文档检索 | 增大 chunk_size；启用 ParentDocumentRetriever |
| 召回率低 | 单一查询无法命中 | 启用 MultiQueryRetriever；增加 `k` |
| 排序不精确 | 缺少重排序 | 接入 CrossEncoderReranker |
| 多轮对话指代失败 | 未启用历史感知 | 使用 `create_history_aware_retriever` |
| 长文档检索丢失上下文 | 检索粒度与返回粒度一致 | 使用 ParentDocumentRetriever（小块检索+大块返回） |

## 相关页面

- [[agent/RAG/RAG核心概念与演进]] — RAG 定义与三代演进
- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析
- [[agent/RAG/RAG_LangChain完整实现]] — 完整代码实现
- [[agent/RAG/向量数据库]] — 向量数据库实体详解
- [[langchain/modules/LangChain模块详解]] — LangChain 16 个模块总览

## 来源

- LangChain 官方文档：Text Splitters, Retrievers, Vector Stores API
- 2025 RAG 调优实践指南
- BGE-M3 / BGE-Reranker 模型文档
