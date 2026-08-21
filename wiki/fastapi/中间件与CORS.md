---
tags: [fastapi, middleware, cors]
created: 2026-07-22
updated: 2026-07-22
type: source
source_url: https://fastapi.tiangolo.com/tutorial/middleware/
fastapi_version: ">=0.115"
---

# 中间件与 CORS

FastAPI 中间件在**请求处理前/后**插入逻辑——类似 LangChain 的 Hook 机制，但作用于 HTTP 层。

## 一、内置中间件

| 中间件 | 来源 | 作用 |
|--------|------|------|
| `CORSMiddleware` | `fastapi.middleware.cors` | 跨域资源共享 |
| `GZipMiddleware` | `starlette.middleware.gzip` | 响应压缩 |
| `HTTPSRedirectMiddleware` | `starlette.middleware.httpsredirect` | HTTP → HTTPS 重定向 |
| `TrustedHostMiddleware` | `starlette.middleware.trustedhost` | 限制允许的 Host |
| `SessionMiddleware` | `starlette.middleware.sessions` | 基于 Cookie 的会话 |

## 二、自定义中间件

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    # 1. 请求前逻辑
    start_time = time.time()

    # 2. 调用下一个中间件/路由
    response = await call_next(request)

    # 3. 响应后逻辑
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

### 数据流向

```
请求到达
    ↓
中间件1 请求前逻辑 → 中间件2 请求前逻辑 → ... → 路由函数
                                                ↓
中间件1 响应后逻辑 ← 中间件2 响应后逻辑 ← ... ← 响应
```

## 三、CORS 配置

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],    # 允许的前端域名
    allow_credentials=True,                      # 是否允许携带 Cookie
    allow_methods=["GET", "POST"],               # 允许的 HTTP 方法
    allow_headers=["*"],                          # 允许的请求头
)
```

> **开发时**：`allow_origins=["*"]`（全部允许）。**生产环境**：必须指定明确的域名列表。

## 四、与 LangChain 中间件对比

| 对比维度 | FastAPI 中间件 | LangChain 中间件 |
|---------|---------------|-----------------|
| **作用时机** | HTTP 请求到达/离开时 | Agent 执行流程中 |
| **覆盖范围** | 所有路由 | Agent 内部模型/工具调用 |
| **典型用途** | 日志、CORS、压缩、认证 | 摘要、重试、HITL、PII |
| **执行方式** | 顺序链式 | Node-style(顺序) + Wrap-style(嵌套) |
| **状态修改** | 请求/响应对象 | AgentState（消息列表 + 自定义字段） |

> **详见**：[[langchain/middleware/钩子机制详解]] — LangChain Hook 机制

## 五、相关页面

- [[fastapi/安全认证]] — 认证中间件
- [[fastapi/与LangChain集成]] — Agent 级部署
- [[langchain/middleware/中间件概述]] — LangChain 中间件系统
