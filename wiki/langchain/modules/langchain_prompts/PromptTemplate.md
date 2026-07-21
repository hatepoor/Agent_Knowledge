---
tags: [langchain, prompts, PromptTemplate, template]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# PromptTemplate

`PromptTemplate` 是 LangChain 中最基础的模板类，用于创建**单条字符串提示词**。它接受一个包含 `{变量}` 占位符的模板字符串，在运行时用实际值替换这些占位符，生成最终的提示词文本。

> 对于聊天场景（需要 system + human 等多条消息），推荐使用 [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]]。

## 基础用法

```python
from langchain_core.prompts import PromptTemplate

# 方式1：显式指定 input_variables
template = PromptTemplate(
    template="请解释什么是{concept}，用{language}回答",
    input_variables=["concept", "language"]
)

# 格式化
result = template.format(concept="机器学习", language="中文")
print(result)
# 输出：请解释什么是机器学习，用中文回答

# 方式2：自动推断 input_variables（推荐）
template = PromptTemplate.from_template(
    "你是一个{role}专家。请回答：{question}"
)
# 自动推断 input_variables = ["role", "question"][[

result = template.format(role="Python", question="什么是装饰器？")
print(result)
```

## 与 LLM 链式调用

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

# 创建模板
template = PromptTemplate.from_template(
    "用简单的语言解释什么是{concept}"
)

# 创建链：template -> model -> parser
model = ChatOpenAI(model="gpt-4o-mini")
chain = template | model | StrOutputParser()

# 调用
result = chain.invoke({"concept": "量子计算"})
print(result)
```

## format_prompt vs format

```python
from langchain_core.prompts import PromptTemplate

template = PromptTemplate.from_template("你好，{name}")

# format()：返回字符串
text = template.format(name="Alice")
print(type(text))  # <class 'str'>

# format_prompt()：返回 PromptValue 对象（可转换为字符串或消息）
prompt_value = template.format_prompt(name="Bob")
print(type(prompt_value))  # <class 'langchain_core.prompt_values.StringPromptValue'>
print(prompt_value.to_string())  # 转为字符串
print(prompt_value.to_messages())  # 转为消息列表
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/from_template方法|from_template 方法]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]] — 聊天场景推荐
