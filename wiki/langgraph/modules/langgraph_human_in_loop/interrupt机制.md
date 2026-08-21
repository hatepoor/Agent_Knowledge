---
tags: [langgraph, human-in-loop, interrupt, command, resume]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# interrupt() 机制

`interrupt()` 是 LangGraph 人机交互的核心函数，它允许你在图执行过程中的任意点暂停，等待人类输入，然后恢复执行。理解其工作原理对于构建可靠的人机交互应用至关重要。

## 基本语法

```python
from langgraph.types import interrupt, Command

# 在节点中调用
def human_input_node(state):
    # 暂停执行，等待人类输入
    human_value = interrupt(interrupt_value)
    
    # 恢复后继续执行
    # human_value 包含人类提供的输入
    return {"result": human_value}
```

## 工作流程

### 1. 暂停阶段

当调用 `interrupt(value)` 时：

1. 图立即暂停执行
2. 当前状态保存到检查点
3. `interrupt()` 的 `value` 参数被记录
4. 图返回，等待外部输入

```python
def node_with_interrupt(state):
    # 执行一些逻辑
    result = process(state["data"])
    
    # 暂停，传递上下文信息给人类
    human_input = interrupt({
        "context": "需要审批",
        "data": result,
        "options": ["approve", "reject", "modify"]
    })
    
    # 这里不会立即执行
    # 等待人类输入后才会继续
```

### 2. 恢复阶段

使用 `Command(resume=value)` 恢复执行：

```python
from langgraph.types import Command

# 恢复执行并传递人类输入
result = app.invoke(
    Command(resume="approve"),
    config={"configurable": {"thread_id": "1"}}
)
```

恢复后：

1. `interrupt()` 返回 `resume` 的值
2. 节点继续执行
3. 图继续运行直到完成或再次遇到 interrupt

## 完整示例

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import MemorySaver
from typing_extensions import TypedDict

class State(TypedDict):
    data: str
    approved: bool
    result: str

def process_node(state: State):
    """处理数据"""
    processed = f"已处理: {state['data']}"
    return {"data": processed}

def approval_node(state: State):
    """等待人类审批"""
    # 暂停，传递审批请求
    decision = interrupt({
        "message": "请审批以下处理结果",
        "data": state["data"],
        "options": ["approve", "reject"]
    })
    
    # 恢复后，decision 包含人类的决策
    if decision == "approve":
        return {"approved": True, "result": state["data"]}
    else:
        return {"approved": False, "result": "已拒绝"}

def finalize_node(state: State):
    """完成处理"""
    if state["approved"]:
        return {"result": f"最终结果: {state['result']}"}
    else:
        return {"result": "流程已取消"}

# 构建图
graph = StateGraph(State)
graph.add_node("process", process_node)
graph.add_node("approval", approval_node)
graph.add_node("finalize", finalize_node)

graph.add_edge(START, "process")
graph.add_edge("process", "approval")
graph.add_edge("approval", "finalize")
graph.add_edge("finalize", END)

# 编译图（必须使用检查点）
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 执行流程
config = {"configurable": {"thread_id": "1"}}

# 第一次执行（会在 approval 节点暂停）
initial_state = {"data": "原始数据", "approved": False, "result": ""}
result = app.invoke(initial_state, config)

print("图已暂停，等待审批...")
print(f"当前状态: {result}")

# 人类审批
app.invoke(Command(resume="approve"), config)

# 获取最终结果
final_state = app.get_state(config)
print(f"最终状态: {final_state.values}")
```

## interrupt_before 和 interrupt_after

除了在节点内部使用 `interrupt()`，还可以在编译图时指定在哪些节点前后暂停。

### interrupt_before

在节点**执行前**暂停：

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["sensitive_node"]
)

# 执行图
result = app.invoke(inputs, config)

# 图会在 sensitive_node 执行前暂停
# 此时 sensitive_node 还没有执行

# 恢复执行
app.invoke(Command(resume=None), config)
```

### interrupt_after

在节点**执行后**暂停：

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_after=["review_node"]
)

# 执行图
result = app.invoke(inputs, config)

# 图会在 review_node 执行后暂停
# 此时 review_node 已经完成，但后续节点还没执行

# 恢复执行
app.invoke(Command(resume=None), config)
```

### 对比

| 特性 | interrupt() | interrupt_before | interrupt_after |
|------|-------------|------------------|-----------------|
| 暂停位置 | 节点内部任意位置 | 节点执行前 | 节点执行后 |
| 传递数据 | 可以传递上下文 | 不能传递上下文 | 不能传递上下文 |
| 灵活性 | 高 | 中 | 中 |
| 使用场景 | 复杂交互 | 简单审批 | 输出审核 |

## 常见模式

### 1. 审批模式

```python
def approval_node(state):
    """审批模式"""
    data_to_approve = state["pending_data"]
    
    # 请求审批
    decision = interrupt({
        "type": "approval",
        "data": data_to_approve,
        "message": "是否批准？"
    })
    
    # 处理决策
    if decision == "approve":
        return {"status": "approved", "data": data_to_approve}
    else:
        return {"status": "rejected", "data": None}
```

### 2. 编辑模式

```python
def edit_node(state):
    """编辑模式"""
    content = state["generated_content"]
    
    # 请求编辑
    edited_content = interrupt({
        "type": "edit",
        "content": content,
        "message": "请编辑以下内容，或直接提交"
    })
    
    # 使用编辑后的内容
    return {"content": edited_content}
```

### 3. 数据输入模式

```python
def data_entry_node(state):
    """数据输入模式"""
    required_fields = state["required_fields"]
    
    # 请求数据输入
    user_data = interrupt({
        "type": "data_entry",
        "fields": required_fields,
        "message": "请填写以下信息"
    })
    
    # 处理输入的数据
    return {"user_data": user_data}
```

### 4. 多步审批模式

```python
def multi_step_approval(state):
    """多步审批模式"""
    step = state.get("approval_step", 1)
    
    if step == 1:
        # 第一步：经理审批
        decision = interrupt({
            "step": 1,
            "message": "请经理审批"
        })
        
        if decision == "approve":
            return {"approval_step": 2}
        else:
            return {"status": "rejected"}
    
    elif step == 2:
        # 第二步：总监审批
        decision = interrupt({
            "step": 2,
            "message": "请总监审批"
        })
        
        if decision == "approve":
            return {"status": "approved"}
        else:
            return {"status": "rejected"}
```

### 5. 条件中断模式

```python
def conditional_interrupt(state):
    """条件中断模式"""
    risk_level = calculate_risk(state["data"])
    
    # 只在高风险时需要审批
    if risk_level > 0.8:
        decision = interrupt({
            "type": "high_risk_approval",
            "risk_level": risk_level,
            "data": state["data"],
            "message": "高风险操作，需要审批"
        })
        
        if decision != "approve":
            return {"status": "blocked"}
    
    # 低风险或已审批，继续执行
    return {"status": "proceed"}
```

## 状态管理

### 检查状态

```python
# 获取当前状态
state = app.get_state(config)
print(f"当前状态: {state.values}")
print(f"下一步: {state.next}")
```

### 更新状态

```python
# 在恢复前更新状态
app.update_state(config, {"data": "更新的数据"})

# 然后恢复执行
app.invoke(Command(resume="approve"), config)
```

### 状态历史

```python
# 获取状态历史
history = list(app.get_state_history(config))

for state_snapshot in history:
    print(f"时间戳: {state_snapshot.created_at}")
    print(f"状态: {state_snapshot.values}")
```

## 错误处理

### 超时处理

```python
import asyncio
from langgraph.types import interrupt, Command

async def approval_with_timeout(state, timeout_seconds=3600):
    """带超时的审批"""
    try:
        # 等待人类输入，设置超时
        decision = await asyncio.wait_for(
            interrupt({"message": "请审批"}),
            timeout=timeout_seconds
        )
        return {"decision": decision}
    except asyncio.TimeoutError:
        return {"decision": "timeout", "message": "审批超时"}
```

### 默认值处理

```python
def approval_with_default(state):
    """带默认值的审批"""
    decision = interrupt({
        "message": "请审批（30秒内未响应将自动拒绝）"
    })
    
    if decision is None:
        # 超时或默认值
        return {"status": "auto_rejected"}
    
    return {"status": decision}
```

## 最佳实践

### 1. 提供清晰的上下文

```python
# 好的做法
decision = interrupt({
    "message": "请审批以下数据处理结果",
    "data": result,
    "options": ["approve", "reject", "modify"],
    "deadline": "2024-01-01 12:00",
    "priority": "high"
})

# 不好的做法
decision = interrupt("审批？")
```

### 2. 验证人类输入

```python
def validation_node(state):
    """验证人类输入"""
    while True:
        user_input = interrupt({"message": "请输入有效的邮箱地址"})
        
        if is_valid_email(user_input):
            return {"email": user_input}
        else:
            # 输入无效，继续循环
            continue
```

### 3. 记录审计日志

```python
def auditable_interrupt(state):
    """可审计的中断"""
    interrupt_value = {
        "message": "请审批",
        "data": state["data"],
        "timestamp": datetime.now().isoformat(),
        "requester": state["user_id"]
    }
    
    decision = interrupt(interrupt_value)
    
    # 记录审计日志
    log_audit({
        "action": "approval",
        "decision": decision,
        "timestamp": datetime.now().isoformat(),
        "approver": get_current_user()
    })
    
    return {"decision": decision}
```

## 相关主题

- [[langgraph/modules/langgraph_human_in_loop/概述|人机交互概述]] - 了解人机交互的基本概念
- [[langgraph/modules/langgraph_human_in_loop/审批工作流|审批工作流]] - 审批模式的详细实现
- [[langgraph/modules/langgraph_checkpoint/概述|检查点]] - 理解状态持久化
- [[langgraph/modules/langgraph_state/概述|状态管理]] - 状态如何保存和恢复
