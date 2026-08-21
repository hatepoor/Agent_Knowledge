---
tags: [deepagents, HITL, human-in-the-loop, 人工审批, middleware]
created: 2026-08-03
updated: 2026-08-03
type: concept
source_url: https://docs.langchain.com/oss/python/deepagents/customization
deepagents_version: ">=0.7"
---

# HumanInTheLoop 详解

HITL（Human-in-the-Loop，人在环）让 Agent 在**执行敏感工具调用前暂停**，等待人工审批。是 DeepAgents 默认中间件栈的第 13 层（最后一层），只在 `interrupt_on` 非空时安装。

## 一、核心机制

### 暂停点

HITL 拦截的是**工具调用**，不是消息生成。流程：

```
LLM 决定调用 send_email 工具
  │
  ▼
HumanInTheLoopMiddleware 检查 interrupt_on
  │  send_email: True → 暂停！
  ▼
图执行中断（__interrupt__），返回给调用方
  │
  ▼
人工审批：approve / edit / reject / respond
  │
  ▼
Command(resume=...) 恢复图执行
```

### 必需条件

```python
# ❌ interrupt_on 不配 checkpointer 会直接报错
agent = create_deep_agent(model=llm, interrupt_on={"send_email": True})
# HITL 需要 checkpointer

# ✅ 正确配置
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
agent = create_deep_agent(
    model=llm,
    tools=[send_email, delete_file],
    interrupt_on={
        "send_email": True,
        "delete_file": {"allowed_decisions": ["approve", "reject"]},
    },
    checkpointer=checkpointer,   # ← 必需
)
```

checkpointer 是 HITL 的地基：中断时必须把图的执行状态保存下来，恢复时才能从暂停点继续。

## 二、interrupt_on 配置

### 两种配置形式

```python
# 形式 1：布尔值 —— 允许全部决策（默认）
interrupt_on={"send_email": True}

# 形式 2：字典 —— 限制该工具的决策类型
interrupt_on={
    "delete_file": {"allowed_decisions": ["approve", "reject"]},
    # 只允许批准或拒绝，不允许修改参数
}
```

### 四种决策类型

| 决策 | 含义 | 恢复方式 |
|------|------|----------|
| `approve` | 批准，按原参数执行工具 | `Command(resume={"decisions": [{"type": "approve"}]})` |
| `reject` | 拒绝，跳过该工具调用 | `Command(resume={"decisions": [{"type": "reject"}]})` |
| `edit` | 修改工具参数后执行 | `Command(resume={"decisions": [{"type": "edit", "args": {...}}]})` |
| `respond` | 不执行工具，直接给 LLM 发消息 | `Command(resume={"decisions": [{"type": "respond", "message": "..."}}]})` |

`allowed_decisions` 默认值为 `["approve", "edit", "reject", "respond"]`，可以按工具收紧。

## 三、完整流程（FastAPI 示例）

```python
from typing import Annotated
from langgraph.types import Command
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph.message import add_messages

from deepagents import create_deep_agent

agent = create_deep_agent(
    model=llm,
    tools=[send_email],
    interrupt_on={"send_email": True},
    checkpointer=MemorySaver(),
)

# 第 1 步：发起对话（会中断）
config = {"configurable": {"thread_id": "thread-1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "给张三发一封会议邀请邮件"}]},
    config=config,
)

# 检查是否中断
if result.get("__interrupt__"):
    interrupt = result["__interrupt__"][0]
    print("等待审批的工具调用:", interrupt.value)
    # 输出: 工具名 + 参数（收件人、内容等）

# 第 2 步：人工审批后恢复
result2 = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
)
# 恢复后：send_email 按原参数执行，Agent 继续后续流程

# 审批时修改参数
result3 = agent.invoke(
    Command(resume={"decisions": [
        {"type": "edit", "args": {"to": "zhangsan@example.com", "body": "修改后的内容"}}
    ]}),
    config=config,
)
```

### 拒绝场景

```python
# 拒绝执行 —— 工具被跳过，Agent 继续（或结束）
result = agent.invoke(
    Command(resume={"decisions": [{"type": "reject"}]}),
    config=config,
)
```

### 多个待审批工具

LLM 一轮可能调用多个工具，中断会包含多个决策，恢复时**按顺序逐个审批**：

```python
# 待审批: [send_email, delete_file]
agent.invoke(
    Command(resume={"decisions": [
        {"type": "approve"},        # 批准 send_email
        {"type": "reject"},         # 拒绝 delete_file
    ]}),
    config=config,
)
```

决策列表必须与中断的工具调用列表一一对应。

## 四、WebSocket 实时审批

```python
@app.websocket("/chat")
async def chat_ws(ws: WebSocket):
    await ws.accept()
    config = {"configurable": {"thread_id": "thread-1"}}

    while True:
        data = await ws.receive_json()
        if data["type"] == "message":
            # 发起对话
            result = agent.invoke(
                {"messages": [{"role": "user", "content": data["content"]}]},
                config=config,
            )
            if result.get("__interrupt__"):
                # 推送审批请求给前端
                interrupt = result["__interrupt__"][0]
                await ws.send_json({
                    "type": "approval_request",
                    "tool": interrupt.value["tool"],
                    "args": interrupt.value["args"],
                })
                # 等前端回传审批结果
                approval = await ws.receive_json()
                await ws.send_json({
                    "type": "agent_response",
                    "content": agent.invoke(
                        Command(resume={"decisions": approval["decisions"]}),
                        config=config,
                    )["messages"][-1].content,
                })
```

## 五、中间件栈中的位置

HITL 是默认栈的**第 13 层**（最后一层），位于 Memory 之后：

```
12. MemoryMiddleware        ← 先加载记忆
13. HumanInTheLoopMiddleware ← 最后审批（拦截所有工具调用）
```

放在最后一层意味着：**所有**中间件提供的工具（包括 `write_todos`、`write_file`、`task`）都可以被 `interrupt_on` 拦截，只要你把工具名加进去。

## 六、适用场景与最佳实践

| 场景 | 配置建议 |
|------|----------|
| 发邮件/消息（不可撤销） | `interrupt_on={"send_email": True}` |
| 删除文件（不可恢复） | `interrupt_on={"delete_file": {"allowed_decisions": ["approve", "reject"]}}` |
| 写生产数据库 | `interrupt_on={"update_db": True}` |
| 部署/发布操作 | `interrupt_on={"deploy": True}` |
| CLI 工具快速使用 | `auto_approve` 参数（跳过 HITL） |

**只拦截真正不可逆或高影响的操作**，不要什么都拦——每拦截一次就多一轮人工交互，Agent 的自主性会大打折扣。

## 七、相关页面

- [[deepagents/middleware/中间件栈概述]] — HITL 在 13 层栈中的位置
- [[deepagents/customization/create_deep_agent参数详解]] — interrupt_on 参数
- [[deepagents/production/生产部署]] — 生产环境 HITL 配置
- [[deepagents/production/Streaming与事件流]] — streaming 中捕获 __interrupt__
- [[langchain/middleware/中间件概述]] — LangChain HITL 中间件基础
