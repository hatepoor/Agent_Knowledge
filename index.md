# 知识库内容索引

> 本索引按领域分类组织所有 wiki 页面。每页列出链接 + 一行摘要。查询时先读此索引找相关页面，再深入阅读。
> 维护规则：每次摄入/新增页面后更新此索引。最后更新：2026-10-23

---

## meta — 核心方法论与导航

- [[meta/Welcome]] — 知识库总入口与全局导航
- [[meta/LLM_Wiki_方法论]] — 持久化 wiki 增量构建方法论
- [[meta/知识库三层架构]] — 原始来源 / Wiki / Schema 分层设计
- [[meta/概览]] — 导航性页面设计原则
- [[meta/综合]] — 跨来源深度整合能力
- [[meta/RAG_vs_Wiki]] — RAG 检索 vs Wiki 编译模式对比

---

## agent — AI Agent 知识体系

### patterns/ — Agent 模式（集中化目录）
- [[agent/patterns/Agent模式总览]] — 单智能体/多智能体模式总入口与选型决策树
- [[agent/patterns/CoT]] — 思维链：分步推理再作答
- [[agent/patterns/ReAct]] — 边想边做：Thought→Action→Observation 闭环
- [[agent/patterns/PlanAndExecute]] — 先规划完整计划再逐步执行
- [[agent/patterns/Reflection]] — 输出后自我审查与循环改进
- [[agent/patterns/ToT]] — 树状思维：多分支搜索与回溯
- [[agent/patterns/Reflexion]] — 经验写入记忆，重试时教训复用
- [[agent/patterns/MultiAgent协作]] — Supervisor / Network / Hierarchical 多智能体协作

### harness/ — Agent 架构与工程实践
- [[agent/harness/Agent_三层架构]] — 感知→思考→执行框架
- [[agent/harness/Agent_六种架构]] — 六种主流 Agent 架构模式对比
- [[agent/harness/六大工程模式]] — Agent 六大工程模式详解
- [[agent/harness/四条反直觉铁律]] — Agent 工程的反直觉原则
- [[agent/harness/Agent_Harness_Engineering]] — Harness 工程实践方法论
- [[agent/harness/Agent_Harness_工程实践_阿里云开发者]] — 阿里云 Agent Harness 实践
- [[agent/harness/Agent沙箱与安全隔离]] — Agent 沙箱五种隔离层次与配置方案
- [[agent/harness/Function_Call]] — 函数调用/工具调用机制

### memory/ — Agent 记忆
- [[agent/memory/Agent_Memory]] — Agent 记忆系统概述
- [[agent/memory/短期记忆]] — 短期记忆机制与实现

### RAG/ — Agent 视角的 RAG（精简版，系统学习见 rag/ 目录）
- [[agent/RAG/RAG]] — 检索增强生成概念页
- [[agent/RAG/RAG核心概念与演进]] — RAG 三代演进（Naive→Advanced→Modular）
- [[agent/RAG/RAG关键模块解析]] — RAG 核心模块拆解
- [[agent/RAG/RAG分块策略进阶]] — 分块策略进阶
- [[agent/RAG/RAG混合检索与重排序]] — 混合检索与重排序实践
- [[agent/RAG/RAG参数调优手册]] — RAG 参数调优参考
- [[agent/RAG/RAG评估与优化]] — RAG 评估与优化方法
- [[agent/RAG/RAG生产部署]] — RAG 生产部署方案
- [[agent/RAG/RAG_LangChain完整实现]] — LangChain RAG 端到端代码
- [[agent/RAG/向量数据库]] — 向量数据库概念
- [[agent/RAG/多模态RAG]] — 多模态 RAG 方法
- [[agent/RAG/LlamaIndex]] — LlamaIndex 框架介绍

### 其他 agent 子目录
- [[agent/prompt/提示词工程]] — 提示词工程方法与模式
- [[agent/prompt/上下文工程]] — 上下文工程：信息密度与窗口管理
- [[agent/orchestration/LangChain]] — LangChain 框架实体页
- [[agent/orchestration/AutoGen]] — AutoGen 框架介绍
- [[agent/orchestration/开发Agent实操指南]] — Agent 开发实操
- [[agent/perception/多模态]] — 多模态感知
- [[agent/perception/Agent多模态集成]] — Agent 多模态集成方案
- [[agent/skills/Skills]] — Skills 系统概念
- [[agent/skills/Skills实现与集成]] — Skills 实现与集成方法
- [[agent/reinforcement/Agentic_RL_工程实践]] — Agentic RL 工程实践
- [[agent/reinforcement/Agentic_RL_算法对比]] — Agentic RL 算法对比
- [[agent/reinforcement/Agentic_RL_正在重新定义AI的下限]] — Agentic RL 前沿趋势
- [[agent/api/api请求]] — API 请求方法
- [[agent/api/deepseek]] — DeepSeek API 集成
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 菜鸟教程 Agent 系列摘要

### interview/ — 面试题库（9 篇、78 题）
> Agent 通用能力层题库，**不绑定编排框架**；图编排机制见 [[langgraph/interview/README|LangGraph 面试题库]]。

- [[agent/interview/01-Agent基础与能力边界]] — 定义、组成、自主性分级、适用边界、失败模式（9 题）
- [[agent/interview/02-ReAct与推理循环]] — ReAct、CoT 对比、停止条件、死循环治理（9 题）
- [[agent/interview/03-工具调用与MCP]] — Function Calling、schema、并行、MCP 安全（9 题）
- [[agent/interview/04-规划反思与任务分解]] — Plan-and-Execute、Reflection、Reflexion（8 题）
- [[agent/interview/05-记忆与上下文工程]] — 长短记忆、压缩、写入/遗忘、多租户（9 题）
- [[agent/interview/06-Skills与能力加载]] — Skills vs Prompt/MCP、渐进加载（8 题）
- [[agent/interview/07-多智能体协作模型]] — 四种拓扑、角色、通信、A2A（9 题）
- [[agent/interview/08-Agent评估与可观测性]] — 指标、轨迹评估、LLM-as-judge（8 题）
- [[agent/interview/09-安全与生产落地]] — 注入、沙箱、权限、成本、灰度（9 题）

---

## rag — RAG 流程化学习路径（14 步全流程）

> 入口：[[rag/RAG_学习路径导航|RAG 学习路径导航]]

### 00 导览与基础
- [[rag/00_导览与基础/RAG基础与演进全景]] — RAG 基础概念与三代演进全景
- [[rag/00_导览与基础/RAG零基础一问一答]] — 新手问答总览
- [[rag/00_导览与基础/LangChain_RAG端到端实现]] — LangChain RAG 端到端代码实例

### 01 数据接入与解析（9 页）
- [[rag/01_多源数据接入/多源数据接入]] — 接入总览：数据源分类、接入四件事、页面地图
- [[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格]] — 解析总览：文本层 vs 视觉层、8 款解析器选型对比
- [[rag/01_多源数据接入/PDF解析]] — 数字版/扫描版分流、多栏阅读顺序、OCR 选型、表格重建
- [[rag/01_多源数据接入/Word解析]] — DOCX 的 OOXML 结构、docling 元素层机制、图片提取、最小依赖链
- [[rag/01_多源数据接入/HTML与网页解析]] — 正文抽取、动态渲染、结构化元数据、导航噪声剔除
- [[rag/01_多源数据接入/表格提取与处理]] — 整块不切、表头前置、摘要 + 原表双轨
- [[rag/01_多源数据接入/图片提取与描述化]] — 元素层取图、图注识别、上下文组装、视觉模型描述
- [[rag/01_多源数据接入/元数据与溯源锚点]] — 字段清单、打标时机、DOCX 无页码的替代锚点
- [[rag/01_多源数据接入/增量接入与去重]] — 三级 hash、变更检测粒度、幂等与删除处理

### 02 数据清洗与规范化（6 页）
- [[rag/02_数据清洗与规范化/数据清洗与规范化]] — 清洗总览：两条并行线、五类动作、分源模板
- [[rag/02_数据清洗与规范化/文本规范化]] — Unicode NFC、全半角、空白与编码、双文本策略
- [[rag/02_数据清洗与规范化/噪声剔除]] — 页眉页脚/目录/水印的识别与剥离、可审计性
- [[rag/02_数据清洗与规范化/去重策略]] — 精确 / 近重复（MinHash+LSH）/ 语义三层去重
- [[rag/02_数据清洗与规范化/质量过滤]] — 长度、语言、信息密度、乱码率的过滤维度
- [[rag/02_数据清洗与规范化/脱敏与隐私]] — PII 识别、脱敏策略取舍、全链路一致性

### 03 智能分块（10 页）
- [[rag/03_智能分块/文档分块策略]] — 分块总览：三维分类框架与选型决策路径
- [[rag/03_智能分块/固定与递归分块]] — 长度切分与分隔符优先级降级机制
- [[rag/03_智能分块/结构感知分块]] — 两阶段切分、标题链前置、metadata 继承
- [[rag/03_智能分块/语义分块]] — 相似度突变点切分、成本与适用边界
- [[rag/03_智能分块/chunk大小与overlap取值]] — 512 默认值依据、混合索引下的 overlap 约束
- [[rag/03_智能分块/特殊内容分块]] — 代码 AST、表格整块、公式与列表保护
- [[rag/03_智能分块/上下文增强分块]] — Late Chunking 与 Contextual Retrieval
- [[rag/03_智能分块/分块工具选型]] — LangChain / Chonkie / semchunk / LlamaIndex / docling
- [[rag/03_智能分块/层级父子与句子窗口检索]] — 父子分块与句子窗口检索
- [[rag/03_智能分块/混合索引的分块策略]] — 混合索引下稠密/稀疏双路的切分约束

### 04 混合索引构建
- [[rag/04_混合索引构建/Embedding与向量检索]] — Embedding 模型与向量检索基础
- [[rag/04_混合索引构建/向量数据库选型指南]] — 向量数据库选型对比
- [[rag/04_混合索引构建/检索索引类型与混合索引]] — 索引类型与混合索引构建
- [[rag/04_混合索引构建/ANN近似最近邻搜索]] — HNSW/IVF/PQ/ScaNN 算法详解
- [[rag/04_混合索引构建/索引与部署流水线]] — 索引构建与部署流水线
- [[rag/04_混合索引构建/多语言与中英混排检索]] — 中英混排语料的 Embedding 选型与索引设计

### 05 查询转换与改写
- [[rag/05_查询转换与改写/查询改写与扩展]] — 查询改写与扩展策略
- [[rag/05_查询转换与改写/查询结构化与槽位映射]] — 结构化查询与槽位映射

### 06 混合检索
- [[rag/06_混合检索/混合检索]] — 稀疏+稠密检索与 RRF 融合
- [[rag/06_混合检索/BM25算法详解]] — BM25 算法公式拆解与计算实例
- [[rag/06_混合检索/倒排索引]] — 倒排索引结构与查询执行机制

### 07 元数据过滤
- [[rag/07_元数据过滤/元数据过滤]] — 结构化过滤与门槛过滤

### 08 交叉编码器重排
- [[rag/08_交叉编码器重排/重排序与交叉编码器]] — Cross-Encoder 精排原理与实践

### 09 上下文构建与溯源
- [[rag/09_上下文构建与溯源/上下文构建]] — 上下文组装策略
- [[rag/09_上下文构建与溯源/上下文压缩]] — 上下文压缩方法
- [[rag/09_上下文构建与溯源/溯源与引用]] — 溯源引用与前端渲染

### 10 LLM 生成
- [[rag/10_LLM生成/LLM生成]] — 提示词、结构化输出与模型路由
- [[rag/10_LLM生成/生成校验与防幻觉]] — 生成校验与防幻觉设计

### 11 安全审核与日志
- [[rag/11_安全审核与日志/安全与合规]] — 安全合规与 OWASP LLM Top 10
- [[rag/11_安全审核与日志/审核与审计日志]] — 内容审核与审计日志

### 12 评估与监控
- [[rag/12_评估与监控/离线评估集]] — Golden Set 构建与 RAGAS 评估
- [[rag/12_评估与监控/评估体系与RAGAS]] — RAGAS 评估体系详解
- [[rag/12_评估与监控/在线监控与可观测性]] — 在线监控指标与告警

### 13 工程横切
- [[rag/13_工程横切/生产级RAG系统架构]] — 生产级 RAG 系统架构
- [[rag/13_工程横切/性能与成本优化]] — 性能与成本优化策略
- [[rag/13_工程横切/多租户与大并发]] — 多租户隔离与大并发处理

### 14 范式与前沿
- [[rag/14_范式与前沿/Agentic_RAG]] — Agentic RAG 范式
- [[rag/14_范式与前沿/GraphRAG与知识图谱]] — GraphRAG 与知识图谱
- [[rag/14_范式与前沿/多模态与长上下文RAG]] — 多模态与长上下文 RAG 前沿

---

## a2a — Agent2Agent 协议（跨智能体通信）

> 入口：[[a2a/README|A2A 知识库首页]] · 版本基线：**v1.0.1**（2026-05-28），治理归 Linux Foundation AAIF。**前置：MCP 概念先行**（[[langchain/modules/langchain-mcp/MCP协议概述]]）。
> 定位：A2A 让智能体找到同行（横向），MCP 让智能体拿到工具（纵向）。同一代码库内的 Agent **不需要** A2A。

### 导航与索引（4 页）
- [[a2a/README]] — 总览导航、**阅读门槛声明**（「卡住时该退回哪里」对照）、版本基线、目录结构
- [[a2a/学习路线图]] — 概念优先 / 落地优先两条路径、阶段门槛、毕业自测清单（10 题）
- [[a2a/A2A术语与关键字手册]] — ★ **专有名词总索引**：治理类 / 核心对象 / 角色 / 8 状态 / 交互机制 / 安全 / 生态 / 「不是什么」八组
- [[a2a/速查索引]] — Task 状态表、核心方法名、Agent Card 字段、Part 四形态、版本时间线、安全速查、判定规则、SDK 命令

### 01 入门阶段 · 协议基础（10 页）
> 零基础起点，不要求任何前置知识。知识有强线性依赖，须按章节顺序推进。
> 入口：[[a2a/01-入门阶段-协议基础/00-章节目录|章节目录]]

**第00章 学前准备（3 篇）** — 定位、动机、版本史
- [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.1-A2A是什么]] — 一句话定义、纵向/横向心智模型、官方四条「不是什么」、**你自己的项目需不需要它**
- [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.2-为什么需要A2A]] — 四类困境（集成爆炸 N×(N-1)/2、长任务跟踪、能力发现、跨组织鉴权）、HTTP 类比、设计取舍
- [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.3-版本演进与治理历程]] — 完整时间线、Google→LF→AAIF 三步、v1.0 五项变化、**量化生态热度**

**第01章 核心对象（4 篇）** ★ 本阶段重点
- [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.1-AgentCard与智能体发现]] — 逐字段详解、三条发现路径、扩展卡、签名卡防冒名、**卡是公开的不能写密钥**
- [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.2-Task-Message-Part-Artifact四件套]] — 职责边界、Part 四形态、**结构化数据是一等公民**、Message vs Artifact、contextId
- [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.3-三种交互机制与协议绑定]] — 轮询/流式/推送三层规范结构、三套绑定、extensions 扩展机制
- [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.4-Task生命周期与状态机]] — 8 状态拆解、**`failed` vs `rejected` 的关键区别**、终态不可复活、与 LangGraph 的 `interrupt()` 对照

**第02章 工程实践（2 篇）**
- [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.1-Python-SDK最小可用实现]] — 服务端三件拼装、客户端四步、**亲手实现 `input-required`**、本地 vs 跨组织差别
- [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.2-把远端Agent包装成本地工具]] — agent-as-tool 模式、为什么不包成 MCP server、LangGraph 里的位置、四件必做事

### 02 中段阶段 · 生产实践（6 专题 + 导航）
> 平行专题可跳读；前置：入门阶段已学完。每篇开头标注「什么时候该读」。
> 入口：[[a2a/02-中段阶段-生产实践/00-专题导航|专题导航]]（含「按目标选路径」表）

- [[a2a/02-中段阶段-生产实践/01-安全模型与鉴权]] — ★ 规范强制三件事 vs 不强制一件、五种鉴权方案、**SSRF 与 webhook 校验**、37 项上线检查清单
- [[a2a/02-中段阶段-生产实践/02-版本演进与迁移陷阱]] — v0.3→v1.0 字段级差异、识别旧资料三信号、迁移六步、**为什么单方面切换最危险**
- [[a2a/02-中段阶段-生产实践/03-A2A与MCP选型决策树]] — 核心判定规则、决策树、官方对照表、打分表、混合架构、对方只有私有 API 的折中
- [[a2a/02-中段阶段-生产实践/04-落地时机与不适用场景]] — ★ 五个「该用」信号、五种「不该用」、成熟度 L0-L4、上线自检清单
- [[a2a/02-中段阶段-生产实践/05-与LangGraph多智能体的关系]] — 治理边界分界线、三层分工、`interrupt()`≈`input-required`、**LangGraph 没有 `rejected` 概念**、迁移信号
- [[a2a/02-中段阶段-生产实践/06-生态现状与厂商支持]] — AAIF 协议栈全景、**量化对比（MCP 1:21）**、三大云支持、SDK 与工具、面试答法

---

## docling — 文档解析与结构化

> 按「由浅入深」六层组织：定位与场景 → 整体架构 → 由简到繁的示例 → 文档结构 → 后续处理 → 自定义扩展。全库入口 [[docling/README]]。
> 版本基线：**docling 2.129.0**（2026-09 核实）。

**① 定位与适用场景**
- [[docling/01_定位与适用场景]] — 解决哪些实际问题、三个定位关键词、8 款替代品对比、适用与不适用

**② 整体架构与核心组件**
- [[docling/02_整体架构与核心组件]] — 三层架构（backend / pipeline / datamodel）、完整数据流、26 个后端与 7 类管线、组件职责总表

**③ 由简到繁的示例**
- [[docling/03_快速上手_最小可用示例]] — 3 行代码起步 → 格式选项 → 多格式导出 → 图片落盘；含 CLI 用法
- [[docling/04_进阶用法_配置与选项]] — 两类选项分工、常用开关（标注哪些仅 PDF 生效）、导出模式、批量与并发、加速器

**④ 处理后的文档结构**
- [[docling/05_DoclingDocument结构详解]] — 文档树、元素类型完整清单、prov 位置信息、content_layer 五层、self_ref 引用机制、遍历与导出 API

**⑤ 在此基础上进一步处理**
- [[docling/06_后续处理_导出分块与抽取]] — serializer 族、HybridChunker 四步流程、TreeChunkExpander 父级回溯、可视化调试

**⑥ 自定义与扩展**
- [[docling/07_自定义与扩展]] — 扩展点全景、自定义 backend / pipeline / serializer / chunker、换模型、服务化

**横向支撑**
- [[docling/08_格式支持与依赖管理]] — 34 种格式、extras 全清单、管线映射、按场景的安装决策表
- [[docling/09_已知陷阱与边界]] — 按 DOCX/PDF/分块/依赖/通用五类共 17 条「现象 → 根因 → 应对」，附上线自查清单

> 与 RAG 的关系：docling 属 RAG 流程第 01 环节的解析工具，见 [[rag/01_多源数据接入/文档解析_PDF_Word_图片与表格|解析总览]]。

## langchain — LangChain 框架学习

> 入口：[[langchain/LangChain_学习笔记|LangChain 学习笔记]] · [[langchain/LangChain使用手册|使用手册]] · [[langchain/modules/LangChain模块详解|模块详解]]

### 概述与入门
- [[langchain/overview/LangChain概述]] — 框架定位与 LangChain vs LangGraph vs DeepAgents
- [[langchain/install/安装LangChain]] — 安装方法与提供商包
- [[langchain/quickstart/快速入门]] — 构建首个 Agent
- [[langchain/LangChain使用手册]] — 章节化查询总入口
- [[langchain/LangChain_学习笔记]] — 学习导航与目录结构

### 核心组件
- [[langchain/agents/Agent代理]] — Agent = Model + Harness
- [[langchain/agents/Agent循环实现]] — Agent 循环执行逻辑
- [[langchain/agents/系统提示词]] — 系统提示词编写
- [[langchain/tools/工具]] — 工具定义与使用
- [[langchain/tools/工具来源与集成]] — 工具来源（MCP/自定义/社区）
- [[langchain/tools/工具注册与调度]] — 工具注册与动态调度
- [[langchain/tools/Tavily搜索工具]] — Tavily 搜索工具集成
- [[langchain/models/模型]] — 模型初始化、调用与多提供商配置
- [[langchain/basics/LangChain核心方法对比]] — invoke/stream/batch 对比
- [[langchain/basics/Agent_invoke_vs_ChatModel_invoke]] — Agent vs ChatModel 调用差异

### 记忆与运行时
- [[langchain/memory/短期记忆]] — Checkpointer、裁剪、删除、摘要化
- [[langchain/memory/长期记忆]] — Store 长期记忆机制
- [[langchain/memory/上下文压缩]] — 上下文压缩中间件
- [[langchain/memory/记忆系统性能]] — 记忆系统性能分析与 OOM 防护
- [[langchain/memory/Checkpointer存储后端]] — Checkpointer 后端对比
- [[langchain/memory/示例代码]] — 记忆系统代码示例
- [[langchain/runtime/运行时上下文]] — 依赖注入、Context、Store、Stream Writer

### 高级特性
- [[langchain/streaming/流式传输]] — updates/messages/custom 流式模式
- [[langchain/multi-agent/多Agent系统]] — 子 Agent、交接、路由器
- [[langchain/structured-output/结构化输出]] — ProviderStrategy 与 ToolStrategy
- [[langchain/middleware/中间件概述]] — 钩子系统与内置中间件
- [[langchain/middleware/自定义中间件开发]] — 自定义中间件开发指南
- [[langchain/middleware/最佳实践]] — 中间件最佳实践
- [[langchain/middleware/与LangGraph关系]] — 中间件与 LangGraph 关系
- [[langchain/middleware/钩子机制详解]] — 钩子机制详解
- [[langchain/middleware/ContextEditingMiddleware]] — 上下文编辑中间件
- [[langchain/middleware/FilesystemMiddleware]] — 文件系统中间件
- [[langchain/middleware/FilesystemFileSearchMiddleware]] — 文件搜索中间件
- [[langchain/middleware/HumanInTheLoopMiddleware]] — 人机交互中间件
- [[langchain/middleware/LLMToolEmulator]] — LLM 工具模拟器
- [[langchain/middleware/LLMToolSelectorMiddleware]] — 工具选择中间件
- [[langchain/middleware/ModelCallLimitMiddleware]] — 模型调用限制中间件
- [[langchain/middleware/ModelFallbackMiddleware]] — 模型降级中间件
- [[langchain/middleware/ModelRetryMiddleware]] — 模型重试中间件
- [[langchain/middleware/PIIMiddleware]] — 隐私信息过滤中间件
- [[langchain/middleware/ProviderToolSearchMiddleware]] — 工具搜索中间件
- [[langchain/middleware/RubricMiddleware]] — 评分中间件
- [[langchain/middleware/ShellToolMiddleware]] — Shell 工具中间件
- [[langchain/middleware/SubAgentMiddleware]] — 子 Agent 中间件
- [[langchain/middleware/SummarizationMiddleware]] — 摘要中间件
- [[langchain/middleware/TodoListMiddleware]] — 待办列表中间件
- [[langchain/middleware/ToolCallLimitMiddleware]] — 工具调用限制中间件
- [[langchain/middleware/ToolRetryMiddleware]] — 工具重试中间件

### modules/ — 核心模块 API 详解
- [[langchain/modules/LangChain模块详解]] — 16 个核心模块总览
- [[langchain/modules/langchain_core/概述]] — langchain-core 基础类型
- [[langchain/modules/langchain_agents/概述]] — langchain-agents 模块
- [[langchain/modules/langchain_agents/create_agent参数]] — create_agent 参数详解
- [[langchain/modules/langchain_agents/Agent与LangGraph关系]] — Agent 与 LangGraph 关系
- [[langchain/modules/langchain_callbacks/概述]] — 回调系统
- [[langchain/modules/langchain_callbacks/回调事件类型]] — 回调事件类型
- [[langchain/modules/langchain_callbacks/自定义CallbackHandler]] — 自定义回调处理器
- [[langchain/modules/langchain_callbacks/LangSmith追踪]] — LangSmith 追踪
- [[langchain/modules/langchain_callbacks/Token统计]] — Token 统计
- [[langchain/modules/langchain_checkpoints/概述]] — Checkpoint 模块
- [[langchain/modules/langchain_checkpoints/多轮对话与thread_id]] — 多轮对话与 thread_id
- [[langchain/modules/langchain_checkpoints/人机交互中断与恢复]] — 中断与恢复
- [[langchain/modules/langchain_checkpoints/时间旅行]] — 时间旅行
- [[langchain/modules/langchain_checkpoints/Checkpoint元数据]] — Checkpoint 元数据
- [[langchain/modules/langchain_checkpoints/Checkpointer创建链示例]] — Checkpointer 创建链
- [[langchain/modules/langchain_documents/概述]] — 文档模块
- [[langchain/modules/langchain_documents/文档加载器]] — 文档加载器
- [[langchain/modules/langchain_documents/文本分割器]] — 文本分割器
- [[langchain/modules/langchain_documents/Document数据模型]] — Document 数据模型
- [[langchain/modules/langchain_embeddings/概述]] — Embedding 模块
- [[langchain/modules/langchain_embeddings/OpenAIEmbeddings]] — OpenAI Embeddings
- [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings]] — HuggingFace Embeddings
- [[langchain/modules/langchain_embeddings/智谱AI_Embedding]] — 智谱 AI Embedding
- [[langchain/modules/langchain_embeddings/自定义Embedding实现]] — 自定义 Embedding
- [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings]] — 缓存 Embeddings
- [[langchain/modules/langchain_embeddings/embed_query_vs_embed_documents]] — embed_query vs embed_documents
- [[langchain/modules/langchain_messages/概述]] — 消息模块
- [[langchain/modules/langchain_messages/消息类型详解]] — 消息类型详解
- [[langchain/modules/langchain_messages/多模态消息]] — 多模态消息
- [[langchain/modules/langchain_messages/RemoveMessage历史裁剪]] — RemoveMessage 裁剪
- [[langchain/modules/langchain_models/概述]] — 模型模块
- [[langchain/modules/langchain_models/多提供商配置]] — 多提供商配置
- [[langchain/modules/langchain_models/模型输入输出]] — 模型输入输出
- [[langchain/modules/langchain_models/结构化输出]] — 结构化输出
- [[langchain/modules/langchain_models/bind_tools工具绑定]] — bind_tools 工具绑定
- [[langchain/modules/langchain_output_parsers/概述]] — 输出解析器
- [[langchain/modules/langchain_output_parsers/StrOutputParser]] — 字符串解析器
- [[langchain/modules/langchain_output_parsers/JsonOutputParser]] — JSON 解析器
- [[langchain/modules/langchain_output_parsers/PydanticOutputParser]] — Pydantic 解析器
- [[langchain/modules/langchain_output_parsers/ListOutputParser]] — 列表解析器
- [[langchain/modules/langchain_output_parsers/EnumOutputParser]] — 枚举解析器
- [[langchain/modules/langchain_output_parsers/OutputFixingParser]] — 自动修正解析器
- [[langchain/modules/langchain_output_parsers/错误处理]] — 输出解析错误处理
- [[langchain/modules/langchain_output_parsers/with_structured_output]] — with_structured_output
- [[langchain/modules/langchain_prompts/概述]] — 提示词模块
- [[langchain/modules/langchain_prompts/PromptTemplate]] — PromptTemplate
- [[langchain/modules/langchain_prompts/ChatPromptTemplate]] — ChatPromptTemplate
- [[langchain/modules/langchain_prompts/from_template方法]] — from_template
- [[langchain/modules/langchain_prompts/MessagesPlaceholder]] — MessagesPlaceholder
- [[langchain/modules/langchain_prompts/SystemMessagePromptTemplate]] — SystemMessagePromptTemplate
- [[langchain/modules/langchain_prompts/FewShotPromptTemplate]] — FewShotPromptTemplate
- [[langchain/modules/langchain_prompts/FewShotChatMessagePromptTemplate]] — FewShotChatMessagePromptTemplate
- [[langchain/modules/langchain_prompts/Prompt高级技巧]] — Prompt 高级技巧
- [[langchain/modules/langchain_prompts/Prompt应用场景]] — Prompt 应用场景
- [[langchain/modules/langchain_retrievers/概述]] — 检索器模块
- [[langchain/modules/langchain_retrievers/VectorStoreRetriever]] — 向量存储检索器
- [[langchain/modules/langchain_retrievers/BM25Retriever]] — BM25 检索器
- [[langchain/modules/langchain_retrievers/EnsembleRetriever]] — 集成检索器
- [[langchain/modules/langchain_retrievers/MultiQueryRetriever]] — 多查询检索器
- [[langchain/modules/langchain_retrievers/ContextualCompressionRetriever]] — 上下文压缩检索器
- [[langchain/modules/langchain_retrievers/SelfQueryRetriever]] — 自查询检索器
- [[langchain/modules/langchain_retrievers/自定义Retriever]] — 自定义检索器
- [[langchain/modules/langchain_runnables/概述]] — Runnable 模块
- [[langchain/modules/langchain_runnables/LCEL链式组合]] — LCEL 链式组合
- [[langchain/modules/langchain_runnables/RunnableLambda]] — RunnableLambda
- [[langchain/modules/langchain_runnables/RunnablePassthrough]] — RunnablePassthrough
- [[langchain/modules/langchain_runnables/RunnableParallel]] — RunnableParallel
- [[langchain/modules/langchain_runnables/RunnableBranch]] — RunnableBranch
- [[langchain/modules/langchain_runnables/RunnableSerializable]] — RunnableSerializable
- [[langchain/modules/langchain_runnables/RunnableWithFallbacks]] — RunnableWithFallbacks
- [[langchain/modules/langchain_runnables/RunnableWithMessageHistory]] — RunnableWithMessageHistory
- [[langchain/modules/langchain_stores/概述]] — Store 模块
- [[langchain/modules/langchain_stores/Store核心API]] — Store 核心 API
- [[langchain/modules/langchain_stores/命名空间设计]] — 命名空间设计
- [[langchain/modules/langchain_stores/Store与Checkpointer对比]] — Store vs Checkpointer
- [[langchain/modules/langchain_tools/概述]] — 工具模块
- [[langchain/modules/langchain_tools/工具返回值与状态更新]] — 工具返回值与状态更新
- [[langchain/modules/langchain_tools/ToolRuntime]] — ToolRuntime
- [[langchain/modules/langchain_vectorstores/概述]] — 向量存储模块
- [[langchain/modules/langchain_vectorstores/Chroma]] — Chroma
- [[langchain/modules/langchain_vectorstores/FAISS]] — FAISS
- [[langchain/modules/langchain_vectorstores/InMemoryVectorStore]] — InMemoryVectorStore
- [[langchain/modules/langchain_vectorstores/PGVector]] — PGVector
- [[langchain/modules/langchain_vectorstores/Pinecone]] — Pinecone
- [[langchain/modules/langchain-mcp/概述]] — MCP 协议概述
- [[langchain/modules/langchain-mcp/MCP协议概述]] — MCP 协议详解
- [[langchain/modules/langchain-mcp/MCP服务器开发]] — MCP 服务器开发
- [[langchain/modules/langchain-mcp/MCP服务器示例]] — MCP 服务器示例
- [[langchain/modules/langchain-mcp/MCP安全模型]] — MCP 安全模型
- [[langchain/modules/langchain-mcp/传输方式对比]] — MCP 传输方式对比
- [[langchain/modules/langchain-mcp/MCP提示词编写与执行流程]] — MCP 提示词
- [[langchain/modules/langchain-mcp/MCP与Agent中间件集成]] — MCP 与中间件集成
- [[langchain/modules/langchain-mcp/MultiServerMCPClient]] — 多服务器 MCP 客户端

### 实战教程
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]] — 4 大模块综合实战

---

## langgraph — LangGraph 框架学习

### 概述与入门
- [[langgraph/overview/LangGraph概述]] — LangGraph 定位与核心特性
- [[langgraph/quickstart/快速入门]] — 构建首个图
- [[langgraph/concepts/LangGraph_vs_LangChain]] — LangGraph vs LangChain
- [[langgraph/concepts/图计算范式]] — 图计算范式概念

### modules/ — 核心模块
- [[langgraph/modules/LangGraph模块详解]] — LangGraph 模块总览
- [[langgraph/modules/langgraph_core/概述]] — 核心概念（节点/边/状态/编译）
- [[langgraph/modules/langgraph_core/节点]] — 节点定义
- [[langgraph/modules/langgraph_core/边]] — 边与条件路由
- [[langgraph/modules/langgraph_core/StateGraph]] — StateGraph
- [[langgraph/modules/langgraph_core/Command与状态更新]] — Command 与状态更新
- [[langgraph/modules/langgraph_core/编译与执行]] — 编译与执行
- [[langgraph/modules/langgraph_core/并行执行模型]] — 并行执行模型
- [[langgraph/modules/langgraph_core/错误处理与恢复]] — 错误处理与恢复
- [[langgraph/modules/langgraph_core/可视化与调试]] — 可视化与调试
- [[langgraph/modules/langgraph_state/概述]] — 状态模块
- [[langgraph/modules/langgraph_state/状态定义]] — 状态定义
- [[langgraph/modules/langgraph_state/状态更新模式]] — 状态更新模式
- [[langgraph/modules/langgraph_state/Reducer]] — Reducer
- [[langgraph/modules/langgraph_checkpoint/概述]] — Checkpoint 模块
- [[langgraph/modules/langgraph_checkpoint/InMemorySaver]] — InMemorySaver
- [[langgraph/modules/langgraph_checkpoint/SqliteSaver]] — SqliteSaver
- [[langgraph/modules/langgraph_checkpoint/PostgresSaver]] — PostgresSaver
- [[langgraph/modules/langgraph_checkpoint/thread_id与会话管理]] — thread_id 与会话管理
- [[langgraph/modules/langgraph_human_in_loop/概述]] — 人机交互
- [[langgraph/modules/langgraph_human_in_loop/interrupt机制]] — interrupt 机制
- [[langgraph/modules/langgraph_human_in_loop/审批工作流]] — 审批工作流
- [[langgraph/modules/langgraph_multi_agent/概述]] — 多智能体概述
- [[langgraph/modules/langgraph_multi_agent/主管模式]] — Supervisor 模式
- [[langgraph/modules/langgraph_multi_agent/网络模式]] — Network 模式
- [[langgraph/modules/langgraph_multi_agent/层级模式]] — Hierarchical 模式
- [[langgraph/modules/langgraph_subgraph/概述]] — 子图模块
- [[langgraph/modules/langgraph_subgraph/子图组合模式]] — 子图组合模式
- [[langgraph/modules/langgraph_streaming/概述]] — 流式传输
- [[langgraph/modules/langgraph_streaming/流式模式对比]] — 流式模式对比
- [[langgraph/modules/langgraph_streaming/异步流式]] — 异步流式

### tutorial/ — 实战教程
- [[langgraph/tutorial/ReAct_Agent实现]] — ReAct Agent 实现
- [[langgraph/tutorial/RAG_Agent]] — RAG Agent 实现
- [[langgraph/tutorial/多轮对话Agent]] — 多轮对话 Agent
- [[langgraph/tutorial/综合示例_全概念演示]] — 全概念演示
- [[langgraph/tutorial/测试模式]] — 测试模式
- [[langgraph/tutorial/生产部署清单]] — 生产部署清单

### interview/ — 面试题库（8 篇、69 题）
> 按主题归档的逐题问答库，**每篇 ≤10 题、同篇同主题**。入口：[[langgraph/interview/README]]

- [[langgraph/interview/00-面试总纲与答题框架]] — 迁移自旧 interview 目录：30 秒答法、项目话术、高频追问速查
- [[langgraph/interview/01-核心概念与运行模型]] — 定位、LangChain 关系、superstep、Channel（9 题）
- [[langgraph/interview/02-状态与Reducer]] — TypedDict、Annotated、并行写入、add_messages（8 题）
- [[langgraph/interview/03-控制流与动态编排]] — 条件边、`Send`、`Command`、循环、fan-in（9 题）
- [[langgraph/interview/04-持久化与Checkpointer]] — 快照粒度、thread_id、时间旅行、Store（9 题）
- [[langgraph/interview/05-人机协同与中断恢复]] — interrupt 原理、幂等、多级审批（8 题）
- [[langgraph/interview/06-流式输出与可观测性]] — stream_mode、astream_events、SSE（8 题）
- [[langgraph/interview/07-多智能体与子图]] — Supervisor、Swarm、层级、上下文隔离（9 题）
- [[langgraph/interview/08-生产实践与高频追问]] — 部署、并发、超时重试、事故清单（9 题）

---

## deepagents — DeepAgents 框架

- [[deepagents/overview/DeepAgents概述]] — DeepAgents 定位与默认中间件栈
- [[deepagents/overview/快速入门]] — 快速上手
- [[deepagents/overview/与LangChain_Agent对比]] — 与 LangChain Agent 对比
- [[deepagents/overview/组件依赖全景]] — 组件依赖全景图
- [[deepagents/middleware/中间件栈概述]] — 默认中间件栈
- [[deepagents/middleware/FilesystemMiddleware详解]] — 文件系统中间件详解
- [[deepagents/middleware/HumanInTheLoop详解]] — 人机交互中间件详解
- [[deepagents/customization/自定义概述]] — 自定义概述
- [[deepagents/customization/create_deep_agent参数详解]] — create_deep_agent 参数
- [[deepagents/customization/HarnessProfile深度用法]] — HarnessProfile 用法
- [[deepagents/customization/State扩展与DeltaChannel]] — State 扩展与 DeltaChannel
- [[deepagents/context/上下文管理概述]] — 上下文管理
- [[deepagents/context/长期记忆与AGENTS]] — 长期记忆与 AGENTS.md
- [[deepagents/context/自动摘要机制]] — 自动摘要机制
- [[deepagents/subagents/子代理概述]] — 子代理概述
- [[deepagents/subagents/子代理配置实战]] — 子代理配置实战
- [[deepagents/tools/内置工具概述]] — 内置工具
- [[deepagents/tools/文件系统工具详解]] — 文件系统工具详解
- [[deepagents/tools/MCP集成]] — MCP 集成
- [[deepagents/tools/Task子代理工具详解]] — Task 子代理工具
- [[deepagents/skills/Skills系统]] — Skills 系统
- [[deepagents/backend/后端概述]] — 后端概述
- [[deepagents/backend/Backend实战配置]] — 后端实战配置
- [[deepagents/backend/Sandbox后端详解]] — 沙箱后端详解
- [[deepagents/production/测试策略]] — 测试策略
- [[deepagents/production/错误处理与重试]] — 错误处理与重试
- [[deepagents/production/生产部署]] — 生产部署
- [[deepagents/production/源码架构]] — 源码架构
- [[deepagents/production/Streaming与事件流]] — Streaming 与事件流

---

## docker — Docker 容器技术

> 三阶段 + 一个应用专题。阅读前提、版本基线与"卡住时查哪里"见 [[docker/README]]。

### 导航与索引（4 页）
- [[docker/README]] — 总览导航、版本基线与阅读前提声明
- [[docker/学习路线图]] — 三阶段路径、能力出口与自测清单
- [[docker/Docker命令与术语手册]] — 命令、参数、名词、报错用词总索引
- [[docker/速查索引]] — 命令骨架、Dockerfile 指令、compose 字段速查

### 01 入门阶段 · 顺序章节式（6 章 20 篇）
> 零基础起点，不要求任何前置知识。知识有线性依赖，须按章节顺序推进。
> 入口：[[docker/01-入门阶段-基础教程/00-章节目录]]

**第00章 学前准备（5 篇）** — 容器是什么、Docker 架构、装环境跑通、看懂报错
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.1-容器与Docker是什么]] — 容器 vs 虚拟机、镜像/容器/仓库三件套
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.2-Docker的组成与运行架构]] — CLI/daemon/containerd/runc 四层与 run 的完整旅程
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.3-跑通第一个容器]] — Windows/WSL2 与 Linux 安装、验证三连、镜像加速器
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] — 五层定位法与 12 类常见报错
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.5-学前名词与学习地图]] — 20 个核心名词、六章地图、8 个误区

**第01章 镜像与容器基础操作（3 篇）** — 会管镜像、会控容器
- [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.1-镜像的获取与管理]] — 镜像名结构、tag/latest 陷阱、save/load
- [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.2-容器的创建与运行]] — run/create/start 区别、PID 1、容器为何立刻退出
- [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.3-进入容器与查看日志]] — exec vs attach、logs、inspect、cp、stats

**第02章 镜像构建与 Dockerfile（3 篇）** — 会写生产可用的 Dockerfile
- [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] — 全部指令、CMD vs ENTRYPOINT、ENV vs ARG
- [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] — 构建上下文、.dockerignore、缓存命中规则
- [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.3-多阶段构建与最佳实践]] — 多阶段构建与 20 条最佳实践

**第03章 数据卷与容器网络（3 篇）** — 数据不丢、容器能互通
- [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] — 卷/绑定/tmpfs 对比、UID 权限坑、备份恢复
- [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.2-容器网络模型]] — bridge/host/none、自定义网络与内置 DNS
- [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.3-端口映射与访问]] — 端口方向、四种绑定形式、访问不到的两大原因

**第04章 多容器编排 Compose（3 篇）** — 一条命令起一套服务
- [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.1-Compose是什么与文件结构]] — 四个顶层字段、项目名前缀、depends_on 陷阱
- [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.2-用Compose编排多容器服务]] — 应用+PG+Redis 完整编排与操作闭环
- [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.3-环境变量与多环境配置]] — .env/env_file 的区别、优先级、多文件合并、密钥

**第05章 资源限制与日常维护（3 篇）** — 让容器跑在可控范围内
- [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.1-资源限制]] — 内存/CPU/进程数/句柄、swap 语义、OOM 诊断
- [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] — 五个占用来源、日志轮转、清理三档分级
- [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.3-容器排错流程]] — 七类现象的排查序列与诊断信息收集

### 02 中段阶段 · 平行专题式（8 专题 8 篇）
> 专题之间无依赖，可跳读。**前置：入门阶段已学完。**
> 入口：[[docker/02-中段阶段-专题精讲/00-专题导航]]

- [[docker/02-中段阶段-专题精讲/镜像优化与瘦身/专题概览]] — 镜像为什么大、六个瘦身手段、构建与拉取加速
- [[docker/02-中段阶段-专题精讲/数据持久化与备份/专题概览]] — 存储层次、备份三要素、逻辑/物理备份、迁移与升级
- [[docker/02-中段阶段-专题精讲/网络进阶与多容器互联/专题概览]] — veth/bridge/NAT/内置 DNS、分层网络与隔离
- [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] — 隔离强度、capabilities、seccomp、只读根文件系统、加固清单
- [[docker/02-中段阶段-专题精讲/资源限制与性能/专题概览]] — cgroup 原理、CPU 限流诊断、容量规划
- [[docker/02-中段阶段-专题精讲/日志与监控/专题概览]] — 日志驱动、PLG 集中日志栈、三层监控、健康检查
- [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] — profiles/锚点/include/watch、Docker SDK 动态控容器
- [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] — tag 策略、buildx 多平台、CI 缓存、供应链安全、回滚

### 沙箱 · 应用专题（1 导航 + 12 篇）
> 用 Docker 给 AI Agent 提供安全的代码执行环境。**前置：入门 + 中段已学完。**
> 入口：[[docker/沙箱/00-沙箱专题导航]]

- [[docker/沙箱/00-沙箱专题导航]] — 专区定位、前置要求与 12 篇索引
- [[docker/沙箱/Docker沙箱概述]] — 三类风险、整体架构、三种方案对比与选型
- [[docker/沙箱/Docker_Sandbox核心类]] — SandboxConfig 与 DockerSandbox 完整实现
- [[docker/沙箱/沙箱实战手册]] — 可直接照着做的操作手册
- [[docker/沙箱/沙箱安全加固]] — 网络隔离、镜像扫描、密钥管理、只读文件系统
- [[docker/沙箱/沙箱多租户隔离]] — 多用户场景的命名空间、配额与数据边界
- [[docker/沙箱/沙箱性能优化]] — 镜像瘦身、预热池、缓存策略、并发控制
- [[docker/沙箱/沙箱监控与日志]] — 结构化日志、审计追踪、metrics
- [[docker/沙箱/沙箱错误处理与容错]] — 重试、超时、优雅降级
- [[docker/沙箱/沙箱调试与故障排查]] — 排错流程与常见故障
- [[docker/沙箱/沙箱生产部署清单]] — 上线检查清单、Dockerfile 模板、CI/CD
- [[docker/沙箱/沙箱与Agent集成实践]] — 与 LangChain / LangGraph Agent 的完整组装
- [[docker/沙箱/沙箱常见使用案例]] — 典型落地场景

## redis — Redis 学习知识库

> 入门（15 篇）+ 中段生产实践（12 专题）已建成；进阶为规划。阅读前提、版本基线（Redis 8.10.2 / redis-py 8.1.0）与"卡住时查哪里"见 [[redis/README]]。

### 导航与索引（4 页）
- [[redis/README]] — 总览导航、版本基线（许可证变更、Stack 并入开源版、redis-py 8 默认值）与阅读前提声明
- [[redis/学习路线图]] — 三阶段路径、入门毕业自测清单、中段 12 专题与进阶 4 专题规划
- [[redis/Redis命令与术语手册]] — 命令、选项、redis-py 方法、术语总索引（含中段/进阶术语防恐慌表）
- [[redis/速查索引]] — 命令骨架、响应前缀↔Python 返回类型对照、危险命令清单、速记口诀

### 01 入门阶段 · 顺序章节式（3 章 + 学前准备，15 篇）
> 零基础起点，不要求任何前置知识。知识有线性依赖，须按章节顺序推进。
> 入口：[[redis/01-入门阶段-基础教程/00-章节目录]]

**第00章 学前准备（4 篇）** — Redis 是什么、装环境跑通、看懂报错
- [[redis/01-入门阶段-基础教程/第00章-学前准备/0.1-Redis是什么]] — 心智模型（超级字典）、为什么快、与 PostgreSQL 分工
- [[redis/01-入门阶段-基础教程/第00章-学前准备/0.2-安装与启动]] — Docker/WSL2 安装、PING→PONG、端口与挂卷
- [[redis/01-入门阶段-基础教程/第00章-学前准备/0.3-redis-cli第一次连接]] — 五种响应前缀、引号规则、HELP 体系
- [[redis/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] — 排错五步法、WRONGTYPE 等 8 类报错

**第01章 键与五种核心数据类型（7 篇）** — 会选型、会操作
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.1-Key与Value的世界观]] — 字节序列、冒号命名、16 个 db、KEYS 为何禁用
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.2-五种核心数据类型总览]] — 按高频操作选型的一张表
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] — SET 全选项、INCR 原子计数、MGET 批量
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.4-List列表]] — 双端序列、BLPOP 阻塞队列、LTRIM
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.5-Hash哈希]] — 字段级读写、HINCRBY、会话存储
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.6-Set集合]] — 去重、O(1) 判在、交并差
- [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.7-SortedSet有序集合]] — 名次 vs 分数区间（BYSCORE）、排行榜

**第02章 过期机制与通用 Key 命令（2 篇）** — 数据有寿命、遍历不阻塞
- [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.1-Key的过期机制]] — TTL 三返回、惰性+定期删除、SET 清 TTL 大坑
- [[redis/01-入门阶段-基础教程/第02章-过期机制与通用Key命令/2.2-通用Key命令与SCAN]] — DEL vs UNLINK、SCAN 游标与三条约定

**第03章 Python 客户端 redis-py（2 篇）** — 接入项目
- [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.1-连接与基础操作]] — decode_responses、方法映射、连接池、asyncio
- [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.2-操作五种数据类型]] — 返回类型对照、pipeline、Agent 缓存小案例

### 02 中段阶段 · 生产实践（8 专题）
> 平行专题可跳读；前置：入门阶段已学完。每篇含生产配置、监控指标与演练步骤。
> 入口：[[redis/02-中段阶段-生产实践/00-专题导航]]（含"按目标选路径"表）

- [[redis/02-中段阶段-生产实践/01-持久化-RDB与AOF]] — 丢失窗口三角、fork/COW 机制、混合持久化配置与恢复演练
- [[redis/02-中段阶段-生产实践/02-发布订阅与Stream]] — 消费者组三件套、pending/XAUTOCLAIM、at-least-once 任务分发设计
- [[redis/02-中段阶段-生产实践/03-缓存三大问题与缓存模式]] — Cache-Aside、穿透/击穿/雪崩解法、延迟双删、Agent 缓存落地
- [[redis/02-中段阶段-生产实践/04-事务与Lua脚本]] — 不回滚语义、WATCH 乐观锁、Lua 原子脚本与三者选型
- [[redis/02-中段阶段-生产实践/05-管道与性能]] — RTT 数学、SLOWLOG/bigkeys/LATENCY 四板斧、三条性能守则
- [[redis/02-中段阶段-生产实践/06-分布式锁]] — 三零件锁、看门狗续期、Redlock 争议与效率锁/正确性锁分界
- [[redis/02-中段阶段-生产实践/07-内存淘汰与maxmemory]] — 8 种策略选型、大 key 治理、水位监控
- [[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]] — 复制/哨兵/集群三级选型、脑裂防护、故障转移演练
- [[redis/02-中段阶段-生产实践/09-缓存一致性-与数据库同步]] — 四种写序推演、延迟双删生产化、重试队列/binlog 订阅选型、读己之写
- [[redis/02-中段阶段-生产实践/10-大Key与热Key治理]] — 判定标准量化、发现工具全景、治理 SOP、热 key 三招与应急预案
- [[redis/02-中段阶段-生产实践/11-安全加固与生产监控]] — 网络隔离、ACL、危险命令治理、INFO 全景告警、上线 checklist
- [[redis/02-中段阶段-生产实践/12-Agent场景实战-缓存会话与任务分发]] — 四角色架构、LLM/embedding 缓存、会话与 RedisSaver、Stream 任务分发、限流

## fastapi — FastAPI 集成

- [[fastapi/FastAPI概述]] — FastAPI 概述与定位
- [[fastapi/概述]] — FastAPI 核心概念
- [[fastapi/快速入门]] — 快速入门
- [[fastapi/项目结构]] — 项目结构
- [[fastapi/路径操作]] — 路径操作与参数
- [[fastapi/请求体与Pydantic]] — 请求体与 Pydantic 验证
- [[fastapi/依赖注入]] — 依赖注入系统
- [[fastapi/中间件与CORS]] — 中间件与 CORS
- [[fastapi/异步处理]] — 异步处理
- [[fastapi/流式响应]] — 流式响应（SSE）
- [[fastapi/错误处理与响应模型]] — 错误处理与响应模型
- [[fastapi/安全认证]] — 安全认证（JWT/OAuth2）
- [[fastapi/后台任务与长时间Agent作业]] — 后台任务与 Agent 作业
- [[fastapi/WebSocket与Agent实时通信]] — WebSocket 实时通信
- [[fastapi/文件上传与RAG文档接口]] — 文件上传与 RAG 接口
- [[fastapi/与LangChain集成]] — 与 LangChain 集成
- [[fastapi/测试]] — 测试
- [[fastapi/部署]] — 部署

---

## python — Python 基础

### basics/ — 基础
- [[python/basics/Python_基础概览]] — Python 基础概览
- [[python/basics/数值类型]] — 数值类型
- [[python/basics/字符串]] — 字符串操作
- [[python/basics/控制流]] — 控制流
- [[python/basics/异常处理]] — 异常处理
- [[python/basics/模块与包]] — 模块与包
- [[python/basics/正则表达式]] — 正则表达式
- [[python/basics/数学函数]] — 数学函数与 NumPy

### data_structures/ — 数据结构
- [[python/data_structures/列表]] — 列表
- [[python/data_structures/元组]] — 元组
- [[python/data_structures/字典]] — 字典
- [[python/data_structures/集合]] — 集合

### advanced/ — 进阶
- [[python/advanced/类型提示进阶]] — 类型提示进阶
- [[python/advanced/异步编程]] — 异步编程（asyncio）
- [[python/advanced/装饰器进阶]] — 装饰器进阶
- [[python/advanced/生成器与异步迭代器]] — 生成器与异步迭代器
- [[python/advanced/functools函数工具]] — functools 函数工具
- [[python/advanced/Pydantic数据验证]] — Pydantic 数据验证
- [[python/advanced/序列化与状态管理]] — 序列化与状态管理
- [[python/advanced/SimpleNamespace轻量对象]] — SimpleNamespace 轻量对象

### 其他
- [[python/functions/函数]] — 函数定义与用法
- [[python/oop/类与面向对象]] — 类与面向对象
- [[python/oop/魔法方法]] — 魔法方法
- [[python/io/输入输出]] — 输入输出
- [[python/io/文件操作]] — 文件操作
- [[python/testing/pytest与fixture]] — pytest 与 fixture

---

## sqlmodel — SQLModel ORM

- [[sqlmodel/概述]] — SQLModel 概述与定位
- [[sqlmodel/模型定义]] — 模型定义
- [[sqlmodel/引擎与Session]] — 引擎与 Session
- [[sqlmodel/CRUD操作]] — CRUD 操作
- [[sqlmodel/关系与关联]] — 关系与关联
- [[sqlmodel/查询进阶]] — 查询进阶

---

## tools — 工具集

### 向量数据库
- [[tools/向量数据库/概述]] — 向量数据库总览
- [[tools/向量数据库/chroma/概述]] — Chroma 概述
- [[tools/向量数据库/chroma/核心原理]] — Chroma 核心原理
- [[tools/向量数据库/chroma/快速入门]] — Chroma 快速入门
- [[tools/向量数据库/chroma/持久化与部署]] — Chroma 持久化与部署
- [[tools/向量数据库/chroma/与LangChain集成]] — Chroma 与 LangChain 集成
- [[tools/向量数据库/faiss/概述]] — FAISS 概述
- [[tools/向量数据库/milvus/概述]] — Milvus 概述
- [[tools/向量数据库/pgvector/概述]] — pgvector 概述
- [[tools/向量数据库/pinecone/概述]] — Pinecone 概述
- [[tools/向量数据库/qdrant/概述]] — Qdrant 概述
- [[tools/向量数据库/weaviate/概述]] — Weaviate 概述

### 其他工具
- [[tools/Chroma]] — Chroma 快速参考
- [[tools/Docker]] — 本机 PostgreSQL 容器的命令速记（**非 Docker 教程**，系统学习见 [[docker/README]]）
- [[tools/Git]] — Git 版本控制
- [[tools/Gradio]] — Gradio UI 框架
- [[tools/Streamlit]] — Streamlit UI 框架
- [[tools/Dataview]] — Obsidian Dataview 插件
- [[tools/Graph_View]] — Obsidian 图谱视图
- [[tools/Marp]] — Markdown 演示文稿
- [[tools/Memex]] — Memex 知识管理概念
- [[tools/qmd]] — qmd 本地搜索引擎

---

## postgresql — PostgreSQL 学习知识库

> 三阶段按三种不同组织范式编排：入门＝顺序章节式、中段＝平行专题式、进阶＝分层剖析式。版本基线：PG 18.6（最新稳定版）。入口：[[postgresql/README]]

### 导航（5 页）
- [[postgresql/README]] — 总览导航、阅读前提（零基础入口与"卡住时查哪里"对照表）、难度地图、版本基线、三阶段组织范式选型理由
- [[postgresql/学习路线图]] — 三阶段路径、能力出口与自测清单、各阶段阅读门槛说明
- [[postgresql/术语与关键字手册]] — ★ **名词与关键字总索引**：SQL 关键字／标点符号／psql 元命令／PG 专有术语／系统列名／系统目录列名／配置参数／报错用词
- [[postgresql/函数手册]] — ★ **函数总索引**：全套文档出现过的每个函数（作用／签名／示例／详解位置）＋ 查不到函数时的四步法
- [[postgresql/速查索引]] — psql/系统视图/类型/SQL 骨架/调优参数/扩展速查

### 01 入门阶段 · 基础教程式（27 页）
- [[postgresql/01-入门阶段-基础教程/00-章节目录]] — 章节目录、学习顺序说明、统一实践数据集

**第00章 学前准备（零基础起点，不要求任何前置知识）**
- [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.1-数据库与SQL是什么]] — 数据库解决的四个问题、库/表/行/列、SQL 与声明式、客户端与服务端、三条书写规则
- [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.2-一条SQL语句的组成]] — 关键字/标识符/字面量/运算符、单引号 vs 双引号、标点符号速查、psql 元命令为何不加分号
- [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.3-跑通第一条语句]] — Docker 拉实例、psql 连接、建库→建表→增删改查全流程、卡住时的应对表
- [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] — 报错三段结构（ERROR/LINE/HINT）、12 类常见报错逐条排查、提示符变化不等于报错
- [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.5-学前名词与学习地图]] — 20+ 核心名词一句话解释、学习顺序的依赖链、看不懂时的三个查询入口

**第01章 环境与初识**
- [[postgresql/01-入门阶段-基础教程/第01章-环境与初识/1.1-PostgreSQL概览与版本演进]] — PostgreSQL 三个特点、本篇读法（后几节可跳过）、版本号规则与支持周期、PG18 五大变化
- [[postgresql/01-入门阶段-基础教程/第01章-环境与初识/1.2-安装与psql入门]] — Docker 拉实例、psql 元命令、三个配置文件、运行期改参

**第02章 数据类型**
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.1-数值类型]] — 整数/精确小数/浮点取舍、IDENTITY 优于 serial、取整函数、width_bucket 数值分桶
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.2-字符与二进制类型]] — varchar 与 text 无性能差异、排序规则影响索引、bytea
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.3-日期时间类型]] — timestamptz 存 UTC 瞬时、interval 的月日依赖、半开区间
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.4-布尔枚举与UUID]] — 三值逻辑、枚举 vs CHECK vs 查找表、PG18 uuidv7 与索引局部性
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.5-数组与JSON初步]] — 数组运算符、复合类型、域、json vs jsonb
- [[postgresql/01-入门阶段-基础教程/第02章-数据类型/2.6-NULL语义与类型转换]] — 三值逻辑真值表、NOT IN 的 NULL 陷阱、隐式转换致索引失效

**第03章 建表与约束**
- [[postgresql/01-入门阶段-基础教程/第03章-建表与约束/3.0-表主键与外键]] — **概念篇**：为什么一张表只能描述一种事物、主键的三种做法、外键拦住的孤儿数据、一对一/一对多/多对多三种关系与中间表
- [[postgresql/01-入门阶段-基础教程/第03章-建表与约束/3.1-DDL建表与列定义]] — 生产级建表模板、生成列 STORED/VIRTUAL（PG18 默认变 VIRTUAL）、ALTER 在线化
- [[postgresql/01-入门阶段-基础教程/第03章-建表与约束/3.2-约束体系与数据完整性]] — 五类约束、外键列不自动建索引、CHECK 的 NULL 语义、EXCLUDE

**第04章 基础操作 CRUD**
- [[postgresql/01-入门阶段-基础教程/第04章-基础操作CRUD/4.1-INSERT与UPSERT]] — ON CONFLICT 幂等写入、xmax 判插入/更新、COPY 批量导入
- [[postgresql/01-入门阶段-基础教程/第04章-基础操作CRUD/4.2-SELECT基础查询]] — 逻辑执行顺序、DISTINCT ON、CASE、不用 SELECT *
- [[postgresql/01-入门阶段-基础教程/第04章-基础操作CRUD/4.3-UPDATE与DELETE]] — UPDATE FROM、追加写与膨胀、TRUNCATE 对比、分批处理

**第05章 查询核心**
- [[postgresql/01-入门阶段-基础教程/第05章-查询核心/5.1-WHERE与运算符]] — BETWEEN 闭区间陷阱、LIKE 与 C collation、NOT IN 与 NULL、EXISTS
- [[postgresql/01-入门阶段-基础教程/第05章-查询核心/5.2-ORDER-BY与分页]] — NULL 位置与 collation、键集分页、排序与索引方向匹配
- [[postgresql/01-入门阶段-基础教程/第05章-查询核心/5.3-聚合函数与GROUP-BY]] — 聚合忽略 NULL、WHERE vs HAVING、FILTER、ROLLUP/CUBE
- [[postgresql/01-入门阶段-基础教程/第05章-查询核心/5.4-多表连接JOIN]] — 五类 JOIN、LEFT JOIN 条件下放陷阱、半/反连接、LATERAL

**第06章 事务与权限入门**
- [[postgresql/01-入门阶段-基础教程/第06章-事务与权限入门/6.1-事务基础]] — ACID、SAVEPOINT、隔离级别实现映射、40001 重试、长事务危害
- [[postgresql/01-入门阶段-基础教程/第06章-事务与权限入门/6.2-角色与权限入门]] — 角色即用户、权限层次链、ALTER DEFAULT PRIVILEGES、RLS、scram-sha-256

### 02 中段阶段 · 专题精讲式（22 页）
- [[postgresql/02-中段阶段-专题精讲/00-专题导航]] — 11 个平行专题导航与选型索引

**窗口函数**
- [[postgresql/02-中段阶段-专题精讲/窗口函数/专题概览]] — 保留明细行做聚合、frame 默认值陷阱
- [[postgresql/02-中段阶段-专题精讲/窗口函数/语法与函数速查]] — 全部函数与 frame 语法、last_value 陷阱
- [[postgresql/02-中段阶段-专题精讲/窗口函数/实战-排名与同比环比]] — 组内 TopN、累计、移动平均、同比环比、连续 streak

**CTE 与递归查询**
- [[postgresql/02-中段阶段-专题精讲/CTE与递归查询/专题概览]] — 物化 vs 内联（PG12+）、MATERIALIZED 显式控制、DML CTE
- [[postgresql/02-中段阶段-专题精讲/CTE与递归查询/递归查询与树形结构]] — WITH RECURSIVE、三种树存储方案、图遍历限制、SEARCH/CYCLE

**JSON 与 JSONB**
- [[postgresql/02-中段阶段-专题精讲/JSON与JSONB/专题概览]] — jsonb vs json、操作符、SQL/JSON 路径、索引策略、建模边界
- [[postgresql/02-中段阶段-专题精讲/JSON与JSONB/操作符与函数速查]] — 操作符/构造/展开/修改函数全表
- [[postgresql/02-中段阶段-专题精讲/JSON与JSONB/实战-半结构化建模与GIN索引]] — 核心列+JSONB、生成列、jsonb_ops vs jsonb_path_ops

**数组 / 范围 / 枚举**
- [[postgresql/02-中段阶段-专题精讲/数组范围与枚举/专题概览]] — @> 与 = ANY 语义差异、范围规范化、EXCLUDE 防重叠、枚举三方案决策树

**索引实战应用**
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/专题概览]] — 索引代价、六种方法一览、查询匹配速查、无用索引排查
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/索引类型选型与设计]] — B-tree 最左前缀矩阵、表达式/部分/覆盖索引、GIN/GiST/SP-GiST/BRIN
- [[postgresql/02-中段阶段-专题精讲/索引实战应用/实战-慢查询索引优化]] — 优化流程 + 8 个真实案例

**执行计划与性能诊断**
- [[postgresql/02-中段阶段-专题精讲/执行计划与性能诊断/专题概览]] — 工具箱、节点类型、诊断决策树
- [[postgresql/02-中段阶段-专题精讲/执行计划与性能诊断/EXPLAIN与EXPLAIN-ANALYZE解读]] — 逐字段解读、Index Cond vs Filter、Buffers、SSD 必调 random_page_cost

**其他专题**
- [[postgresql/02-中段阶段-专题精讲/全文检索/专题概览]] — tsvector/tsquery、中文三种方案、GIN 索引、pg_trgm
- [[postgresql/02-中段阶段-专题精讲/分区表/专题概览]] — 三种分区策略、裁剪写法约束、DETACH 归档、PG18 行为变化
- [[postgresql/02-中段阶段-专题精讲/视图函数与触发器/专题概览]] — 视图/物化视图、函数易变性、触发器与 security_invoker
- [[postgresql/02-中段阶段-专题精讲/视图函数与触发器/PLpgSQL编程]] — PL/pgSQL 语法、EXCEPTION 即子事务、性能要点
- [[postgresql/02-中段阶段-专题精讲/备份恢复与高可用/专题概览]] — pg_dump/pg_basebackup/PITR/流复制/逻辑复制、synchronous_commit 五档
- [[postgresql/02-中段阶段-专题精讲/备份恢复与高可用/升级与迁移]] — 三种升级方式、pg_upgrade 与 PG18 保留统计、升级前检查清单
- [[postgresql/02-中段阶段-专题精讲/扩展生态/专题概览]] — pg_stat_statements/pg_trgm/pgvector/PostGIS/pg_cron/FDW

### 03 进阶阶段 · 深度剖析式（20 页）
- [[postgresql/03-进阶阶段-深度剖析/00-剖析方法论]] — 六段模板、九模块依赖图、源码阅读方法、必备工具箱

**进程与内存架构**
- [[postgresql/03-进阶阶段-深度剖析/进程与内存架构/原理剖析-进程模型与连接]] — 多进程模型、三类进程、ProcArray 与 MVCC 的关系、连接池
- [[postgresql/03-进阶阶段-深度剖析/进程与内存架构/原理剖析-共享内存与缓冲池]] — 双层缓存、work_mem 是"每操作"、时钟扫描、bgwriter

**存储引擎与页结构**
- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/原理剖析-堆表与页结构]] — 8KB 页布局、pd_lower/pd_upper 几何、行指针与元组头、hint bits
- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/原理剖析-TOAST与行版本]] — TOAST 四级策略、压缩 pglz/lz4、TOAST 表带索引的写入放大
- [[postgresql/03-进阶阶段-深度剖析/存储引擎与页结构/实验验证-物理布局观测]] — 10 个可执行实验（页头/元组头/VM/TOAST/膨胀/amcheck）

**事务与 MVCC 内核**
- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-MVCC与快照隔离]] — 追加式 MVCC、快照三字段、SI 允许写偏斜、长事务阻塞 VACUUM
- [[postgresql/03-进阶阶段-深度剖析/事务与MVCC内核/原理剖析-可见性与事务状态]] — 可见性判定伪码、pg_xact、hint bits、冻结与 xid 回绕防护

**索引内核**
- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-BTree结构]] — 页类型、去重、高键、右链 Lehman-Yao、PG18 Skip Scan
- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-GIN-GiST-BRIN对比]] — 四种方法的结构与适用、BRIN 的 correlation 前提
- [[postgresql/03-进阶阶段-深度剖析/索引内核/原理剖析-HOT更新与索引维护]] — HOT 三条件、LP_REDIRECT、fillfactor、索引膨胀治理

**查询优化器**
- [[postgresql/03-进阶阶段-深度剖析/查询优化器/原理剖析-优化阶段与代价模型]] — 四阶段、代价公式、为何有索引仍选 Seq Scan、Nested Loop 事故
- [[postgresql/03-进阶阶段-深度剖析/查询优化器/原理剖析-统计信息与基数估计]] — pg_stats 字段、扩展统计修正列相关性、autoanalyze 触发

**WAL 与崩溃恢复**
- [[postgresql/03-进阶阶段-深度剖析/WAL与崩溃恢复/原理剖析-WAL与LSN机制]] — 预写铁律、LSN 语义、full-page writes、synchronous_commit 五档
- [[postgresql/03-进阶阶段-深度剖析/WAL与崩溃恢复/原理剖析-Checkpoint与崩溃恢复]] — REDO 起点、检查点参数、恢复流程、pg_control 解读

**并发控制与锁内核**
- [[postgresql/03-进阶阶段-深度剖析/并发控制与锁内核/原理剖析-锁管理器与死锁]] — 四层锁体系、行锁靠 xmax、8 种表锁冲突矩阵、SKIP LOCKED、死锁检测
- [[postgresql/03-进阶阶段-深度剖析/并发控制与锁内核/原理剖析-SSI可串行化隔离]] — rw 反依赖环、SIREAD 谓词锁、写偏斜实证、40001 必须重试整个事务

**复制与高可用内核**
- [[postgresql/03-进阶阶段-深度剖析/复制与高可用内核/原理剖析-流复制与同步提交]] — walsender/walreceiver/startup、三个 LSN、复制槽高危性、仲裁提交、备库查询冲突
- [[postgresql/03-进阶阶段-深度剖析/复制与高可用内核/原理剖析-逻辑复制与解码]] — 逻辑解码原理、REPLICA IDENTITY、大事务延迟、PG19 复制序列值

**扩展与内核开发**
- [[postgresql/03-进阶阶段-深度剖析/扩展与内核开发/原理剖析-扩展框架与FDW]] — 扩展三件套、钩子与后台 worker、安全模型、FDW 条件下推验证

### interview/ — 面试题库（8 篇、67 题）
> 按主题归档的逐题问答库，**每篇 ≤10 题、同篇同主题**。入口：[[postgresql/interview/README]]

- [[postgresql/interview/01-架构与存储模型]] — 进程模型、shared_buffers、页、TOAST、WAL（8 题）
- [[postgresql/interview/02-MVCC与事务隔离]] — 行版本、快照、四种隔离级别、XID 回卷（9 题）
- [[postgresql/interview/03-锁与并发控制]] — 八种表锁、四种行锁、死锁排查（8 题）
- [[postgresql/interview/04-索引原理与选型]] — B-tree、GIN/GiST/BRIN、HOT（9 题）
- [[postgresql/interview/05-查询优化与执行计划]] — EXPLAIN、代价模型、连接算法（8 题）
- [[postgresql/interview/06-VACUUM与空间回收]] — 死元组、autovacuum、膨胀、freeze（8 题）
- [[postgresql/interview/07-复制备份与高可用]] — 流复制、逻辑复制、PITR、切换（8 题）
- [[postgresql/interview/08-PG在AI应用中的实践]] — pgvector、混合检索、Checkpointer（9 题）

---

## mysql — MySQL 学习知识库

> 入口：[[mysql/README|MySQL 首页]] · 版本基线：**LTS 选 8.4 或 9.7**（⚠️ **8.0 已于 2026-04-30 EOL**；⚠️ 26.7 起改日历版本号 YY.M.P）。
> 与 [[postgresql/README|PostgreSQL]] 是并列姊妹库，⚠️ **语法和 MVCC 实现差异极大，照搬会踩坑**——先读差异页。
> 三阶段同构：入门（顺序章节式，**不可跳读**）→ 中段（平行专题，跳读）→ 进阶（原理剖析，需 C 与 OS 基础）。

### 导航与索引（5 页）
- [[mysql/README]] — 总览导航、**阅读门槛声明**、⭐ **四个版本基线事实**（8.0 EOL / 日历版本号 / LTS 不能跳级 / 治理风波）、目录结构
- [[mysql/学习路线图]] — 三阶段路径、阶段门槛、**毕业自测清单**
- [[mysql/术语与关键字手册]] — ★ **专有名词总索引**：架构进程 / 存储引擎 / 事务并发 / 索引优化 / 复制 / 备份运维 / 类型函数 / 配置参数 / 与其他协议关系
- [[mysql/函数手册]] — ★ **函数总索引**：十类函数 + 「查不到函数时的四步法」+ 按场景找
- [[mysql/速查索引]] — 客户端命令 / DDL·DML / ⭐ **MySQL↔PG 语法差异速查** / 诊断命令 / 配置变量 / 备份命令 / **15 条高频陷阱速记**

### 01 入门阶段 · 基础教程（18 页）
> 零基础起点，不要求任何前置知识。知识有强线性依赖，须按章节顺序推进。
> 入口：[[mysql/01-入门阶段-基础教程/00-章节目录|章节目录]]（含练习数据集）

**第00章 学前准备（5 篇）**
- [[mysql/01-入门阶段-基础教程/第00章-学前准备/0.1-数据库与SQL是什么]] — 四个问题、库表行列、**声明式与优化器**、三条书写规则
- [[mysql/01-入门阶段-基础教程/第00章-学前准备/0.2-MySQL架构与客户端]] — **mysqld 三层**、存储引擎插件化、动态/静态参数、⭐ **`sql_mode` 生产必开**
- [[mysql/01-入门阶段-基础教程/第00章-学前准备/0.3-MySQL与PostgreSQL的差异]] — ⭐ **根本差异是 MVCC 实现**、语法对照、⚠️ **行为差异更危险**、照搬 PG 的五个坑、选型判断
- [[mysql/01-入门阶段-基础教程/第00章-学前准备/0.4-安装与连接]] — Docker 与原生、⭐ **初始化必做四件事**、数据目录、Python/SQLAlchemy 连接
- [[mysql/01-入门阶段-基础教程/第00章-学前准备/0.5-看懂报错信息]] — 报错三段、**SQLSTATE 前缀分类**、固定排错路径、12 类常见报错

**第01章 环境与数据类型（8 篇，实际落盘）** ⚠️ 2026-10-17 修复：原 3 条链接指向早期规划页名（页面从未创建），已重定向到实际承载内容的页面
- [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.1-数据类型全景]] · [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.2-数值类型]] — ⭐ **金额必用 DECIMAL**、`UNSIGNED` 溢出、auto_increment 行为、选型五问
- [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.3-字符串类型]] · [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.4-日期时间类型]] — ⭐ **`utf8mb4` 的坑**、五层级、**时区陷阱**、日期函数（`%i` 不是 `%M`）
- [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.5-JSON与特殊类型]] · [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.6-字符集排序规则]] · [[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/1.7-数据库与表初始化实操]] — ⭐ **JSON 三大陷阱**、`JSON_TABLE`、生成列、⭐ **collation 不一致导致 JOIN 索引失效**、初始化四步
- 本章入口：[[mysql/01-入门阶段-基础教程/第01章-环境与数据类型/00-本章导读与预备知识]]

**第02章 建表与约束（3 篇）**
- [[mysql/01-入门阶段-基础教程/第02章-建表与约束/2.1-DDL建表与列定义]] — 生产模板、默认值陷阱、⭐ **`ALTER` 必加 `ALGORITHM=INPLACE, LOCK=NONE`**、`INSTANT`
- [[mysql/01-入门阶段-基础教程/第02章-建表与约束/2.2-约束体系与数据完整性]] — ⭐ **外键取舍**、⚠️ **外键列不会自动建索引**（与 PG 相反）、⚠️ **`REPLACE` 危险**、软删除与唯一约束冲突
- [[mysql/01-入门阶段-基础教程/第02章-建表与约束/2.3-表设计规范与反范式]] — ⭐ **一张表只描述一种事物**、正确的冗余、软删除三后果、**为分库分表做的设计准备**

**第03章 查询与 CRUD（4 篇）**
- [[mysql/01-入门阶段-基础教程/第03章-查询与CRUD/3.1-INSERT与REPLACE]] — 三种冲突策略、⚠️ **`REPLACE` 六个风险**、原子计数、`LOAD DATA INFILE`、`LAST_INSERT_ID`
- [[mysql/01-入门阶段-基础教程/第03章-查询与CRUD/3.2-WHERE条件与函数]] — ⭐ **NULL 三值逻辑**、⚠️ **函数让索引失效**、日期函数、⚠️ **`GROUP_CONCAT` 截断**
- [[mysql/01-入门阶段-基础教程/第03章-查询与CRUD/3.3-ORDER-BY排序与分页]] — ⚠️ **NULL 排序与 PG 相反**、⚠️ **`limit n,m` 语义**、⭐ **深分页与键集分页**
- [[mysql/01-入门阶段-基础教程/第03章-查询与CRUD/3.4-JOIN连接查询]] — 五种 JOIN、⚠️ **`ON` vs `WHERE`**、⚠️ **COUNT(*) vs COUNT(列)**、隐式 JOIN 膨胀、三个性能坑

**第04章 事务与权限（2 篇）**
- [[mysql/01-入门阶段-基础教程/第04章-事务与权限/4.1-事务与ACID]] — ACID 实现、⭐ **MySQL RR 额外用间隙锁防幻读**、⚠️ **DDL 隐式提交**、⭐ **长事务四大危害**
- [[mysql/01-入门阶段-基础教程/第04章-事务与权限/4.2-角色权限与账号管理]] — ⭐ **账号 host 的坑**、最小权限原则、⚠️ **`USER()` 可伪造**、认证插件与 `Public Key Retrieval`

### 02 中段阶段 · 专题精讲（12 页 + 导航）
> 平行专题可跳读；前置：入门阶段已学完。每篇开头标注「什么时候该读」。
> 入口：[[mysql/02-中段阶段-专题精讲/00-专题导航|专题导航]]（含「按目标选路径」表）

- [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引类型与选型]] — 四种索引、聚簇 vs 二级、⭐ **最左前缀**、复合索引设计、三种特殊索引、⭐ **不可见索引在线删**
- [[mysql/02-中段阶段-专题精讲/索引与查询优化/EXPLAIN解读]] — 逐字段、type 优劣、⭐ **Extra 关键字诊断**、`EXPLAIN ANALYZE`、四个实战案例
- [[mysql/02-中段阶段-专题精讲/索引与查询优化/覆盖索引与回表优化]] — 回表代价、⭐ **覆盖索引**、ICP、⚠️ **`Using where` + 高 rows 的危险**、四个优化层次
- [[mysql/02-中段阶段-专题精讲/索引与查询优化/索引诊断与慢查询优化]] — 三个数据源、⭐ **删索引正确流程**、⭐ **优化器估错的解法**、`optimizer_trace`、六个实战案例
- [[mysql/02-中段阶段-专题精讲/索引与查询优化/SQL优化实战-改写技巧与真实案例]] — ⭐ **优化四层定位**（索引/改写/结构/架构）、**8 个改写手法**（深分页 / COUNT / OR / 子查询 / 批量写 / JOIN / 排序 / 提前终止）、**6 个线上案例**（耗时前后对比 + 根因拆解）
- [[mysql/02-中段阶段-专题精讲/InnoDB存储引擎/专题概览]] — 页结构、⭐ **聚簇索引叶子存整行**、⭐ **Buffer Pool 与 LRU 两段式**、Change Buffer、⚠️ **膨胀治理**
- [[mysql/02-中段阶段-专题精讲/存储引擎与选型/为什么只有InnoDB支持事务]] — ⭐ **事务 = 四套机制咬合**（undo / 行锁+MVCC / redo / 约束）、⚠️ **ACID 被下放到引擎层**（PG 在核心层）、MyISAM 只有 `.MYD`/`.MYI` 无日志、MEMORY 不承诺持久、⚠️ **NDB 是「内存型 + 事务」的反例**
- [[mysql/02-中段阶段-专题精讲/存储引擎与选型/引擎全景与迁移实操]] — 9.7 全引擎对照表（含 CSV/ARCHIVE/BLACKHOLE/MERGE/FEDERATED/EXAMPLE）、⚠️ **「8.0 已移除 MyISAM」是谣言**、`SHOW ENGINES` 盘点、⚠️ **混引擎事务不原子**、⚠️ **9.7 起 MyISAM 无分区**
- [[mysql/02-中段阶段-专题精讲/事务与并发控制/长事务与undo膨胀]] — 两种形态、四大危害、⭐ **`trx_rseg_history_len` 监控**、⭐ **找出空闲但开着事务的连接**、KILL 的代价
- [[mysql/02-中段阶段-专题精讲/主从复制与高可用/复制原理与拓扑]] — 复制链路、⭐ **为什么必须 ROW**、GTID、⚠️⚠️ **`Seconds_Behind_Source` 会骗人**、延迟八个根因、MGR
- [[mysql/02-中段阶段-专题精讲/备份恢复与安全/备份体系与恢复演练]] — 逻辑 vs 物理、PITR、⭐ **三个必须为 1 的参数**、⭐ **恢复演练**、安全加固、版本升级路径
- [[mysql/02-中段阶段-专题精讲/工具与可观测性/工具链总览]] — ⭐ **`sys` schema 优先**、performance_schema 核心表、慢查询日志、`Handler_*` 读法、连接管理、⭐ **排查顺序**
- [[mysql/02-中段阶段-专题精讲/大表与在线DDL/在线DDL方案对比]] — 大表阈值、⚠️ **`gh-ost` 优于 `pt-osc`**（不动 binlog）、⭐ **分区表 DROP PARTITION**、JSON 多值建模

### 03 进阶阶段 · 深度剖析（6 页）
> 需要 C 语言与操作系统基础。⭐ **核心方法：带问题读 + 配实验验证**。
> 入口：[[mysql/03-进阶阶段-深度剖析/00-剖析方法论|剖析方法论]]（含模块依赖图与实验清单）

- [[mysql/03-进阶阶段-深度剖析/00-剖析方法论]] — 五模块依赖图、**阅读顺序建议**、⭐ **带问题读的三种切入方式**、实验三原则、**MySQL vs PG 实现分歧**
- [[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-undo日志与MVCC]] — ⭐⭐ **undo 版本链**、⭐⭐ **ReadView 四字段与可见性五步判定**、⭐ **RC vs RR 的唯一区别**、快照读 vs 当前读、purge 机制
- [[mysql/03-进阶阶段-深度剖析/事务与锁内核/原理剖析-锁体系与死锁]] — ⭐ **Next-Key Lock**、⭐ **RR 怎么用间隙锁防幻读**、意向锁、⚠️ **MDL 冻结链条**、死锁检测与重试
- [[mysql/03-进阶阶段-深度剖析/日志与复制内核/原理剖析-redo日志与WAL]] — ⭐ **WAL 原理**、三个 LSN、checkpoint、⭐⭐ **redo vs binlog 五个区别**、⭐⭐⭐ **「为什么 binlog 独立于 redo」三层答案**
- [[mysql/03-进阶阶段-深度剖析/优化器内核/原理剖析-优化器四阶段与成本模型]] — ⭐ **四阶段**、semi-join、谓词下推、成本模型、⭐ **直方图修基数估算**、启发式行为
- [[mysql/03-进阶阶段-深度剖析/生态与扩展/生态全景与商业版边界]] — 插件体系、⭐ **Community/Enterprise 边界在移动**、治理现状、⚠️ **国产数据库三类别**、云托管
---

## java — Java 学习知识库（语言基础 + JVM + 并发 + Spring Boot + 面试题库）

- [[java/README]] — 导航、门槛声明、⭐ **版本基线**（Java 25 LTS / Spring Boot 4.1 / ⚠️ 4.1 不支持 Java 27）
- [[java/术语与关键字手册]] — ⭐ 14 大类名词兜底索引（类型系统 / OOP / 异常 / 集合 / 泛型 / 并发 / JVM / Spring）

### 入门阶段（5 章 · 17 篇）
> 零基础可读。会 Python 的先读 0.3 篇「与 Python 的差异」。

- [[java/01-入门阶段-基础教程/00-章节目录]] — 阶段目录、门槛声明、各章「读完能做」

**第 00 章 · 学前准备（3 篇）**
- [[java/01-入门阶段-基础教程/第00章-学前准备/0.1-Java是什么与生态定位]] — ⭐ 字节码 → JVM 三段式、2026 生态位、与 Python 的分工
- [[java/01-入门阶段-基础教程/第00章-学前准备/0.2-JDK安装与最小语法]] — JDK 25 安装、javac/java、最小语法集、Maven 骨架
- [[java/01-入门阶段-基础教程/第00章-学前准备/0.3-Java与Python的差异]] — ⭐ 静态类型 vs 动态类型、五个必踩坑（equals / 类型 / get / 受检异常 / BigDecimal）

**第 01 章 · 核心语法（4 篇）**
- [[java/01-入门阶段-基础教程/第01章-核心语法/1.1-变量与数据类型]] — 变量三分类与作用域、8 种基本类型取值表、三类类型转换、⭐ **编译期常量**、`var` 的四条硬限制
- [[java/01-入门阶段-基础教程/第01章-核心语法/1.2-运算符与流程控制]] — ⭐ **短路求值**、位运算与补码、三种循环、`switch` 表达式与模式匹配（Java 21+）
- [[java/01-入门阶段-基础教程/第01章-核心语法/1.3-方法与参数传递]] — ⭐⭐ **值传递机制**（改内容生效、换引用不生效）、重载与重写规则、可变参数、递归
- [[java/01-入门阶段-基础教程/第01章-核心语法/1.4-字符串与包装类]] — String 不可变三理由、⭐ **常量池与 `intern()`**、`==` vs `equals`、⭐ **`BigDecimal` 算钱**

**第 02 章 · 面向对象（4 篇）**
- [[java/01-入门阶段-基础教程/第02章-面向对象/2.1-类与对象基础]] — 构造器与构造器链、⭐ **自定义构造器后不再有默认无参**、`this`/`static`、初始化块顺序
- [[java/01-入门阶段-基础教程/第02章-面向对象/2.2-封装继承与多态]] — 四个访问修饰符、方法重写、⭐ **运行时多态**（方法看运行时类型、字段看声明类型）、组合优于继承
- [[java/01-入门阶段-基础教程/第02章-面向对象/2.3-抽象类与接口]] — 抽象类 vs 接口对照、⭐ **default 方法与钻石问题三规则**、`sealed` + 模式匹配 switch
- [[java/01-入门阶段-基础教程/第02章-面向对象/2.4-内部类与Lambda表达式]] — 四种内部类、函数式接口与 SAM、Lambda 与方法引用四种形式、⭐ **`invokedynamic` 底层**

**第 03 章 · 异常处理（2 篇）**
- [[java/01-入门阶段-基础教程/第03章-异常处理/3.1-异常体系与分类]] — ⭐ Throwable 继承树、⭐ **受检 vs 非受检**判定与设计意图、常见异常速查、异常沿调用栈传播
- [[java/01-入门阶段-基础教程/第03章-异常处理/3.2-捕获处理与自定义异常]] — `catch` 顺序规则、`throw` vs `throws`、⭐ **`try-with-resources` 与异常抑制**、自定义异常与 cause 链

**第 04 章 · 集合框架（4 篇）**
- [[java/01-入门阶段-基础教程/第04章-集合框架/4.1-集合接口总览与选型]] — Collection/Map 体系、五项语义契约、⭐ **选型决策表**、不可变集合
- [[java/01-入门阶段-基础教程/第04章-集合框架/4.2-List集合详解]] — ArrayList 数组结构与 ⭐ **1.5 倍扩容**、`remove` 重载陷阱、⭐ **fail-fast 真实触发条件**
- [[java/01-入门阶段-基础教程/第04章-集合框架/4.3-Set与Map集合详解]] — Set/Map 三兄弟的顺序差异、`TreeMap` 排序与范围查询、`computeIfAbsent`/`merge`
- [[java/01-入门阶段-基础教程/第04章-集合框架/4.4-HashMap底层原理]] — ⭐ **hash 扰动**、桶结构、**put 五步**、⭐ **2 倍扩容重分配**、⭐ **equals/hashCode 契约**

### 中段阶段（3 专题 · 12 篇）

- [[java/02-中段阶段-专题精讲/00-专题导航]] — 专题导航与七条通用原则

**JVM（4 篇，面试主战场）**
- [[java/02-中段阶段-专题精讲/JVM/JVM内存区域与对象布局]] — ⭐ 运行时数据区 5 块、对象创建 5 步、Mark Word、JIT 与逃逸分析
- [[java/02-中段阶段-专题精讲/JVM/垃圾回收与调优]] — ⭐ 可达性分析与 GC Roots、四种引用、分代模型、G1/ZGC、调优四步
- [[java/02-中段阶段-专题精讲/JVM/类加载机制]] — ⭐ 七阶段、准备 vs 初始化、双亲委派、三个打破场景
- [[java/02-中段阶段-专题精讲/JVM/排查工具与故障案例]] — jps/jstat/jmap/jstack、Arthas、四类故障固定路径（含脱敏案例）

**并发编程（5 篇，新增）**
- [[java/02-中段阶段-专题精讲/并发编程/01-线程模型与生命周期]] — 进程 vs 线程、三种创建方式、⭐ **6 种线程状态与转换图**、`sleep`/`join`/`interrupt`、⭐ **ThreadLocal 内存泄漏**
- [[java/02-中段阶段-专题精讲/并发编程/02-线程安全与Java内存模型]] — ⭐ **JMM 主内存/工作内存**、并发三大问题、⭐ **volatile 的语义与局限**、**happens-before 七条规则**、DCL 半初始化
- [[java/02-中段阶段-专题精讲/并发编程/03-锁机制与原子操作]] — ⭐ **synchronized 锁的是对象**、锁升级四阶段（⚠️ 偏向锁 JDK 15 废弃）、ReentrantLock/Condition、⭐ **CAS 与 AQS**、死锁四条件
- [[java/02-中段阶段-专题精讲/并发编程/04-线程池与任务调度]] — ⭐ **ThreadPoolExecutor 七参数**、⭐ **「先入队、再扩容」执行流程**、队列选型与 OOM 坑、四种拒绝策略、⭐ **`@Async` 四个坑**、优雅停机
- [[java/02-中段阶段-专题精讲/并发编程/05-并发容器与虚拟线程]] — ⭐ **ConcurrentHashMap（CAS + synchronized 锁桶头）**、BlockingQueue 家族、同步工具类、⭐ **虚拟线程与 pinning（JEP 491）**、三路选型对比

**Spring Boot（3 篇）**
- [[java/02-中段阶段-专题精讲/SpringBoot/启动原理与自动配置]] — ⭐ refresh 关键三步、自动配置四步链路、条件注解、`--debug` 排查
- [[java/02-中段阶段-专题精讲/SpringBoot/IoC与Bean生命周期]] — ⭐ Bean 生命周期 8 步、三级缓存、AOP 代理、`@Transactional` 四种失效
- [[java/02-中段阶段-专题精讲/SpringBoot/Web开发与分层实践]] — 三层边界、DTO/VO/Entity、全局异常、事务与连接池、容器部署

### java/interview — Java 面试八股题库（7 篇 · 70 题 · 190 条追问）⭐
> 入口：[[java/interview/README|Java 面试题库]]（版本口径 + 文件清单 + 使用方式 + 答题总原则）
> ⚠️ 与 [[interview/README|跨主题面试知识库]] 的分工：那套是跨主题通用与项目话术，本套是 Java 岗位专属八股。

- [[java/interview/01-核心语法与数据类型]] — 类型系统、自动装箱与 Integer 缓存、类型转换、⭐ **String 不可变与常量池**、`==` vs `equals`、修饰符
- [[java/interview/02-面向对象与设计模式]] — 三大特性实现原理、重载 vs 重写、抽象类 vs 接口、⭐ **多态底层与 instanceof**、equals/hashCode 契约、六种设计模式、SOLID
- [[java/interview/03-集合框架]] — 三大接口选型、ArrayList 扩容、⭐ **HashMap 原理与扰动**、⭐ **树化与扩容**、⭐ **fail-fast**、不可变集合
- [[java/interview/04-异常处理与泛型]] — 异常体系树、⭐ **受检 vs 非受检设计意图**、finally 陷阱、try-with-resources、自定义异常、⭐ **类型擦除与 PECS**
- [[java/interview/05-JVM内存与垃圾回收]] — 运行时数据区、对象布局与 Mark Word、GC 算法与收集器选型、类加载、⭐ **OOM 排查思路**、调优与诊断工具
- [[java/interview/06-并发编程]] — 线程状态、⭐ **线程池七参数与执行流程**、拒绝策略、JMM 与 volatile、⭐ **锁升级**、CAS 与 ABA、⭐ **死锁排查**、虚拟线程
- [[java/interview/07-Spring与工程实践]] — IoC/DI 三种方式、Bean 生命周期、循环依赖、⭐ **AOP 代理**、⭐ **`@Transactional` 完整失效清单**、自动配置原理、⭐ **线上排查思路**

## typescript — TypeScript 知识库（语言 + 工程化 + LangChain.js/LangGraph.js）

> 入口：[[typescript/README|TypeScript 知识库首页]] · ⭐ **三条主线**：语言本身 → AI 生态连接 → 面试八股。

- [[typescript/README]] — 导航、门槛声明、⭐ **版本基线**（TS 7.0 Go 原生编译器 / 6.0 默认值变更 / LangChain.js v1）
- [[typescript/术语与关键字手册]] — ⭐ 8 大类名词兜底索引（类型系统 / 核心语法 / 模块系统 / 工程化 / Node / AI 生态）

### 入门阶段（3 章 · 11 篇）
> 零基础可读。会 Python 的先读 0.3 篇「与 Python 的差异」。

- [[typescript/01-入门阶段-基础教程/00-章节目录]] — 阶段目录、门槛声明、各章「读完能做」

**第 00 章 · 学前准备（3 篇）**
- [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.1-TypeScript是什么与生态定位]] — ⭐ **编译期超集的本质**、TS 类型在运行时不存在、2026 生态版图、什么时候该用
- [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.2-环境搭建与第一个程序]] — Node.js/npm、tsc、tsconfig 最小配置、⭐ **三种运行 TS 的方式**（tsc/tsx/ts-node）
- [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]] — 静态强类型、编译模型、模块、包管理逐项对照；⚠️ **五个最容易照搬踩坑的点**

**第 01 章 · 核心语法（4 篇）**
- [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.1-变量与基本类型]] — `const`/`let`/`var`、8 种基本类型、类型注解、⭐ **`undefined`/`null`/`NaN` 三者区别**、TDZ、`??` vs `||`
- [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.2-函数与参数]] — 三种函数写法、函数类型、⭐ **`this` 绑定差异**、TS 重载写法
- [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.3-对象与数组]] — 可选/只读属性、索引签名、元组、解构、⭐ **展开是浅拷贝**、`for...in` vs `for...of`
- [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.4-类与接口]] — 访问修饰符（⭐ **`private` 只是编译期约束**）、`implements`/`extends`、⭐ **结构化类型**

**第 02 章 · 类型系统（4 篇）**
- [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]] — 何时写注解、上下文类型、⭐ **`any` vs `unknown`**、⭐ **类型断言 `as` vs `satisfies`**
- [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.2-联合类型与字面量类型]] — 联合 `|`、字面量类型、⭐ **判别联合**、⭐ **`enum` 的取舍**（会生成运行时代码）
- [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]] — 泛型函数/类、⭐ **`keyof`/`typeof`**、⭐ **条件类型与映射类型**、映射类型会移除 readonly
- [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]] — `typeof`/`instanceof`/`in` 收窄、⭐ **用户自定义类型守卫**、断言函数、⭐ **`never` 穷尽检查**

### 中段阶段（2 专题 · 6 篇）

- [[typescript/02-中段阶段-专题精讲/00-专题导航]] — 专题导航与七条通用原则

**工程化与工具链（3 篇）**
- [[typescript/02-中段阶段-专题精讲/工程化与工具链/1-编译流程与tsconfig]] — ⭐ **类型检查与转译是两件事**、tsconfig 关键字段、⭐ **`strict` 家族 7 个子开关**、⚠️ **TS 6.0 默认值变更**（`types: []` 最坑）、TS 7.0 废弃项
- [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]] — ESM vs CJS、`type: module`、条件导出、动态 `import()`、顶层 await、⚠️ **相对导入必须带扩展名**
- [[typescript/02-中段阶段-专题精讲/工程化与工具链/3-构建工具与包管理]] — npm/pnpm/yarn/bun、⭐ **tsc vs esbuild vs vite vs tsup**、`tsx`、声明文件 `.d.ts`、发布 TS 库

**Node 生态与异步（3 篇）**
- [[typescript/02-中段阶段-专题精讲/Node生态与异步/1-Node运行时与HTTP服务]] — ⭐ **事件循环与 libuv**、⚠️ **CPU 密集会卡死事件循环**、`node:http` 起服务、Express vs Fastify、优雅停机
- [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] — Promise 三态、`async/await`、⭐ **四个组合子语义差异**、⭐ **串行 vs 并行**、⚠️ **未处理 rejection 终止进程**、背压
- [[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]] — Vitest、⭐ **`toBe` vs `toEqual` vs `toStrictEqual`**、mock 与 spy、错误处理分层、`unknown` 收窄

### ⭐ 03 生态连接 · LangChain.js 与 LangGraph.js（7 篇）
> ⭐ **本库核心**：把 TS 的类型能力接到 AI Agent 项目上。入口：[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|生态连接专题导航]]

- [[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]] — ⭐ **LangChain.js 是一等公民实现**、三选一决策表、⭐ **双栈协作三种模式**、Zod 路线优势、版本配套策略
- [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]] — ⭐ `tool()` + Zod、`createAgent` 全参数、模型接入三写法、结构化输出、流式、⭐ **6 个中间件钩子**、`contextSchema`、20 条坑清单
- [[typescript/03-生态连接-LangChain与LangGraph/3-从TypeScript到LangChain-js实战迁移]] — ⭐ 端到端天气 Agent 项目、**类型贯通链路**、Python 概念映射表、错误三等级、Vitest 测试策略、五个跨语言坑
- [[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]] — ⭐ **`StateSchema` 四类通道**、⭐ **reducer 与并发正确性**、节点/边/条件边、多 schema、`Command`、`Send`、`ToolNode`、13 条坑清单
- [[typescript/03-生态连接-LangChain与LangGraph/5-LangGraph-js持久化与人工介入]] — checkpointer 三档后端、`thread_id` 会话管理、⭐ **HITL 完整链路**、流式与前端、⚠️ **`subgraphs: true` 必需**
- [[typescript/03-生态连接-LangChain与LangGraph/6-Python与TypeScript双栈对照]] — ⭐ **完整 API 对照表**、命名转换规律、Pydantic vs Zod、⭐ **生态差异的实话**、别照搬的 5 个 Python 习惯、面试答法

### typescript/interview — TypeScript 面试八股题库（4 篇 · 40 题 · 82 条追问）
> 入口：[[typescript/interview/README|TypeScript 面试题库]]（版本口径 + 文件清单 + 使用方式 + 答题原则）

- [[typescript/interview/01-核心语法与类型系统]] — 类型系统本质、`any`/`unknown`、`as` vs `satisfies`、联合与字面量、`never`、可选链、结构化类型
- [[typescript/interview/02-泛型与类型体操]] — 泛型约束、`keyof`/`typeof`、条件类型、映射类型、模板字面量类型、`infer`、变型
- [[typescript/interview/03-工程化与工具链]] — tsconfig、`strict` 家族、模块解析、声明文件、构建工具选型、TS 6/7 迁移
- [[typescript/interview/04-异步编程与Node]] — 事件循环、微任务/宏任务、Promise 组合子、错误处理、Node 单线程模型、流与背压

## interview — 面试与求职

> 入口：[[interview/README|面试知识库首页]] · 定位：**把已有 672 页内容转化成「能说出口的东西」**。
> ⚠️ 本目录和其他主题性质不同——不教新知识，是 output 层。
> ⚠️ **开工假设**：三个项目均可用于简历；目标岗位 Agent 工程为主、兼容后端。⚠️ **量化数据一律不编造，留 `【待补充】`。**

### 使用方式（⚠️ 三遍法，不是「读完」）

```
第一遍：通读，标出「看过但说不出口」的条目
第二遍：只看标出的条目，合上书复述，录音
第三遍：只练「高频追问」小节，练到不被问倒
```

⭐ **判断标准：能脱稿讲清 + 接得住追问 = 会了。看懂了但说不出口 = 不会。**

### 五个模块的优先级（⚠️ 不要平均用力）

| 模块 | 优先级 | 理由 |
|------|--------|------|
| **04 项目经验** | ⭐⭐⭐ | ⭐ **面试 60% 时间在聊项目** |
| **02 Agent 专项** | ⭐⭐⭐ | 岗位核心 + 有真实项目背书 |
| **03 RAG 专项** | ⭐⭐ | 第二强项，RAG 是热点 |
| **01 八股基础** | ⭐⭐ | 逃不掉，但可速成 |
| **05 系统设计** | ⭐ | 加分项，不是门槛 |

### 01 八股基础（3 篇）
- [[interview/01-八股基础/Python八股]] — GIL/asyncio、装饰器、生成器、GC、默认参数坑、LEGB、并发选型、10 条自测
- [[interview/01-八股基础/数据库八股]] — B+树/聚簇与二级索引/最左前缀、⭐ **ReadView 可见性五步判定**、Next-Key Lock、redo vs binlog、索引失效、深分页、Redis 快的原因/持久化/三大问题/分布式锁、⭐ **MySQL vs PG 的 MVCC 差异**
- [[interview/01-八股基础/网络与操作系统八股]] — HTTPS 握手、HTTP/2/3、401 vs 403、三次握手/四次挥手/TIME_WAIT、⭐ **流量控制 vs 拥塞控制**、死锁四条件、`select` vs `epoll`、零拷贝

### 02 Agent 专项（4 篇）⭐ 主战场
- [[interview/02-Agent专项/ReAct与Agent循环]] — 标准答法 + 四种变体 + `create_agent` 内部四步 + ⭐ **生产四个坑**（无限循环/上下文爆炸/工具失败/可观测性）
- ⚠️ **LangGraph 相关已迁出** → [[langgraph/interview/README|LangGraph 面试题库]]（原「LangGraph调度与状态」页现为 [[langgraph/interview/00-面试总纲与答题框架]]）
- [[interview/02-Agent专项/中间件与Checkpointer]] — 三类钩子、`wrap_tool_call` 为什么是「包裹」、SummarizationMiddleware 实践、⭐ **Checkpointer vs Store 四维度对比**、长短记忆三层设计
- [[interview/02-Agent专项/Skills渐进加载]] — ⭐ **三层加载模型**、Skills vs MCP vs RAG 三者区别、Skill 粒度决策、你项目的 list/load 实践
- [[interview/02-Agent专项/高频追问与答法]] — 原理类/工程类/⭐ **场景设计题**（「设计一个能查数据库的 Agent」六步）、反问环节

### 03 RAG 专项（4 篇）
- [[interview/03-RAG专项/RAG全流程问答]] — 七步流程、混合检索必要性、重排范式、⭐ **评估为什么必须分两层**、RAG 五个局限、你项目的实践模板
- [[interview/03-RAG专项/分块与检索策略]] — 三种分块选型、块大小推理、⭐ **四种特殊内容处理（表格/代码/列表/图片）**、RRF 公式、查询侧优化、上下文构建
- [[interview/03-RAG专项/评估体系ragas]] — 两层指标体系、⭐ **「高 recall 低 faithfulness」的诊断价值**、Golden Set 三来源、LLM-as-Judge 降噪、三个常见误区
- [[interview/03-RAG专项/高频追问与答法]] — ⭐ **RAG vs 微调对比表**、召回不理想五步排查、降幻觉四层、引用编号校验、⭐ **「一周内提升效果」的优先级排序**

### 04 项目经验（3 篇）⚠️ 含量最高，需亲自补充
- [[interview/04-项目经验/项目一-终端多智能体系统]] — 简历三版本、⭐ **STAR 重构**、三个设计决策、8 个预判追问、⭐ **常见自杀式回答清单**
- [[interview/04-项目经验/项目二-RAG检索系统]] — 主打「解析+评估」两头、docling 元素级解析、表格摘要双轨、ragas 分层评估、⭐ **「高 recall 低 faithfulness」实战案例位**
- [[interview/04-项目经验/项目三-个人知识库]] — ⭐ **三层架构 + 写作基线 + 三项自动化校验**、断链校验四类误报、⭐ **「和 RAG 是同一个问题的两种解法」的串联话术**

### 05 系统设计（4 篇）
- [[interview/05-系统设计/系统设计方法论]] — ⭐ **六步框架**（澄清→估算→接口→架构→权衡→运维）、提问清单、规模估算速查、trade-off 表达模板
- [[interview/05-系统设计/高频题-多智能体编排]] — 六步走一遍、三个深入点、四个 trade-off 表、⭐ **加试题「跨组织」的 A2A 接入**
- [[interview/05-系统设计/高频题-RAG架构设计]] — 读写两条链路、⭐ **权限过滤在检索层 vs 检索后**、增量更新两坑、拒答机制
- [[interview/05-系统设计/高频题-高并发后端]] — 缓存三大问题、缓存一致性、分布式锁、幂等、防超卖、⭐ **「Agent 系统就是高并发 IO 密集服务」的串联**
