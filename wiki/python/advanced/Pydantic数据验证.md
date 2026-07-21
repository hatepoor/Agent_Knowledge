---
tags: [concept, Python, Pydantic, 数据验证, 结构化输出, Agent]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# Pydantic 数据验证

> Pydantic 是 Python 最流行的数据验证库，也是 LangChain 的核心依赖。它基于类型提示提供运行时数据验证、序列化和文档生成。

## 基础用法

```python
from pydantic import BaseModel, Field
from typing import Optional, List

class User(BaseModel):
    id: int
    name: str
    email: str
    age: Optional[int] = None  # 可选字段，默认 None
    tags: List[str] = []       # 默认空列表

# 创建实例（自动验证和类型转换）
user = User(
    id=1,
    name="Alice",
    email="alice@example.com",
    age="25"  # 字符串 "25" 自动转为 int
)

print(user.id)       # 1
print(user.model_dump())  # {'id': 1, 'name': 'Alice', ...}
```

## 字段验证

### Field 高级配置

```python
from pydantic import BaseModel, Field
from typing import Annotated

class Product(BaseModel):
    name: str = Field(
        min_length=1,
        max_length=100,
        description="产品名称"
    )
    price: float = Field(
        gt=0,           # 必须大于 0
        le=10000,       # 小于等于 10000
        description="价格（元）"
    )
    stock: int = Field(
        ge=0,           # 大于等于 0
        default=0
    )
    # 使用 Annotated 语法（Python 3.9+）
    sku: Annotated[str, Field(pattern=r"^[A-Z]{3}-\d{4}$")]
```

### 自定义验证器

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Self

class User(BaseModel):
    username: str
    password: str
    email: str
    
    # 字段级验证器
    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("用户名必须是字母数字")
        return v.lower()
    
    # 模型级验证器（跨字段验证）
    @model_validator(mode="after")
    def check_password_strength(self) -> Self:
        if len(self.password) < 8:
            raise ValueError("密码至少 8 位")
        return self
```

## 序列化与反序列化

```python
from pydantic import BaseModel
from datetime import datetime

class Event(BaseModel):
    name: str
    timestamp: datetime
    data: dict

event = Event(
    name="会议",
    timestamp="2026-07-08T10:00:00",  # 自动解析
    data={"attendees": 5}
)

# 序列化
event.model_dump()           # dict
event.model_dump_json()      # JSON 字符串
event.model_dump_json(indent=2)  # 格式化 JSON

# 反序列化
data = {"name": "会议", "timestamp": "2026-07-08T10:00:00", "data": {}}
event = Event.model_validate(data)

# 从 JSON 字符串
json_str = '{"name": "会议", "timestamp": "2026-07-08T10:00:00", "data": {}}'
event = Event.model_validate_json(json_str)
```

## LangChain 中的 Pydantic 应用

### 结构化输出（Structured Output）

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class MovieRecommendation(BaseModel):
    title: str = Field(description="电影名称")
    year: int = Field(description="上映年份")
    reason: str = Field(description="推荐理由")

llm = ChatOpenAI(model="gpt-4o-mini")

# 方式1：with_structured_output（推荐）
structured_llm = llm.with_structured_output(MovieRecommendation)
result = structured_llm.invoke("推荐一部科幻电影")
# result 是 MovieRecommendation 实例

# 方式2：使用 PydanticOutputParser
from langchain.output_parsers import PydanticOutputParser
from langchain_core.prompts import PromptTemplate

parser = PydanticOutputParser(pydantic_object=MovieRecommendation)

prompt = PromptTemplate(
    template="推荐一部电影\n{format_instructions}\n{query}",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()}
)

chain = prompt | llm | parser
result = chain.invoke({"query": "推荐一部科幻电影"})
```

### Tool 定义

```python
from langchain.tools import tool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大结果数")

@tool(args_schema=SearchInput)
def search_web(query: str, max_results: int = 5) -> str:
    """搜索网络获取信息"""
    return f"搜索 '{query}'，返回 {max_results} 条结果"

# Agent 会自动使用 SearchInput 作为参数 schema
```

### Agent 状态定义

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    """Agent 状态定义"""
    messages: Annotated[list, add_messages]  # 消息列表，自动追加
    current_tool: str | None                 # 当前工具
    iteration: int                           # 迭代次数
```

## 高级特性

### 模型继承

```python
class BaseUser(BaseModel):
    id: int
    name: str

class AdminUser(BaseUser):
    permissions: list[str]

admin = AdminUser(id=1, name="Alice", permissions=["read", "write"])
```

### 配置类（Config）

```python
from pydantic import BaseModel, ConfigDict

class Settings(BaseModel):
    model_config = ConfigDict(
        extra="forbid",        # 禁止额外字段
        frozen=True,           # 不可变
        validate_assignment=True  # 赋值时验证
    )
    
    api_key: str
    timeout: int = 30

settings = Settings(api_key="sk-xxx")
# settings.api_key = "new"  # 报错：frozen
```

### 泛型模型

```python
from pydantic import BaseModel
from typing import Generic, TypeVar, List

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    page_size: int

class User(BaseModel):
    id: int
    name: str

# 使用
response = PaginatedResponse[User](
    items=[User(id=1, name="Alice")],
    total=100,
    page=1,
    page_size=10
)
```

## 常用类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `Optional[T]` | 可选类型 | `Optional[str]` |
| `List[T]` | 列表 | `List[int]` |
| `Dict[K, V]` | 字典 | `Dict[str, int]` |
| `Tuple[T, ...]` | 元组 | `Tuple[int, str]` |
| `Union[A, B]` | 联合类型 | `Union[int, str]` |
| `Literal[...]` | 字面量 | `Literal["a", "b"]` |
| `Annotated[T, ...]` | 带元数据的类型 | `Annotated[int, Field(gt=0)][[` |

## 相关页面

- [[python/advanced/类型提示进阶]] — Python 类型提示详解
- [[python/advanced/序列化与状态管理]] — JSON/序列化在 Agent 中的应用
- [[langchain/structured-output/结构化输出]] — LangChain 结构化输出详解
