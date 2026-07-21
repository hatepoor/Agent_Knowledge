---
tags: [meta, welcome, 导航, 入口]
created: 2026-06-20
updated: 2026-07-07
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
- [SCHEMA](SCHEMA.md) — 知识库操作规范和工作流定义
- [index](index.md) — 所有 wiki 页面的内容索引
- [log](log.md) — 操作日志（按时间顺序记录）

### 学习专区
- [[langchain/LangChain_学习笔记]] — LangChain 框架系统化学习
- [[agent/harness/Agent_三层架构]] — AI Agent 感知→思考→执行框架
- [[agent/harness/Agent_六种架构]] — 六种主流 Agent 架构模式对比

---

## 快速开始

1. **添加来源**：将文档放入 `raw/` 目录
2. **摄入**：告诉 LLM 处理来源，它会阅读、摘要、整合进 wiki
3. **查询**：对知识库提问，LLM 综合答案并附带引用
4. **浏览**：用 Obsidian 打开此文件夹，探索图谱视图和链接

---

*最后更新：2026-07-07*
