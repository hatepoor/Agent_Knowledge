---
tags: [entity, 工具, Obsidian, 插件, 数据查询]
created: 2026-06-20
updated: 2026-07-07
type: entity
---

# Dataview

## 概述

Dataview 是 Obsidian 的一款社区插件，可以对笔记的 YAML frontmatter 运行结构化查询，生成动态表格和列表。它是 wiki 元数据的强大展示工具。

## 核心特性

- **YAML 查询**：基于 frontmatter 字段（如 tags、created、type）筛选和排序页面
- **动态表格**：从查询结果自动生成 Markdown 表格
- **列表和任务**：生成列表、任务视图等多种展示形式
- **实时更新**：当笔记内容变化时，查询结果自动刷新

## 在知识库中的应用

当 LLM 在 wiki 页面中添加 YAML frontmatter（如 tags、created、updated、type）时，Dataview 可以：

```dataview
TABLE type, created, updated
FROM "wiki"
SORT created DESC
```

- 生成按日期排序的所有页面目录
- 按类型（entity、concept、source、comparison）分类展示
- 按标签筛选特定主题的页面

## 使用示例

```dataview
TABLE summary
FROM "wiki/agent"
WHERE type = "concept"
SORT file.name ASC
```

## 相关实体

- [[tools/Obsidian]] — Dataview 的运行宿主

## 来源

- [[meta/LLM_Wiki_方法论]] — 原文"技巧与贴士"章节
