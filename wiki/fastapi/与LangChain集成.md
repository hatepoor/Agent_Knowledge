---
tags: [fastapi, langchain, langgraph, agent, integration, deployment]
created: 2026-07-22
updated: 2026-07-22
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in
fastapi_version: ">=0.115"
langchain_version: ">=1.0"
---

# FastAPI 与 LangChain 集成

将 LangChain/LangGraph Agent 暴露为 REST API，是 Agent 走向生产的关键一步。FastAPI 作为 HTTP 外壳，Agent 作为业务核心。

## 一、架构定位

```
                     HTTP 层                    |       AI 层
─────────────────────────────────────────────────┼──────────────────
 浏览器/前端 → FastAPI → 路由函数 → Agent.invoke() / stream()
                 ↑                              ↑
            CORS/认证/限流                   ReAct 循环/工具调用
            (FastAPI 中间件)                  (LangChain 中间件)
```

**职责划分**：

| 层 | 技术 | 做什么 | 不做什么 |
|----|------|--------|---------|
| HTTP 层 | FastAPI | 路由、参数解析、CORS、认证、流式输出格式 | 不做 AI 决策 |
| AI 层 | LangChain/LangGraph | 模型调用、工具选择、ReAct 循环 | 不管 HTTP 协议 |

## 二、最简集成

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from langchain_deepseek import ChatDeepSeek
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver
import os
from dotenv import load_dotenv

load_dotenv()

# ── 初始化 Agent（启动时执行一次）──
llm = ChatDeepSeek(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    temperature=1.0,
)
checkpointer = InMemorySaver()  # 生产用 SqliteSaver / PostgresSaver
agent = create_agent(model=llm, tools=[], checkpointer=checkpointer)

# ── FastAPI 应用 ──
app = FastAPI(title="AI Agent API")

class ChatRequest(BaseModel):
    message: str
    thread_id: str = "default"

class ChatResponse(BaseModel):
    reply: str
    thread_id: str

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    # 第1步：构建 Agent 输入
    config = {"configurable": {"thread_id": req.thread_id}}

    # 第2步：异步调用 Agent
    result = await agent.ainvoke(
        {"messages": [HumanMessage(content=req.message)]},
        config=config,
    )

    # 第3步：提取最后一条 AI 消息
    reply = result["messages"][-1].content
    return ChatResponse(reply=reply, thread_id=req.thread_id)

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### 数据流向

```
用户 POST /chat {"message": "你好", "thread_id": "user_001"}
    ↓ FastAPI 路由匹配 → @app.post("/chat")
    ↓ Pydantic 解析 → ChatRequest(message="你好", thread_id="user_001")
    ↓
agent.ainvoke({"messages": [HumanMessage("你好")]}, config={"thread_id": "user_001"})
    ↓ ReAct 循环（可能有工具调用）→ {"messages": [...HumanMessage, AIMessage]}
    ↓
提取 result["messages"][-1].content = "你好！有什么可以帮你的？"
    ↓ Pydantic 序列化 ChatResponse → {"reply": "你好！...", "thread_id": "user_001"}
    ↓
返回 HTTP 200 JSON
```

## 三、带工具和中间件的 Agent

```python
from langchain_tavily import TavilySearch
from langchain.agents.middleware import (
    SummarizationMiddleware,
    HumanInTheLoopMiddleware,
    PIIMiddleware,
)

# 初始化工具和中间件
tavily_tool = TavilySearch(max_results=5, api_key=os.getenv("TAVILY_API_KEY"))
summary_llm = ChatDeepSeek(model="deepseek-v4-flash", temperature=0.0)

agent = create_agent(
    model=llm,
    tools=[tavily_tool],
    checkpointer=SqliteSaver(sqlite3.connect("sessions.db")),
    system_prompt="你是专业助手，用中文回答，语气简洁。",
    middleware=[
        SummarizationMiddleware(model=summary_llm, trigger=("tokens", 4000)),
        PIIMiddleware(pii_type="email", strategy="redact"),
    ],
)

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    result = await agent.ainvoke(
        {"messages": [HumanMessage(content=req.message)]},
        config={"configurable": {"thread_id": req.thread_id}},
    )
    return ChatResponse(
        reply=result["messages"][-1].content,
        thread_id=req.thread_id,
    )
```

## 四、流式响应（SSE）

流式是生产环境 Agent API 的标准——用户看到逐字输出，而非等待 10-30 秒：

```python
from fastapi.responses import StreamingResponse
import json

@app.post("/chat/stream")
async def chat_stream(req: ChatRequest):
    async def event_generator():
        config = {"configurable": {"thread_id": req.thread_id}}
        async for event in agent.astream_events(
            {"messages": [HumanMessage(content=req.message)]},
            config=config,
            version="v3",
        ):
            kind = event["event"]
            if kind == "on_chat_model_stream":
                content = event["data"]["chunk"].content
                if content:
                    yield f"data: {json.dumps({'type': 'token', 'content': content})}\n\n"
            elif kind == "on_tool_start":
                yield f"data: {json.dumps({'type': 'tool_start', 'name': event['name']})}\n\n"
            elif kind == "on_tool_end":
                yield f"data: {json.dumps({'type': 'tool_end', 'name': event['name']})}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### SSE 事件流格式

```
data: {"type": "tool_start", "name": "tavily_search"}
data: {"type": "tool_end", "name": "tavily_search"}
data: {"type": "token", "content": "根据"}
data: {"type": "token", "content": "搜索结果"}
data: {"type": "token", "content": "，今天"}
...
data: {"type": "done"}
```

> **详见**：[[fastapi/流式响应]]

## 五、Human-in-the-Loop 的双端点模式

```python
from langgraph.types import Command

@app.post("/chat/hitl")
async def chat_with_hitl(req: ChatRequest):
    """发送消息，可能触发 HITL 中断"""
    config = {"configurable": {"thread_id": req.thread_id}}
    result = await agent.ainvoke(
        {"messages": [HumanMessage(content=req.message)]},
        config=config,
    )
    return {"result": result, "thread_id": req.thread_id}

class ResumeRequest(BaseModel):
    thread_id: str
    decision: str  # "approve" | "reject" | "edit"
    edited_args: dict | None = None

@app.post("/chat/hitl/resume")
async def resume_hitl(req: ResumeRequest):
    """审批通过后恢复执行"""
    config = {"configurable": {"thread_id": req.thread_id}}
    result = await agent.ainvoke(
        Command(resume={"decision": req.decision}),
        config=config,
    )
    return {"result": result, "thread_id": req.thread_id}
```

## 六、生产环境完整骨架

```python
from fastapi import FastAPI, Depends, Header, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from functools import lru_cache
import time

# 生命周期管理
@asynccontextmanager
async def lifespan(app: FastAPI):
    print("[lifespan] 初始化 Agent...")
    yield  # 应用运行中
    print("[lifespan] 清理资源...")

app = FastAPI(title="Production Agent API", lifespan=lifespan)

# CORS
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

# 请求日志中间件
@app.middleware("http")
async def log_requests(request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    print(f"[{request.method}] {request.url.path} - {duration:.3f}s - {response.status_code}")
    return response

# 认证依赖
async def verify_token(authorization: str = Header(default="")):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="未提供 token")
    token = authorization.split(" ", 1)[1]
    if token != os.getenv("API_TOKEN"):
        raise HTTPException(status_code=403, detail="token 无效")
    return token

# Agent 依赖（单例）
@lru_cache
def get_agent():
    return create_agent(...)

@app.post("/chat")
async def chat(
    req: ChatRequest,
    token: str = Depends(verify_token),  # 认证
    agent=Depends(get_agent),            # Agent 实例
):
    ...
```

## 七、常见陷阱

1. **InMemorySaver 重启丢数据**——生产必须用 SqliteSaver 或 PostgresSaver
2. **不设 timeout**——Agent 可能陷入死循环，API 和客户端一直等待
3. **LLM 初始化在路由函数内**——每次请求都创建新实例，严重浪费
4. **同步 `invoke()` 阻塞事件循环**——必须用 `ainvoke()` 或 `run_in_executor()`
5. **checkpointer 无连接池**——PostgresSaver 创建新连接每次调用，需复用

## 八、相关页面

- [[fastapi/快速入门]] — 基础 FastAPI
- [[fastapi/依赖注入]] — @lru_cache + Depends 注入 Agent
- [[fastapi/流式响应]] — SSE 流式输出
- [[fastapi/安全认证]] — Bearer Token 认证
- [[langchain/middleware/中间件概述]] — Agent 中间件
- [[langchain/memory/Checkpointer存储后端]] — 选择合适的 checkpointer
