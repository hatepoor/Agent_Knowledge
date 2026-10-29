---
tags: [interview, typescript, 异步编程, 面试, 题库]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# 异步编程与 Node

> **什么时候该读**：面试官问「事件循环的执行顺序」「`Promise.all` 和 `allSettled` 怎么选」「为什么会有 unhandled rejection」「Node 是单线程吗」「什么是背压」，答不上来在 Node 岗位基本会被卡住。
> **答题主线**：**Node 的单线程只负责调度，真正的并行靠把任务甩出去；所有异步问题的根都在"事件循环的调度顺序"和"错误在异步边界上的传播"这两条线上。** 顺着这条线，Promise 组合子的选择、worker_threads 的使用、流与背压的设计就都是自然结论。

---

## Q1. Node 的事件循环是怎么回事？微任务和宏任务谁先执行？

**答：**

**Node 的事件循环是"一个while循环反复取阶段"，每个阶段处理一类回调；微任务（Promise 回调）在每个阶段之间被清空。**

### Node 事件循环的六个阶段

```
   ┌─→ timers（定时器：setTimeout/setInterval）─┐
   │                                              │
   ├─→ pending callbacks（部分系统回调）        │
   │                                              │
   ├─→ poll（I/O 事件：网络连接、数据到达）      │  ← 大部分时间待在这里
   │                                              │
   ├─→ check（setImmediate 回调）                │
   │                                              │
   ├─→ close callbacks（close 事件）              │
   │                                              │
   └─→ 每次阶段切换时：清空整个微任务队列 ←──────┘
```

### 核心规则（面试就答这三条）

1. **同步代码先跑完**，然后进入事件循环；
2. **每个宏任务（macrotask）执行完，立刻清空整个微任务队列**（不只是当前微任务）；
3. **微任务队列清空后，才进入下一个宏任务**。

### 微任务 vs 宏任务

| | 微任务（microtask） | 宏任务（macrotask） |
|---|-------------------|---------------------|
| **队列** | 微任务队列 | 各阶段的宏任务队列 |
| **典型来源** | `Promise.then` / `.catch` / `.finally`、`queueMicrotask`、`process.nextTick`（Node 特有，优先级更高） | `setTimeout` / `setInterval`、`setImmediate`、`I/O 回调`、`queueMicrotask` 之外的大多数异步 API |
| **清空时机** | **每个宏任务之后**（Node 中是每个阶段之间） | 轮转到对应阶段时 |
| **数量** | 不限，可以饿死宏任务 | 有限 |

### 经典执行顺序题

```ts
console.log('1');                                  // 同步

setTimeout(() => console.log('2'), 0);            // 宏任务：timers
setImmediate(() => console.log('3'));              // 宏任务：check
Promise.resolve().then(() => console.log('4'));    // 微任务
process.nextTick(() => console.log('5'));          // 微任务（更高优先级）

console.log('6');                                  // 同步
```

**输出：`1 6 5 4 2 3`**

**解释**：
- `1` `6` 是同步代码，先执行；
- `5` 是 `process.nextTick`，**在微任务里优先级最高**，先于 Promise 回调；
- `4` 是 Promise 微任务，接着执行；
- 然后进入宏任务：`2`（timers）和 `3`（check）**谁先是不确定的**！

**⚠️ 关键细节**：`setTimeout(0)` 和 `setImmediate` 的顺序**在顶层（main module）是不确定的**！

- 在 **Node 主模块**里：受事件循环启动时机影响，**不确定**；
- 在 **I/O 回调里**：**`setImmediate` 一定先于 `setTimeout(0)`**（因为 poll 阶段之后紧接 check 阶段）。

**这是一个非常高频的面试陷阱**——很多候选人说"setTimeout 总是先"，这在 I/O 回调里是错的。

**追问：`process.nextTick` 和 Promise 微任务谁先？**
→ **`process.nextTick` 优先**。Node 会把 nextTick 队列维护成一个**独立的、更高优先级的队列**，每次是"先清空 nextTick 队列，再清空 Promise 微任务队列"。⚠️ 但 nextTick 里如果递归调用 nextTick，**会把整个事件循环饿死**（微任务队列无限制，宏任务永远排不上）——这是 Node 里的经典坑。

**追问：浏览器和 Node 的事件循环有区别吗？**
→ 有三处：
1. **Node 有 `process.nextTick`**，浏览器没有；
2. **Node 有 `setImmediate`**，浏览器没有（浏览器用 `postMessage` 模拟）；
3. **Node 的 poll 阶段会阻塞等待 I/O**，浏览器没有对应阶段；
4. 微任务的清空时机略有不同——浏览器是"每个宏任务之后"，Node 是"每个阶段的回调之后"。

---

## Q2. Promise 的三种状态是什么？状态能逆转吗？`.then` 链是怎么工作的？

**答：**

**Promise 有三种状态：`pending`（等待）、`fulfilled`（成功）、`rejected`（失败）。状态一旦从 pending 变成 fulfilled 或 rejected，就永远不可再变。**

```
        调用 executor
            ↓
        ┌─ pending ─┬─→ fulfilled ──→ .then(onFulfilled) 的返回值
        │          │                        ↓
        │          └─→ rejected ────→ .then(onRejected) 的返回值
        │                                   ↓
        └────────────────────────────────────┴─→ 新的 Promise（继续链）
```

**状态不可逆是 Promise 的核心契约**：

```ts
let resolveP: (v: string) => void;
const p = new Promise<string>((resolve) => { resolveP = resolve; });

p.then(v => console.log('第一次', v));
resolveP!('a');
resolveP!('b');          // ⚠️ 第二次调用被忽略
p.then(v => console.log('第二次', v));
// 输出：第一次 a / 第二次 a
```

### `.then` 链的工作机制

**`.then` 总是返回一个新的 Promise**，回调的返回值会成为新 Promise 的结果：

```ts
Promise.resolve(1)
  .then(v => v + 1)          // → 2（返回数字 → 新 Promise fulfilled with 2）
  .then(v => { throw new Error('boom') })  // → 抛异常 → 新 Promise rejected
  .catch(e => e.message)     // → 'boom'（捕获并恢复）
  .then(v => console.log(v)); // → 打印 boom
```

**三条必须记住的规则**：

| 规则 | 说明 |
|------|------|
| **返回值透传** | 回调返回普通值 → 新 Promise 以该值 fulfilled |
| **抛异常透传** | 回调 `throw` → 新 Promise rejected，**且会跳过后续的 `.then(onFulfilled)`**，直接进 `.catch` |
| **返回 Promise 会等待** | 回调返回 Promise → 新 Promise 等待它 settle，**采用其结果**（类似 `await`） |

**第 3 条是"Promise 传染性"的核心**：

```ts
function getUserId(): Promise<number> {
  return Promise.resolve(42);
}
Promise.resolve(1)
  .then(() => getUserId())   // 返回 Promise<number>
  .then(id => console.log(id + 1));  // ✅ id 是 number，类型正确
```

**类型层面**：`.then` 的返回类型由回调返回值推导，`Promise<void>` → `Promise<number>` 这类映射靠的是 `Awaited<T>`（TS 4.5+）。

### 为什么 `.then` 不会同步执行

```ts
console.log('A');
Promise.resolve().then(() => console.log('B'));
console.log('C');
// A C B
```

**因为 `.then` 的回调被放进微任务队列**，即使 Promise 已经 settled，回调也要等当前同步代码块跑完。

**追问：`.then` 和 `await` 的执行顺序有区别吗？**
→ **微任务内部的顺序基本一致**（`await` 底层就是 `.then` + 微任务）。但有个细微差别：**连续的 `await` 会在每个 `await` 处让出一次微任务**，而 `.then` 链是连续排队的。实际影响很小，但面试可以说出来。

**追问：`Promise` 和 `async/await` 是什么关系？**
→ **`async/await` 是 Promise 的语法糖**。`async function` 的返回值**永远是一个 Promise**（哪怕你 `return 42`，实际返回 `Promise<42>`）。`await` 等价于 `await Promise.resolve(值)`。

**追问：Promise 一旦被 reject 就"不可挽回"，那 `.catch` 之后还能拿到值吗？**
→ 能，但**是新的 Promise**。`p.catch(handler)` 返回新 Promise，其状态由 `handler` 的返回值决定。**原 Promise 的状态永远是 rejected。** 这是"每个 `.then` 返回新 Promise"这个设计的必然结果。

---

## Q3. `Promise.all`、`allSettled`、`race`、`any` 有什么区别？分别什么场景用？

**答：**

**一句话区分：看你要「全都要」「全都要但不因失败中止」「谁先赢」「谁先成功」。**

| 方法 | 结算条件 | 失败时 | 结果类型 |
|------|---------|--------|---------|
| **`Promise.all`** | **全部成功**才成功 | ⚠️ **快速失败**（第一个 reject 就整体 reject） | `Promise<T[]>` |
| **`Promise.allSettled`** | **全部结算**（无论成败） | 永不失败 | `Promise<SettledResult<T>[]>` |
| **`Promise.race`** | **第一个 settle**（成功或失败都算） | 第一个失败就失败 | `Promise<T>`（联合） |
| **`Promise.any`** | **第一个成功** | ⚠️ **全部失败才失败**（抛 `AggregateError`） | `Promise<T>`（联合） |

### `Promise.all` —— 快速失败

```ts
const [user, posts] = await Promise.all([
  fetchUser(),
  fetchPosts(),
]);
```

**⚠️ 关键行为：快速失败（fail fast）**。第二个请求即使还会成功，第一个失败了，`Promise.all` **立刻 reject**，不等其余的。

**但要注意：其余的 Promise 不会被取消！** 它们继续跑，只是结果没人接收了。这会导致：

- HTTP 请求发出去了服务端仍然处理；
- 如果里面有 `catch` 抛错，可能变成**未处理的 rejection**。

**解法**（TS 里需要处理）：

```ts
// 方案1：每个 Promise 内部自己 catch，转换结果
const results = await Promise.all(
  tasks.map(t => t().then(v => ({ ok: true, v })).catch(e => ({ ok: false, e })))
);
// 方案2：不用 all，用 allSettled
```

### `Promise.allSettled` —— 永不失败

```ts
const results = await Promise.allSettled([fetchUser(), fetchPosts()]);
for (const r of results) {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.log(r.reason);
}
```

**类型**：`PromiseSettledResult<T>` 是一个判别联合：

```ts
type SettledResult<T> =
  | { status: 'fulfilled'; value: T }
  | { status: 'rejected'; reason: unknown };
```

⚠️ **TS 5.0+ 注意**：`Promise.allSettled` 的 `reason` 类型是 `any`（TS 5.0 之前）或 `unknown`（TS 5.0 之后受 `useUnknownInCatchVariables` 影响）——**实际用的时候要自己收窄 `reason`**，它可能是任何东西。

### `Promise.race` —— 谁先到谁赢

**典型用途：超时控制**。

```ts
function withTimeout<T>(p: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    p,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`超时 ${ms}ms`)), ms)
    ),
  ]);
}
```

**⚠️ 陷阱**：`Promise.race` **不会取消**输掉的那个 Promise。超时后原请求还在跑——**真要取消得用 `AbortController`**。

### `Promise.any` —— 第一个成功的赢

**典型用途：多个镜像源/多个服务节点，取最快成功的**。

```ts
const result = await Promise.any([
  fetchFromMirror('a'),
  fetchFromMirror('b'),
]);
```

**⚠️ 与 `Promise.all` 完全相反的失败语义**：所有都失败才 reject，错误是 `AggregateError`（`err.errors` 数组里是所有原因）。

**Node 特有**：`Promise.any` 在 Node 15+ 才可用（`AggregateError` 同批加入）。`Promise.allSettled` 是 Node 12.9+。

### 选型决策表

| 需求 | 用 |
|------|-----|
| 所有结果都要，**任何一个失败就算整体失败** | `Promise.all` |
| 所有结果都要，**单个失败不影响其他**（批量上报、清理任务） | `Promise.allSettled` |
| 只要**最先完成**的（超时、限流取最快） | `Promise.race` |
| 只要**最先成功**的（多副本、镜像降级） | `Promise.any` |

**⚠️ 最常见的错误**：批量操作里用 `Promise.all`，结果一个失败导致其他成功的结果全部丢失、且报错信息只有第一个错误。**批量场景（通知、埋点、清理）应该用 `allSettled`。**

**追问：`Promise.all` 传入非 Promise 的值会怎样？**
→ 会被自动包装成 `Promise.resolve(值)`。所以 `Promise.all([1, Promise.resolve(2)])` 完全合法，返回 `[1, 2]`。**这也是 `Promise.all` 和 `Promise.allSettled` 在类型上的区别：前者的输入类型被推断为 `Iterable<T | PromiseLike<T>>`。**

**追问：`Promise.all` 的输入数组是空的话？**
→ 返回 `Promise.resolve([])`，**立即 fulfilled**。注意它返回一个**空数组**而不是 rejected。

---

## Q4. 串行执行和并行执行的性能差多少？为什么 `for await` 不等于并行？

**答：**

**串行 = 等一个做完再做下一个；并行 = 全部发起同时等。差距等于「最慢的那个」vs「全部耗时之和」。**

### 错误写法：串行

```ts
// ❌ 总耗时 = 100 + 200 + 300 = 600ms
const user = await fetchUser();
const posts = await fetchPosts();
const comments = await fetchComments();
```

### 正确写法：并行

```ts
// ✅ 总耗时 = max(100, 200, 300) = 300ms
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments(),
]);
```

**⚠️ 关键点：`Promise.all([fetchUser(), ...])` 里的三个 `fetchUser()` 是同步被调用的**（在 `Promise.all` 执行前就都发起了），**并行发生在发起的那一刻，不是 await 的那一刻**。

### 循环里的并行

```ts
// ❌ 串行：每个都要等前一个
for (const id of ids) {
  results.push(await fetchOne(id));
}

// ✅ 并行：全部发起
const results = await Promise.all(ids.map(id => fetchOne(id)));
```

### 什么时候**必须**串行

**有依赖关系时必须串行**：

```ts
// 步骤2 依赖步骤1 的结果 —— 无法并行
const token = await login();
const data = await fetchWithToken(token);
```

**优化模式：用 Promise 链代替 await 的串行写法**，保证并行：

```ts
// ❌ 错误：await 在循环里导致串行
const results = [];
for (const id of ids) {
  results.push(await fetchOne(id));
}

// ✅ 正确：改成链式，Promise.all 里全部并行
const results = await Promise.all(
  ids.reduce((acc, id) => acc.then(a => fetchOne(id).then(v => [...a, v])), Promise.resolve([]))
);
```

**更好的写法**（可读性优先）：

```ts
const results = await Promise.all(ids.map(async (id) => {
  const user = await fetchOne(id);      // 这里面的 await 是并发的
  return process(user);
}));
```

**注意最后这个写法**：每个 `map` 的回调是 `async`，`fetchOne(id)` **同步发起**，所以是并行的。`map` 回调返回的是 Promise，`Promise.all` 等它们全部完成。

### `for await...of` 的含义

```ts
for await (const chunk of stream) {
  handle(chunk);
}
```

**⚠️ `for await` 是"逐个等待迭代器的下一个值"，本质是串行的**——虽然源可以是异步的。**它不等于并行。**

**真正的并行要靠**：

```ts
// 手动控制并发数的并行
async function mapLimit<T, R>(
  items: T[],
  limit: number,
  fn: (item: T) => Promise<R>
): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let index = 0;

  const workers = Array.from({ length: Math.min(limit, items.length) }, async () => {
    while (index < items.length) {
      const current = index++;
      results[current] = await fn(items[current]);
    }
  });

  await Promise.all(workers);
  return results;
}
```

**这是处理"上百个请求要打后端"的实用工具**——无限并行会打爆连接池。

**追问：为什么 `for await` 不能并行？**
→ 因为迭代器协议（`next()` 返回 `Promise<IteratorResult>`）是**一次一个**的语义。`for await` 每次循环都要 `await` 下一个值。要并行只能一次性把源全部读出来（`Promise.all([...iter])`），那反而失去了流式的内存优势。

**追问：真实的性能差距取决于什么？**
→ 取决于**任务类型**：
- **I/O 密集**（网络、数据库、文件）：并行收益巨大，因为等待时间占绝大部分；
- **CPU 密集**（计算、加密、图像处理）：**并行无效**——单线程跑 CPU 任务，并行只是让多个任务抢同一个线程，总耗时不变。**这时需要 `worker_threads`。**

---

## Q5. 什么是未处理的 rejection（unhandled rejection）？为什么会发生？怎么防？

**答：**

**未处理的 rejection = 一个 Promise 被 reject 了，但没有任何 `.catch`（或 `try/catch`）接住它。Node 会因此打印警告甚至终止进程。**

### 典型触发场景

```ts
// 1. 忘了 catch
Promise.reject(new Error('boom'));      // ❌ 未处理

// 2. catch 挂在错误的链上
Promise.reject(new Error('boom')).catch(handle);
// 正确：catch 接在链末才有效

// 3. async 函数里抛错，调用方没 await 也没 catch
async function risky() { throw new Error('boom'); }
risky();                                 // ❌ 返回的 Promise 没人管

// 4. Promise.all 快速失败后，其余的 reject 无人接管
Promise.all([p1, p2]);   // p1 快速失败 → 整体 reject
// p2 后来也 reject → 它没有 catch → 未处理 rejection ⚠️

// 5. 事件回调里抛错（不是 Promise 场景，是 uncaught exception）
emitter.on('data', () => { throw new Error('boom'); });  // 直接崩进程
```

### Node 的处理策略（版本敏感，必须写版本号）

| Node 版本 | 默认行为 |
|-----------|---------|
| **Node ≤ 14** | 只打印 `UnhandledPromiseRejectionWarning`，**不终止进程** |
| **Node 15+** | **默认 `--unhandled-rejections=throw`** → 打印错误并**终止进程**（和未捕获异常同等级别） |

**这意味着：Node 15 之后，一个漏掉的 rejection 会让整个服务挂掉。** 这是从 Node 14 升 15 时的经典"上线即崩"事故。

**三种处理模式**（通过 `--unhandled-rejections` 切换）：

| 模式 | 行为 |
|------|------|
| `throw`（Node 15+ 默认） | 抛未捕获异常 → **进程终止** |
| `strict` | 始终终止（无论是否注册了 `process.on('unhandledRejection')`） |
| `warn` | 只打印警告，继续运行（**不推荐**，会掩盖问题） |
| `none` | 静默忽略（**绝对不要用**） |
| `warn-with-error-code` | 打印警告并设置退出码 |

### 防御手段

**1. 全局兜底（作为最后一道防线）**：

```ts
process.on('unhandledRejection', (reason, promise) => {
  logger.error('未处理的 Promise rejection', reason);
  // ⚠️ 不要在这里吞掉：至少上报 + 记录，然后按业务决定是否退出
  process.exitCode = 1;
});
```

**⚠️ 但要清楚**：全局 handler 是**兜底，不是解法**。真正该做的是**每个 rejection 都有明确的归属**。

**2. 根本做法：让每个 async 调用都有明确的 catch**。

```ts
// 工具函数：封装"故意不 await"的场景
function fireAndForget(p: Promise<unknown>, ctx: string): void {
  p.catch((err) => logger.error(`${ctx} 失败`, err));
}

// 用在事件回调里
emitter.on('data', (chunk) => {
  fireAndForget(processChunk(chunk), 'processChunk');
});
```

**3. Express / Fastify 路由层的处理**：异步 handler 里抛错必须显式 catch，否则请求会挂住。

```ts
// Express 5 起自动把 async handler 的 rejection 交给错误中间件
app.get('/data', async (req, res) => {
  const data = await fetchData();   // 抛错会被 Express 5 捕获
  res.json(data);
});
```

**⚠️ Express 4 不会**：async handler 抛出的错误**既不会进错误中间件，也不会响应客户端**——请求会一直挂到超时。**Express 4 必须手动 `try/catch` 或用 `express-async-errors` 这类补丁包。**

**追问：`unhandledRejection` 和 `uncaughtException` 有什么区别？**
→ `uncaughtException` 是**同步**代码抛出的未捕获异常（会跳过 try/catch 直接冒泡到进程顶）；`unhandledRejection` 是 **Promise 被 reject 但没有 handler**。处理方式不同，但 Node 15+ 它们的严重级别一样了（都是终止进程）。

**追问：为什么 `Promise.all` 快速失败会导致未处理 rejection？**
→ 因为 `Promise.all` 内部只给**第一个** rejection 建了 handler（用来触发整体 reject），**其余的 reject 无人接管**。解法就是给每个 Promise 单独挂 catch（用 `allSettled` 即可）。

---

## Q6. `async/await` 的异常处理怎么写？`await` 到底做了什么？

**答：**

**`await` 的本质是「暂停当前函数、注册一个 `.then` 回调、等 Promise settle 后恢复执行」。`try/catch` 能抓住它的原因，是因为它被编译成了 Promise 链的 `.then(onFulfilled, onRejected)`。**

### `await` 的三步语义

```ts
const value = await somePromise;
```

等价于：

```ts
const value = await Promise.resolve(somePromise).then(
  v => v,
  e => { throw e; }     // ← 这就是 try/catch 能接住的原因
);
```

**具体做了三件事**：

1. **把值转成 Promise**（`await 42` 等价于 `await Promise.resolve(42)`）；
2. **挂起当前函数**（后面的代码不执行）；
3. **Promise settle 后恢复**，resolve 值则继续，reject 则抛出到调用栈。

### 异常处理的两种写法

**写法 1：`try/catch`（推荐）**：

```ts
async function handle(): Promise<void> {
  try {
    const data = await fetchData();
    return process(data);
  } catch (err) {
    // ⚠️ err 的类型是 unknown（strict 模式下 useUnknownInCatchVariables）
    logger.error(err);
    throw new AppError('处理失败', { cause: err });
  }
}
```

**写法 2：`catch` 回调**：

```ts
const data = await fetchData().catch(err => {
  logger.error(err);
  return fallbackValue;
});
```

**⚠️ 关键区别**：`try/catch` 能捕获 **`try` 块里任何位置的异常**（包括非 await 的同步代码）；`.catch()` **只能捕获它前面那个 Promise 链上的 rejection**。

```ts
// try/catch 能抓到
try {
  const a = JSON.parse(bad);     // 同步抛错 ✅ 被抓到
  const b = await fetchData();    // async 抛错 ✅ 被抓到
  process(a, b);                  // 同步抛错 ✅ 也被抓到
} catch (e) {}

// .catch() 抓不到同步抛错
JSON.parse(bad).catch(handle);    // ❌ 同步异常直接抛出，.catch 根本没机会执行
```

### `Error` 的 `cause` 链（TS 4.2+ / ES2022）

```ts
try {
  await riskyOperation();
} catch (err) {
  throw new Error('读取用户档案失败', { cause: err });
  //    ^^^^^^^^ 原始错误被挂在 cause 上，调试时能看到根因
}
```

**这是错误处理的最佳实践**：**包装错误时一定要带 `cause`**，否则原始堆栈丢失，排查时只看到"读取用户档案失败"而无从下手。

### 自定义错误类型

```ts
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode = 500,
    options?: ErrorOptions,
  ) {
    super(message, options);       // ES2022：把 cause 传给父类
    this.name = 'AppError';
  }
}

class NotFoundError extends AppError {
  constructor(what: string) {
    super(`${what} 不存在`, 'NOT_FOUND', 404);
    this.name = 'NotFoundError';
  }
}
```

### ⚠️ `catch` 里的 `err` 是 `unknown`，必须收窄

```ts
try {
  await riskyOperation();
} catch (err) {
  if (err instanceof Error) {
    logger.error(err.message, { stack: err.stack });
  } else {
    logger.error('非 Error 抛出', { err });   // ⚠️ 有人会 throw 一个字符串
  }
}
```

**这也是为什么 `throw` 应该只抛 `Error` 对象**——抛字符串在 TS 里是允许的（`any` 类型），但会丢掉堆栈。**即使要抛非 Error 对象，也要包装成 `Error` 并挂 `cause`。**

**追问：`await` 会阻塞吗？**
→ **不会阻塞线程**。`await` 期间当前 async 函数暂停，**但事件循环继续处理其他任务**（I/O、定时器、其他微任务）。**Node 的单线程在 await 期间是空闲的**——这正是 Node 能扛住大量并发连接的原因，也是它扛不住 CPU 密集任务的原因。

**追问：`async` 函数一定返回 Promise 吗？**
→ **是**。`async function f() { return 42 }` 实际返回 `Promise<number>`。**但反过来不成立**——普通函数返回 Promise 不代表它是 async 函数（没有隐式的错误捕获和 await 支持）。

**追问：`await` 一个非 Promise 的值会怎样？**
→ 直接返回该值（自动包装再立即 resolve）。所以 `await 42` → `42`，`await null` → `null`。**这也是为什么 `await` 一个可能被解析成 thenable 的普通对象时要小心**——如果对象有 `then` 方法，`await` 会把它当 Promise 处理。

---

## Q7. Node 是单线程吗？什么任务会阻塞它？怎么用 `worker_threads`？

**答：**

**Node 的 JS 执行是单线程的（一个事件循环），但进程不是单线程——`worker_threads` 可以开真正的 OS 线程。而且 Node 内部大量工作跑在 C++ 的线程池上。**

### Node 的线程模型

```
主线程（JS 事件循环）
  ├── 你的 JS 代码、事件循环
  └── libuv 线程池（默认 4 个线程）
        ├── 文件系统 I/O
        ├── DNS 查询
        ├── 加密（crypto）
        └── zlib 压缩
```

**关键**：Node 的 `libuv` 线程池默认大小是 **4**（`UV_THREADPOOL_SIZE` 环境变量可调）。**这就是"并发读 100 个文件反而比读 4 个文件还慢"的常见原因**——100 个任务排在 4 个线程上串行跑。

**Node 20+ 的 `fs.promises` API 会自动用线程池**（并发读文件不会阻塞事件循环，但会占满线程池）。

### 什么会阻塞事件循环（**这是 Node 面试最高频的实际问题**）

| 阻塞源 | 例子 | 后果 |
|--------|------|------|
| **同步 I/O** | `fs.readFileSync`、`fs.writeFileSync` | 整个服务卡住 |
| **同步加密** | `crypto.pbkdf2Sync`、大文件的 `hashSync` | 同上 |
| **大 JSON 解析** | `JSON.parse(hugeString)` | 同上 |
| **CPU 密集计算** | 图像处理、加解密、大量正则匹配、压缩 | 同上 |
| **死循环** | `while(true){}` | 服务彻底挂死 |

**⚠️ 症状**：所有请求一起变慢、监控看到 event loop lag 飙升、连接堆积。

**检测**：

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
// 定期读取 h.mean / h.percentile(99)（纳秒）
```

**解法优先级**：

1. **改用异步 API**（`fs.promises.readFile` 而不是 `readFileSync`）——**首选，成本最低**；
2. **把重活切小块**，让出事件循环（用 `setImmediate` 分片）；
3. **扔给线程池**（`fs.promises` 类的 API 会自动这么做）；
4. **用 `worker_threads` 跑真正的 CPU 密集任务**；
5. **上多进程**（`cluster` 模块 / PM2 / Kubernetes 多副本）——**最通用的横向扩展方案**。

### `worker_threads` 详解

```ts
// main.ts
import { Worker } from 'node:worker_threads';

function runHeavyTask(data: number[]): Promise<number> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', {
      workerData: data,               // 传给 worker 的初始数据
    });
    worker.on('message', resolve);    // worker 发消息 → resolve
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`worker 退出，code=${code}`));
    });
  });
}
```

```js
// worker.js
import { parentPort, workerData } from 'node:worker_threads';

// ⚠️ 注意：worker 文件是独立的 JS 上下文，不能共享主线程的闭包变量
parentPort.postMessage(heavyCompute(workerData));
```

**`worker_threads` 的特点**：

| 维度 | 说明 |
|------|------|
| **真并行** | ✅ 每个 worker 有独立的 V8 隔离区和事件循环，**能真正利用多核** |
| **通信** | 通过 `postMessage` / `on('message')`（结构化克隆）或 `SharedArrayBuffer` + `Atomics`（共享内存） |
| **启动成本** | ⚠️ 约 **几十毫秒**，不是免费的——短任务用 worker 反而更慢 |
| **共享内存** | `SharedArrayBuffer` + `Atomics.wait` / `Atomics.notify` 实现零拷贝共享，但要自己管理同步 |
| **Node 版本** | Node 12+ 稳定（`Atomics.wait` 在 Node 16+ 才允许在主线程调用） |

**⚠️ worker 的关键限制**：

1. **worker 里不能直接访问主线程的变量**——只能通过 `postMessage` 传数据；
2. **每个 worker 有独立的内存**（不共享堆），所以大对象传递有序列化成本；
3. **worker 里也有自己的事件循环**，别把 I/O 任务扔进去（没必要，主线程的异步 API 已经不阻塞了）。

### 选型决策

| 场景 | 方案 |
|------|------|
| **网络/数据库请求** | 主线程的异步 API（**不要**用 worker） |
| **读文件** | `fs.promises`（走 libuv 线程池） |
| **CPU 密集计算（超过几十毫秒）** | **`worker_threads`** |
| **需要多实例扩展** | **多进程 / 多容器**（比 worker 更简单、更易扩容） |

**我的判断**：**`worker_threads` 是个"重武器"，用之前先确认任务确实是 CPU 密集且耗时可观。** 90% 的 Node 性能问题用"改同步 API 为异步 API"就解决了。

**追问：`cluster` 模块和 `worker_threads` 有什么区别？**
→ `cluster` 是**多进程**（每个进程一个独立的 Node 实例，有独立的内存），靠主进程分发请求——**可以跨机器**（配合 PM2）。`worker_threads` 是**多线程**（同一进程内），共享内存但不能跨机器。**要横向扩展用 cluster/多容器，要共享内存用 worker_threads。**

**追问：Node 什么时候会变成多线程？**
→ Node 从 v0.10 就有 worker_threads 的前身（早期叫 Web Workers）了，但长期不推荐（性能差），到 Node 12 才稳定重写。**Node 一直的设计取向是"单线程 + 异步 I/O"，多线程只是特殊场景的补充。**

---

## Q8. 什么是流（Stream）和背压（backpressure）？为什么大文件不能 `readFile`？

**答：**

**流是"分块处理数据"的抽象；背压是"下游处理不过来时，通知上游停下来"的机制。没有背压的流会无限缓冲，把内存吃光。**

### 为什么不能 `readFile`

```ts
// ❌ 2GB 文件
const buf = await fs.readFile('./huge.iso');
```

**内存里会同时出现两份 2GB**（Buffer + 字符串），**直接把进程 OOM 掉**。

```ts
// ✅ 流式处理，内存占用与文件大小无关
const stream = fs.createReadStream('./huge.iso');
stream.on('data', (chunk) => {
  // chunk 是个 Buffer，默认 64KB
});
```

**内存占用是 O(chunk 大小)，不是 O(文件大小)。**

### Node 的四种流

| 流 | 方向 | 典型场景 |
|----|------|---------|
| **Readable** | 数据源 → 消费者 | 读文件、HTTP 请求体 |
| **Writable** | 消费者 → 数据源 | 写文件、HTTP 响应 |
| **Duplex** | 双向独立 | TCP socket |
| **Transform** | 读 → 处理 → 写 | gzip、加密、格式转换 |

### 背压的机制

**场景**：上游（读文件）产出速度 100MB/s，下游（写数据库）处理速度 10MB/s。

- **没有背压**：上游不停写，数据堆在下游的内存缓冲区里 → 内存爆掉；
- **有背压**：下游说"我满了，别给我了"，上游暂停读取 → 数据留在上游的缓冲区里 → 内存占用稳定。

**Node 的实现机制**：

```ts
// 手动控制背压：写入方返回 false 表示"缓冲区满了"
const writable = fs.createWriteStream('out.txt');

readable.on('data', (chunk) => {
  if (!writable.write(chunk)) {
    readable.pause();          // 下游满了 → 暂停上游
    writable.once('drain', () => readable.resume());  // 下游空了 → 恢复
  }
});
```

**但更好的做法是直接用 `pipe`，Node 会自动处理背压**：

```ts
// ✅ 自动背压
fs.createReadStream('in.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('out.txt.gz'));
```

`pipe` 内部就是上面那套 pause/resume 逻辑。

### ⚠️ `pipe` 的错误处理是经典坑

**`pipe` **不会自动转发错误**。源流报错时，目标流不知道：

```ts
// ❌ 错误被吞掉，可能导致进程挂起
src.pipe(dest);
```

**正确写法**：

```ts
import { pipeline } from 'node:stream/promises';

// ✅ 自动处理错误传播、资源清理
await pipeline(
  fs.createReadStream('in.txt'),
  zlib.createGzip(),
  fs.createWriteStream('out.txt.gz'),
);
```

**`pipeline`（来自 `node:stream/promises`）是现代 Node 的标准做法**：
- 自动传播错误；
- 自动销毁所有流（避免泄漏 fd）；
- 支持异步管道（`await` 完成后流都结束）。

**⚠️ 老代码里的 `stream.finished` + `promisify` 就是在模拟 `pipeline`**——新代码直接用 `pipeline`。

### Node 流的两种模式

| 模式 | API | 特点 |
|------|-----|------|
| **流式（streaming）** | `on('data')` | 边读边处理，内存恒定，但控制流靠事件 |
| **异步迭代器** | `for await (const chunk of stream)` | **现代推荐**，支持 `break` 提前退出、可读性更好 |

```ts
// ✅ 现代写法
const stream = fs.createReadStream('in.txt');
let total = 0;
for await (const chunk of stream) {
  total += chunk.length;
  if (total > LIMIT) break;    // break 会自动销毁流 ⚠️ 这是异步迭代器相对 on('data') 的优势
}
```

**⚠️ 但注意**：`for await` 是**串行**的（每次等一个 chunk），对单流来说没问题（流本身就是有背压的）。**不要用 `for await` 去做"多个任务并行"**（见 Q4）。

### 背压的"手动 vs 自动"

| 场景 | 做法 |
|------|------|
| 简单的文件/网络流串联 | **`pipe`** 或 **`pipeline`** |
| 需要中间干预（过滤、限流） | `pipe` + 手写 pause/resume，或用 Transform 流 |
| 需要精确控制读取速度 | 手写 `write()` 返回值 + `drain` 事件 |
| 从 HTTP 响应读 | `for await (const chunk of response.body)` |

**追问：Web Streams API（`ReadableStream`）和 Node Stream 有什么区别？**
→ Web Streams 是**浏览器标准**（`fetch` 的响应体就是它），现在 Node 18+ 也支持。两者 API 不同（`ReadableStream` 用 `getReader()`，Node Stream 是 EventEmitter）。**Node 内置模块（`fs`、`http`）用 Node Stream；`fetch` / `Response.body` 用 Web Streams。** 需要互转时用 `Readable.fromWeb()` / `Readable.toWeb()`。

**追问：`highWaterMark` 是什么？**
→ 每个流的**缓冲区高水位标记**（默认 64KB）。它是"背压阈值"——缓冲区里待写数据超过它，`write()` 就返回 `false`。**它不是硬性上限**，你可以手动调大来换吞吐（代价是内存）。**读流的 `highWaterMark` 和写流的是两个独立配置。**

---

## Q9. Node 的定时器为什么不精确？`setTimeout(fn, 0)` 和 `setImmediate` 有什么区别？

**答：**

**Node 的定时器保证"不早于"给定延迟执行，但不保证精确；且定时器回调的**实际**触发时间会被事件循环的其他工作推迟。**

### 为什么不准

```ts
setTimeout(() => console.log('timer'), 100);
```

**它保证的不是"100ms 后执行"，而是"至少 100ms 后，且要等事件循环空闲"。** 如果这期间有同步代码执行了 500ms，timer 就会在 600ms 左右才跑。

**常见误解**：`setTimeout(fn, 100)` 不是"精确 100ms"。它还要加上**定时器本身的注册开销**和**事件循环轮转的延迟**。Node 内部用 1ms 的列表按到期时间排序（Node ≥ 11 改进了定时器实现），但仍是"最少等待这么久"。

### `setTimeout(fn, 0)` vs `setImmediate`

| | `setTimeout(fn, 0)` | `setImmediate(fn)` |
|---|---------------------|-------------------|
| **所属阶段** | timers | check |
| **执行时机** | 下一个 timers 阶段 | 当前轮次的 check 阶段 |
| **相对顺序** | 取决于当前处于哪个阶段 | 同上 |
| **精度** | 有最小延迟（Node 中至少 1ms） | 同阶段内按注册顺序立即执行 |

**⚠️ 关键结论**：`setTimeout(0)` 和 `setImmediate` 的顺序**在 Node 主模块里是不确定的**。

**原因**：Node 启动时执行主模块代码，此时事件循环**还没开始**。定时器的到期时间（至少 1ms 后）可能在这期间就已经到了，于是 timers 阶段先跑；也可能没到，就先跑到 check 阶段。

**但在 I/O 回调里，顺序是确定的**：

```ts
fs.readFile('file.txt', () => {
  setTimeout(() => console.log('timeout'), 0);   // 第二
  setImmediate(() => console.log('immediate'));   // 第一（确定）
});
```

**原因**：I/O 回调在 poll 阶段执行，紧接着就是 check 阶段，所以 `setImmediate` 一定先跑。

### 面试怎么答

> **`setTimeout(0)` 和 `setImmediate` 的顺序在顶层代码里是不确定的，在 I/O 回调里是 `setImmediate` 先。** 根本原因是事件循环的阶段顺序：I/O 回调在 poll 阶段，下一个阶段就是 check（`setImmediate`），而 timers 阶段在下一轮，所以 I/O 里 `setImmediate` 一定先。而在顶层，主模块执行时事件循环尚未启动，谁先取决于定时器到期时机。

**这个不确定性不是 bug，是设计使然**（Node 官方文档明确说明）。

### 定时器的其他坑

**1. 定时器会保持事件循环存活**：

```ts
setInterval(() => tick(), 1000);
server.close();
// ⚠️ 进程不会退出，因为还有活跃的定时器
```

**解法**：`timer.unref()` —— 让定时器**不阻止事件循环退出**。

```ts
const timer = setInterval(() => tick(), 1000);
timer.unref();    // 进程可以在没有其他工作时退出
```

**2. 嵌套定时器会被最小化到 1ms**：

```ts
setTimeout(() => setTimeout(() => console.log('x'), 0), 0);
// 两个都至少 1ms
```

**3. `setTimeout(fn, 0)` 不等于"立刻"**，它至少要等一个 timers 阶段（≈1ms+）。

**追问：`setTimeout` 的回调里能写同步的重活吗？**
→ 能，但**这正是阻塞事件循环的典型方式**。回调是同步执行的，重活会把整个服务卡住。**任何 `setTimeout` / `setInterval` / 事件回调里的重活都应该考虑往 `worker_threads` 挪。**

**追问：`setInterval` 有什么坑？**
→ **它不保证精确间隔**。如果回调执行耗时 100ms 而间隔是 50ms，Node **不会并发执行**（避免重叠），但下一次触发时间会**顺延**——实际间隔变成 100ms+。**密集的重复任务用 `setTimeout` 递归比 `setInterval` 更可控**（可以在每次执行后重新计算下次时间）。

---

## Q10. 异步编程里最常见的几个 bug 是什么？怎么避免？

**答：**

**我按"面试中被问到的概率"排个序，前四个是必答。**

### Bug 1：`forEach` 里用 `async` 不等待

```ts
// ❌ 三个请求串行发出，但都不等
items.forEach(async (item) => {
  await process(item);
});
console.log('done');   // ❌ 这里立刻就打印了

// ✅ 用 for...of 或 Promise.all
for (const item of items) {
  await process(item);   // 串行但正确
}
// 或
await Promise.all(items.map(item => process(item)));   // 并行
```

**根因**：`forEach` 的回调返回的 Promise **不被 `forEach` 等待**。**`forEach` 是为同步遍历设计的。**

### Bug 2：`for...of` 里漏了 `await`（看起来像串行其实不是）

```ts
// ❌ 以为是串行，实际并行
for (const item of items) {
  process(item);      // 没有 await！返回的 Promise 被丢弃
}
```

**现象**：结果和 `Promise.all` 一样是并行的，但**错误处理完全失效**（rejection 变成未处理 rejection）。

### Bug 3：漏掉 `await`（fire-and-forget）

```ts
// ❌ 函数返回 Promise，但调用方没 await 也没 catch
async function save() { await db.insert(data); }

save();          // 出错时无人知道 → unhandled rejection
```

**修法**：要么 `await`，要么显式 `save().catch(handle)`。

### Bug 4：`Promise.all` 的快速失败丢结果

```ts
// ❌ 批量任务：一个失败，其他成功结果全丢
const results = await Promise.all(tasks.map(t => t()));

// ✅
const results = await Promise.allSettled(tasks.map(t => t()));
```

### Bug 5：循环里创建过多并发

```ts
// ❌ 1 万个并发请求 → 打爆连接池 / 触发限流
await Promise.all(ids.map(id => fetchOne(id)));

// ✅ 限制并发数（用 p-limit 或手写 worker pool）
```

### Bug 6：错误被吞掉

```ts
// ❌ catch 了但什么都不做
try { await risky(); } catch (e) {}

// ✅ 至少要处理
try { await risky(); } catch (e) {
  logger.error(e);
  throw new AppError('操作失败', { cause: e });   // ⚠️ 带上 cause
}
```

**⚠️ 特别提醒**：捕获后**要么处理、要么重抛**。空的 `catch` 块是线上 bug 的温床。

### Bug 7：`Promise` 忘了 `await` 却用了 `.then` 的返回值

```ts
// ❌
Promise.resolve(1).then(v => v + 1).then(v => console.log(v));  // 其实是对的
// ❌ 但这样：
async function f() {
  Promise.resolve(1).then(v => v + 1);   // 返回的 Promise 被丢弃，f 立即返回
}
```

**规则**：**函数返回 Promise 时，要么 `await` 它，要么明确 `void` 掉并挂 catch。**

### Bug 8：事件回调里的异常导致进程崩溃

```ts
// ❌ 抛错直接崩进程（uncaught exception）
emitter.on('data', (chunk) => {
  JSON.parse(chunk);    // 抛错 → 进程退出
});

// ✅
emitter.on('data', (chunk) => {
  try {
    JSON.parse(chunk);
  } catch (e) {
    logger.warn('解析失败', e);
  }
});
```

### 防御清单（实战用）

1. **ESLint 规则**：`@typescript-eslint/no-floating-promises`、`no-misused-promises`——**这两个规则能自动抓出上面一半的 bug**，务必开；
2. **`useUnknownInCatchVariables`**（`strict` 包含）——强制你在 catch 里收窄；
3. **全局兜底**：`process.on('unhandledRejection')` + `process.on('uncaughtException')` 上报；
4. **超时兜底**：所有外部调用加超时（`AbortSignal.timeout(ms)`，Node 17.3+）；
5. **监控**：event loop lag、活跃句柄数、未处理 rejection 计数。

**追问：怎么防止一个请求把整个进程拖死？**
→ ① 加超时（`AbortSignal.timeout`）；② 用 `Promise.race` + 超时兜底；③ 对 CPU 密集部分用 `worker_threads`；④ **最外层加一个看门狗**（`--max-old-space-size` 限制堆 + 进程守护（PM2 / systemd）自动重启）。**工程上永远是"多层防御"，没有单点方案。**

**追问：Node 里怎么主动取消一个正在进行的请求？**
→ **`AbortController`**（Node 15+ 全局可用）：

```ts
const controller = new AbortController();
setTimeout(() => controller.abort(), 1000);

try {
  const res = await fetch('https://example.com', { signal: controller.signal });
} catch (err) {
  if (err.name === 'AbortError') {
    // 超时取消
  }
}
```

**这是唯一能真正"取消"的方式**——`Promise.race` 只是"不等了"，底层任务还在跑。

---

## 本篇自测

- [ ] Node 事件循环的六个阶段，微任务和宏任务的执行规则
- [ ] `setTimeout(0)` 和 `setImmediate` 的顺序，为什么顶层不确定、I/O 里确定
- [ ] Promise 的三种状态，`.then` 链的三条透传规则
- [ ] `all` / `allSettled` / `race` / `any` 的语义差异和选型
- [ ] 串行 vs 并行的性能差距，`for await` 为什么不是并行
- [ ] 未处理 rejection 的触发场景，Node 15+ 的行为变化
- [ ] `await` 的本质，`try/catch` 和 `.catch()` 的区别，`Error.cause`
- [ ] Node 的线程模型，什么会阻塞事件循环，`worker_threads` 的取舍
- [ ] 什么是背压，`pipe` 和 `pipeline` 的区别
- [ ] 异步编程的八个常见 bug 及防御手段

---

## 导航

[[typescript/interview/README|TypeScript 面试题库首页]] · 上一篇：[[typescript/interview/03-工程化与工具链]] · 下一篇：返回：[[typescript/interview/README|TypeScript 面试题库首页]]
