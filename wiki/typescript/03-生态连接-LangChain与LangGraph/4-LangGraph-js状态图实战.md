---
tags: [typescript, langgraph, 状态图, StateSchema, reducer]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# LangGraph.js 状态图实战

> **什么时候该读**：你已经能用 LangChain.js 的 `createAgent` 写出一个会调工具的 Agent，但业务开始出现"分支"（要不要走人工审核）、"并行"（同时查三个数据源）、"中途等人"（拿到工单号再问人）这类需求，`createAgent` 的固定循环就装不下了——这时候该上状态图。
> **阅读门槛**：读完 [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]]（知道 `tool()`、`createAgent` 怎么用）以及 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]]（泛型、`typeof`、条件类型）。本篇大量使用 `typeof X.State` 这种"从值推出类型"的写法，这是 TS 泛型能力最直接的落点，看不懂可以回去重读 2.3。

## 学习目标

读完本篇，你应当能够：

1. 说清 `createAgent` 与状态图的分界线，并判断一个需求"该不该上图"
2. 用 `StateSchema`（基于 Zod）定义状态，并分辨四种通道类型（LastValue / ReducedValue / MessagesValue / UntrackedValue）各自的行为
3. ⭐ 用 `typeof State.State` 和 `typeof State.Update` 推导类型，并用它约束节点函数的入参与返回值
4. ⭐ 判断哪些字段必须显式给 reducer，并解释"为什么并行节点写同一通道会直接抛错"而不是"静默丢数据"
5. 用 `addNode` / `addEdge` / `addConditionalEdges` / `START` / `END` / `compile()` 搭出可运行的图
6. 用 `Command` 实现"更新状态 + 跳转"，用 `Send` 实现动态扇出（map-reduce）
7. 用 `{ state, input, output }` 多 schema 把内部状态和对外接口分开
8. 用 `ToolNode` + `toolsCondition` 把工具接进图，并解释它和 `createAgent` 的差别

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]] | `createAgent`、`tool()`、Zod schema 的用法是本篇的前提 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]] | `typeof`（类型位置）、泛型、条件类型是推导状态类型的基础 |
| [[langgraph/overview/LangGraph概述]] | 状态图的整体概念（节点 / 边 / 状态 / 超步） |
| [[langgraph/modules/langgraph_state/状态定义]] | Python 版的 `Annotated` reducer 概念，本篇讲它对应物 |
| [[langgraph/modules/langgraph_core/节点]] | 节点在 Python 版里的定义，读 Python 页时需要 |

---

## 一、先想清楚：什么时候该上图

### 为什么不能什么都用 `createAgent`

`createAgent`（LangChain.js v1）帮你封装了一个**固定循环**：调模型 → 模型要调工具就执行工具 → 把工具结果塞回消息 → 再调模型 → 直到模型不再要工具。

这个循环覆盖了绝大多数"聊天 + 工具"的场景。但下面这些需求它装不下：

| 需求 | `createAgent` 能否胜任 | 原因 |
|------|----------------------|------|
| 调工具、循环直到模型满意 | ✅ | 这就是它的设计目标 |
| 固定顺序的流水线（先分类、再检索、再生成） | ❌ | 循环是"模型驱动"的，节点顺序是"代码驱动"的 |
| 某些步骤必须**并行**执行（同时查 3 个库） | ❌ | 工具调用是串行 await 的 |
| 中途**等人**审批才能继续 | ⚠️ 有中间件方案，但语义受限 | 只能在工具调用处停 |
| 需要把状态分给不同的人看（客服看 A 部分、财务看 B 部分） | ❌ | 状态被 `messages` 一个通道包住 |

⭐ **判断标准（三条同时满足才上图）**：

1. **需要显式的节点与边**——你能画出流程图，能说出"第几步做什么"
2. **需要中断与恢复**——执行到某处停下来，等人回来再继续
3. **需要确定性 + 自主性混合**——一部分步骤写死（查数据库），一部分交给模型决定（要不要再查一次）

只满足第 1 条，可以考虑用普通的 TypeScript 函数编排；三条都不满足，`createAgent` 就够了。整体概念见 [[langgraph/overview/LangGraph概述]]。

### 状态图的心智模型

状态图（**StateGraph**）由三样东西组成：

- **状态（State）**：所有节点共享的一份数据快照
- **节点（Node）**：接收状态、干活、返回"状态要改哪些部分"的函数
- **边（Edge）**：决定状态更新后接下来跑哪个节点

引擎不是按"行号顺序"跑，而是按**超步（superstep）**推进：同一超步里的节点**同时**执行，它们对状态的写入在超步边界合并。所以"两个节点同时改同一个字段"是这套模型里最需要小心的事——这正是本篇第三节要讲透的。

Python 版的同一套概念见 [[langgraph/modules/langgraph_core/StateGraph]]。

---

## 二、⭐ 用 `StateSchema` 定义状态

### 2.1 为什么是 Zod 而不是 TypeScript interface

LangGraph.js v1 里定义状态有两种方式：

| 方式 | 状态 | 说明 |
|------|------|------|
| `new StateSchema({...})` | ✅ 当前推荐 | 基于 Zod，**一套 schema 同时给运行时校验和静态类型** |
| `Annotation.Root({...})` | ⚠️ 遗留别名 | 仍可用，但新代码优先 `StateSchema` |

`StateSchema` 的写法：

```ts
import { StateSchema, ReducedValue, MessagesValue, UntrackedValue } from "@langchain/langgraph";
import { z } from "zod";

const AgentState = new StateSchema({
  // ① 普通 Zod schema → LastValue 通道：后写的覆盖先写的
  currentStep: z.string(),
  // 带默认值的普通 schema
  retryCount: z.number().default(0),
  // ② ReducedValue → 自定义 reducer，合并并发写入
  allSteps: new ReducedValue(
    z.array(z.string()).default(() => []),
    { reducer: (current, next) => [...current, next] },
  ),
  // ③ MessagesValue → 消息专用通道：追加 + 按 ID 去重
  messages: MessagesValue,
  // ④ UntrackedValue → 执行期临时值，不写进 checkpoint
  tempCache: new UntrackedValue(z.record(z.string(), z.unknown())),
});
```

> **版本说明**：`StateSchema` 是 `@langchain/langgraph` v1 线（1.4.x / 1.5.x）推荐的写法。`Annotation.Root` 至今仍可用（`import { Annotation } from "@langchain/langgraph"`），旧项目里的代码不用改，但新写的一律用 `StateSchema`。本文所有代码基于 `@langchain/langgraph@1.4.21` 实测通过。

### 2.2 四类通道，逐个说清

#### ① 普通 Zod schema → LastValue 通道

**行为**：后写覆盖先写。这就是"默认值"。

```ts
const State = new StateSchema({ step: z.string() });

// 节点 a 写 step:"A"，节点 b 写 step:"B" → 最终 step 是 "B"
```

⚠️ 但"覆盖"只在**顺序执行**时成立。两个节点在**同一超步并行**写 `step`，会直接抛错——详见第三节。

#### ② `ReducedValue` → 自定义 reducer 通道

**行为**：每次写入都过一遍你的 reducer，由你决定怎么合。

```ts
const State = new StateSchema({
  logs: new ReducedValue(
    z.array(z.string()).default(() => []),
    {
      // current 是当前累计值，next 是本次写入值
      reducer: (current: string[], next: string[]) => [...current, ...next],
    },
  ),
});
```

`ReducedValue` 的第二个参数还有两个可选字段：

| 字段 | 作用 |
|------|------|
| `reducer` | 必填。合并函数 `(current, next) => newValue` |
| `inputSchema` | 可选。声明"**节点写入时的类型**"，可以与存储类型不同 |
| `jsonSchemaExtra` | 可选。附加到导出 JSON Schema 的元信息 |

⭐ **`inputSchema` 是个很有用的细节**，它把"存储类型"和"写入类型"解耦了：

```ts
const State = new StateSchema({
  // 存储的是 string[]，但节点每次只写一段 string
  logs: new ReducedValue(
    z.array(z.string()).default(() => []),
    {
      inputSchema: z.string(),                          // ← 节点写 string
      reducer: (current, next) => [...current, next],   // ← 内部拼成 string[]
    },
  ),
});

// 节点这样写（干净，不用每次包一层数组）
const a = () => ({ logs: "from-a" });
```

⚠️ **但 `inputSchema` 有个副作用必须知道**：图的**入参校验**也按 `inputSchema` 走。上面的图调 `invoke` 时，`logs` 字段必须传**字符串**，传数组会直接报校验错误：

```ts
// ✅ 对：入参也是 inputSchema 的类型
await graph.invoke({ logs: "seed" });
// ❌ 错：Validation failed for field "logs": expected string, received array
await graph.invoke({ logs: [] });
```

如果不想要这个约束，就**别写 `inputSchema`**，让 reducer 收两个同类型的参数（节点写数组、reducer 也收数组）：

```ts
const State = new StateSchema({
  logs: new ReducedValue(
    z.array(z.string()).default(() => []),
    { reducer: (current: string[], next: string[]) => [...current, ...next] },
  ),
});
await graph.invoke({ logs: [] });   // ✅
```

#### ③ `MessagesValue` → 消息通道
**行为**：追加新消息；**如果新消息的 ID 和已有消息相同，就替换那条**（而不是追加一条重复的）。

```ts
import { MessagesValue, StateSchema } from "@langchain/langgraph";
import { HumanMessage, AIMessage } from "@langchain/core/messages";

const State = new StateSchema({ messages: MessagesValue });

// 节点 1 写入 id:"m1" 的消息，节点 2 又写入 id:"m1" 的消息
// → 最终只有一条 id:"m1"，内容是节点 2 那次写的
```

⭐ **为什么要用现成的，别自己写 reducer**：

1. **按 ID 去重**是消息场景的刚需。自己写 `reducer: (a, b) => [...a, ...b]` 会得到两条 ID 相同的重复消息，前端渲染两次、模型看到矛盾历史
2. **删除语义**——`MessagesValue` 还支持用 `REMOVE_ALL_MESSAGES` 之类的标记删除消息
3. **消息对象结构复杂**（`tool_calls`、`additional_kwargs`、多模态 content blocks），合并逻辑不该自己维护

自己实现一个"能追加但不去重"的 reducer 看起来省事，实际是在给未来埋雷。Python 版对应物是 `Annotated[list, add_messages]`，见 [[langgraph/modules/langgraph_state/Reducer]]。

#### ④ `UntrackedValue` → 不进 checkpoint 的临时通道

**行为**：执行期间和普通状态一样可用，但**创建检查点时被排除**，恢复后从空值/默认值重新开始。

```ts
const State = new StateSchema({
  dbConnection: new UntrackedValue<DatabaseConnection>(),   // 存不进去的连接对象
  tempCache: new UntrackedValue(z.record(z.string(), z.unknown())),
});
```

适合放：数据库连接、每次运行都该重建的缓存、很大的临时对象、运行时配置。

⚠️ **默认 `guard: true` 的含义**：同一超步里多个节点写这个通道会**抛错**（防止状态被意外替换）。要允许多次写并保留最后一个，声明 `{ guard: false }`。

⭐ **不要把需要跨中断/时间旅行保留的数据放进 `UntrackedValue`**——恢复时它是空的。

#### ⑤ 附：`Overwrite`——单次绕过 reducer

这不是一种通道类型，而是**一种写入方式**。当某个节点想"跳过 reducer、直接覆盖"时用它：

```ts
import { Overwrite } from "@langchain/langgraph";

const State = new StateSchema({
  errors: new ReducedValue(
    z.array(z.string()).default(() => []),
    { reducer: (current, next) => [...current, ...next] },
  ),
});

// 普通节点：追加
const appendNode = () => ({ errors: ["bad sql"] });
// 清空节点：用 Overwrite 绕过 reducer
const clearNode = () => ({ errors: new Overwrite([]) });
```

⭐ **为什么需要它**：带 reducer 的通道**无法通过写空数组来清空**——`[]` 会被 reducer 当成"追加零条"，原值原封不动。实测：

| 节点写法 | 实际结果 |
|---------|---------|
| `{ errors: [] }` | `["bad sql"]` ← **没清掉**（合并了零条） |
| `{ errors: new Overwrite([]) }` | `[]` ← 清掉了 |

### 2.3 ⭐ 类型推导：TS 泛型能力最好的落点

这是 TypeScript 版相对 Python 版最大的优势。`StateSchema` 实例上有三个**类型属性**：

| 属性 | 是什么 | 用法 |
|------|--------|------|
| `State` | **完整状态**的类型（所有字段都是必填、有确切类型） | `typeof MyState.State` |
| `Update` | **局部更新**的类型（所有字段可选，写入类型按通道而定） | `typeof MyState.Update` |
| `Node` | 节点函数的类型（入参 `State`，返回 `Update`） | `typeof MyState.Node` |

```ts
const State = new StateSchema({
  messages: MessagesValue,
  step: z.string(),
  count: z.number().default(0),
  logs: new ReducedValue(
    z.array(z.string()).default(() => []),
    { inputSchema: z.string(), reducer: (current, next) => [...current, next] },
  ),
});

// 完整状态
type MyState = typeof State.State;
// { messages: BaseMessage[]; step: string; count: number; logs: string[] }

// 局部更新：字段全部可选，且 logs 的写入类型是 string（inputSchema 的类型）
type MyUpdate = typeof State.Update;
// { messages?: ...; step?: string; count?: number; logs?: string }
```

⭐ **`State` 和 `Update` 的区别不是"随便定的"，它由通道类型机械推导出来**：

- `ReducedValue<Value, Input>` → `State` 里是 `Value`（存储类型），`Update` 里是 `Input`（写入类型）
- `UntrackedValue<Value>` → 两边都是 `Value`
- 普通 Zod schema → `State` 里是 Zod 的**输出类型**（`z.number().default(0)` → `number`），`Update` 里是 Zod 的**输入类型**

这解释了一个容易懵的点：`z.number().default(0)` 在 `State` 里是 `number`（不是 `number | undefined`），因为 Zod 的 default 在**解析后**一定有值。

**用它约束节点签名**：

```ts
// ✅ 节点入参自动是完整状态，返回值自动是局部更新
const increment: typeof State.Node = (state) => {
  const s: string = state.step;      // state.step 推断为 string
  const c: number = state.count;     // state.count 推断为 number
  return { count: state.count + 1 };  // 只写要改的字段
};

// ❌ 写错字段名，编译期就报错
const bad: typeof State.Node = (state) => {
  return { stepp: "typo" };   // TS 编译报错：'stepp' 不在 Update 里
};

// ❌ 给 logs 传数组，编译期就报错（因为 Update.logs 是 string）
const bad2: typeof State.Update = { logs: ["array"] };
```

⭐ **这就是"一套 schema 同时给运行时校验和静态类型"的含义**：

- **运行时**：Zod 校验图的入参、校验每个字段的形状
- **编译期**：`State` / `Update` 推导出的类型直接约束你的节点函数

写错字段名、类型不匹配，**在 `tsc` 阶段就暴露**，而不是等到线上跑图时才炸。对比 Python：Python 的 `TypedDict` 只是类型提示，**运行时不存在**，写错字段名要到真正执行到那一行才会抛 `KeyError`。Pydantic 能做运行时校验，但**运行时校验和静态类型是两条独立的路**（Pydantic v2 有 `TypeAdapter` 之类的桥接，但不如 Zod 一套 schema 顺滑）。

这个差异的完整展开见 [[typescript/03-生态连接-LangChain与LangGraph/6-Python与TypeScript双栈对照]]。

---

## 三、⭐ reducer 是并发正确性的关键

### 3.1 先说清楚真相：并行写同一通道会**抛错**

这是本篇最重要的一条，也是最容易被网上文章误导的一条。

⭐ **默认 reducer（LastValue）的真实行为分两种情况**：

| 场景 | 行为 |
|------|------|
| **顺序执行**（节点 a 之后才是节点 b），两者写同一通道 | 静默覆盖，后写的赢 |
| **并行执行**（a、b 在同一超步），两者写同一通道 | ⚠️ **直接抛 `InvalidUpdateError`**，不是静默丢数据 |

实测（`@langchain/langgraph@1.4.21`）：

```ts
import { StateGraph, START, END, StateSchema } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({ step: z.string() });

const graph = new StateGraph(State)
  .addNode("a", () => ({ step: "from-a" }))
  .addNode("b", () => ({ step: "from-b" }))     // 和 a 并行
  .addNode("join", (s) => ({ logs: `join saw ${s.step}` }))
  .addEdge(START, "a")
  .addEdge(START, "b")      // ← a、b 同时被激活
  .addEdge("a", "join")
  .addEdge("b", "join")
  .addEdge("join", END)
  .compile();

await graph.invoke({ step: "init" });
// InvalidUpdateError: Invalid update for channel "step" with values ["from-a","from-b"]:
// LastValue can only receive one value per step.
```

⚠️ **这条要记牢**：很多资料说"默认 reducer 是覆盖，并行写会丢数据"。在 **LangGraph.js v1** 里，同超步并发写 LastValue 通道是**硬报错**（`INVALID_CONCURRENT_GRAPH_UPDATE`）——这是好事，因为它**不会静默丢数据**，你一定会在测试里发现。真正的"静默丢数据"发生在**顺序执行**的场景：a 写了 `step:"A"`，b 写了 `step:"B"`，最终只剩 `"B"`，a 的贡献被无声吃掉。

### 3.2 那什么时候该给 reducer

**规则：只要一个通道可能被多个来源写入，就必须显式给 reducer。**

典型的"会被多写"场景：

- 数组类累积（收集日志、收集检索结果、收集各分支的产出）
- 计数器（并行分片各处理 N 条，要累加）
- 消息历史（永远用 `MessagesValue`，别自己写）
- 任何"扇出后汇总"的字段

### 3.3 完整例子：两个并行节点都往数组里写

```ts
import { StateGraph, START, END, StateSchema, ReducedValue } from "@langchain/langgraph";
import { z } from "zod";

// ✅ 正确：给 reducer，声明"节点写一段 string"
const State = new StateSchema({
  findings: new ReducedValue(
    z.array(z.string()).default(() => []),
    {
      inputSchema: z.string(),
      reducer: (current, next) => [...current, next],
    },
  ),
});

const graph = new StateGraph(State)
  .addNode("querySourceA", () => ({ findings: "A 的结果" }))
  .addNode("querySourceB", () => ({ findings: "B 的结果" }))
  .addNode("merge", (state) => {
    // state.findings 是 string[]，已经是两路合并后的完整结果
    return { findings: `合并 ${state.findings.length} 条` };
  })
  .addEdge(START, "querySourceA")
  .addEdge(START, "querySourceB")   // 并行
  .addEdge("querySourceA", "merge")
  .addEdge("querySourceB", "merge")
  .addEdge("merge", END)
  .compile();

const result = await graph.invoke({ findings: "起始" });
console.log(result.findings);
// ["起始", "A 的结果", "B 的结果", "合并 3 条"]
```

⭐ 三个要点：

1. **节点写 `string` 不是 `string[]`**——因为 `inputSchema: z.string()`，类型上就是 string，写起来干净
2. **合并顺序不保证**——并行节点谁先完成谁先进数组，别依赖顺序
3. **入参也要符合 `inputSchema`**（上面传的是 `"起始"` 这个字符串）

### 3.4 ❌ 对照：不给 reducer 会怎样

```ts
// ❌ 错误：findings 是普通 schema（LastValue 通道）
const State = new StateSchema({
  findings: z.array(z.string()).default(() => []),
});

// 顺序执行时：后者覆盖前者，前者的数据静默消失
// 并行执行时：抛 InvalidUpdateError
```

⚠️ **两种失败模式都很难受**：顺序场景你连报错都拿不到，只能靠人肉 review 代码发现数据不见了；并行场景虽然报错，但错误信息只说"LastValue can only receive one value per step"，不告诉你是哪个通道、哪两个节点。

**对策**：
- 凡是"可能被多写"的字段，一律显式给 reducer
- 给 reducer 时加一句注释说明"这里会并发写"，让后来人知道为什么不能改成普通 schema

---

## 四、节点、边、条件边、编译执行

### 4.1 五分钟跑通一张图

```ts
import { StateGraph, START, END, StateSchema, MessagesValue } from "@langchain/langgraph";
import { HumanMessage, AIMessage } from "@langchain/core/messages";
import { z } from "zod";

const State = new StateSchema({
  messages: MessagesValue,
  step: z.string(),
});

const graph = new StateGraph(State)
  .addNode("greet", () => ({
    step: "greeted",
    messages: [new AIMessage({ id: "g1", content: "你好！" })],
  }))
  .addNode("bye", (state) => {
    // state.step 推断为 string
    return { messages: [new AIMessage({ id: "b1", content: `再见，${state.step}` })] };
  })
  .addEdge(START, "greet")      // 入口
  .addEdge("greet", "bye")       // 普通边：greet 跑完去 bye
  .addEdge("bye", END)           // 终点
  .compile();                    // 必须 compile 才能跑

const result = await graph.invoke({
  messages: [new HumanMessage("嗨")],
  step: "init",
});
console.log(result.messages.map((m) => m.content));
// ["嗨", "你好！", "再见，greeted"]
```

### 4.2 三种边

| 边类型 | 写法 | 语义 |
|--------|------|------|
| **普通边** | `addEdge("a", "b")` | a 跑完必去 b |
| **入口 / 终点** | `addEdge(START, "a")` / `addEdge("a", END)` | 图从哪里开始、在哪里结束 |
| **条件边** | `addConditionalEdges("a", router, pathMap?)` | 由 router 函数返回的**字符串**决定去哪 |

`START` 和 `END` 是 `@langchain/langgraph` 导出的两个常量：

```ts
import { START, END } from "@langchain/langgraph";
```

⚠️ **v1 里的写法变化**：老代码常见的 `setEntryPoint("a")` / `setFinishPoint("b")` 在 v1 里**已标记 `@deprecated`**（虽然还留着能用），新代码一律用 `addEdge(START, "a")` / `addEdge("b", END)`。

### 4.3 条件边：让流程分叉

条件边的 router 函数接收当前状态，返回**目标节点名**（或节点名数组）：

```ts
import { StateGraph, StateSchema, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  question: z.string(),
  answer: z.string().default(""),
});

const router = (state: typeof State.State) => {
  if (state.question.includes("紧急")) return "escalate";
  if (state.question.length > 50) return "detail";
  return "quick";
};

const graph = new StateGraph(State)
  .addNode("quick", () => ({ answer: "标准答复" }))
  .addNode("detail", () => ({ answer: "详细答复" }))
  .addNode("escalate", () => ({ answer: "已转人工" }))
  .addEdge(START, "quick")     // 入口先走 quick
  .addConditionalEdges("quick", router, {
    escalate: "escalate",      // 映射表：router 返回什么 → 去哪
    detail: "detail",
    quick: "quick",
  })
  .addEdge("quick", END)
  .addEdge("detail", END)
  .addEdge("escalate", END)
  .compile();
```

⭐ **第三个参数 `pathMap` 是可选的映射表**，作用是"router 返回的逻辑名 → 实际节点名"。它的价值是**让 router 的返回值和节点名解耦**：router 返回 `"escalate"` 这个**业务语义**，映射表决定它对应哪个节点。改节点名时不用动 router。

不传 `pathMap` 时，router 返回的字符串会被**直接当作节点名**。

⚠️ **同一节点不要混用普通边和条件边**：如果 `"quick"` 既被 `addEdge("quick", END)` 连到 END，又被 `addConditionalEdges("quick", router)` 连到别处，两条路径都会执行，图会跑飞。

### 4.4 运行时上下文：`config.context`

有些参数**不属于状态**——比如"这次用哪个模型"、"当前用户是谁"。它们应该走**运行时上下文**，而不是塞进状态（状态会被 checkpoint 持久化，每轮都存一遍）。

```ts
import { StateGraph, StateSchema, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({ out: z.string() });
const ContextSchema = z.object({
  model: z.enum(["fast", "accurate"]),
});

const graph = new StateGraph(State, ContextSchema)   // ← 第二个参数是 context schema
  .addNode("run", (state, config) => {
    // config.context 就是调用时传进来的上下文
    return { out: `用 ${config.context?.model} 跑完了` };
  })
  .addEdge(START, "run")
  .addEdge("run", END)
  .compile();

await graph.invoke({ out: "" }, { context: { model: "accurate" } });
// { out: "用 accurate 跑完了" }
```

⚠️ **v1 的变化点**：以前这类参数习惯塞进 `config.configurable`，v1 起改用**有类型的 `config.context`**——声明了 `ContextSchema`（Zod）就有运行时校验和类型提示。`config.configurable` 现在主要留给框架自己用的键（比如 `thread_id`），自定义参数走 `context`。Python v1 的对应物是 `context_schema` + `Runtime.context`，见第 6 页。

---

## 五、多 schema：把内部状态和对外接口分开

默认情况下，图的**入参、状态、出参是同一个 schema**。这意味着你必须把内部字段也暴露给调用方。

`{ state, input, output }` 让你把三者拆开：

```ts
import { StateGraph, StateSchema, START, END } from "@langchain/langgraph";
import { z } from "zod";

// 对外输入：调用方只需要给这个
const InputState = new StateSchema({ userInput: z.string() });
// 对外输出：调用方只会收到这个
const OutputState = new StateSchema({ graphOutput: z.string() });
// 内部状态：节点之间随便传
const OverallState = new StateSchema({
  foo: z.string(),
  userInput: z.string(),
  graphOutput: z.string(),
});

const graph = new StateGraph({
  state: OverallState,   // 内部完整状态
  input: InputState,     // 对外入参
  output: OutputState,   // 对外出参
})
  .addNode("node1", (state) => ({ foo: state.userInput + " name" }))
  .addNode("node2", (state) => ({ graphOutput: state.foo + " is" }))
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", END)
  .compile();

const result = await graph.invoke({ userInput: "My" });
console.log(result);
// { graphOutput: "My name is" }   ← 只有 output schema 的字段
```

⭐ **实际价值**：

1. **接口契约清晰**——外部看得到的就是 `InputState` / `OutputState`，内部重构不会破坏调用方
2. **防止字段泄漏**——`userInput` 这种内部中间变量不会被返回给前端
3. **私有通道**——节点可以通过 `addNode(name, fn, { input: PrivateState })` 声明自己读一个额外的"私有通道"，这个通道不在 `state` 里声明也能注册

⚠️ **注意**：`input` / `output` 只约束 `invoke` 的入参与返回值，**不会自动从 `stream` 里隐藏内部通道**。想控制流式输出里出现什么，用 `streamMode: "updates"`（只发各节点实际产生的增量），或 `outputKeys`。

---

## 六、⭐ `Command`：同时更新状态和跳转

有时候一个节点要做的不是"改状态然后顺着边走"，而是"改状态**并且**决定下一步去哪"。这时用 `Command`。

```ts
import { StateGraph, StateSchema, START, END, Command } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  step: z.string(),
  count: z.number().default(0),
});

const graph = new StateGraph(State)
  .addNode("router", (state) => {
    // 一次返回里既改状态又指定去向
    return new Command({
      update: { count: state.count + 1 },
      goto: state.count >= 2 ? "done" : "loop",
    });
  }, { ends: ["loop", "done"] })   // ← 声明可能的目标，否则编译报 UnreachableNodeError
  .addNode("loop", () => ({ step: "looping" }))
  .addNode("done", () => ({ step: "finished" }))
  .addEdge(START, "router")
  .addEdge("loop", "router")       // 回到 router，形成循环
  .addEdge("done", END)
  .compile();

const result = await graph.invoke({ step: "init" });
console.log(result);
// { step: "finished", count: 3 }
```

⭐ **`Command` 的四个字段**：

| 字段 | 作用 |
|------|------|
| `update` | 要写入状态的更新（等价于节点正常返回的对象） |
| `goto` | 下一步去哪个节点（也可以是 `Send` 对象或它们的数组） |
| `graph` | 发给哪个图。`Command.PARENT` 表示发给父图（子图场景） |
| `resume` | 恢复中断时提供的值（配合 `interrupt()`） |

⭐ **`ends` 选项是必填的**：节点返回 `Command` 时，框架**无法静态推断**它可能跳到哪，所以必须显式声明：

```ts
const router = async (state) => {
  return new Command({ goto: "loop", update: { count: state.count + 1 } });
};

graph.addNode("router", router, { ends: ["loop", "done"] });
```

不写会得到 `UnreachableNodeError: Node "loop" is not reachable`——错误信息会提示你加上 `ends`。

⚠️ **同一节点不要同时用 `addEdge` 和 `Command.goto`**——两条路径都会执行。

### 什么时候用 `Command` 而不是条件边

| 场景 | 选择 |
|------|------|
| 只路由，不改状态 | 条件边（更清晰） |
| 只改状态，去向由边决定 | 普通返回对象 |
| **既改状态又自己决定去向** | `Command` |

---

## 七、⭐ `Send`：动态扇出（map-reduce）

`Send` 让你在**运行时**决定"要启动几个节点任务，每个任务带什么输入"。这是条件边做不到的——条件边的 router 返回节点名，扇出数量是写死的；`Send` 返回的是**一串任务**。

```ts
import { StateGraph, StateSchema, ReducedValue, Send, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  subjects: z.array(z.string()),
  jokes: new ReducedValue(
    z.array(z.string()).default(() => []),
    { inputSchema: z.string(), reducer: (current, next) => [...current, next] },
  ),
});

const graph = new StateGraph(State)
  .addNode("start", () => ({}), { ends: ["generateJoke"] })
  .addEdge(START, "start")
  .addConditionalEdges("start", (state) =>
    // 每个 subject 生成一个 Send 任务，扇出数量运行时才知道
    state.subjects.map((subject) => new Send("generateJoke", { subject })),
  )
  .addNode("generateJoke", (state: { subject: string }) => ({
    jokes: `关于 ${state.subject} 的笑话`,
  }))
  .addEdge("generateJoke", END)
  .compile();

const result = await graph.invoke({ subjects: ["猫", "狗"], jokes: "" });
console.log(result.jokes);
// ["", "关于 猫 的笑话", "关于 狗 的笑话"]
```

⭐ **三个关键点**：

1. **`new Send(目标节点名, 该任务的输入)`**——输入可以**只包含目标节点需要的字段**，不必是完整状态。上例中 `generateJoke` 收到的是 `{ subject }` 而不是整个 state
2. **每个 `Send` 是一个独立任务**——它们并行跑，各自的更新在超步边界按 reducer 合并
3. **`ends` 还是要声明**——`start` 节点的潜在目标只有 `generateJoke`

⚠️ **初值也要符合 `inputSchema`**：上面 `jokes: ""` 传的是字符串（因为 `inputSchema: z.string()`），不是数组。

⭐ **这就是经典的 map-reduce 模式**：map 阶段用 `Send` 并行扇出，reduce 阶段用一个节点（或 reducer 本身）汇总。

---

## 八、⭐ `ToolNode` + `toolsCondition`：把工具接进图

`@langchain/langgraph/prebuilt` 提供两个现成件：

- **`ToolNode`**：一个节点，执行最后一条 AI 消息里请求的所有工具调用（**并行执行**），产出 `ToolMessage` 列表
- **`toolsCondition`**：一个条件边路由函数——最后一条消息是带 `tool_calls` 的 AI 消息就返回 `"tools"`，否则返回 `END`

```ts
import { StateGraph, StateSchema, MessagesValue, START, END } from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { HumanMessage, AIMessage } from "@langchain/core/messages";
import { z } from "zod";

const getWeather = tool(
  async ({ city }) => `晴，${city}，25℃`,
  {
    name: "get_weather",
    description: "查询指定城市的天气",
    schema: z.object({ city: z.string().describe("城市名") }),
  },
);

const State = new StateSchema({ messages: MessagesValue });

// 模拟模型：第一次要调工具，第二次给出最终答案
let modelRuns = 0;
const modelNode = () => {
  modelRuns++;
  const first = modelRuns === 1;
  return {
    messages: [new AIMessage({
      id: first ? "m1" : "m2",
      content: first ? "" : "北京现在晴，25℃",
      tool_calls: first
        ? [{ name: "get_weather", args: { city: "北京" }, id: "c1", type: "tool_call" }]
        : [],
    })],
  };
};

const graph = new StateGraph(State)
  .addNode("model", modelNode)
  .addNode("tools", new ToolNode([getWeather]))
  .addEdge(START, "model")
  // ⭐ toolsCondition 自己会返回 END，所以这里不要再 addEdge("model", END)
  .addConditionalEdges("model", toolsCondition, ["tools", END])
  .addEdge("tools", "model")       // 工具结果回到模型，让模型决定下一步
  .compile();

const result = await graph.invoke({ messages: [new HumanMessage("北京天气？")] });
// 4 条消息：human / ai(带 tool_call) / tool("晴，北京，25℃") / ai("北京现在晴，25℃")
// model 节点共执行 2 次
```

⚠️ **名字注意**：JS 版叫 **`toolsCondition`**（驼峰），Python 版叫 `tools_condition`（下划线）。这个差异在第 6 页会专门讲。

⭐ **`ToolNode` 的两个细节**：

1. **工具调用默认并行执行**——一条 AI 消息里请求 3 个工具，3 个同时跑
2. **错误处理可配**——`new ToolNode([...], { handleToolErrors: true })` 让工具异常变成一条错误 `ToolMessage` 而不是让整个图崩掉

### 什么时候用图，什么时候用 `createAgent`

| 需求 | 选择 |
|------|------|
| 标准 ReAct 循环（模型→工具→模型，直到不需要工具） | `createAgent` |
| 想在工具之外插入自定义逻辑（检索、审核、限流） | 图 + `ToolNode` |
| 需要在**工具执行前**等人审批 | 图 + `interrupt()`，或 `createAgent` + `humanInTheLoopMiddleware`（见第 5 页） |
| 需要工具调用**并发扇出**并自定义归并 | 图 + `Send` |

---

## 陷阱

1. **⚠️ 以为默认 reducer 是"静默覆盖"，并行写不会报错**——**LangGraph.js v1 里同一超步并发写 LastValue 通道会抛 `InvalidUpdateError`**（`LastValue can only receive one value per step` / `INVALID_CONCURRENT_GRAPH_UPDATE`）；**顺序执行**才是静默覆盖，那才是真正会悄悄丢数据的场景；
2. **⚠️ 忘了给会被多写的字段加 reducer**——凡是"扇出后汇总"的数组/计数字段都要显式 `ReducedValue`，否则顺序场景下前一个节点的贡献被无声吃掉；
3. **⚠️ `ReducedValue` 加了 `inputSchema` 后，入参校验也跟着变**——`inputSchema: z.string()` 意味着 `invoke({ logs: [] })` 会抛校验错误，必须传字符串；
4. **⚠️ 把 `Annotation.Root` 和 `StateSchema` 混着写**——两者都能用（混用也能编译通过），但同一项目里两种风格并存会让"状态定义在哪"变得难找，新代码一律 `StateSchema`；
5. **⚠️ 还在用 `setEntryPoint` / `setFinishPoint`**——v1 里已 `@deprecated`，改用 `addEdge(START, "x")` / `addEdge("x", END)`；
6. **⚠️ 把运行时上下文塞进 `config.configurable`**——v1 的正确做法是 `new StateGraph(State, ContextSchema)` + `config.context`，有类型、有校验；`configurable` 留给框架自用的键（如 `thread_id`）；
7. **⚠️ 忘记 `ends` 选项**——返回 `Command` 或用 `Send` 的节点必须声明潜在目标，否则编译期报 `UnreachableNodeError`；
8. **⚠️ 同一节点混用普通边和条件边**——两条路径都会执行，图会跑飞；一个节点只选一种路由机制；
9. **⚠️ 以为 `subgraphs: true` 是可选的**——图里有子图时，不加这个选项，子图的输出**根本不会出现在流里**（详见第 5 页）；
10. **⚠️ 依赖并行节点的完成顺序**——并行节点谁先完成谁先进数组，合并顺序**不保证**；要顺序稳定就别并行；
11. **⚠️ 用 `UntrackedValue` 存需要跨中断保留的数据**——它不进 checkpoint，恢复后是空的；
12. **⚠️ 以为给通道加了 reducer 就能用空数组清空它**——`{ errors: [] }` 是"追加零条"，原值不变；**要清空得用 `new Overwrite([])`**；
13. **⚠️ 忘了 `compile()`**——`StateGraph` 只是构建器，`.compile()` 之后才是可执行对象。

## 小结

- **`createAgent` vs 状态图**：需要显式节点/边、需要中断恢复、需要确定性与自主性混合，三条都满足才上图
- **`StateSchema`（Zod）** 是 v1 推荐的��态定义方式，`Annotation.Root` 是遗留别名
- **四类通道**：普通 Zod schema → LastValue（覆盖）；`ReducedValue` → 自定义 reducer；`MessagesValue` → 追加 + 按 ID 去重；`UntrackedValue` → 不进 checkpoint。另有 `Overwrite` 可单次绕过 reducer（用来清空累积型通道）
- ⭐ **类型推导**：`typeof State.State`（完整状态）、`typeof State.Update`（局部更新，写入类型按通道而定）、`typeof State.Node`（节点函数）；这让 Zod 一套 schema 同时提供运行时校验与编译期检查
- ⭐ **并发正确性**：同超步并发写 LastValue 通道**抛 `InvalidUpdateError`**；顺序写才是静默覆盖。会被多写的字段一律显式给 reducer
- **多 schema** `{ state, input, output }` 把内部状态与对外契约分开
- **`Command`** 同时更新状态和跳转（需要 `ends` 声明目标）；**`Send`** 运行时动态扇出，map-reduce 的核心
- **`ToolNode` + `toolsCondition`** 把工具接进图；JS 名是驼峰 `toolsCondition`，Python 是下划线 `tools_condition`

## 练习

1. 写一个 `StateSchema`，包含：一个 `MessagesValue` 消息通道、一个 `ReducedValue` 累加通道（计数器）、一个普通 string 字段、一个 `UntrackedValue` 缓存字段。然后用 `typeof` 导出 `State` 和 `Update` 两个类型别名。
2. 搭一张图：入口有两条并行分支各自查"库存"和"价格"，汇合节点生成一句文案。**故意先不给累加通道加 reducer**，跑一次看报错长什么样；然后加上 `ReducedValue` 再跑一次，对比两次的差异。
3. 把上一题的累加通道改成 `inputSchema: z.string()` 版本（节点每次写一段字符串）。运行时会发现 `invoke` 的入参也要改——把这个报错信息原样抄下来，并写一句话说明"为什么入参校验也跟着 inputSchema 走"。
4. 用 `Command` 实现一个"最多重试 3 次"的重试节点：前两次失败回到重试节点，第 3 次失败走 `giveUp` 分支。补全 `ends` 声明。
5. 用 `Send` 实现一个"给 N 个文档各生成一个摘要"的任务：N 由入参决定，摘要列表用 `ReducedValue` 收集。
6. 把 `ToolNode` 那张工具图改造一下：加一个 `guard` 节点，在工具执行前检查"城市是否在白名单里"，不在白名单就跳过工具直接回答。
7. ⭐ 用 `new StateGraph({ state, input, output })` 改造第 6 题的图：让调用方只能传 `{ question }`，只能收到 `{ answer }`，而内部自由传递中间结果。
8. ⭐ 写一个 `typeof State.Node` 标注的节点，故意写错一个字段名，用 `tsc --noEmit` 验证编译器能抓到；再故意给 `ReducedValue` 通道传错写入类型，验证也能被抓到。
9. ⭐ 验证"空数组清空"这条：建一个带 reducer 的数组通道，一个节点追加 `["x"]`，下一个节点写 `[]`，观察值没被清掉；再改用 `new Overwrite([])`，确认这次清掉了。

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：[[typescript/03-生态连接-LangChain与LangGraph/3-从TypeScript到LangChain-js实战迁移]] · 下一篇：[[typescript/03-生态连接-LangChain与LangGraph/5-LangGraph-js持久化与人工介入]]
