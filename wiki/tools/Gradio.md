---
tags: [entity, 工具, Python, UI, 原型]
type: entity
created: 2026-06-23
updated: 2026-07-08
---

# Gradio

## 概述

Gradio 是一个开源 Python 库，用于快速为机器学习模型创建可交互的 Web 界面。只需几行代码，就能将模型打包为可分享的 Web 应用。由 Hugging Face 维护，是其 Spaces 平台的默认部署框架。

## 核心特性

- **快速原型**：`gr.Interface` 一行代码创建完整 UI
- **多种组件**：文本、图像、音频、视频、3D 模型、数据表格等 30+ 种输入输出组件
- **自动分享**：`share=True` 生成 72 小时有效的公共 URL
- **Hugging Face 集成**：一键部署到 HF Spaces，与 Model Hub 深度联动
- **Blocks API**：高级布局 API，支持复杂多步骤交互界面
- **实时流式**：支持音频/文本的流式输入输出

## 最简示例

```python
import gradio as gr

def greet(name, intensity):
    return "Hello, " + name + "!" * int(intensity)

demo = gr.Interface(
    fn=greet,
    inputs=["text", "slider"],
    outputs="text",
)
demo.launch()
```

## 在 Agent 开发中的应用

Gradio 常用于快速搭建 Agent 的测试与演示界面：

- **对话界面**：`gr.ChatInterface` 快速构建聊天式 Agent 前端
- **推理链可视化**：展示 Agent 的思考过程、工具调用和中间结果
- **多模态交互**：支持图像上传、音频输入等多模态 Agent 测试
- **快速对比**：并排展示不同模型/Agent 的输出结果

```python
import gradio as gr

def agent_chat(message, history):
    # 接入 Agent 逻辑
    response = agent.invoke({"input": message})
    return response["output"][[

demo = gr.ChatInterface(fn=agent_chat, type="messages")
demo.launch()
```

## 与 Streamlit 的对比

| 特性 | Gradio | [[tools/Streamlit|Streamlit]] |
|------|--------|-----------|
| 定位 | ML 模型演示、快速原型 | 数据应用、仪表板 |
| 代码量 | 更少（Interface 一行） | 较少（但需更多布局代码） |
| 自定义能力 | 中等（Blocks 可扩展） | 更高（组件更灵活） |
| 多页面应用 | 较弱 | 原生支持 `st.Page` |
| ML 生态 | Hugging Face 深度集成 | 更偏向数据分析 |
| 部署 | HF Spaces 一键部署 | Streamlit Community Cloud |
| 适合场景 | 模型 Demo、Agent 测试 | 数据看板、分析报告 |

## 局限性

- 复杂 UI 定制不如前端框架灵活
- 多页面应用支持较弱
- 生产环境部署需要额外配置（认证、负载均衡等）

## 相关页面

- [[tools/Streamlit]] — 定位互补的 Python 数据应用框架
- [[agent/orchestration/LangChain]] — Gradio 常作为 LangChain Agent 的前端
- [[tools/Chroma]] — Agent 原型中常用的向量数据库

## 来源

- [[agent/orchestration/开发Agent实操指南]] — 原文提到 Gradio 用于 Agent 测试版部署
