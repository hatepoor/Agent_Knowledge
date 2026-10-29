---
tags: [typescript, langgraph, 持久化, checkpoint, 人工介入, HITL]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# LangGraph.js 持久化与人工介入

> **什么时候该读**：你的图跑起来了，但每次请求都是"从零开始"——用户刷新页面，前面的对话没了；或者你需要"图跑到一半停下来，等人工审批再继续"。这两个问题的解法都在这一页：**checkpointer**（持久化）和 `interrupt()`（人工介入）。
> **阅读门槛**：读完 [[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]]（会写状态、节点、边，理解 `StateSchema` 和 reducer）。本篇会大量用 `await` 和异步迭代器，如果对 `Promise` / `for await` 不熟，先看 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]]。

## 学习目标

读完本篇，你应当能够：

1. 说清 checkpointer 是什么、它解决哪四类问题（会话续跑 / 中断恢复 / 时间旅行 / 审计）
2. 在三档后端（内存 / SQLite / PostgreSQL）之间做出正确选型，并说出为什么生产不能用内存版
3. 用 `thread_id` 管理多条会话线程，知道 `getState` / `getStateHistory` / `updateState` 各能做什么
4. ⭐ 跑通一条完整的 HITL 链路：`interrupt()` → 抛出 `GraphInterrupt` → 状态落盘 → `Command({ resume })` → 继续执行
5. 判断"该用 `createAgent` + `humanInTheLoopMiddleware`"还是"该用图 + 手写 `interrupt()`"
6. 选对 `streamMode`，并知道有子图时为什么**必须**加 `subgraphs: true`
7. 说出"中断后进程会挂起"这个误解错在哪

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]] | 状态、节点、边的写法是本篇的基础 |
| [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] | `for await...of` 异步迭代器、`Promise` 三态 |
| [[langgraph/modules/langgraph_checkpoint/概述]] | Python 版的持久化机制，本篇讲 JS 对应物 |
| [[langgraph/modules/langgraph_human_in_loop/interrupt机制]] | Python 版的 `interrupt()` 机制，本篇讲 JS 对应物 |

---

## 一、⭐ checkpointer 是什么

### 1.1 一句话

**checkpointer（检查点保存器）是一个在每个超步边界把图状态存下来的组件。** 存下来的东西叫**检查点（checkpoint）**。

有了检查点，图就具备了"可暂停、可恢复、可回溯"的能力。Python 版的详细机制见 [[langgraph/modules/langgraph_checkpoint/概述]]。

### 1.2 它解决四类问题

| 问题 | 没有 checkpointer | 有 checkpointer |
|------|----------------|---------------|
| **会话续跑** | 每次请求都从空状态开始，多轮对话记不住上文 | 同一 `thread_id` 的调用能接着上次的状态继续 |
| **中断恢复** | 图执行到一半人没来，状态丢了，一切从头再来 | 状态在中断点落盘，人回来后从断点继续 |
| **时间旅行** | 无法回到历史某一步 | `getStateHistory()` 列出全部历史快照，可回到任意一步重跑 |
| **审计 / 可观测** | 不知道过去发生了什么 | 每个状态快照都是一条可追溯的记录 |

⭐ **一句话抓住重点**：checkpointer 存的不只是"最终结果"，而是**每一步的完整状态快照**。这正是它能支撑"中断恢复"和"时间旅行"的原因。

### 1.3 怎么用

```ts
import { StateGraph, START, END, StateSchema, MemorySaver } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({ step: z.string() });

const graph = new StateGraph(State)
  .addNode("a", () => ({ step: "A" }))
  .addEdge(START, "a")
  .addEdge("a", END)
  .compile({ checkpointer: new MemorySaver() });   // ← 这里挂上 checkpointer

// ⚠️ 挂上 checkpointer 后，invoke 必须传 thread_id
const config = { configurable: { thread_id: "user-123" } };
await graph.invoke({ step: "" }, config);
```

⭐ **`thread_id` 在 `config.configurable` 里**（注意是 `configurable`，不是 `config.context`——后者是给业务运行时上下文用的，见第 4 页）。

⚠️ **漏了 `thread_id` 会直接抛错**（实测报错原文）：

```
Failed to put checkpoint. The passed RunnableConfig is missing a required "thread_id"
field in its "configurable" property. When using a checkpointer, you must pass a
"thread_id" so the checkpointer knows which conversation thread to persist state for.
```

---

## 二、⭐ 三档后端对比

### 2.1 选型表

| 包 | 导出的类 | 后端 | 进程重启后 | 适用场景 |
|------|---------|------|-----------|---------|
| `@langchain/langgraph-checkpoint` | `MemorySaver` | 内存 | ❌ 全丢 | 开发、调试、单元测试 |
| `@langchain/langgraph-checkpoint-sqlite` | `SqliteSaver` | SQLite 文件 | ✅ 保留 | 单机服务、小规模生产 |
| `@langchain/langgraph-checkpoint-postgres` | `PostgresSaver` | PostgreSQL | ✅ 保留 | 生产多实例、高并发 |

> **版本说明**：以上基于 `@langchain/langgraph@1.4.21` 与对应 checkpoint 包 v1.x 实测。另有社区/官方维护的 `@langchain/langgraph-checkpoint-mongodb`（`MongoDBSaver`）与 `@langchain/langgraph-checkpoint-redis`（`RedisSaver` / `ShallowRedisSaver`），按需选用。

⭐ **`MemorySaver` 其实不用单独装**：`@langchain/langgraph` 本身就把 `@langchain/langgraph-checkpoint` 作为依赖带进来了，直接 `import { MemorySaver } from "@langchain/langgraph"` 即可。**SQLite 和 PostgreSQL 才需要额外安装**：

```bash
# 内存版：无需额外安装
npm install @langchain/langgraph

# SQLite 版
npm install @langchain/langgraph-checkpoint-sqlite

# PostgreSQL 版
npm install @langchain/langgraph-checkpoint-postgres
```

### 2.2 内存版

```ts
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();
```

⚠️ **它是 `Map` 一样的内存结构**——进程一退出，所有会话历史全部消失。所以：

- ✅ 开发调试：快速迭代，不用起数据库
- ✅ 单元测试：每个测试用例 new 一个，天然隔离
- ❌ **生产环境**：重启即丢，用户会话全没了

### 2.3 SQLite 版

```ts
import { SqliteSaver } from "@langchain/langgraph-checkpoint-sqlite";

// 传文件路径；表结构由 LangGraph 自动创建
const checkpointer = SqliteSaver.fromConnString("./checkpoints.db");

const graph = builder.compile({ checkpointer });
```

⭐ **SQLite 版是"单机生产"的性价比之选**：零运维（就是个文件）、性能够用、进程重启不丢数据。

⚠️ **但它是单机的**：文件锁决定了**同一个文件不能被多个进程同时写**。多实例部署（比如 3 个 Node 进程做负载均衡）会出问题——要么加文件锁串行化，要么换 Postgres。

### 2.4 PostgreSQL 版

```ts
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const checkpointer = PostgresSaver.fromConnString(
  process.env.DATABASE_URL!,
  { schema: "public" },        // 可选，默认 "public"
);

// ⚠️ 必须手动调用 setup() 建表（首次运行一次即可）
await checkpointer.setup();

// builder 是前面搭好的 StateGraph 构建器
const graph = builder.compile({ checkpointer });
```

⭐ **Postgres 是生产多实例的唯一正确答案**：所有实例连同一个数据库，任意实例都能接管任意会话。

⚠️ **`setup()` 必须自己调**——它负责建表和跑迁移。这是 `PostgresSaver` 和 `SqliteSaver` 的重要差别（SQLite 版在内部调用时会自动 setup）。

### 2.5 ⚠️ 两条硬性生产约束

**约束一：生产不能用内存版。**

这不是"性能问题"，是"数据直接没了"。进程重启、容器滚动更新、`pm2` reload——任何一次都会让所有用户的会话历史归零。

**约束二：checkpointer 的写入在热路径上。**

每执行一个超步就要写一次检查点。这意味着**每个节点执行都有 I/O**。

⭐ **推论**：如果你的服务是**无状态的短请求**（用户问一句、模型答一句、结束），挂 checkpointer 是**纯开销**——你花了 I/O 成本，却从来没读过任何历史。这种场景**不要挂**。

判断标准很简单：

| 场景 | 挂 checkpointer 吗 |
|------|----------------|
| 单轮问答（无状态短请求） | ❌ 别挂，纯浪费 |
| 多轮对话 | ✅ 必须挂 |
| 有人工审批环节 | ✅ 必须挂（不挂就没法恢复） |
| 长时间运行的任务（几分钟以上） | ✅ 必须挂（崩了能续） |
| 需要审计"过去发生了什么" | ✅ 必须挂 |

---

## 三、`thread_id` 与会话管理

### 3.1 `thread_id` 是什么

**`thread_id` 是持久化的游标**，不是业务分类字段。框架靠它回答两个问题：

- "这次执行该往哪条历史里写？"
- "这次执行该从哪条历史的哪个点继续？"

⭐ **一个 `thread_id` = 一条独立的会话线**。同 ID 续跑，不同 ID 隔离。

### 3.2 续跑与隔离（实测）

```ts
import { StateGraph, START, END, StateSchema, MemorySaver, MessagesValue } from "@langchain/langgraph";
import { HumanMessage, AIMessage } from "@langchain/core/messages";
import { z } from "zod";

const State = new StateSchema({ messages: MessagesValue });

const graph = new StateGraph(State)
  .addNode("chat", (state) => ({
    // 用固定 id：每次回复覆盖上一条（MessagesValue 按 ID 去重）
    messages: [new AIMessage({ id: "reply", content: `收到 ${state.messages.length} 条` })],
  }))
  .addEdge(START, "chat")
  .addEdge("chat", END)
  .compile({ checkpointer: new MemorySaver() });

// 同一条线程
const cfg = { configurable: { thread_id: "conv-1" } };
const r1 = await graph.invoke({ messages: [new HumanMessage("我叫小明")] }, cfg);
console.log(r1.messages.length);        // 2

const r2 = await graph.invoke({ messages: [new HumanMessage("我叫什么名字？")] }, cfg);
console.log(r2.messages.length);        // 3  ← 上一轮的消息还在
console.log(r2.messages.map((m) => String(m.content)));
// ["我叫小明", "收到 3 条", "我叫什么名字？"]

// 不同线程：完全隔离
const cfg2 = { configurable: { thread_id: "conv-2" } };
const r3 = await graph.invoke({ messages: [new HumanMessage("新会话")] }, cfg2);
console.log(r3.messages.length);        // 2  ← 只有本轮的消息
```

⭐ 注意一个细节：第二轮的结果里，那条 `"收到 3 条"` 是**本轮**节点产出的（它看到了 3 条历史消息），而**上一轮**的 `"收到 1 条"` 因为 ID 相同被覆盖了——这正是 `MessagesValue` 按 ID 去重的效果。**多轮记忆的工作方式就是：历史消息留着，模型的每次回复覆盖上一条回复。**

### 3.3 ⚠️ `thread_id` 混用是隐蔽的 bug

**典型事故**：

```ts
// 前端每次请求都用"当前时间戳"当 thread_id
const config = { configurable: { thread_id: `chat-${Date.now()}` } };
```

结果：每轮都是一条新线程，上下文全丢。**而且不会报任何错**——每条线程都"正常"地跑完了。

**正确做法**：thread_id 要**稳定标识一条会话**：

- 服务端会话 ID（推荐）
- 数据库里的 conversation_id
- 前端在会话创建时生成一个 UUID，**整个会话期间保持不变**

⭐ **判断方法**：如果你换了 thread_id 之后"历史不见了"，那基本就是这个原因。

### 3.4 查看与修改状态

```ts
// ⚠️ 下面是紧接上一段图的续写，graph / config 沿用上文
const snapshot = await graph.getState(config);
console.log(snapshot.values);   // 当前状态值
console.log(snapshot.next);     // 下一个要执行的节点（中断时停在这里）

// 查看全部历史快照（时间旅行的基础）
for await (const s of graph.getStateHistory(config)) {
  console.log(s.values, s.next);
}

// 手动修改状态（可指定"假装是哪个节点写的"）
await graph.updateState(config, { step: "手动改的" }, "someNode");
```

⭐ **`getState` 在排查问题时极其有用**：图卡住了、状态不对、想看"现在到底停在哪一步"，先调它。

⭐ **`updateState` 是"人工干预状态"的入口**：比如运营发现某个会话状态错了，可以直接改。但要清楚——**改状态不会自动重跑图**，你还得再 `invoke` 一次。

---

## 四、⭐ 人工介入（HITL）完整链路

### 4.1 什么是人工介入

**HITL（Human-in-the-Loop，人在回路）**：图执行到某一步暂停，把当前情况交给人看，人给出决策后从断点继续。

典型场景：

- 大额转账要财务审批
- 敏感操作（删数据、发邮件）要人工确认
- AI 生成的对外文案要编辑后再发
- 缺少关键信息，要问用户补全

### 4.2 机制：`interrupt()` 抛异常，状态落盘

⭐ **核心机制一句话**：节点里调用 `interrupt()`，它会**抛出一个特殊异常 `GraphInterrupt`**，导致整个图暂停；暂停瞬间的状态被 checkpointer 存下来；等外部用 `Command({ resume })` 恢复时，**`interrupt()` 这一次调用返回恢复值，节点从 `interrupt()` 那一行继续往下执行**。

注意"从 `interrupt()` 那一行继续"这个细节——它意味着**节点里 `interrupt()` 之前的代码会在恢复时重新执行一次**。这是后面陷阱清单里的重点。

### 4.3 完整链路（实测可跑）

```ts
import {
  StateGraph, START, END, StateSchema,
  MemorySaver, Command, interrupt, isInterrupted,
} from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  action: z.string(),
  approved: z.boolean().default(false),
  result: z.string().default(""),
});

const graph = new StateGraph(State)
  .addNode("approval", (state) => {
    // ⭐ 这行会抛 GraphInterrupt，图在这里停住
    const decision = interrupt({
      question: "是否批准以下操作？",
      detail: state.action,
      options: ["yes", "no"],
    });
    // 恢复后，decision 就是 Command({ resume }) 里传的值
    // 用 Command 同时更新状态并决定往哪走
    return new Command({
      update: { approved: decision === "yes" },
      goto: decision === "yes" ? "proceed" : "cancel",
    });
  }, { ends: ["proceed", "cancel"] })   // ← 声明可能的目标，否则编译报 UnreachableNodeError
  .addNode("proceed", () => ({ result: "已批准并执行" }))
  .addNode("cancel", () => ({ result: "已拒绝" }))
  .addEdge(START, "approval")
  .addEdge("proceed", END)
  .addEdge("cancel", END)
  .compile({ checkpointer: new MemorySaver() });   // ← 必须有 checkpointer

const config = { configurable: { thread_id: "approval-123" } };

// ===== 第一步：跑到中断 =====
const paused = await graph.invoke({ action: "转账 500 元" }, config);

// ⭐ 用类型守卫收窄，才能在编译期访问 __interrupt__
if (isInterrupted(paused)) {
  console.log(paused.__interrupt__);
  // [{ id: "39157825...", value: { question: "是否批准以下操作？", detail: "转账 500 元", options: ["yes","no"] } }]
}

// ===== 第二步：查看状态（可选）=====
const snapshot = await graph.getState(config);
console.log(snapshot.next);      // ["approval"]  ← 停在这一步

// ===== 第三步：人工决策后恢复 =====
const result = await graph.invoke(new Command({ resume: "yes" }), config);
console.log(result);
// { action: "转账 500 元", approved: true, result: "已批准并执行" }
```

⭐ **注意示例里的 `Command`**：审批节点用 `Command` 的 `goto` 决定去 `proceed` 还是 `cancel`。如果改成普通返回对象，`proceed` / `cancel` 就永远不会被执行——因为没有边从 `approval` 指向它们。**审批分支的路由必须显式写出来。**

⭐ **三个必须记住的细节**：

1. **中断载荷在结果的 `__interrupt__` 字段**——不是普通字段，是一个数组
2. **`isInterrupted(result)` 是一个类型守卫**——用它包住，TS 才知道 `__interrupt__` 存在，访问它不需要 `as any` 断言
3. **恢复必须用同一个 `thread_id`**——换了 ID 就恢复不了（因为找不到那个检查点）

### 4.4 中断载荷的格式：可以用 Zod 约束

`interrupt()` 的第二个参数可以传一个 Zod schema，用来**约束"人应该返回什么形状的值"**：

```ts
const decision = interrupt(
  { question: "是否批准？", detail: state.action },
  { responseSchema: z.object({ approved: z.boolean(), reason: z.string() }) },
);
// 恢复时传 { approved: true, reason: "金额在预算内" }
// ⚠️ 形状不对会直接抛 ZodError
```

⭐ 这个能力很值钱：前端可以**根据 schema 渲染表单**，后端**自动校验人的输入**。

### 4.5 ⚠️ 不要用 try/catch 吞掉中断

```ts
// ❌ 错误：中断异常被吞，图永远不会停
const node = (state) => {
  try {
    const decision = interrupt("批准吗？");
    return { result: decision };
  } catch (e) {
    return { result: "出错了" };   // ← 这里把 GraphInterrupt 吃了
  }
};
```

⭐ `interrupt()` **是靠抛异常实现暂停的**。你的 `catch` 会把它当成普通错误处理掉，图就继续往下跑了——你精心设计的审批流程**静默失效**，而且没有任何报错。

正确做法：**把 `interrupt()` 放在 try 之外**，或者在 catch 里**重新抛出**：

```ts
import { interrupt } from "@langchain/langgraph";

// ⚠️ 下面是"节点函数长什么样"的片段，不是完整程序
// ✅ 正确：把 interrupt() 放在 try 之外
const node = async (state: { question: string }) => {
  const decision = interrupt("批准吗？");   // ← try 外面
  try {
    await doSomething();                    // ← 你自己的业务调用
  } catch (e) {
    console.error(e);
  }
  return { result: decision };
};
```

### 4.6 ⚠️ 中断前的副作用必须幂等

因为恢复时节点**从头重新执行**，`interrupt()` 之前的代码会跑第二遍。

```ts
// ❌ 危险：恢复时会多写一条审计日志
const node = async (state) => {
  const auditId = await db.createAuditLog({ action: "pending_approval" });  // ← 会执行两次
  const decision = interrupt("批准吗？");
  return { decision, auditId };
};

// ✅ 安全：interrupt 之前只做幂等操作
const node = async (state) => {
  await db.upsertAuditLog({ id: state.requestId, action: "pending_approval" });  // upsert 幂等
  const decision = interrupt("批准吗？");
  return { decision };
};
```

⭐ **原则**：副作用（发邮件、扣款、写库）放在 `interrupt()` **之后**；不可逆的副作用最好**拆到独立节点**里。

### 4.7 ⚠️ 中断不等于"进程挂起"

这是本篇最重要的一个误解。

⭐ **中断时进程可以正常退出。** 状态已经落盘了，下次有人来恢复时，**新进程**读检查点、接着跑。

这正是持久化的意义：

```
进程 A                          数据库
  │  执行到 interrupt()
  ├──── 写检查点 ────────────►  ✓ 状态在这里
  │  返回给调用方
  ▼
进程退出（或被 K8s 杀掉 / 缩容到 0）

  ... 一天后，审批人点了"同意" ...

进程 B（新）
  │  invoke(new Command({ resume: "yes" }), { thread_id })
  ├──── 读检查点 ◄────────────  恢复现场
  │  从 interrupt() 那行继续
  ▼
执行完成
```

⚠️ **这意味着架构上可以做"轮询式"的人机协作**：不保持长连接、不占着一个 worker 进程等人类输入。用户点按钮时发起一次普通 HTTP 请求就行。

### 4.8 多个并行中断

如果两个节点在同一超步各自调了 `interrupt()`，会得到两个中断（`__interrupt__` 数组有两项，各带一个 `id`）。恢复时可以用**中断 ID 映射表**分别给值：

```ts
import { StateGraph, START, END, StateSchema, ReducedValue, MemorySaver, Command, interrupt, isInterrupted, INTERRUPT } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  // ⚠️ 这里必须给 reducer：a、b 会在同一超步并行写 vals
  vals: new ReducedValue(z.array(z.string()).default(() => []), {
    reducer: (l, r) => l.concat(r),
  }),
});

const graph = new StateGraph(State)
  .addNode("a", () => {
    const answer = interrupt("question_a") as string;
    return { vals: [`a:${answer}`] };
  })
  .addNode("b", () => {
    const answer = interrupt("question_b") as string;
    return { vals: [`b:${answer}`] };
  })
  .addEdge(START, "a").addEdge(START, "b")
  .addEdge("a", END).addEdge("b", END)
  .compile({ checkpointer: new MemorySaver() });

const config = { configurable: { thread_id: "1" } };
const interrupted = await graph.invoke({ vals: [] }, config);

// interrupted[INTERRUPT] = [
//   { id: "cefc8a19...", value: "question_a" },
//   { id: "c88c2a2c...", value: "question_b" },
// ]

// 用「中断 ID → 回答」映射表分别恢复
const resumeMap: Record<string, string> = {};
if (isInterrupted(interrupted)) {
  for (const i of interrupted[INTERRUPT]) {
    if (i.id != null) resumeMap[i.id] = `answer for ${i.value}`;
  }
}
const result = await graph.invoke(new Command({ resume: resumeMap }), config);
console.log(result);
// { vals: ["a:answer for question_a", "b:answer for question_b"] }
```

⚠️ **这个例子又撞上了第 4 页那条并发规则**：`a` 和 `b` 在同一超步并行写 `vals`，如果 `vals` 是普通 schema（LastValue 通道），恢复时会直接抛 `InvalidUpdateError: LastValue can only receive one value per step`。**并行写就要给 reducer**——这条规则在中断场景同样成立。

---

## 五、⭐ 在 `createAgent` 里做 HITL

### 5.1 两条路

LangChain.js v1 提供了**预置中间件** `humanInTheLoopMiddleware`，专门做"敏感工具调用要审批"：

| | 图 + 手写 `interrupt()` | `createAgent` + `humanInTheLoopMiddleware` |
|---|---|---|
| 审批点 | **任意位置**（任何节点里） | **仅限工具调用前** |
| 审批内容 | 你自己定义（对象、Zod schema 约束） | 工具名 + 参数 |
| 决策类型 | 你自己定义 | 固定四种：`approve` / `edit` / `reject` / `respond` |
| 代码量 | 多（要写节点、边、检查点） | 少（加一个中间件） |
| 灵活性 | 高 | 受限 |

⭐ **选择标准很简单**：

- **只是"某些工具要人工确认"** → 用 `humanInTheLoopMiddleware`
- **要审批的不是一个工具调用，而是流程里的某个判断/某个外部操作** → 用图 + `interrupt()`

### 5.2 `humanInTheLoopMiddleware` 用法

```ts
import { createAgent, humanInTheLoopMiddleware } from "langchain";
import { MemorySaver, Command } from "@langchain/langgraph";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const writeFile = tool(
  async ({ path, content }) => `已写入 ${path}`,
  {
    name: "write_file",
    description: "写入文件",
    schema: z.object({ path: z.string(), content: z.string() }),
  },
);

const agent = createAgent({
  model: "openai:gpt-4o-mini",
  tools: [writeFile],
  middleware: [
    humanInTheLoopMiddleware({
      interruptOn: {
        // write_file 每次都要审批
        write_file: {
          allowedDecisions: ["approve", "edit", "reject"],
          description: "⚠️ 写文件需要审批",
        },
        // 其他工具（如果还有）自动放行
        // read_file: false,
      },
    }),
  ],
  checkpointer: new MemorySaver(),   // ⚠️ HITL 必须有 checkpointer
});

const config = { configurable: { thread_id: "hitl-1" } };

// 第一次：跑到需要审批的地方停下
const paused = await agent.invoke({ messages: [{ role: "user", content: "写个文件" }] }, config);
const request = paused.__interrupt__?.[0]?.value;
// request.actionRequests  → [{ name: "write_file", args: {...} }]
// request.reviewConfigs   → [{ actionName: "write_file", allowedDecisions: [...] }]

// 人工决策后恢复
const resumed = await agent.invoke(
  new Command({
    resume: {
      decisions: [
        { type: "approve" },                              // 直接批准
        // { type: "edit", editedAction: { name: "write_file", args: {...} } }  // 改参数后批准
        // { type: "reject", message: "不允许" }          // 拒绝并告知模型
        // { type: "respond", message: "..." }            // 直接代替模型回答
      ],
    },
  }),
  config,
);
```

⭐ **四种决策类型**：

| 类型 | 含义 |
|------|------|
| `approve` | 批准，按原参数执行工具 |
| `edit` | 修改工具名/参数后再执行 |
| `reject` | 拒绝，把 `message` 告诉模型 |
| `respond` | 不用工具，直接把 `message` 当作工具结果 |

⭐ **`interruptOn` 的值可以是 `false`**——表示该工具自动放行，不需要审批。这让"同一套工具里只拦危险的那个"很容易表达。

⚠️ **HITL 中间件必须有 checkpointer**——没有它就存不下中断点，审批流程直接失效。

---

## 六、⭐ 流式与前端

### 6.1 `streamMode` 各模式的语义

LangGraph.js v1 的 `streamMode` 支持这些取值（实测 `@langchain/langgraph@1.4.21`）：

| 模式 | 产出什么 | 什么时候用 |
|------|---------|-----------|
| `values` | **每步之后的完整状态** | 前端要同步完整状态；调试 |
| `updates` | **每步的增量更新**（按节点名分组） | 前端只要变化的部分；效率最高 |
| `messages` | LLM 的 token 流（`[消息块, 元数据]` 二元组） | 聊天界面逐字输出 |
| `custom` | 节点通过 `config.writer()` 主动发的自定义数据 | 进度条、日志 |
| `tools` | 工具调用的生命周期事件 | 展示工具执行进度 |
| `debug` | 执行期间的全部调试信息 | 深度排查 |
| `tasks` | 任务的开始/结束事件 | 看并行执行情况 |
| `checkpoints` | 检查点写入事件 | 持久化调试 |

> **版本说明**：`tools` 模式是 **LangGraph.js 特有**的——Python v1 的 `StreamMode` 里没有它（Python 是 `values` / `updates` / `checkpoints` / `tasks` / `debug` / `messages` / `custom`）。

⭐ **`values` vs `updates` 的选择**：`values` 每步都发全量状态，状态大时带宽浪费；`updates` 只发变化，前端自己合并。**生产环境通常用 `updates`**（或 `messages`）。

### 6.2 多路复用

可以同时开多个模式，数据块变成 `[模式名, 数据]` 二元组：

```ts
import { HumanMessage } from "@langchain/core/messages";

// ⚠️ 下面是"怎么读多路复用的流"的片段，graph 沿用上文
for await (const [mode, chunk] of await graph.stream(
  { messages: [new HumanMessage("北京天气？")] },
  { streamMode: ["messages", "updates"] },
)) {
  if (mode === "messages") {
    // chunk 是 [消息块, 元数据]
    console.log(chunk[0].content);
  } else if (mode === "updates") {
    // chunk 是 { 节点名: 该节点的更新 }
    console.log(Object.keys(chunk));
  }
}
```

实测输出：

```
mode = messages | chunk = [ AIMessage 消息块, 元数据 ]
mode = updates  | chunk = { model: { messages: [...] } }
mode = updates  | chunk = { tools: { messages: [...] } }
```

⚠️ **用数组形式时必须按 `mode` 分派**——不同模式的数据结构完全不同，不能假设每块形状一致。

### 6.3 ⚠️ `subgraphs: true` 是子图场景的必需项

**这是本节最关键的一条。**

实测对比（父图里嵌了一个子图）：

```ts
// ⚠️ 下面是紧接上一段图的续写，parent 沿用上文
// ❌ 不加 subgraphs:true
for await (const chunk of await parent.stream({ foo: "" }, { streamMode: "updates" })) {
  console.log(chunk);
}
// { p1: { foo: "hi" } }
// { p2: { foo: "hi!" } }        ← 只有父图节点的结果，子图内部节点的输出不见了

// ✅ 加 subgraphs:true
for await (const chunk of await parent.stream(
  { foo: "" },
  { streamMode: "updates", subgraphs: true },
)) {
  console.log(chunk);
}
// [[], { p1: { foo: "hi" } }]
// [["p2:862bcf98-..."], { subNode: { foo: "hi!" } }]   ← 子图内部节点的输出在这里
// [[], { p2: { foo: "hi!" } }]
```

⚠️ **数据形状变了**：加了 `subgraphs: true` 之后，每个数据块是 `[命名空间, 数据]` 二元组。命名空间是**路径数组**：

- `[]`（空数组）= 根图
- `["p2:862bcf98-..."]` = 某个子图调用

⭐ **前端必须同时处理这两种形状**。写成这样最稳：

```ts
// ⚠️ 下面是紧接上一段图的续写，graph / input 沿用上文
for await (const chunk of await graph.stream(input, {
  streamMode: ["messages", "updates"],
  subgraphs: true,
})) {
  // 兼容有/无命名空间两种形状
  const [ns, data] = Array.isArray(chunk) ? chunk : [[], chunk];
  if (ns.length === 0) {
    // 根图数据
  } else {
    // 子图数据，ns 是子图路径
  }
}
```

⭐ **为什么子图输出默认被"吞掉"**：子图是一个**独立的图实例**，它的节点有自己的状态和流。父图的 `stream()` 默认只报告父图自己的节点。要看到子图内部发生了什么，必须显式要求。

Python 版流式模式的完整对照见 [[langgraph/modules/langgraph_streaming/流式模式对比]]。

### 6.4 React 端的 `useStream`

LangGraph 的 React 前端集成的核心是 **`useStream` Hook**，它把"建连、发消息、收流、处理中断、重连"都封装好了。

```tsx
import { useStream } from "@langchain/react";
// 旧版 SDK 的路径是：import { useStream } from "@langchain/langgraph-sdk/react";

const thread = useStream({
  apiUrl: "http://localhost:2024",   // LangGraph Agent Server 地址
  assistantId: "my-agent",            // 图的 ID
  threadId,                           // 当前会话 ID
  messagesKey: "messages",            // 消息在状态里的键名，默认就是 messages
  onThreadId: (id) => setThreadId(id),
});

// 常用返回值
thread.messages      // 已组装好的完整消息数组
thread.values        // 最新的状态快照
thread.interrupts    // 待处理的中断
thread.isLoading     // 是否正在流式接收
thread.submit(...)   // 发送新消息
thread.stop()        // 中断当前流
thread.respond(...)  // 恢复一个中断
```

⚠️ **包名要说准**：`useStream` **不在 `@langchain/langgraph` 主包里**（我逐一检查了 0.1.x 到 1.5.x 的所有版本，`@langchain/langgraph` 从未导出过 `useStream`）。它有两个来源：

| 包 | 版本 | 说明 |
|----|------|------|
| `@langchain/react` | v1.2.x | 较新的 React 绑定（面向 v2 原生流运行时），**当前推荐** |
| `@langchain/langgraph-sdk/react` | v1.12.x | 旧版 SDK 的 React 绑定 |

⭐ **前端断点续跑的关键**：把 `threadId` 存进 `localStorage` 或 URL 查询参数。页面刷新后用同一个 `threadId` 挂载 `useStream`，就能接上之前的对话——**这正是 checkpointer 在后端支撑的能力**。

---

## 陷阱

1. **⚠️ 生产环境用内存版 checkpointer**——`MemorySaver` 存在进程内存里，**容器重启 / 滚动更新 / 进程崩溃都会让所有会话历史归零**；生产必须用 SQLite（单机）或 PostgreSQL（多实例）；
2. **⚠️ `thread_id` 每次请求都换**（比如用 `Date.now()` 生成）——每轮都是新线程，上下文全丢，**而且不报任何错**；thread_id 必须稳定标识一条会话；
3. **⚠️ 恢复时用了不同的 `thread_id`**——`Command({ resume })` 必须配**原来那个** `thread_id`，换了就找不到检查点；
4. **⚠️ 恢复时传了普通对象而不是 `Command`**——`invoke({ messages: [...] })` 是"新一轮输入"，`invoke(new Command({ resume: x }))` 才是"恢复"。**写成普通对象会导致图从中断点继续执行、看起来"卡住不动"**；
5. **⚠️ 以为 `interrupt()` 后进程会挂起等着**——**中断是状态持久化，进程可以正常退出**；恢复时可能是另一个新进程。架构上应按"轮询式"设计，不要保持长连接占着 worker；
6. **⚠️ 用 try/catch 吞掉了中断异常**——`interrupt()` 靠抛 `GraphInterrupt` 实现暂停，宽泛的 `catch` 会把它当普通错误处理，**审批流程静默失效**；
7. **⚠️ `interrupt()` 之前有不可逆副作用**——恢复时节点从头重跑，之前的写库/发邮件会**执行第二次**；副作用应放在 `interrupt()` 之后或拆到独立节点；
8. **⚠️ 有子图但忘了 `subgraphs: true`**——子图内部节点的输出**根本不会出现在流里**（父图节点的结果还在，容易误以为"子图没跑"）；
9. **⚠️ 加了 `subgraphs: true` 却按原来的形状解析数据**——加了之后数据块变成 `[命名空间, 数据]` 二元组，不做兼容会解析错；
10. **⚠️ 给无状态短请求挂 checkpointer**——每超步一次 I/O 写入，是**纯开销**；只有需要续跑/中断/审计的场景才挂；
11. **⚠️ 忘了 `PostgresSaver` 要手动 `await setup()`**——不调用会因缺表而失败；
12. **⚠️ 多个实例共享同一个 SQLite 文件**——文件锁不允许并发写，多实例部署必须换 Postgres；
13. **⚠️ 以为 `updateState` 改了状态就会重跑图**——它只改状态，要继续执行还得再 `invoke` 一次；
14. **⚠️ HITL 中间件没配 checkpointer**——`humanInTheLoopMiddleware` 依赖 checkpointer 存中断点，缺了就等于没生效。

## 小结

- ⭐ **checkpointer** 在每个超步边界存状态快照，支撑四件事：**会话续跑 / 中断恢复 / 时间旅行 / 审计**
- ⭐ **三档后端**：内存版（开发测试，重启即丢）、SQLite 版（单机生产）、PostgreSQL 版（生产多实例，必须 `await setup()`）；`MemorySaver` 随主包自动安装，SQLite/Postgres 需单独装
- ⚠️ **生产不能用内存版**；⚠️ **checkpointer 写入在热路径上**，无状态短请求别挂
- ⭐ **`thread_id` 是持久化游标**：同 ID 续跑、不同 ID 隔离；`getState` / `getStateHistory` / `updateState` 分别用于查状态、看历史、改状态
- ⭐ **HITL 链路**：`interrupt()` 抛 `GraphInterrupt` → 状态落盘 → 结果的 `__interrupt__` 里是载荷 → `Command({ resume })` 用同一 `thread_id` 恢复 → 节点从 `interrupt()` 那行继续
- ⚠️ **中断 ≠ 进程挂起**：状态已落盘，进程可退出，恢复可能是另一个进程
- ⚠️ **不要 try/catch 吞中断**；⚠️ **interrupt 之前的副作用要幂等**
- ⭐ **`createAgent` + `humanInTheLoopMiddleware`**：只拦工具调用，四种决策 `approve`/`edit`/`reject`/`respond`；要审批流程里的其他环节，得用图 + 手写 `interrupt()`
- ⭐ **`streamMode`**：JS v1 有 8 种（比 Python 多一个 `tools`）；多路复用用数组形式；⚠️ **有子图必须加 `subgraphs: true`**，且数据块形状会变成 `[命名空间, 数据]`
- ⭐ **React 的 `useStream`** 来自 `@langchain/react`（或旧版 `@langchain/langgraph-sdk/react`），**不是** `@langchain/langgraph` 主包

## 练习

1. 用 `MemorySaver` 搭一个带 `messages` 通道的图，用**同一个** `thread_id` 连续 `invoke` 三次，打印每轮 `messages.length`，验证历史在累积；再换一个 `thread_id` 调一次，验证它是隔离的。
2. 把第 1 题的 `MemorySaver` 换成 `SqliteSaver.fromConnString("./x.db")`（记得 `npm install @langchain/langgraph-checkpoint-sqlite`），**重启 Node 进程**后再用同一个 `thread_id` 调一次，验证历史还在。
3. 实现一个审批节点：`interrupt()` 抛出待审批内容 → 恢复时根据 `resume` 值分别走 `execute` 或 `reject` 分支。故意在 `interrupt()` 之前加一句 `console.log`，观察恢复时它是否重复执行。
4. ⭐ 改造第 3 题：把 `interrupt()` 放进一个 `try` 块里并在 `catch` 中返回默认值，观察图是否还会停——用这个实验说明"为什么不能 catch 掉中断"。
5. 用 `interrupt()` 的第二个参数传一个 Zod `responseSchema`（比如 `{ approved: z.boolean(), reason: z.string() }`），然后分别用正确和错误的形状恢复，对比报错信息。
6. ⭐ 搭一个父图内嵌子图的图，用 `streamMode: "updates"` 分别在**不加**和**加** `subgraphs: true` 两种情况下打印流，整理出两者的数据形状差异。
7. 用 `humanInTheLoopMiddleware` 搭一个带审批的 Agent：先 `invoke` 拿到 `__interrupt__`，再用 `approve` / `edit` / `reject` 三种决策分别恢复，观察消息历史的差异。
8. ⭐ 做一个"进程可退出"的验证：先跑到中断点，打印 `getState(config).next`；**退出进程**；新起一个进程用同一个 `thread_id` 恢复，验证能接着跑完。

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：[[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]] · 下一篇：[[typescript/03-生态连接-LangChain与LangGraph/6-Python与TypeScript双栈对照]]
