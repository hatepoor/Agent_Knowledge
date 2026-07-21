---
tags: [langchain, output-parsers, error-fixing]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# OutputFixingParser

## 是什么

`OutputFixingParser` 是一个**包装器解析器**。当底层解析器解析失败时，它会**再次调用 LLM 来修正错误输出**，从而显著提高解析成功率。它不单独使用，总是包装另一个解析器。

## 导入

```python
from langchain.output_parsers import OutputFixingParser
```

---

## 构造函数：from_llm

```python
OutputFixingParser.from_llm(
    llm: BaseChatModel,       # 用于修正错误的 LLM 实例
    parser: BaseOutputParser,  # 底层解析器
) -> OutputFixingParser
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `llm` | `BaseChatModel` | 是 | 用于修正输出的 LLM，可以和主链用同一个模型 |
| `parser` | `BaseOutputParser` | 是 | 底层解析器，如 `PydanticOutputParser`、`EnumOutputParser` 等 |

**注意：** 必须使用 `from_llm()` 类方法创建实例，不能直接 `OutputFixingParser()`。

---

## 完整端到端示例

```python
from langchain.output_parsers import OutputFixingParser
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

# 1. 定义 Pydantic 模型
class MovieReview(BaseModel):
    """电影评论的结构化数据"""
    title: str = Field(description="电影名称")
    rating: float = Field(description="评分，1-10分")
    summary: str = Field(description="一句话总结")
    recommend: bool = Field(description="是否推荐")

# 2. 创建底层解析器
base_parser = PydanticOutputParser(pydantic_object=MovieReview)

# 3. 用 OutputFixingParser 包装
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

# 4. 构建完整链
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个影评人。{format_instructions}"),
    ("human", "给电影《{movie}》写一个短评。"),
])

chain = prompt | model | fixing_parser

# 5. 调用链
result = chain.invoke({
    "movie": "肖申克的救赎",
    "format_instructions": base_parser.get_format_instructions(),
})

print(result)
# title='肖申克的救赎' rating=9.5 summary='一部关于希望与自由的经典杰作' recommend=True
print(result.title)       # 肖申克的救赎
print(result.rating)      # 9.5
print(result.recommend)   # True
```

---

## 内部工作原理

`OutputFixingParser` 的工作流程：

```
LLM 初始输出
      ↓
底层 parser 尝试解析
      ↓
   成功？ ──是──→ 返回解析结果
      │
     否
      ↓
构造修正提示词，包含：
  - 原始错误输出
  - 解析错误信息
  - 格式要求
      ↓
再次调用 LLM（修正 LLM）
      ↓
底层 parser 再次尝试解析修正后的输出
      ↓
   成功？ ──是──→ 返回解析结果
      │
     否
      ↓
抛出 OutputParserException
```

**详细流程：**

```python
# 内部伪代码（简化版）
class OutputFixingParser:
    def parse(self, text: str):
        try:
            return self.parser.parse(text)
        except OutputParserException as e:
            # 构造修正提示
            fix_prompt = f"""
            以下输出格式不正确：
            ---
            {text}
            ---
            错误信息：{e}
            请修正输出格式，使其符合要求。只输出修正后的内容。
            """
            # 再次调用 LLM
            corrected = self.llm.invoke(fix_prompt)
            # 用底层解析器解析修正后的输出
            return self.parser.parse(corrected.content)
```

### 关键要点

1. **只在解析失败时触发** — 如果底层解析器成功解析，不会有额外 LLM 调用
2. **修正提示词包含原始错误** — LLM 能看到自己哪里错了，从而针对性修正
3. **只重试一次** — 如果修正后仍然失败，直接抛出异常，不会无限重试

---

## 重试行为

`OutputFixingParser` **默认只重试一次**。如果修正后的输出仍然无法解析，会直接抛出 `OutputParserException`。

如果你需要更多重试次数，可以使用 `RetryWithErrorOutputParser`（参见下方）或手动实现重试逻辑。

### 与 RetryWithErrorOutputParser 的对比

```python
from langchain.output_parsers import RetryWithErrorOutputParser
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
base_parser = PydanticOutputParser(pydantic_object=MovieReview)

# RetryWithErrorOutputParser 会将完整的原始提示词发给 LLM 重新生成
# 而 OutputFixingParser 只发送修正提示
retry_parser = RetryWithErrorOutputParser.from_llm(
    llm=model,
    parser=base_parser,
)
```

| 特性 | OutputFixingParser | RetryWithErrorOutputParser |
|------|-------------------|--------------------------|
| 修正方式 | 发送修正提示（只修正格式） | 发送完整原始提示（重新生成） |
| 需要原始提示词 | 否 | 是 |
| 重试次数 | 1次 | 1次 |
| 适用场景 | 格式小错误 | 内容或格式严重错误 |
| Token 消耗 | 较少 | 较多 |

---

## 成本与延迟权衡

### Token 估算

假设主链的输出约 200 tokens：

| 场景 | LLM 调用次数 | 估算 Token 消耗 | 估算延迟 |
|------|-------------|----------------|---------|
| **首次解析成功** | 0次额外调用 | 0 额外 tokens | +0s |
| **首次失败，修正成功** | 1次额外调用 | ~300-500 tokens（修正提示+新输出） | +1-3s |
| **修正也失败** | 1次额外调用 | ~300-500 tokens（浪费） | +1-3s 后报错 |

**修正提示词的 Token 构成：**

- 原始错误输出：~200 tokens
- 错误信息：~50-100 tokens
- 格式要求：~50-100 tokens
- 修正后输出：~200 tokens
- **合计：约 500 tokens**

### 成本计算示例（以 gpt-4o-mini 为例）

```python
# gpt-4o-mini 定价（约）：
# 输入: $0.15 / 1M tokens
# 输出: $0.60 / 1M tokens

# 假设每次修正：300 input tokens + 200 output tokens
fix_cost = (300 * 0.15 + 200 * 0.60) / 1_000_000
# ≈ $0.000165 per fix attempt

# 如果 100 次调用中有 20% 需要修正：
monthly_fix_cost = fix_cost * 100 * 0.20
# ≈ $0.0033（几乎可忽略）
```

### 何时值得使用

```python
# ✅ 值得使用 OutputFixingParser 的场景：
# - PydanticOutputParser（格式复杂，LLM 容易出错）
# - 生产环境需要高可靠性
# - 每个请求的成本较低（使用便宜模型做修正）

# ❌ 不推荐使用的场景：
# - StrOutputParser（几乎不会解析失败）
# - CommaSeparatedListOutputParser（格式简单，提示词优化即可）
# - 对延迟极度敏感的实时场景（多一次 LLM 调用增加 1-3s）
# - 每个请求都很贵的场景（如长文档处理）
```

### 降低成本的技巧

```python
from langchain.output_parsers import OutputFixingParser
from langchain_core.output_parsers import PydanticOutputParser
from langchain_openai import ChatOpenAI

# 技巧1：主链用强模型，修正用便宜模型
main_model = ChatOpenAI(model="gpt-4o-mini")        # 主链
cheap_model = ChatOpenAI(model="gpt-4o-mini")       # 修正用便宜模型

base_parser = PydanticOutputParser(pydantic_object=MovieReview)
fixing_parser = OutputFixingParser.from_llm(llm=cheap_model, parser=base_parser)

chain = prompt | main_model | fixing_parser
```

---

## 配合不同解析器使用

### 配合 EnumOutputParser

```python
from langchain.output_parsers import OutputFixingParser
from langchain_core.output_parsers import EnumOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from enum import Enum

model = ChatOpenAI(model="gpt-4o-mini")

class Priority(Enum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

base_parser = EnumOutputParser(enum=Priority)
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

prompt = ChatPromptTemplate.from_messages([
    ("system", "判断优先级。{format_instructions}"),
    ("human", "{task}"),
])

chain = prompt | model | fixing_parser

result = chain.invoke({
    "task": "服务器宕机，用户数据可能丢失",
    "format_instructions": base_parser.get_format_instructions(),
})
print(result)  # Priority.HIGH
```

### 配合 PydanticOutputParser（最常用）

```python
from langchain.output_parsers import OutputFixingParser
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

model = ChatOpenAI(model="gpt-4o-mini")

class UserInfo(BaseModel):
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")
    skills: list[str] = Field(description="技能列表")

base_parser = PydanticOutputParser(pydantic_object=UserInfo)
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

prompt = ChatPromptTemplate.from_messages([
    ("system", "提取人物信息。{format_instructions}"),
    ("human", "{text}"),
])

chain = prompt | model | fixing_parser

result = chain.invoke({
    "text": "张伟，32岁，擅长Python、机器学习和数据分析",
    "format_instructions": base_parser.get_format_instructions(),
})

print(result.name)    # 张伟
print(result.age)     # 32
print(result.skills)  # ['Python', '机器学习', '数据分析'][[
```

---

## 错误处理

### 修正也失败的情况

```python
from langchain_core.exceptions import OutputParserException

try:
    result = chain.invoke({...})
except OutputParserException as e:
    print(f"即使修正后也无法解析: {e}")
    # 降级处理
    result = None
```

### 配合 try/except 实现降级

```python
from langchain_core.exceptions import OutputParserException
from langchain_core.output_parsers import StrOutputParser

fallback_parser = StrOutputParser()

try:
    result = chain.invoke({...})
except OutputParserException:
    # 降级为纯文本
    chain_without_parser = prompt | model
    raw = chain_without_parser.invoke({...})
    print(f"降级到原始输出: {raw.content}")
```

---

## 实用技巧

### 技巧1：不要用于简单解析器

```python
from langchain_core.output_parsers import StrOutputParser
from langchain.output_parsers import OutputFixingParser

# ❌ StrOutputParser 几乎不会失败，包装它没有意义
# 白白增加了代码复杂度和潜在延迟
bad_parser = OutputFixingParser.from_llm(llm=model, parser=StrOutputParser())

# ✅ 只用于容易失败的复杂解析器
good_parser = OutputFixingParser.from_llm(
    llm=model,
    parser=PydanticOutputParser(pydantic_object=ComplexModel)
)
```

### 技巧2：现代替代方案

如果你的模型支持 tool calling（OpenAI、Anthropic、Google 等），推荐使用 `with_structured_output`，它能从根本上避免格式错误：

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class MovieReview(BaseModel):
    title: str
    rating: float
    summary: str
    recommend: bool

# 使用 tool calling 机制，保证输出符合 schema
model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(MovieReview)
result = model.invoke("给《肖申克的救赎》写一个短评")
# 不需要 OutputFixingParser，因为输出一定符合 Pydantic 模型
```

参见：[[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]

---

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/PydanticOutputParser|PydanticOutputParser - 最常配合使用]]
- [[langchain/modules/langchain_output_parsers/EnumOutputParser|EnumOutputParser - 也可配合使用]]
- [[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]
- [[langchain/modules/langchain_output_parsers/错误处理|错误处理]]
