---
tags: [langchain, install, source]
created: 2026-07-07
updated: 2026-07-07
type: source
source_url: https://docs.langchain.com/oss/python/langchain/install
---

# 安装 LangChain

## 安装

安装 LangChain 包：

```bash
pip install -U langchain
# 需要 Python 3.10+
```

LangChain 提供数百个 LLM 和数千个其他集成的支持，这些集成位于独立的 provider 包中：

```bash
# 安装 OpenAI 集成
pip install -U langchain-openai

# 安装 Anthropic 集成
pip install -U langchain-anthropic
```

参见 [Integrations 页面](https://docs.langchain.com/oss/python/integrations/providers/overview) 获取完整的可用集成列表。

安装完成后，可参考 [[langchain/quickstart/快速入门|快速入门指南]] 开始使用。

设置 [LangSmith](https://smith.langchain.com) 追踪以调试你的首个 LangChain 应用。

## 相关页面

- [[langchain/quickstart/快速入门]] — 构建首个 Agent
- [[langchain/agents/Agent代理]] — Agent 核心概念
