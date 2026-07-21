---
tags: [langchain, prompts, FewShotChatMessagePromptTemplate, few-shot, chat]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# FewShotChatMessagePromptTemplate

`FewShotChatMessagePromptTemplate` 用于为 **chat 模型**（如 `ChatOpenAI`、`ChatAnthropic`）创建 few-shot 提示词。输出是**消息列表**，每个示例格式化为 HumanMessage/AIMessage 等，可以嵌入到 `ChatPromptTemplate` 中与 system 消息组合使用。

> 这是当前主流方案，推荐在所有 chat 模型场景中使用。

## 与 FewShotPromptTemplate 的核心区别

| 维度 | FewShotPromptTemplate | FewShotChatMessagePromptTemplate |
|------|----------------------|----------------------------------|
| 输出类型 | 纯字符串 (`str`) | 消息列表 (`list[BaseMessage]`) |
| 示例模板 | `PromptTemplate` | `ChatPromptTemplate` |
| 适用模型 | text completion 模型（已过时） | chat 模型（当前主流） |
| 能否嵌入 ChatPromptTemplate | 不能 | 能，作为消息列表插入 |
| prefix/suffix | 有，用于拼接字符串 | 无，通过外层 ChatPromptTemplate 组织 |
| example_separator | 有，字符串分隔符 | 无，消息天然分隔 |
| 当前推荐度 | 低（仅兼容老接口） | 高（推荐方案） |

## 基础用法

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    FewShotChatMessagePromptTemplate,
)

# 定义示例
examples = [
    {"input": "2+2", "output": "4"},
    {"input": "3+5", "output": "8"},
    {"input": "10-3", "output": "7"},
]

# 创建 few-shot 模板（注意 example_prompt 必须是 ChatPromptTemplate）
few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=ChatPromptTemplate.from_messages([
        ("human", "{input}"),
        ("ai", "{output}"),
    ]),
    examples=examples,
)

# 组合到主模板
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个计算器"),
    few_shot_prompt,
    ("human", "{input}"),
])

messages = final_prompt.format_messages(input="6*7")

for msg in messages:
    print(f"{msg.type}: {msg.content}")
# system: 你是一个计算器
# human: 2+2
# ai: 4
# human: 3+5
# ai: 8
# human: 10-3
# ai: 7
# human: 6*7
```

## 动态示例选择

```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore

# 所有示例
all_examples = [
    {"input": "2+2", "output": "4"},
    {"input": "3+5", "output": "8"},
    {"input": "10-3", "output": "7"},
    {"input": "2*3", "output": "6"},
    {"input": "8/2", "output": "4"},
]

# 使用示例选择器（根据相似度选择最相关的示例）
example_selector = SemanticSimilarityExampleSelector.from_examples(
    examples=all_examples,
    embeddings=OpenAIEmbeddings(),
    vectorstore_cls=InMemoryVectorStore,
    k=2,  # 选择 2 个最相关的示例
)

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=ChatPromptTemplate.from_messages([
        ("human", "{input}"),
        ("ai", "{output}"),
    ][[),
    example_selector=example_selector,
)

# 调用时会自动选择最相关的示例
messages = few_shot_prompt.format_messages(input="5*6")
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/FewShotPromptTemplate|FewShotPromptTemplate]] — 字符串版 few-shot（已过时）
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]] — example_prompt 必须使用此类
