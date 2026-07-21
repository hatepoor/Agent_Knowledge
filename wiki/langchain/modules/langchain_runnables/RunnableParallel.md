---
tags: [langchain, runnables, parallel]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RunnableParallel

## 是什么

`RunnableParallel` 让多个 Runnable **并行执行**，将同一个输入分别传给各个分支，最终将结果合并为一个字典。

## 导入

```python
from langchain_core.runnables import RunnableParallel
```

## 用法

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda
from langchain_core.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

model = init_chat_model("openai:gpt-5.5")

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a translator to {language}."),
    ("human", "{text}"),
])

# 多个链并行执行，合并结果
parallel_chain = RunnableParallel(
    summary=prompt | model | StrOutputParser(),
    word_count=RunnableLambda(lambda x: len(x["text"].split())),
)
result = parallel_chain.invoke({"language": "English", "text": "LangChain is great"})
print(result)  # {'summary': '...', 'word_count': 3}
```

## 字典语法简写

```python
# 使用字典语法也可以创建并行链（自动转为 RunnableParallel）
parallel_chain = {
    "translation": prompt | model | StrOutputParser(),
    "length": RunnableLambda(lambda x: len(x["text"][[)),
}
```

## 执行流程

```
输入 → ┬→ branch_a（Runnable 1）→ result_a
       └→ branch_b（Runnable 2）→ result_b

输出 = {"branch_a": result_a, "branch_b": result_b}
```

## 特点

- 所有分支**同时执行**，提升性能
- 每个分支接收相同的输入
- 输出为字典，key 为分支名称
- 支持任意数量的并行分支

## 何时使用

- 需要对同一输入执行多种不同处理
- 同时获取翻译、摘要、字数统计等多种结果
- 多模型对比（同一输入发给不同模型）
- 性能优化：可以并行的独立步骤

## 相关

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 概述]]
- [[langchain/modules/langchain_runnables/LCEL链式组合|LCEL 链式组合 - 串行]]
- [[langchain/modules/langchain_runnables/RunnableBranch|RunnableBranch - 条件分支]]
