---
tags: [interview, langgraph, state, reducer, 面试]
type: guide
created: 2026-10-18
updated: 2026-10-18
---

# 状态与 Reducer

> **适用场景**：LangGraph 面试的「分水岭」。会写 `StateGraph` 的人很多，能说清 reducer 为什么存在的人很少。
> **一句话主线**：State 是声明，Channel 是容器，reducer 是合并规则。

---

## Q1. State 怎么定义？TypedDict、Pydantic、dataclass 怎么选？

**答：**

三种都支持，生产上最常用 **TypedDict**。

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
    user_id: str
    retry_count: int
```

| 方式 | 优点 | 代价 | 适用 |
|------|------|------|------|
| **TypedDict** | 轻量、无运行时开销、官方示例默认 | 不做运行时校验，字段写错只能靠类型检查 | 绝大多数场景 |
| **Pydantic BaseModel** | 运行时校验、默认值、类型转换 | 每次状态更新都要校验，开销高；与部分 reducer 配合需注意 | 输入边界、强校验需求 |
| **dataclass** | 标准库、可加方法 | 生态示例少 | 偏好 OOP 的团队 |

**关键点**：State 只是**声明**。运行时真正存值的是每个字段对应的 Channel；类型注解不会自动帮你校验。

**追问：TypedDict 不校验，字段写错了怎么办？**
→ 两道防线：① 开发期用 mypy/pyright 做静态检查；② 在图的入口节点做一次显式校验（Pydantic 解析输入）。**不要指望运行时替你把关。**

---

## Q2. reducer 是什么？不配 reducer 会怎样？

**答：**

**reducer 是一个二元函数：`(旧值, 新值) -> 合并后的值`，用来定义「同一个状态字段被多次写入时怎么合并」。**

```python
from typing import Annotated

def add(a: int, b: int) -> int:
    return a + b

class State(TypedDict):
    total: Annotated[int, add]     # 每次写入做累加
    note: str                       # 不配 reducer → 默认覆盖
```

不配 reducer 时，该字段使用 **LastValue** 语义：

| 场景 | 结果 |
|------|------|
| 串行执行，A 写 `x=1`，B 写 `x=2` | 最终 `x=2`，符合直觉 |
| **同一个 superstep 内，两个并行节点都写 `x`** | **抛 `InvalidUpdateError`**：LastValue 每步只接受一个值 |
| 循环里想累积列表 | 每次都被覆盖，只剩最后一次 |

⚠️ **这是最容易答错的一题**。很多人以为并行写同一字段会「静默覆盖」，实际 LangGraph 会**直接报错**，提示你用 `Annotated` 处理多值。真正静默出错的是「原地修改 state」那种写法。

**追问：为什么默认设计成「报错」而不是「后写赢」？**
→ 因为并行写入的顺序不确定，静默取最后一个会让结果**不可复现**。报错是逼你显式声明合并语义。

---

## Q3. `Annotated` 在这里到底起什么作用？

**答：**

`Annotated[T, reducer]` 是 Python 的类型注解语法：**第一个参数是类型，后面的元数据是附加信息。**

LangGraph 读取这段元数据，把第二个位置的可调用对象识别为该字段的 reducer：

```python
messages: Annotated[list, add_messages]
#          ↑类型    ↑reducer（LangGraph 解析这个位置）
```

它**不改变运行时类型**，只是给框架的声明式配置。等价于「给这个字段挂一个合并策略」。

**追问：可以挂多个元数据吗？**
→ 语法上可以，但 LangGraph 只认它约定的那个 reducer 位置。不要往里塞业务元数据，容易和框架解析冲突。

**追问：能不能在运行时动态换 reducer？**
→ 不能。reducer 在 `compile()` 解析 schema 时就绑定到 Channel 上了，属于图的静态结构。要换只能重新构图。

---

## Q4. `add_messages` 具体做了什么？它和普通 `list` 有什么区别？

**答：**

`add_messages` 是消息列表专用的 reducer，做了四件事：

| 行为 | 说明 |
|------|------|
| **类型归一** | 传入 `("user", "hi")` 元组或 dict，会转成 `HumanMessage` 等消息对象 |
| **按 id 合并** | 新消息 id 已存在 → **替换**该条；不存在 → **追加** |
| **支持删除** | 传入 `RemoveMessage(id=...)` → 从列表移除对应消息 |
| **保持顺序** | 按合并后的顺序返回，不主动排序 |

普通 `list` 没有 reducer，是 LastValue：整个列表被替换。**这就是为什么多轮对话必须给 `messages` 配 `add_messages`**，否则第二轮会把第一轮历史冲掉。

**追问：两个人同时往 messages 里写，会乱序吗？**
→ 同一 superstep 的多个写入会按 reducer 依次合并，**顺序由框架调度决定，不保证等于业务发生顺序**。如果顺序重要，给消息加时间戳/序号，在消费端排序，不要依赖合并顺序。

**追问：`add_messages` 会去重吗？**
→ 只按 **id** 去重（同 id 视为同一条消息的更新）。内容相同但 id 不同的两条消息会都保留。

---

## Q5. 并行节点写同一个字段，reducer 要注意什么？

**答：**

三条硬要求：

1. **必须满足交换律**：`f(a, b) == f(b, a)`。并行顺序不定，不满足就会得到不同结果。
2. **必须满足结合律**：`f(f(a, b), c) == f(a, f(b, c))`。多路合并时分组方式不定。
3. **不要依赖外部可变状态**：reducer 里读全局变量、当前时间、数据库，会让重放结果不一致。

安全的 reducer：加法、集合并集、列表拼接、`add_messages`。
危险的 reducer：`lambda a, b: b`（取后者）、`lambda a, b: a[-1]`（依赖顺序）、带副作用的写入。

```python
# ✅ 可交换、可结合
def merge_sources(a: list, b: list) -> list:
    return list(dict.fromkeys(a + b))   # 保序去重

# ❌ 依赖顺序
def last_wins(a, b):
    return b
```

**追问：如果业务真的要求「最后一个赢」怎么办？**
→ 不要让 reducer 承担这个语义。给每个写入带**单调递增的版本号或时间戳**，在 reducer 里按版本比较，而不是按到达顺序。

---

## Q6. 怎么删除或替换消息历史？

**答：**

三种手段，按场景选：

**① 删除单条：`RemoveMessage`**

```python
from langchain_core.messages import RemoveMessage

def drop_tool_noise(state: State) -> dict:
    return {"messages": [RemoveMessage(id=m.id) for m in state["messages"]
                         if m.type == "tool"]}
```

**② 清空全部：`REMOVE_ALL_MESSAGES`**

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES
return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

**③ 压缩而非删除：摘要**

把旧消息交给模型总结成一条 `SystemMessage`，再用 `RemoveMessage` 删掉原文。这也是 `SummarizationMiddleware` 的核心思路。

⚠️ **不要直接 `return {"messages": []}` 想清空**。`add_messages` 会把空列表当成「没有新增」，历史仍然在。必须用 `RemoveMessage`。

**追问：删除后 checkpoint 里还在吗？**
→ 在。Checkpointer 存的是**每一步的快照**，删除只影响当前状态和后续快照。历史快照仍可通过 `get_state_history` 查到——这是时间旅行的基础，也是「删除不等于物理擦除」的合规风险点。

---

## Q7. State 应该设计得多大？大对象放哪？

**答：**

**原则：State 只放「需要在节点间流转的小数据」，大对象放外部，State 存引用。**

原因：**每个 superstep 结束都会把 State 序列化进 checkpoint。**State 越大：

| 代价 | 表现 |
|------|------|
| 写放大 | 每步都写全量快照，PG 的写入量和存储量暴涨 |
| 延迟 | 序列化/反序列化占用执行时间 |
| 传输 | 分布式部署时跨节点传状态 |
| 恢复变慢 | 反序列化一个大 State |

推荐做法：

| 数据 | 放哪 |
|------|------|
| 消息历史 | State，但要裁剪/摘要 |
| 用户输入、路由标记、计数 | State |
| 原始文档、图片、大 JSON、检索全量结果 | 对象存储 / 业务表 / LangGraph Store，State 只存 id |
| 中间推理草稿 | 尽量不留，或写日志 |

**追问：消息历史越来越长怎么办？**
→ 三层：① 滑动窗口只留最近 N 轮；② 摘要压缩旧轮次；③ 长期事实抽到 Store（按 namespace 存），不放在 messages 里。生产上通常 ①+② 组合。

---

## Q8. 为什么要区分 input schema 和 output schema？

**答：**

因为**「调用方能写什么」和「调用方能看什么」不是一回事**。

```python
class InputState(TypedDict):
    question: str

class OutputState(TypedDict):
    answer: str

class InternalState(InputState, OutputState):
    retrieved_docs: list
    retry_count: int
    debug_trace: list

graph = StateGraph(InternalState, input_schema=InputState, output_schema=OutputState)
```

| 作用 | 说明 |
|------|------|
| **输入收敛** | 调用方只能传 `question`，不能注入 `retry_count` 之类的内部字段 |
| **输出裁剪** | 只返回 `answer`，不把检索原文、调试信息泄露给前端 |
| **契约稳定** | 内部字段改名不影响外部调用方 |

⚠️ **注意**：output schema 只影响**返回值**，不影响 checkpoint 里存的内容。内部状态仍然完整持久化。

**追问：不区分会有什么问题？**
→ 两个真实风险：① 调用方能覆盖内部字段（比如把 `is_admin` 传成 true），形成越权；② 返回体里带出内部 prompt、检索内容，造成信息泄露。

---

## 本篇自测

- [ ] TypedDict / Pydantic / dataclass 的取舍，为什么默认 TypedDict？
- [ ] 写出一个合法 reducer 的签名，并说出三个必须满足的性质
- [ ] 不配 reducer 时，串行和并行分别会发生什么？（并行是报错，不是覆盖）
- [ ] `add_messages` 的四件事，以及它和普通 list 的区别
- [ ] 三种删除消息的方式，为什么 `[]` 清不空
- [ ] State 为什么不能放大型对象？大对象应该放哪？
- [ ] input/output schema 分别防住什么风险？

---

## 导航

[[langgraph/interview/README|LangGraph 面试题库首页]] · 上一篇：[[langgraph/interview/01-核心概念与运行模型]] · 下一篇：[[langgraph/interview/03-控制流与动态编排]] · 详解：[[langgraph/modules/langgraph_state/概述]] · [[langgraph/modules/langgraph_state/Reducer]]
