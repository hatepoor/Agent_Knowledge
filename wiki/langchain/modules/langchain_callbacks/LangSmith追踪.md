---
tags: [langchain, callbacks, langsmith, tracing, observability]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# LangSmith 追踪

LangSmith 是 LangChain 官方的**生产级追踪与可观测性平台**，通过回调系统自动采集所有 LLM 调用链数据。

## 环境变量配置

```python
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "lsv2_pt_..."
os.environ["LANGCHAIN_PROJECT"] = "my-project"
# 此后所有 LangChain 调用自动上报 LangSmith
```

## 配置说明

| 环境变量 | 作用 |
|----------|------|
| `LANGCHAIN_TRACING_V2` | 启用 v2 追踪（设为 `true`） |
| `LANGCHAIN_API_KEY` | LangSmith API 密钥 |
| `LANGCHAIN_PROJECT` | 项目名称（在 LangSmith 面板中分组） |
| `LANGCHAIN_ENDPOINT` | 自定义端点（可选，默认 `https://api.smith.langchain.com`） |

## 自动追踪

配置环境变量后，所有 LangChain 组件的调用链会自动上报：

```python
from langchain.chat_models import init_chat_model
from langchain.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")
prompt = ChatPromptTemplate.from_template("Tell me about {topic}")
chain = prompt | model | StrOutputParser()

# 自动追踪（无需手动注入 callback）
result = chain.invoke({"topic": "Python"})
```

## 手动使用 LangChainTracer

```python
from langchain.callbacks.tracers.langchain import LangChainTracer

tracer = LangChainTracer(project_name="my-project")

# 注入到特定调用
result = chain.invoke(
    {"topic": "Python"},
    config={"callbacks": [tracer][[},
)
```

## LangSmith 面板功能

| 功能 | 说明 |
|------|------|
| Traces | 查看每次调用的完整链路 |
| Runs | 按项目浏览所有运行 |
| Datasets | 管理评估数据集 |
| Evaluations | 运行自动化评估 |
| Monitoring | 成本、延迟、错误率监控 |

## 注意事项

- 环境变量方式最常用，零代码侵入
- LangSmith 有免费额度，生产环境需付费计划
- 追踪数据包含完整的 prompt、response、token 用量
- 敏感数据场景需评估数据安全策略

## 相关

- [[langchain/modules/langchain_callbacks/概述|<- 返回 callbacks 概述]]
- [[langchain/modules/langchain_callbacks/Token统计|Token 统计]]
