# 知识库操作日志

本日志按时间顺序记录知识库的所有操作。每条记录以统一前缀开头，便于检索。

---

## [2026-07-13] lint | 知识库健康检查与修复
- 执行全面健康检查（Lint），扫描 228 个 wiki 页面
- 修复问题：
  - 修复 60+ 个 markdown 表格中 wiki 链接的 `\|` 转义问题（改为 `|`）

## [2026-07-13] supplement | 批量重写 langchain_retrievers 模块（7 页）
- 全部从 "API 说明书" 升级为 "实战指南"，每页补充：决策指南、成本分析、故障排查
- VectorStoreRetriever: 三种 search_type 详解、MMR 调参范围、阈值设置策略、元数据过滤语法
- MultiQueryRetriever: 自定义 prompt、include_original/num_queries、成本量化（token+检索次数）、vs Ensemble 选择
- ContextualCompressionRetriever: 四种压缩器详解（含 EmbeddingsFilter 代码）、Pipeline 链式压缩、成本对比表
- EnsembleRetriever: RRF 算法公式、权重调参策略表、三路融合、单路 recall 诊断方法
- SelfQueryRetriever: 完整过滤语法（$gte/$in/$and/$or）、AttributeInfo 详解、VectorStore 兼容性矩阵
- BM25Retriever: BM25 vs 向量对比表、什么时候该用/不该用、标准 Ensemble 搭配
- 自定义Retriever: 三个实战示例（关键词/外部API/数据库）、异步实现约定

## [2026-07-13] supplement | 补充 langchain_embeddings 模块
- 新增 `自定义Embedding实现.md` — 用第三方 SDK 封装 LangChain Embeddings 的通用指南（两种模式、智谱示例）
- 新增 `智谱AI_Embedding.md` — 智谱 embedding-3 完整集成：
  - 方案A：langchain_community 内置（最快上手，基于 zai-sdk v0.2.3）
  - 方案B：zai-sdk 自定义封装（推荐，最新 SDK）
  - 含模型参数表、维度选择指南、VectorStore 集成示例
- 更新 `概述.md` 子页面链接，更新 `index.md` 条目
- 资料时效：基于智谱官方文档 2026.07 最新版（zai-sdk 0.2.3, embedding-3）

## [2026-07-13] supplement | 重写 RunnablePassthrough 页面
- 基于官方 API 文档（langchain-core v1.4.9）重写
- 补充内容：构造时传入 callable 的副作用模式、`.assign()` 三种参数类型、`.pick()`、5 个实战模式、vs RunnableLambda 对比、4 个常见陷阱
  - 修复 `python/io/输入输出与文件` 断链（文件已拆分为 `输入输出` + `文件操作`）
  - 修复 `Welcome.md` 和 `LangChain_学习笔记.md` 中的模板链接（`[[SCHEMA]]` → `[SCHEMA](SCHEMA.md)`）
  - 消除 3 个孤立页面（为 `python/basics/数学函数` 添加导航链接）
  - 补充 `index.md` 缺失条目：`langgraph/tutorial/测试模式`、`langgraph/tutorial/生产部署清单`
- 检查结果：
  - 断链：13 个（多数为 raw 来源引用或 lint 脚本多行匹配假阳性）
  - 孤立页面：0 个
  - frontmatter 问题：0 个
  - 未索引页面：0 个

## [2026-06-20] init | 知识库初始化
- 创建目录结构：`raw/`、`wiki/`
- 创建 schema 文件：`SCHEMA.md`
- 创建索引文件：`index.md`
- 创建日志文件：`log.md`

## [2026-06-20] init | 基于 LLM Wiki 方法论重建知识库
- 深度阅读 `LLM wiki.md`，提取核心方法论
- 将 `LLM wiki.md` 复制到 `raw/` 作为第一个来源
- 用中文重写 `SCHEMA.md`，使之成为完整的 LLM 操作指南（含三层架构、四种页面类型、三种工作流）
- 创建来源摘要页：`wiki/LLM_Wiki_方法论.md`
- 创建概念页：`wiki/知识库三层架构.md`
- 用中文重写 `wiki/Welcome.md` 作为导航着陆页
- 更新 `index.md` 为中文索引

## [2026-06-20] ingest | 来源：LLM wiki.md — 概念页
- 来源：`raw/LLM wiki.md`
- 创建概念页：
  - `wiki/综合.md` — 综合 (Synthesis)：跨来源深度整合能力
  - `wiki/概览.md` — 概览 (Overview)：知识库导航性页面设计
  - `wiki/RAG_vs_Wiki.md` — RAG vs Wiki 模式的详细对比
- 更新 `index.md`、`log.md`

## [2026-06-20] ingest | 来源：LLM wiki.md — 实体页（工具与产品）
- 来源：`raw/LLM wiki.md`
- 创建实体页：
  - `wiki/RAG.md` — 检索增强生成技术详解
  - `wiki/Obsidian.md` — 知识管理应用详解
  - `wiki/NotebookLM.md` — Google AI 笔记工具
  - `wiki/ChatGPT_File_Uploads.md` — OpenAI 文件上传功能
  - `wiki/Memex.md` — Bush 的个人知识管理概念先驱

## [2026-06-20] ingest | 来源：LLM wiki.md — 实体页（Obsidian 功能/插件）
- 来源：`raw/LLM wiki.md`
- 创建实体页：
  - `wiki/Graph_View.md` — Obsidian 图谱视图
  - `wiki/Obsidian_Web_Clipper.md` — 网页剪裁浏览器扩展
  - `wiki/Dataview.md` — 结构化数据查询插件
  - `wiki/Marp.md` — Markdown 演示文稿

## [2026-06-20] ingest | 来源：LLM wiki.md — 实体页（搜索/版本控制）
- 来源：`raw/LLM wiki.md`
- 创建实体页：
  - `wiki/qmd.md` — 本地 Markdown 搜索引擎
  - `wiki/Git.md` — 分布式版本控制系统

## [2026-06-20] update | 更新已有页面的交叉引用
- 更新 `wiki/LLM_Wiki_方法论.md`：添加"涉及实体"和"涉及概念"完整列表
- 更新 `wiki/知识库三层架构.md`：添加相关实体和概念引用
- 更新 `wiki/Welcome.md`：扩展导航入口，分类列出所有页面

## [2026-06-20] update | 更新索引和日志
- 重写 `index.md`：完整列出所有 17 个 wiki 页面
- 重写 `log.md`：记录全部操作历史

---

## [2026-06-23] ingest | 来源：如何开发一个Agent，来手把手教你
- 来源：`raw/Clippings/如何开发一个Agent，来手把手教你.md`
- 创建来源摘要页：`wiki/agent/开发Agent实操指南.md`
- 创建概念页：
  - `wiki/agent/Agent_三层架构.md` — 感知层-思考层-执行层的统一框架
  - `wiki/agent/Agent_Memory.md` — Agent 短期/长期记忆系统设计
  - `wiki/agent/Function_Call.md` — LLM 工具调用核心机制
- 创建实体页：
  - `wiki/agent/LangChain.md` — 最全面的 LLM 应用开发框架
  - `wiki/agent/LlamaIndex.md` — 知识融合与数据连接框架
  - `wiki/agent/AutoGen.md` — 多 Agent 协同框架
- 更新 `index.md`、`log.md`

---

## [2026-06-23] ingest | 来源：菜鸟教程 AI Agent 系列（14篇）
- 来源：`raw/AI Agent教程/` 目录下 14 篇教程
- 创建来源摘要页：`wiki/agent/菜鸟教程_AI_Agent系列.md`
- 创建概念页：
  - `wiki/agent/Agent_六种架构.md` — 六种主流 Agent 架构模式对比
  - `wiki/agent/ReAct.md` — 推理+行动核心范式详解
  - `wiki/agent/上下文工程.md` — 上下文系统化设计与优化
  - `wiki/agent/提示词工程.md` — 提示词构造与精炼技巧
  - `wiki/agent/多模态.md` — 多模态大模型能力解析
- 创建实体页：
  - `wiki/agent/向量数据库.md` — 向量数据库原理、算法与选型
  - `wiki/agent/MCP.md` — 模型上下文协议
- 复用已有页面：[[agent/harness/Agent_三层架构]]、[[agent/memory/Agent_Memory]]、[[agent/harness/Function_Call]]、[[agent/RAG/RAG]]、[[agent/orchestration/LangChain]]、[[agent/RAG/LlamaIndex]]、[[agent/orchestration/AutoGen]]、[[agent/orchestration/开发Agent实操指南]]
- 更新 `index.md`、`log.md`

---

## [2026-07-07] update | 创建 LangChain 模块详解目录
- 创建 `wiki/langchain/modules/` 子目录
- 创建导航页 `wiki/langchain/modules/LangChain模块详解.md`：16 个核心模块的地图
- 创建 16 个模块详解页（每个含"是什么/有什么用/关键 API/完整示例"）：
  - 01_langchain_core · 02_langchain_agents · 03_langchain_tools
  - 04_langchain_models · 05_langchain_messages · 06_langchain_prompts
  - 07_langchain_output_parsers · 08_langchain_runnables
  - 09_langchain_documents · 10_langchain_embeddings
  - 11_langchain_vectorstores · 12_langchain_retrievers
  - 13_langchain_callbacks · 14_langchain_stores
  - 15_langchain_checkpoints · 16_langchain_mcp_adapters
- 更新 `index.md` 添加新专题
- 更新入口页 `wiki/langchain/LangChain_学习笔记.md` 添加模块详解链接

---

## [2026-07-07] update | 重组 LangChain 知识库章节结构
- 创建总览页：`wiki/langchain/LangChain使用手册.md` — 按 12 章节组织的查询入口
- 内容覆盖：安装、框架总览、快速入门、模型、工具、记忆、运行时、Agent、流式、结构化输出、中间件、多 Agent
- 每个章节含功能说明、关键参数、使用场景
- 附加：API 速查表（创建/记忆/中间件/工具运行时/响应处理）+ 学习路径分级
- 更新入口页 `wiki/langchain/LangChain_学习笔记.md`：指向新的章节化使用手册
- 更新 `index.md`：添加新条目

---

## [2026-07-07] ingest | 来源：综合生成 | 智能体核心技术教程
- 模式：Workflow E 快速草稿（团队调度降级）
- 创建来源摘要页：`wiki/langchain/tutorial/智能体核心技术教程/智能体核心技术教程.md`
- 内容覆盖 4 大模块：智能体创建、上下文管理、记忆管理、代码执行
- 附加：Skills 实现 + MCP 集成 + 关系图谱 + 跨章一致性矩阵
- 建立与已有 Skills/MCP/Agent 三层架构等概念的交叉引用
- 更新 `index.md` 添加新专题链接

---

## [2026-07-07] ingest | 来源：LangChain 官方文档 — 核心文档中文化
- 来源：https://docs.langchain.com/oss/python/ 官方文档
- 抓取 12 篇核心页面：概述、安装、快速入门、Agent、工具、模型、短期记忆、运行时、流式传输、多 Agent、结构化输出、中间件
- 翻译为中文并创建对应页面，保留全部代码示例和原文档结构：
  - `wiki/langchain/overview/LangChain概述.md`
  - `wiki/langchain/install/安装LangChain.md`
  - `wiki/langchain/quickstart/快速入门.md`
  - `wiki/langchain/agents/Agent代理.md`
  - `wiki/langchain/tools/工具.md`
  - `wiki/langchain/models/模型.md`
  - `wiki/langchain/memory/短期记忆.md`
  - `wiki/langchain/runtime/运行时上下文.md`
  - `wiki/langchain/streaming/流式传输.md`
  - `wiki/langchain/multi-agent/多Agent系统.md`
  - `wiki/langchain/structured-output/结构化输出.md`
  - `wiki/langchain/middleware/中间件概述.md`
- 更新 `wiki/langchain/LangChain_学习笔记.md`：添加完整文档索引与学习路径

---

## [2026-07-07] update | 创建 LangChain 学习专区
- 创建目录结构：`wiki/langchain/` 及其子目录 `concepts/`、`tutorials/`、`cheatsheets/`、`examples/`
- 创建导航页：`wiki/langchain/LangChain_学习笔记.md`
- 更新 `index.md`：添加“学习专题”分类并链接到 [[langchain/LangChain_学习笔记]]

---

## [2026-07-01] ingest | 来源：Skills 教程 | 菜鸟教程
- 来源：`raw/AI Agent教程/Skills 教程  菜鸟教程.md`
- 创建概念页：`wiki/agent/Skills.md` — AI Agent 可复用能力模块，按需加载的知识复用机制
- 更新实体页：`wiki/agent/MCP.md` — 补充 Skills 对比与交叉引用
- 更新 `index.md`、`log.md`

---

## [2026-07-01] update | 重组 agent/ 目录结构
- 创建子目录：harness/、mermory/、prompt/、Planning & Reasoning/、Grounding & Perception/、Orchestration & Execution/、RAG/、mcp/、skills/
- 移动 11 个文件到对应子目录：
  - harness/ → Agent_三层架构、Agent_六种架构、Function_Call
  - mermory/ → Agent_Memory
  - prompt/ → 上下文工程、提示词工程
  - Planning & Reasoning/ → ReAct
  - Grounding & Perception/ → 多模态
  - Orchestration & Execution/ → LangChain、AutoGen、开发Agent实操指南
  - RAG/ → LlamaIndex（RAG、RAG_vs_Wiki、向量数据库已由用户手动移入）
- 留空目录：Guardrails & Safety/、Observability & Evaluation/
- 批量更新所有 wiki 文件中的 wikilink 路径
- 更新 `index.md`、`log.md`

---

## [2026-07-01] ingest | 来源：Agent Harness 工程实践 - 阿里云开发者
- 来源：ima AI Agent 知识库
- 创建来源摘要页：`wiki/agent/harness/Agent_Harness_工程实践_阿里云开发者.md`
- 创建概念页：
  - `wiki/agent/harness/Agent_Harness_Engineering.md`
  - `wiki/agent/harness/四条反直觉铁律.md`
  - `wiki/agent/harness/六大工程模式.md`
- 更新 `index.md`、`log.md`

---

## [2026-07-01] ingest | 来源：Agentic RL 正在重新定义 AI 的下限
- 来源：ima AI Agent 知识库
- 创建来源摘要页：`wiki/agent/reinforcement_learning/Agentic_RL_正在重新定义AI的下限.md`
- 创建概念页：
  - `wiki/agent/reinforcement_learning/Agentic_RL_算法对比.md` — PPO vs GRPO 详细对比
  - `wiki/agent/reinforcement_learning/Agentic_RL_工程实践.md` — Agentic RL 八大工程原则
- 更新 `index.md`、`log.md`

---

## [2026-07-07] restructure | 知识库全面重组与链接修复
- 创建新的目录结构：`wiki/meta/`、`wiki/tools/`、`wiki/agent/sources/`
- 修正 agent/ 目录命名：
  - `mermory/` → `memory/`（修正拼写）
  - `Planning & Reasoning/` → `planning/`（去除空格和 &）
  - `Grounding & Perception/` → `perception/`（去除空格和 &）
  - `Orchestration & Execution/` → `orchestration/`（去除空格和 &）
  - `reinforcement_learning/` → `reinforcement/`（简化名称）
  - `api请求/` → `api/`（简化名称）
- 重建缺失的核心元文件：
  - `wiki/meta/LLM_Wiki_方法论.md` — 从 raw/LLM wiki.md 提取核心方法论
  - `wiki/meta/知识库三层架构.md` — 三层架构概念
  - `wiki/meta/综合.md` — 跨来源深度整合能力
  - `wiki/meta/概览.md` — 导航性页面设计
  - `wiki/meta/Welcome.md` — 知识库总入口导航
  - `wiki/meta/RAG_vs_Wiki.md` — 从 agent/RAG/ 移至 meta/（元概念对比）
- 重建缺失的工具实体文件：
  - `wiki/tools/Obsidian.md`、`NotebookLM.md`、`ChatGPT_File_Uploads.md`
  - `wiki/tools/Obsidian_Web_Clipper.md`、`Dataview.md`、`Marp.md`
  - `wiki/tools/qmd.md`、`Memex.md`、`Graph_View.md`
  - `wiki/tools/Git.md`（从 wiki/ 根目录移入）
  - `wiki/tools/Gradio.md`、`Streamlit.md`、`Chroma.md`（新创建）
- 移动文件：
  - `wiki/Git.md` → `wiki/tools/Git.md`
  - `wiki/agent/菜鸟教程_AI_Agent系列.md` → `wiki/agent/sources/`
- 创建缺失来源占位文件：
  - `raw/Clippings/如何开发一个Agent，来手把手教你.md`
  - `raw/Clippings/晚风.md`
- 修正 raw/ 来源引用路径，指向正确的子目录位置
- 删除根目录空文件 `Graph View.md`
- 移动 `Token (词元)/` 到 `raw/AI Agent教程/`
- 批量更新所有 wiki 文件中的 wikilink（84 个唯一链接全部修复）
- 更新 `index.md`：添加新工具条目，修正路径，更新日期
- 更新 `log.md`：追加本次重组记录
- 最终验证：所有内部链接无悬挂，0 个缺失引用

---

## [2026-07-07] ingest | 回答归档：Tavily 搜索工具与 System Prompt 编写指南
- 来源：用户查询驱动的知识归档（Tavily + System Prompt）
- 创建概念页：
  - `wiki/langchain/tools/Tavily搜索工具.md` — Tavily 搜索工具集成、参数详解、场景配置速查表与使用技巧
  - `wiki/langchain/agents/系统提示词.md` — System Prompt 三种写法（字符串/SystemMessage/@dynamic_prompt）、推荐结构与编写原则
- 更新 `index.md`：在"概念"分类下添加两个 LangChain 专题条目
- 更新 `log.md`：追加本次操作记录

---

## [2026-07-07] ingest | 归档：工具来源与集成方式
- 创建 `wiki/langchain/tools/工具来源与集成.md`：三种来源（pip包/MCP/@tool手写）对比、pip vs MCP 逐项对比与选型建议、Skills vs Tools 区别
- 更新 `index.md`：添加 LangChain 专题概念条目
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] ingest | 归档：工具注册与调度
- 创建 `wiki/langchain/tools/工具注册与调度.md`：四种工具管理方案对比（直接传参、中间件动态筛选、MCP 协议注册发现、自建 ToolRegistry 管理器），含完整代码示例和方案对比表
- 更新 `index.md`：添加 LangChain 专题概念条目
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] update | 补充数据库存储原理与表结构
- `wiki/langchain/memory/短期记忆.md`：新增"数据库存储原理"章节（内存 vs 数据库 OOM 对比、PostgresSaver 表结构 SQL、PostgresStore 表结构 SQL、常见误解澄清）
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] update | 补充 OOM 风险分析与预防
- `wiki/langchain/memory/短期记忆.md`：新增"OOM 风险分析与预防"章节（内存消耗来源排序、三种高发场景、危险信号速查表、四项预防措施）
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] update | 补充 Checkpointer 内存分析与 Python 线程内存模型
- `wiki/langchain/memory/短期记忆.md`：新增"Checkpointer 内存占用分析"（InMemorySaver 存储结构、单线程/多线程内存估算表、后端对比）和"Python 进程内存模型"（线程共享内存模型、实际限制因素、线程 vs 进程对比）
- 更新 `index.md`：在 LangChain 专题概念中添加 [[langchain/memory/短期记忆]] 条目
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] update | 补充自定义搜索工具三种方案
- `wiki/langchain/tools/Tavily搜索工具.md`：新增"自定义搜索工具"章节（方案 A 标题+URL / 方案 B 分级内容 / 方案 C 结构化返回 + token 消耗对比表 + 适用场景判断）
- 更新 `index.md`：更新 Tavily 条目摘要
- 更新 `log.md`：追加本次操作记录

## [2026-07-07] update | 补充 Tavily 调用方式与 Token 优化策略
- 重写 `wiki/langchain/tools/Tavily搜索工具.md`：
  - 新增 5 种调用方式：新版 `langchain-tavily.TavilySearch`、旧版 `TavilySearchResults`、原生 `TavilyClient` SDK、REST API 直接调用、Agent 动态参数调用
  - 新增 10 项 Token 优化策略：缩减 max_results / 关闭 raw_content / 使用 basic 深度 / 关闭 answer / topic 限定 / time_range / 域名限定 / 缓存 / 提示词控制 / 分层搜索
  - 新增方式对比表和推荐配置模板
- 更新 `index.md`：更新 Tavily 条目摘要
- 更新 `log.md`：追加本次操作记录

---

## [2026-07-08] update | 补充 Checkpointer 工作流程与性能开销
- `wiki/langchain/memory/短期记忆.md`：新增"Checkpointer 工作流程与性能开销"章节（get/put 完整流程图、LLM 不参与数据库检索的说明、存储/网络/内存资源开销表、数据库与 LLM 耗时对比、伪慢原因排查表）
- 更新 `log.md`：追加本次操作记录

## [2026-07-08] restructure | 拆分短期记忆.md 为三份独立文件
- `wiki/langchain/memory/短期记忆.md`：精简为核心概念篇（概述、用法、自定义状态、策略、工具访问）→ ~170 行
- 新建 `wiki/langchain/memory/Checkpointer存储后端.md`：后端对比、表结构 SQL、常见误解
- 新建 `wiki/langchain/memory/记忆系统性能.md`：内存估算、Python 内存模型、OOM 预防、工作流程与开销
- 更新 `index.md`：拆为三个独立条目
- 更新 `log.md`：追加本次操作记录
- 链接验证：0 新增缺失

---

## [2026-07-08] ingest | 归档：上下文压缩
- 新建 `wiki/langchain/memory/上下文压缩.md`：两种压缩场景（对话历史压缩的 SummarizationMiddleware 和 @before_model 硬裁剪、检索结果压缩的 ContextualCompressionRetriever 及其压缩器选型）
- 更新 `index.md`：添加条目
- 更新 `log.md`：追加本次操作记录

---

## [2026-07-08] update | 补充示例代码的 Checkpointer 检索流程讲解
- `wiki/langchain/memory/示例代码.md`：重写为带详细注释的讲解版，逐段标注 get/put 内部 SQL、框架调用时机、SQLite 表结构、调用时序
- 更新 `log.md`：追加本次操作记录

---

## [2026-07-08] ingest | 归档：Agent 循环实现原理
- 新建 `wiki/langchain/agents/Agent循环实现.md`：四层架构详解（LangGraph 图编译 / ReAct 循环伪代码 / 中间件 6 个钩子位置 / Checkpointer 持久化时机），含完整流程图和示例
- 更新 `index.md`：添加条目
- 更新 `log.md`：追加本次操作记录

---

## [2026-07-08] lint | 知识库健康检查与修复

### 检查结果
- 73 个 wiki 页面，19 个 raw 来源
- 发现 30 个页面未收录 index.md
- 发现 8 个断链（集中在 harness 页面的英文命名链接）
- 发现 2 个页面完全缺失 frontmatter（agent/api/ 目录）
- 发现 11 个页面 frontmatter 字段不完整
- 发现 18 个孤立页面（零入站链接）

### 修复项
1. **修复断链**：`Agent_Harness_Engineering.md` 和 `Agent_Harness_工程实践_阿里云开发者.md` 中 8 个英文链接映射到正确中文页面
2. **补全索引**：index.md 新增 30 个页面条目（含 LangChain 模块 16 个、API 页面 2 个、LangChain 专题 12 个）
3. **修复 frontmatter**：`api请求.md` 和 `deepseek.md` 添加完整 frontmatter 和 wiki 链接
4. **统一元数据**：11 个页面补全 type/created/updated 字段，tags 统一为内联格式
5. **消除孤立**：`deepseek.md` 和 `api请求.md` 添加相关页面互链

---

## [2026-07-08] update | 知识库链接网络完善与页面充实

### 导航枢纽建设
- `wiki/langchain/modules/` 下 16 个子模块页面全部添加 `← 返回模块总览` 反向链接，与 `LangChain模块详解.md` 形成双向导航

### 孤立页面消除
- `langchain/agents/Agent代理.md` → 新增指向 `Agent循环实现` 的链接
- `langchain/memory/短期记忆.md` → 新增指向 `上下文压缩` 和 `示例代码` 的链接
- `langchain/tools/工具.md` → 新增指向 `工具来源与集成`、`工具注册与调度`、`Tavily搜索工具` 的链接

### 偏薄页面充实
- `tools/Gradio.md`：从 30 行扩充至 ~90 行，新增最简代码示例、ChatInterface Agent 示例、与 Streamlit 对比表、局限性分析、相关页面链接
- `tools/ChatGPT_File_Uploads.md`：从 32 行扩充至 ~80 行，新增 Advanced Data Analysis 说明、典型使用场景、局限性、与 Wiki 模式对比表、与 NotebookLM 对比表、相关页面链接

---

## [2026-07-08] refactor | 消除知识库内容冗余与高耦合

### 问题诊断
- `Agent_Harness_Engineering.md` 与 `Agent_Harness_工程实践_阿里云开发者.md` 约 80% 内容重复（三代范式、四条铁律、六大模式、行业标杆、未来趋势各出现两次）
- 同时 `四条反直觉铁律.md` 和 `六大工程模式.md` 作为独立详情页存在，导致同一内容在知识库中出现三份
- LangChain 实体页与学习专题页面缺少双向导航

### 解决方案：Hub-and-Spoke 模式
- **`Agent_Harness_Engineering.md`** → 重构为概念导航页（Hub），保留核心定义和三代范式表，铁律/模式/案例均改为摘要+链接指向子页面
- **`Agent_Harness_工程实践_阿里云开发者.md`** → 重构为来源摘要页，聚焦独有的悟空招聘实战案例和六句心法，重复理论内容改为链接引用
- **`四条反直觉铁律.md`** 和 **`六大工程模式.md`** → 保持为权威详情页（Spoke），内容不变
- **`LangChain.md`（实体页）** → 添加学习入口链接至 `LangChain概述`、`LangChain使用手册`、`LangChain模块详解`
- **`Agent_Memory.md`** → 添加指向 LangChain 记忆实现页面的跨层链接

### 结果
- `Agent_Harness_Engineering.md`：~110 行 → ~75 行（精简 32%）
- `Agent_Harness_工程实践_阿里云开发者.md`：~140 行 → ~75 行（精简 46%）
- 消除重复内容约 120 行，页面间职责清晰分离
- 工具系列（4 页）和记忆系列（6 页）经检查耦合度合理，无需额外调整
- 同步更新 `index.md` 中对应条目的描述

---

## [2026-07-08] ingest | RAG 技术体系录入（4 个新页面）

### 新建页面

1. **`wiki/agent/RAG/RAG核心概念与演进.md`**（概念页）
   - RAG 定义与解决的三大问题（幻觉、时效性、私域知识）
   - 三代演进：Naive → Advanced → Modular → Agentic/GraphRAG
   - Mermaid 架构图：索引→检索→生成三阶段数据流
   - 演进对比表（检索方式、流水线结构、灵活性、成熟度）

2. **`wiki/agent/RAG/RAG关键模块解析.md`**（概念页）
   - 六大模块逐个深度解析：文档加载/文本分块/嵌入模型/向量库/检索策略/生成增强
   - 每个模块配参数对照表（默认值、调优范围、影响、建议）
   - 分块器对比（Character/Recursive/Semantic）
   - 嵌入模型选型表（OpenAI/BGE-M3/M3E/GTE）
   - 向量库方案对比（FAISS/Chroma/Milvus/Qdrant/Pinecone）
   - 索引类型权衡（Flat/IVF/HNSW）
   - 检索策略对比（Similarity/MMR/Multi-Query/ParentDocument/Rerank）
   - 引用溯源与无检索结果兜底策略

3. **`wiki/agent/RAG/RAG_LangChain完整实现.md`**（来源/代码页）
   - 6 模块拆分的完整可运行 Python 代码
   - 模块①环境配置（OpenAI/BGE-M3/M3E 三种 Embedding 方案）
   - 模块②文档加载与切分（DirectoryLoader + RecursiveCharacterTextSplitter）
   - 模块③向量库构建（Chroma 持久化 + FAISS 内存）
   - 模块④检索器配置（基础 Similarity vs MMR+Rerank 双策略对比）
   - 模块⑤链式组装（create_history_aware_retriever + create_retrieval_chain）
   - 模块⑥高级技巧（MultiQueryRetriever、ParentDocumentRetriever、对比评测、多轮对话）
   - 每行关键代码有中文注释
   - 策略对比表（速度/多样性/精度/成本/适用场景）

4. **`wiki/agent/RAG/RAG参数调优手册.md`**（参考手册页）
   - 全链路参数总表（参数名/模块/默认值/范围/影响/建议）
   - 按模块分类详解（分块/嵌入/向量库/检索/重排/生成）
   - 常见问题排查表（症状/原因/调优方向）
   - 调优优先级建议

### 交叉引用更新
- `RAG.md`（实体页）→ 新增 4 个详细页面的链接
- `index.md` → 新增 4 个页面条目
- 新页面与 LangChain 模块详解（documents/embeddings/vectorstores/retrievers）建立双向链接

---

## [2026-07-08] update | 补充提示词工程 Zero-shot & Few-shot 详解

- `wiki/agent/prompt/提示词工程.md`：将原"Few-shot 示例"单行描述扩展为完整的"零样本与少样本"独立章节，包含：
  - Zero-shot 和 Few-shot 的定义与代码示例
  - 选型对比表（适用场景/Token消耗/输出可控性/调试难度/典型示例数）
  - 4 条最佳实践（覆盖边界、分隔符、拆分多步、位置偏差）
- 从"进阶概念"表格中移除 Zero-shot 和 Few-shot 两行（已展开为独立章节，避免重复）

---

## [2026-07-08] update | 补充模型提供商接入方式（专用客户端 vs OpenAI 兼容）

- `wiki/langchain/models/模型.md`：新增"模型提供商接入方式"章节
  - 两种接入方式对比：专用客户端（ChatDeepSeek 等）vs OpenAI 兼容客户端（ChatOpenAI + base_url）
  - 代码示例：DeepSeek 专用客户端 vs OpenAI 兼容模式（含 extra_body 逃生舱用法）
  - 核心差异表（特有功能/特殊消息类型/类型安全/维护时效/切换厂商/依赖数量）
  - 选型建议表（5 种场景对应推荐方式）
  - 新增指向 `langchain/modules/04_langchain_models` 的链接

---

## [2026-07-08] ingest | Python 基础知识体系录入（8 个新页面）

### 新建页面

1. **`wiki/python/Python_基础概览.md`**（导航页）
   - 知识导航目录、核心特性速览表、内置类型总览表（10 种类型的可变性/有序性）

2. **`wiki/python/数值类型.md`**（概念页）
   - int/float/bool/complex 详解
   - int 内置方法（bit_length/bit_count/to_bytes/from_bytes）
   - float 内置方法（is_integer/hex/fromhex）+ math 模块常用函数
   - 浮点数精度陷阱与 decimal 精确计算
   - bool 作为 int 子类的特性
   - 运算符优先级表

3. **`wiki/python/字符串.md`**（概念页）
   - 4 种字符串格式化方式（f-string/format/%/拼接）
   - 索引与切片详解
   - 全部 40+ 内置方法分类速查：查找替换、大小写转换、对齐填充、分割拼接、判断类、清理编码
   - 常用内置函数（len/reversed/sorted/chr/ord）

4. **`wiki/python/列表.md`**（概念页）
   - 创建方式（字面量/构造器/推导式）
   - 全部内置方法（append/insert/extend/pop/remove/clear/index/count/sort/reverse/copy）
   - 排序进阶（key 函数、降序、按字典 key 排序）
   - 列表推导式（基础/带条件/嵌套）
   - 栈和队列用法（list vs deque）
   - 浅拷贝 vs 深拷贝

5. **`wiki/python/字典.md`**（概念页）
   - 创建方式（字面量/构造器/推导式/fromkeys）
   - 全部内置方法（keys/values/items/get/setdefault/update/pop/popitem/clear/copy/fromkeys）
   - 遍历方式（键/值/键值对/排序遍历/推导式）
   - 嵌套字典与安全访问
   - defaultdict 和 OrderedDict 用法

6. **`wiki/python/元组与集合.md`**（概念页）
   - tuple：创建、操作、命名元组（namedtuple）、与 list 对比
   - set：创建、增删方法、集合运算（并/交/差/对称差/子集/超集）
   - frozenset：不可变集合
   - 实用场景（去重/快速成员检测/集合运算）

7. **`wiki/python/输入输出与文件.md`**（概念页）
   - print 高级用法（sep/end/file/f-string 格式化）
   - input 与类型转换
   - 文件读写（open/with/模式表/文件对象方法）
   - pathlib 路径操作
   - 数据序列化：JSON/CSV/pickle

8. **`wiki/python/函数.md`**（概念页）
   - 函数定义与 docstring
   - 参数类型（位置/关键字/默认值/*args/**kwargs/仅位置/仅关键字）
   - lambda 表达式
   - 闭包（closure）
   - 装饰器（基础/带参数/functools.wraps）
   - 类型提示（Type Hints）
   - 生成器函数（yield）

9. **`wiki/python/类与面向对象.md`**（概念页）
   - 类定义（类变量/实例变量/构造方法）
   - 实例方法/类方法/静态方法
   - 继承（单继承/多重继承/MRO/super()）
   - property 属性装饰器
   - dataclass 数据类
   - 常用魔术方法表（17 个 dunder methods）
   - 上下文管理器（类实现/contextmanager 装饰器）
   - 抽象基类（ABC）

### 索引更新
- `index.md` 新增 "Python 基础" 板块，含 8 个页面条目

---

## [2026-07-08] ingest | Python 进阶内容录入（6 个新页面）

### 目录重组
- 创建 `wiki/python/advanced/` 子目录
- 将现有文件按分类移入子目录：`basics/`、`data_structures/`、`functions/`、`io/`、`oop/`
- 更新 `Python_基础概览.md` 和 `index.md` 中所有路径

### 新增进阶页面（聚焦 Agent 开发）

1. **`wiki/python/advanced/异步编程.md`**（概念页）
   - asyncio 协程基础（async def/await）
   - 并发执行（gather/create_task/wait）
   - 异步迭代器与异步生成器
   - 异步队列（asyncio.Queue）
   - 超时与取消、异常处理
   - LangChain Agent 异步模式（ainvoke/astream/astream_events）
   - 最佳实践（信号量、asyncio.to_thread）

2. **`wiki/python/advanced/Pydantic数据验证.md`**（概念页）
   - BaseModel 基础用法
   - Field 高级配置（gt/le/pattern/description）
   - 自定义验证器（field_validator/model_validator）
   - 序列化与反序列化（model_dump/model_validate）
   - LangChain 结构化输出（with_structured_output）
   - Tool 参数定义（args_schema）
   - 模型继承、泛型模型、配置类

3. **`wiki/python/advanced/装饰器进阶.md`**（概念页）
   - 装饰器本质与 functools.wraps
   - 带参装饰器、类装饰器
   - LangChain 回调系统（BaseCallbackHandler + 10 个回调方法）
   - LangChain 中间件（wrap_model_call）
   - 工具注册装饰器
   - functools 常用工具（lru_cache/cached_property/partial）

4. **`wiki/python/advanced/生成器与异步迭代器.md`**（概念页）
   - 生成器基础（yield/send/yield from）
   - 生成器 vs 列表对比
   - LangChain 同步/异步流式输出（stream/astream）
   - Agent 流式事件处理（astream_events）
   - 异步迭代器与异步生成器定义
   - 数据管道模式

5. **`wiki/python/advanced/类型提示进阶.md`**（概念页）
   - Union/Optional/Literal/Annotated
   - TypeVar 泛型、Generic、Callable
   - TypeAlias 类型别名
   - LangChain Agent 状态类型定义（TypedDict + Annotated）
   - 工具参数类型（Pydantic + Literal）
   - 回调类型签名
   - 类型检查工具（mypy/pyright）

6. **`wiki/python/advanced/序列化与状态管理.md`**（概念页）
   - JSON 序列化（自定义编码器/解码器）
   - Pickle 序列化与安全警告
   - LangChain 消息序列化（messages_to_dict/messages_from_dict）
   - LangGraph Checkpoint 持久化（SqliteSaver/PostgresSaver/MemorySaver）
   - 状态管理模式（不可变状态/状态快照）
   - 序列化格式对比表

### 索引更新
- `index.md` 新增 "进阶主题（Agent 开发相关）" 子板块
- `Python_基础概览.md` 新增进阶主题导航

---

## [2026-07-08] refactor | 拆分输入输出与文件为两个独立页面

### 操作
- 删除 `wiki/python/io/输入输出与文件.md`（原合并页面）
- 新建 `wiki/python/io/输入输出.md` — 聚焦 print/input 与格式化
- 新建 `wiki/python/io/文件操作.md` — 聚焦文件读写、pathlib、序列化

### 输入输出.md 新增内容
- print() 全部参数详解（sep/end/file/flush）
- f-string 格式化完整参考：
  - 数字格式化（小数位/千分位/百分比/进制转换）
  - 字符串格式化（对齐/填充/截断/repr/str/ascii）
  - 日期时间格式化（%Y-%m-%d 等完整格式符）
  - 对齐表格输出示例
  - 动态宽度和精度
- 三种格式化方式对比（f-string vs format vs %）
- input() 高级用法（多行输入/密码输入/安全转换）

### 文件操作.md 内容
- 文件读写（open/with/模式表/文件对象方法）
- 二进制文件操作
- pathlib 完整参考（属性/拼接/判断/目录/快捷方法/转换/文件操作）
- JSON/CSV/pickle 序列化
- 实用模式（原子写入/文件监控）

### 索引更新
- `index.md` 和 `Python_基础概览.md` 路径同步更新

---

## [2026-07-08] ingest | RAG 内容补充（4 个新页面）

### 背景
审查现有 RAG 内容后发现以下缺陷：
- 缺少 RAG 评估与优化内容（如何衡量 RAG 质量）
- 分块策略内容较基础，缺少高级技术
- 混合检索和重排序仅简单提及，未深入讲解
- 缺少生产部署相关内容（缓存、监控、成本、安全）

### 新增页面

1. **`wiki/agent/RAG/RAG评估与优化.md`**（概念页）
   - 评估维度：检索质量（Context Precision/Recall/Relevancy）+ 生成质量（Faithfulness/Answer Relevancy/Correctness）
   - RAGAS 评估框架：安装、基础用法、指标详解
   - 评估数据集构建：人工标注、LLM 生成、生产日志提取
   - 优化策略：检索优化、生成优化、端到端优化
   - 持续监控：关键指标、告警阈值

2. **`wiki/agent/RAG/RAG分块策略进阶.md`**（概念页）
   - 基础策略回顾：固定大小、递归字符、语义分块
   - 高级策略详解：
     - Parent-Child 分块（小块检索 + 大块上下文）
     - 句子窗口分块（LlamaIndex SentenceWindowNodeParser）
     - Markdown 感知分块（按标题层级）
     - 代码感知分块（按函数/类边界）
     - 表格感知分块（保持表格完整）
   - 分块策略选择指南（按文档类型推荐）
   - 参数调优：chunk_size、chunk_overlap、中文特殊考虑
   - 分块质量评估：指标、常见问题诊断

3. **`wiki/agent/RAG/RAG混合检索与重排序.md`**（概念页）
   - 检索方式对比：向量检索 vs BM25 vs 混合检索
   - 混合检索实现：
     - Ensemble Retriever（LangChain）
     - RRF（Reciprocal Rank Fusion）分数融合算法
     - 加权线性组合
   - 重排序（Reranking）：
     - Cross-Encoder 原理与实现
     - LangChain 重排序实现
     - 主流重排序模型对比（BGE/Cohere/Jina）
   - 完整混合检索 + 重排序流程
   - 性能优化：缓存策略、异步重排序
   - 效果对比表

4. **`wiki/agent/RAG/RAG生产部署.md`**（概念页）
   - 生产架构设计：典型三层架构图
   - 缓存策略：查询缓存、嵌入缓存、Redis 分布式缓存
   - 监控与告警：关键指标、Prometheus 集成、告警阈值
   - 成本控制：Token 成本优化、模型选择策略（ModelRouter）
   - 扩展性设计：水平扩展（FastAPI + workers）、异步处理
   - 安全性：输入验证、输出过滤、访问控制（API Key + 限流）
   - 部署检查清单：上线前检查、运维手册

### 索引更新
- `index.md` 新增 4 个 RAG 页面条目

---

## [2026-07-08] update | 列表页面添加 zip() 函数详解

### 操作
- 在 `wiki/python/data_structures/列表.md` 的"常用操作"和"列表作为栈和队列"之间添加 `zip()` 函数专节

### 新增内容
- `zip()` 函数定义：将多个可迭代对象按位置打包成元组
- 基础用法示例
- 常见场景：
  - 并行遍历多个序列
  - 创建字典（`dict(zip(keys, values))`）
  - 矩阵转置（`zip(*matrix)`）
  - 解包/unzip（`zip(*pairs)`）
- 长度不一致时的行为：默认以最短序列为准
- Python 3.10+ 的 `strict=True` 参数：强制长度一致，否则报错

### 说明
- `zip()` 之前在字典.md 和控制流.md 中仅有简单示例，未做详细讲解
- 作为 Python 内置函数，`zip()` 常与列表配合使用，放在列表页面最合适

---

## [2026-07-08] ingest | 新建数学函数页面

### 操作
- 创建 `wiki/python/basics/数学函数.md`
- 更新 `index.md` 在基础语法部分添加该页面

### 内容概要
- **math 模块**：标准库数学函数
  - `math.sqrt()`：平方根（~15-17 位精度）
  - 幂运算、对数、三角函数、取整函数
- **numpy 数学函数**：向量化操作
  - `np.sqrt()`：数组级平方根运算
  - 特殊值处理（负数→nan、无穷大、NaN）
  - 性能对比：NumPy 比 Python 循环快约 50 倍
- **精度对比表**：math.sqrt / np.sqrt / Decimal.sqrt / mpmath.sqrt
- **高精度计算**：Decimal 和 mpmath 的使用示例
- **选择建议**：根据场景选择合适的库

### 说明
- 用户询问 numpy 的 sqrt 函数精度，发现知识库中缺少相关内容
- 决定创建独立的数学函数页面，同时覆盖 math 和 numpy
- 放在 basics 目录，作为 Python 基础知识的一部分

---

## [2026-07-08] refactor | 拆分元组与集合为独立页面

### 背景
用户指出数据结构部分内容组织不合理：
- 一个文件应该对应一个内容（元组和集合不应混在一起）
- 一个文件夹应该对应一个主题（data_structures 文件夹只放数据结构）
- 原有内容不够详细

### 操作
- 删除 `wiki/python/data_structures/元组与集合.md`
- 创建 `wiki/python/data_structures/元组.md`（独立详细页面）
- 创建 `wiki/python/data_structures/集合.md`（独立详细页面）
- 更新 `index.md` 中的链接

### 元组.md 新增内容
- **为什么需要元组**：数据保护、字典键、集合元素、函数返回、性能优化、语义表达
- **创建方式**：字面量、省略括号、从可迭代对象构造、单元素元组陷阱
- **元组推导式**：说明 Python 没有元组推导式，用 tuple() 包装生成器
- **访问元素**：索引、切片、解包（基础解包、星号解包、嵌套解包）
- **内置方法**：count() 和 index() 详解
- **运算操作**：拼接、重复、成员检测、比较、排序、反转
- **不可变性**：元组本身不可变、元素可以是可变的、哈希值
- **命名元组**：基础用法、高级特性（_asdict、_replace、_fields、_make）、带默认值、应用场景
- **tuple vs list**：详细对比表、性能对比
- **常见用法**：函数多值返回、字典键、数据交换、常量集合

### 集合.md 新增内容
- **为什么需要集合**：去重、成员检测、集合运算、删除重复项、关系测试
- **创建方式**：字面量、空集合陷阱、从可迭代对象构造、集合推导式、注意事项（元素必须可哈希）
- **内置方法**：add、update、remove vs discard、pop、clear
- **集合运算**：并集、交集、差集、对称差集、子集、超集、不相交、原地运算
- **运算对比表**：操作符、方法、原地更新、说明
- **实用场景**：
  - 列表去重（基础去重、保持顺序、辅助集合）
  - 快速成员检测（性能对比：set 比 list 快 50000 倍）
  - 找差异（独有元素、共同元素）
  - 标签系统（Jaccard 相似度计算）
  - 图算法中的已访问集合
- **frozenset**：特性、支持的操作、应用场景（字典键、集合元素、保护数据）
- **set vs frozenset vs list**：详细对比表
- **性能对比**：内存占用、成员检测、添加元素
- **常见陷阱**：空集合创建、集合无序性、迭代时修改、不可哈希元素

### 索引更新
- `index.md` 数据结构部分更新为两个独立条目

---

## [2026-07-08] ingest | 新建 LangChain 核心方法对比页面

### 背景
用户询问 embed_query、embed_documents、invoke 和 stream 的区别，发现知识库中缺少系统性的方法对比说明。

### 操作
- 创建 `wiki/langchain/basics/LangChain核心方法对比.md`
- 更新 `index.md` 在 LangChain 专题概念部分添加该页面

### 内容概要
- **方法分类**：
  - 嵌入方法（Embeddings 类）：embed_query、embed_documents
  - 执行方法（Runnable 接口）：invoke、stream、ainvoke、astream
- **embed_query vs embed_documents**：
  - embed_query：嵌入单个查询，用于用户输入向量化
  - embed_documents：批量嵌入文档，用于构建向量数据库
  - 为什么要分开：某些模型对查询和文档使用不同前缀
- **invoke vs stream**：
  - invoke：同步执行，等待完整结果
  - stream：流式执行，逐步返回生成内容
  - 对比示例：展示两种方式的输出差异
- **在 Chain 中的使用**：RAG Chain 的 invoke/stream 示例
- **方法选择指南**：按场景推荐合适的方法
- **常见错误**：
  - 混淆嵌入方法和执行方法
  - 在流式输出中访问完整内容
  - 忘记 flush 导致输出缓冲

### 说明
- 这些方法属于不同层面，不能直接对比
- embed_query/embed_documents 是 Embeddings 类的方法
- invoke/stream 是 Runnable 接口的方法
- 创建独立页面帮助用户理解 LangChain API 设计


---

## [2026-07-08] update | 大幅扩充 LangChain 提示词模板内容

### 背景
用户询问 LangChain 提示词模板的详细使用方法，发现现有内容不够深入。

### 操作
- 重写 `wiki/langchain/modules/06_langchain_prompts.md`
- 从 173 行扩充到 600+ 行

### 新增内容
- **核心概念**：包结构、变量语法详解
- **关键类详解**：
  - PromptTemplate：基础用法、format vs format_prompt、与 LLM 链式调用
  - ChatPromptTemplate：基础用法、消息类型、带历史消息、部分变量预填充、多轮对话模板
  - SystemMessagePromptTemplate：专门用于系统消息
  - FewShotChatMessagePromptTemplate：few-shot 学习、动态示例选择
  - MessagesPlaceholder：插入消息列表、可选的历史消息
- **实际应用场景**：
  - RAG 问答
  - 多轮对话 Agent
  - 结构化输出
  - 翻译助手
  - 代码生成
- **高级技巧**：
  - 模板组合
  - 条件变量
  - 动态系统提示
  - 模板验证
- **常见陷阱**：
  - 变量名不匹配
  - 忘记 MessagesPlaceholder
  - 部分变量覆盖
  - 模板只在调用时填充
- **最佳实践**：5 条实用建议
- **性能考虑**：模板创建 vs 格式化、缓存模板

### 说明
- 内容覆盖从基础到高级的完整知识体系
- 包含大量可运行的代码示例
- 强调实际应用场景和最佳实践
- 帮助用户避免常见错误

## [2026-07-12] ingest | 新建 LangChain 短期记忆实战页面
- 来源：`raw/AI Agent教程/Short-term memory.md`（LangChain 官方文档）
- 创建概念页：`wiki/agent/memory/短期记忆.md`
- 内容涵盖：Checkpointer 用法、自定义 Agent 状态、四种上下文管理策略（裁剪/删除/摘要/自定义）、三种记忆访问方式（工具/提示词/中间件）
- 全文翻译为中文，代码添加中文注释

## [2026-07-15] supplement | 修正生成器与 yield 描述
- 文件：`wiki/python/advanced/生成器与异步迭代器.md`
- 修正内容：
  - "使用 `yield` 关键字返回值" → "使用 `yield` 关键字**暂停函数执行并产出值**"（更准确）
  - 新增"yield 执行时序"小节：展示 `gen()` 不执行函数体、`next()` 才推进的完整流程
  - 新增"yield vs return"对比表
  - 新增"为什么需要 yield"小节：用 100 万元素的例子说明内存差异
  - 补充"send() 时序"小节：详细拆解 `value = yield total` 的执行步骤
  - 补充"yield from 等价关系"：明确等价于 `for x in item: yield x`
- 更新日期：2026-07-15

## [2026-07-15] supplement | 扩充中间件概述页到完整 18 个
- 文件：`wiki/langchain/middleware/中间件概述.md`
- 基于官方文档（langchain v1.0+）补充完整中间件清单，从原本 7 个扩充到 18 个
- 按 7 大场景分类：
  - 上下文管理（3 个）：Summarization、Context editing、Filesystem
  - 成本与调用控制（2 个）：ModelCallLimit、ToolCallLimit
  - 可靠性（3 个）：ModelFallback、ToolRetry、ModelRetry
  - 安全合规（2 个）：HumanInTheLoop、PII
  - 任务规划（3 个）：TodoList、SubAgent、Rubric
  - 工具优化（3 个）：LLMToolSelector、ProviderToolSearch、LLMToolEmulator
  - 扩展能力（2 个）：ShellTool、FilesystemFileSearch
- 补充：6 个钩子分类速查表、生产环境最小推荐配置组合
- 更新 `index.md`、`log.md`

## [2026-07-15] ingest | 整理 LangChain 中间件系统（18 个内置 + 4 个核心指南）
- **触发原因**：用户要求系统全面整理 LangChain 中间件知识，每个中间件作为独立文档，并按 4 个主题章节组织
- **资料来源**：官方文档（langchain v1.0+）
  - `https://docs.langchain.com/oss/python/langchain/middleware/built-in`
  - `https://docs.langchain.com/oss/python/langchain/middleware/custom`
- **创建文档**（共 22 个新页面）：

  *4 个核心指南文档*
  - `wiki/langchain/middleware/中间件概述.md`（已重写）
  - `wiki/langchain/middleware/钩子机制详解.md`
  - `wiki/langchain/middleware/自定义中间件开发.md`
  - `wiki/langchain/middleware/最佳实践.md`
  - `wiki/langchain/middleware/与LangGraph关系.md`

  *18 个内置中间件独立文档*
  - 上下文管理：SummarizationMiddleware、ContextEditingMiddleware、FilesystemMiddleware
  - 成本控制：ModelCallLimitMiddleware、ToolCallLimitMiddleware
  - 可靠性：ModelFallbackMiddleware、ToolRetryMiddleware、ModelRetryMiddleware
  - 安全合规：HumanInTheLoopMiddleware、PIIMiddleware
  - 任务规划：TodoListMiddleware、SubAgentMiddleware、RubricMiddleware
  - 工具优化：LLMToolSelectorMiddleware、ProviderToolSearchMiddleware、LLMToolEmulator
  - 扩展能力：ShellToolMiddleware、FilesystemFileSearchMiddleware

- **每个中间件文档包含**：定义/使用场景/参数详解/使用方法/注意事项/相关页面
- **核心指南覆盖**：执行时序图、状态更新机制、5 个实战示例、6 种反模式、生产配置模板
- **更新**：`index.md`（添加所有新页面）、`log.md`
- **基于知识库协议**：搜索→核时效（langchain v1.0+ 官方文档）→补充→给代码

## [2026-07-22] ingest | 初始化 FastAPI 知识库专题
- 触发原因：用户开始学习 FastAPI，要求系统化整理并与 LangChain/Agent 知识结合
- 资料来源：FastAPI 官方文档 v0.115+（fastapi.tiangolo.com）、社区部署教程
- 创建文档（共 13 个新页面）：
  * 核心概念：FastAPI概述、快速入门、路径操作、请求体与Pydantic、参数验证
  * 进阶机制：依赖注入、中间件与CORS、错误处理与响应模型、安全认证、异步处理
  * 生产部署：流式响应、部署、与LangChain集成
- 设计原则：每个文档均关联 LangChain/Agent 内容，体现"FastAPI 是 Agent 的 HTTP 外壳"
- 更新：index.md（新增 FastAPI 专题章节）、log.md

## [2026-07-27] supplement | 补充正则表达式知识库 + 增强 lambda
- 创建：`wiki/python/basics/正则表达式.md`
  - 涵盖：re 模块核心函数、元字符速查表、逐模式拆解 5 个常见场景（邮箱/手机号/中文/URL/IP）
  - 进阶：分组与捕获、命名分组、sub 函数式替换、贪婪 vs 非贪婪、编译优化
  - 实战：PIIMiddleware 自定义检测器、3 个常见错误
- 修改：`wiki/python/functions/函数.md`
  - 修正 lambda 部分 typo（`[4, 5][[`）
  - 新增 LangChain 实战示例（RunnablePassthrough.assign）
- 更新：log.md

## [2026-07-27] supplement | 补充 join 和 f-string 详解到字符串知识库
- 文件：`wiki/python/basics/字符串.md`
- 扩充 f-string 节：从 4 行示例扩展为完整教程
  - 逐部分解释 `{name}` 的求值过程
  - 新增格式控制（千分位、百分比、对齐、十六进制、日期）
  - 补充 f-string vs 其他三种方式的对比说明
- 新增 join 详解节：从表格一行扩展为独立教程
  - 核心规则："只在相邻元素间插入分隔符"
  - 与 `+` 的性能对比
  - 配合生成器表达式的实战示例
- 修正：`sorted("hello")` 行末多余的 `[[` typo
- 更新：log.md

## [2026-07-28] supplement | 补充 Skills 渐进式加载的底层消息流转
- 文件：`wiki/agent/skills/Skills实现与集成.md`
- 在"渐进式加载详解"中新增"底层消息流转"和"tool docstring 设计原则"两节

## [2026-07-28] ingest | 新建 Docker 沙箱专题
- 创建目录：`wiki/docker/`（新专题）
- 创建文档：
  - `wiki/docker/Docker沙箱与Agent集成.md` — 总览：架构图、三种方案对比、安全配置速查
  - `wiki/docker/DockerSandboxManager.md` — 完整实现：SandboxConfig/SandboxResult/DockerSandbox 类、@tool 封装、Agent 组装、自定义镜像、目录结构
- 资料来源：LangChain 官方文档（ShellToolMiddleware + Deep Agents Sandbox Backend）、docker-py 官方 API
- 核心价值：Agent 与 Docker 沙箱的三种集成方案（内置中间件 / 自定义 tool / 托管平台），每种含完整可运行代码
- 更新日期：2026-07-28

## [2026-07-28] supplement | Docker 沙箱专题深度生产级重构
- 触发原因：用户要求深度思考、细化到生产级别，满足知识库文件组织规范
- 操作：从 2 个文件拆解为 8 个专题页面，删除旧文件
- 新文件结构：
  - `wiki/docker/Docker沙箱概述.md` — 架构总览：三层方案对比、选择决策树、生产关注点地图
  - `wiki/docker/Docker_Sandbox核心类.md` — 核心类逐层拆解：配置类/结果类/管理器类/@tool封装/Agent组装
  - `wiki/docker/沙箱安全加固.md` — 安全五维度：文件系统隔离/网络隔离/资源限制/内核安全/镜像安全/密钥管理/审计检查清单
  - `wiki/docker/沙箱性能优化.md` — 性能四维度：镜像优化/预热池/exec_run vs containers.run/并发控制/延迟目标
  - `wiki/docker/沙箱监控与日志.md` — structlog JSON 日志/Prometheus metrics/Agent层追踪/trace_id全链路
  - `wiki/docker/沙箱错误处理与容错.md` — 异常分类/指数退避重试/优雅降级/双重超时/Supervisor守护
  - `wiki/docker/沙箱生产部署清单.md` — 8 维度 checklist（镜像/资源/安全/监控/容错/Skills/依赖/CI/CD）
  - `wiki/docker/沙箱与Agent集成实践.md` — 完整项目：config.py + manager.py + tools.py + audit中间件 + FastAPI 入口
- 删除旧文件：Docker沙箱与Agent集成.md、DockerSandboxManager.md
- 遵循 SCHEMA.md：type=concept、中文文件名（可读）与英文文件名（API类）、frontmatter 含 tags/created/updated/type

## [2026-07-28] supplement | Docker 沙箱补充 3 个生产级文档
- 触发原因：用户要求内容足够全面指导 Agent 开发
- 新增文档：
  - `wiki/docker/沙箱常见使用案例.md` — 3 个端到端场景（数据分析/代码审查/代码生成），每个含 skill 文件 + Agent 配置 + 执行流程 + 配置对比表
  - `wiki/docker/沙箱调试与故障排查.md` — 8 类常见故障的诊断流程和修复方案（容器启动失败/超时/OOM/代码异常/网络/挂载/daemon/调试技巧）+ 速查表
  - `wiki/docker/沙箱多租户隔离.md` — 4 维度隔离（网络/资源/文件/日志）、TenantConfig/TenantManager/MultiTenantSandbox 完整实现、4 级配额策略、审计追溯、隔离验证
- 更新：log.md

## [2026-07-29] ingest | 初始化 DeepAgents 知识库专题
- 触发原因：用户开始学习 DeepAgents，要求搜索官方文档并系统化整理
- 资料来源：docs.langchain.com/oss/python/deepagents/、reference.langchain.com/python/deepagents、pypi.org/project/deepagents
- 创建目录：`wiki/deepagents/`（新专题）
- 创建文档（共 9 个新页面）：
  - DeepAgents概述.md — 架构定位、与 create_agent 关系、9 大核心能力
  - 快速入门.md — 最小 Agent、自定义工具、流式、权限、HITL、沙箱
  - 中间件栈.md — 13 层默认栈详解、合并规则、覆盖示例
  - 内置工具.md — write_todos/task/文件系统工具/execute
  - 文件系统后端.md — 6 种 Backend 选型指南
  - 子代理系统.md — 通用/自定义/编译/异步子代理
  - 上下文管理.md — 4 层管理 + Prompt Caching
  - 自定义与扩展.md — HarnessProfile、权限、消息优化
  - 与LangChain_Agent对比.md — create_agent vs create_deep_agent + 迁移指南
- 设计原则：与现有 LangChain/Agent/Middleware/Docker 知识库深度交叉引用，体现代码复用关系

## [2026-07-29] restructure | DeepAgents 重组为子目录 + 新增生产级内容
- 操作：9个平铺文件重组为 9 子目录，新增 6 个深度文档
- 新增：production/源码架构.md、生产部署.md、skills/Skills系统.md、middleware/FilesystemMiddleware详解.md、backend/Backend实战配置.md
- 最终：14 个文件分属 9 子目录

## [2026-07-29] supplement | DeepAgents 细化补充 7 个深度文档
- 工具层：文件系统工具详解.md（6 个工具完整参数）、Task子代理工具详解.md（调用格式/执行流程/描述质量/限制）
- 子代理层：子代理配置实战.md（生产级 prompt 设计/多角色协同/CallSphere 迁��数据）
- 上下文层：自动摘要机制.md（卸载 20k/摘要 85%/Prompt Caching 省 40%实测数据）、长期记忆与AGENTS.md（格式/自动更新/memory vs skills）
- 定制层：HarnessProfile深度用法.md（6 字段/注册匹配/按模型隐藏工具）、State扩展与DeltaChannel.md（O(N) vs O(N²)/自定义字段读写）
- 最终：21 个文件分属 9 子目录

## [2026-07-31] supplement | 补充 Pydantic 组件全景图与四层用法速查
- 文件：`wiki/python/advanced/Pydantic数据验证.md`
- 新增"组件全景图（Pydantic v2）"：BaseModel/Field/RootModel/验证器/类型系统/序列化/配置/泛型/错误/扩展包完整组件树
- 新增组件详解：RootModel（非对象 JSON）、SecretStr（密钥脱敏）、TypeAdapter（免继承验证）、pydantic-settings（BaseSettings 配置管理）
- 新增"Agent 开发中的四层用法速查"：Tool 参数 / 结构化输出 / State 定义 / 配置管理
- 修复 typo：`Annotated[int, Field(gt=0)][[` → `Annotated[int, Field(gt=0)]`
- 更新日期：2026-07-31

## [2026-07-31] supplement | 补充 pydantic-settings 详解（BaseSettings + SettingsConfigDict）
- 文件：`wiki/python/advanced/Pydantic数据验证.md`
- 扩充 pydantic-settings 小节为完整详解：
  - BaseSettings vs BaseModel 对比表（取值来源/典型用途/继承关系）
  - SettingsConfigDict 专属键（env_file/env_prefix/env_nested_delimiter/secrets_dir）
  - 加载优先级：显式参数 > 环境变量 > .env > 默认值
  - 实例化数据流向四步图（读 .env → 环境变量 → 默认值 → 类型校验）
  - Agent 项目典型用法（config.py + .env 完整示例）
- 更新日期：2026-07-31

## [2026-07-31] new | 新建 functools 函数工具独立页面
- 文件：`wiki/python/advanced/functools函数工具.md`
- 触发原因：用户问 `from functools import lru_cache` 的含义，发现知识库仅零散提及无专页
- 内容：
  - 成员一览表：lru_cache/cache/partial/wraps/reduce/cached_property/singledispatch
  - lru_cache 详解：fib 缓存性能对比（2986 万次→36 次）、执行流程数据流向图、maxsize/typed 参数、4 个注意点、Agent 缓存 LLM 实例实战
  - partial 偏函数、wraps 元信息保留、reduce 归约、cached_property 属性缓存
  - 相关页面：装饰器进阶/函数/Pydantic数据验证/fastapi依赖注入
- 更新日期：2026-07-31

## [2026-07-31] supplement | 补充 assert 断言语句小节
- 文件：`wiki/python/basics/异常处理.md`
- 在"抛出异常"之后新增"assert 断言语句"小节：
  - 语法拆解（带/不带提示信息两种写法）
  - 执行流程图（条件为 True 空操作 / False 抛 AssertionError）
  - 与 if+raise 的等价写法
  - 三个典型场景（前置条件/后置条件/不可能分支）
  - ⚠️ 关键坑：`python -O` 优化模式下 assert 被删除不执行，与 raise 对比表
  - 使用注意点（惰性求值误解、不做安全检查）
  - Agent 项目实战（外部输入用 raise，内部配置用 assert）
- 修复 typo：`config["db"]["password"][[` → `config["db"]["password"]`
- 更新日期：2026-07-31

## [2026-07-31] supplement | 补充 Any 任意类型小节
- 文件：`wiki/python/advanced/类型提示进阶.md`
- 在"高级类型"章节（TypeAlias 之后）新增 "Any（任意类型）" 小节：
  - Any 定义：万能通配符，跳过类型检查
  - 三个典型场景：JSON 外部数据 / 通用工具函数 / **kwargs
  - Any vs object vs 不写类型提示 对比表（含 `x + 1` 报错示例）
  - 使用纪律：过度使用 Any 等于放弃类型安全
  - Agent 框架典型位置：Runnable 通用接口、中间件钩子 state/runtime
- 修复 typo：`outputs: Dict[str, Any][[,` → `outputs: Dict[str, Any],`
- 更新日期：2026-07-31

## [2026-07-31] new | 新建 SQLAlchemy 与 SQLModel 数据库引擎页面
- 文件：`wiki/python/advanced/SQLAlchemy与SQLModel数据库引擎.md`
- 触发原因：用户给出 create_db_engine 工厂代码要求解释，知识库无 SQLAlchemy 专页
- 内容（每个参数详解）：
  - 完整代码 + 三件事总览（创建引擎/内存库 StaticPool/事件开外键）
  - 导入详解表：Any/event/Engine/create_engine/StaticPool 各自作用
  - database_url 连接串格式（sqlite 内存/磁盘、postgresql、mysql）
  - 函数参数：database_url / `*` 仅限关键字 / echo 打印 SQL
  - create_engine 参数：connect_args check_same_thread 线程限制、poolclass StaticPool 内存库原理（含 :memory: 连接隔离说明、poolclass 对比表）
  - event.listen：事件名表（connect/checkout/begin...）、回调签名 2 参数、SQLite 外键默认不生效的原因
  - 数据流向总览图 + FastAPI Session 依赖注入调用示例
- 更新日期：2026-07-31

## [2026-07-31] ingest | 创建 SQLModel 学习模块（6 页面）
- 依据：knowledge-base-builder 技能 + SCHEMA.md 规范，用户要系统学习 SQLModel
- 新建 `wiki/sqlmodel/` 模块（每页含 frontmatter/一句话定义/对比表格/交叉引用）：
  - `概述.md` — 模块导航：SQLModel 定位（一个类=SQLAlchemy模型+Pydantic模型）、解决的双类维护问题、学习路径
  - `模型定义.md` — table=True 表模型 vs 纯校验模型、Field 全参数表（列属性+校验双层）、类型映射、create_all、双重身份、常见误区
  - `CRUD操作.md` — 四步流程（add/commit/refresh）数据流向图、first/one/all 对比、变更跟踪、删除、完整示例、误区
  - `查询进阶.md` — where 逻辑运算、操作符表（in_/like/is_/between）、排序/分页/聚合/JOIN 隐式+显式
  - `关系与关联.md` — 一对多（foreign_key+Relationship+back_populates）、多对多 Link 模型、cascade_delete、N+1 解决
  - `引擎与Session.md` — 迁移自 `python/advanced/SQLAlchemy与SQLModel数据库引擎.md` 并扩充 Session 章节（生命周期/方法速查/FastAPI 注入）
- 迁移动作：删除旧文件 `wiki/python/advanced/SQLAlchemy与SQLModel数据库引擎.md`，5 个页面旧链接批量更新为 `[[sqlmodel/引擎与Session|SQLAlchemy]]`
- 资料时效：基于 sqlmodel.tiangolo.com 官方教程 2026 最新版
- 更新日期：2026-07-31

## [2026-07-31] new | 新建 pytest 与 fixture 页面
- 文件：`wiki/python/testing/pytest与fixture.md`（新建 testing 目录，便于后续扩展 pytest 系列）
- 触发原因：用户问 `@pytest.fixture` 注解作用，知识库无 pytest 专页
- 内容：
  - fixture 解决的问题（重复准备/清理痛点）
  - 基本用法 + 执行流程数据流向图（yield 前后 = 准备/清理）
  - scope 生命周期表（function/module/session/class）
  - fixture 依赖链（参数名自动解析）
  - 内置 fixture（tmp_path/capsys/monkeypatch/request）
  - 4 个常见误区 + 与 SQLModel 结合实战
- 交叉引用：sqlmodel/引擎与Session、fastapi/测试、langgraph 测试模式
- 更新日期：2026-07-31

## [2026-07-31] new | 新建 SimpleNamespace 轻量对象页面
- 文件：`wiki/python/advanced/SimpleNamespace轻量对象.md`
- 触发原因：用户问 `from types import SimpleNamespace` 的作用，知识库无相关内容
- 内容：
  - 定位：免定义、属性可随意增删改的轻量对象（object 的升级版）
  - 三种创建方式（含 `SimpleNamespace(**dict)` 字典转对象）
  - 与 dict 对比表（访问语法/非法键/遍历/自动补全）+ 选择规则
  - 4 个典型场景：测试桩、模拟数据库行、配置读取、函数多值打包
  - 与 dataclass/namedtuple 对比表
  - 小细节：repr/vars 互转/相等性
- 更新日期：2026-07-31

## [2026-08-01] new | 新建 create_deep_agent 参数详解页
- 文件：`wiki/deepagents/customization/create_deep_agent参数详解.md`
- 触发原因：用户问 "create_deep_agent 中有哪些参数"
- 资料时效：官方 docs.langchain.com customization 页 + API reference（deepagents>=0.7）
- 内容（18 参数完整清单）：
  - 完整签名 + 参数速查总表（基本配置/四能力/持久化安全/高级定制/辅助 五组）
  - 基本配置：model（字符串 vs 实例）、tools（@tool/函数/MCP）、system_prompt
  - 四能力：subagents、skills、memory（含 skills vs memory 对比表）、backend（6 种后端表）
  - 持久化安全：checkpointer（HITL 必需）、store、permissions、interrupt_on
  - 高级定制：response_format、state_schema、context_schema、middleware
  - 易错点汇总：interrupt_on 必须配 checkpointer、StateBackend 路径 / 前缀、覆盖中间件需完整配置
- 更新日期：2026-08-01

## [2026-08-01] supplement | 补充 AGENTS.md vs system_prompt 对比章节
- 文件：`wiki/deepagents/context/长期记忆与AGENTS.md`
- 触发原因：用户问 "AGENTS.md 存的都是什么东西？和系统提示词有什么区别？"
- 新增"AGENTS.md vs system_prompt"章节（5.1-5.6）：
  - 一句话区别 + 执行流程对比图（代码注入 vs 文件注入）
  - 七维对比表（本质/注入时机/谁定义/可变性/作用范围/更新频率/典型内容）
  - 核心差异：system_prompt 是"死"的，AGENTS.md 是"活"的（Agent 可 write_file 自更新）+ 自进化流程图
  - 生产三层 prompt 体系：system_prompt（人设）/ AGENTS.md（记忆）/ SKILL.md（技能）
  - 生产实践建议表（token 控制/大小/只读/多级文件）
- 修复损坏链接：`[[deepagents/middleware/中���件栈概述]]` → `[[deepagents/middleware/中间件栈概述]]`
- 更新日期：2026-08-01

## [2026-08-04] fix | 修正后端概述页（官方 6 种后端定义）
- 文件：`wiki/deepagents/backend/后端概述.md`
- 修正内容：
  - 按官方文档（deepagents>=0.7）修正"6 种内置后端"定义：State / Filesystem / **LocalShell** / Store / ContextHub / Composite，补上缺失的 `LocalShellBackend`
  - 明确 Sandbox 是**独立的沙箱后端类别**（6 个提供商：LangSmith/Daytona/E2B/Modal/Runloop/Vercel），不再与内置后端并列
  - 新增"官方 2026 版补充要点"：StateBackend 线程内跨轮持久、FilesystemBackend virtual_mode、LocalShellBackend 最高风险、StoreBackend namespace 隔离、ContextHubBackend 云端仓库
  - 新增 LocalShellBackend 专章（含 execute 风险警告、与 Docker 沙箱的关系）
  - 更新选择决策树（按"是否需要执行代码"分流）、修复旧链接（内置工具/中间件栈 → 新子目录路径）
- 资料时效：docs.langchain.com 官方文档 2026-08 最新版
- 更新日期：2026-08-04

## [2026-08-06] fix | DeepAgents 专题全面修复（断链/版本/矛盾/重复/关联）
- 触发原因：用户反馈 DeepAgents 内容粗略、组件关联性不足；诊断发现 5 类问题
- **P0-1 断链修复**：批量修复 23 条断链（8 个文件），旧路径→新路径映射 9 组
  - `deepagents/中间件栈` → `middleware/中间件栈概述`（6 处）等
  - 修复后验证：deepagents 目录内 0 条断链
- **P0-2 代码错误修复**：
  - `快速入门.md`：stream_events 括号不匹配（缺 `}` 闭合）
  - `生产部署.md`：补 `import json`、定义未声明的 `internet_search`/`custom_tool`/`send_email` 工具、移除僵尸 import、修正 `StoreBackend(store=store)` → `StoreBackend()`
- **P1-1 版本统一**：20 个文件 `>=0.5`/`>=0.6` → `>=0.7`（共 29 页全部统一）
  - `源码架构.md` 目录树更新为 v0.6.12 monorepo 结构（libs/deepagents、context_hub.py、local_shell.py、todo_list.py、harness_profiles.py）
- **P1-2 矛盾数据统一**：
  - 与LangChain_Agent对比.md："默认中间件 10 个" → "13 层"
  - 中间件栈概述.md：文件工具 7 个（含 delete）→ 6 个（与内置工具概述一致）
  - 子代理概述.md：`tools=[read_file]` 补充来源说明（FilesystemMiddleware 提供）
- **P2 关联强化 + 去重**：
  - 新建 `overview/组件依赖全景.md`：参数→中间件→工具→backend 完整映射表、read_file 5 消费者链、execute 来源、存储三通道、端到端数据流
  - 概述页导航加入组件依赖全景
  - 消除 Summarization 重复配置：上下文管理概述改为"详见自动摘要机制"，中间件栈概述示例加权威引用
- 资料时效：deepwiki/github 源码核对（v0.6.12+）
- 更新日期：2026-08-06

## [2026-08-21] supplement | 补充 LangGraph 内置 Reducer 完整清单
- 文件：`wiki/langgraph/modules/langgraph_state/Reducer.md`
- 触发原因：用户问"langgraph 中自带的常用的 reducer 函数有哪些"，发现原页面只覆盖 add_messages + 自定义，缺 3 个官方内置机制
- 新增内容：
  - `operator.add`：数值累加 / 列表拼接，vs add_messages 对比表（纯拼接 vs 按 ID 更新）
  - `RemoveMessage` + `REMOVE_ALL_MESSAGES`：按 ID 删除消息（配 add_messages 使用）、清空全部、典型应用
  - `Overwrite`：绕过 reducer 强制覆盖（重置累加字段）
  - 内置 reducer 速查总表（5 项：add_messages/operator.add/RemoveMessage/Overwrite/默认覆盖）
  - 补充 MessagesState 预置状态类说明
- 修复 2 处 typo：`"tags": ["initial"][[,` → `["initial"]`；`result["step_count"][[)` → `result["step_count"])`
- 更新 frontmatter（tags 补充、updated 2026-08-21）
- 资料时效：官方 docs.langchain.com/graph-api + reference 2026-08 核实
- 更新日期：2026-08-21

## [2026-08-22] supplement | 新增 LangGraph 并行执行模型页面（含 mermaid 图）
- 新建：`wiki/langgraph/modules/langgraph_core/并行执行模型.md`
- 触发原因：用户问"节点2/3 是同时执行吗？异步还是先后顺序"，发现知识库只有零散提及（超级步骤定义/Command 并行语法），缺执行细节
- 页面内容：
  - 超级步骤核心概念（Pregel 模型）+ mermaid 图（超级步骤划分 + 屏障时序 + checkpoint 粒度）
  - 同一超级步骤三特性：共享输入快照、屏障同步、输出统一合并
  - 关键问题：invoke() 同步 = 顺序执行（逻辑并行）/ ainvoke() 异步 = 真并发，对比表 + 耗时示例
  - 选型建议（CPU 用 invoke、I/O 用 ainvoke、FastAPI 必须 ainvoke）
  - 与 Checkpointer 关系：快照按超级步骤存（非按节点）
- 关联更新：
  - `Command与状态更新.md` 3.2 并行章节加"详见并行执行模型"引用
  - `错误处理与恢复.md` 4.3 超级步骤章节加引用
- 更新日期：2026-08-22

## [2026-08-22] organize | LangGraph 知识库整体整理（完善 + 示范案例）
- 触发原因：用户要求"整理 langgraph 知识库，首先完善，其次有示范案例（不用很长）"
- **P1 完善**：
  - 修复 63 处 `[[` 残留 typo（38 个文件）——此前批量替换遗留的代码损坏，用正则 lookbehind 定位并修复，验证清零
  - 导航页 `LangGraph模块详解.md` 补全：目录树加并行执行模型.md；子页面链接补错误处理与恢复/Command与状态更新/可视化与调试/并行执行模型；新增 tutorial 分区
  - 并行执行模型.md 补"简短可运行示例"（A→B、C→D 并行分支 + invoke/ainvoke 计时对比）
- **P2 示范案例**：
  - 新建 `tutorial/综合示例_全概念演示.md`：一个文件（订单处理工作流）串起全部核心概念（State/Node/Edge/条件边/Checkpointer/Streaming/会话隔离），无 LLM 依赖可直接运行，附运行结果预览 + 概念映射表
  - 挂入导航页 tutorial 分区（⭐ 标注）
- 验证：断链 0、typo 0，文件总数 40 → 41
- 更新日期：2026-08-22
