---
tags: [langgraph, core, command, state-update, routing, interrupt, resume]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# Command 与状态更新

`Command` 是 LangGraph 提供的高级控制 API，允许节点在一次返回中**同时完成状态更新、路由跳转和中断恢复**。它比简单的字典返回更强大，适用于需要精细控制图执行流程的场景。

## 核心概念

### Command 是什么

`Command` 是 `langgraph.types` 模块中的一个类，它封装了节点对图的**控制指令**：

```python
from langgraph.types import Command

# Command 可以同时携带三种指令
Command(
    update={"key": "value"},   # 状态更新
    goto="next_node",          # 路由跳转
    resume=some_value,         # 恢复中断的值
)
```

### 为什么需要 Command

| 场景 | 字典返回 | Command |
|------|----------|---------|
| 简单状态更新 | `return {"key": "value"}` | 不必要 |
| 状态更新 + 路由控制 | 需要条件边 | `Command(update=..., goto=...)` |
| 恢复中断执行 | 需要特殊处理 | `Command(resume=value)` |
| 动态路由到多个节点 | 条件边映射 | `Command(goto=...)` |

> **原则**：如果你的节点只需要返回状态更新，使用字典即可。只有当你需要**组合多种控制指令**时，才需要 `Command`。

## 一、Command 基本用法

### 1.1 导入与基本结构

```python
from langgraph.types import Command
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    """示例状态"""
    input_text: str
    result: str
    step: str
```

### 1.2 Command 的三种用法

```python
# 用法 1：仅状态更新（等价于 return {"result": "done"}）
def node_with_update(state: State):
    return Command(update={"result": "处理完成", "step": "done"})

# 用法 2：仅路由跳转（不更新状态）
def node_with_routing(state: State):
    return Command(goto="process_node")

# 用法 3：状态更新 + 路由跳转（最常用）
def node_with_both(state: State):
    return Command(
        update={"result": "处理完成"},
        goto="next_step"
    )
```

## 二、Command(update={...})：状态更新

### 2.1 基本状态更新

```python
from langgraph.types import Command

def process_node(state: State) -> Command:
    """
    使用 Command 更新状态
    效果等同于 return {"result": "处理完成", "step": "processed"}
    """
    result = state["input_text"].upper()
    return Command(
        update={
            "result": result,
            "step": "processed"
        }
    )
```

### 2.2 与字典返回的对比

```python
# 方式 A：字典返回（简单场景推荐）
def node_a(state: State) -> dict:
    return {"result": "done", "step": "step1"}

# 方式 B：Command 更新（需要额外控制时推荐）
def node_b(state: State) -> Command:
    return Command(update={"result": "done", "step": "step1"})

# 两者在状态更新方面完全等价
# 但 Command 可以额外指定 goto
```

### 2.3 更新嵌套状态

```python
from typing import TypedDict
from langgraph.types import Command

class NestedState(TypedDict):
    user: dict
    settings: dict

def update_nested_node(state: NestedState) -> Command:
    """
    更新嵌套状态字段
    注意：LangGraph 对状态更新使用浅合并
    """
    return Command(
        update={
            "user": {
                "name": "张三",
                "email": "zhangsan@example.com"
            },
            "settings": {
                "theme": "dark",
                "language": "zh-CN"
            }
        }
    )
```

## 三、Command(goto=...)：路由跳转

### 3.1 基本路由

`Command(goto=...)` 允许节点**直接指定下一个要执行的节点**，无需依赖条件边：

```python
from langgraph.types import Command

def router_node(state: State) -> Command:
    """
    根据输入内容决定下一步
    不需要 add_conditional_edges，直接用 Command 路由
    """
    text = state["input_text"]

    if "搜索" in text:
        return Command(goto="search_node")
    elif "计算" in text:
        return Command(goto="calculate_node")
    else:
        return Command(goto="default_node")

# 构建图
builder = StateGraph(State)
builder.add_node("router", router_node)
builder.add_node("search_node", search_node)
builder.add_node("calculate_node", calculate_node)
builder.add_node("default_node", default_node)

builder.add_edge(START, "router")
# 注意：不需要 add_conditional_edges
# Command(goto=...) 会自动路由到目标节点
builder.add_edge("search_node", END)
builder.add_edge("calculate_node", END)
builder.add_edge("default_node", END)

graph = builder.compile()
```

### 3.2 路由到多个节点（并行执行）

`goto` 可以接收列表，实现**动态并行执行**：

```python
def parallel_router(state: State) -> Command:
    """
    路由到多个节点并行执行
    """
    tasks = []

    if needs_search(state):
        tasks.append("search")
    if needs_calculation(state):
        tasks.append("calculate")
    if needs_summary(state):
        tasks.append("summarize")

    # 如果没有需要执行的任务，直接结束
    if not tasks:
        return Command(goto=END)

    # 并行执行所有任务
    return Command(goto=tasks)
```

### 3.3 Command 路由 vs 条件边

```python
# 方式 A：条件边路由
def route_function(state: State) -> str:
    if state["step"] == "search":
        return "search"
    return "default"

builder.add_conditional_edges(
    "router_node",
    route_function,
    {"search": "search_node", "default": "default_node"}
)

# 方式 B：Command 路由（更灵活）
def router_node(state: State) -> Command:
    if state["step"] == "search":
        return Command(
            update={"status": "searching"},  # 同时更新状态
            goto="search_node"               # 同时路由
        )
    return Command(goto="default_node")

# 不需要 add_conditional_edges
builder.add_edge(START, "router_node")
```

**对比总结**：

| 特性 | 条件边 | Command 路由 |
|------|--------|-------------|
| 路由逻辑位置 | 独立的路由函数 | 节点函数内部 |
| 同时更新状态 | 不可以 | 可以 |
| 可读性 | 路由映射清晰 | 逻辑集中在节点内 |
| 动态性 | 映射表固定 | 可以完全动态 |
| 并行路由 | 不支持 | 支持 `goto=[...]` |

## 四、Command(resume=value)：恢复中断

### 4.1 中断与恢复的基本概念

LangGraph 支持**中断（interrupt）**机制，允许图在执行到某个节点时暂停，等待外部输入后恢复。`Command(resume=...)` 用于在中断恢复时**传入恢复值**。

```python
from langgraph.types import interrupt, Command

class State(TypedDict):
    question: str
    human_feedback: str
    result: str

def process_node(state: State) -> dict:
    """处理节点"""
    return {"result": f"处理: {state['question']}"}

def human_review_node(state: State) -> Command:
    """
    人工审核节点：中断等待人工输入
    interrupt() 会暂停图的执行
    """
    # 暂停执行，等待人工输入
    feedback = interrupt("请审核以下内容并提供反馈")

    # 恢复执行后，feedback 就是传入的恢复值
    return {"human_feedback": feedback}

def finalize_node(state: State) -> dict:
    """最终处理节点"""
    return {"result": f"最终结果: {state['human_feedback']}"}
```

### 4.2 使用 Command(resume=...) 恢复执行

```python
from langgraph.checkpoint.memory import MemorySaver

# 编译图，使用 checkpointer
checkpointer = MemorySaver()
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_review"]  # 在人工审核节点前中断
)

config = {"configurable": {"thread_id": "review-001"}}

# 第一次执行：会在 human_review 节点前暂停
result = graph.invoke(
    {"question": "今天天气怎么样？", "human_feedback": "", "result": ""},
    config=config
)

# 查看当前状态
state = graph.get_state(config)
print(f"当前等待节点: {state.next}")  # ('human_review',)

# 使用 Command(resume=...) 恢复执行
# resume 的值会作为 interrupt() 的返回值
result = graph.invoke(
    Command(resume="内容没问题，可以发布"),
    config=config
)

print(f"最终结果: {result['human_feedback']}")
# 输出: 内容没问题，可以发布
```

### 4.3 resume 值的使用

`Command(resume=value)` 中的 `value` 可以是任意 Python 对象：

```python
# 恢复为字符串
graph.invoke(Command(resume="approved"), config=config)

# 恢复为字典
graph.invoke(Command(resume={"decision": "approved", "comment": "LGTM"}), config=config)

# 恢复为列表
graph.invoke(Command(resume=["option1", "option2"]), config=config)
```

### 4.4 多次中断与恢复

```python
def multi_review_node(state: State) -> Command:
    """
    多次中断节点：每次中断等待不同的审核
    """
    # 第一次中断：技术审核
    tech_feedback = interrupt("请进行技术审核")
    print(f"技术审核反馈: {tech_feedback}")

    # 第二次中断：业务审核
    biz_feedback = interrupt("请进行业务审核")
    print(f"业务审核反馈: {biz_feedback}")

    return {
        "result": f"技术: {tech_feedback}, 业务: {biz_feedback}"
    }

# 第一次恢复
graph.invoke(Command(resume="技术通过"), config=config)

# 第二次恢复
graph.invoke(Command(resume="业务通过"), config=config)
```

## 五、组合使用 Command

### 5.1 update + goto 组合

最常见的组合，在更新状态的同时控制路由：

```python
def classify_and_route(state: State) -> Command:
    """
    分类并路由：
    1. 更新分类结果到状态
    2. 根据分类路由到不同节点
    """
    text = state["input_text"]

    # 分类逻辑
    if len(text) < 10:
        category = "short"
        next_node = "quick_response"
    elif "?" in text:
        category = "question"
        next_node = "detailed_answer"
    else:
        category = "statement"
        next_node = "acknowledge"

    return Command(
        update={"step": category},
        goto=next_node
    )
```

### 5.2 update + goto + 条件逻辑

```python
from langchain_openai import ChatOpenAI

def smart_router(state: State) -> Command:
    """
    智能路由：使用 LLM 判断下一步
    """
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    # 让 LLM 决定路由
    response = llm.invoke(
        f"根据以下输入，判断应该走哪个流程（search/calculate/chat）：{state['input_text']}"
    )

    route = response.content.strip().lower()

    # 映射路由
    route_map = {
        "search": "search_node",
        "calculate": "calculate_node",
        "chat": "chat_node",
    }

    next_node = route_map.get(route, "chat_node")

    return Command(
        update={"step": f"routed_to_{route}"},
        goto=next_node
    )
```

### 5.3 在子图中使用 Command

```python
from langgraph.graph import StateGraph

# 子图
sub_builder = StateGraph(State)
sub_builder.add_node("sub_process", sub_process_node)
sub_builder.add_edge(START, "sub_process")
sub_builder.add_edge("sub_process", END)
sub_graph = sub_builder.compile()

# 父图
parent_builder = StateGraph(State)
parent_builder.add_node("sub_graph", sub_graph)
parent_builder.add_node("post_process", post_process_node)

parent_builder.add_edge(START, "sub_graph")

# 子图节点也可以使用 Command
def sub_graph_wrapper(state: State) -> Command:
    """子图包装节点"""
    result = sub_graph.invoke(state)
    return Command(
        update={"result": result["result"]},
        goto="post_process"
    )
```

## 六、Command 与节点返回方式对比

### 6.1 所有返回方式

```python
# 方式 1：返回字典（最常用）
def node_dict(state: State) -> dict:
    return {"result": "done"}

# 方式 2：返回 Command（需要路由控制时）
def node_command(state: State) -> Command:
    return Command(update={"result": "done"}, goto="next")

# 方式 3：返回 None（不更新状态）
def node_none(state: State) -> None:
    print("side effect only")
    return None

# 方式 4：异步返回
async def node_async(state: State) -> dict:
    return {"result": "async done"}
```

### 6.2 选择指南

```
需要更新状态吗？
├── 不需要 → return None
└── 需要
    ├── 只需要更新状态 → return dict（简单直观）
    └── 还需要控制路由 → return Command
        ├── 需要恢复中断 → Command(resume=value)
        └── 不需要恢复中断 → Command(update=..., goto=...)
```

### 6.3 完整对比示例

```python
"""
对比：字典返回 vs Command 实现相同功能
"""
from typing import TypedDict
from langgraph.types import Command
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    text: str
    category: str
    result: str

# ===== 方式 A：字典 + 条件边 =====

def classify_a(state: State) -> dict:
    """分类节点"""
    if len(state["text"]) < 10:
        return {"category": "short"}
    return {"category": "long"}

def route_a(state: State) -> str:
    """路由函数"""
    return state["category"]

def handle_short_a(state: State) -> dict:
    return {"result": "短文本处理"}

def handle_long_a(state: State) -> dict:
    return {"result": "长文本处理"}

# 方式 A 的图
builder_a = StateGraph(State)
builder_a.add_node("classify", classify_a)
builder_a.add_node("short", handle_short_a)
builder_a.add_node("long", handle_long_a)

builder_a.add_edge(START, "classify")
builder_a.add_conditional_edges("classify", route_a, {
    "short": "short",
    "long": "long"
})
builder_a.add_edge("short", END)
builder_a.add_edge("long", END)

# ===== 方式 B：Command（更简洁） =====

def classify_b(state: State) -> Command:
    """分类节点：同时更新状态和路由"""
    if len(state["text"]) < 10:
        return Command(
            update={"category": "short"},
            goto="short"
        )
    return Command(
        update={"category": "long"},
        goto="long"
    )

def handle_short_b(state: State) -> dict:
    return {"result": "短文本处理"}

def handle_long_b(state: State) -> dict:
    return {"result": "长文本处理"}

# 方式 B 的图（不需要条件边）
builder_b = StateGraph(State)
builder_b.add_node("classify", classify_b)
builder_b.add_node("short", handle_short_b)
builder_b.add_node("long", handle_long_b)

builder_b.add_edge(START, "classify")
# 不需要 add_conditional_edges！
builder_b.add_edge("short", END)
builder_b.add_edge("long", END)
```

## 七、实战示例：带人工审核的工作流

```python
"""
完整示例：内容审核工作流
使用 Command 实现中断、恢复和路由
"""
from typing import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver

class ReviewState(TypedDict):
    """审核工作流状态"""
    content: str
    ai_review: str
    human_decision: str
    final_result: str
    revision_count: int

def ai_review_node(state: ReviewState) -> dict:
    """AI 自动审核"""
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    response = llm.invoke(
        f"请审核以下内容是否合规，给出通过/拒绝/需修改的建议：\n{state['content']}"
    )
    return {"ai_review": response.content}

def human_review_node(state: ReviewState) -> Command:
    """
    人工审核节点
    使用 interrupt 暂停，等待人工决策
    """
    # 显示 AI 审核结果，等待人工决策
    decision = interrupt(
        f"AI 审核结果：{state['ai_review']}\n"
        f"请输入决策（approve/reject/revise）："
    )

    # 恢复后处理决策
    return Command(
        update={"human_decision": decision},
        goto="process_decision"
    )

def process_decision_node(state: ReviewState) -> Command:
    """
    处理人工决策
    根据决策路由到不同的后续节点
    """
    decision = state["human_decision"]

    if decision == "approve":
        return Command(
            update={"final_result": "内容已批准发布"},
            goto="publish"
        )
    elif decision == "reject":
        return Command(
            update={"final_result": "内容被拒绝"},
            goto="reject"
        )
    elif decision == "revise":
        return Command(
            update={"revision_count": state.get("revision_count", 0) + 1},
            goto="revise"
        )
    else:
        return Command(
            update={"final_result": f"未知决策: {decision}"},
            goto="reject"
        )

def publish_node(state: ReviewState) -> dict:
    """发布节点"""
    print(f"[发布] {state['final_result']}")
    return {}

def reject_node(state: ReviewState) -> dict:
    """拒绝节点"""
    print(f"[拒绝] {state['final_result']}")
    return {}

def revise_node(state: ReviewState) -> dict:
    """修改节点"""
    print(f"[修改] 第 {state['revision_count']} 次修改")
    return {}

# 构建图
builder = StateGraph(ReviewState)
builder.add_node("ai_review", ai_review_node)
builder.add_node("human_review", human_review_node)
builder.add_node("process_decision", process_decision_node)
builder.add_node("publish", publish_node)
builder.add_node("reject", reject_node)
builder.add_node("revise", revise_node)

builder.add_edge(START, "ai_review")
builder.add_edge("ai_review", "human_review")
builder.add_edge("publish", END)
builder.add_edge("reject", END)
# 修改后重新进入人工审核
builder.add_edge("revise", "human_review")

# 编译：在 human_review 前中断
checkpointer = MemorySaver()
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_review"]
)

# ===== 使用示例 =====

config = {"configurable": {"thread_id": "review-001"}}

# 步骤 1：提交内容，执行到人工审核前暂停
graph.invoke(
    {
        "content": "这是一篇关于 AI 技术的文章...",
        "ai_review": "",
        "human_decision": "",
        "final_result": "",
        "revision_count": 0,
    },
    config=config
)

# 步骤 2：查看状态，确认等待审核
state = graph.get_state(config)
print(f"等待节点: {state.next}")

# 步骤 3：人工审核通过
result = graph.invoke(
    Command(resume="approve"),
    config=config
)

print(f"最终结果: {result['final_result']}")
```

## 八、注意事项与最佳实践

### 8.1 Command 的返回类型

```python
# 正确：函数签名标注返回 Command
def my_node(state: State) -> Command:
    return Command(update={"key": "value"})

# 正确：混合返回（不同分支返回不同类型）
def mixed_node(state: State):
    if condition:
        return Command(goto="special")
    return {"result": "normal"}  # 普通字典也可以

# 注意：Command 和字典不能在同一返回中混合
# 错误示例：
# return Command(update={"a": 1}), {"b": 2}  # 不要这样做
```

### 8.2 goto 目标必须是有效节点

```python
# goto 的目标必须是图中已定义的节点名或 END
return Command(goto="existing_node")  # 正确
return Command(goto="nonexistent")     # 运行时错误
return Command(goto=END)               # 正确：结束图执行
```

### 8.3 update 遵循状态合并规则

```python
# Command 的 update 遵循与普通字典返回相同的合并规则
# 对于 Annotated 字段，使用 reducer 合并
# 对于普通字段，直接覆盖

from typing import Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # 使用 add_messages reducer
    count: int                                # 直接覆盖

def my_node(state: State) -> Command:
    return Command(
        update={
            "messages": [("assistant", "你好")],  # 追加到消息列表
            "count": 42                           # 覆盖 count 值
        }
    )
```

## 总结

| Command 用法 | 说明 | 典型场景 |
|-------------|------|----------|
| `Command(update={...})` | 更新状态 | 等价于字典返回 |
| `Command(goto="node")` | 路由到指定节点 | 替代条件边 |
| `Command(goto=[...][[)` | 路由到多个节点 | 动态并行执行 |
| `Command(resume=value)` | 恢复中断执行 | 人工审核工作流 |
| `Command(update, goto)` | 更新 + 路由 | 分类后路由 |

## 相关文档

- [[langgraph/modules/langgraph_core/StateGraph|StateGraph 详解]]
- [[langgraph/modules/langgraph_core/节点|节点详解]]
- [[langgraph/modules/langgraph_core/边|边详解]]
- [[langgraph/modules/langgraph_core/编译与执行|编译与执行]]
