---
tags: [langchain, output-parsers, pydantic]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# PydanticOutputParser

## 是什么

`PydanticOutputParser` 将 LLM 的输出解析为 **Pydantic 模型实例**，提供强类型校验和字段约束，是最严谨的结构化输出解析方式。解析成功后返回的是 Pydantic `BaseModel` 实例，支持属性访问（`obj.field`）而非字典访问（`obj["field"]`）。

## 导入

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field
```

## 基础用法

```python
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Person(BaseModel):
    """人物信息"""
    name: str = Field(description="人物全名")
    age: int = Field(description="年龄（岁）")
    occupation: str = Field(description="职业")

parser = PydanticOutputParser(pydantic_object=Person)

prompt = ChatPromptTemplate.from_messages([
    ("system", "回答用户的问题。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
person = chain.invoke({"query": "介绍玛丽·居里"})

# 返回的是 Person 实例，支持属性访问
print(person.name)        # "Marie Curie"
print(person.age)         # 66
print(person.occupation)  # "物理学家/化学家"
print(type(person))       # <class '__main__.Person'>
```

### 关键点

1. **定义 Pydantic 模型**：用 `BaseModel` 定义期望的字段和类型
2. **Field description**：字段的 `description` 会作为格式指令传给 LLM，帮助其理解需要什么输出
3. **get_format_instructions()**：生成包含完整 JSON schema 的格式说明文本，**必须注入提示词中**
4. **类型校验**：解析后的对象自动通过 Pydantic 校验，字段缺失或类型错误会抛异常

## 嵌套模型

Pydantic 支持模型嵌套，`PydanticOutputParser` 能正确处理复杂的层级结构：

```python
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Education(BaseModel):
    """教育经历"""
    school: str = Field(description="学校名称")
    degree: str = Field(description="学位，如学士/硕士/博士")
    major: str = Field(description="专业")
    year: int = Field(description="毕业年份")

class WorkExperience(BaseModel):
    """工作经历"""
    company: str = Field(description="公司名称")
    role: str = Field(description="职位")
    years: int = Field(description="在职年数")

class Resume(BaseModel):
    """简历信息"""
    name: str = Field(description="姓名")
    email: str = Field(description="邮箱地址")
    education: Education = Field(description="最高学历")
    experiences: list[WorkExperience] = Field(description="工作经历列表")
    skills: list[str] = Field(description="技能列表")

parser = PydanticOutputParser(pydantic_object=Resume)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个简历解析助手。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
resume = chain.invoke({"query": "生成一个虚构的软件工程师简历"})

# 嵌套属性访问
print(resume.name)
print(resume.education.school)         # 嵌套对象
print(resume.experiences[0].company)   # 列表中的嵌套对象
print(resume.skills)                   # 字符串列表
```

## 可选字段、默认值与验证器

### 可选字段（Optional）

```python
from pydantic import BaseModel, Field
from typing import Optional

class Product(BaseModel):
    """产品信息"""
    name: str = Field(description="产品名称")
    price: float = Field(description="价格（元）")
    description: Optional[str] = Field(default=None, description="产品描述，可选")
    brand: Optional[str] = Field(default=None, description="品牌名，可选")
    tags: list[str] = Field(default_factory=list, description="标签列表，默认为空")
```

### 默认值

```python
class Config(BaseModel):
    """系统配置"""
    language: str = Field(default="zh-CN", description="语言设置")
    page_size: int = Field(default=20, description="每页条数")
    theme: str = Field(default="light", description="主题：light 或 dark")
    enable_notifications: bool = Field(default=True, description="是否启用通知")
```

> **注意**：有默认值的字段在 `format_instructions` 中会标记为非必填。LLM 可以省略这些字段，Pydantic 会使用默认值。

### 字段约束（validators）

Pydantic v2 的约束参数会被包含在 `format_instructions` 的 JSON schema 中，LLM 可以看到这些约束：

```python
from pydantic import BaseModel, Field

class Review(BaseModel):
    """用户评价"""
    title: str = Field(description="评价标题", min_length=1, max_length=100)
    rating: int = Field(description="评分", ge=1, le=5)
    content: str = Field(description="评价内容", min_length=10)
    recommend: bool = Field(description="是否推荐")

# get_format_instructions() 输出的 schema 会包含：
# "rating": {"type": "integer", "minimum": 1, "maximum": 5}
# LLM 能看到这些约束，更可能输出合法值
```

### 自定义验证器（Pydantic v2）

```python
from pydantic import BaseModel, Field, field_validator

class UserInput(BaseModel):
    """用户输入"""
    email: str = Field(description="邮箱地址")
    age: int = Field(description="年龄", ge=0, le=150)

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("邮箱必须包含 @")
        return v.lower()

# 注意：自定义验证器的错误会触发 OutputParserException
# LLM 看不到 @field_validator 的逻辑，只看到 schema 中的字段描述
```

> **重要**：`@field_validator` 中的逻辑**不会**出现在 `format_instructions` 中，LLM 无法感知。如果验证规则对 LLM 输出很重要，应在 `Field(description=...)` 中显式说明，例如 `Field(description="邮箱地址，必须包含 @ 符号")`。

## `get_format_instructions()` 输出示例

```python
from pydantic import BaseModel, Field
from langchain_core.output_parsers import PydanticOutputParser

class Movie(BaseModel):
    """电影信息"""
    title: str = Field(description="电影名称")
    year: int = Field(description="上映年份")
    rating: float = Field(description="评分，0-10分", ge=0, le=10)

parser = PydanticOutputParser(pydantic_object=Movie)
print(parser.get_format_instructions())
```

输出内容：

```
The output should be formatted as a JSON instance that conforms to the JSON schema below.

As an example, for the schema {"properties": {"foo": {"title": "Foo", "description": "a list of strings", "type": "array", "items": {"type": "string"}}}, "required": ["foo"]}
the object {"foo": ["bar", "baz"]} is valid. The object {"properties": {"foo": ["bar", "baz"]}} is not valid.

This is the schema:
{"$defs": {"Movie": {"title": "Movie", "description": "电影信息", "type": "object", "properties": {"title": {"title": "Title", "description": "电影名称", "type": "string"}, "year": {"title": "Year", "description": "上映年份", "type": "integer"}, "rating": {"title": "Rating", "description": "评分，0-10分", "type": "number", "minimum": 0, "maximum": 10}}, "required": ["title", "year", "rating"]}}}
```

> **要点**：
> - `Field(description=...)` 中的文字直接出现在 schema 的 `description` 字段中
> - 约束如 `ge=0, le=10` 会转为 `minimum` / `maximum`
> - 这个 schema 文本会消耗 token，模型越复杂 token 消耗越大

## 错误处理

### OutputParserException

解析失败时抛出 `OutputParserException`，包含详细的错误上下文：

```python
from langchain_core.exceptions import OutputParserException
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Answer(BaseModel):
    result: str = Field(description="答案")
    confidence: float = Field(description="置信度 0-1", ge=0, le=1)

parser = PydanticOutputParser(pydantic_object=Answer)

prompt = ChatPromptTemplate.from_messages([
    ("system", "回答问题。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser

try:
    answer = chain.invoke({"query": "地球到月球的距离？"})
    print(f"答案: {answer.result}, 置信度: {answer.confidence}")
except OutputParserException as e:
    print("=== 解析失败 ===")
    print(f"错误类型: {type(e).__name__}")
    print(f"LLM 原始输出:\n{e.llm_output}")
    print(f"错误观察: {e.observation}")
    # observation 通常包含类似：
    # "Failed to parse Answer from completion {...}. Got: ValidationError(...)"
```

### 错误消息的常见模式

| 错误原因 | observation 中的关键信息 |
|---------|------------------------|
| LLM 输出不是 JSON | `"Expecting value"` / `"Invalid JSON"` |
| 必填字段缺失 | `"field required"` / `"missing"` |
| 字段类型不匹配 | `"Input should be a valid integer"` / `"type mismatch"` |
| 约束违反（如 ge/le） | `"Input should be less than or equal to 1"` |
| 自定义验证器失败 | `"Value error, ..."` |
| LLM 输出了 markdown 包裹 | 通常不会报错（parser 自动处理） |

### 带重试的容错模式

```python
from langchain_core.output_parsers import PydanticOutputParser, OutputFixingParser
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Result(BaseModel):
    answer: str = Field(description="答案")
    score: int = Field(description="评分 1-5", ge=1, le=5)

base_parser = PydanticOutputParser(pydantic_object=Result)

# 用 OutputFixingParser 包裹：解析失败时自动让 LLM 修正
fixing_parser = OutputFixingParser.from_llm(
    parser=base_parser,
    llm=model,
    max_retries=3,        # 最多重试 3 次
    retry_chain=None,     # 可以用更强的模型做修正
)

# 使用方式和普通 parser 一样
# fixing_parser 会自动处理解析失败的情况
```

## PydanticOutputParser vs with_structured_output 对比

这是 LangChain 中获取结构化输出的两种方式，理解它们的差异很重要：

### 工作原理对比

```python
# ===== 方式 1: PydanticOutputParser（手动解析）=====
# 流程：生成 format_instructions → 注入提示词 → LLM 输出文本 → parser 解析文本为 Pydantic 实例
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

class Joke(BaseModel):
    setup: str = Field(description="笑话铺垫")
    punchline: str = Field(description="笑点")

model = ChatOpenAI(model="gpt-4o-mini")
parser = PydanticOutputParser(pydantic_object=Joke)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个喜剧演员。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
joke = chain.invoke({"query": "讲个程序员笑话"})

# ===== 方式 2: with_structured_output（模型原生能力）=====
# 流程：模型利用 tool calling / JSON mode → 直接输出结构化数据 → 自动转为 Pydantic 实例
structured_model = model.with_structured_output(Joke)
joke = structured_model.invoke("讲个程序员笑话")
# 无需手动写 prompt，无需 format_instructions
```

### 详细对比表

| 维度 | `PydanticOutputParser` | `model.with_structured_output()` |
|------|----------------------|--------------------------------|
| **底层原理** | 提示词注入 schema → LLM 输出文本 → 解析 | 利用模型的 function calling / JSON mode |
| **可靠性** | 中等（LLM 可能不遵循 schema） | 高（模型原生保证输出结构正确） |
| **需要 format_instructions** | 是，必须手动注入 | 否，完全自动 |
| **提示词 token 消耗** | 较多（JSON schema 很长） | 少（schema 通过 API 传递，不占用内容 token） |
| **模型要求** | 任何能输出 JSON 的模型 | 需要模型支持 tool calling 或 JSON mode |
| **支持模型** | 所有 LLM | OpenAI、Anthropic、Google、Mistral 等主流模型 |
| **流式输出** | JSON 完整后才 yield | 部分策略支持逐字段流式 |
| **自定义提示词** | 完全控制（可自定义 system prompt） | 有限（structured output 内部组装 prompt） |
| **嵌套模型** | 支持，但 LLM 可能搞错深层嵌套 | 支持，模型原生处理嵌套 |
| **错误恢复** | 需要手动包裹 OutputFixingParser | 模型层面保证，极少出错 |
| **调试友好度** | 好（可以看到完整 prompt 和 LLM 输出） | 一般（内部封装较深） |
| **代码量** | 较多（定义 parser + 注入 instructions + 组装 chain） | 极少（一行 `with_structured_output`） |

### 推荐方案

```
你的模型支持 tool calling 吗？（GPT-4o、Claude、Gemini 等都支持）
│
├── 是 → 优先使用 with_structured_output
│   ├── 代码更简洁
│   ├── 可靠性更高
│   └── 不需要担心 format_instructions
│
└── 否 → 使用 PydanticOutputParser
    ├── 适用于不支持 tool calling 的模型（如某些开源模型）
    ├── 需要完全自定义提示词的场景
    └── 需要复杂的 prompt engineering 控制
```

```python
# ✅ 推荐方式：with_structured_output（适用于支持 tool calling 的模型）
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class Article(BaseModel):
    title: str = Field(description="文章标题")
    summary: str = Field(description="文章摘要")
    keywords: list[str] = Field(description="关键词列表")

model = ChatOpenAI(model="gpt-4o-mini")
structured_model = model.with_structured_output(Article)
article = structured_model.invoke("写一篇关于 AI 的文章大纲")
print(article.title, article.summary, article.keywords)

# ✅ 使用 PydanticOutputParser 的场景：
# 1. 模型不支持 tool calling
# 2. 需要精细控制提示词
# 3. 在 chain 中需要自定义中间步骤
from langchain_core.output_parsers import PydanticOutputParser
from langchain.prompts import ChatPromptTemplate

parser = PydanticOutputParser(pydantic_object=Article)
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个专业的内容创作者。请用学术风格回答。\n{format_instructions}"),
    ("human", "请写一篇关于{topic}的文章大纲"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
article = chain.invoke({"topic": "AI"})
```

## 配合 Agent 使用（v1 推荐方式）

```python
from langchain.agents import create_agent
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class Answer(BaseModel):
    summary: str = Field(description="简洁的回答摘要")
    confidence: float = Field(description="置信度 0-1", ge=0, le=1)
    sources: list[str] = Field(description="信息来源列表")

agent = create_agent(
    model,
    tools=[],
    response_format=Answer,  # 自动选择 ProviderStrategy 或 ToolStrategy
)
result = agent.invoke({
    "messages": [{"role": "user", "content": "总结 AI 发展趋势"}]
})
print(result["structured_response"])  # Answer 实例
```

## 常见坑与注意事项

### 1. 忘记注入 format_instructions

```python
# ❌ 错误：定义了 parser 但没注入 format_instructions
parser = PydanticOutputParser(pydantic_object=Person)
prompt = ChatPromptTemplate.from_messages([
    ("system", "回答用户问题。"),   # ← 缺少 {format_instructions}
    ("human", "{query}"),
])
# LLM 不知道需要什么格式，输出随机文本，parser 必然失败

# ✅ 正确：
prompt = ChatPromptTemplate.from_messages([
    ("system", "回答用户问题。\n{format_instructions}"),
    ("human", "{query}"),
]).partial(format_instructions=parser.get_format_instructions())
```

### 2. Field description 不够清晰

```python
# ❌ 模糊描述：LLM 不知道具体要什么
class Data(BaseModel):
    value: str = Field(description="值")       # 什么值？什么格式？
    count: int = Field(description="数量")      # 什么的数量？

# ✅ 清晰描述：LLM 能准确输出
class Data(BaseModel):
    value: str = Field(description="用户输入的搜索关键词")
    count: int = Field(description="搜索结果的数量，正整数")
```

### 3. 字段缺失导致解析失败

```python
# 如果 schema 中所有字段都是 required（没有默认值），
# LLM 必须输出所有字段，少一个就报 OutputParserException

# 解决方案：对非核心字段使用 Optional + 默认值
class FlexibleData(BaseModel):
    name: str = Field(description="名称")                          # 必填
    category: str = Field(default="未分类", description="分类")     # 有默认值，LLM 可省略
    notes: str | None = Field(default=None, description="备注")    # 可选
```

### 4. 类型不匹配

```python
# LLM 有时把数字输出为字符串："age": "25" 而非 "age": 25
# Pydantic v2 有自动类型强制转换（coercion），能处理这种情况
# 但最好还是在 description 中强调类型

class Data(BaseModel):
    # ✅ 在描述中明确类型要求
    score: float = Field(description="分数，必须是浮点数如 8.5，不要输出字符串")
    tags: list[str][[ = Field(description="标签列表，必须是数组格式")
```

### 5. format_instructions 消耗大量 token

`get_format_instructions()` 生成的 JSON schema 文本可能很长，尤其对于嵌套模型。这会：
- 增加 API 调用的 token 消耗
- 占用 LLM 的上下文窗口
- 可能影响 LLM 的回答质量

**缓解策略**：
- 只定义必要的字段
- 简化 description，避免冗长
- 减少嵌套层级
- 考虑使用 `with_structured_output`（schema 通过 API 传递，不占用内容 token）

### 6. Pydantic v1 vs v2 兼容性

```python
# LangChain 目前同时支持 Pydantic v1 和 v2
# 但推荐使用 v2，因为类型推断更准确

# Pydantic v2 写法（推荐）
from pydantic import BaseModel, Field
class Model(BaseModel):
    name: str = Field(description="...")

# 如果用 v1，某些写法不同
from pydantic.v1 import BaseModel, Field  # 显式导入 v1
```

## 何时使用

| 场景 | 是否推荐 |
|------|---------|
| 模型支持 tool calling 的结构化输出 | ❌ 用 `with_structured_output` 更好 |
| 模型不支持 tool calling | ✅ 这是最佳选择 |
| 需要完全控制提示词 | ✅ 可以精细调整 format_instructions |
| 需要 Pydantic 实例（非 dict） | ✅ 直接返回模型实例 |
| 复杂嵌套数据结构 | ✅ 支持，但注意 token 消耗 |
| 快速原型、不在乎类型 | ❌ 用 `JsonOutputParser` 更轻量 |
| Agent 的 response_format | ✅ `create_agent(response_format=Model)` |

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/JsonOutputParser|JsonOutputParser - 更轻量的 JSON 解析]]
- [[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 配合使用修正错误]]
- [[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代结构化输出方案（推荐）]]
