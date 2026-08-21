---
tags: [fastapi, websocket, agent, 实时通信, 双向通信]
type: tutorial
created: 2026-07-22
updated: 2026-07-22
---

# WebSocket 与 Agent 实时通信

WebSocket 提供**全双工持久连接**，适合 Agent 对话场景：客户端和服务器可以随时互发消息，不需要每次对话都建立新连接。

## 一、HTTP vs WebSocket

| 维度 | HTTP（REST） | WebSocket |
|------|-------------|-----------|
| 连接方式 | 每次请求建立新连接 | 一次握手，持久连接 |
| 通信方向 | 单向（客户端→服务器→客户端） | 双向（任意一方随时发） |
| 适用场景 | 一问一答 | 实时对话、流式输出、中途取消 |
| Agent 场景 | 简单问答 API | 聊天室、实时协作、中途打断 |

```
HTTP 模式（每次对话一个请求）：
  用户发消息 → POST /chat → 等 10 秒 → 收到完整回复 → 连接断开
  用户再发 → POST /chat → 又等 10 秒 → ...

WebSocket 模式（持久连接）：
  用户 ←──WebSocket 连接──→ 服务器
  用户发: "你好"
  服务器推: "你"
  服务器推: "好"
  服务器推: "！"（逐字推送）
  用户发: "帮我查个东西"（随时打断）
  服务器推: "好的，正在查..."
```

## 二、基础 WebSocket 端点

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 第1步：接受连接（握手）
    await websocket.accept()
    
    try:
        while True:
            # 第2步：接收客户端消息（阻塞等待）
            data = await websocket.receive_text()
            
            # 第3步：处理并回复
            await websocket.send_text(f"你说了: {data}")
    
    except WebSocketDisconnect:
        # 第4步：客户端断开连接
        print("客户端断开")
```

### 连接生命周期

```
客户端                        服务器
  │                            │
  │── HTTP Upgrade 请求 ──────→│  握手
  │←── 101 Switching ─────────│  协议切换
  │                            │
  │── send_text("你好") ─────→│  双向通信
  │←── send_text("你好！") ───│
  │                            │
  │── close() ───────────────→│  断开
  │                            │
```

## 三、Agent 对话 WebSocket（完整实现）

```python
import asyncio
import json
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver

app = FastAPI()

# 初始化 Agent（启动时一次）
checkpointer = InMemorySaver()
agent = create_agent("openai:gpt-4o", tools=[], checkpointer=checkpointer)

# 管理多个客户端连接
class ConnectionManager:
    def __init__(self):
        self.active: dict[str, WebSocket] = {}  # thread_id → websocket
    
    async def connect(self, thread_id: str, websocket: WebSocket):
        await websocket.accept()
        self.active[thread_id] = websocket
    
    def disconnect(self, thread_id: str):
        self.active.pop(thread_id, None)
    
    async def send_token(self, thread_id: str, token: str):
        """向指定客户端推送一个 token"""
        ws = self.active.get(thread_id)
        if ws:
            await ws.send_json({"type": "token", "content": token})
    
    async def send_event(self, thread_id: str, event_type: str, data: dict):
        """向指定客户端推送事件"""
        ws = self.active.get(thread_id)
        if ws:
            await ws.send_json({"type": event_type, **data})

manager = ConnectionManager()

@app.websocket("/chat/{thread_id}")
async def agent_chat(websocket: WebSocket, thread_id: str):
    """Agent 对话 WebSocket 端点"""
    await manager.connect(thread_id, websocket)
    
    try:
        while True:
            # 接收用户消息
            raw = await websocket.receive_text()
            message = json.loads(raw)
            user_text = message.get("content", "")
            
            # 通知客户端：开始处理
            await manager.send_event(thread_id, "start", {"message": user_text})
            
            # 流式调用 Agent，逐 token 推送
            config = {"configurable": {"thread_id": thread_id}}
            async for event in agent.astream_events(
                {"messages": [HumanMessage(content=user_text)]},
                config=config,
                version="v3",
            ):
                kind = event["event"]
                
                if kind == "on_chat_model_stream":
                    content = event["data"]["chunk"].content
                    if content:
                        await manager.send_token(thread_id, content)
                
                elif kind == "on_tool_start":
                    await manager.send_event(thread_id, "tool_start", {
                        "name": event["name"],
                    })
                
                elif kind == "on_tool_end":
                    await manager.send_event(thread_id, "tool_end", {
                        "name": event["name"],
                        "result": str(event["data"].get("output", ""))[:200],
                    })
            
            # 通知客户端：回复完成
            await manager.send_event(thread_id, "done", {})
    
    except WebSocketDisconnect:
        manager.disconnect(thread_id)
```

### 消息协议

```json
// 客户端 → 服务器
{"content": "帮我查一下天气"}

// 服务器 → 客户端（逐条推送）
{"type": "start", "message": "帮我查一下天气"}
{"type": "tool_start", "name": "weather_search"}
{"type": "tool_end", "name": "weather_search", "result": "北京: 晴天 28°C"}
{"type": "token", "content": "根据"}
{"type": "token", "content": "查询"}
{"type": "token", "content": "结果，"}
{"type": "token", "content": "北京今天晴天，28°C。"}
{"type": "done"}
```

## 四、客户端代码（Python）

```python
import asyncio
import json
import websockets

async def chat_client():
    thread_id = "user_001"
    uri = f"ws://localhost:8000/chat/{thread_id}"
    
    async with websockets.connect(uri) as ws:
        # 发送消息
        await ws.send(json.dumps({"content": "你好，介绍一下自己"}))
        
        # 接收流式回复
        full_reply = ""
        while True:
            raw = await ws.recv()
            msg = json.loads(raw)
            
            if msg["type"] == "token":
                print(msg["content"], end="", flush=True)  # 逐字打印
                full_reply += msg["content"]
            elif msg["type"] == "tool_start":
                print(f"\n[调用工具: {msg['name']}]", flush=True)
            elif msg["type"] == "done":
                print("\n--- 回复完成 ---")
                break

asyncio.run(chat_client())
```

### 客户端代码（JavaScript/前端）

```javascript
const ws = new WebSocket("ws://localhost:8000/chat/user_001");

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    
    if (msg.type === "token") {
        // 逐字追加到页面
        document.getElementById("reply").textContent += msg.content;
    } else if (msg.type === "tool_start") {
        console.log(`调用工具: ${msg.name}`);
    } else if (msg.type === "done") {
        console.log("回复完成");
    }
};

// 发送消息
function sendMessage(text) {
    ws.send(JSON.stringify({ content: text }));
}
```

## 五、中途取消（用户打断 Agent）

WebSocket 的优势：用户可以随时打断正在执行的 Agent。

```python
@app.websocket("/chat/{thread_id}")
async def agent_chat_with_cancel(websocket: WebSocket, thread_id: str):
    await manager.connect(thread_id, websocket)
    current_task = None
    
    try:
        while True:
            raw = await websocket.receive_text()
            message = json.loads(raw)
            
            # 收到取消指令
            if message.get("action") == "cancel":
                if current_task and not current_task.done():
                    current_task.cancel()
                    await manager.send_event(thread_id, "cancelled", {})
                continue
            
            # 正常消息：启动 Agent 任务
            user_text = message.get("content", "")
            current_task = asyncio.create_task(
                run_agent_stream(thread_id, user_text)
            )
    
    except WebSocketDisconnect:
        if current_task:
            current_task.cancel()
        manager.disconnect(thread_id)

async def run_agent_stream(thread_id: str, user_text: str):
    """后台执行 Agent 流式输出"""
    config = {"configurable": {"thread_id": thread_id}}
    async for event in agent.astream_events(
        {"messages": [HumanMessage(content=user_text)]},
        config=config,
        version="v3",
    ):
        if event["event"] == "on_chat_model_stream":
            content = event["data"]["chunk"].content
            if content:
                await manager.send_token(thread_id, content)
    await manager.send_event(thread_id, "done", {})
```

## 六、WebSocket vs SSE 选择

| 维度 | SSE（流式响应） | WebSocket |
|------|----------------|-----------|
| 方向 | 单向（服务器→客户端） | 双向 |
| 中途取消 | 需要额外 API 端点 | 直接发消息即可 |
| 多轮对话 | 每轮一个新请求 | 一个连接搞定 |
| 实现复杂度 | 低 | 中 |
| 适用场景 | 简单流式输出 | 聊天室、实时协作、需要打断 |

**建议：** 简单的"一问一答 + 流式输出"用 SSE 就够了（见 [[fastapi/流式响应]]）。需要"多轮对话 + 中途取消 + 服务器主动推送"时用 WebSocket。

## 七、常见陷阱

### 1. 忘记处理断连

```python
# 错误：客户端断开后继续发消息 → 报错
await websocket.send_text("hello")  # 客户端已断开！

# 正确：用 try-except 包裹
try:
    await websocket.send_text("hello")
except WebSocketDisconnect:
    manager.disconnect(thread_id)
```

### 2. 在 WebSocket 中用同步 invoke

```python
# 错误：阻塞事件循环，所有连接卡住
result = agent.invoke(...)  # 同步！

# 正确：用异步
result = await agent.ainvoke(...)
```

### 3. 没有心跳机制

```python
# 长时间无消息，连接可能被中间代理（Nginx）断开
# 解决：定期发 ping
async def heartbeat(websocket: WebSocket):
    while True:
        await asyncio.sleep(30)
        await websocket.send_json({"type": "ping"})
```

## 相关

- [[fastapi/流式响应|流式响应（SSE）]] — 单向流式，更简单的替代方案
- [[fastapi/与LangChain集成|与 LangChain 集成]] — HTTP 方式的 Agent API
- [[fastapi/异步处理|异步处理]] — async/await 基础
- [[fastapi/FastAPI概述|<- 返回 FastAPI 概述]]
