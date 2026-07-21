---
tags: [langchain, output-parsers, structured-output, modern]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# with_structured_output

## 是什么

`with_structured_output()` 是 LangChain 提供的**现代结构化输出方法**。它不是传统的输出解析器类，而是通过模型的 **tool calling 机制**直接让 LLM 输出符合 Pydantic schema 的结构化数据。

```python
model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(PydanticModel)
```

## 为什么更好

| 问题 | 传统解析器 (PydanticOutputParser) | with_structured_output |
|------|----------------------------------|----------------------|
| **格式错误** | 经常发生，LLM 可能输出格式不正确的 JSON | ✅ 几乎不会发生，模型被约束输出正确 schema |
| **format_instructions** | 必须手动注入提示词 | ✅ 不需要，模型自动遵循 schema |
| **解析异常** | 需要 try/except 或 OutputFixingParser | ✅ 几乎不需要，输出保证符合 schema |
| **链的复杂度** | `prompt \| model \| parser` | ✅ `prompt \| model`（更简单） |
| **嵌套类型** | 复杂类型容易解析失败 | ✅ 原生支持复杂嵌套 |
| **可靠性** | 依赖提示词质量 | ✅ 依赖模型原生能力（更可靠） |

---

## 基础用法

### 简单 Pydantic 模型

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class Person(BaseModel):
    """人物信息"""
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")
    occupation: str = Field(description="职业")

# 创建带结构化输出的模型
model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(Person)

# 直接调用，返回 Pydantic 实例
result = model.invoke("介绍一位名叫李明的软件工程师，今年35岁")

print(result)
# name='李明' age=35 occupation='软件工程师'
print(result.name)         # 李明
print(result.age)          # 35
print(result.occupation)   # 软件工程师
print(type(result))        # <class '__main__.Person'>
```

### 在 LCEL 链中使用

```python
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

class CityInfo(BaseModel):
    """城市信息"""
    name: str = Field(description="城市名称")
    country: str = Field(description="所属国家")
    population: int = Field(description="人口数量（万）")
    landmarks: list[str] = Field(description="著名地标")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(CityInfo)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个地理助手。"),
    ("human", "介绍{city}这座城市。"),
])

# 链比传统方式更简洁：不需要 parser
chain = prompt | model

result = chain.invoke({"city": "巴黎"})
print(result.name)        # 巴黎
print(result.country)     # 法国
print(result.population)  # 215
print(result.landmarks)   # ['埃菲尔铁塔', '卢浮宫', '凯旋门']
```

---

## 嵌套模型与复杂类型

### 嵌套 Pydantic 模型

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from datetime import date

class Address(BaseModel):
    """地址信息"""
    street: str = Field(description="街道地址")
    city: str = Field(description="城市")
    country: str = Field(description="国家")

class Employee(BaseModel):
    """员工信息"""
    name: str = Field(description="姓名")
    title: str = Field(description="职位")
    hire_date: date = Field(description="入职日期")
    address: Address = Field(description="住址")
    skills: list[str] = Field(description="技能列表")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(Employee)

result = model.invoke("王小明，高级开发工程师，2022年3月入职，住在北京朝阳区建国路88号，擅长Python和Go")

print(result.name)                    # 王小明
print(result.title)                   # 高级开发工程师
print(result.hire_date)               # 2022-03-01
print(result.address.city)            # 北京
print(result.skills)                  # ['Python', 'Go']
```

### 列表类型

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class Task(BaseModel):
    """任务"""
    title: str = Field(description="任务标题")
    priority: str = Field(description="优先级：high/medium/low")
    estimated_hours: float = Field(description="预计耗时（小时）")

class TaskList(BaseModel):
    """任务列表"""
    tasks: list[Task] = Field(description="任务列表")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(TaskList)

result = model.invoke("为一个新项目列出5个关键任务步骤")

for task in result.tasks:
    print(f"[{task.priority:>6}] {task.title} ({task.estimated_hours}h)")
# [  high] 需求分析 (8.0h)
# [medium] 技术方案设计 (16.0h)
# [  high] 核心功能开发 (40.0h)
# [medium] 测试 (24.0h)
# [   low] 部署上线 (8.0h)
```

### 枚举类型

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from enum import Enum

class Difficulty(str, Enum):
    EASY = "easy"
    MEDIUM = "medium"
    HARD = "hard"

class QuizQuestion(BaseModel):
    """测验题目"""
    question: str = Field(description="题目")
    options: list[str] = Field(description="选项列表")
    correct_answer: int = Field(description="正确选项的索引（0开始）")
    difficulty: Difficulty = Field(description="难度")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(QuizQuestion)

result = model.invoke("出一道关于Python基础的选择题")

print(result.question)
for i, opt in enumerate(result.options):
    print(f"  {i}. {opt}")
print(f"答案: {result.correct_answer}, 难度: {result.difficulty.value}")
```

---

## Union 类型：多种输出类型

### 使用 Union 支持不同输出结构

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from typing import Union

class SuccessResponse(BaseModel):
    """成功响应"""
    status: str = Field(description="状态：success")
    data: str = Field(description="返回的数据")
    confidence: float = Field(description="置信度")

class ErrorResponse(BaseModel):
    """错误响应"""
    status: str = Field(description="状态：error")
    error_message: str = Field(description="错误信息")
    suggested_action: str = Field(description="建议操作")

class APIResponse(BaseModel):
    """API 响应，可以是成功或错误"""
    response: Union[SuccessResponse, ErrorResponse] = Field(
        description="响应内容"
    )

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(APIResponse)

# 成功的场景
result = model.invoke("查询北京今天的天气")
print(result.response)

# 可能返回 SuccessResponse 或 ErrorResponse
```

### 使用 Literal 实现简单分类

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from typing import Literal

class SentimentResult(BaseModel):
    """情感分析结果"""
    text: str = Field(description="原始文本")
    sentiment: Literal["positive", "negative", "neutral"] = Field(
        description="情感标签"
    )
    confidence: float = Field(description="置信度 0-1")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(SentimentResult)

result = model.invoke("这家餐厅太棒了，强烈推荐！")
print(result.sentiment)    # positive
print(result.confidence)   # 0.95
```

---

## 与传统解析器的对比

### 同一任务的两种实现方式

**传统方式：PydanticOutputParser + OutputFixingParser**

```python
from pydantic import BaseModel, Field
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain.output_parsers import OutputFixingParser
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

class BookRecommendation(BaseModel):
    title: str = Field(description="书名")
    author: str = Field(description="作者")
    reason: str = Field(description="推荐理由")

# 步骤1：创建解析器
base_parser = PydanticOutputParser(pydantic_object=BookRecommendation)

# 步骤2：包装为修正解析器
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

# 步骤3：在提示词中注入 format_instructions
prompt = ChatPromptTemplate.from_messages([
    ("system", "推荐书籍。{format_instructions}"),
    ("human", "推荐一本关于{topic}的书。"),
])

# 步骤4：构建链
chain = prompt | model | fixing_parser

# 步骤5：调用时必须传入 format_instructions
result = chain.invoke({
    "topic": "Python",
    "format_instructions": base_parser.get_format_instructions(),
})
```

**现代方式：with_structured_output**

```python
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

class BookRecommendation(BaseModel):
    title: str = Field(description="书名")
    author: str = Field(description="作者")
    reason: str = Field(description="推荐理由")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(BookRecommendation)

prompt = ChatPromptTemplate.from_messages([
    ("system", "推荐书籍。"),
    ("human", "推荐一本关于{topic}的书。"),
])

# 链更简洁：不需要 parser
chain = prompt | model

# 调用时不需要 format_instructions
result = chain.invoke({"topic": "Python"})
print(result.title)   # 'Python Crash Course'
print(result.author)  # 'Eric Matthes'
```

### 对比总结

| 方面 | 传统方式 | with_structured_output |
|------|---------|----------------------|
| 代码行数 | ~15行 | ~10行 |
| 需要导入 | 4-5个模块 | 2-3个模块 |
| format_instructions | 必须管理 | 不需要 |
| 错误修正 | 需要 OutputFixingParser | 通常不需要 |
| 链结构 | `prompt \| model \| parser` | `prompt \| model` |
| 可靠性 | 中等 | 高 |
| 适用模型 | 所有模型 | 仅支持 tool calling 的模型 |

---

## 支持的模型

`with_structured_output` 依赖模型的 **tool calling / function calling** 能力：

| 模型提供商 | 支持情况 | 说明 |
|-----------|---------|------|
| **OpenAI** (gpt-4o, gpt-4o-mini 等) | ✅ 完全支持 | 原生 tool calling |
| **Anthropic** (Claude 3.5/4) | ✅ 完全支持 | 原生 tool use |
| **Google** (Gemini) | ✅ 完全支持 | 原生 function calling |
| **Mistral** | ✅ 支持 | 部分模型支持 tool calling |
| **开源模型** (Llama, Qwen 等) | ⚠️ 视情况 | 需要支持 tool calling 的部署 |
| **不支持 tool calling 的模型** | ❌ | 回退到传统解析器 |

### 检查模型是否支持

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# 尝试使用 with_structured_output
try:
    structured_model = model.with_structured_output(MyModel)
    print("支持结构化输出")
except Exception as e:
    print(f"不支持: {e}")
    # 回退到传统解析器
```

---

## 何时使用 vs 何时不使用

### ✅ 适合使用的场景

```python
# 1. 需要可靠的结构化输出
class ExtractionResult(BaseModel):
    entities: list[str]
    relationships: list[dict]

# 2. 复杂的嵌套数据结构
class NestedData(BaseModel):
    sections: list[Section]  # Section 也是 Pydantic 模型

# 3. 生产环境，不能容忍解析失败
# 4. 想要简化代码，减少 format_instructions 管理
```

### ❌ 不适合的场景

```python
# 1. 模型不支持 tool calling（如某些本地部署的开源模型）
#    → 使用 PydanticOutputParser + OutputFixingParser

# 2. 只需要纯文本输出
#    → 使用 StrOutputParser

# 3. 需要流式输出结构化的每个字段（with_structured_output 的流式支持有限）
#    → 使用传统解析器 + stream

# 4. 需要极度控制提示词格式（如特定的 JSON schema）
#    → 使用 JsonOutputParser
```

---

## 异步用法

```python
import asyncio
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

class Translation(BaseModel):
    original: str = Field(description="原文")
    translated: str = Field(description="译文")
    language: str = Field(description="目标语言")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(Translation)

prompt = ChatPromptTemplate.from_messages([
    ("system", "翻译文本。"),
    ("human", "将以下内容翻译成{language}：{text}"),
])

chain = prompt | model

async def main():
    # 异步单次调用
    result = await chain.ainvoke({
        "language": "英语",
        "text": "今天天气真好",
    })
    print(result.translated)  # "The weather is really nice today"

    # 异步批量调用
    results = await chain.abatch([
        {"language": "日语", "text": "你好"},
        {"language": "法语", "text": "谢谢"},
        {"language": "德语", "text": "再见"},
    ])
    for r in results:
        print(f"{r.language}: {r.translated}")

asyncio.run(main())
```

---

## 流式输出

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class Story(BaseModel):
    title: str = Field(description="标题")
    content: str = Field(description="故事内容")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(Story)

# with_structured_output 的流式输出返回的是部分构建的对象
# 注意：不是所有模型都支持结构化流式
for chunk in model.stream("写一个关于AI的短故事"):
    print(chunk)
    # 每次 chunk 是部分填充的 Story 对象
```

---

## 实用技巧

### 技巧1：Pydantic Field 的 description 很重要

```python
# ❌ 没有描述：模型只能靠字段名猜测
class Bad(BaseModel):
    x: float
    y: float

# ✅ 有描述：模型知道要填什么
class Good(BaseModel):
    latitude: float = Field(description="纬度，如 39.9042")
    longitude: float = Field(description="经度，如 116.4074")
```

### 技巧2：提供枚举约束输出范围

```python
from enum import Enum
from pydantic import BaseModel, Field

class Status(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"

class UserStatus(BaseModel):
    user_id: str = Field(description="用户ID")
    status: Status = Field(description="用户状态")

# 模型的 status 字段只能是 "active" 或 "inactive"
```

### 技巧3：Optional 字段

```python
from pydantic import BaseModel, Field
from typing import Optional

class Contact(BaseModel):
    name: str = Field(description="姓名")
    phone: str = Field(description="电话号码")
    email: Optional[str][[ = Field(default=None, description="邮箱，如果没有则为null")
```

---

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/PydanticOutputParser|PydanticOutputParser - 传统替代方案]]
- [[langchain/modules/langchain_output_parsers/JsonOutputParser|JsonOutputParser - JSON 解析]]
- [[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 传统错误修正]]
- [[langchain/modules/langchain_output_parsers/错误处理|错误处理]]
