---
tags: [entity, 工具, AI, 框架, 知识融合]
created: 2026-06-23
updated: 2026-06-23
type: entity
---

# LlamaIndex

## 概述

LlamaIndex 是一个专注于**数据连接和知识融合**的 LLM 应用框架。与 LangChain 的通用性不同，LlamaIndex 的核心优势在于将私有数据高效地连接到 LLM，擅长构建 RAG 系统和知识密集型 Agent。

## 核心能力

- **数据连接器**：支持 160+ 数据源（文件、数据库、API、SaaS 工具等）
- **索引构建**：多种索引结构（向量索引、树索引、关键词索引等）
- **查询引擎**：针对不同索引类型的高效检索策略
- **SimpleVectorStore**：极简的向量存储方案，几行代码实现长期记忆

## 在 Agent 开发中的角色

在 [[agent/orchestration/开发Agent实操指南]] 中，LlamaIndex 被推荐用于：
- **知识融合场景**：将各种数据源连接到 Agent
- **长期记忆实现**：用 SimpleVectorStore 快速搭建向量检索记忆
- 与 LangChain 可互补使用

## 与 LangChain 的差异

| 方面 | LlamaIndex | LangChain |
|------|-----------|-----------|
| 核心定位 | 数据连接与知识融合 | 通用 LLM 应用开发 |
| 优势场景 | RAG、知识库问答 | Agent、对话、工作流 |
| 向量存储 | SimpleVectorStore（极简） | 需对接外部向量数据库 |
| 学习曲线 | 较低，聚焦单一领域 | 较高，功能覆盖广 |

## 相关实体
- [[agent/orchestration/LangChain]] — 通用 LLM 应用框架，可与 LlamaIndex 互补
- [[agent/RAG/RAG]] — LlamaIndex 的核心应用场景

## 相关概念
- [[agent/memory/Agent_Memory]] — LlamaIndex 的 SimpleVectorStore 是长期记忆的极简方案
- [[agent/RAG/RAG]] — LlamaIndex 是 RAG 领域的专业框架

## 来源
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 原文推荐的知识融合框架
