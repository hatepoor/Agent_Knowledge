---
tags: [meta, index]
updated: 2026-07-08
---

# 知识库索引

本索引按类别组织 wiki 中的所有页面，每页附带链接、一行摘要和可选元数据。每次摄入新来源后更新。

---

## 导航
- [[meta/Welcome]] — 知识库入门导航、快速开始和使用指南

## 学习专题
- [[langchain/overview/LangChain概述]] — LangChain 框架整体概述与定位
- [[langchain/install/安装LangChain]] — LangChain 安装与环境配置指南
- [[langchain/quickstart/快速入门]] — LangChain 快速入门教程与基础用法
- [[langchain/LangChain_学习笔记]] — LangChain 框架系统化学习专区，含概念、教程、速查与示例
- [[langchain/LangChain使用手册]] — 按章节结构组织的查询入口（12 章 + API 速查 + 学习路径）
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]] — 4 大模块完整技术教程（创建/上下文/记忆/代码执行），含代码与逐行解释
- [[langchain/modules/LangChain模块详解]] — 16 个核心模块详解（每个模块用途、关键 API、完整示例）

### LangGraph 专题
- [[langgraph/overview/LangGraph概述]] — LangGraph 框架概述：图计算范式、核心概念、与 LangChain 关系
- [[langgraph/quickstart/快速入门]] — LangGraph 快速入门：安装、StateGraph、节点/边/状态、编译执行
- [[langgraph/modules/LangGraph模块详解]] — 7 个核心模块详解（图引擎/状态/持久化/流式/人机交互/多Agent/子图）
- [[langgraph/tutorial/ReAct_Agent实现]] — 用 LangGraph 从零实现 ReAct Agent（完整代码 + 执行流程解析）
- [[langgraph/tutorial/多轮对话Agent]] — 多轮对话 Agent：checkpointer 记忆、消息裁剪、时间旅行
- [[langgraph/tutorial/RAG_Agent]] — RAG Agent：检索增强生成图、条件路由、向量存储集成
- [[langgraph/tutorial/测试模式]] — LangGraph 测试策略：单元测试、集成测试、Mock LLM
- [[langgraph/tutorial/生产部署清单]] — 生产部署检查清单：持久化、监控、扩展、安全
- [[langgraph/concepts/图计算范式]] — 为什么用图：线性链的局限、循环的必要性、DAG vs 状态机
- [[langgraph/concepts/LangGraph_vs_LangChain]] — LangGraph vs LangChain 定位对比：组件层 vs 编排层

## Python 基础
### 基础语法
- [[python/basics/Python_基础概览]] — Python 基础导航页：内置类型总览、核心特性速览
- [[python/basics/数值类型]] — int/float/bool/complex 及运算符、内置方法、类型转换
- [[python/basics/字符串]] — str 类型、格式化（f-string/format）、全部 40+ 内置方法速查
- [[python/basics/控制流]] — if/elif/else、for/while 循环、推导式、模式匹配
- [[python/basics/异常处理]] — try/except/else/finally、自定义异常、异常链
- [[python/basics/模块与包]] — import 机制、标准库概览、包管理、虚拟环境
- [[python/basics/数学函数]] — math 模块与 numpy 数学函数、sqrt/三角函数/对数、精度对比、向量化操作

### 数据结构
- [[python/data_structures/列表]] — list 类型、切片、全部内置方法、列表推导式、栈/队列用法
- [[python/data_structures/字典]] — dict 类型、全部内置方法、字典推导式、defaultdict/OrderedDict
- [[python/data_structures/元组]] — tuple 类型、不可变序列、命名元组、解包
- [[python/data_structures/集合]] — set/frozenset、集合运算、去重、成员检测

### 输入输出与文件
- [[python/io/输入输出]] — print/input、f-string 格式化（数字/字符串/日期/表格对齐）、三种格式化方式对比
- [[python/io/文件操作]] — 文件读写（open/with/模式表）、pathlib 路径操作、JSON/CSV/pickle 序列化

### 函数
- [[python/functions/函数]] — def、参数类型（*args/**kwargs）、lambda、闭包、装饰器、生成器、类型提示

### 面向对象
- [[python/oop/类与面向对象]] — class 定义、继承、property、dataclass、魔术方法、ABC 抽象基类
- [[python/oop/魔法方法]] — 魔术方法（dunder methods）分类详解：生命周期、字符串、比较、算术、容器、属性访问、上下文管理器

### 进阶主题（Agent 开发相关）
- [[python/advanced/异步编程]] — asyncio 协程、gather/create_task 并发、异步迭代器、Agent 异步调用模式
- [[python/advanced/Pydantic数据验证]] — BaseModel、字段验证、序列化、LangChain 结构化输出与 Tool 定义
- [[python/advanced/装饰器进阶]] — 带参装饰器、类装饰器、LangChain 回调系统、中间件、工具注册
- [[python/advanced/生成器与异步迭代器]] — yield/send、异步生成器、LangChain 流式输出（stream/astream）
- [[python/advanced/类型提示进阶]] — TypeVar、Generic、Annotated、Literal、LangChain Agent 状态类型定义
- [[python/advanced/序列化与状态管理]] — JSON/pickle、消息序列化、LangGraph Checkpoint 持久化、状态快照

## 来源摘要
- [[meta/LLM_Wiki_方法论]] — 用 LLM 构建个人知识库的模式：从 RAG 检索转向持久化 wiki 编译 *(来源: raw/LLM wiki.md)*
- [[agent/orchestration/开发Agent实操指南]] — AI Agent 开发实操入门：三层架构起步，按记忆→工具→规划叠加能力 *(来源: raw/Clippings/如何开发一个Agent，来手把手教你.md)*
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 菜鸟教程 AI Agent 完整教程系列（14篇）：从 LLM 基础到多 Agent 协作的全链路 *(来源: raw/AI Agent教程/)*
- [[agent/harness/Agent_Harness_工程实践_阿里云开发者]] — Agent Harness 工程实践：悟空 AI 招聘 Agent 实战案例与六句心法 *(来源: ima AI Agent 知识库)*
- [[agent/reinforcement/Agentic_RL_正在重新定义AI的下限]] — Agentic RL 正在重新定义 AI 的下限：从单轮静态问答转向多轮动态交互 *(来源: ima AI Agent 知识库)*

## 概念
- [[meta/知识库三层架构]] — 原始来源层、Wiki 层、Schema 层的分离设计与协作原理
- [[meta/综合]] — 跨来源的深度整合能力，wiki 相对于 RAG 的核心优势
- [[meta/概览]] — 知识库的导航性页面设计
- [[meta/RAG_vs_Wiki]] — RAG 检索增强生成 vs Wiki 持久化编译的详细对比
- [[agent/harness/Agent_三层架构]] — AI Agent 的感知层-思考层-执行层统一框架
- [[agent/memory/Agent_Memory]] — Agent 的短期/长期记忆系统设计与向量检索方案
- [[agent/memory/短期记忆]] — LangChain 短期记忆实战：Checkpointer、裁剪/删除/摘要压缩、工具/提示词/中间件三种访问方式
- [[agent/harness/Function_Call]] — LLM 工具调用的核心机制，连接思考层与执行层
- [[agent/harness/Agent_六种架构]] — 单Agent循环、规划执行、多Agent协作、反思修正、RAG+Agent、工作流DAG
- [[agent/planning/ReAct]] — 推理+行动交替的核心 Agent 推理范式
- [[agent/prompt/上下文工程]] — 系统化设计 Agent 上下文信息的工程实践
- [[agent/prompt/提示词工程]] — 构造和精炼提示词以最大化模型性能的技巧
- [[agent/perception/多模态]] — AI 同时处理文本、图像、音频、视频等多种信息的能力
- [[agent/harness/Agent_Harness_Engineering]] — Agent Harness Engineering：概念定义与知识体系导航（第三代范式）
- [[agent/harness/四条反直觉铁律]] — 上下文越少越好、专才赢过通才、状态写文件、约束可执行
- [[agent/harness/六大工程模式]] — 双阶段架构、工具签名即文档、Sub-Agent 隔离、上下游反压、智能体审智能体、熵管理
- [[agent/reinforcement/Agentic_RL_算法对比]] — PPO vs GRPO 详细对比：显存需求、训练稳定性、适用场景
- [[agent/reinforcement/Agentic_RL_工程实践]] — Agentic RL 八大工程原则：模块化设计、训练信号、算法优化、训练策略
- [[agent/api/api请求]] — 大模型 API 请求规范：OpenAI Chat Completions 事实标准、三层结构与五大核心参数
- [[agent/api/deepseek]] — DeepSeek API 调用示例：curl 与 Python SDK 代码片段

### LangChain 专题概念
- [[langchain/basics/LangChain核心方法对比]] — embed_query/embed_documents、invoke/stream、format/invoke 的区别与使用场景
- [[langchain/basics/Agent_invoke_vs_ChatModel_invoke]] — create_agent().invoke() 与 ChatModel.invoke() 返回类型、过程记录、工具执行的完整对比
- [[langchain/agents/Agent代理]] — Agent 核心定义：模型在循环中调用工具直到任务完成，Agent = 模型 + 框架（Harness）
- [[langchain/agents/系统提示词]] — System Prompt 的三种写法（字符串/SystemMessage/动态生成）与编写原则
- [[langchain/agents/Agent循环实现]] — ReAct 循环四层架构（LangGraph 图编译/核心循环伪代码/中间件钩子位置/Checkpointer 持久化点）
- [[langchain/tools/工具]] — 工具概述：扩展 Agent 能力的可调用函数，含创建、注册、调度全流程
- [[langchain/tools/Tavily搜索工具]] — Tavily 五种调用方式、10 项 Token 优化策略与自定义搜索工具三种方案（标题仅/分级内容/结构返回）
- [[langchain/tools/工具注册与调度]] — 四种工具管理方案（tools[]/中间件筛选/MCP协议/自建ToolRegistry类）与按角色分组调度
- [[langchain/tools/工具来源与集成]] — 工具三种来源（pip包/MCP/@tool手写）对比、pip vs MCP 选型、Skills与Tools区别
- [[langchain/models/模型]] — LangChain 的聊天模型抽象：标准接口、流式输出、工具调用、结构化输出
- [[langchain/memory/短期记忆]] — 短期记忆核心概念（Checkpointer 用法、自定义状态、裁剪/摘要策略、工具中访问记忆）
- [[langchain/memory/Checkpointer存储后端]] — 后端对比（InMemory/Postgres/SQLite/AzureCosmos）、数据库表结构 SQL、常见误解
- [[langchain/memory/记忆系统性能]] — 内存估算、Python 线程内存模型、OOM 风险分析与预防、get/put 工作流程与开销
- [[langchain/memory/上下文压缩]] — 对话历史压缩（SummarizationMiddleware/硬裁剪）与检索结果压缩（ContextualCompressionRetriever）两种场景
- [[langchain/memory/示例代码]] — 记忆系统相关示例代码合集
- [[langchain/middleware/中间件概述]] — LangChain 中间件系统：18 个内置中间件、6 个钩子、4 种使用模式
  - [[langchain/middleware/钩子机制详解]] — 6 个钩子的执行时序、Node-style vs Wrap-style、状态更新、Agent Jumps
  - [[langchain/middleware/自定义中间件开发]] — AgentMiddleware 类方式、装饰器方式、5 个实战示例（请求日志/敏感词过滤/调用限制/动态工具/Prompt 缓存）
  - [[langchain/middleware/最佳实践]] — 执行顺序影响、生产环境推荐配置、性能优化、监控告警
  - [[langchain/middleware/与LangGraph关系]] — 中间件是 LangGraph 包装层、嵌入 Agent 作子图、直接在 LangGraph 中使用钩子

  **内置中间件独立文档（18 个）**：

  *上下文管理*
  - [[langchain/middleware/SummarizationMiddleware]] — 对话摘要：接近 token 限制时自动压缩历史
  - [[langchain/middleware/ContextEditingMiddleware]] — 上下文编辑：达到 token 限制时清除旧工具输出
  - [[langchain/middleware/FilesystemMiddleware]] — 文件系统：ls/read/write/edit 工具（deepagents）

  *成本与调用控制*
  - [[langchain/middleware/ModelCallLimitMiddleware]] — 模型调用限制：防无限循环
  - [[langchain/middleware/ToolCallLimitMiddleware]] — 工具调用限制：按工具或全局

  *可靠性*
  - [[langchain/middleware/ModelFallbackMiddleware]] — 模型回退：主模型失败切备用
  - [[langchain/middleware/ToolRetryMiddleware]] — 工具重试：指数退避
  - [[langchain/middleware/ModelRetryMiddleware]] — 模型重试：指数退避

  *安全合规*
  - [[langchain/middleware/HumanInTheLoopMiddleware]] — 人在环：关键操作人工审批
  - [[langchain/middleware/PIIMiddleware]] — PII 检测：邮箱/手机/IP/信用卡脱敏

  *任务规划*
  - [[langchain/middleware/TodoListMiddleware]] — 待办列表：write_todos 工具
  - [[langchain/middleware/SubAgentMiddleware]] — 子代理：派生子 Agent 隔离上下文（deepagents）
  - [[langchain/middleware/RubricMiddleware]] — 评分量规：自评迭代至满足标准（Beta, deepagents）

  *工具优化*
  - [[langchain/middleware/LLMToolSelectorMiddleware]] — LLM 工具选择器：用小模型筛选
  - [[langchain/middleware/ProviderToolSearchMiddleware]] — Provider 工具搜索：服务端按需发现（Anthropic/OpenAI）
  - [[langchain/middleware/LLMToolEmulator]] — LLM 工具模拟器：测试用

  *扩展能力*
  - [[langchain/middleware/ShellToolMiddleware]] — Shell 工具：持久 shell 会话
  - [[langchain/middleware/FilesystemFileSearchMiddleware]] — 文件搜索：glob_search + grep_search
- [[langchain/runtime/运行时上下文]] — 运行时上下文管理：RunnableConfig、回调、可运行对象间的上下文传递
- [[langchain/streaming/流式传输]] — LangChain 流式传输机制：astream_events、token 级流式、自定义流式工具
- [[langchain/structured-output/结构化输出]] — 结构化输出：with_structured_output 方法、Pydantic 模型绑定、工具调用模式
- [[langchain/multi-agent/多Agent系统]] — 多 Agent 系统：网络拓扑、协作模式、LangGraph 多 Agent 编排

### LangChain 模块详解
- [[langchain/modules/langchain_core/概述]] — langchain-core：基础抽象层，定义 Runnable、消息、工具等核心接口
- [[langchain/modules/langchain_agents/概述]] — langchain-agents：Agent 框架，ReAct 循环与工具调用编排
- [[langchain/modules/langchain_tools/概述]] — langchain-tools：工具注册、MCP 集成与工具链管理
- [[langchain/modules/langchain_models/概述]] — langchain-models：聊天模型封装，标准接口与供应商适配
- [[langchain/modules/langchain_messages/概述]] — langchain-messages：消息类型体系（Human/AI/System/Tool）
- [[langchain/modules/langchain_prompts/概述]] — langchain-prompts：提示词模板与变量注入
- [[langchain/modules/langchain_output_parsers/概述]] — langchain-output-parsers：输出解析器，将模型输出转为结构化数据
- [[langchain/modules/langchain_runnables/概述]] — langchain-runnables：Runnable 协议与链式组合（pipe/invoke/batch）
- [[langchain/modules/langchain_documents/概述]] — langchain-documents：文档加载、切分与转换
- [[langchain/modules/langchain_embeddings/概述]] — langchain-embeddings：文本嵌入模型接口与多供应商适配
  - [[langchain/modules/langchain_embeddings/OpenAIEmbeddings|OpenAIEmbeddings]]
  - [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings|HuggingFaceEmbeddings]]
  - [[langchain/modules/langchain_embeddings/智谱AI_Embedding|智谱AI_Embedding]] — embedding-3 集成（zai-sdk v0.2.3 自定义封装）
  - [[langchain/modules/langchain_embeddings/自定义Embedding实现|自定义 Embedding 实现]] — 通用第三方 SDK 封装模式
  - [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings|CacheBackedEmbeddings]]
  - [[langchain/modules/langchain_embeddings/embed_query_vs_embed_documents|embed_query vs embed_documents]]
- [[langchain/modules/langchain_vectorstores/概述]] — langchain-vectorstores：向量存储抽象层与主流数据库集成
- [[langchain/modules/langchain_retrievers/概述]] — langchain-retrievers：检索器抽象、多查询检索与上下文压缩检索
- [[langchain/modules/langchain_callbacks/概述]] — langchain-callbacks：回调系统，事件追踪与可观测性
- [[langchain/modules/langchain_stores/概述]] — langchain-stores：键值存储抽象，用于缓存与状态管理
- [[langchain/modules/langchain_checkpoints/概述]] — langchain-checkpoints：检查点系统，对话持久化与断点续传
- [[langchain/modules/langchain_mcp_adapters/概述]] — langchain-mcp-adapters：MCP 协议适配器，连接外部工具服务

## 实体
### 技术与概念
- [[agent/RAG/RAG]] — 检索增强生成，当前主流的 LLM 知识增强方案
- [[agent/RAG/RAG核心概念与演进]] — RAG 定义、Naive→Advanced→Modular→Agentic 演进脉络与 Mermaid 架构图
- [[agent/RAG/RAG关键模块解析]] — 六大模块深度解析：文档加载/分块/嵌入/向量库/检索/生成，含参数对照表
- [[agent/RAG/RAG_LangChain完整实现]] — 从文档加载到问答 Agent 的完整可运行代码（双检索策略对比 + 多轮对话）
- [[agent/RAG/RAG参数调优手册]] — RAG pipeline 全链路参数速查表与常见问题排查
- [[agent/RAG/RAG评估与优化]] — RAG 评估指标（Faithfulness/Relevancy/Recall）、RAGAS 框架、优化策略
- [[agent/RAG/RAG分块策略进阶]] — 高级分块技术：Parent-Child、句子窗口、Markdown/代码/表格感知分块
- [[agent/RAG/RAG混合检索与重排序]] — BM25+向量混合检索、RRF 分数融合、Cross-Encoder 重排序
- [[agent/RAG/RAG生产部署]] — 缓存策略、监控告警、成本控制、扩展性设计、安全性
- [[tools/Memex]] — Vannevar Bush 1945 年提出的个人知识管理假想设备

### Agent 框架
- [[agent/orchestration/LangChain]] — Python/JS 生态最全的 LLM 应用开发框架
- [[agent/RAG/LlamaIndex]] — 侧重知识融合与数据连接的 RAG/Agent 框架
- [[agent/orchestration/AutoGen]] — 微软开源的多 Agent 协同框架

### Agent 基础设施
- [[agent/RAG/向量数据库]] — 存储和检索高维向量的专用数据库，RAG 和记忆系统的核心
- [[agent/mcp/MCP]] — 模型上下文协议，Agent 连接外部工具的标准化通信协议
- [[agent/skills/Skills]] — AI Agent 可复用能力模块，按需加载的操作说明书（SOP）

### 工具与产品
- [[tools/Obsidian]] — 基于本地 Markdown 的知识管理应用，wiki 的浏览环境
- [[tools/NotebookLM]] — Google 的 AI 笔记和研究工具，RAG 的产品实现
- [[tools/ChatGPT_File_Uploads]] — OpenAI ChatGPT 的文件上传分析功能

### Obsidian 功能与插件
- [[tools/Graph_View]] — Obsidian 的节点-连接图可视化功能
- [[tools/Obsidian_Web_Clipper]] — 浏览器扩展，将网页文章转为 Markdown
- [[tools/Dataview]] — Obsidian 的结构化数据查询插件
- [[tools/Marp]] — Markdown 演示文稿格式及 Obsidian 插件

### 搜索与版本控制
- [[tools/qmd]] — 本地 Markdown 搜索引擎，混合 BM25/向量搜索 + LLM 重排序
- [[tools/Git]] — 分布式版本控制系统，wiki 的版本控制底层

- [[tools/Gradio]] — Python 快速原型库，用于构建机器学习模型的可交互 Web 界面
- [[tools/Streamlit]] — Python 数据应用框架，快速构建数据应用和 ML 工具的前端
- [[tools/Chroma]] — 开源嵌入式向量数据库，轻量级，适合本地开发和原型验证

## 对比分析
- [[meta/RAG_vs_Wiki]] — RAG 与 Wiki 模式的多维度对比，含适用场景分析

## 模板
<!-- 页面模板将在此列出 -->
- 实体页模板（参见 [[SCHEMA]] 中的格式定义）
- 概念页模板
- 来源摘要模板
- 对比分析模板

---

*最后更新：2026-07-08*
