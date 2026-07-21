---
tags: [entity, 工具, Obsidian, 浏览器扩展]
created: 2026-06-20
updated: 2026-07-07
type: entity
---

# Obsidian Web Clipper

## 概述

Obsidian Web Clipper 是一个浏览器扩展，可以将网页文章转换为 Markdown 格式，并保存到 Obsidian vault 中。它是快速获取来源到知识库原始收集目录的高效工具。

## 核心特性

- **一键剪裁**：将网页内容转换为 Markdown
- **格式保留**：尽量保留原文的标题、列表、链接等结构
- **图片处理**：支持下载图片到本地附件目录
- **元数据提取**：自动提取标题、作者、日期等信息

## 在知识库工作流中的角色

1. **发现来源**：浏览网页时遇到有价值的文章
2. **一键剪裁**：使用 Web Clipper 将文章转为 Markdown
3. **保存到 raw/**：将文件放入知识库的 `raw/` 目录
4. **LLM 摄入**：告诉 LLM 处理新来源，整合进 wiki

## 图片处理建议

在 Obsidian 设置中：
- 将"附件文件夹路径"设置为固定目录（如 `raw/assets/`）
- 绑定快捷键下载当前文件的附件
- 这样所有图片都会下载到本地，LLM 可以直接查看和引用

## 相关实体

- [[tools/Obsidian]] — Web Clipper 的目标 vault 应用
- [[tools/Git]] — 剪裁的文章可通过 Git 追踪变更

## 来源

- [[meta/LLM_Wiki_方法论]] — 原文"技巧与贴士"章节
