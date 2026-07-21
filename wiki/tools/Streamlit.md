---
tags: [entity, 工具, Python, UI, 原型]
created: 2026-06-23
updated: 2026-07-07
type: entity
---

# Streamlit

## 概述

Streamlit 是一个 Python 库，用于快速构建数据应用和机器学习工具的前端界面。它将数据脚本转化为可分享的 Web 应用，是 Python 开发者最常用的快速原型工具之一。

## 核心特性

- **纯 Python**：无需前端知识，全部用 Python 编写
- **即时刷新**：保存代码后页面自动更新
- **丰富组件**：表格、图表、地图、表单、文件上传等
- **缓存机制**：支持数据缓存，提升性能

## 在 Agent 开发中的应用

Streamlit 常用于：
- 快速搭建 Agent 的交互界面
- 展示 Agent 的推理链和中间状态
- 可视化 Agent 的工具调用结果
- 构建多页面应用展示不同 Agent 能力

## 与 Gradio 的对比

| 特性 | Gradio | Streamlit |
|------|--------|-----------|
| 定位 | ML 模型演示 | 数据应用构建 |
| 学习曲线 | 更低 | 较低 |
| 自定义能力 | 中等 | 更高 |
| 多页面 | 较弱 | 原生支持 |

## 来源

- [[agent/orchestration/开发Agent实操指南]] — 原文提到 Streamlit 用于 Agent 测试版部署
