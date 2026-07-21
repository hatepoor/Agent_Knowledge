---
tags: [langchain, output-parsers, enum]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# EnumOutputParser

## 是什么

`EnumOutputParser` 将 LLM 的输出解析为 **Python 枚举值** (`Enum` 成员)，适用于需要从预定义选项中做离散选择的场景，如分类、情感分析、路由决策等。

## 导入

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum
```

---

## 构造函数

```python
EnumOutputParser(enum: Type[Enum])
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `enum` | `Type[Enum]` | 必填。Python 枚举类，解析器会根据其 `value` 值来匹配 LLM 输出 |

**关键点：** 解析器匹配的是枚举的 **`value`**，而非 `name`。LLM 输出的文本必须与某个枚举成员的 `value` 完全一致（去除首尾空白后）。

---

## 基础用法

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Color(Enum):
    RED = "red"
    GREEN = "green"
    BLUE = "blue"

parser = EnumOutputParser(enum=Color)

prompt = ChatPromptTemplate.from_messages([
    ("system", "根据描述选择最合适的颜色。{format_instructions}"),
    ("human", "{description}"),
])

chain = prompt | model | parser

result = chain.invoke({
    "description": "和平、自然、森林",
    "format_instructions": parser.get_format_instructions(),
})

print(result)          # Color.GREEN
print(result.value)    # green
print(type(result))    # <enum 'Color'>
```

---

## get_format_instructions() 输出

`get_format_instructions()` 生成的指令会将所有枚举值展示给 LLM，告诉它只能从中选择一个：

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum

class Sentiment(Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

parser = EnumOutputParser(enum=Sentiment)
print(parser.get_format_instructions())
```

输出类似：

```
Your response should be one of the following values: positive, negative, neutral
```

**注意：** 格式指令中展示的是枚举的 `value`，不是 `name`。因此 `value` 应该是 LLM 能理解的有意义的字符串。

---

## 中文枚举值

枚举的 `value` 可以是中文，适合处理中文 LLM 输出：

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class 情感(Enum):
    正面 = "正面"
    负面 = "负面"
    中性 = "中性"

parser = EnumOutputParser(enum=情感)

prompt = ChatPromptTemplate.from_messages([
    ("system", "分析以下文本的情感倾向，只输出一个情感标签。{format_instructions}"),
    ("human", "{text}"),
])

chain = prompt | model | parser

result = chain.invoke({
    "text": "这家餐厅的食物非常美味，服务也很棒！",
    "format_instructions": parser.get_format_instructions(),
})

print(result)        # 情感.正面
print(result.value)  # 正面
```

**中文枚举的格式指令输出：**

```python
print(parser.get_format_instructions())
# Your response should be one of the following values: 正面, 负面, 中性
```

---

## 错误处理

### 错误1：LLM 输出了不在枚举范围内的值

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum

class Priority(Enum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

parser = EnumOutputParser(enum=Priority)

try:
    result = parser.parse("urgent")
except Exception as e:
    print(f"解析失败: {e}")
    # OutputParserException: 'urgent' is not a valid Priority
```

`EnumOutputParser` 会抛出 `OutputParserException`，而不是 `ValueError`。这使得 `OutputFixingParser` 能够捕获并自动修正。

### 错误2：大小写不匹配

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum

class Status(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"

parser = EnumOutputParser(enum=Status)

# ✅ 正确匹配
result = parser.parse("active")
print(result)  # Status.ACTIVE

# ❌ 大小写不匹配
try:
    result = parser.parse("Active")
except Exception as e:
    print(f"解析失败: {e}")
    # OutputParserException: 'Active' is not a valid Status

# ❌ 全大写也不匹配
try:
    result = parser.parse("ACTIVE")
except Exception as e:
    print(f"解析失败: {e}")
```

**解决方案：在枚举中包含常见的大小写变体**

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum

class Status(Enum):
    ACTIVE = "active"
    ACTIVE_UPPER = "ACTIVE"    # 兼容大写
    ACTIVE_TITLE = "Active"    # 兼容首字母大写
    INACTIVE = "inactive"
    INACTIVE_UPPER = "INACTIVE"
    INACTIVE_TITLE = "Inactive"

parser = EnumOutputParser(enum=Status)

# 现在可以处理不同大小写
result = parser.parse("Active")
print(result)  # Status.ACTIVE_TITLE

# 在代码中统一处理：
def normalize_status(status: Status) -> str:
    return status.value.lower()
```

### 错误3：LLM 输出了额外的文字

```python
from langchain_core.output_parsers import EnumOutputParser
from enum import Enum

class Answer(Enum):
    YES = "yes"
    NO = "no"

parser = EnumOutputParser(enum=Answer)

# ❌ LLM 输出了完整句子而非单个值
try:
    result = parser.parse("The answer is yes")
except Exception as e:
    print(f"解析失败: {e}")
```

**解决方案：在提示词中明确要求只输出枚举值本身**

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "只输出一个值，不要任何解释文字。\n{format_instructions}"),
    ("human", "{question}"),
])
```

---

## 与 OutputFixingParser 组合使用

`OutputFixingParser` 可以在解析失败时自动调用 LLM 修正输出，是提高 `EnumOutputParser` 可靠性的推荐方式：

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain.output_parsers import OutputFixingParser
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Category(Enum):
    TECH = "tech"
    FINANCE = "finance"
    HEALTH = "health"
    EDUCATION = "education"

base_parser = EnumOutputParser(enum=Category)
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

prompt = ChatPromptTemplate.from_messages([
    ("system", "将文章分类。{format_instructions}"),
    ("human", "{article}"),
][[)

chain = prompt | model | fixing_parser

# 即使 LLM 首次输出了 "technology" 而非 "tech"，
# OutputFixingParser 会自动修正为 "tech"
result = chain.invoke({
    "article": "最新的AI芯片性能提升了30%",
    "format_instructions": base_parser.get_format_instructions(),
})

print(result)        # Category.TECH
print(result.value)  # tech
```

参见：[[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 自动修正解析错误]]

---

## 实用场景

### 场景1：情感分析

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Sentiment(Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

parser = EnumOutputParser(enum=Sentiment)

prompt = ChatPromptTemplate.from_messages([
    ("system", "分析文本的情感倾向。只输出一个标签。\n{format_instructions}"),
    ("human", "{text}"),
])

chain = prompt | model | parser

reviews = [
    "这家餐厅太棒了，强烈推荐！",
    "产品收到了，但质量很差，很失望。",
    "快递按时到达了。",
]

for review in reviews:
    result = chain.invoke({
        "text": review,
        "format_instructions": parser.get_format_instructions(),
    })
    print(f"[{result.value:>8}] {review}")
# [positive] 这家餐厅太棒了，强烈推荐！
# [negative] 产品收到了，但质量很差，很失望。
# [neutral] 快递按时到达了。
```

### 场景2：文本分类

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Intent(Enum):
    QUERY = "query"          # 查询
    COMMAND = "command"      # 命令
    CHITCHAT = "chitchat"    # 闲聊
    COMPLAINT = "complaint"  # 投诉

parser = EnumOutputParser(enum=Intent)

prompt = ChatPromptTemplate.from_messages([
    ("system", "判断用户消息的意图类型。{format_instructions}"),
    ("human", "{message}"),
])

chain = prompt | model | parser

result = chain.invoke({
    "message": "帮我查一下北京明天的天气",
    "format_instructions": parser.get_format_instructions(),
})
print(result)  # Intent.QUERY
```

### 场景3：路由决策

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum
from langchain_core.runnables import RunnableBranch

model = ChatOpenAI(model="gpt-4o-mini")

class Route(Enum):
    SALES = "sales"
    SUPPORT = "support"
    GENERAL = "general"

parser = EnumOutputParser(enum=Route)

classify_prompt = ChatPromptTemplate.from_messages([
    ("system", "将用户消息路由到正确的部门。{format_instructions}"),
    ("human", "{message}"),
])

classify_chain = classify_prompt | model | parser

def handle_sales(input: dict) -> str:
    return "已转接到销售部门"

def handle_support(input: dict) -> str:
    return "已转接到技术支持"

def handle_general(input: dict) -> str:
    return "已转接到总机"

def route(input: dict) -> str:
    category = classify_chain.invoke({
        "message": input["message"],
        "format_instructions": parser.get_format_instructions(),
    })
    if category == Route.SALES:
        return handle_sales(input)
    elif category == Route.SUPPORT:
        return handle_support(input)
    else:
        return handle_general(input)

result = route({"message": "我想购买你们的企业版"})
print(result)  # 已转接到销售部门
```

### 场景4：批量情感分析

```python
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Sentiment(Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

parser = EnumOutputParser(enum=Sentiment)

prompt = ChatPromptTemplate.from_messages([
    ("system", "分析情感。{format_instructions}"),
    ("human", "{text}"),
])

chain = prompt | model | parser

format_inst = parser.get_format_instructions()
results = chain.batch([
    {"text": "太好用了！", "format_instructions": format_inst},
    {"text": "浪费了钱", "format_instructions": format_inst},
    {"text": "还行吧", "format_instructions": format_inst},
][[)

for r in results:
    print(r.value)
```

---

## 实用技巧

### 技巧1：在 if 判断中使用

```python
# 枚举比字符串比较更安全
result = chain.invoke({...})

# ✅ 类型安全：IDE 能自动补全，拼写错误会被编译器发现
if result == Sentiment.POSITIVE:
    print("正面评价")

# ❌ 字符串比较容易拼写错误
if result == "positve":  # 拼错了！但不会报错
    print("正面评价")
```

### 技巧2：枚举值设计建议

```python
# ✅ 好的枚举值：简洁、小写、无歧义
class Topic(Enum):
    SPORTS = "sports"
    POLITICS = "politics"
    TECHNOLOGY = "technology"

# ❌ 避免：过长、有空格、大小写混乱
class Topic(Enum):
    SPORTS_AND_FITNESS = "Sports and Fitness"   # 太长，LLM 容易写错
    TECH = "Tech"                                 # 大小写不一致
    POLI = "poli"                                 # 缩写不直观
```

### 技巧3：何时考虑 with_structured_output

如果你的分类任务更复杂（需要同时输出类别 + 置信度 + 理由），考虑使用 `with_structured_output`：

```python
from pydantic import BaseModel, Field
from enum import Enum
from langchain_openai import ChatOpenAI

class Category(str, Enum):
    TECH = "tech"
    FINANCE = "finance"

class Classification(BaseModel):
    category: Category = Field(description="文章分类")
    confidence: float = Field(description="置信度 0-1")
    reason: str = Field(description="分类理由")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(Classification)
result = model.invoke("最新的AI芯片性能提升了30%")
print(result.category)    # Category.TECH
print(result.confidence)  # 0.95
print(result.reason)      # "文章讨论的是AI芯片技术"
```

参见：[[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]

---

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 自动修正解析错误]]
- [[langchain/modules/langchain_output_parsers/ListOutputParser|ListOutputParser - 列表解析]]
- [[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]
- [[langchain/modules/langchain_output_parsers/错误处理|错误处理]]
