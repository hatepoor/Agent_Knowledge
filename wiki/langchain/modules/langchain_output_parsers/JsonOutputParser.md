---
tags: [langchain, output-parsers, json]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# JsonOutputParser

## 是什么

`JsonOutputParser` 将 LLM 的输出解析为 **JSON 对象（dict）**，适合需要提取结构化字段但不想定义 Pydantic 模型的场景。支持三种使用模式：无 schema 约束、带 `pydantic_object` schema 约束、以及配合 `format_instructions` 注入提示词。

## 导入

```python
from langchain_core.output_parsers import JsonOutputParser
```

## 三种用法模式

### 模式一：无 schema 约束（最简用法）

不传入任何参数，直接将 LLM 输出解析为 `dict`。适合快速实验或 LLM 输出格式比较稳定的场景。

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

model = ChatOpenAI(model="gpt-4o-mini")
parser = JsonOutputParser()  # 无参数

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个数据助手。始终以 JSON 格式回答，包含 name 和 age 字段。"),
    ("human", "介绍{person}"),
])

chain = prompt | model | parser
data = chain.invoke({"person": "爱因斯坦"})
print(data)          # {"name": "Albert Einstein", "age": 76}
print(type(data))    # <class 'dict'>
print(data["name"])  # "Albert Einstein"
```

> **注意**：此模式下没有 `format_instructions`，完全依赖提示词中手动描述的格式。LLM 不一定严格遵守，适合原型开发。

### 模式二：带 `pydantic_object` 参数（推荐）

传入 `pydantic_object` 参数后，`JsonOutputParser` 会：
1. 根据 Pydantic 模型生成 `format_instructions`（可注入提示词）
2. 解析后用该模型做类型校验，但最终返回 `dict` 而非 Pydantic 实例

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class PersonInfo(BaseModel):
    """人物信息结构"""
    name: str = Field(description="人物全名")
    age: int = Field(description="年龄（岁）")
    nationality: str = Field(description="国籍")
    achievements: list[str] = Field(description="主要成就列表")

# 传入 pydantic_object — parser 会基于此生成 format_instructions
parser = JsonOutputParser(pydantic_object=PersonInfo)

# 将 format_instructions 注入提示词
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个百科全书助手。回答用户的问题。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
data = chain.invoke({"query": "介绍居里夫人"})

print(data)
# {
#     "name": "Marie Curie",
#     "age": 66,
#     "nationality": "波兰/法国",
#     "achievements": ["发现镭元素", "发现钋元素", "两次获得诺贝尔奖"]
# }

# 注意：返回值是 dict，不是 PersonInfo 实例
print(type(data))  # <class 'dict'>
print(data["name"])  # 用字典方式访问
```

#### `pydantic_object` 参数详解

| 属性 | 说明 |
|------|------|
| **类型** | `type[BaseModel]`，即一个 Pydantic 模型类 |
| **作用 1** | 让 `get_format_instructions()` 生成包含字段名、类型、描述的 JSON schema |
| **作用 2** | 解析 JSON 后用该模型做校验（字段类型、必填项等） |
| **返回值** | 仍然是 `dict`（校验通过后调用 `model_dump()` 返回字典） |
| **可选** | 是，不传则无 schema 约束，也不生成有意义的 format_instructions |

### 模式三：仅使用 `get_format_instructions()` 但不传 `pydantic_object`

有时你只想让 LLM 知道需要输出 JSON，但不想做严格校验：

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

model = ChatOpenAI(model="gpt-4o-mini")
parser = JsonOutputParser()

# 即使没有 pydantic_object，也可以获取基础的格式说明
instructions = parser.get_format_instructions()
print(instructions)
# 输出：
# Return a JSON object.

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=instructions)

chain = prompt | model | parser
data = chain.invoke({"query": "列出三种水果及其颜色，以 JSON 格式输出"})
print(data)  # {"fruits": [{"name": "苹果", "color": "红色"}, ...]}
```

## `get_format_instructions()` 输出示例

### 无 `pydantic_object` 时

```python
parser = JsonOutputParser()
print(parser.get_format_instructions())
# 输出：
# Return a JSON object.
```

### 有 `pydantic_object` 时

```python
from pydantic import BaseModel, Field

class Movie(BaseModel):
    """电影信息"""
    title: str = Field(description="电影名称")
    year: int = Field(description="上映年份")
    rating: float = Field(description="评分，0-10")

parser = JsonOutputParser(pydantic_object=Movie)
print(parser.get_format_instructions())
```

输出大致如下：

```
The output should be formatted as a JSON instance that conforms to the JSON schema below.

As an example, for the schema {"properties": {"foo": {"title": "Foo", "description": "a list of strings", "type": "array", "items": {"type": "string"}}}, "required": ["foo"]}
the object {"foo": ["bar", "baz"]} is valid. The object {"properties": {"foo": ["bar", "baz"]}} is not valid.

This is the schema:
{"properties": {"title": {"title": "Title", "description": "电影名称", "type": "string"}, "year": {"title": "Year", "description": "上映年份", "type": "integer"}, "rating": {"title": "Rating", "description": "评分，0-10", "type": "number"}}, "required": ["title", "year", "rating"]}
```

> **要点**：`Field(description=...)` 的内容会直接出现在 schema 中，写清楚描述能显著提高 LLM 输出准确性。

## 处理 markdown 代码块

LLM（尤其是 GPT-4o、Claude）经常将 JSON 包裹在 markdown 代码块中：

````
```json
{"name": "Einstein", "age": 76}
```
````

**`JsonOutputParser` 会自动处理这种情况**，无需手动 strip markdown：

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()

# 以下两种 LLM 输出都能正确解析：
result1 = parser.parse('{"name": "Einstein", "age": 76}')
result2 = parser.parse('```json\n{"name": "Einstein", "age": 76}\n```')

print(result1 == result2)  # True
```

解析器内部会自动检测并移除 ```` ```json ... ``` ```` 包裹，这是相比手动 `json.loads()` 的一大优势。

## 嵌套 JSON 示例

`JsonOutputParser` 能处理任意深度的嵌套 JSON：

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

# 嵌套模型定义
class Address(BaseModel):
    city: str = Field(description="城市")
    country: str = Field(description="国家")

class Company(BaseModel):
    name: str = Field(description="公司名称")
    founded: int = Field(description="成立年份")
    address: Address = Field(description="公司地址")
    products: list[str] = Field(description="主要产品列表")

parser = JsonOutputParser(pydantic_object=Company)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个商业信息助手。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
data = chain.invoke({"query": "介绍苹果公司"})

print(data)
# {
#     "name": "Apple Inc.",
#     "founded": 1976,
#     "address": {
#         "city": "Cupertino",
#         "country": "USA"
#     },
#     "products": ["iPhone", "Mac", "iPad", "Apple Watch"]
# }

# 嵌套字段访问
print(data["address"]["city"])    # "Cupertino"
print(data["products"][0])        # "iPhone"
```

## 流式输出行为

使用 `.stream()` 时，`JsonOutputParser` 的行为需要注意：

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

model = ChatOpenAI(model="gpt-4o-mini")
parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "输出 JSON：{{\"name\": ..., \"bio\": ...}}"),
    ("human", "{query}"),
])

chain = prompt | model | parser

# 流式输出
for chunk in chain.stream({"query": "介绍牛顿"}):
    print(chunk, flush=True)
```

**流式行为说明**：

| 阶段 | 输出 |
|------|------|
| JSON 未完整时 | parser 不会 yield 任何内容（因为无法解析不完整的 JSON） |
| JSON 完整后 | yield 完整的 `dict` |

> **要点**：`JsonOutputParser` 在流式模式下**不会逐 token yield 部分 JSON**。它会等待整个 JSON 字符串完整后才一次性 yield 解析结果。如果需要逐字段的流式体验，考虑使用 `with_structured_output` 的某些策略（如 `method="json_schema"`）。

## 错误处理

### OutputParserException

当 LLM 输出不是有效 JSON 时，解析器抛出 `OutputParserException`：

```python
from langchain_core.exceptions import OutputParserException
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

model = ChatOpenAI(model="gpt-4o-mini")
parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "输出 JSON 格式数据。"),
    ("human", "{query}"),
])

chain = prompt | model | parser

try:
    result = chain.invoke({"query": "介绍量子力学"})
    print(result)
except OutputParserException as e:
    # e.llm_output — LLM 的原始文本输出
    # e.observation — 人类可读的错误描述
    print(f"JSON 解析失败！")
    print(f"LLM 原始输出: {e.llm_output}")
    print(f"错误信息: {e.observation}")

    # 降级处理：尝试手动从文本中提取 JSON
    import json
    import re
    json_match = re.search(r'\{.*\}', e.llm_output, re.DOTALL)
    if json_match:
        fallback_data = json.loads(json_match.group())
        print(f"降级解析成功: {fallback_data}")
```

### 常见错误场景

```python
# 错误 1: LLM 输出了多余的解释文本
# LLM 输出: "好的，这是 JSON：\n{\"name\": \"test\"}"
# → JsonOutputParser 能处理（会尝试提取 JSON 部分）

# 错误 2: JSON 格式不合法
# LLM 输出: {"name": "test", age: 25}  ← key 没加引号
# → OutputParserException

# 错误 3: 有 pydantic_object 但字段不匹配
# LLM 输出: {"name": "test"} 但 schema 要求必须有 name, age, email
# → OutputParserException（pydantic 校验失败）
```

## JsonOutputParser vs with_structured_output 对比

| 维度 | `JsonOutputParser` | `model.with_structured_output()` |
|------|-------------------|--------------------------------|
| **返回类型** | `dict` | Pydantic 实例（或 dict，取决于参数） |
| **原理** | 文本解析（LLM 输出文本 → 提取 JSON） | 利用模型的 tool calling 或 JSON mode |
| **需要 format_instructions** | 有 pydantic_object 时需要注入 | 不需要，自动处理 |
| **可靠性** | 中等（依赖 LLM 遵循格式说明） | 高（模型原生保证结构） |
| **模型兼容性** | 任何能输出 JSON 的模型 | 需要模型支持 tool calling 或 JSON mode |
| **嵌套 JSON** | 支持，但 LLM 可能搞错深层嵌套 | 支持，模型原生处理 |
| **流式输出** | JSON 完整后才 yield | 部分策略支持逐字段流式 |
| **Markdown 代码块处理** | 自动处理 | 无需处理（模型直接输出结构化数据） |
| **使用复杂度** | 需要手动组装 prompt + parser | 一行代码搞定 |
| **推荐场景** | 不支持 tool calling 的模型、自定义解析 | 支持 tool calling 的现代模型 |

### 推荐决策

```python
# ✅ 现代推荐方式（模型支持 tool calling 时）
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")

model = ChatOpenAI(model="gpt-4o-mini")
structured_model = model.with_structured_output(Person)
person = structured_model.invoke("介绍爱因斯坦")  # 直接返回 Person 实例

# ✅ 使用 JsonOutputParser 的场景：模型不支持 tool calling
from langchain_core.output_parsers import JsonOutputParser
parser = JsonOutputParser(pydantic_object=Person)
# ... 需要手动注入 format_instructions
```

## 常见坑与注意事项

### 1. 忘记注入 format_instructions

```python
# ❌ 错误：有 pydantic_object 但没注入 format_instructions
parser = JsonOutputParser(pydantic_object=PersonInfo)
prompt = ChatPromptTemplate.from_messages([
    ("system", "回答用户问题。"),  # ← 缺少 {format_instructions}
    ("human", "{query}"),
])
# LLM 不知道需要什么格式，输出随机结构

# ✅ 正确：注入 format_instructions
prompt = ChatPromptTemplate.from_messages([
    ("system", "回答用户问题。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())
```

### 2. 混淆 dict 和 Pydantic 实例

```python
# JsonOutputParser 返回 dict，即使传了 pydantic_object
parser = JsonOutputParser(pydantic_object=PersonInfo)
data = chain.invoke({"query": "..."})
# data 是 dict，不是 PersonInfo 实例
print(data["name"][[)       # ✅ 字典访问
# print(data.name)        # ❌ AttributeError

# 如果需要 Pydantic 实例，用 PydanticOutputParser
# 见：[[langchain/modules/langchain_output_parsers/PydanticOutputParser|PydanticOutputParser]]
```

### 3. JSON key 大小写不匹配

LLM 可能输出 `{"Name": "test"}` 而 schema 期望 `{"name": "test"}`。`Field(description=...)` 中明确提及 key 名可以减少此问题。

### 4. 数值类型被输出为字符串

LLM 有时输出 `{"age": "76"}` 而非 `{"age": 76}`。Pydantic v2 有自动类型转换（coercion），但最好还是在 `Field(description=...)` 中强调类型要求。

### 5. 大 JSON 输出被截断

LLM 有 `max_tokens` 限制，大 JSON 可能被截断导致解析失败。解决方案：
- 增加 `max_tokens` 参数
- 简化 schema，减少字段数量
- 分步提取，每次只提取一部分

## 何时使用

- 需要结构化数据（dict）但不需要 Pydantic 实例
- 快速原型和实验
- 使用的模型不支持 tool calling
- 需要自定义解析逻辑（如容错处理）
- 从已有 JSON 文本中提取数据

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/PydanticOutputParser|PydanticOutputParser - 更强的类型约束]]
- [[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 配合使用自动修正]]
