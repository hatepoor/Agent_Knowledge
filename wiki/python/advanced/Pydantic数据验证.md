---
tags: [concept, Python, Pydantic, 数据验证, 结构化输出, Agent]
type: concept
created: 2026-07-08
updated: 2026-07-31
---

# Pydantic 数据验证

> Pydantic 是 Python 最流行的数据验证库，也是 LangChain 的核心依赖。它基于类型提示提供运行时数据验证、序列化和文档生成。

## 组件全景图（Pydantic v2）

```
pydantic
├── BaseModel             核心基类：所有数据模型的起点
├── Field()               字段级元数据：校验规则、描述、别名
├── RootModel              根模型：处理 list/dict 等非对象结构的 JSON
│
├── 验证器
│   ├── @field_validator       字段级验证
│   ├── @model_validator       模型级验证（跨字段）
│   ├── BeforeValidator        管道式验证（Annotated 语法）
│   ├── AfterValidator
│   └── WrapValidator
│
├── 类型系统
│   ├── EmailStr / HttpUrl / IPvAny    网络类型（需 pydantic-extra-types）
│   ├── conint / constr / confloat      约束类型（快捷版 Field）
│   ├── SecretStr / SecretBytes        密钥类型（打印时自动隐藏）
│   ├── StrictBool / StrictInt         严格类型（不自动转换）
│   └── TypeAdapter                    轻量级验证（无需继承 BaseModel）
│
├── 序列化
│   ├── .model_dump()          → dict
│   ├── .model_dump_json()     → JSON 字符串
│   ├── .model_validate()      dict → 模型
│   └── .model_validate_json() JSON 字符串 → 模型
│
├── 配置
│   ├── ConfigDict              模型级配置（extra/frozen/populate_by_name）
│   └── model_config = ConfigDict(...)
│
├── 通用模型
│   └── Generic[T]              泛型模型（如 PaginatedResponse[User]）
│
├── 错误
│   └── ValidationError         校验失败时抛出的异常（含 loc/msg/type）
│
└── 扩展包（独立 pip install）
    ├── pydantic-settings        BaseSettings：从 .env / 环境变量加载配置
    └── pydantic-extra-types     额外类型：PhoneNumber / Color / MACAddress 等
```

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

### RootModel — 非对象结构的 JSON

当 JSON 顶层不是对象（而是 list 或 dict）时，用 `RootModel`：

```python
from pydantic import RootModel

class User(BaseModel):
    name: str
    age: int

# 顶层是列表
UserList = RootModel[list[User]]
users = UserList.model_validate([{"name": "A", "age": 20}, {"name": "B", "age": 25}])
print(users.root)   # [User(name='A', age=20), User(name='B', age=25)]

# 顶层是字典
Config = RootModel[dict[str, int]]
config = Config.model_validate({"timeout": 30, "retries": 3})
```

### SecretStr — 密钥自动脱敏

API Key、密码等敏感字段用 `SecretStr`，打印/序列化时自动隐藏：

```python
from pydantic import BaseModel, SecretStr

class AppConfig(BaseModel):
    api_key: SecretStr

config = AppConfig(api_key="sk-1234567890")

print(config.api_key)            # **********
print(config.api_key.get_secret_value())  # sk-1234567890（需要时显式取出）
print(config.model_dump())       # {'api_key': '**********'}
```

### TypeAdapter — 不继承 BaseModel 也能验证

```python
from pydantic import TypeAdapter

# 验证单个值
adapter = TypeAdapter(int)
adapter.validate_python("42")    # 42（字符串自动转 int）

# 验证复杂结构
list_adapter = TypeAdapter(list[int])
list_adapter.validate_python([1, 2, 3])
```

### pydantic-settings — 配置管理（BaseSettings + SettingsConfigDict）

> `pydantic_settings` 是 Pydantic 官方扩展包（`pip install pydantic-settings`），职责是把 `.env` 文件、环境变量、密钥目录中的配置自动注入到类型安全、带校验的 Python 类中。

**BaseSettings vs BaseModel**：

| | `BaseModel` | `BaseSettings` |
|---|---|---|
| 取值来源 | 手动传入 | 环境变量 / `.env` / 默认值 自动匹配 |
| 典型用途 | 业务数据结构（请求体、Tool 参数） | 全局配置（API Key、端口、超时） |
| 继承关系 | pydantic 核心 | 是 `BaseModel` 的子类，自动带 settings 能力 |

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # SettingsConfigDict 控制"怎么找值"——Settings 专属的配置选项
    # （取代 BaseModel 时代的 ConfigDict，新增 env_file/env_prefix 等键）
    model_config = SettingsConfigDict(
        env_file=".env",                  # 从哪个文件读配置
        env_file_encoding="utf-8",        # 文件编码
        env_prefix="AGENT_",              # 字段自动加前缀去环境变量中找
        case_sensitive=False,             # 环境变量名是否区分大小写
        extra="ignore",                   # 多余的键忽略（不报错）
    )

    api_key: str            # 找 AGENT_API_KEY（环境变量或 .env）
    model_name: str = "deepseek-v4-flash"  # 有默认值，找不到就用默认
    max_tokens: int = 4000
    timeout: float = 30.0

settings = Settings()
print(settings.api_key)     # .env 中的值自动注入
```

**SettingsConfigDict 专属键**（ConfigDict 没有的）：

| 键 | 作用 |
|----|------|
| `env_file` | 指定 `.env` 配置文件路径 |
| `env_prefix` | 字段名自动加前缀去环境变量中找（`api_key` → 找 `AGENT_API_KEY`） |
| `env_nested_delimiter` | 嵌套模型用分隔符展开（如 `database.host`） |
| `secrets_dir` | 从密钥文件目录读取（Docker secrets） |

**加载优先级（从高到低）**：

```
显式传入参数  >  系统环境变量  >  .env 文件  >  字段默认值
Settings(api_key="x")   AGENT_API_KEY   API_KEY=sk-xxx    api_key: str = "default"
```

**实例化时的数据流向**：

```
Settings() 被实例化
    │
    ├── 1. 检查 .env 文件（env_file 指定的路径）
    │      API_KEY=sk-123456  ──→  api_key: "sk-123456"
    │
    ├── 2. 检查系统环境变量（优先级更高，可覆盖 .env）
    │      AGENT_MODEL_NAME=gpt-4o  ──→  model_name: "gpt-4o"
    │
    ├── 3. 检查字段默认值（兜底）
    │      max_tokens = 4000
    │
    └── 4. 按字段类型做 Pydantic 校验/转换
           "30.0" → 30.0 (float)
```

**Agent 项目中的典型用法**：

```python
# config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="AGENT_")

    deepseek_api_key: str          # 读 AGENT_DEEPSEEK_API_KEY
    tavily_api_key: str            # 读 AGENT_TAVILY_API_KEY
    db_path: str = "./memory.db"
    max_tokens: int = 4000

settings = Settings()   # 全项目共享一份配置
```

```bash
# .env
AGENT_DEEPSEEK_API_KEY=sk-xxx
AGENT_TAVILY_API_KEY=tvly-xxx
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
| `Annotated[T, ...]` | 带元数据的类型 | `Annotated[int, Field(gt=0)]` |

## Agent 开发中的四层用法速查

| 层 | 用什么 | 场景 |
|----|--------|------|
| **Tool 参数** | `BaseModel` + `@tool(args_schema=...)` | 定义工具的参数类型和校验规则 |
| **结构化输出** | `llm.with_structured_output(Schema)` | 强制 LLM 返回符合 Schema 的 JSON |
| **State 定义** | `TypedDict` + `Annotated` | LangGraph 的 AgentState |
| **配置管理** | `BaseSettings` + `ConfigDict` | 统一管理 API Key、模型参数 |

## 相关页面

- [[python/advanced/类型提示进阶]] — Python 类型提示详解
- [[python/advanced/序列化与状态管理]] — JSON/序列化在 Agent 中的应用
- [[langchain/structured-output/结构化输出]] — LangChain 结构化输出详解
