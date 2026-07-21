---
tags: [langchain, runnables, lambda]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RunnableLambda

## 是什么

`RunnableLambda` 将**普通 Python 函数**包装为 Runnable，使其可以无缝嵌入 LCEL 链中。

## 导入

```python
from langchain_core.runnables import RunnableLambda
```

## 用法

```python
from langchain_core.runnables import RunnableLambda
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a translator to {language}."),
    ("human", "{text}"),
][[)

# 用 lambda 包装普通函数
uppercase = RunnableLambda(lambda x: x.upper())
chain = prompt | model | StrOutputParser() | uppercase
print(chain.invoke({"language": "Spanish", "text": "hello"}))  # "HOLA MUNDO"
```

## 更复杂的函数

```python
def process_output(text: str) -> dict:
    """将文本输出转换为结构化数据"""
    words = text.split()
    return {
        "text": text,
        "word_count": len(words),
        "char_count": len(text),
    }

processor = RunnableLambda(process_output)
chain = prompt | model | StrOutputParser() | processor
result = chain.invoke({"language": "English", "text": "Hello world"})
# result = {"text": "...", "word_count": 10, "char_count": 50}
```

## 特点

- 接受任何可调用对象（函数、lambda、方法）
- 自动继承 Runnable 的 `invoke` / `stream` / `batch` 接口
- 输入输出类型由包装的函数决定
- 适合插入自定义的数据转换步骤

## 何时使用

- 需要在链中插入自定义处理逻辑
- 数据格式转换（字符串到字典、列表到字符串等）
- 添加日志、调试等辅助步骤
- 不适合用已有 Runnable 组件表达的自定义逻辑

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [[langchain/modules/langchain_runnables/LCEL链式组合|LCEL 链式组合]]
- [[langchain/modules/langchain_runnables/RunnablePassthrough|RunnablePassthrough - 透传输入]]
