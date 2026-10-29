---
tags: [typescript, langchain, 生态选型]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# TypeScript 生态定位与选型

> **什么时候该读**：你决定用 AI Agent 技术，但面对 LangChain.js（JS/TS 版）和 Python 版 LangChain 两套东西，不知道该选哪个、能不能混用、要不要硬迁。
> **阅读门槛**：读完 [[typescript/01-入门阶段-基础教程/00-章节目录]]，知道 TS 编译成 JS、在 Node 上跑。不需要会 LangChain——本篇就是给你做决策的。术语「Agent」「工具」「模型」会在首次出现时解释。

## 学习目标

读完本篇，你应当能够：

1. 说清 **LangChain.js 是一等公民的独立实现**，而不是 Python 版的逐行翻译
2. 用给定的**三选一决策表**，为任意一个具体项目做出语言/框架选型，并解释理由
3. 设计一个**双栈架构**（Python 侧算、TS 侧编排），并说清它用什么协议连接
4. 画出 LangChain.js / LangGraph.js / Deep Agents / LangSmith 四者的关系图
5. 解释 **LangChain.js 为什么用 Zod 而不是 Pydantic**，并说清这对写 TS 代码意味着什么
6. 在 TS 项目里**正确安装并锁定** LangChain.js v1 的依赖版本
7. 认出 **`@langchain/classic`** 是什么，在什么情况下才需要它

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]] | 本篇大量做「Python 版 vs TS 版」对照；不懂差异就体会不到「为什么 TS 版不是翻译」 |
| [[typescript/02-中段阶段-专题精讲/工程化与工具链/3-构建工具与包管理]] | 本篇要装 npm 依赖、pin 版本；还要知道 `package.json` / `npm install` 怎么用 |
| [[langchain/overview/LangChain概述]] | LangChain 是什么、Agent / 工具 / 模型分别是什么——本篇只在必要处重复 |
| [[python/basics/Python_基础概览]] | 本库主线是 TS，但对照需要知道 Python 侧的能力边界 |

---

## 一、⭐ LangChain.js 不是「Python 版的翻译」

### 1.1 先说清几个专有名词

阅读本篇需要先认识四个词（后文不再重复解释）：

| 术语 | 一句话定义 |
|------|-----------|
| **大语言模型（LLM）** | 能读文本、生成文本的模型，比如 GPT、Claude。你调它一次叫一次「调用」 |
| **工具（Tool）** | 一个**有明确入参和出参**的普通函数，模型可以决定「要不要调用它、传什么参数」。比如查天气、查数据库 |
| **Agent（智能体）** | 一个**循环**：调模型 → 模型说「我要调工具 X，参数是 Y」 → 执行工具 → 把结果喂回模型 → 直到模型不再要工具为止 |
| **中间件（Middleware）** | 插在 Agent 循环各处的**拦截器**，用来改提示词、限流、脱敏、审批、记录日志 |

### 1.2 「翻译版」是个陷阱

很多人以为 LangChain.js 就是 Python 版换个语法：

```text
❌ 错误的心智模型

Python 版                        TS 版（想象成翻译）
create_agent(...)        →      createAgent(...)      ← 恰好一样，但只是巧合
@tool 装饰器             →      ???                    ← 完全不同
class MyState(BaseModel) →      interface MyState {}  ← 完全不同
response_format=Weather  →      responseFormat=Weather ← 又恰好一样
```

⭐ **核心事实**：LangChain.js（`langchain` npm 包，**v1** 线）是 LangChain 团队**独立设计、独立发版**的 JS/TS 实现。它有自己的 API 风格（工具用 `tool()` 函数而不是装饰器、schema 用 Zod 而不是 Pydantic）、自己的演进节奏（有时 TS 侧先出特性，有时 Python 侧先出），以及自己的社区。

**它不是「Python 版翻译」的意思是**：

| 维度 | Python 版 | LangChain.js（v1） |
|------|-----------|-------------------|
| 定义工具 | `@tool` **装饰器** + **函数 docstring** 作为描述 | `tool(fn, { name, description, schema })` **函数式** + 显式 `description` 字段 |
| 参数 schema | **Pydantic** `BaseModel` 类 | **Zod** schema 对象（`z.object({...})`） |
| 静态类型 | Python 类型注解（运行时会被反射读取） | TS 类型 + `z.infer` 从 schema 反推（见第五节） |
| 建 Agent | `create_agent(model=..., tools=...)` | `createAgent({ model, tools })`（**一个对象参数**） |
| 状态定义 | `class AppState(AgentState)` 继承 | 中间件里的 `stateSchema: z.object({...})`（Zod 对象） |
| 运行时上下文 | `config={"configurable":{...}}` | `invoke(input, { context: {...} })` + `contextSchema` |
| 工具绑定 | `model.bind_tools(tools)` **手动** | `createAgent` **内部自动绑定**（手动绑会抛 `MultipleToolsBoundError`） |

⭐ **为什么这些差异是「设计」而不是「偷懒」**：JS/TS 世界没有 `Protocol` / 运行时反射式的装饰器生态，Zod 又同时能提供**运行时校验**和**静态类型推导**——所以 TS 侧选择了 Zod 路线，这是**更适合 JS/TS 的方案**，而不是抄一遍 Python。

> 📖 想看 Python 侧的对应关系，本库有 [[langchain/agents/Agent代理]]（Agent 概念）和 [[langchain/tools/工具]]（工具概念）的 Python 版详解。**但请注意**：那两页讲的是 Python，API 名字和写法都不能直接抄到 TS。

### 1.3 版本基线（本篇涉及的所有版本事实）

| 项 | 基线 |
|---|---|
| **LangChain.js** | **v1** 线（npm 包 `langchain`）；核心入口 `createAgent` / `tool` / `createMiddleware` |
| **LangGraph.js** | **v1** 线（npm 包 `@langchain/langgraph`） |
| **Node.js** | 本库基线 **20+**（`langchain@1.5.x` 的 `engines` 字段就是 `node >= 20`）；生产请用 **22 / 24 LTS** |
| **TypeScript** | 本库基线 **TS 6 / 7**（见 [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]]） |
| **Zod** | **v3 / v4 均可**（`langchain` 依赖 `zod ^3.25.76 \|\| ^4`） |

---

## 二、⭐ 三选一决策表

### 2.1 决策表

| 场景 | 选择 | 理由 |
|---|---|---|
| 团队是 TS/前端为主，要接浏览器/边缘 | **LangChain.js** | 同构、类型贯通 |
| 已有 Python AI 服务（模型微调、数据管道、评估） | **Python 版** | 生态成熟、库更全 |
| 前后端混合 | **双栈 + HTTP/MCP 打通** | 边界清晰 |

### 2.2 逐条展开：什么时候选 LangChain.js

选 TS 侧的真实理由（不是「因为我们熟 TS」，而是这些）：

1. **同构（isomorphic）**：同一份 Agent 代码能在浏览器、Node、边缘运行时跑。浏览器端可以用 `dangerouslyAllowBrowser` 谨慎地跑轻量 Agent，或者把 Agent 放在边缘（Vercel Edge、Cloudflare Workers）。
2. **类型贯通**：从 Zod schema → 工具函数参数 → 结构化响应，全链路一个类型系统（见第五节和下一篇）。**Python 做不到这种端到端推导**——Python 的类型注解在运行时不被强制检查。
3. **前端已有基建**：你的团队已经有 React、类型化 API client、Zod 校验中间件。引入 LangChain.js 几乎不增加认知负担。
4. **和现有 TS 服务同构部署**：一套构建、一套测试、一套 CI。

### 2.3 什么时候选 Python 版

1. **生态成熟度**：Python 侧第三方集成更多、更新更快——尤其**评估（evaluation）**、**微调**、**数据管道**这三块。
2. **机器学习生态**：PyTorch、HuggingFace、Transformers、peft、datasets——这些在 TS 侧没有对等物。
3. **已有的 AI 服务资产**：你的模型微调脚本、离线评估流水线、数据清洗管道都在 Python 里。
4. **社区示例密度**：Stack Overflow / GitHub 上 Python 版的坑和答案最多。

### 2.4 什么时候选双栈（最常见的真实答案）

**双栈不是妥协，是正确的架构。** 关键在于：**用一道清晰的边界把两种语言隔开**。

```
┌─────────────────────────────────────────────────────────┐
│  前端 / Node BFF（TypeScript）                            │
│  · 交互、流式 UI、SSE/WebSocket 推送                     │
│  · 用户会话、鉴权、租户隔离                               │
│  · Agent 编排（LangChain.js）——需要调工具、需人工介入    │
└────────────────────────┬────────────────────────────────┘
                         │  ← 这条边界是关键
                         │  HTTP / SSE  或  MCP
┌────────────────────────▼────────────────────────────────┐
│  AI 服务（Python）                                        │
│  · 模型微调、离线评估、数据处理                            │
│  · 重型检索（RAG 索引构建、embedding 批处理）              │
│  · 内部不对前端暴露的模型服务                             │
└─────────────────────────────────────────────────────────┘
```

⭐ **边界划在哪**：把「**慢的、批量的、离线计算的**」放 Python，把「**快的、交互的、面向用户的**」放 TS。

---

## 三、⭐ 双栈协作的现实做法

### 3.1 两种连接协议

| 协议 | 适合什么 | 代价 |
|------|---------|------|
| **REST / HTTP**（+ SSE 做流式） | 服务之间边界清晰、团队独立部署、已有网关 | 要自己设计接口契约、要处理鉴权、重试、序列化 |
| **MCP**（Model Context Protocol，模型上下文协议） | **把「工具」跨语言暴露**——Python 侧提供服务，TS 侧当工具用 | 需要 MCP server / client 基础设施；生态还在演进 |

### 3.2 用 MCP 打通（推荐用于「跨语言暴露工具」）

⭐ **MCP 是一个开放协议**，标准化了「应用程序如何向 LLM 提供工具和上下文」。**它不关心工具那侧是什么语言写的**——这正是它解决双栈问题的关键。

TS 侧通过 `@langchain/mcp-adapters` 连 MCP server：

```ts
import { createAgent } from "langchain";
import { ChatAnthropic } from "@langchain/anthropic";
import { MultiServerMCPClient } from "@langchain/mcp-adapters";

// MCP server 可以是 Python 写的——协议层不关心语言
const client = new MultiServerMCPClient({
  math: {
    transport: "stdio",
    command: "python",                                  // ⚠️ 这行后面接的是 Python 解释器
    args: ["/srv/mcp/math_server.py"],
  },
  weather: {
    transport: "http",
    url: "http://localhost:8000/mcp",
  },
});

const tools = await client.getTools();   // MCP 工具变成 LangChain 工具，和本地工具没区别
const agent = createAgent({
  model: new ChatAnthropic({ model: "claude-sonnet-4-6" }),
  tools,
});
```

> 📖 MCP 协议本身的机制、传输方式、server 怎么写，见 [[langchain/modules/langchain-mcp/概述]] 和 [[langchain/modules/langchain-mcp/MCP服务器开发]]（以 Python 侧讲解，但协议本身是跨语言的）。

### 3.3 用 REST 打通（推荐用于「跨语言暴露完整能力」）

如果 Python 侧要暴露的不是单个工具，而是**一整套能力**（比如「跑一次完整 RAG 检索并重排」），REST 更合适：

```ts
// TS 侧：把 Python 的 RAG 服务包装成一个工具
import { tool } from "langchain";
import * as z from "zod";

const searchKnowledgeBase = tool(
  async ({ query, topK }) => {
    const res = await fetch("http://ai-service.internal/search", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ query, topK }),
      signal: AbortSignal.timeout(10_000),   // ⚠️ 一定要设超时，否则工具挂住会拖死整个 Agent
    });
    if (!res.ok) throw new Error(`RAG 服务返回 ${res.status}`);
    return JSON.stringify(await res.json());
  },
  {
    name: "search_knowledge_base",
    description: "在企业知识库中检索相关文档片段。用户询问公司内部制度、产品文档、历史工单时使用。",
    schema: z.object({
      query: z.string().describe("检索关键词或自然语言问题"),
      topK: z.number().int().min(1).max(20).default(5),
    }),
  },
);
```

⭐ **关键点**：TS 侧的 Agent **完全不知道** `search_knowledge_base` 背后是 Python、Go 还是 Rust——它只看到一个「有名字、有描述、有 schema 的函数」。**这就是接口隔离的力量**，也是 MCP 想标准化的事情。

### 3.4 ⚠️ 双栈最常见的三个坑

1. **边界画在中间**：把「提示词拼装」放 Python、把「工具调用」放 TS，最后两边都要改才能加一个工具。**边界应该画在「一次远程调用」这个粒度上**。
2. **错误语义不统一**：Python 抛 `ValueError`，TS 侧看到的是 HTTP 500。**必须约定错误契约**（比如统一 `{ error: { code, message } }`），否则 TS 侧的 Agent 无法判断「这是可重试的错误还是业务错误」。
3. **双份依赖版本漂移**：Python 侧升级 LangChain、TS 侧没升级，MCP 工具的 schema 结构可能变。**给两边都加契约测试**。

---

## 四、生态定位图

### 4.1 四个包/服务的关系

```
                        ┌──────────────────────────┐
                        │      LangSmith           │
                        │  追踪 / 调试 / 评估       │  ← 观测层，跨所有框架
                        │  (langsmith SDK)         │
                        └────────────▲─────────────┘
                                     │ 上报 trace
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
┌───────▼────────┐          ┌────────▼─────────┐        ┌─────────▼────────┐
│  Deep Agents   │          │   LangChain.js   │        │  LangGraph.js    │
│  (deepagents)  │          │    (langchain)   │        │(@langchain/      │
│                │          │                 │        │   langgraph)     │
│ 「 batteries   │          │  Agent 外壳：     │        │                 │
│   included 」  │          │  createAgent     │        │ 状态图编排：      │
│                │          │  tool            │        │  StateGraph      │
│ · 规划工具      │          │  createMiddleware│        │  节点/边/条件边   │
│ · 虚拟文件系统  │          │  中间件           │        │  编译            │
│ · 子 Agent     │          │                 │        │                 │
└───────┬────────┘          └────────┬─────────┘        └─────────┬────────┘
        │                            │                            │
        └────────────────────────────┴────────────────────────────┘
                                     │
                            两者都建立在
                       LangChain.js v1 的 createAgent 之上
```

### 4.2 逐个说明

| 名字 | npm 包 | 定位 | 什么时候用 |
|------|--------|------|-----------|
| **LangChain.js** | `langchain` | **Agent 外壳**（harness）：`createAgent` + 工具 + 中间件 | 绝大多数 Agent 需求；95% 的场景到这里就够了 |
| **LangGraph.js** | `@langchain/langgraph` | **状态图编排**：显式的状态、节点、边、条件跳转 | 需要确定性流程 + Agent 混合；需要把多个 Agent 拼成工作流；需要细粒度控制状态合并 |
| **Deep Agents** | `deepagents` | **「开箱即用」的 Agent 脚手架**：内置规划、虚拟文件系统、子 Agent 派生 | 长任务、需要文件工作区、需要委派子任务 |
| **LangSmith** | `langsmith` | **可观测性 / 评估**：把每次调用记录成 trace 树，在 UI 里回放 | 调试「模型为什么这么答」；回归评估；生产监控 |

⭐ **一句话记住四层关系**：

> **LangChain.js** 是「Agent 的运行时」，**LangGraph.js** 是「更底层的编排引擎」，**Deep Agents** 是「在 LangChain.js 之上预装了一堆能力的成品」，**LangSmith** 是「在旁边看着并记录一切的眼睛」。

### 4.3 ⭐ 关键事实：`createAgent` 底层就是 LangGraph 的 ReAct 循环

这决定了「用 `createAgent` 就自动获得什么」：

| 自动获得 | 含义 |
|---------|------|
| **持久化（checkpoint）** | 传 `checkpointer` 后，对话状态按 `thread_id` 落盘，重启还在 |
| **流式（streaming）** | `agent.stream()` 直接可用 |
| **人工介入（HITL）** | 中断 → 等人工批准 → 从断点恢复 |
| **Time travel** | 回滚到任意历史检查点，重跑一条分支 |

> 📖 LangGraph 的状态图模型见 [[langgraph/overview/LangGraph概述]] 和 [[langgraph/modules/langgraph_core/StateGraph]]；本专题第 4–5 篇会专门讲 LangGraph.js。

---

## 五、⭐ 为什么 LangChain.js 用 Zod 而不是 Pydantic

### 5.1 Pydantic 在 Python 里到底做了什么

Python 的 Pydantic 干两件事：

1. **运行时校验**：拿到的数据不符合声明就抛错；
2. **类型提示来源**：配合 `mypy` / `pyright` 做静态检查。

⚠️ 但 Python 的这两件事是**分开**的——`class Weather(BaseModel)` 和 `def f(w: Weather)` 之间靠**约定**（注解）连接，运行时无法保证 `f` 收到的真是 `Weather` 实例（除非你自己写 `isinstance` 检查）。

### 5.2 Zod 在 TS 里把两件事合成一件

**Zod schema 同时是：**

- ✅ **运行时校验器**——`schema.parse(data)` 真的会检查；
- ✅ **静态类型来源**——`z.infer<typeof schema>` 让 TS 推导出类型；
- ✅ **两者的唯一真相源**——你只写一次。

```ts
import * as z from "zod";

// 一份声明，两个用途
const WeatherInput = z.object({
  city: z.string().describe("要查询天气的城市名"),
  units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
});

type WeatherInput = z.infer<typeof WeatherInput>;
//             ^^^^^^^^^^^^^^^ 从 schema 反推出类型，不用手写 interface

function showWeather(input: WeatherInput): void {
  console.log(input.city, input.units);   // ⭐ TS 知道 units 是 "celsius" | "fahrenheit"
}
```

⭐ **为什么这对 Agent 特别重要**：Agent 的数据流是「模型输出的 JSON → 结构化响应」。这个 JSON 是**运行时才产生的、来源不可信的数据**。TS 侧必须在运行时校验它，否则一个 `city: 42` 就能让下游 `.toUpperCase()` 崩溃。**Zod 让「校验」和「类型」共用一份声明**，这在 Python 里是做不到的。

> 📖 `z.infer` 背后的原理是**条件类型 + `infer`**（TS 的类型推导机制），见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]]。

### 5.3 三点具体差别

| 方面 | Python（Pydantic） | TS（Zod） |
|------|-------------------|-----------|
| **声明位置** | 必须先 `class` 一个类 | 直接写 `z.object({...})`，不产生运行时代码之外的类 |
| **类型来源** | 注解 `w: Weather`（和运行时类同名但**可以不一致**） | `z.infer` **从 schema 推导**，不可能不一致 |
| **传给框架** | `args_schema=WeatherInput`（要另写一个类） | `schema: WeatherInput`（就是那个对象） |

> 📖 Python 版的工具 schema 写法见 [[langchain/tools/工具]] 的「高级 Schema 定义」一节——注意它需要**额外定义一个 Pydantic 类**，TS 侧不需要。

---

## 六、与 Python 版的差异概览

### 6.1 ⚠️ 核心规律：Python 先出特性，TS 跟进

⭐ **这是选型时必须知道的事实**：LangChain 的新特性**通常先在 Python 侧落地**，TS 侧随后跟进（时间差通常是几周到几个月）。

⭐ **具体表现（checkpointer 后端数量）**：LangGraph 的**持久化后端（checkpointer，即保存对话状态的地方）在 Python 侧明显更多**：

| 后端 | Python 包 | TS 包 |
|------|-----------|-------|
| 内存 | `langgraph-checkpoint` | `@langchain/langgraph`（`MemorySaver`）/ `@langchain/langgraph-checkpoint` |
| SQLite | `langgraph-checkpoint-sqlite` | `@langchain/langgraph-checkpoint-sqlite` |
| PostgreSQL | `langgraph-checkpoint-postgres` | `@langchain/langgraph-checkpoint-postgres` |
| MongoDB | `langgraph-checkpoint-mongodb` | `@langchain/langgraph-checkpoint-mongodb` |
| Redis | `langgraph-checkpoint-redis` | `@langchain/langgraph-checkpoint-redis` |
| **AWS / DynamoDB / Bedrock / Valkey** | `langgraph-checkpoint-aws` | ⚠️ **无官方 TS 包** |
| **Azure Cosmos DB** | `langgraph-azure-cosmosdb` | ⚠️ **无官方 TS 包** |
| **CockroachDB / Aerospike / ScyllaDB / Tigris / TypeDB** | 各有独立包 | ⚠️ **无官方 TS 包** |

**结论**：如果你的技术选型依赖某个特定数据库做 Agent 状态持久化，**先去确认 TS 侧有没有对应包**。这是「Python 先行」最常咬人的地方。

⭐ **另一个容易忽略的差异**：Python 侧的 `langgraph-checkpoint`（内存版）和 TS 侧的 `MemorySaver` **类名不同**——迁移时别直接搜 `InMemorySaver`。

### 6.2 差异总览表（选型时对照用）

| 关注点 | Python 版 | LangChain.js（v1） |
|--------|-----------|-------------------|
| 主入口 | `create_agent(...)` | `createAgent({...})` |
| 工具定义 | `@tool` 装饰器 + docstring | `tool(fn, { name, description, schema })` |
| 参数 schema | Pydantic `BaseModel` | Zod `z.object({...})` |
| 提示词参数 | `prompt=` | ⚠️ **`systemPrompt=`**（`prompt` 是旧名，v1 已改名） |
| 状态定义 | `class AppState(AgentState)` | 中间件里的 `stateSchema`（Zod） |
| 运行时上下文 | `config={"configurable":{...}}` | ⚠️ **`{ context: {...} }`**（`configurable` 只留给 `thread_id` 这类图配置） |
| 工具绑定 | 手动 `model.bind_tools(tools)` | ⚠️ **自动**（手动绑定会抛 `MultipleToolsBoundError`） |
| 中间件钩子 | 装饰器 `@before_model` 等 | `createMiddleware({ beforeModel, wrapModelCall, ... })` |
| 流式节点名 | `"agent"` | ⚠️ **`"model"`**（v1 改的） |
| 遗留 API | — | ⚠️ `createOpenAIFunctionsAgent` / `AgentExecutor` **在 1.x 已不再导出** |

> 📖 Python 版钩子的详细机制见 [[langchain/middleware/钩子机制详解]]；TS 版在下一篇。

---

## 七、环境准备

### 7.1 安装

```bash
# 核心三件套 + schema 库
npm install langchain @langchain/core zod

# 至少一个模型集成包（按你用的提供商选）
npm install @langchain/openai        # 或 @langchain/anthropic / @langchain/google-genai
```

| 包 | 作用 |
|----|------|
| `langchain` | v1 主包：`createAgent` / `tool` / `createMiddleware` / `initChatModel` / 预置中间件 |
| `@langchain/core` | 底层抽象：消息类型、工具基类、Runnable 协议。多数功能从这里**重新导出**，直接用 `langchain` 即可 |
| `zod` | schema 库。**它是 peer 依赖——不装会直接报模块找不到** |
| `@langchain/openai` 等 | 具体模型集成包。**不在 `langchain` 主包里** |

### 7.2 ⚠️ 显式 pin 版本

⭐ **为什么必须 pin**：LangChain.js 发版非常频繁，minor 版本就可能改行为（比如 `streamEvents` 的节点名、预置中间件的导出名）。**不锁版本的 CI 会在某天突然红。**

```bash
# ⚠️ 显式指定版本范围，不要用 ^ 或 latest
npm install langchain@1.5 @langchain/core@1.2 zod@4
```

对应的 `package.json`：

```json
{
  "dependencies": {
    "langchain": "~1.5.0",
    "@langchain/core": "~1.2.0",
    "@langchain/openai": "~1.6.0",
    "zod": "~4.6.0"
  }
}
```

⭐ **版本要配套，不能各挑各的最新版**：`langchain@1.5.x` 的 peer 依赖是 **`@langchain/core@^1.2.20`**——所以 core 必须锁在 1.2.x，升到 1.3.x 会 **peer 依赖冲突**。模型集成包也一样：`@langchain/openai@1.6.x` 要求 `core@^1.2.14`，`@langchain/openai@1.7.x` 就要求 `core@^1.3.0` 了。

⭐ **锁文件的纪律**：`package.json` 写**范围**（`~1.5.0`），`package-lock.json` 锁**精确版本**。两者都提交进仓库（详见 [[typescript/02-中段阶段-专题精讲/工程化与工具链/3-构建工具与包管理]]）。

### 7.3 冒烟测试：确认环境是通的

```ts
// smoke.ts
import { createAgent } from "langchain";

const agent = createAgent({
  model: "openai:gpt-4o",           // 字符串 ID：内部走 initChatModel 自动选提供商
  systemPrompt: "只回答一句话。",
});

const result = await agent.invoke({
  messages: [{ role: "user", content: "1+1 等于几？" }],
});

console.log(result.messages.at(-1)?.text);
```

运行前先设好环境变量：

```bash
export OPENAI_API_KEY="sk-..."
npx tsx smoke.ts     # 用 tsx 直接跑 TS，见工程化那篇
```

⚠️ **必须用 `await`**：Agent 的调用是异步的（返回 Promise），忘了 `await` 会拿到一个 Promise 对象而不是结果——这正是 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] 讲过的经典坑。

---

## 八、⚠️ `@langchain/classic`：遗留功能去哪了

### 8.1 发生了什么

LangChain.js **v1** 把主包「瘦身」了：`langchain` 现在**只导出构建 Agent 必需的东西**。所有 v0.x 的遗留功能被搬到了单独的包 **`@langchain/classic`**。

```ts
// ❌ v0.x 写法——v1 里会报「找不到导出」
import { LLMChain } from "langchain/chains";
import { ConversationalRetrievalQAChain } from "langchain";

// ✅ v1 写法
import { LLMChain } from "@langchain/classic/chains";
import { ConversationalRetrievalQAChain } from "@langchain/classic";
```

```bash
npm install @langchain/classic
```

### 8.2 `@langchain/classic` 里有什么

| 内容 | 说明 |
|------|------|
| **旧版 Chains** | `LLMChain`、`ConversationalRetrievalQAChain`、`RetrievalQAChain`、`StuffDocumentsChain`、`MapReduceDocumentsChain`、`RefineDocumentsChain` 等 |
| **Indexing API** | `RecordManager` 等向量库索引管理 |
| **`@langchain/community` 的再导出** | 以前从 `langchain` 直接 import 的社区集成 |
| **其他已弃用功能** | 各种被 v1 新 API 取代的工具函数 |

### 8.3 ⭐ 什么时候该用它，什么时候不该

| 场景 | 做法 |
|------|------|
| **新项目** | ❌ **不要装**。直接用 `createAgent` |
| **老项目升级，想先不重写** | ✅ 装上，改 import 路径，先跑起来再逐步迁移 |
| **在找某个 v0.x 的 API** | ⚠️ 先查它是否**已被废弃**。如果它有 v1 等价物，**用 v1 的** |

⭐ **判断口诀**：`@langchain/classic` 是**临时过渡工具**，不是「另一个选择」。看到自己的代码里出现 `@langchain/classic`，就把它当成一个待办事项。

---

## 陷阱

1. **⚠️ 以为 LangChain.js 是 Python 版的翻译**——它是独立实现：工具用 `tool()` 函数不是装饰器，schema 用 Zod 不是 Pydantic，状态用中间件声明不是继承类；把 Python 代码逐行翻成 TS 一定错；
2. **⚠️ 以为「双栈」意味着把同一段逻辑写两遍**——双栈意味着**用一道边界把两种语言隔开**（一次远程调用），不是同一逻辑写两遍；
3. **⚠️ 忘了给跨语言调用设超时**——`fetch` 不带超时时，下游挂住会拖死整个 Agent 循环；用 `AbortSignal.timeout(ms)`；
4. **⚠️ 忘了跨栈的错误契约**——Python 抛的异常在 TS 侧只表现为 5xx，Agent 无法判断该不该重试；两边必须约定统一错误结构；
5. **⚠️ 以为「Python 先出特性、TS 跟进」意味着 TS 侧总是落后一点**——有的特性 TS 侧先落地（比如 v1 的 `contentBlocks` 内容块在 JS 侧对齐更快）；**关键能力一律去查当前版本的官方文档**，别靠印象；
6. **⚠️ 选型时忽略 checkpointer 后端数量**——Python 侧的持久化后端明显更多，AWS / Cosmos DB / CockroachDB 等在 TS 侧**没有官方包**；选型前先确认；
7. **⚠️ 不 pin 依赖版本**——LangChain.js minor 版本就会改行为（节点名、导出名），不锁版本 CI 迟早红；`package.json` 写范围 + lock 文件锁精确版本；
8. **⚠️ 忘了装 `zod`**——`zod` 是 peer 依赖，不装直接 `Cannot find module 'zod'`；
9. **⚠️ 以为装了 `langchain` 就有了 OpenAI 支持**——模型集成是独立包（`@langchain/openai` 等），主包不内置任何提供商；
10. **⚠️ 在新项目里装 `@langchain/classic`**——它是 v0.x 的过渡包，只维护 bug 和安全修复，不会有新特性；看到它就当成待办；
11. **⚠️ 以为 `npm install langchain` 会自动装 `@langchain/core`**——虽然很多功能从 core 再导出，但显式装上并 pin 版本更可控；
12. **⚠️ 用 `^` 而不是 `~` 锁版本**——`^1.5.0` 允许自动升级到 `1.6.0`，跨 minor 就可能变行为；用 `~` 限制在 minor 内；
13. **⚠️ 以为 TS 侧版本号和 Python 侧能对应**——`langchain@1.5`（TS）和 `langchain==1.5`（Python）是**两套独立的版本线**，能力不同、发布节奏不同，不要拿一边推断另一边；
14. **⚠️ 各包各挑最新版，结果 peer 依赖冲突**——`langchain@1.5.x` 要求 `@langchain/core@^1.2.20`；模型集成包还有自己的 core 版本要求（`@langchain/openai@1.6.x` → `core@^1.2.14`，`1.7.x` → `core@^1.3.0`）。**三个包要锁在同一条版本线上**；
15. **⚠️ 以为 `createAgent` 等于「全部的 LangChain」**——它是 Agent 外壳；需要显式状态图编排要用 LangGraph.js，需要预装能力用 Deep Agents。

## 小结

- ⭐ **LangChain.js 是独立实现**，不是 Python 版翻译：工具用 `tool()` 函数 + Zod、schema 用 Zod、状态在中间件里声明、上下文走 `{ context }`
- ⭐ **三选一**：TS/前端/边缘 → LangChain.js；已有 Python AI 资产（微调/评估/管道）→ Python 版；混合 → **双栈 + HTTP/MCP 打通**
- ⭐ **双栈的边界画在「一次远程调用」上**；工具级跨语言暴露用 **MCP**（`@langchain/mcp-adapters`），能力级用 **REST**
- ⭐ **生态四层**：LangChain.js（Agent 运行时）/ LangGraph.js（编排引擎）/ Deep Agents（成品脚手架）/ LangSmith（观测与评估）
- ⭐ **`createAgent` 底层是 LangGraph 的 ReAct 循环** → 自动获得持久化、流式、HITL、time travel
- ⭐ **Zod 一份声明两用**：运行时校验 + `z.infer` 静态类型；这是 TS 侧相对 Python 的**结构性优势**
- ⚠️ **Python 先出特性、TS 跟进**：TS 侧 checkpointer 后端明显更少，选型前先确认
- ⚠️ **显式 pin 版本**：`langchain@1.5` / `@langchain/core@1.2` / `zod@4`，配 `~` 范围 + lock 文件；**三个包锁在同一条版本线上**
- ⚠️ **`@langchain/classic` 是 v0.x 遗留功能的临时安置所**，新项目不要装

## 练习

1. 用本篇的**三选一决策表**，给下面三个项目各写一句选型结论 + 理由：(a) 一个 Next.js 应用，要给用户一个带联网搜索的对话助手；(b) 一个要微调 Llama 并做离线评估的团队；(c) 一个前端做交互、后端已有 Python RAG 服务的创业项目。
2. 查一下 `@langchain/langgraph-checkpoint-postgres` 的最新版本号，说明它和 `langchain` 主包**不是同一个版本线**这件事对依赖管理意味着什么。
3. 写一个 `package.json` 的 `dependencies` 片段，要求：LangChain.js v1、Zod v4、OpenAI 集成，全部用 `~` 范围，并说明为什么不用 `^`。
4. 设计一个双栈架构图：Python 侧跑「文档切分 + embedding + 向量检索」，TS 侧跑「Agent 编排 + SSE 推送」。明确写出边界处的接口契约（请求/响应/错误三部分）。
5. 解释：`@langchain/classic` 里的 `RetrievalQAChain` 在 v1 里有没有 `createAgent` 的等价物？如果没有，说明为什么这个能力值得用 REST 而不是强行在 TS 侧重写。
6. 用 Zod 写一个 `z.object({ city: z.string(), days: z.number().int().min(1).max(7) })`，然后用 `z.infer` 推出类型，并写一个函数接受该类型、返回「城市 + 天数」的字符串。说明：如果不用 `z.infer` 而手写 `interface`，会失去什么保障？

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：无（本专题第一篇） · 下一篇：[[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]]
