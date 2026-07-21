---
tags: [langchain, prompts, application, rag, agent, structured-output]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# Prompt 应用场景

本文展示 `langchain_core.prompts` 在实际开发中的常见应用场景，每个场景都提供完整的可运行代码。

## 场景 1：RAG 问答

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

rag_prompt = ChatPromptTemplate.from_template("""
根据以下上下文回答问题。如果上下文中没有答案，请说"我不知道"。

上下文：
{context}

问题：{question}

回答：
""")

# 在 RAG 链中使用
model = ChatOpenAI(model="gpt-4o-mini")
chain = rag_prompt | model | StrOutputParser()

result = chain.invoke({
    "context": "LangChain 是一个 LLM 应用开发框架",
    "question": "LangChain 是什么？"
})
print(result)
```

## 场景 2：多轮对话 Agent

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

agent_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}。请用{language}回答"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

model = ChatOpenAI(model="gpt-4o-mini")
chain = agent_prompt | model | StrOutputParser()

# 第一轮
response = chain.invoke({
    "role": "Python 专家",
    "language": "中文",
    "chat_history": [],
    "input": "什么是装饰器？"
})

# 第二轮（传入历史）
response = chain.invoke({
    "role": "Python 专家",
    "language": "中文",
    "chat_history": [
        HumanMessage(content="什么是装饰器？"),
        AIMessage(content=response),
    ],
    "input": "能给个例子吗？"
})
```

## 场景 3：结构化输出

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class MovieRecommendation(BaseModel):
    title: str = Field(description="电影名称")
    year: int = Field(description="上映年份")
    reason: str = Field(description="推荐理由")

prompt = ChatPromptTemplate.from_template(
    "推荐一部{genre}电影，以JSON格式返回"
)

# 使用 with_structured_output
model = ChatOpenAI(model="gpt-4o-mini")
structured_model = model.with_structured_output(MovieRecommendation)

chain = prompt | structured_model
result = chain.invoke({"genre": "科幻"})
print(result.title)  # 电影名称
```

## 场景 4：翻译助手

```python
from langchain_core.prompts import ChatPromptTemplate

translation_prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个专业翻译。请将用户输入从{source_lang}翻译成{target_lang}。
    
规则：
1. 保持原文的语气和风格
2. 不要添加额外解释
3. 只返回翻译结果"""),
    ("human", "{text}"),
])

# 使用
messages = translation_prompt.format_messages(
    source_lang="英文",
    target_lang="中文",
    text="Hello, how are you?"
)

for msg in messages:
    print(f"{msg.type}: {msg.content}")
```

## 场景 5：代码生成

```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

# Few-shot 示例
examples = [
    {
        "description": "计算两个数的和",
        "code": "def add(a, b):\n    return a + b"
    },
    {
        "description": "计算列表的最大值",
        "code": "def max_value(lst):\n    return max(lst)"
    },
]

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=ChatPromptTemplate.from_messages([
        ("human", "写一个Python函数：{description}"),
        ("ai", "```python\n{code}\n```"),
    ]),
    examples=examples,
)

code_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个Python专家。请根据描述编写函数。"),
    few_shot_prompt,
    ("human", "写一个Python函数：{description}"),
][[)

messages = code_prompt.format_messages(description="计算两个数的乘积")

for msg in messages:
    print(f"{msg.type}: {msg.content}")
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]]
- [[langchain/modules/langchain_prompts/FewShotChatMessagePromptTemplate|FewShotChatMessagePromptTemplate]]
- [[langchain/modules/langchain_prompts/MessagesPlaceholder|MessagesPlaceholder]]
