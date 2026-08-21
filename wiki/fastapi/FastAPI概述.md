---
tags: [fastapi, overview]
created: 2026-07-22
updated: 2026-07-22
type: source
source_url: https://fastapi.tiangolo.com
fastapi_version: ">=0.115"
---

# FastAPI 概述

FastAPI 是一个基于 Python 类型提示的**现代、高性能 Web 框架**，专用于构建 REST API。它站在两个巨人的肩膀上：**Starlette** 处理 Web/HTTP 层，**Pydantic** 处理数据层。

> **一句话**：用 Python 类型提示定义 API → FastAPI 自动完成数据校验、序列化、文档生成。

## 一、FastAPI 解决了什么

传统 Python Web 框架（Flask/Django REST）的核心痛点：

| 痛点 | 传统方案 | FastAPI 方案 |
|------|----------|-------------|
| 参数校验代码冗余 | 手写 if-else + 异常处理 | 类型提示自动校验 |
| 文档维护成本高 | 手写 Swagger YAML / Postman | 自动生成 `/docs` + `/redoc` |
| 性能瓶颈 | 同步 I/O 阻塞 | 原生 async/await + ASGI |
| IDE 体验差 | 路由函数无类型提示 | 类型驱动 → 全自动补全 |
| 序列化/反序列化 | 手写 `json.dumps/loads` | Pydantic → 自动转换 |

**关键数据**（官方自述）：开发速度提高 200-300%，人为错误减少约 40%。

## 二、三层架构

```
┌─────────────────────────────────┐
│  FastAPI（业务逻辑 + 路由）      │  ← 你写代码的层面
├─────────────────────────────────┤
│  Pydantic（数据校验 + 序列化）    │  ← JSON ↔ Python 对象自动转换
├─────────────────────────────────┤
│  Starlette（Web/HTTP 核心）      │  ← 请求路由、中间件、会话
│  + Uvicorn（ASGI 服务器）        │  ← 高性能异步事件循环
└─────────────────────────────────┘
```

### 各层职责

| 层级 | 谁负责 | 做什么 |
|------|--------|--------|
| FastAPI | 你写的 `@app.get("/")` 路由函数 | 业务逻辑、API 设计 |
| Pydantic | `class Item(BaseModel)` | 数据校验、类型转换、OpenAPI Schema 生成 |
| Starlette + Uvicorn | `uvicorn main:app` | HTTP 协议解析、并发处理、WebSocket |

### 在 LangChain Agent 中的位置

```
用户/前端 ←─HTTP─→ FastAPI ←─invoke()─→ LangGraph/LangChain Agent
                          ←─stream()──→ (流式响应)
                    ↑
               中间件层（CORS、认证、限流）
```

FastAPI 充当 **Agent 的 HTTP 外壳**——它不做 AI 逻辑，只负责：
1. 接收 HTTP 请求，提取参数
2. 调用 Agent 的 `invoke()` / `stream()`
3. 把结果序列化为 JSON 或 SSE 流返回

> **详见**：[[fastapi/与LangChain集成]]

## 三、核心特性

### 3.1 类型驱动的参数声明

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str          # FastAPI 会自动校验 str 类型
    price: float       # 自动校验 float 类型
    is_offer: bool | None = None  # 可选字段

@app.post("/items/")
async def create_item(item: Item):     # 自动从 JSON body 解析
    return {"item_name": item.name, "price": item.price}
```

声明 `item: Item` 后，FastAPI 自动完成 5 件事：
- 从请求 body 读取 JSON
- 校验 `name` 必须是 `str`，`price` 必须是 `float`
- 校验失败返回 422 + 清晰错误信息
- 自动生成 Swagger/ReDoc 文档
- IDE 全程 `.` 出属性名

### 3.2 自动交互式文档

- `/docs` — Swagger UI（可在线测试 API）
- `/redoc` — ReDoc（只读文档）
- `/openapi.json` — 原始 OpenAPI 3.1 schema

### 3.3 依赖注入（Depends）

一个函数可以作为多个路由的"依赖"，FastAPI 自动注入：

```python
from fastapi import Depends

# 定义可复用的依赖
async def common_parameters(q: str | None = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

# 多个路由共享
@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users/")
async def read_users(commons: dict = Depends(common_parameters)):
    return commons
```

> **详见**：[[fastapi/依赖注入]]

### 3.4 原生 async/await

```python
@app.get("/slow")
async def slow_endpoint():
    result = await some_async_operation()   # 不阻塞其他请求
    return result
```

FastAPI 自动检测函数是 `async def` 还是 `def`，放入对应的线程池。

> **详见**：[[fastapi/异步处理]]

### 3.5 中间件系统

基于 Starlette 的中间件，可在请求前/后插入逻辑（与 LangChain 中间件概念类似）：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> **详见**：[[fastapi/中间件与CORS]]

## 四、与 LangChain 的关系

| LangChain 概念 | FastAPI 对应 | 说明 |
|---------------|-------------|------|
| Agent / Chain | FastAPI 路由函数 | 路由函数内调用 `agent.invoke()` |
| Middleware（钩子） | FastAPI 中间件 | 请求级别的拦截：CORS、日志、认证 |
| Checkpointer | 不需要（无状态） | API 是无状态的，状态在 Agent 侧 |
| RunnableConfig | `Depends(get_config)` | 从请求头/JWT 提取 thread_id 等 |
| stream_events | `StreamingResponse` + SSE | Agent 流式事件 → HTTP SSE |
| HumanInTheLoop | 两个 API 端点 | 一个 `invoke` 暂停，一个 `resume` 继续 |

> **详见**：[[fastapi/与LangChain集成]]

## 五、适用场景

| 场景 | 说明 |
|------|------|
| AI Agent 的 Web 接口 | 把 LangChain/LangGraph Agent 暴露为 REST API |
| 微服务 | 高性能、类型安全、文档自动生成 |
| 内部工具 / 管理后台 | 快速搭建带自动文档的数据 API |
| 机器学习模型服务 | 模型推理接口 + 请求校验 |
| WebSocket + AI | 实时对话、流式 Token 输出 |

## 六、快速安装

```bash
# 推荐：用 uv 创建项目
uv init my-api --bare
cd my-api
uv add "fastapi[standard]"

# 或传统 pip
pip install "fastapi[standard]"
```

## 七、学习路径建议

```
第一周：路径操作 → 请求体与Pydantic → 参数验证 → 响应模型
第二周：依赖注入 → 中间件与CORS → 错误处理 → 安全认证
第三周：异步处理 → 后台任务 → 测试 → 部署
第四周：与LangChain集成 → 流式响应 → 生产化Agent API
```

## 八、相关页面

- [[fastapi/快速入门]] — 第一个 FastAPI 应用
- [[fastapi/路径操作]] — Path/Query 参数、HTTP 方法
- [[fastapi/请求体与Pydantic]] — Pydantic Model 集成
- [[fastapi/依赖注入]] — Depends 核心机制
- [[fastapi/与LangChain集成]] — FastAPI + LangChain Agent 实战
- [[langchain/middleware/中间件概述]] — LangChain 中间件（对比理解）
