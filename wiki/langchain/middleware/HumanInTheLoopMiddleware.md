---
tags: [langchain, middleware, safety, hitl, approval]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#human-in-the-loop
langchain_version: ">=1.0"
---

# HumanInTheLoopMiddleware（人在环中间件）

## 一、定义与作用

`HumanInTheLoopMiddleware` 在 Agent **调用关键工具前暂停**，等待**人工审批**（approve / edit / reject）。只有获得批准后，工具才会真正执行。

它把"Agent 自动执行"变成"Agent 半自动执行"——重要操作必须有人点头。

**核心价值**：给 Agent 加上"安全锁"。涉及钱、数据安全、对外通信的操作，不会未经允许就执行。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 财务操作 | Agent 提议"给供应商转账 10 万"，必须财务经理批准 |
| 邮件外发 | Agent 写好邮件草稿，等用户审阅后才发 |
| 数据库变更 | `DROP TABLE` / `DELETE` 前需要 DBA 审批 |
| 文件删除 | `delete_file` 必须二次确认 |
| API 调用 | 付费 API 必须经过业务方确认 |

**不适合 HITL**：
- 纯查询操作（`web_search`、`read_file`）— 没必要打断
- 高频低风险操作（`calculator`）— 打断用户会烦

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `interrupt_on` | `dict[str, bool \| AllowDecision]` | 关键参数：每个工具的审批配置 |
| `description_prompt` | `string` | 可选，描述工具的提示模板 |

### interrupt_on 字典的取值

| 取值 | 行为 |
|------|------|
| `True` | 必须审批，可批准/编辑/拒绝 |
| `False` | 不需要审批（默认） |
| `AllowDecision` 对象 | 细粒度控制允许的决策（仅 approve / 仅 approve+edit） |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

# ⚠️ 必须有 checkpointer 才能跨中断恢复
checkpointer = InMemorySaver()

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[read_email, send_email, delete_file, web_search],
    checkpointer=checkpointer,
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,     # 发邮件前必须审批
                "delete_file": True,    # 删文件前必须审批
                "web_search": False,    # 搜索不审批
                "read_email": False,    # 读邮件不审批
            },
        ),
    ],
)
```

### 细粒度决策控制

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware, AllowDecision

HumanInTheLoopMiddleware(
    interrupt_on={
        # 只能批准或拒绝，不能编辑
        "send_email": AllowDecision(allowed_decisions=["approve", "reject"]),
        # 可以编辑后再批准
        "execute_query": AllowDecision(allowed_decisions=["approve", "edit", "reject"]),
    },
)
```

### 配合 LangGraph Command 恢复

```python
from langgraph.types import Command
from langchain_core.messages import HumanMessage

config = {"configurable": {"thread_id": "user_001"}}

# 第 1 次：Agent 提议发邮件
result = agent.invoke(
    {"messages": [HumanMessage("帮我发邮件给 alice@example.com")]},
    config=config,
)

# 看到 interrupt 信号后，提示用户
# 用户输入"批准"
# 用 Command 恢复执行
final = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
)
```

## 五、注意事项

1. **必须有 checkpointer**：没有 checkpointer，Agent 暂停后无法恢复状态

2. **必须用 `thread_id` 保持会话**：HITL 依赖 checkpointer 的 `thread_id` 恢复

3. **首次 `invoke` 会返回 `__interrupt__`**：需要从结果中提取中断信息，提示用户决策

4. **可与流式输出配合**：但中断点会触发特殊的 stream event，需要前端专门处理

5. **审批粒度到工具级别**：不能针对"工具 + 参数"做更细粒度判断。如需要，写自定义中间件

6. **多个工具同时调用**：如果模型一次返回多个 `tool_calls`，每个工具按 `interrupt_on` 独立决定

7. **适合 batch 操作**：如果你想"批量审批 5 个操作"，可以设计一个工具，HITL 审批工具本身，内部批量执行

8. **在 LangGraph 工作流中嵌入**：把整个 Agent 作为节点，HITL 机制会传递到父图

## 六、相关页面

- [[langchain/middleware/ToolCallLimitMiddleware]] — 用调用次数限制来补充 HITL
- [[langchain/middleware/PIIMiddleware]] — 配合 PII 检测，先脱敏再审批
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/memory/Checkpointer存储后端]] — 选择合适的 checkpointer
