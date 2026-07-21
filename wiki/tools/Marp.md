---
tags: [entity, 工具, 演示文稿, Markdown]
created: 2026-06-20
updated: 2026-07-07
type: entity
---

# Marp

## 概述

Marp 是一种基于 Markdown 的演示文稿格式，Obsidian 有相应的插件支持。它允许用户直接从 Markdown 内容生成幻灯片，非常适合从 wiki 知识库快速生成演示材料。

## 核心特性

- **Markdown 原生**：使用标准 Markdown 语法，学习成本极低
- **幻灯片分隔**：使用 `---` 分隔幻灯片页面
- **主题支持**：内置多种主题，支持自定义样式
- **导出格式**：支持导出为 PDF、HTML、PPTX 等格式

## 在知识库工作流中的应用

当需要向他人展示 wiki 中的研究成果时：
1. 从 wiki 页面中提取关键内容
2. 用 Marp 格式组织为幻灯片结构
3. 使用 Obsidian Marp 插件预览和导出

## 基本语法

```markdown
---
marp: true
theme: default
---

# 标题

内容...

---

# 第二页

- 要点 1
- 要点 2
```

## 相关实体

- [[tools/Obsidian]] — Marp 插件的运行宿主

## 来源

- [[meta/LLM_Wiki_方法论]] — 原文"技巧与贴士"章节
