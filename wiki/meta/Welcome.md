---
tags: [meta, welcome, 导航, 入口]
created: 2026-06-20
updated: 2026-09-10
type: overview
---

# Welcome

欢迎来到 **LLM 驱动的知识库**。

本知识库基于 [[meta/LLM_Wiki_方法论]] 构建：LLM 增量式地构建和维护一个持久化的 wiki。人类负责策展来源、提出好问题；LLM 负责所有编写、维护、交叉引用和一致性工作。

---

## 快速导航

### 核心方法论
- [[meta/LLM_Wiki_方法论]] — 构建知识库的完整方法论
- [[meta/知识库三层架构]] — 原始来源层、Wiki 层、Schema 层的分离设计
- [[meta/综合]] — 跨来源深度整合能力
- [[meta/概览]] — 知识库导航性页面设计
- [[meta/RAG_vs_Wiki]] — RAG 检索 vs Wiki 编译的详细对比

### 知识库元文件
- [SCHEMA](../../SCHEMA.md) — 知识库操作规范和工作流定义
- [index](../../index.md) — 所有 wiki 页面的内容索引（384 页）
- [log](../../log.md) — 操作日志（按时间顺序记录）

---

## 学习专区

### Agent 系统（45 页）
- [[agent/patterns/Agent模式总览]] — **Agent 模式集中入口**：CoT / ReAct / Plan&Execute / Reflection / ToT / Reflexion / 多智能体协作，含选型决策树
- [[agent/harness/Agent_三层架构]] — 感知→思考→执行框架
- [[agent/harness/Agent_六种架构]] — 六种主流 Agent 架构模式对比
- [[agent/harness/Agent沙箱与安全隔离]] — Agent 沙箱五种隔离层次与配置方案
- [[agent/harness/Function_Call]] — 函数调用/工具调用机制
- [[agent/prompt/提示词工程]] — 提示词工程方法与模式
- [[agent/memory/Agent_Memory]] — Agent 记忆系统
- [[agent/reinforcement/Agentic_RL_工程实践]] — Agentic RL 工程实践
- [[agent/skills/Skills]] — Skills 系统
- [[agent/sources/菜鸟教程_AI_Agent系列]] — 菜鸟教程 Agent 系列

### RAG 流程化学习路径（37 页）
- [[rag/RAG_学习路径导航]] — **RAG 全流程入口**：从数据接入到评估监控的 14 步
- [[rag/00_导览与基础/RAG零基础一问一答]] — 新手问答总览
- [[rag/00_导览与基础/RAG基础与演进全景]] — RAG 基础与三代演进
- [[rag/00_导览与基础/LangChain_RAG端到端实现]] — 端到端代码实例
- [[rag/06_混合检索/BM25算法详解]] — BM25 算法详解
- [[rag/04_混合索引构建/ANN近似最近邻搜索]] — ANN 近似最近邻搜索
- [[rag/14_范式与前沿/Agentic_RAG]] — Agentic RAG 前沿范式
- [[rag/14_范式与前沿/GraphRAG与知识图谱]] — GraphRAG 与知识图谱

### LangChain 框架（143 页）
- [[langchain/LangChain_学习笔记]] — LangChain 学习导航入口
- [[langchain/LangChain使用手册]] — 章节化查询总入口
- [[langchain/modules/LangChain模块详解]] — 16 个核心模块地图
- [[langchain/overview/LangChain概述]] — 框架定位
- [[langchain/quickstart/快速入门]] — 快速入门
- [[langchain/middleware/中间件概述]] — 中间件系统（23 篇详解）
- [[langchain/modules/langchain-mcp/概述]] — MCP 协议集成

### LangGraph 框架（41 页）
- [[langgraph/overview/LangGraph概述]] — LangGraph 定位与核心特性
- [[langgraph/quickstart/快速入门]] — 构建首个图
- [[langgraph/modules/langgraph_multi_agent/概述]] — 多智能体（Supervisor/Network/Hierarchical）
- [[langgraph/modules/langgraph_human_in_loop/概述]] — 人机交互与审批
- [[langgraph/tutorial/综合示例_全概念演示]] — 全概念综合演示

### DeepAgents 框架（29 页）
- [[deepagents/overview/DeepAgents概述]] — DeepAgents 定位与默认中间件栈
- [[deepagents/overview/快速入门]] — 快速上手
- [[deepagents/overview/与LangChain_Agent对比]] — 与 LangChain Agent 对比
- [[deepagents/overview/组件依赖全景]] — 组件依赖全景图

### 基础技术与工具
- [[python/basics/Python_基础概览]] — Python 基础（26 页）
- [[sqlmodel/概述]] — SQLModel ORM（6 页）
- [[fastapi/FastAPI概述]] — FastAPI 集成（18 页）
- [[docker/Docker沙箱概述]] — Docker 沙箱（12 页）
- [[tools/向量数据库/概述]] — 向量数据库选型（22 页）
- [[tools/Dataview]] — Obsidian Dataview 插件
- [[tools/Marp]] — Markdown 演示文稿
- [[tools/qmd]] — 本地 Markdown 搜索引擎

---

## 快速开始

1. **添加来源**：将文档放入 `raw/` 目录
2. **摄入**：告诉 LLM 处理来源，它会阅读、摘要、整合进 wiki
3. **查询**：对知识库提问，LLM 综合答案并附带引用
4. **浏览**：用 Obsidian 打开此文件夹，探索图谱视图和链接

---

## 知识库统计

| 板块 | 页面数 | 说明 |
|------|--------|------|
| agent | 45 | Agent 模式、架构、记忆、RAG、强化学习 |
| rag | 37 | RAG 14 步全流程（数据→检索→生成→评估） |
| langchain | 143 | LangChain 核心 + 16 模块 + 中间件 |
| langgraph | 41 | LangGraph 核心 + 多智能体 + 人机交互 |
| deepagents | 29 | DeepAgents 框架全解 |
| python | 26 | Python 基础与进阶 |
| tools | 22 | 向量数据库 + 开发工具 |
| fastapi | 18 | FastAPI 集成 |
| docker | 12 | Docker 沙箱 |
| sqlmodel | 6 | SQLModel ORM |
| meta | 6 | 方法论与导航 |
| **合计** | **384** | |

---

*最后更新：2026-09-10*
