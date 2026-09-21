# LLM 驱动的个人学习知识库

> 一个由 LLM 增量式构建和维护的持久化 wiki，专注于 **AI Agent 工程化**的系统性学习：LangChain / LangGraph / DeepAgents / RAG / Docker 沙箱 / FastAPI / PostgreSQL。

本知识库基于 `LLM wiki.md` 的方法论构建：LLM 读取来源后**编译**进一个持久化的 wiki——交叉引用预先建立，矛盾预先标记，综合分析持续更新。与每次查询从头检索的 RAG 不同，wiki 是一个**不断累积的工件**。

---

## 知识库规模

| 层 | 数量 | 说明 |
|---|---|---|
| `wiki/` | 533 页 | LLM 生成维护的知识页面，按 12 个主题域组织 |
| `raw/` | 20 篇 | 不可变的来源文档（菜鸟教程 AI Agent 系列、网页剪藏等） |
| 根文件 | 4 个 | `index.md`（内容索引）、`log.md`（操作日志）、`SCHEMA.md`（规范）、`README.md`（本文件） |

各主题域页面数：langchain 143 · postgresql 74 · python 73 · rag 58 · agent 46 · langgraph 41 · deepagents 29 · tools 26 · fastapi 18 · docker 12 · meta 7 · sqlmodel 6

---

## 主题域地图

### 🤖 Agent 核心（主学习线）

| 主题域 | 页数 | 内容与组织方式 |
|---|---|---|
| **agent/** | 46 | AI Agent 通识：`patterns/`（CoT、ReAct、PlanAndExecute、Reflection、ToT、Reflexion、多智能体协作七大模式）、`harness/`（三层架构、六种架构、沙箱安全隔离、Function Call）、`memory/`（记忆系统）、`skills/`、`prompt/`（提示词与上下文工程）、`RAG/`（Agent 视角的精简版 RAG）、`reinforcement/`（Agentic RL） |
| **langchain/** | 143 | LangChain v2 系统学习：`create_agent`、Agent 循环、中间件体系（20+ 个内置中间件逐一详解）、记忆（Checkpointer/Store/压缩）、流式传输、多 Agent、`modules/`（16 个核心模块 API 详解，含 MCP 协议） |
| **langgraph/** | 41 | LangGraph 框架：StateGraph 构建与编译、状态语义与 Reducer、动态扇入/扇出、Checkpoint 持久化、人机交互（interrupt）、多智能体模式、流式模式、`tutorial/` 实战教程 |
| **deepagents/** | 29 | DeepAgents v0.5+：默认中间件栈、`create_deep_agent` 参数、子代理、Skills 系统、沙箱后端、生产部署与源码架构 |
| **docker/** | 12 | Docker 沙箱：核心类、安全加固、多租户隔离、监控日志、性能优化、生产部署清单 |
| **fastapi/** | 18 | FastAPI 全链路：从路径操作、依赖注入到 SSE 流式响应、WebSocket、Agent 作业集成 |

### 📚 RAG 体系

| 主题域 | 页数 | 内容与组织方式 |
|---|---|---|
| **rag/** | 58 | **14 步全流程学习路径**（从数据接入到范式前沿），每步一个子目录：多源数据接入与解析（PDF/Word/HTML/表格/图片）→ 数据清洗 → 智能分块（10 页）→ 混合索引（Embedding/ANN）→ 查询转换 → 混合检索（BM25/倒排索引）→ 元数据过滤 → 重排序 → 上下文构建 → LLM 生成 → 安全合规 → 评估监控（RAGAS）→ 工程横切（多租户/大并发）→ 范式前沿（Agentic RAG/GraphRAG）。入口：[[rag/RAG_学习路径导航]] |

### 🗄️ 数据与基础

| 主题域 | 页数 | 内容与组织方式 |
|---|---|---|
| **postgresql/** | 74 | **三阶段学习体系**（版本基线 PG 18.6）：入门 27 页（顺序章节式，含零基础「第 00 章」）、中段 22 页（专题精讲式：窗口函数/CTE/JSONB/索引/执行计划/分区/备份恢复）、进阶 20 页（深度剖析式：MVCC 内核/页结构/BTree/WAL/锁/SSI/流复制）。配有三大总索引：[术语与关键字手册]、[函数手册]、[速查索引]。入口：[[postgresql/README]] |
| **python/** | 73 | 基础（数值/字符串/控制流/异常/正则）、数据结构、进阶（类型提示/asyncio/装饰器/生成器/Pydantic）、OOP、pytest |
| **sqlmodel/** | 6 | SQLModel ORM：模型定义、引擎与 Session、CRUD、关系关联 |
| **tools/** | 26 | 向量数据库（Chroma 5 页详解 + FAISS/Milvus/pgvector/Pinecone/Qdrant/Weaviate 概览）及 Git、Gradio、Streamlit、Obsidian 插件等工具 |
| **meta/** | 7 | 知识库自身的方法论：LLM Wiki 方法、三层架构、RAG vs Wiki 对比。入口：[[meta/Welcome]] |

---

## 目录结构

```
my_agend_study_knowledge/
├── raw/            # 来源文档（不可变，LLM 只读不改）
├── wiki/           # LLM 生成维护的 wiki 页面（按主题域分目录）
│   ├── agent/      # AI Agent 通识
│   ├── langchain/  # LangChain 框架
│   ├── langgraph/  # LangGraph 框架
│   ├── deepagents/ # DeepAgents 框架
│   ├── rag/        # RAG 14 步全流程
│   ├── docker/     # Docker 沙箱
│   ├── fastapi/    # FastAPI
│   ├── postgresql/ # PostgreSQL 三阶段
│   ├── python/     # Python 基础与进阶
│   ├── sqlmodel/   # SQLModel ORM
│   ├── tools/      # 工具集
│   └── meta/       # 知识库方法论
├── SCHEMA.md       # wiki 结构定义与 LLM 操作规范（人机协议）
├── index.md        # 全部 wiki 页面的分类索引（每页带一行摘要）
├── log.md          # 操作日志（按时间倒序追加）
└── README.md       # 本文件
```

---

## 快速开始

### 四个基本操作

1. **摄入来源**：将文档放入 `raw/`，然后告诉 LLM「请摄入 [文件名]」——LLM 阅读并整合进 wiki
2. **提问**：对知识库提问——LLM 先读 `index.md` 定位相关页面，再深入阅读并综合答案
3. **健康检查**：让 LLM 检查 wiki 的断链、矛盾、过时信息或缺失连接
4. **浏览**：用 Obsidian 打开本文件夹，利用图谱视图（Graph View）和双链探索

### 推荐阅读路径

- **零基础读 PostgreSQL** → 从 [[postgresql/01-入门阶段-基础教程/第00章-学前准备/0.1-数据库与SQL是什么]] 的第 00 章开始，**不要求任何前置知识**
- **系统学 RAG** → 从 [[rag/RAG_学习路径导航]] 进入 14 步流程
- **学 LangChain** → 入口 [[langchain/LangChain_学习笔记]] + 查询总入口 [[langchain/LangChain使用手册]]
- **想了解本知识库如何运作** → 读 [[meta/Welcome]] 与 [[meta/LLM_Wiki_方法论]]
- **查任何页面** → 先查 [[index|index.md]]，它按主题域组织了全部 533 页的链接与一行摘要

---

## 写作基线（重要约定）

**本知识库的默认读者是零基础学习者。** 任何主题的内容都不得假设读者已具备该领域的基础知识。

硬性规则：

1. 任何关键字、名词、函数、运算符、命令**首次出现时必须在同页解释**，或显式指向索引页
2. 入门层必须有「第 00 章」式的零基础入口，明确声明"不要求任何前置知识"
3. 每个难度层级有明确的「阅读门槛」声明，说明看不懂时该退回哪里
4. 低难度层级不得出现未解释的高难度术语；必须提及时标注"这是更后面的内容"
5. 每页写到「机制 / 实操 / 陷阱」三层——宁可写透一个点，不要罗列十个点
6. 每个主题配**总索引页兜底**（如 PostgreSQL 的函数手册、术语与关键字手册、速查索引），索引页只给签名，解释放正文
7. 技术内容须核实**当前版本现状**再写，并标注特性引入版本（如 `PG14+`）

## 页面规范

- 每个 wiki 页面必须带 YAML frontmatter：`tags` / `type`（entity/concept/source 等）/ `created` / `updated`
- 内部链接一律使用 Obsidian 双链 `[[路径/文件名]]`（不带 `.md`）
- 页面类型、命名规则、各类型的模板详见 `SCHEMA.md`
- 每次新增/改动页面后：更新 `index.md`，并在 `log.md` 顶部追加操作记录（类型：init/ingest/query/lint/update/fix/supplement）

## 质量保障

改动后必跑三项自动化校验（Python 脚本）：

1. **断链校验**——遍历全部 wikilink，比对目标文件是否存在（需处理表格内 `\|` 转义与行内代码中的语法示例两类误报）
2. **函数覆盖率校验**——提取正文中出现的函数名，比对函数手册是否已收录
3. **关键字/名词覆盖率校验**——提取代码块中的 SQL 关键字，比对手册收录情况

> 核心方法：**任何"读者可能不认识的东西"都应有一个可自动校验的索引页。** 把「索引页」与「正文」做全量比对，即可把"有没有用了没讲的东西"从人工抽样变成自动化核查。

---

## 核心理念

> **人类负责**：策展来源、指导分析、提出好问题
> **LLM 负责**：所有编写、维护、交叉引用和一致性工作

详细的方法论阐述见 `SCHEMA.md`（核心理念章节）与 [[meta/LLM_Wiki_方法论]]。

## 提示

- 用 Obsidian 的 **Graph View** 可视化知识连接关系
- wiki 本质是 Markdown 文件——可直接用 git 版本控制
- 在实践中迭代 `SCHEMA.md`，它是活文档
- 技术内容有版本时效性——各主题的版本基线以页面内标注为准（如 PostgreSQL 以 PG 18.6 为基线，2026-09 核实）
