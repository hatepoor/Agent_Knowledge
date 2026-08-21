---
tags: [deepagents, streaming, events, websocket, 实时]
created: 2026-08-03
updated: 2026-08-03
type: howto
source_url: https://docs.langchain.com/oss/python/deepagents/customization
deepagents_version: ">=0.7"
---

# DeepAgents Streaming 与事件流

DeepAgents 构建在 LangGraph 之上，支持两种流式方案：**`astream`**（结构化、推荐）和 **`astream_events`**（细粒度事件）。

## 一、astream —— 三种流模式（推荐）

DeepAgents 基于 LangGraph 的 `astream`，固定组合三种 stream_mode，每个 chunk 是一个三元组 `(namespace, stream_mode, data)`：

```python
async for chunk in agent.astream(
    stream_input,
    stream_mode=["messages", "updates", "custom"],
    subgraphs=True,
    config=config,
):
    namespace, current_stream_mode, data = chunk

    if current_stream_mode == "custom":
        # 子代理事件 / 评分事件（dict）
        pass
    elif current_stream_mode == "updates":
        if "__interrupt__" in data:
            # HITL 中断事件
            pass
    elif current_stream_mode == "messages":
        message_obj, metadata = data
        if isinstance(message_obj, AIMessage):
            # 模型回复（可能是增量 chunk）
            pass
        elif isinstance(message_obj, ToolMessage):
            # 工具执行结果
            pass
```

| stream_mode | data 内容 | 用途 |
|-------------|-----------|------|
| `messages` | `(AIMessage \| ToolMessage, metadata)` 元组 | 展示 LLM 增量回复、工具调用与结果 |
| `updates` | 状态更新 dict，HITL 时含 `__interrupt__` | 检测人工审批中断 |
| `custom` | 子代理/评分事件 dict | 展示子代理进度 |

### 增量文本拼接

```python
async for namespace, mode, data in agent.astream(input, stream_mode=["messages"], subgraphs=True):
    if mode == "messages":
        msg, metadata = data
        if isinstance(msg, AIMessage) and msg.content:
            print(msg.content, end="", flush=True)  # 逐字输出
```

## 二、astream_events —— 细粒度事件

需要精确事件（工具开始/结束、LLM 流式 token）时用 `astream_events`：

```python
async for event in agent.astream_events(input, config=config, version="v3"):
    kind = event["event"]

    if kind == "on_chat_model_stream":
        # LLM token 增量
        chunk = event["data"]["chunk"]
        if chunk.content:
            print(chunk.content, end="", flush=True)

    elif kind == "on_tool_start":
        # 工具开始执行
        print(f"\n[工具] {event['name']} 参数: {event['data']['input']}")

    elif kind == "on_tool_end":
        # 工具执行完成
        print(f"\n[工具完成] {event['name']}")

    elif kind == "on_chain_end":
        # 节点执行完成
        print(f"\n[节点] {event['name']} 完成")
```

常用事件类型：

| 事件 | 时机 | 关键字段 |
|------|------|----------|
| `on_chat_model_start` | LLM 调用开始 | `data.input` |
| `on_chat_model_stream` | LLM 输出增量 | `data.chunk.content` |
| `on_chat_model_end` | LLM 调用结束 | `data.output` |
| `on_tool_start` | 工具执行开始 | `name`, `data.input` |
| `on_tool_end` | 工具执行结束 | `name`, `data.output` |
| `on_tool_error` | 工具执行出错 | `name`, `data.error` |
| `on_chain_end` | 节点结束 | `name` |

## 三、WebSocket 实时推送（FastAPI）

把流式输出推给前端：

```python
import json
from fastapi import FastAPI, WebSocket

@app.websocket("/chat/stream")
async def chat_stream_ws(ws: WebSocket):
    await ws.accept()
    config = {"configurable": {"thread_id": "thread-1"}}

    async for namespace, mode, data in agent.astream(
        {"messages": [{"role": "user", "content": "帮我写一个快速排序"}]},
        stream_mode=["messages", "updates", "custom"],
        subgraphs=True,
        config=config,
    ):
        if mode == "messages":
            msg, _ = data
            if isinstance(msg, AIMessage) and msg.content:
                # 推给前端，前端追加显示
                await ws.send_text(json.dumps({
                    "type": "token",
                    "content": msg.content,
                }))
            elif isinstance(msg, ToolMessage):
                await ws.send_text(json.dumps({
                    "type": "tool_result",
                    "name": msg.name,
                    "content": msg.content[:200],  # 截断避免过大
                }))
        elif mode == "updates" and "__interrupt__" in data:
            # HITL：推送审批请求
            await ws.send_text(json.dumps({
                "type": "approval_request",
                "data": data["__interrupt__"],
            }))
        elif mode == "custom":
            await ws.send_text(json.dumps({
                "type": "subagent_event",
                "data": data,
            }))

    await ws.send_text(json.dumps({"type": "done"}))
```

## 四、SSE（Server-Sent Events）

浏览器原生支持 SSE，不需要 WebSocket：

```python
from fastapi.responses import StreamingResponse

@app.post("/chat/sse")
async def chat_sse():
    async def generator():
        async for chunk in agent.astream(
            {"messages": [{"role": "user", "content": "..."}]},
            stream_mode=["messages"],
            config=config,
        ):
            namespace, mode, data = chunk
            if mode == "messages":
                msg, _ = data
                if isinstance(msg, AIMessage) and msg.content:
                    yield f"data: {json.dumps({'content': msg.content})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generator(), media_type="text/event-stream")
```

## 五、Tool call chunks（工具参数流式）

工具调用参数是分块传输的，需要拼接：

```python
async for event in agent.astream_events(input, version="v3"):
    if event["event"] == "on_chain_stream":
        chunk = event["data"]["chunk"]
        if hasattr(chunk, "tool_call_chunks"):
            for tc in chunk.tool_call_chunks:
                # tc: {"id": "...", "name": "tool_name", "args": "...", "index": 0}
                print(f"工具 {tc['name']} 参数片段: {tc['args']}")
```

## 六、与 HITL 配合

Streaming 模式下检测中断并恢复：

```python
interrupted = False
async for namespace, mode, data in agent.astream(
    {"messages": [{"role": "user", "content": "发送邮件"}]},
    stream_mode=["messages", "updates"],
    subgraphs=True,
    config=config,
):
    if mode == "updates" and "__interrupt__" in data:
        interrupted = True
        # 展示待审批信息，然后人工决策
        break

if interrupted:
    # 恢复执行
    async for chunk in agent.astream(
        Command(resume={"decisions": [{"type": "approve"}]}),
        stream_mode=["messages", "updates"],
        config=config,
    ):
        pass  # 处理恢复后的输出
```

## 七、性能与注意事项

| 注意点 | 说明 |
|--------|------|
| 版本参数 | `astream_events` 必须传 `version="v3"` |
| 子代理事件 | 需要 `subgraphs=True` 才能看到子代理的流 |
| Token 量 | `messages` 模式比 `custom` 模式事件多，按需选择模式 |
| 连接断开 | 前端断连时停止生成，用 generator 的 `finally` 清理 |
| 多客户端 | 每个 WebSocket 连接对应独立 `thread_id`，不要共享 |

## 相关页面

- [[deepagents/middleware/HumanInTheLoop详解]] — streaming 中处理 __interrupt__
- [[deepagents/production/生产部署]] — 生产环境流式 API 骨架
- [[fastapi/WebSocket与Agent实时通信]] — WebSocket 基础
- [[fastapi/后台任务与长时间Agent作业]] — 长时间任务处理
