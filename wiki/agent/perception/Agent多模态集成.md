---
tags: [agent, multimodal, image, base64, practical]
type: howto
created: 2026-07-26
updated: 2026-07-26
---

# Agent 多模态集成

## 前置条件

模型本身必须支持多模态输入（如 GPT-4o、Claude 3、Gemini、通义千问 VL）。LangChain 只负责传递消息格式，不会帮你校验模型能力。

## 核心原理

多模态的关键：**模型 API 是远程服务，无法访问用户本地文件**。所有非文本内容必须经过你的 Agent 进程读取、编码后，通过 HTTP 请求体发给模型。

```
本地文件  →  Agent 进程读取  →  base64 编码  →  组装消息  →  HTTP POST  →  模型解析
```

## 消息格式

将 `HumanMessage` 的 `content` 从字符串改为**内容块列表**，每个块是一个 `dict`：

```python
from langchain_core.messages import HumanMessage

msg = HumanMessage(content=[
    {"type": "text", "text": "分析这张图"},
    {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}},
])
```

图片支持两种传入方式：

```python
# 方式 1：URL（模型服务端去下载，要求公网可访问）
{"type": "image_url", "image_url": {"url": "https://example.com/cat.png"}}

# 方式 2：Base64 内嵌（适合本地文件、上传文件）
{"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}}
```

## 本地文件处理

用户本地有图片文件时，Agent 进程负责读取并编码：

```python
import base64
from langchain_core.messages import HumanMessage

def build_image_message(text: str, image_path: str) -> HumanMessage:
    """读取本地图片，构建多模态消息"""
    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()

    # 根据后缀推断 MIME 类型
    suffix = image_path.rsplit(".", 1)[-1].lower()
    mime_map = {
        "png": "image/png", "jpg": "image/jpeg",
        "jpeg": "image/jpeg", "gif": "image/gif",
        "webp": "image/webp",
    }
    mime = mime_map.get(suffix, "image/png")

    return HumanMessage(content=[
        {"type": "text", "text": text},
        {"type": "image_url", "image_url": {"url": f"data:{mime};base64,{img_b64}"}},
    ])

# 使用
msg = build_image_message("这张图里有什么？", r"C:\Users\20624\Desktop\photo.png")
result = await agent.ainvoke({"messages": [msg]})
```

## 结合 create_agent

用 `create_agent` 创建的 Agent，调用方式与纯文本完全一致，只需在 `messages` 中传入多模态消息：

```python
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
import base64

agent = create_agent(model="gpt-4o", tools=[search, calculator])

# 读取本地图片
with open("error_screenshot.png", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

result = await agent.ainvoke({
    "messages": [
        HumanMessage(content=[
            {"type": "text", "text": "分析截图中的报错，告诉我怎么修复"},
            {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}},
        ])
    ]
})

# Agent 正常推理：看到图片 → 决定调哪个工具 → 返回文本回复
# 工具调用流程不受多模态影响，不需要修改任何工具定义
```

## 多图输入

一条消息可以放多张图：

```python
msg = HumanMessage(content=[
    {"type": "text", "text": "对比这两张设计稿，哪个布局更好？"},
    {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img1_b64}"}},
    {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img2_b64}"}},
])
```

## 音频输入（部分模型支持）

GPT-4o-audio 等模型支持音频块：

```python
with open("recording.wav", "rb") as f:
    audio_b64 = base64.b64encode(f.read()).decode()

msg = HumanMessage(content=[
    {"type": "text", "text": "这段录音说了什么？"},
    {
        "type": "input_audio",
        "input_audio": {"data": audio_b64, "format": "wav"},
    },
])
```

## 结合 FastAPI 文件上传

用户通过浏览器上传文件时，FastAPI 的 `UploadFile` 帮你拿到字节流，后续处理和读取本地文件完全一致：

```python
from fastapi import UploadFile
import base64

@app.post("/chat")
async def chat(message: str, file: UploadFile | None = None):
    content = [{"type": "text", "text": message}]

    if file:
        raw = await file.read()  # 拿到 bytes，和 open().read() 一样
        b64 = base64.b64encode(raw).decode()
        mime = file.content_type or "image/png"
        content.append({
            "type": "image_url",
            "image_url": {"url": f"data:{mime};base64,{b64}"},
        })

    result = await agent.ainvoke({
        "messages": [HumanMessage(content=content)]
    })
    return {"reply": result["messages"][-1].content}
```

CLI 和 Web 的区别只是"读文件"这一步：CLI 自己 `open()` 读，Web 通过 `UploadFile` 接收。后面的 base64 编码、消息组装代码完全相同。

## WebSocket 场景

WebSocket 传输的是文本/JSON，图片需要客户端先转 base64 再发送：

```python
# 客户端
import base64, json, websockets

with open("photo.png", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

await ws.send(json.dumps({
    "message": "这张图是什么？",
    "image": img_b64,
    "mime": "image/png",
}))
```

```python
# 服务端
data = json.loads(await ws.receive_text())
content = [{"type": "text", "text": data["message"]}]

if data.get("image"):
    content.append({
        "type": "image_url",
        "image_url": {"url": f"data:{data['mime']};base64,{data['image']}"},
    })
```

## Token 消耗

图片会被模型转换为 token 计费：

| 分辨率 | 大致 token 消耗 |
|--------|----------------|
| 512×512 | ~200 tokens |
| 1024×1024 | ~800 tokens |
| 2048×2048 | ~3000 tokens |

多图、高分辨率图会显著增加成本。可以考虑在发送给模型前做压缩：

```python
from PIL import Image
import io

def compress_image(image_bytes: bytes, max_size=1024) -> bytes:
    """压缩图片到指定最大尺寸，减少 token 消耗"""
    img = Image.open(io.BytesIO(image_bytes))
    img.thumbnail((max_size, max_size))
    buf = io.BytesIO()
    img.save(buf, format="JPEG", quality=85)
    return buf.getvalue()
```

## Checkpointer 存储膨胀

如果使用了 checkpointer 保存对话历史，图片的 base64 数据也会被存入数据库。长期运行会导致：

- SQLite 文件快速膨胀（单张图 base64 可能数百 KB）
- PostgreSQL 存储压力增大

解决方式——对历史消息做裁剪，把图片块替换为文本占位：

```python
from langchain_core.messages import HumanMessage

def strip_images_from_history(messages: list) -> list:
    """将历史消息中的图片替换为占位文本，防止数据库膨胀"""
    cleaned = []
    for msg in messages:
        if isinstance(msg.content, list):
            new_content = []
            for block in msg.content:
                if block.get("type") == "image_url":
                    new_content.append({"type": "text", "text": "[图片已省略]"})
                else:
                    new_content.append(block)
            cleaned.append(msg.model_copy(update={"content": new_content}))
        else:
            cleaned.append(msg)
    return cleaned
```

只对**历史消息**裁剪，当前这条用户消息保留完整图片数据。

## 工具调用不受影响

模型看到图片后做推理，决定调哪个工具——这个流程和纯文本一模一样。不需要为多模态修改任何工具定义。

```python
# 工具定义保持不变
@tool
def search_web(query: str) -> str:
    """搜索网络信息"""
    ...

# Agent 看到图片后，如果需要搜索，照常调用 search_web
# 多模态只影响输入给模型的内容，不影响 Agent 的推理循环
```

## 常见错误

| 错误 | 原因 | 修复 |
|------|------|------|
| `400 Bad Request` | 模型不支持图片输入 | 确认使用的是多模态模型（如 gpt-4o 而非 gpt-4o-mini） |
| `Image too large` | 图片超过模型限制 | 压缩后再编码，一般限制在 20MB 以内 |
| base64 解码失败 | 编码不完整或格式错误 | 检查 `data:image/png;base64,` 前缀是否正确 |
| 历史对话越来越慢 | checkpointer 存了大量 base64 | 对历史消息做图片裁剪 |

## 相关页面

- [[langchain/modules/langchain_messages/多模态消息|消息格式详解]]
- [[agent/perception/多模态|多模态概念]]
- [[fastapi/文件上传与RAG文档接口|FastAPI 文件上传]]
- [[langchain/memory/Checkpointer存储后端|Checkpointer 存储]]
