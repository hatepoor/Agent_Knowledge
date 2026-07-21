---
tags: [entity, 工具, 知识管理, Obsidian]
created: 2026-06-20
updated: 2026-07-07
type: entity
---

# Obsidian

## 概述

Obsidian 是一款基于本地 Markdown 文件的知识管理应用。在本知识库中，Obsidian 是**wiki 的浏览环境**——提供图谱视图、双向链接、插件生态等核心浏览体验。

## 核心特性

- **本地优先**：所有数据存储在本地文件系统，完全拥有你的数据
- **双向链接**：`[[Page_Name]]` 格式支持页面间快速跳转和引用
- **图谱视图**：可视化页面间的连接关系，识别枢纽和孤岛
- **插件生态**：社区提供数千插件，扩展功能几乎无限
- **Markdown 原生**：纯文本格式，不锁定数据

## 在知识库工作流中的角色

> "Obsidian 是 IDE；LLM 是程序员；wiki 是代码库。" — [[meta/LLM_Wiki_方法论]]

### 具体使用方式

1. **左侧 LLM，右侧 Obsidian**：LLM 根据对话进行编辑，用户实时浏览结果
2. **跟随链接**：点击 wikilink 跳转相关页面，探索知识网络
3. **查看图谱**：Graph View 展示知识库的整体形态
4. **插件增强**：使用 Dataview、Marp 等插件扩展功能

## 与 Git 的配合

Obsidian vault 本质上是普通文件夹，可以直接用 [[tools/Git]] 管理。社区也有 Obsidian Git 插件提供 vault 内的 Git 操作界面。

## 相关实体

- [[tools/Obsidian_Web_Clipper]] — 浏览器扩展，快速将网页转为 Markdown
- [[tools/Git]] — 管理 Obsidian vault 的版本控制
- [[tools/Dataview]] — Obsidian 的结构化数据查询插件
- [[tools/Marp]] — 基于 Markdown 的演示文稿插件
- [[tools/Graph_View]] — Obsidian 的节点-连接图可视化功能

## 来源

- [[meta/LLM_Wiki_方法论]] — 原文"技巧与贴士"章节
