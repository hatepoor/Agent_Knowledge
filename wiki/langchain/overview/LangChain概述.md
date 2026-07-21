---
tags: [langchain, overview, agent, source]
created: 2026-07-07
updated: 2026-07-07
type: source
source_url: https://docs.langchain.com/oss/python/langchain/overview
---

# LangChain 概述

LangChain 提供 `create_agent`：一个极简、高度可配置的 Agent 框架（Harness）。你可以从模型、工具、提示词和中间件出发，精确组合出适合你用例的 Agent。

**Agent = 模型 + 框架（Harness）。** LangChain 提供 `create_agent`：一个极简、高度可配置的框架。框架涵盖了模型循环周围的一切：提示词、工具以及任何塑造行为的中间件。从基础组件出发，精确组合你所需的方案。支持 [OpenAI、Anthropic、Google 等提供商](https://docs.langchain.com/oss/python/integrations/providers/overview)。

## LangChain vs. LangGraph vs. Deep Agents

- **Deep Agents**：入门首选——提供"开箱即用"的 Agent，具备自动上下文压缩、虚拟文件系统和子 Agent 生成等功能。Deep Agents 构建于 LangChain Agent 之上，你也可以直接使用后者。
- **LangChain**（`create_agent`）：提供高度可定制的框架，可根据用例和数据轻松调整。
- **LangGraph**：低级编排框架，适用于需要混合确定性工作流和 Agent 工作流的高级需求。
- **LangSmith**：追踪、调试和评估使用上述任何框架构建的 Agent。

## 创建 Agent

以下示例演示如何创建一个带自定义工具的简单 LangChain Agent：

```python
# pip install -qU langchain "langchain[openai]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1][[.content_blocks)
```

参考 [[langchain/install/安装LangChain|安装指南]] 和 [[langchain/quickstart/快速入门|快速入门]] 开始构建自定义 Agent 和应用。

## 相关页面

- [[agent/orchestration/LangChain]] — LangChain 在 Agent 生态中的定位
- [[langchain/agents/Agent代理]] — Agent 核心组件详解
- [[langchain/tools/工具]] — 工具定义与使用
- [[langchain/models/模型]] — 模型配置与管理
