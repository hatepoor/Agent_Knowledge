---
tags: [langchain, callbacks, custom-handler, monitoring]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# 自定义 CallbackHandler

通过继承 `BaseCallbackHandler` 或 `AsyncCallbackHandler`，可以实现自定义的监控、日志和指标采集逻辑。

## 同步版本

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult

class TokenCountingHandler(BaseCallbackHandler):
    """统计每次 LLM 调用的 token 数"""
    def __init__(self):
        self.total_tokens = 0
        self.prompt_tokens = 0
        self.completion_tokens = 0
        self.call_count = 0

    def on_llm_start(self, serialized: dict, prompts: list[str], **kwargs) -> None:
        self.call_count += 1
        print(f"[LLM #{self.call_count}] Starting with {len(prompts)} prompts")

    def on_llm_end(self, response: LLMResult, **kwargs) -> None:
        for gen_list in response.generations:
            for gen in gen_list:
                usage = gen.message.usage_metadata or {}
                self.prompt_tokens += usage.get("input_tokens", 0)
                self.completion_tokens += usage.get("output_tokens", 0)
                self.total_tokens += usage.get("total_tokens", 0)

    def on_llm_error(self, error: BaseException, **kwargs) -> None:
        print(f"[LLM ERROR] {error}")
```

## 通过 config 注入回调

```python
from langchain.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5.5")
handler = TokenCountingHandler()

prompt = ChatPromptTemplate.from_template("Tell me about {topic}")
chain = prompt | model | StrOutputParser()

result = chain.invoke(
    {"topic": "Python"},
    config={"callbacks": [handler]},
)
print(f"Stats: calls={handler.call_count}, prompt={handler.prompt_tokens}, "
      f"completion={handler.completion_tokens}, total={handler.total_tokens}")
```

## 流式 token 回调

```python
class StreamingPrinter(BaseCallbackHandler):
    """实时打印 token"""
    def on_llm_new_token(self, token: str, **kwargs) -> None:
        print(token, end="", flush=True)

    def on_llm_end(self, response, **kwargs) -> None:
        print()  # 换行

model.invoke("Tell me a story", config={"callbacks": [StreamingPrinter()]})
```

## 工具回调

```python
class ToolLogger(BaseCallbackHandler):
    def on_tool_start(self, serialized: dict, input_str: str, **kwargs) -> None:
        print(f"[TOOL] {serialized.get('name')} called with: {input_str}")

    def on_tool_end(self, output: str, **kwargs) -> None:
        print(f"[TOOL] returned: {output[:100]}")

    def on_tool_error(self, error: BaseException, **kwargs) -> None:
        print(f"[TOOL ERROR] {error}")
```

## 异步版本

```python
from langchain_core.callbacks import AsyncCallbackHandler
import asyncio

class AsyncCounter(AsyncCallbackHandler):
    async def on_llm_start(self, *args, **kwargs):
        print("Async LLM start")

    async def on_llm_end(self, response, **kwargs):
        print("Async LLM end")

async def main():
    result = await chain.ainvoke(
        {"topic": "Async"},
        config={"callbacks": [AsyncCounter()]},
    )

asyncio.run(main())
```

## 注意事项

- 回调处理器实例**共享状态**，多次调用间数据会累积
- 异步场景**必须**使用 `AsyncCallbackHandler`，否则会阻塞事件循环
- 可以通过 `config={"callbacks": [...][[}` 注入，无需修改业务代码
- 多个 handler 可以同时注入，按列表顺序执行

## 相关

- [[langchain/modules/langchain_callbacks/概述|<- 返回 callbacks 概述]]
- [[langchain/modules/langchain_callbacks/回调事件类型|回调事件类型]]
