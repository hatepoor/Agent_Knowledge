---
tags: [typescript, langgraph, langchain, 双栈对照, 迁移]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# Python 与 TypeScript 双栈对照

> **什么时候该读**：你已经会写 Python 版 LangChain/LangGraph，现在要在 TypeScript 里落地同一个需求；或者你同时维护 Python 和 TS 两套服务，需要决定"这个功能放哪边"、怎么让两边行为一致。
> **阅读门槛**：读完 [[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]] 和 [[typescript/03-生态连接-LangChain与LangGraph/5-LangGraph-js持久化与人工介入]]，并且熟悉 Python 侧的 LangChain/LangGraph（本库 [[langgraph/overview/LangGraph概述]]、[[langchain/overview/LangChain概述]]）。另外建议读过 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]]——本页会对比两套类型系统的根本差异。

## 学习目标

读完本篇，你应当能够：

1. 查表完成 Python LangChain/LangGraph ↔ TypeScript LangChain.js/LangGraph.js 的 API 映射
2. 说出命名风格的转换规律，并识别出"哪些名字不转换"（LangChain 自己的 API）
3. ⭐ 讲清 Pydantic 与 Zod 的根本差异，以及它对"类型错误何时被发现"的影响
4. ⭐ 说清生态差异的实话：哪边特性更全、哪边更有优势，以及"Python 先出特性、TS 跟进"这个规律
5. 在 HTTP、MCP、纯 TS 三种双栈协作模式里做出选择
6. 用迁移决策树判断一个 Python 服务是否值得迁到 TS
7. 列出 5 个"照搬 Python 习惯会踩的坑"并解释原因
8. 准备好"TS 与 Python 双栈如何协作"的面试答法

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]] | TS 侧的状态、节点、边、reducer 概念 |
| [[typescript/03-生态连接-LangChain与LangGraph/5-LangGraph-js持久化与人工介入]] | TS 侧的 checkpointer、HITL、流式 |
| [[langgraph/overview/LangGraph概述]] | Python 侧的概念基准 |
| [[langchain/overview/LangChain概述]] | Python 侧 LangChain 的定位 |
| [[langchain/modules/langchain-mcp/概述]] | MCP 在双栈协作中的作用 |
| [[python/basics/Python_基础概览]] | Python 语言特性基础 |

---

## 一、⭐ 完整 API 对照表

> **版本基线**：Python 侧 `langchain` 1.4.x / `langgraph` 1.2.x；TS 侧 `langchain` 1.5.x / `@langchain/langgraph` 1.4.x~1.5.x。两边都是 **v1 线**。

### 1.1 核心 API 映射

| 概念 | Python | TypeScript |
|------|--------|-----------|
| 模型初始化 | `init_chat_model` / `ChatOpenAI` | `initChatModel` / `ChatOpenAI` |
| 工具定义 | `@tool` 装饰器 + Pydantic 参数模型 | `tool()` 调用式 + Zod schema |
| Agent | `create_agent` | `createAgent` |
| 中间件 | `AgentMiddleware` 基类子类 / 钩子装饰器 | `createMiddleware({...})` 工厂函数 |
| 预置 HITL 中间件 | `HumanInTheLoopMiddleware` | `humanInTheLoopMiddleware` |
| 状态定义 | `Annotation.Root` / `TypedDict` / Pydantic | `StateSchema`（Zod） |
| Reducer | `Annotated[list, add_messages]` | `new ReducedValue(...)` / `MessagesValue` |
| 消息状态 | `MessagesState` / `MessagesAnnotation` | `MessagesValue` |
| 图 | `StateGraph` | `StateGraph` |
| 入口 / 终点 | `START` / `END` | `START` / `END` |
| 节点 / 边 | `add_node` / `add_edge` / `add_conditional_edges` | `addNode` / `addEdge` / `addConditionalEdges` |
| 编译 | `graph.compile(...)` | `graph.compile(...)` |
| 运行时上下文 | `StateGraph(State, context_schema=Ctx)` + `Runtime.context` | `new StateGraph(State, ContextSchema)` + `config.context` |
| 检查点 | `InMemorySaver` / `SqliteSaver` / `PostgresSaver` | `MemorySaver` / `SqliteSaver` / `PostgresSaver` |
| 人工介入 | `interrupt()`（动态）+ `interrupt_before`（静态） | `interrupt()`（动态）+ `interruptBefore`（静态） |
| 恢复 | `Command(resume=...)` | `new Command({ resume: ... })` |
| 跳转 | `Command(goto=...)` | `new Command({ goto: ... })` |
| 动态扇出 | `Send(node, arg)` | `new Send(node, arg)` |
| 流式 | `graph.stream(...)` / `astream(...)` | `graph.stream(...)` |
| 状态查询 | `graph.get_state(config)` | `await graph.getState(config)` |
| 递归上限 | `config={"recursion_limit": 25}` | `config={ recursionLimit: 25 }` |

> **导入路径速查**：
> - Python：`from langchain.agents import create_agent`、`from langgraph.graph import StateGraph, START, END`、`from langgraph.types import interrupt, Command, Send`
> - TS：`import { createAgent } from "langchain"`、`import { StateGraph, START, END } from "@langchain/langgraph"`

### 1.2 同一段逻辑的两种写法

**Python 版**：

```python
from typing import Annotated, TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    step: str

def node_a(state: State) -> dict:
    return {"step": "A"}

graph = StateGraph(State)
graph.add_node("a", node_a)
graph.add_edge(START, "a")
graph.add_edge("a", END)
app = graph.compile()
```

**TS 版**：

```ts
import { StateGraph, StateSchema, MessagesValue, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  messages: MessagesValue,
  step: z.string(),
});

const graph = new StateGraph(State)
  .addNode("a", () => ({ step: "A" }))
  .addEdge(START, "a")
  .addEdge("a", END)
  .compile();
```

⭐ **一眼能看出的差别**：

| 维度 | Python | TypeScript |
|------|--------|-----------|
| 状态定义 | 独立的 class / TypedDict 声明 | `new StateSchema({...})` 一个对象 |
| 节点声明 | 独立函数 + `add_node` 引用 | 箭头函数内联在 `addNode` 里 |
| 构建方式 | 逐行调用（可读性靠缩进） | 链式调用 |
| 类型来源 | 显式注解 | **自动推导**（不用写返回类型） |

---

## 二、⭐ 命名风格差异

### 2.1 转换规律

**基本规律：Python 的 `snake_case` → TS 的 `camelCase`。**

| Python | TypeScript |
|--------|-----------|
| `add_node` | `addNode` |
| `add_edge` | `addEdge` |
| `add_conditional_edges` | `addConditionalEdges` |
| `get_state` | `getState` |
| `update_state` | `updateState` |
| `init_chat_model` | `initChatModel` |
| `create_agent` | `createAgent` |
| `system_prompt` | `systemPrompt` |
| `allowed_decisions` | `allowedDecisions` |
| `interrupt_before` | `interruptBefore` |
| `recursion_limit` | `recursionLimit` |
| `thread_id` | `thread_id`（⚠️ 见下） |

⚠️ **`thread_id` 不转换**——因为它是 LangChain 内部约定的键名，两边都写成 `thread_id`。你也可以在 TS 里写 `threadId`，但框架读的是 `thread_id`，**改了就不生效**。

### 2.2 ⭐ 三个"不转换"的地方

这是最容易踩坑的部分——**不是所有 snake_case 都会变 camelCase**：

| 名字 | 为什么不转 | 说明 |
|------|-----------|------|
| `thread_id` | 协议约定的键名 | 放在 `config.configurable.thread_id` |
| `stopped` / `__interrupt__` | 框架内部约定的字段名 | 中断信息在结果的 `__interrupt__` 字段 |
| `actionRequests` / `reviewConfigs` | **LangChain 自己的 API 就这么定** | HITL 中间件的载荷字段是 camelCase |

⭐ **规律总结**：

> **LangChain / LangGraph 框架自己定义的 API 名，一律保持原样**（本来就是 camelCase 或带下划线的协议字段）；**只有"用户要传的参数名"才做风格转换**。

判断方法：看这个参数是**框架的 API 名**还是**你的业务参数**。`addNode` 是框架 API（camelCase）；`systemPrompt` 是你的参数（从 Python 的 `system_prompt` 转过来）；`thread_id` 是协议键（不动）。

### 2.3 一个明显的"半 snake_case"

`tool()` 的参数里有个有意思的：

```ts
// TypeScript
tool(fn, { name: "get_weather", description: "查询天气", schema: WeatherSchema });
```

```python
# Python
@tool
def get_weather(city: str) -> str:
    """查询天气"""
```

⭐ **Python 用函数名 + docstring 当工具描述，TS 用显式的 `name` / `description` / `schema` 三个字段**。这不只是风格差异——**TS 侧工具的入参校验发生在运行时（Zod）**，而 Python 侧靠 Pydantic 从注解生成 schema。

### 2.4 异步写法

| | Python | TypeScript |
|---|--------|-----------|
| 定义异步函数 | `async def f():` | `async function f() {}` / `async () => {}` |
| 等待 | `await f()` | `await f()` |
| 异步迭代 | `async for x in f():` | `for await (const x of f())` |
| 并发 | `asyncio.gather(...)` | `Promise.all([...])` |

⭐ **两边都有 `await`，但 TS 的 `Promise.all` 和 Python 的 `asyncio.gather` 有一个关键差别**：

```ts
// TypeScript：任一失败 → 整个 Promise.all 立刻 reject
const [a, b] = await Promise.all([callA(), callB()]);
```

```python
# Python：默认 gather 也会因一个异常而让整个 await 抛出
a, b = await asyncio.gather(callA(), callB())
```

⚠️ 但**错误语义不同**：TS 里 `Promise.all` reject 后，其他 promise 仍在后台跑（结果被丢弃）；Python 的 `asyncio.gather` 默认会**取消**其余任务。需要"全部跑完再收集错误"时，TS 用 `Promise.allSettled`，Python 用 `asyncio.gather(..., return_exceptions=True)`。详见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]]。

---

## 三、⭐ 类型系统的根本差异

### 3.1 Pydantic vs Zod

这是双栈协作中**最值得深聊**的技术差异。

| 维度 | Python + Pydantic | TypeScript + Zod |
|------|------------------|-----------------|
| 运行时校验 | ✅ Pydantic 核心能力 | ✅ Zod 核心能力 |
| 静态类型 | ⚠️ **靠 Mypy / Pyright 单独推导** | ✅ **TypeScript 编译器直接推导** |
| 两者关系 | **两条独立的路**，需要手动对齐 | **同一套 schema** |
| 写错字段名 | 运行到那行才 `KeyError` / `ValidationError` | **`tsc` 阶段就报错** |
| 序列化 | Pydantic 内置 `.model_dump()` | Zod 无内置序列化 |
| JSON Schema | `.model_json_schema()` | `z.toJSONSchema()` |
| 生态位置 | 数据模型库（也用于 FastAPI 等） | 校验库（也用于表单、配置等） |

### 3.2 ⭐ 关键差异：错误的发现时机

```ts
import { StateSchema } from "@langchain/langgraph";
import { z } from "zod";

// TS + Zod：写错字段名，tsc 直接报错
const State = new StateSchema({
  step: z.string(),
  count: z.number().default(0),
});

const bad: typeof State.Update = { stepp: "typo" };
//                    ^^^^^^ TS 编译错误：'stepp' 不在 Update 的属性里
```

```python
# Python + Pydantic：类型注解在运行时不生效
class State(TypedDict):
    step: str
    count: int

def node(state: State) -> dict:
    return {"stepp": "typo"}   # 运行到这里才炸 KeyError
```

⭐ **这就是"一套 schema 同时给运行时校验和静态类型"的含金量**：

- **TS 侧**：`StateSchema` 的 Zod 定义 → 推导出 `State` / `Update` 类型 → 约束节点函数签名。写错字段名、类型不匹配，**`tsc` 就拦住你**
- **Python 侧**：Pydantic 负责运行时校验，Mypy/Pyright 负责静态检查，**两者要手工保持一致**。Pydantic v2 虽然有 `TypeAdapter`、插件等桥接手段，但终究是两条路

⭐ **对 LangGraph 场景的意义**：

Python 的 `StateGraph` 节点返回 `dict`——**返回值不受任何静态检查约束**，写错 key 要等运行时。而 TS 的 `StateGraph` 节点返回 `Update` 类型，**返回值受编译期检查**。对于分支多、状态字段多的图，这个差异会实打实地减少线上事故。

### 3.3 泛型能力的差异

| | Python | TypeScript |
|---|--------|-----------|
| 泛型语法 | `TypeVar` + `Generic[T]`（或 3.12+ 的 `class Box[T]`） | `<T>` 放在名字后面 |
| 类型编程 | 基本只能"参数化" | **条件类型、映射类型、模板字面量、`infer`** |
| 推导时机 | 调用时 | 调用时 |

⭐ **TS 能写出"根据状态定义自动推导节点类型"这种能力，Python 做不到**——Python 没有 `typeof` 的值级推导，也没有条件类型。`StateSchema` 的 `State` / `Update` 推导，正是靠 TS 的这套能力实现的。详见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]]。

---

## 四、⭐ 生态差异的实话

这一节要说得直白，不粉饰。

### 4.1 Python 先出特性，TS 跟进

⭐ **这是 LangChain 生态的一条稳定规律**：

> **新特性通常先在 Python 版落地，然后跟进到 TypeScript 版。**

原因不神秘：Python 是 LangChain 的"原生语言"，团队内部先用它做实验、验证 API 设计，稳定后再移植到 TS。

**具体例子**（本库写作时的事实）：

| 特性 | Python 版 | TypeScript 版 |
|------|----------|--------------|
| `create_agent` 统一入口 | ✅ 更早 | ✅ 已有 |
| 中间件体系 | ✅ 类 + 钩子装饰器 | ✅ `createMiddleware` 工厂 |
| `StateSchema`（Zod 版状态定义） | ⚠️ Python 用 Pydantic 路线 | ✅ TS 是 Zod 路线 |
| `content_blocks` 标准内容块 | ✅ | ✅ |
| `tools` 流式模式 | ❌ 没有 | ✅ **JS 独有** |
| 某些新的 checkpointer 后端 | ✅ 先有 | ⚠️ 后跟 |

⭐ **注意最后一行的方向是反的**——不是所有特性都是 Python 先行。`tools` 流式模式就是 **JS 独有**的（Python v1 的 `StreamMode` 里没有 `tools`）。

### 4.2 checkpointer 后端：数量上的实话

| | Python | TypeScript |
|---|--------|-----------|
| 内存版 | `InMemorySaver` | `MemorySaver` |
| SQLite | `SqliteSaver` | `SqliteSaver` |
| PostgreSQL | `PostgresSaver` | `PostgresSaver` |
| MongoDB | `MongoDBSaver` | `MongoDBSaver` |
| Redis | `RedisSaver` | `RedisSaver` / `ShallowRedisSaver` |
| 独立包 | `langgraph-checkpoint` / `-sqlite` / `-postgres` / `-mongodb` / `-redis` | `@langchain/langgraph-checkpoint` / `-sqlite` / `-postgres` / `-mongodb` / `-redis` |

⭐ **官方后端两边基本对齐**（v1 之后两边都把 checkpointer 拆成了独立包，这是个明显的有意对齐）。

⚠️ **真正的差距在社区第三方后端**：很多社区维护的 LangGraph checkpointer **只提供了 Python 实现**（JS 端的贡献者少、测试基础设施也难搭）。如果你要用某个小众后端（比如某个云厂商的托管数据库），**大概率只有 Python 版**。

### 4.3 TS 版的真正优势

抛开"新特性滞后"，TS 版有几个**结构性**优势：

| 优势 | 说明 |
|------|------|
| ⭐ **与前端同构** | 同一份 Zod schema，前端表单和后端校验可以共用；`useStream` 让浏览器直接消费图的流 |
| ⭐ **单一运行时** | Node.js 一个运行时跑前后端 + Agent，不用维护 Python 和 Node 两套部署链路 |
| ⭐ **边缘运行时友好** | Cloudflare Workers / Vercel Edge / Deno 这些边缘平台原生跑 JS/TS，**Python 通常跑不了** |
| ⭐ **类型安全是编译期保证** | 见第三节——这是 Python 侧目前给不了的 |
| ⭐ **前端生态可以直接用** | 状态管理、动画、组件库，TS 侧现成 |

⭐ **一句话总结**：

> **Python 版的优势是"生态更全、特性更早"；TS 版的优势是"同构、单一运行时、边缘友好、类型安全是编译期保证"。**
>
> **选型不是"哪个更好"，而是"你的约束是什么"。**

---

## 五、⭐ 双栈协作的三种模式

### 5.1 模式一：HTTP / REST

```
┌──────────────┐   HTTP    ┌────────────────────┐   模型 API   ┌────────┐
│  TS 前端/网关 │ ────────► │  Python 编排服务    │ ──────────► │  LLM   │
│              │ ◄──────── │  (LangGraph 图)    │ ◄────────── │        │
└──────────────┘   JSON   └────────────────────┘             └────────┘
```

⭐ **这是最常见的分工**：

- **Python 侧**：跑 LangGraph 图、做检索、跑模型——生态最全的地方
- **TS 侧**：面向前端的 BFF（Backend for Frontend）、鉴权、聚合、WebSocket 推送

**为什么这么分**：把"重逻辑、需要丰富 Python 生态的部分"放 Python，把"轻逻辑、要贴近前端的部分"放 TS。

**TS 侧调用**：

```ts
const res = await fetch("http://python-service:8000/graph", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ question: "北京天气？" }),
});
const { answer } = await res.json();
```

⚠️ **要约定好的三件事**：
1. **错误语义**——Python 侧 500 了，TS 侧是透传 500 还是包装成业务错误码
2. **超时与重试**——图执行可能几十秒，HTTP 超时必须配好；重试要考虑**幂等**
3. **流式协议**——如果图要流式输出，用 SSE（Server-Sent Events）而不是普通 JSON

### 5.2 模式二：MCP（Model Context Protocol）

```
┌────────────────────┐         MCP 协议         ┌──────────────────┐
│  TS 编排服务        │ ◄──────────────────────► │  Python MCP Server│
│  (LangGraph.js)    │        tools / resources │  (你的数据/工具)  │
└────────────────────┘                           └──────────────────┘
```

⭐ **MCP 是"把 Python 里的能力暴露成工具，让 TS 的 Agent 调用"的标准化做法**。

**Python 侧起 MCP Server**，把内部能力（数据库查询、内部 API、算法）包成 MCP tool；**TS 侧用 MCP Client 接入**，这些工具就和本地工具一样用。

**TS 侧接 MCP**：

```ts
import { createAgent } from "langchain";
import { MultiServerMCPClient } from "@langchain/mcp-adapters";

// 接入 Python 侧起的 MCP Server
const client = new MultiServerMCPClient({
  servers: {
    "python-tools": {
      transport: "http",                        // HTTP 传输
      url: "http://python-service:8000/mcp",   // MCP Server 地址
    },
  },
});

const tools = await client.getTools();   // 拿到远程工具（按服务器分组的用 listToolsets()）

const model = "openai:gpt-4o-mini";       // 也可以是 new ChatOpenAI({...})
const agent = createAgent({ model, tools });
```

> **版本说明**：MCP 适配器在 `@langchain/mcp-adapters` v2.x。v2 里 `MultiServerMCPClient` 是 `MCPAdapter` 的别名；旧版的 `getTools()` 仍可用，新代码推荐 `listToolsets()`（返回按服务器名分组的工具）。传输取值是 `"stdio"` / `"http"` / `"sse"`——**不是** `"streamable_http"`（那是 Python 侧的写法）。

⭐ **MCP 模式的价值**：

- **能力与编排解耦**——Python 侧专注做"业务能力"，TS 侧专注做"流程编排"
- **跨语言、跨进程**——不需要两边都是 Python
- **标准化**——未来换语言、换框架，工具接口不变

⚠️ **要注意的**：MCP 引入了网络往返延迟，**工具调用频率高时延迟会累积**。MCP 的详细机制见 [[langchain/modules/langchain-mcp/概述]] 和 [[langchain/modules/langchain-mcp/MCP服务器开发]]。

### 5.3 模式三：纯 TS 单栈

```
┌────────────────────┐   模型 API   ┌────────┐
│  TS 应用(前后端一体) │ ──────────► │  LLM   │
│  LangChain.js      │ ◄────────── │        │
│  + LangGraph.js    │             └────────┘
└────────────────────┘
```

⭐ **完全不涉及 Python**：模型 API 是 HTTP 的，任何语言都能调。

**什么时候选这个**：

- 团队**只会 TS**（或不想维护两套语言的技术栈）
- 需要**边缘部署**（Cloudflare Workers、Vercel Edge）
- 需求是**标准 Agent 循环**（工具调用、多轮对话），用不到 Python 独有的生态
- 想要**前后端同构**（同一份 schema、同一份类型）

### 5.4 三种模式怎么选

| 模式 | 选它当 | 别选它当 |
|------|--------|---------|
| HTTP / REST | 两边各有擅长；Python 侧要用重生态 | 只需要一个轻量 Agent——那会多一跳网络 |
| MCP | Python 侧有**成套的业务能力**要暴露给 TS | 只是传一两个简单参数——直接 HTTP 更省事 |
| 纯 TS | 团队只有 TS；要边缘部署；需求标准 | 强依赖 Python 独有的库（科学计算、特定的 RAG 组件等） |

---

## 六、⭐ 迁移决策树

### 6.1 决策树

```
这个 Python 服务要不要迁到 TypeScript？
│
├─ 它在调用 Python 独有的生态吗？
│   （科学计算库、特定的 PDF/文档处理、专有的检索组件）
│   ├─ 是 → ❌ 不迁。迁过去要么找不到替代，要么自己重写，成本远大于收益
│   └─ 否 ↓
│
├─ 它需要部署到边缘运行时吗？
│   （Cloudflare Workers / Vercel Edge / Deno Deploy）
│   ├─ 是 → ✅ 强烈建议迁。Python 在这些平台基本跑不了
│   └─ 否 ↓
│
├─ 前后端需要共享类型 / schema 吗？
│   ├─ 是 → ✅ 建议迁。Zod 一套 schema 两端共用，收益很直接
│   └─ 否 ↓
│
├─ 团队的主要技能栈是 TS 吗？或者维护两套语言的成本很高？
│   ├─ 是 → ✅ 建议迁。技术栈收敛的收益会随时间放大
│   └─ 否 ↓
│
├─ 它的核心是标准 Agent 循环（工具调用 + 多轮对话）吗？
│   ├─ 是 → ✅ 可以迁。LangChain.js v1 完全够用
│   └─ 否 → ⚠️ 再想想。复杂的图编排迁过去要重写的量不小
│
└─ 结论：先别迁。
   把它当成"通过 HTTP/MCP 调用的能力服务"，
   等 TS 侧真的需要掌控编排时再迁。
```

### 6.2 ⭐ 反向的决策树：什么时候**不该**迁

同样重要：

```
❌ 不该迁的信号：
│
├─ Python 侧用了大量 Python 独有的库
├─ 迁移的主要动机是"Python 慢"——Agent 应用的瓶颈 90% 在模型 API，不在编排代码
├─ 迁移的主要动机是"TypeScript 更酷"——这不是工程理由
├─ 服务已经稳定运行，没有新需求催着改
└─ 迁移会连带改掉前端、部署、监控、CI——那是一次大重构，不是一次迁移
```

⭐ **最容易被忽略的一条**：

> **"TypeScript 比 Python 快"不是迁移 Agent 服务的有效论据。** Agent 应用的时间几乎全花在**等待模型 API 返回**上。编排代码（状态管理、reducer、路由）占总耗时的比例通常在 1% 以下。**用"快"当迁移理由，多半是在给自己找麻烦。**

真正有效的迁移理由只有三类：**边缘运行时**、**技术栈收敛**、**前后端同构**。

### 6.3 回到选型

选型的完整讨论见 [[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]]。这里只强调一条：

> **双栈不是妥协，是分工。** 最好的架构往往不是"全部迁到一边"，而是**各用其所长**：Python 做重生态的算法与数据，TS 做贴近前端的编排与界面。

---

## 七、⚠️ 不要照搬的 5 个 Python 习惯

这一节是本页最实用的部分。每一条都是"从 Python 迁到 TS 时真的会踩"的坑。

### 7.1 以为类型注解在运行时存在

```python
# Python：注解是"给人看的"，运行时基本被忽略
def process(state: State) -> dict:
    # 即使 State 是个普通 dict，IDE 也能补全，但运行时没有这回事
    return {"result": "ok"}
```

```ts
// TS：类型在编译后被完全擦除，运行时什么也没有
```

⚠️ **坑在哪**：Python 里 `isinstance(x, SomeModel)` 能真的检查；很多人以为 TS 的类型标注也能在运行时检查。

⭐ **TS 的正确做法**：想要运行时校验，**显式用 Zod**：

```ts
import { z } from "zod";

const Result = z.object({ result: z.string() });
const parsed = Result.safeParse(input);
if (!parsed.success) {
  console.error(parsed.error.issues);   // 运行时真的能拿到错误
}
```

**记住**：TS 的类型**只在编译期存在**；要运行时能力，得有**运行时的值**（Zod schema 对象）。

### 7.2 照搬 `config["configurable"]`

```python
# Python（老写法 / 遗留代码）
config = {"configurable": {"user_id": "u-1", "thread_id": "t-1"}}
graph.invoke(inputs, config)
```

```ts
// ❌ 照搬：user_id 塞进 configurable
const config = { configurable: { user_id: "u-1", thread_id: "t-1" } };
```

⚠️ **坑在哪**：在 LangGraph.js **v1** 里，**业务运行时上下文应该走 `config.context`**（有 schema、有类型），`configurable` 主要留给框架自用的键。

⭐ **正确做法**：

```ts
import { StateGraph } from "@langchain/langgraph";
import { z } from "zod";

// ✅ 业务上下文：声明 schema，走 config.context
const ContextSchema = z.object({ userId: z.string() });
const graph = new StateGraph(State, ContextSchema);
await graph.invoke(inputs, {
  configurable: { thread_id: "t-1" },   // 框架键
  context: { userId: "u-1" },           // 业务上下文
});
```

⚠️ **但别搞反**：`thread_id` 依然在 `configurable` 里，而且**永远是 `thread_id` 这个下划线写法**，不是 `threadId`。

### 7.3 混淆 `AsyncIterator` 与 async generator

```python
# Python
async def stream():
    yield chunk          # async generator
    async for x in src:  # 异步迭代
        yield x
```

```ts
// TS
async function* gen() { yield chunk; }        // async generator
for await (const x of src) { }                // for await
```

⚠️ **坑在哪**：三个概念在 TS 里长得像但不同：

| 概念 | 语法 | 类型 |
|------|------|------|
| async generator | `async function* f()` | 返回 `AsyncGenerator<T>` |
| `for await...of` | `for await (const x of it)` | 接受 `AsyncIterable<T>` |
| `Promise` | `await p` | 单个值 |

⭐ **在 LangGraph 里的具体表现**：`graph.stream()` 返回的对象**同时是 `AsyncIterable`**（可以 `for await`），但**不是** async generator（你不能 `next()` 手动拉取）。而节点里用 `config.writer()` 发的自定义数据，走的是另一条通道。

⚠️ **另一个常见错误**：`for await` 忘了写 `await`：

```ts
// ❌ 拿到的是 Promise 数组，不是值
const chunks = [];
for await (const c of stream) { chunks.push(c); }   // ✅ 正确
```

### 7.4 把 Python 的 `dict` 和 TS 的对象字面量混为一谈

```python
# Python
state = {"step": "A", "count": 1}
state["step"]        # 取
state.get("nope")   # 安全取
state.update({"x": 1})
```

```ts
// TS
const state = { step: "A", count: 1 };
state.step          // 取
state["step"]       // ❌ TS 里对象不能用字符串索引（除非有索引签名）
state.nope          // ❌ 编译报错
```

⚠️ **坑在哪**：Python 的 `dict` 是**键值对容器**，可以任意动态增删键；TS 的对象是**固定形状的结构**。TS 的 `Record<string, unknown>` 才相当于 Python 的 `dict`。

⭐ **在 LangGraph 里的具体表现**：`State.Update` 类型的字段**都是可选的**（因为节点只写要改的字段），但**不能写不存在的字段**：

```ts
// ✅ 正确：只写存在的字段
const update: typeof State.Update = { step: "A" };

// ❌ 错误：字段名拼错
const bad: typeof State.Update = { stepp: "A" };   // TS 编译报错
```

Python 那边写成 `{"stepp": "A"}` **编译期毫无反应**，运行时才炸。

### 7.5 用 Python 的装饰器思路写 TS 工具

```python
# Python：装饰器 + 注解自动生成 schema
@tool
def get_weather(city: str) -> str:
    """查询天气"""
    return f"晴，{city}"
```

```ts
// ❌ 照搬：以为 TS 里也有装饰器
function getWeather(@Param() city: string) { }   // TS 没有参数装饰器
```

⭐ **TS 的正确做法**：**调用式定义**——`tool()` 是一个普通函数，第一个参数是实现，第二个参数是配置对象：

```ts
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(
  async ({ city }: { city: string }) => `晴，${city}`,
  {
    name: "get_weather",           // 必须给：模型靠它识别工具
    description: "查询指定城市的天气",  // 必须给：模型靠它决定何时调用
    schema: z.object({              // 入参 schema：运行时校验 + 类型来源
      city: z.string().describe("城市名"),
    }),
  },
);
```

⚠️ **两个关键差异**：

1. **TS 的 `name` 和 `description` 是显式字段**——Python 用函数名 + docstring，TS 必须显式写
2. **TS 没有"从类型注解自动生成 schema"**——你得**手写 Zod schema**，或者用 `zod` 的类型推导 + `z.infer` 反向拿类型

⭐ **TS 侧避免重复写两遍的技巧**：

```ts
const WeatherArgs = z.object({ city: z.string().describe("城市名") });
type WeatherArgs = z.infer<typeof WeatherArgs>;

const getWeather = tool(
  async ({ city }: WeatherArgs) => `晴，${city}`,
  { name: "get_weather", description: "查询指定城市的天气", schema: WeatherArgs },
);
```

---

## 八、⭐ 面试价值

### 8.1 场景题

> **面试官问**：我们有个 Agent 服务是 Python 写的，要支持人工审批。现在要加一个 React 前端，让用户能在界面上看到流式输出并点"同意/拒绝"。你怎么做？

### 8.2 参考答法

⭐ **好的答法有三个层次**：

**第一层：说清技术方案**（这是及格线）

> "我会用**双栈**：Python 侧继续跑 LangGraph 图，用 `interrupt()` 实现人工介入；TS 侧写一个 BFF，负责鉴权和把 Python 的流转成 SSE 推给前端。前端用 `useStream` 或直接读 SSE 流。人工决策通过 HTTP 调 Python 侧的恢复接口，传 `Command({ resume })`，用同一个 `thread_id` 恢复执行。"

**第二层：说清边界与坑**（这会让面试官眼前一亮）

> "几个关键点：第一，**审批的状态必须在后端持久化**，不能靠前端传——所以 Python 侧一定要配 checkpointer，生产用 Postgres 不能用内存版，否则重启审批全丢。
>
> 第二，**中断不等于进程挂起**。恢复可能是另一个进程实例，所以架构上按'轮询式'设计，用户点按钮时发起一次普通请求就行，不要保持长连接占着 worker。
>
> 第三，**`interrupt()` 之前的副作用必须幂等**，因为恢复时节点会从头重跑。
>
> 第四，如果审批点只是'某个工具调用前'，其实不用上图——LangChain.js v1 的 `humanInTheLoopMiddleware` 直接能做。但我们这个需求是审批流程中间的一个判断，跨语言做不到，只能在 Python 侧手写 `interrupt()`。"

**第三层：给出取舍判断**（这是资深候选人的答法）

> "我**不会**把这个服务整体迁到 TypeScript。理由是：Agent 服务的瓶颈在模型 API 调用，不在编排代码，'TS 更快'不构成迁移理由；而且 Python 侧已经稳定运行、还可能用到 Python 独有的生态。**双栈不是妥协，是分工**——Python 做重生态，TS 做贴近前端的部分。
>
> 什么情况下我会考虑迁？三个：**需要部署到边缘运行时**、**团队技术栈要收敛**、**前后端要共享 Zod schema**。目前这三条都不成立。"

### 8.3 追问准备

| 追问 | 答法要点 |
|------|---------|
| "为什么不用 MCP？" | MCP 适合"Python 侧有一成套能力要暴露成工具给 TS 调用"。如果只是审批这一条链路，HTTP 更直接、少一跳、少一层协议。 |
| "TS 和 Python 的状态管理有什么本质区别？" | 核心是**类型系统的差异**。TS 侧 Zod 一套 schema 同时给运行时校验和静态类型，写错字段名 `tsc` 就拦住；Python 侧 Pydantic 负责运行时、Mypy 负责静态，是两条路。TS 侧节点返回值受 `Update` 类型约束，Python 侧返回 `dict` 不受静态约束。 |
| "LangGraph 的 checkpointer 两边一样吗？" | 官方后端基本对齐（内存/SQLite/Postgres/MongoDB/Redis），v1 之后两边都拆成了独立包。**差距在社区第三方后端**——很多只做了 Python 版。 |
| "新特性两边的节奏是怎样的？" | 总体是 **Python 先出、TS 跟进**，因为 Python 是 LangChain 的原生语言。但也有反例，比如 `tools` 流式模式是 **JS 独有**的。 |
| "TS 侧的状态 schema 怎么保证和 Python 侧一致？" | 如果两端都要维护同一套图定义，**别用代码生成**，直接共享一份 JSON Schema 或用 MCP 协议约定数据结构。硬要双份实现，早晚会漂移。 |

---

## 陷阱

1. **⚠️ 以为所有 snake_case 都变成 camelCase**——`thread_id` 永远不变（协议键），`__interrupt__`、`actionRequests` 这类框架 API 名也保持原样；**只有"用户要传的参数名"才转换**；
2. **⚠️ 以为 TS 的类型标注在运行时存在**——编译后完全擦除；要运行时校验必须显式用 Zod 的 `safeParse`；
3. **⚠️ 在 TS 里把业务上下文塞进 `config.configurable`**——v1 应该用 `config.context` + 声明式 `ContextSchema`；但 `thread_id` 仍在 `configurable` 且必须写成下划线；
4. **⚠️ 忘了 `thread_id` 写成 `threadId`**——框架读的是 `thread_id`，改了就不生效，症状是"每个请求都是新会话"；
5. **⚠️ 以为 Python 的 `dict` 等同于 TS 的对象字面量**——TS 的对象形状是固定的，`State.Update` 不接受不存在的字段（这是**好事**，但和 Python 的行为不同，要习惯）；
6. **⚠️ 照搬 `@tool` 装饰器的思路**——TS 没有参数装饰器，必须用**调用式** `tool(fn, { name, description, schema })`，且 `name` / `description` 要显式写；
7. **⚠️ 用"TypeScript 更快"当迁移理由**——Agent 应用的瓶颈在模型 API，编排代码占 1% 都不到；有效理由只有边缘运行时、技术栈收敛、前后端同构；
8. **⚠️ 以为 TS 版特性总是更全**——总体是 Python 先出，但 `tools` 流式模式是 **JS 独有**；
9. **⚠️ 以为两边 checkpointer 后端一样多**——官方后端基本对齐，但**社区第三方后端大多只有 Python 版**；
10. **⚠️ 在 TS 里混用 `for await` 和 `Promise.all` 的错误语义**——`Promise.all` 遇到一个 reject 就整体 reject，但其他 promise 仍在后台跑（结果被丢弃）；需要收集全部错误用 `Promise.allSettled`；
11. **⚠️ 双栈各维护一份图的定义**——同一套流程用两种语言写两遍，**早晚漂移**。要么共享 schema，要么只让一侧拥有编排逻辑。

## 小结

- ⭐ **API 对照**：模型 `initChatModel` / 工具 `tool()` / Agent `createAgent` / 中间件 `createMiddleware` / 状态 `StateSchema` / 图 `StateGraph` / 检查点 `MemorySaver`~`PostgresSaver` / 人工介入 `interrupt()` + `Command({ resume })`
- ⭐ **命名风格**：`snake_case → camelCase`，但**框架自己的 API 名和协议键名保持原样**（`thread_id`、`__interrupt__`）
- ⭐ **类型系统根本差异**：Pydantic（运行时校验 + 静态类型**两条独立的路**）↔ Zod（**一套 schema 同时给两者**）；TS 的 `StateSchema` 能推导出 `State` / `Update` 并约束节点签名，Python 做不到
- ⭐ **生态实话**：**Python 先出特性、TS 跟进**（但 `tools` 流式模式是 JS 独有）；checkpointer 官方后端基本对齐，**社区第三方后端 TS 明显少**；TS 的结构性优势是**同构、单一运行时、边缘友好、编译期类型安全**
- ⭐ **三种双栈模式**：HTTP/REST（各展所长）、MCP（Python 能力暴露成工具）、纯 TS 单栈；**双栈不是妥协，是分工**
- ⭐ **迁移决策树**：有效的迁移理由只有三个——**边缘运行时、技术栈收敛、前后端同构**；"TS 更快"不是理由
- ⚠️ **5 个别照搬的习惯**：类型注解运行时不存在、`configurable` 旧写法、`AsyncIterator` vs async generator、`dict` vs 对象字面量、`@tool` 装饰器 vs `tool()` 调用式
- ⭐ **面试答法**：说清方案 → 说清边界与坑 → 给出取舍判断（三层递进）

## 练习

1. 写一张**你自己项目里的 API 对照表**（不要抄本页的）：把 Python 侧用到的 10 个 LangChain/LangGraph API 逐个查出来，写上 TS 对应物。查完你会发现有些是 1:1 直译，有些是"概念对应但写法不同"。
2. 写一个**双栈最小可运行 demo**：Python 侧起一个 HTTP 服务返回 `"hello from python"`，TS 侧用 `fetch` 调它并打印。体会一下"多一跳网络"在延迟上的感觉。
3. ⭐ 验证第 7.1 条的坑：写一个 TS 函数，参数标注为 `{ a: string }`，然后在运行时传 `{ a: 123 }`。用 `tsc` 确认它编译不过，再加 `as any` 让它跑起来，观察运行时发生了什么——这就是"类型标注在运行时不存在"。
4. ⭐ 验证第 7.4 条的坑：在 TS 里写 `const u: typeof State.Update = { stepp: "x" }`，确认 `tsc` 报错；再在 Python 里写同样的 dict 传给节点，确认**运行前毫无反应**。
5. 验证第 7.6 条：分别用 Python 的 `@tool` 和 TS 的 `tool()` 定义一个"查天气"工具，把两边的**工具描述**（name / description / schema）列出来对照，思考"如果模型只看得懂英文描述，TS 侧要不要把 description 写成英文？"
6. ⭐ 用本库的迁移决策树分析一个具体场景："我们有个 Python 写的 PDF 解析服务，要给一个 React 应用用。里面用了 `pdfplumber` 和一个内部自研的版面分析模型。"——按决策树一步步走，写出结论和理由。
7. 准备面试答法：把第 8.2 节的三层答法**用自己的话**重写一遍（不要背原文），并准备第 8.3 节的五个追问。

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：[[typescript/03-生态连接-LangChain与LangGraph/5-LangGraph-js持久化与人工介入]] · 下一篇：[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]]
