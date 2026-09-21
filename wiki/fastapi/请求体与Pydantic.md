---
tags: [fastapi, pydantic, request-body]
created: 2026-07-22
updated: 2026-07-22
type: source
source_url: https://fastapi.tiangolo.com/tutorial/body/
fastapi_version: ">=0.115"
---

# 请求体与 Pydantic

FastAPI 使用 Pydantic v2 的 `BaseModel` 来定义请求体结构。声明一个 Pydantic Model → FastAPI 自动完成 JSON→Python 转换、校验和文档生成。

## 一、基础：Pydantic Model = Request Body

```python
from pydantic import BaseModel

# 第1步：定义数据模型
class Item(BaseModel):
    name: str               # 必填，必须是字符串
    price: float            # 必填，必须是浮点数
    description: str | None = None  # 可选，不传则为 None
    tax: float | None = None        # 可选

# 第2步：声明为路径操作函数参数
@app.post("/items/")
async def create_item(item: Item):
#                     ^^^^ Pydantic Model 作为类型提示
# FastAPI 自动：
#   1. 从请求 body 读取 JSON
#   2. 转换为 Item 对象
#   3. 校验所有字段
#   4. 生成 OpenAPI 文档
    return {"item": item}
```

### 数据流向

```
POST /items/  Body: {"name": "iPhone", "price": 999.99}
    ↓
FastAPI 读取 JSON body
    ↓
Pydantic 解析：{"name": "iPhone", "price": 999.99} → Item(name="iPhone", price=999.99)
    ↓
调用 create_item(item=Item(name="iPhone", price=999.99))
    ↓
函数返回 {"item": Item(...)}
    ↓
Pydantic 序列化：Item → {"name": "iPhone", "price": 999.99, "description": null, "tax": null}
    ↓
返回 JSON
```

## 二、嵌套模型

```python
class Image(BaseModel):
    url: str
    name: str

class Item(BaseModel):
    name: str
    price: float
    image: Image | None = None       # 嵌套 Pydantic 模型
    tags: list[str] = []             # 字符串列表

# 请求体：
# {
#   "name": "iPhone",
#   "price": 999.99,
#   "image": {"url": "https://...", "name": "front"},
#   "tags": ["electronics", "phone"]
# }
```

Pydantic 自动递归校验——`image` 内部字段也按规则校验。

## 三、特殊类型

| 类型 | Pydantic 校验规则 |
|------|------------------|
| `str` | 必须是字符串 |
| `int` | 必须是整数（自动转换 `"42"` → `42`） |
| `float` | 必须是浮点数 |
| `bool` | 布尔值 |
| `EmailStr` | 必须符合 email 格式（需 `email-validator`） |
| `HttpUrl` | 必须是有效 URL |
| `UUID` | 必须是 UUID4 格式 |
| `datetime` | ISO 8601 格式自动转换 |
| `Literal["a", "b"]` | 必须是枚举值之一 |

```python
from pydantic import BaseModel, EmailStr, HttpUrl
from datetime import datetime
from uuid import UUID

class User(BaseModel):
    id: UUID
    name: str
    email: EmailStr          # 自动校验邮箱格式
    website: HttpUrl | None = None
    created_at: datetime
```

## 四、Field：字段级校验

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(min_length=1, max_length=100, description="商品名称")
    price: float = Field(gt=0, description="价格，必须大于 0")
    tags: list[str] = Field(default_factory=list, max_length=10)
    # gt = greater than, ge = greater or equal, lt, le
```

## 五、与 LangChain Pydantic 结合

FastAPI 和 LangChain 都用 Pydantic，可以**复用同一个 Model**：

```python
from pydantic import BaseModel

# 同一个模型用于 FastAPI 请求体和 LangChain 结构化输出
class AgentQuery(BaseModel):
    question: str
    context: str | None = None
    max_tokens: int = Field(default=500, ge=1, le=4000)

# FastAPI 用：
@app.post("/agent")
async def run_agent(query: AgentQuery):
    result = await agent.ainvoke({"input": query.question})
    return result

# LangChain 用（结构化输出绑定）：
chain = llm.with_structured_output(AgentQuery)
```

## 六、多 Body 参数

```python
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,           # 路径参数
    item: Item,             # Body 参数（JSON body）
    user: User,             # 第二个 Body 参数
    importance: int = Body(gt=0),   # 单个 Body 字段
):
    ...
```

当函数有**多个 Pydantic 模型**参数时，FastAPI 期望 JSON body 中包含键：

```json
{
  "item": {"name": "iPhone", "price": 999.99},
  "user": {"name": "Alice", "email": "alice@example.com"},
  "importance": 5
}
```

## 七、相关页面

- [[fastapi/请求体与Pydantic]] — Path/Query 参数校验
- [[fastapi/路径操作]] — 路径参数与查询参数
- [[fastapi/错误处理与响应模型]] — response_model 控制输出
- [[fastapi/与LangChain集成]] — Pydantic Model 在 Agent API 中的实践
