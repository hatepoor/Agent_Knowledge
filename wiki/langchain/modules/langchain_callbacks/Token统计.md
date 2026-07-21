---
tags: [langchain, callbacks, token-counting, openai, cost]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Token 统计

LangChain 提供了 `get_openai_callback` 上下文管理器，可以方便地统计 OpenAI 模型的 **token 用量和费用**。

## 导入

```python
# ⚠️ langchain_community 已弃用，推荐使用 OpenAI 原生的 usage_metadata
from langchain_community.callbacks import get_openai_callback
```

## 基本用法

```python
from langchain.chat_models import init_chat_model
from langchain.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")
prompt = ChatPromptTemplate.from_template("Tell me about {topic}")
chain = prompt | model | StrOutputParser()

with get_openai_callback() as cb:
    result = chain.invoke({"topic": "token counting"})
    result = chain.invoke({"topic": "another call"})

    print(f"Total Tokens: {cb.total_tokens}")
    print(f"Prompt Tokens: {cb.prompt_tokens}")
    print(f"Completion Tokens: {cb.completion_tokens}")
    print(f"Total Cost (USD): ${cb.total_cost}")
```

## 可用属性

| 属性 | 含义 |
|------|------|
| `total_tokens` | 总 token 数 |
| `prompt_tokens` | 输入 token 数 |
| `completion_tokens` | 输出 token 数 |
| `total_cost` | 总费用（USD） |
| `successful_requests` | 成功请求数 |

## 嵌套使用

```python
with get_openai_callback() as outer_cb:
    chain.invoke({"topic": "first"})

    with get_openai_callback() as inner_cb:
        chain.invoke({"topic": "second"})
        chain.invoke({"topic": "third"})
        print(f"Inner: {inner_cb.total_tokens} tokens, ${inner_cb.total_cost}")

    chain.invoke({"topic": "fourth"})
    print(f"Outer: {outer_cb.total_tokens} tokens, ${outer_cb.total_cost}")
```

## 自定义 Token 统计 Handler

如果不使用 OpenAI，可以通过自定义 handler 统计：

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult

class UniversalTokenCounter(BaseCallbackHandler):
    def __init__(self):
        self.total_tokens = 0
        self.prompt_tokens = 0
        self.completion_tokens = 0

    def on_llm_end(self, response: LLMResult, **kwargs) -> None:
        for gen_list in response.generations:
            for gen in gen_list:
                usage = gen.message.usage_metadata or {}
                self.prompt_tokens += usage.get("input_tokens", 0)
                self.completion_tokens += usage.get("output_tokens", 0)
                self.total_tokens += usage.get("total_tokens", 0)

counter = UniversalTokenCounter()
chain.invoke({"topic": "test"}, config={"callbacks": [counter][[})
print(f"Tokens: {counter.total_tokens}")
```

## 注意事项

- `get_openai_callback` 仅适用于 OpenAI 模型（包括 Azure OpenAI）
- 非 OpenAI 模型使用自定义 handler 或查看模型返回的 `usage_metadata`
- `total_cost` 基于 OpenAI 官方定价自动计算
- 上下文管理器内的所有 LLM 调用都会被统计

## 相关

- [[langchain/modules/langchain_callbacks/概述|<- 返回 callbacks 概述]]
- [[langchain/modules/langchain_callbacks/LangSmith追踪|LangSmith 追踪]] -- 生产级监控方案
- [[langchain/modules/langchain_callbacks/自定义CallbackHandler|自定义 CallbackHandler]]
