---
tags: [typescript, 中段, Node, 异步, Promise]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# Promise 与异步流

> **什么时候该读**：你写了 `await`，但不确定它到底在等什么；或者线上出现「接口偶发超时」「Promise 报错但进程直接挂了」「请求没并行、慢得离谱」。
> **阅读门槛**：读完 [[typescript/02-中段阶段-专题精讲/Node生态与异步/1-Node运行时与HTTP服务]]，知道 Node 是单线程事件循环 + 线程池。本篇会大量使用 `async` / `await` / `Promise`，如果你还没写过，先看第 2.1 节的语法糖拆解再往下。

## 学习目标

读完本篇，你应当能够：

1. 说清 Promise 的三个状态和它们的转换规则，解释「Promise 一旦 settle 就不可逆」
2. 正确使用 `.then` / `.catch` / `.finally` 写出链式调用，并说清 `.finally` 不会改变结果值
3. 判断一段代码里 `await` 的**实际作用范围**，解释「忘记 `await` 会让异步错误被吞掉」
4. 在 `all` / `allSettled` / `race` / `any` 之间做出正确选择，并说出各自的失败语义
5. 把一段串行 `for` 循环改写成并行，并说清性能差异来自哪里
6. 用 `for await...of` 消费异步流，理解背压（backpressure）为什么存在
7. 解释「未处理的 Promise rejection」为什么会终止进程，并给出兜底方案
8. 自己实现一个**并发池**，避免 `Promise.all` 打爆下游

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/02-中段阶段-专题精讲/Node生态与异步/1-Node运行时与HTTP服务]] | 事件循环与微任务队列的关系；`node:http` 的 `req` / `res` 是流 |
| [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.2-函数与参数]] | 回调函数、箭头函数、默认参数——Promise 的 `.then` 接收的就是回调 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]] | `Promise<T>` 的 `T` 是什么；`catch` 里的错误是 `unknown`（第 3 篇详解） |
| [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]] | 顶层 `await` 依赖 ESM；本篇的示例都用 ESM 写法 |

---

## 一、Promise：代表「未来某个时刻会有的结果」

### 1.1 ⭐ 三个状态

**Promise** 是一个代表「异步操作最终结果」的对象。它的核心是**一个不可逆的状态机**：

```
                    ┌──────────────┐
      new Promise(  │   pending    │  等待中（初始状态）
          executor  └──────┬───────┘
                 │          │
      调用 resolve()│        │调用 reject()
                 ▼          ▼
          ┌────────────┐  ┌────────────┐
          │ fulfilled  │  │ rejected   │
          │ 已成功     │  │ 已失败     │
          └────────────┘  └────────────┘
                 ▲          ▲
                 └────┬─────┘
                      │
              ⭐ 一旦 settle（成功或失败），状态永久固定，不可回退、不可重置
```

| 状态 | 含义 | 之后能拿到什么 |
|------|------|---------------|
| **`pending`** | 等待中 | 还不知道结果 |
| **`fulfilled`**（已兑现 / 成功） | 操作成功 | 成功值（value） |
| **`rejected`**（已拒绝 / 失败） | 操作失败 | 失败原因（reason），通常是一个 `Error` |

⭐ **关键性质：状态不可逆。** 一个 Promise 一旦 `fulfilled`，就永远 fulfilled；你之后再调用 `reject()` 不会有任何效果。这和「回调可以被调用多次」的直觉相反。

```ts
// 创建 Promise：new Promise 时传一个执行器函数（executor）
// executor 会被**同步**执行一次，里面调用 resolve 表示成功、reject 表示失败
const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

const p = new Promise<number>((resolve, reject) => {
  setTimeout(() => {
    if (Math.random() > 0.5) resolve(42);       // → fulfilled(42)
    else reject(new Error("随机失败"));          // → rejected(Error)
  }, 100);
});
```

> **对照 Python**：Python 的 `asyncio.Future` / coroutine 是同类的概念，但 Python 没有一个「对象自带状态机、只能 resolve 一次」的 `Promise` 类型——`asyncio.Future` 最接近，可以手动 `set_result()`（也可以多次调用，最后一次生效）。

### 1.2 ⭐ `.then` / `.catch` / `.finally` 链式调用

Promise 的结果不会「自动跑」，必须用 `.then` 等方法**订阅**它。

| 方法 | 什么时候被调用 | 回调参数 | 返回值 |
|------|--------------|---------|-------|
| **`.then(onFulfilled, onRejected)`** | Promise 成功时调 `onFulfilled`；失败时调 `onRejected`（可省略） | 成功值 / 失败原因 | **返回一个新的 Promise**，这是链式调用的关键 |
| **`.catch(onRejected)`** | 等价于 `.then(undefined, onRejected)` | 失败原因 | 新的 Promise |
| **`.finally(onSettled)`** | 成功或失败**都会**调用（必定执行） | 无（除非回调返回 Promise） | ⚠️ **不改变结果**：上一步的值/错误原样透传 |

⭐ **最关键的一点：`.then` 总是返回新 Promise**。所以：

- 回调里 `return` 一个值 → 新 Promise 以该值 fulfilled；
- 回调里 `throw` 一个错误 → 新 Promise 以该错误 rejected；
- 回调里 `return` 另一个 Promise → 新 Promise **跟随**它的结果（自动"展开"）。

这就是链式调用的全部魔力。

```ts
async function getUserName(id: string): Promise<string> {
  const user = await fetchUser(id);          // 失败会直接抛出去
  return user.name;
}

getUserName("u1")
  .then((name) => {
    console.log("拿到名字:", name);
    return name.length;                     // ⭐ return 的值传给下一个 then
  })
  .then((len) => {
    console.log("名字长度:", len);
  })
  .catch((err: unknown) => {                // ⭐ 中间任何一步失败都会跳到这里
    console.error("出错了:", err);
  })
  .finally(() => {
    console.log("无论成败都会执行（适合做清理）");  // ⚠️ 这里 return 的值会被丢弃
  });
```

⚠️ **`.finally` 的返回值会被忽略**。在 `.finally` 里 `return 某个值`，那个值**不会**成为最终结果。想在收尾时改变结果，必须用 `.then`：

```ts
// ❌ 错误：想包装成 Result，却写在 finally 里
Promise.resolve(1).finally(() => ({ ok: true }));   // 结果仍然是 1，不是 { ok: true }

// ✅ 正确：用 then 承接
Promise.resolve(1).then((v) => ({ ok: true, value: v }));
```

### 1.3 Promise 回调什么时候执行

这是理解 Node 事件循环的关键一环。Node 的任务队列分两种：

| 队列 | 名字 | 里面装什么 | 清空时机 |
|------|------|-----------|---------|
| **微任务队列（microtask queue）** | microtask | **Promise 的 `.then` / `.catch` / `.finally` 回调**、`queueMicrotask` | ⭐ **当前宏任务执行完就立刻全部清空**，然后才进入下一个宏任务 |
| **宏任务队列（macro task）** | macrotask | `setTimeout` / `setInterval` 回调、IO 回调、`setImmediate` | 一次只取一个 |

```ts
console.log("1 同步代码");

setTimeout(() => console.log("5 setTimeout（宏任务）"), 0);
setImmediate(() => console.log("6 setImmediate（宏任务）"));
Promise.resolve().then(() => console.log("3 Promise.then（微任务）"));

console.log("2 同步代码");

// 输出顺序：
// 1 同步代码
// 2 同步代码
// 3 Promise.then（微任务）    ← 微任务队列在第一个宏任务之前就被清空
// 5 setTimeout（宏任务）
// 6 setImmediate（宏任务）
```

⭐ **一句话**：**微任务插队**——只要当前同步代码跑完，所有已经排队的 Promise 回调会在进入下一个宏任务之前**全部**执行完。

> 这解释了上一篇那幅事件循环图里「微任务队列」的位置：它在 `poll`（拿 IO 事件）之前被清空，所以一个 IO 回调里排队的 Promise 回调会在**下一次** poll 之前跑完。

## 二、`async` / `await` ⭐

### 2.1 本质：语法糖，不是新机制

**`async`** 修饰一个函数，表示「这是个异步函数」；**`await`** 表示「等这个 Promise 出结果」。

```ts
async function f(): Promise<number> { return 1; }
```

⭐ **完全等价于**：

```ts
function f(): Promise<number> {
  return Promise.resolve(1);
}
```

也就是说：**`async` 函数的返回值一定是一个 Promise**，哪怕你 `return` 的是普通值。

```ts
async function f() { return 1; }
const r = f();          // r 是 Promise<number>，不是 number！
console.log(r instanceof Promise);   // true
```

⭐ **推论**：`await` 一个**非 Promise** 值时，JS 也会把它包成 Promise 再等，所以 `await 42` 只是「多等一个微任务」，不会卡住。

### 2.2 ⭐ `await` 的作用范围

**`await` 会「让出」当前函数剩余的执行权**，回到事件循环，让别的任务先跑。但它**只让出 `await` 所在的这一层函数**——外层同步代码继续往下走。

```ts
async function a(): Promise<string> {
  console.log("a1");
  await fetchUser("u1");          // ⭐ 这里让出执行权
  console.log("a2");              // 恢复后执行
  return "done";
}

async function b(): Promise<void> {
  console.log("b1");
  await a();                      // ⭐ 这里也让出
  console.log("b2");
}

console.log("main1");
b();
console.log("main2");
```

```
main1
b1
a1
main2          ← main2 在 await 处同步打印，不等 a/b 完成
a2             ← 微任务恢复
b2
```

⚠️ **推论**：`await` 之后的代码，执行时机**不早于**当前同步代码块的末尾。也就是说，`await` **不能**用来「在同一个同步块里等一个结果再用」——想同步拿到结果，只能 `await`（要求外层也是 async）或改用回调。

### 2.3 ⚠️ 忘记 `await` 会让异步错误被吞掉

这是本篇最重要的一个坑。

```ts
// ❌ 危险：saveUser 返回 Promise，这里既没 await 也没 catch
async function handler(): Promise<void> {
  saveUser({ id: 1 });     // ⚠️ 只是「发起」了调用，错误变成了未处理的 rejection
  console.log("保存完成"); // ⚠️ 这行会在保存**真正成功之前**就打印
}
```

三个后果：

1. **错误被吞掉**——`saveUser` 内部的失败变成「未处理的 Promise 拒绝」，不会走到你的 `try/catch`；
2. **时序错误**——后续代码在异步操作完成前就继续跑了；
3. **进程可能被直接干掉**——Node 默认在未处理的 rejection 发生时终止进程（第八节）。

```ts
// ✅ 正确：await 让错误能被 try/catch 捕获
async function handler(): Promise<void> {
  try {
    await saveUser({ id: 1 });
    console.log("保存完成");   // ⭐ 现在这行确实是保存成功之后才打印
  } catch (err) {
    console.error("保存失败:", err);
  }
}
```

⭐ **记住这条规则**：**每一个返回 Promise 的调用，要么 `await` 它，要么 `.catch()` 它。** 二选一，不能都不做。

```ts
// ❌ 三种典型错误写法
saveUser(u);                        // 忘了 await
await saveUser(u);                  // await 了，但外面没有 catch
const x = saveUser(u);              // 把 Promise 当成结果用了（x 是 Promise 不是 User）
```

⚠️ **TypeScript 帮不了你**：TS **不会**警告「你忘了 await」——`saveUser(u)` 作为一条语句在 TS 看来完全合法。（ESLint 的 `@typescript-eslint/no-floating-promises` 规则能查这个，属于 lint 范畴。）

> **对照 Python**：Python 里 `await` 是**唯一**的异步语法，忘了写 `await` 会直接报 `SyntaxError: 'await' outside async function` 或 `coroutine was never awaited` 警告。**TypeScript 不会**——这是 TS 读者最容易从 Python 带过来的坏习惯。

## 三、⭐ Promise 组合子

「组合子」= 把多个 Promise 组合成一个 Promise 的静态方法，都挂在 `Promise` 上。

| 方法 | 语义 | 什么时候用 |
|------|------|-----------|
| **`Promise.all`** | 全部成功才成功，**任一失败立刻失败** | 相互独立、都必须成功 |
| **`Promise.allSettled`** | 全部结束（不管成败），返回成功/失败明细 | 部分失败可容忍 |
| **`Promise.race`** | **第一个 settle 的赢**（成功失败都算） | 超时控制 |
| **`Promise.any`** | **第一个成功的赢**（全失败才失败） | 多路兜底 |

### 3.1 `Promise.all`

```ts
const [user, posts, settings] = await Promise.all([
  fetchUser("u1"),
  fetchPosts("u1"),
  fetchSettings("u1"),
]);
// 三个都成功了才走到这里；返回值按**输入顺序**排列，不按完成顺序
```

⚠️ **注意两点**：

1. **返回值顺序 = 输入顺序**，不是完成顺序。如果你依赖的是"谁先回来"，需要自己处理。
2. **失败语义**：任意一个 reject，`all` 立刻 reject，且 reject 的是**第一个**失败者的错误（不是 `AggregateError`）。

### 3.2 `Promise.allSettled`

```ts
const results = await Promise.allSettled([
  fetchUser("u1"),
  fetchPosts("u1"),
  fetchSettings("u1"),
]);

for (const r of results) {
  if (r.status === "fulfilled") {
    console.log("成功:", r.value);           // ⭐ fulfilled 分支的 value
  } else {
    console.error("失败:", r.reason);        // ⭐ rejected 分支的 reason
  }
}
// ⭐ 它**永远不 reject**，所以不需要 try/catch
```

⭐ **`allSettled` 的结果类型是判别联合**（discriminated union，见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.2-联合类型与字面量类型]]）：`{ status: "fulfilled"; value: T } | { status: "rejected"; reason: unknown }`。用 `status` 收窄，TS 就能自动知道 `value` 和 `reason` 分别存在。

### 3.3 `Promise.race`

```ts
// ⭐ 超时控制的经典写法
function withTimeout<T>(p: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    p,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`超时 ${ms}ms`)), ms)
    ),
  ]);
}

const data = await withTimeout(fetchBigReport(), 3000);
```

⭐ **注意 `race` 的语义是「第一个 settle」**：如果 `p` 成功在前 → 成功；如果**超时在前** → 超时错误胜出（哪怕 `p` 后来成功了，那份结果被丢弃）。这不是「第一个成功的赢」，那是 `any`。

⚠️ **被 race 输掉的那个 Promise 不会被取消**——它仍在后台跑，仍然会消耗资源。

### 3.4 `Promise.any`

```ts
// ⭐ 多路兜底：主备切换
const data = await Promise.any([
  fetchFromPrimary(),
  fetchFromBackup1(),
  fetchFromBackup2(),
]);
// 第一个**成功**的胜出；全部失败才 reject，错误是 AggregateError
```

⭐ **`any` 与 `all` 的对比是本节最值得记的**：

| | `Promise.all` | `Promise.any` |
|---|---|---|
| 成功条件 | **全部**成功 | **任意一个**成功 |
| 失败条件 | **任意一个**失败 | **全部**失败 |
| 失败时抛什么 | 第一个失败者的 error | `AggregateError`（里面装所有错误） |

### 3.5 一张表记住四个方法

```
         第一个 settle           全部 settle
        ├──────────────────────┼──────────────────────┤
成功优先 │  race  (谁先谁赢)     │  all    (全成功才成功)
失败优先 │  any   (第一个成功)   │  allSettled (永不失败)
```

> **对照 Python**：`Promise.all` ≈ `asyncio.gather`（但 `gather` 默认首个异常即返回，且不自动取消其余任务）；`Promise.allSettled` ≈ `asyncio.gather(..., return_exceptions=True)`；`Promise.race` ≈ `asyncio.wait(..., return_when=FIRST_COMPLETED)`；`Promise.any` 在 Python 3.11+ 有对应的 `asyncio.wait` 组合写法。

```python
# Python：等价物长这样，注意 gather 的异常语义和 Promise.all 并不完全一样
import asyncio

async def main():
    # ✅ 并行：三个一起发
    user, posts = await asyncio.gather(fetch_user("u1"), fetch_posts("u1"))

    # ⭐ 部分失败可容忍时用 return_exceptions=True（≈ Promise.allSettled）
    results = await asyncio.gather(
        fetch_user("u1"), fetch_posts("u1"), return_exceptions=True
    )
    # results 里成功的是值，失败的是 Exception 实例（不是 allSettled 那种 {status,...} 结构）

asyncio.run(main())
```

⚠️ **一处关键差异**：`asyncio.gather` 在首个异常时就把异常抛出去，但**其余协程不会被取消**，会继续在后台跑——和 `Promise.all` 一样是「快速失败但不取消」。

## 四、⭐ 串行 vs 并行：最容易犯的性能错误

```ts
// ❌ 串行：total ≈ 100 + 200 + 300 = 600ms
const start = Date.now();
for (const id of ["u1", "u2", "u3"]) {
  const user = await fetchUser(id);      // ⚠️ 每次都等上一个完成
  console.log(user);
}
console.log(Date.now() - start);          // ≈ 600
```

```ts
// ✅ 并行：total ≈ max(100, 200, 300) = 300ms
const start = Date.now();
const users = await Promise.all(
  ["u1", "u2", "u3"].map((id) => fetchUser(id))   // ⭐ 先全部发起，再一起等
);
console.log(Date.now() - start);          // ≈ 300
```

⭐ **差异的根源**：`for` 循环里的 `await` 意味着「发一个请求 → 等它回来 → 再发下一个」；`Promise.all` 意味着「**三个请求同时发出去** → 一起等」。

⚠️ **写并行时最容易搞错的一点**：不要写成下面这样——

```ts
// ❌ 仍然是串行！map 是同步的，await 在循环体里
const users = await Promise.all(
  ids.map(async (id) => {
    const u = await fetchUser(id);       // ⭐ map 会同步执行完所有 callback
    return u;
  })
);
```

这段之所以能并行，是因为 `map` 的回调函数**没有被 await**——`map` 同步执行完，把 3 个 Promise 都创建出来了，`Promise.all` 再一起等。**关键是不要在 `map` 回调里对创建 Promise 的那一行加 `await`**，加了就把"发起"推迟到了"上一个完成之后"。

⭐ **更简洁的等价写法**（现代 TS / Node 20+）：`Promise.all(ids.map(fetchUser))`。

## 五、⚠️ `Promise.all` 的失败语义陷阱

```ts
const results = await Promise.all([
  fetchA(),   // 100ms 成功
  fetchB(),   // 50ms  失败 ⚠️
  fetchC(),   // 5000ms 成功
]);
```

**发生了什么**：

1. `Promise.all` 在 50ms 时**立刻 reject**，`await` 抛出，`results` 根本拿不到；
2. ⚠️ **但 `fetchA` 和 `fetchC` 仍在后台继续跑**——`Promise.all` **没有取消**它们的能力；
3. `fetchC` 跑了 5 秒，资源照占；如果它写的是数据库，**副作用已经发生了**。

⭐ **一句话**：`Promise.all` 的语义是「**快速失败**」，不是「**取消**」。

**补救手段**：

| 手段 | 做法 | 代价 |
|------|------|------|
| **`AbortController`** | 给每个请求传 `signal`，失败时 `controller.abort()` 把其余请求全部取消 | 需要下游支持 `AbortSignal`；被取消的请求通常没有"副作用已回滚"的保证 |
| **改用 `allSettled`** | 不再快速失败，等全部跑完再逐个判断 | 失去"快速返回"的好处，总耗时 = 最慢的那个 |
| **两阶段** | 先并发跑，失败后单独重试失败的 | 逻辑最复杂，但重试语义最清晰 |

```ts
// ⭐ 带取消的并行
async function fetchAll(ids: string[]): Promise<User[]> {
  const controller = new AbortController();
  try {
    return await Promise.all(
      ids.map((id) => fetchUser(id, { signal: controller.signal }))
    );
  } catch (err) {
    controller.abort();      // ⭐ 一失败就把其余全部取消
    throw err;
  }
}
```

## 六、⭐ 异步迭代器与 `for await...of`

**异步迭代器（async iterator）**：一个实现了 `Symbol.asyncIterator` 的对象。它的 `next()` 返回的是一个 **Promise**，需要 `await` 才能拿到 `{ value, done }`。

**`for await...of`**：遍历异步可迭代对象的语法糖。每一轮循环体结束后，会 `await` 一次 `next()`。

```ts
// ⭐ 自己实现一个异步迭代器
async function* countTo(max: number): AsyncGenerator<number> {
  for (let i = 1; i <= max; i++) {
    await sleep(100);      // 模拟"数据慢慢到"
    yield i;               // ⭐ yield 产生一个值，并暂停生成器
  }
}

for await (const n of countTo(3)) {
  console.log(n);         // 1, 2, 3（每次之间隔 100ms）
}
```

⭐ **典型用途：流式数据处理**。

```ts
// ⭐ 用 fetch 的流式响应逐块消费
const res = await fetch("https://example.com/big-file");
for await (const chunk of res.body!) {
  process.stdout.write(chunk);
  // ⭐ 逐块处理，内存占用与文件大小无关
}
```

> **对照 Python**：`for await...of` ≈ Python 的 `async for`；`async function*` + `yield` ≈ Python 的 `async def` + `yield`（异步生成器）。Python 的 `async for` 只能遍历 `__aiter__` / `__anext__` 协议的对象，JS 的 `for await` 同时支持同步可迭代对象和异步可迭代对象。

⭐ **关键区别（容易踩）**：`for await` **只等 `next()`，不等循环体**。循环体里的 `await` 会把控制权交出去，但**迭代器本身不会被并发推进**——所以消费是有序的、一次一个。

```ts
// ⚠️ 循环体里没有 await：会快速消费完
for await (const n of countTo(3)) {
  console.log(n);              // 三个几乎同时打印（因为 next() 不等循环体）
}
```

## 七、⭐ 背压（backpressure）

### 7.1 为什么需要背压

**问题**：生产者（producer）产生数据的速度 > 消费者（consumer）处理速度。如果不管，数据会**堆在内存里**，直到进程 OOM。

> **对照 Python**：Python 的 `asyncio` 里最常见的对应物是 `asyncio.Queue`——它自带容量限制，`put` 在队列满时会**等待**，这就是一种背压。

**背压（backpressure）**：让「下游处理不过来」这个信号**往上游传**，让上游**减速或暂停**。

```
无背压：  生产者 ──────────────▶ [ 内存缓冲区 ] ──▶ 消费者
                                      ↑ 越堆越大 → OOM

有背压：  生产者 ──▶ [缓冲区] ──▶ 消费者
                     ↑ 满了
              生产者被阻塞（等待）
```

### 7.2 Node 的流与 `pipeline`

**流（stream）**：Node 处理「数据分段到达」的抽象。有四种：**Readable**（可读流，如文件、HTTP 请求体）、**Writable**（可写流，如 HTTP 响应体、文件）、**Duplex**（双向，如 TCP socket）、**Transform**（转换流，如压缩、加密）。

⭐ **Node 的流天然内置背压**：当你用 `readable.pipe(writable)` 时，如果 `writable` 的内部缓冲满了，pipe 会**暂停** readable 的数据读取——**这就是自动背压**。

```ts
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

// ⭐ pipeline：把多个流串起来，自动处理背压 + 自动传播错误
await pipeline(
  createReadStream("big.log"),
  createGzip(),
  createWriteStream("big.log.gz")
);
```

⭐ **为什么一定要用 `pipeline`**：手动 `pipe` 串联时，**错误不会自动传播**——中间某个流出错，后面的流不会知道，最终表现为「数据悄悄少了一截」。`pipeline` 保证：任一环节出错 → 全部销毁 → `pipeline` 自身 reject。

⚠️ **手动 `pipe` 的背压陷阱**：

```ts
// ❌ 手动写入却不处理背压信号
for await (const chunk of source) {
  dest.write(chunk);              // ⭐ write() 返回 false 表示"缓冲区满了"
}
```

`writable.write()` 的返回值是**背压信号**：`true` 表示还能继续写，`false` 表示「缓冲区满了，别再写了，去等 `drain` 事件」。

```ts
// ✅ 尊重背压
import { once } from "node:events";

for await (const chunk of source) {
  if (!dest.write(chunk)) {
    await once(dest, "drain");    // ⭐ 等到能写了再继续
  }
}
```

## 八、⚠️ 未处理的 Promise rejection

### 8.1 为什么这是生产事故常见原因

**未处理的 rejection**：一个 Promise 被 reject 了，但**没有任何人接**（既没 `.catch()`，也没有被 `await`）。

Node 的默认行为（**Node 15+ 起**）：**打印错误并终止进程**。

```
[UnhandledPromiseRejection] Error: connect ECONNREFUSED 127.0.0.1:5432
    at TCPConnectWrap.afterConnect ...
Node.js v20.11.0
```

⭐ **为什么这是事故**：一个孤零零的 rejection 就能让整个服务进程死掉——所有在途请求被重置，K8s 判定容器异常并重启。**一个本该被忽略的、非关键的错误，杀掉了整个服务。**

⚠️ **Node 15 之前只是打印警告，Node 15+ 直接终止**。这意味着**升级 Node 大版本可能让你的服务从「偶尔报个错」变成「直接崩」**。

### 8.2 为什么会漏掉

```ts
// 1. 忘了 await 也没有 catch（最常见）
saveUser(u);

// 2. catch 里的代码自己又抛了错
promise.catch((err) => {
  logger.error(err);
  throw err;              // ⚠️ 这个新错误没人接 → 又是未处理 rejection
});

// 3. Promise.all 里一个失败，其余的 rejection 全部没人管
await Promise.all([a(), b(), c()]);   // b 失败 → all reject → a/c 的成功无所谓
//                                     但如果 a 或 c 之后也失败 → 未处理 rejection

// 4. 事件回调 / setTimeout 里直接写 async 函数
setTimeout(async () => {
  await doSomething();     // ⚠️ 这里的错误 nobody catches
}, 100);

// 5. 数组方法 + async 回调 —— ⚠️ 最隐蔽的一种
["a", "b", "c"].forEach(async (id) => {
  await save(id);          // ⚠️ forEach 丢弃了返回的 Promise，失败无人知晓
});
// ✅ 正确：换成 map + Promise.all，或用 for...of + try/catch
await Promise.all(["a", "b", "c"].map(async (id) => save(id)));

// 6. 入口 main() 没接住
main();                     // ⚠️ main 内部的 rejection 无人接
// ✅ 正确：main().catch(err => { log(err); process.exit(1); })
```

⭐ **第 5 条值得单独强调**：**`forEach` 不等待 async 回调**——它接收回调的返回值直接丢掉。所以 `forEach(async ...)` 是「看起来并发、实际错误处理全丢」的写法。**在 TS 里这是最常见的未处理 rejection 来源。**

### 8.3 三层兜底

```ts
// ① 全局兜底：把「崩溃」降级成「记录 + 优雅退出」
process.on("unhandledRejection", (reason) => {
  console.error("[unhandledRejection]", reason);
  // ⚠️ 关键决策：是继续跑，还是退出？
  //    推荐：记录到监控，然后走优雅停机（见第 1 篇第五节）
  shutdown("unhandledRejection");
});

process.on("uncaughtException", (err) => {
  console.error("[uncaughtException]", err);
  shutdown("uncaughtException");
});
```

⚠️ **兜底 handler 里的判断很关键**：

| 策略 | 什么时候用 | 风险 |
|------|-----------|------|
| **记录后退出**（推荐生产） | 未处理 rejection 通常意味着有 bug 或状态不一致，继续跑可能更糟 | 会有一次重启，但状态干净 |
| **记录后继续跑** | 确认这个错误完全无关紧要（比如某个非关键的后台任务） | ⚠️ 进程可能已经处于未定义状态，内存泄漏、状态错乱 |
| **完全忽略** | ❌ 不要这么做 | 你会永久失去这个信号，直到线上出事 |

⭐ **一个必须知道的机制**：`process.on("unhandledRejection", ...)` **一旦注册，就会覆盖 Node 的默认终止行为**——进程不再自动退出，而是由你的 handler 决定。这意味着：

- **注册了 handler 却没写退出逻辑** → 进程带着未知状态继续跑 → 内存泄漏、状态错乱，比直接崩溃更难查；
- **正确姿势**是 handler 里**记录 + 走优雅停机 + 退出**（见第 1 篇第五节的 `shutdown`）。

⚠️ **另一个常见误区**：以为「装上 handler 就修好了」。**handler 是安全网，不是修复**——它只是让你能优雅地崩溃并留下日志。真正的修复是去把那个「漏掉的 `await` / `catch`」找出来。

```bash
# 临时把行为改回 Node 14 的"只警告不退出"，方便在大代码库里定位问题
node --unhandled-rejections=warn app.js
```

| `--unhandled-rejections` 取值 | 行为 | 用途 |
|---|---|---|
| `throw` | **默认**，抛成未捕获异常并终止进程 | 生产环境 |
| `strict` | 即使已注册 handler 也抛 | 更严格，排查用 |
| `warn` | 只打警告，进程继续 | ⭐ 大代码库迁移期临时用 |
| `none` | 完全静默 | ❌ 只用于隔离调试 |

```ts
// ② 局部兜底：给可能失败的异步调用加超时 + 兜底
async function safeCall<T>(fn: () => Promise<T>, fallback: T): Promise<T> {
  try {
    return await fn();
  } catch {
    return fallback;
  }
}
```

```ts
// ③ 结构化兜底：用 .catch 明确接住
void saveUser(u).catch((err: unknown) => {
  logger.error({ err, userId: u.id }, "保存失败");
});
```

## 九、⚠️ 并发陷阱：需要「并发池」

### 9.1 问题

```ts
// ⚠️ 1000 个 id 一次性全打出去
const users = await Promise.all(ids.map((id) => fetchUser(id)));
```

**后果**：

- 瞬间 1000 个 TCP 连接 → 打爆本地 fd（文件描述符）上限；
- 打爆下游服务的连接池 / 限流阈值 → **你在给自己制造 429**；
- 数据库连接池被瞬间抽干 → **整站变慢**（不只是这个接口）。

⭐ **根因**：`Promise.all` **没有并发上限**。它只知道「全部启动」，不知道「下游能承受多少」。

### 9.2 自己实现并发池

```ts
/**
 * 并发池（concurrency pool）：同时最多跑 limit 个任务，其余排队。
 * @param items    待处理的数据
 * @param limit    最大并发数
 * @param worker   单个任务的处理函数
 */
async function mapWithLimit<T, R>(
  items: readonly T[],
  limit: number,
  worker: (item: T, index: number) => Promise<R>
): Promise<R[]> {
  if (limit < 1) throw new RangeError("limit 必须 >= 1");

  const results = new Array<R>(items.length);
  let cursor = 0;                                  // ⭐ 共享游标，谁空闲谁取下一个

  const run = async (): Promise<void> => {
    while (true) {
      const index = cursor++;                       // ⭐ 取号（同步，不 await）
      if (index >= items.length) return;            // 没活干了，退出循环
      results[index] = await worker(items[index]!, index);   // ⭐ 这里才 await
    }
  };

  const workers = Array.from(
    { length: Math.min(limit, items.length) },
    () => run()
  );
  await Promise.all(workers);
  return results;
}
```

⭐ **两个设计要点**：

1. **游标 `cursor++` 是同步的**——JS 单线程 guarantees 它不会被打断，所以天然无锁；
2. **每个 worker 是一个「取号 → 干活 → 再取号」的循环**——`limit` 个 worker 天然形成并发上限。

```ts
// 使用
const users = await mapWithLimit(ids, 10, (id) => fetchUser(id));
// 任何时刻最多 10 个请求在飞
```

⚠️ **取舍**：

- 库现成的有 `p-limit`（NPM）、`Bottleneck`（功能更全）；
- **但队列系统（RABBITMQ / Kafka / BullMQ）本身就有 worker 数量控制**——在那种架构里，并发池通常不需要你手写。

## 陷阱

1. **⚠️ 以为「`.then` 回调立刻执行」**——它进的是**微任务队列**，要等当前同步代码跑完才执行；不是同步调用；
2. **⚠️ 以为「`await` 之后的代码马上执行」**——`await` 让出的是**这一层 async 函数**，外层同步代码照跑；顺序很容易想错；
3. **⚠️ 忘记 `await`**——错误变成未处理的 rejection（**可能直接杀掉进程**），而且后续代码在异步操作完成前就跑了；TS 编译器**不会**提醒你；
4. **⚠️ 以为 `await` 一个非 Promise 会卡住**——不会，只是多等一个微任务；
5. **⚠️ 在 `for` 循环里 `await`**——这是**串行**，性能可能差好几倍；并行的关键是「先全部发起，再一起等」；
6. **⚠️ 在 `map` 的 async 回调里对创建 Promise 的那行加 `await`**——看起来写了 `Promise.all`，实际还是串行；
7. **⚠️ 以为 `Promise.all` 会取消其余请求**——它**快速失败**但**不取消**；其他请求继续跑，副作用照发生；要取消得自己用 `AbortController`；
8. **⚠️ 以为 `Promise.all` 的结果按完成顺序排列**——它按**输入顺序**排列；
9. **⚠️ 混淆 `race` 和 `any`**——`race` 是「第一个 settle」（成功失败都算），`any` 是「第一个成功」；
10. **⚠️ 以为 `allSettled` 的失败会抛出去**——它**永不 reject**；要区分成功失败必须检查 `status`；
11. **⚠️ 在 `.finally` 里 `return` 来改变结果**——`finally` 的返回值被丢弃，结果不变；
12. **⚠️ 以为 `Promise.any` 全部失败时抛的是第一个错误**——它抛 `AggregateError`；
13. **⚠️ 用 `Promise.all` 打爆下游**——没有并发上限，会打爆 fd、连接池、触发限流；需要**并发池**；
14. **⚠️ 忽略背压**——`dest.write()` 返回 `false` 时不等待 `drain`，内存会被撑爆；链式处理用 `pipeline`；
15. **⚠️ 手动 `pipe` 串联多个流**——错误不会自动传播，会出现"数据悄悄少一截"；用 `node:stream/promises` 的 `pipeline`；
16. **⚠️ 认为「未处理的 rejection 只是打个警告」**——**Node 15+ 默认终止进程**；升级 Node 大版本时这条行为变化尤其致命；
17. **⚠️ 用 `forEach(async ...)`**——`forEach` 丢弃回调返回的 Promise，错误处理全部丢失；用 `map` + `Promise.all` 或 `for...of`；
18. **⚠️ 注册了 `unhandledRejection` handler 却没写退出逻辑**——注册 handler 会**覆盖** Node 的默认终止行为，进程会带着未知状态继续跑；
19. **⚠️ 以为「装了全局 handler 就修好了」**——handler 是安全网不是修复，真正要找的是那个漏掉的 `await` / `catch`；
20. **⚠️ 在全局 `unhandledRejection` handler 里直接 `console.log` 就完事**——生产环境应该记录 + 走优雅停机，而不是假装没事发生。

## 小结

- ⭐ **Promise 三态**：`pending` → `fulfilled` / `rejected`，**一旦 settle 不可逆**
- **`.then` / `.catch` / `.finally`**：`.then` 永远返回**新 Promise**，这是链式调用的基础；`.finally` **不改变结果**，其返回值被丢弃
- **微任务（Promise 回调）插队**在宏任务（`setTimeout` / IO 回调）之前
- ⭐ **`async` / `await` 是语法糖**：`async` 函数返回的永远是 Promise；`await` 让出的是**本层 async 函数**的执行权
- ⚠️ **忘记 `await` → 错误被吞 + 时序错 + 可能崩进程**；规则：**返回 Promise 的调用，要么 `await`，要么 `.catch`**
- ⭐ **四个组合子**：`all`（全成功）/ `allSettled`（永不失败，出明细）/ `race`（第一个 settle）/ `any`（第一个成功，全失败才 `AggregateError`）
- ⭐ **串行 vs 并行**：`for` + `await` = 串行；`Promise.all(map(...))` = 并行（关键是**不要**在 map 回调里 await 创建 Promise 的那行）
- ⚠️ **`Promise.all` 是「快速失败」不是「取消」**；要真取消用 `AbortController`
- **`for await...of`** 消费异步迭代器（`async function*` + `yield`），适合流式处理；它**只等 `next()`，不等循环体**
- ⭐ **背压**：Node 的流内置背压，`pipeline` 自动处理背压 + 自动传播错误；手动 `write()` 返回 `false` 要等 `drain`
- ⚠️ **未处理的 rejection 在 Node 15+ 会终止进程**；全局 `process.on("unhandledRejection")` 兜底，生产上应「记录 + 优雅停机」；⚠️ **注册 handler 会覆盖默认终止行为**，handler 里必须自己退出
- ⚠️ **`Promise.all` 无并发上限** → 自己实现 `mapWithLimit` 并发池（或用队列系统的 worker 数控制）

## 练习

1. 写一个 `withTimeout`，用 `Promise.race` 实现，并说明「超时后原 Promise 仍在跑」会带来什么问题。
2. 把下面这段串行代码改成并行，并预测耗时从多少变成多少；再解释为什么 `map` 回调里不能加 `await`。
   ```ts
   const ids = ["a", "b", "c"];
   const out: string[] = [];
   for (const id of ids) out.push(await slow(id));
   ```
3. 用 `Promise.allSettled` 重写上面第 5 节的例子，让失败项也能拿到结果，并说明 `results[0].value` 为什么在 TS 里需要先判 `status`。
4. 写一段代码，故意制造一个未处理的 rejection，分别在 Node 14 和 Node 20 上运行，说明你会看到什么不同（不要真的在生产跑）。
5. 实现 `mapWithLimit`，并写一个测试验证「任意时刻并发数不超过 limit」。（测试怎么写，见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]]。）
6. 用 `async function*` 实现一个「分页拉取」的异步生成器，每页 `await` 一次网络请求，用 `for await` 消费它。
7. 用 `fetch` + `for await (const chunk of res.body)` 写一个流式下载器，把内容写到 `node:fs` 的 WriteStream 上；对比用 `res.arrayBuffer()` 一次性读完的内存占用差异。
8. 写一段手动 `pipe` 串联三个流的代码，然后故意让中间的 Transform 流抛错，观察下游发生了什么；再用 `pipeline` 重写并对比。
9. 解释「`write()` 返回 `false`」在背压里的确切含义，并写出正确的等待写法。
10. 设计一个全局 `unhandledRejection` 处理策略：什么情况下记录后继续跑，什么情况下必须退出？说出你的判断标准和理由。
11. 讨论：`Promise.all` 快速失败 + 不取消，在「批量调用支付接口」这种有副作用的场景下会造成什么后果？你会怎么改？

## 导航

[[typescript/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[typescript/02-中段阶段-专题精讲/Node生态与异步/1-Node运行时与HTTP服务]] · 下一篇：[[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]]
