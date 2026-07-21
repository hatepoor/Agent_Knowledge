---
tags: [langchain, prompts, from_template, classmethod]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# from_template() 方法

`from_template()` 是多个模板类的**类方法（classmethod）**，用于从字符串模板快速创建实例。核心优势是**自动扫描 `{变量}` 并推断 `input_variables`**，省去手动声明。

## 各类型上的行为

| 类 | 返回类型 | 效果 |
|---|---------|------|
| `PromptTemplate.from_template(...)` | `PromptTemplate` | 自动推断变量的字符串模板 |
| `ChatPromptTemplate.from_template(...)` | `ChatPromptTemplate` | 等价于 `[("human", "...")]`，只生成一条 human 消息 |
| `SystemMessagePromptTemplate.from_template(...)` | `SystemMessagePromptTemplate` | 一条 system 消息模板 |
| `HumanMessagePromptTemplate.from_template(...)` | `HumanMessagePromptTemplate` | 一条 human 消息模板 |
| `AIMessagePromptTemplate.from_template(...)` | `AIMessagePromptTemplate` | 一条 AI 消息模板 |

## 自动推断机制

```python
from langchain_core.prompts import PromptTemplate

# from_template 自动扫描 {} 提取变量名
template = PromptTemplate.from_template("请解释什么是{concept}，用{language}回答")

# 等价于手动写：
# PromptTemplate(
#     template="请解释什么是{concept}，用{language}回答",
#     input_variables=["concept", "language"],
# )

print(template.input_variables)  # ['concept', 'language']
```

## from_template() vs from_messages()

`from_template()` 只适合**单条消息**场景。需要多条消息（system + human + few-shot 等）时用 `from_messages()`：

```python
from langchain_core.prompts import ChatPromptTemplate

# 单条消息 — from_template 简洁
prompt = ChatPromptTemplate.from_template("请解释{concept}")

# 多条消息 — 必须用 from_messages
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家"),
    ("human", "请解释{concept}"),
])

# from_template 无法创建多条消息
prompt = ChatPromptTemplate.from_template("你是一个专家。请解释{concept}")
# 只会生成一条 human 消息，"你是一个专家" 也变成了 human 消息的内容
```

## 在 FewShotPromptTemplate 中的典型用法

`from_template()` 常用于创建 `example_prompt`，配合 `FewShotPromptTemplate` 使用：

```python
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

# from_template 创建示例模板（自动推断变量）
example_prompt = PromptTemplate.from_template("Input: {word}\nOutput: {antonym}")

few_shot = FewShotPromptTemplate(
    examples=[
        {"word": "happy", "antonym": "sad"},
        {"word": "tall", "antonym": "short"},
    ],
    example_prompt=example_prompt,
    prefix="给出反义词：",
    suffix="Input: {adjective}\nOutput: ",
    input_variables=["adjective"][[,
)
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/PromptTemplate|PromptTemplate]]
- [[langchain/modules/langchain_prompts/ChatPromptTemplate|ChatPromptTemplate]]
- [[langchain/modules/langchain_prompts/FewShotPromptTemplate|FewShotPromptTemplate]]
