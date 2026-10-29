---
tags: [typescript, 中段, Node, HTTP]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# Node 运行时与 HTTP 服务

> **什么时候该读**：你要用 TypeScript 写一个后端服务（API / BFF / 小工具服务），需要搞清「Node 到底是怎么跑的」「为什么服务会卡死」「怎么优雅地把服务停掉」。
> **阅读门槛**：读完 [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.2-环境搭建与第一个程序]]（装好 Node.js、会跑 `npm`）和 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]]（会写类型守卫做运行时校验）。异步本身是下一篇的内容，本篇只用到「`await` 会让出执行权」这一个结论。

## 学习目标

读完本篇，你应当能够：

1. 用自己的话说明 Node.js 由哪两部分组成（V8 与 libuv），并画出「事件循环 + 线程池」的分工图
2. 解释为什么 Node 适合 IO 密集型任务，而 CPU 密集型任务会卡死整个进程，并说出两种解法
3. 说清 `process.env` / `process.argv` / `process.cwd()` 各自的作用，并知道密钥应该怎么管理
4. **独立写出一个可运行的 `node:http` 服务**，能路由、能读请求体、能返回 JSON 和状态码
5. 说出 Express 与 Fastify 的关键差异，判断一个项目该选哪个
6. 实现一个完整的优雅停机流程：`SIGTERM` → 停止接新请求 → 等存量跑完 → 退出

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.2-环境搭建与第一个程序]] | Node.js 的安装、`npm`、用 `tsx` 跑 TS 文件的方式 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]] | 请求体来自网络，是 `unknown`；用类型守卫把它变成可信类型 |
| [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]] | 本篇代码用 ESM 写法（`import`），顶层 `await` 也依赖 ESM |
| [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] | 优雅停机最终是为容器化部署服务的：容器收到 `SIGTERM` 后必须尽快退出，否则会被 `SIGKILL` 强杀 |

---

## 一、Node.js 是什么

### 1.1 先破除一个误解

Python 是「解释执行」的：`.py` 文件交给 `CPython` 解释器边读边执行。Node.js 属于**运行时（runtime）**——它是一套完整的「操作系统 + 语言 + 标准库」的组合体，把 JavaScript 变成一种可以跑长驻进程的语言。

Node.js = **V8 引擎**（跑 JS 代码）+ **libuv**（处理所有 IO） + **Node 标准库**（`fs` / `http` / `crypto` / `stream` …）。

| 部件 | 是什么 | 类比 Python |
|------|--------|-------------|
| **V8** | Google 的 JavaScript 引擎，负责把 JS 编译成机器码并执行 | 相当于 `CPython` 解释器，但它是 **JIT**（运行时把热点代码编译成本地机器码） |
| **libuv** | 跨平台异步 IO 库，内部包着一个**线程池** | 相当于 `asyncio` 底层的 `selector` + 线程池 |
| **Node 标准库** | `node:http`、`node:fs`、`node:stream` 等模块 | 相当于 Python 标准库（`http.server`、`asyncio`…） |

⚠️ **V8 是 JavaScript 引擎，不是 Node 独有的**。Chrome 浏览器也用 V8。Node 独有的是「V8 + libuv + 标准库 + npm 生态」这个组合。

### 1.2 libuv：把「等 IO」变成「不用等」

**IO（Input/Output，输入输出）** 指程序与外部世界交换数据：读文件、发网络请求、查数据库、等待磁盘。

传统做法是**阻塞式**的：

```
主线程: 发起请求 ────► 等待 500ms ────► 拿到数据 ────► 继续干活
                            ↑
                     这 500ms 主线程什么都不干，白白占着
```

Node 的做法是**异步**的：发起请求后立刻把控制权交还给别人，IO 由操作系统在后台做，数据好了再通知 JS。

```
主线程: 发起请求 ─► 立刻干别的 ─► ... ─► 回调被调用 ─► 继续干活
                          ↑
                  这段等待期间主线程是空闲的，能处理别的请求
```

### 1.3 ⭐ 核心机制：单线程事件循环 + 线程池 ⭐

这是 Node 最重要、也最容易被误解的一张图：

```
┌──────────────────── Node.js 进程 ─────────────────────────────┐
│                                                                 │
│  ┌──────────── 主线程（只有 1 个，负责执行所有 JS 代码）──────────┐ │
│  │                                                                │ │
│  │   ┌──────────────────────────────────────────────────┐        │ │
│  │   │  事件循环（event loop）                          │        │ │
│  │   │                                                  │        │ │
│  │   │  timers ─► pending callbacks ─► poll ─► check     │        │ │
│  │   │  （定时器）  （被推迟的回调）    （拿 IO 事件）（setImmediate）│ │
│  │   └──────────────────────────────────────────────────┘        │ │
│  │                    ▲                                          │ │
│  │        微任务队列（Promise 的 .then 回调）在这里插队执行        │ │
│  └────────────────────────┬───────────────────────────────────────┘ │
│                           │ 派发真正的 IO 工作                      │
│                           ▼                                        │
│  ┌────────── libuv 线程池（默认 4 个线程，可调）──────────────────┐ │
│  │   fs 读文件 · dns 域名解析 · crypto 哈希 · zlib 压缩           │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**事件循环（event loop）**：一个永不停止的循环，反复做三件事——

1. 看有没有到期的**定时器**（`setTimeout` / `setInterval`）；
2. 看有没有完成的 **IO 事件**（文件读完了、socket 有数据了）；
3. 看**微任务队列**（Promise 回调）有没有待执行的，有就**全部清空**再进入下一轮。

⭐ **一句话**：**JS 代码永远只在一个线程上跑；需要等待的活儿丢给线程池；线程池干完活儿通过事件循环把 JS 回调叫醒。**

> 微任务（Promise 回调）为什么能"插队"、和 `setTimeout` 的先后关系，见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] 的「Promise 回调什么时候执行」小节。

### 1.4 ⭐ 为什么 Node 适合 IO 密集型

**IO 密集型（I/O-bound）**：大部分时间在**等**，不在算。典型场景——REST API 网关、爬虫、Webhook 接收器、LLM 流式转发服务、数据库代理。

Node 的优势来自 1.3 的结构：**等待 IO 时，主线程根本不参与等待**。

```
100 个客户端同时请求，每个请求都要等数据库 50ms：

Node（单线程 + 线程池）：
  t=0ms    发出 100 个查询 → 主线程立刻空出来
  t=0~50ms 主线程还能处理新来的请求、还能处理别的业务逻辑
  t=50ms   100 个结果几乎同时回来

多线程（Python 的 ThreadPoolExecutor 思路）：
  100 个线程各自阻塞着等 —— 占着 100 个 OS 线程，内存开销大，上下文切换有成本
```

⚠️ 但要诚实说清代价：**Node 的单线程意味着同一时刻只能执行一段 JS**。如果一段 JS 要跑 200ms，这 200ms 内整个进程什么都干不了。**所以「适合 IO 密集」这句话有个前提：你的 JS 代码本身要短。**

### 1.5 ⚠️ CPU 密集型任务会卡死整个事件循环

**CPU 密集型（CPU-bound）**：大部分时间在**算**。典型场景——图像处理、视频转码、大数据量排序、加密解密、同步的大 JSON 解析、机器学习推理。

```ts
// ⚠️ 这段代码会卡住整个进程
function hashPasswords(passwords: string[]): string[] {
  return passwords.map((p) => {
    // 假设这是很重的计算（bcrypt / 大数运算 / 图像算法）
    let acc = 0;
    for (let i = 0; i < 5_000_000; i++) acc = (acc + i * 31) % 1_000_003;
    return `${p}:${acc}`;
  });
}

const server = createServer((req, res) => {
  const out = hashPasswords([/* 1000 个密码 */]);   // ⚠️ 这 2 秒内，事件循环完全停摆
  res.end(JSON.stringify(out));
});
```

在这 2 秒里：

- 其他所有在途请求**全部超时**（不是变慢，是完全不响应）；
- 定时器不触发、健康检查失败；
- 编排系统（Kubernetes）判定进程「假死」，可能直接重启它。

⭐ **根因**：事件循环只有一个线程，**没有「让路」的机制**。Python 有 `GIL`（全局解释器锁）导致多线程也跑不快，但 Python 的 CPU 密集任务可以靠 `multiprocessing` 逃出解释器；Node 的 JS 代码没有 GIL，但也**必须靠多进程/多线程逃出主线程**。

### 1.6 解法：`worker_threads` 或独立进程

| 方案 | 怎么用 | 优点 | 代价 |
|------|--------|------|------|
| **`worker_threads`** | `new Worker(new URL('./heavy.ts', import.meta.url))` | 同一个进程内起多个真线程，能共享内存（`SharedArrayBuffer`）、通信比进程间快 | 受 Node 启动参数 `--experimental-worker` 影响的旧版本已不适用；Node 20+ 开箱可用。注意：worker 里的代码不会自动继承主线程的 TS 转译配置 |
| **独立进程** | `child_process.fork()` 或直接 spawn 另一个 Node 进程 | 隔离最彻底；崩了不影响主进程；可以用现成的进程池（如 `pm2`、`cluster` 模块） | 进程间通信要序列化（有真实开销） |
| **交给别的系统** | 把重活丢给队列（Redis/BullMQ/Kafka），由专门的 worker 服务消费 | 服务水平扩展的正解 | 引入额外基础设施，链路变长 |

```ts
// worker_threads：把 CPU 密集任务挪到别的线程
import { Worker } from "node:worker_threads";

function runHeavyInWorker(data: number[]): Promise<number[]> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL("./heavy-worker.js", import.meta.url));
    worker.once("message", (result: number[]) => resolve(result));
    worker.once("error", reject);
    worker.postMessage(data);
  });
}
```

⭐ **选型口诀**：

- 只是偶尔重一点 → 先优化算法，把计算挪到请求之外（定时任务 / 后台任务）；
- 确实要并行算 → **`worker_threads`**（同进程、低开销）；
- 算得非常重 / 想要故障隔离 → **独立进程**；
- 需要弹性扩容 → **异步队列 + 独立 worker 服务**。

```python
# Python 的对应物：CPU 密集任务靠 multiprocessing 逃出解释器
# （线程受 GIL 限制，多线程对纯计算没有加速）
import multiprocessing

def heavy(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with multiprocessing.Pool(processes=4) as pool:
        results = pool.map(heavy, [10**7] * 8)   # ⭐ 真正的并行计算
```

⚠️ **Python 与 Node 的关键差异**：Python 的 `threading` 受 **GIL** 限制，对纯 CPU 计算几乎无加速，所以只能靠 `multiprocessing`；Node 的 JS 代码**没有 GIL**，但**主线程只有一个**，`worker_threads` 起的是真线程——所以 Node 里 `worker_threads` 就是 Python `multiprocessing` 的等价物。

## 二、进程与全局对象

### 2.1 `process`：Node 里的「全局变量」

**`process`** 是 Node 注入的全局对象，代表**当前操作系统进程**。在浏览器里没有这个东西（浏览器里对应的是 `window`）。TS 里它的类型由 `@types/node` 提供，**不需要 import 就能直接用**。

| API | 作用 | 示例 |
|-----|------|------|
| **`process.env`** | **环境变量**（键值字符串表，操作系统在启动进程时塞进来的） | `process.env.NODE_ENV` |
| **`process.argv`** | **命令行参数**数组 | `["/usr/bin/node", "/app/server.js", "--port", "3000"]` |
| **`process.cwd()`** | **当前工作目录**（进程启动时所在的目录） | `"/app"` |
| `process.pid` | 进程号 | `12345` |
| `process.exitCode` | 退出码（`0` 成功，非 `0` 失败） | `0` |
| `process.on(...)` | 注册信号 / 异常事件监听 | 见第五节 |

```ts
// argv[0] 是 node 可执行文件，argv[1] 是脚本路径，argv[2] 起才是用户传的参数
const port = Number(process.argv[2] ?? process.env.PORT ?? 3000);
```

⚠️ **两个高频坑**：

1. **`process.env` 的值类型永远是 `string | undefined`。** 环境变量在操作系统层面就是字符串，TS 官方把它建模成 `string | undefined`。所以 `process.env.PORT` 直接当数字用会报类型错误——必须 `Number(...)`，而且要考虑 `NaN`：

   ```ts
   const raw = process.env.PORT;
   const port = raw ? Number(raw) : 3000;
   if (!Number.isInteger(port) || port < 1 || port > 65535) {
     throw new Error(`非法端口: ${raw}`);
   }
   ```

2. **`process.cwd()` 不等于「当前文件所在目录」。** 它是**你敲命令时所在的目录**。在 Docker 里 `WORKDIR /app`，那 `cwd()` 就是 `/app`；而在本地你可能是在项目根目录跑的脚本，但脚本在 `src/` 下——路径就错了。要拿到「文件自己的目录」用 ESM 的 `import.meta.dirname`（Node 20.11+）：

   ```ts
   console.log(import.meta.dirname);   // ⭐ 文件所在目录（Node 20.11+）
   ```

### 2.2 ⚠️ 不要把密钥硬编码

```ts
// ⚠️ 绝对不要这样
const API_KEY = "sk-abc123def456";   // 会被提交进 git，进 CI 日志，进镜像层
```

硬编码密钥的四个泄漏渠道：**进了 Git 历史**（删了也还在历史里）、**进了前端产物**（浏览器里谁都能看）、**进了镜像层**（`docker history` 就能看到）、**进了 CI 日志**。

⭐ **正确做法**：密钥只从**环境变量**读；开发期用 `.env` 文件本地模拟；生产期由部署平台注入。

### 2.3 `.env` + `dotenv`

**`.env` 文件**：一行一个 `键=值`，放在项目根目录，**必须写进 `.gitignore`**。

```bash
# .env（⚠️ 绝不提交）
DATABASE_URL=postgres://app:secret@db:5432/app
OPENAI_API_KEY=sk-xxx
PORT=3000
```

**`dotenv`**：一个第三方包，启动时把 `.env` 里的键值读进 `process.env`。

```ts
// app.ts
import "dotenv/config";   // ⭐ 副作用导入：仅靠这条语句就完成加载

const key = process.env.OPENAI_API_KEY;
if (!key) {
  throw new Error("缺少 OPENAI_API_KEY，请检查 .env 或部署平台的 Secret 配置");
}
```

⚠️ **但 Node 已经内置了 `.env` 支持，新项目优先用内置方案**：

| 能力 | 版本 | 用法 |
|------|------|------|
| **`--env-file` 启动参数** | Node.js **20.6+** | `node --env-file=.env app.js` |
| **多个 `.env` 文件**（后加载的覆盖先加载的） | Node.js 20.7+ | `node --env-file=.env --env-file=.env.local app.js` |
| **`process.loadEnvFile()`**（代码里按需加载） | Node.js **20.12+** | `process.loadEnvFile(".env")` |
| **`--env-file-if-exists`**（文件不存在也不报错） | Node.js **22.9+** | `node --env-file-if-exists=.env app.js` |

```bash
# package.json —— 生产和开发都能用，不用装 dotenv
{
  "scripts": {
    "dev": "node --env-file=.env --watch dist/app.js"
  }
}
```

```ts
// 需要在代码里控制加载时机时（Node 20.12+）
import { loadEnvFile } from "node:process";

try {
  loadEnvFile(".env");
} catch (err) {
  // ⭐ 生产环境通常没有 .env（平台注入了环境变量），ENOENT 要忽略，其他错误要抛
  if ((err as NodeJS.ErrnoException).code !== "ENOENT") throw err;
}
```

⚠️ **四点注意**：

- `.env` 只是**开发期便利**，生产环境**不要**把 `.env` 打进镜像——用 K8s Secret / 云平台的密钥管理注入环境变量。
- ⭐ **「操作系统里已有的环境变量优先」**：内置 `--env-file` / `loadEnvFile()` **不会覆盖**已存在的 `process.env.X`（`dotenv` 默认也不覆盖，除非显式 `override: true`）。所以容器里注入的值不会被 `.env` 意外顶掉——**这条规则要记住**，它决定了「本地 `.env` 和生产注入谁说了算」。
- 内置方案**不支持变量插值**（`URL=http://${HOST}:3000` 不会展开）；需要插值才用 `dotenv` + `dotenv-expand`。
- 内置方案在 `.env` **缺失时会直接报错**（`loadEnvFile` 抛异常、`--env-file` 退出进程），`dotenv` 则是**静默跳过**。生产环境用内置方案要自己 try/catch，或用 `--env-file-if-exists`。

## 三、用 `node:http` 起一个最小 HTTP 服务

### 3.1 先把几个词讲清

| 术语 | 含义 |
|------|------|
| **HTTP（HyperText Transfer Protocol）** | 浏览器和服务器之间传数据的一套文本协议。请求是「方法 + 路径 + 头 + 体」，响应是「状态码 + 头 + 体」 |
| **请求 / 响应（request / response）** | 一次 HTTP 交互的两半。服务器里分别是 `req` 和 `res` 两个对象 |
| **路由（routing）** | 根据「方法 + 路径」决定执行哪段逻辑。比如 `GET /users` 和 `POST /users` 走不同分支 |
| **状态码（status code）** | 响应的一个数字，表示结果类别：`200` 成功、`201` 已创建、`400` 请求参数错、`404` 找不到、`500` 服务器内部错 |
| **请求体（body）** | 请求携带的**内容**。`GET` 通常没有 body，`POST`/`PUT` 常用 JSON body |

### 3.2 完整可运行的最小服务

`node:http` 是 Node 内置模块，**不需要装任何依赖**。下面这个文件可以直接跑：

```ts
// server.ts —— 完整可运行，用 `npx tsx server.ts` 启动
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";

const PORT = Number(process.env.PORT ?? 3000);

/** 读请求体并解析成 JSON；空体返回 {} */
async function readJson(req: IncomingMessage): Promise<unknown> {
  const chunks: Buffer[] = [];
  for await (const chunk of req) {
    chunks.push(chunk as Buffer);        // ⭐ 必须消费请求流，否则连接会出问题
  }
  const raw = Buffer.concat(chunks).toString("utf8");
  return raw ? JSON.parse(raw) : {};
}

/** 类型守卫：请求体是不是合法的用户对象（见 2.4 节） */
function isUserBody(v: unknown): v is { name: string; age: number } {
  if (typeof v !== "object" || v === null) return false;
  const o = v as { name?: unknown; age?: unknown };
  return typeof o.name === "string" && typeof o.age === "number";
}

const server = createServer((req, res) => {
  // ⚠️ createServer 的回调不是 async；异步出错必须自己兜住，否则进程会崩
  void handle(req, res).catch((err: unknown) => {
    console.error("请求处理失败:", err);
    if (!res.headersSent) {
      res.statusCode = 500;
      res.end(JSON.stringify({ error: "internal server error" }));
    } else {
      res.end();                        // 响应头已发出，只能把流收尾
    }
  });
});

async function handle(req: IncomingMessage, res: ServerResponse): Promise<void> {
  const url = new URL(req.url ?? "/", "http://localhost");
  res.setHeader("Content-Type", "application/json; charset=utf-8");

  if (req.method === "GET" && url.pathname === "/health") {
    res.statusCode = 200;
    res.end(JSON.stringify({ ok: true }));       // ⭐ 一定要 end，否则请求永远不返回
    return;
  }

  if (req.method === "POST" && url.pathname === "/users") {
    const body = await readJson(req);
    if (!isUserBody(body)) {
      res.statusCode = 400;
      res.end(JSON.stringify({ error: "参数不合法" }));
      return;
    }
    res.statusCode = 201;
    res.end(JSON.stringify({ id: 1, ...body }));
    return;
  }

  res.statusCode = 404;
  res.end(JSON.stringify({ error: "not found" }));
}

server.listen(PORT, () => {
  console.log(`listening on http://localhost:${PORT}`);
});
```

### 3.3 逐点讲透

**① `createServer(handler)`** 创建一个服务器。`handler` 是「每个请求进来时调用一次」的函数，签名是 `(req, res) => void`。

**② `req.url` 是路径 + 查询串**，例如 `/users?page=2`。想分开就 `new URL(req.url ?? "/", "http://localhost")`，然后取 `url.pathname`（路径）和 `url.searchParams`（查询参数）。

**③ `req.method` 是大写字符串**：`"GET"` / `"POST"` / `"PUT"` / `"DELETE"`。

**④ `res` 有三个「结束」方式**，必须恰好用一次：

| 写法 | 效果 |
|------|------|
| `res.end()` | 结束响应（不带数据） |
| `res.end(JSON.stringify(obj))` | 结束响应并写入数据 |
| `res.write(chunk)` | 写入一块数据，**不结束**——流式响应时用，最后仍要 `res.end()` |

**⑤ 状态码有两种写法**：`res.statusCode = 201;`（Node 原生风格）或 `res.writeHead(201, { "Content-Type": "application/json" });`（一次写状态码 + 头）。上面代码用前者 + `setHeader`，更直观。

**⑥ `req` 是一个可读流**（stream），`for await...of` 是消费它的正确姿势。Node 里几乎所有「数据从网络/磁盘来」的东西都是流，流与背压见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] 的「背压」小节。

**⑦ `createServer` 的回调不能直接写成 `async`**。写成 `async (req, res) => {...}` 后，返回的 Promise 没人接，函数体内抛出的错误会变成「未处理的 Promise 拒绝」，Node 会**直接终止进程**（原因见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] 的「未处理的 rejection」小节）。上面代码用 `void handle(...).catch(...)` 显式兜底。

**⑧ 监听端口**：`server.listen(PORT, callback)`。生产环境想监听所有网卡（容器里必须），用 `server.listen({ port: PORT, host: "0.0.0.0" })`。

### 3.4 三个必踩的坑（先在这里点一次名）

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| **忘了读请求体** | `req` 是流，不消费它，socket 上的数据排不空。轻则连接无法复用（keep-alive 失效），重则客户端一直写入被阻塞、双方僵持，客户端看到 `ECONNRESET` | 任何有 body 的请求都要把流读完（哪怕丢弃） |
| **忘了 `res.end()`** | 客户端**一直挂着**直到超时（响应头可能已经发出去了，所以看起来"有响应但没内容"） | 每个分支都要 `res.end()`，或用 `try/finally` 兜底 |
| **未处理的异常** | 异步回调里抛错 → 进程崩溃 → 所有在途请求被重置 | 每个 async 边界都 `.catch()`，并注册全局兜底（下一节） |

## 四、Express vs Fastify

两个都是 Node 主流的 **Web 框架（web framework）**：在裸 `node:http` 之上补齐了路由、参数解析、中间件、错误处理。

**中间件（middleware）**：Express 风格的「一串函数接力」——每个中间件拿到 `req`/`res`，处理完就 `next()` 交给下一个；不调用 `next()` 就短路结束。Fastify 也有类似的钩子（hook）机制，但插件模型不同。

| 维度 | **Express 5**（2024 年发布） | **Fastify 5**（需 Node 20+） |
|------|------|------------------------------|
| 生态 | 最大的第三方中间件生态，资料最多 | 中等偏小但在长，插件质量普遍更高 |
| 性能 | 中等；每个中间件都过一遍 | 更快；序列化由 schema 驱动（`fast-json-stringify`） |
| **入参校验** | 默认**不校验**，要自己接 `zod` / `joi` / `express-validator` | ⭐ **内置 JSON Schema 校验**，基于 Ajv |
| **类型安全** | 类型要手写，路由和 handler 之间容易脱节 | ⭐ **schema → TypeScript 类型自动推断** |
| 错误处理 | 靠 4 参数中间件 `(err, req, res, next)` | 抛错即 `async` handler 抛错，框架自动接管 |
| 上手成本 | 极低，"万物皆中间件" | 需要理解 schema 优先的设计 |

⭐ **Fastify 的杀手锏：JSON Schema 既是运行时校验器，又是类型来源。**

**JSON Schema** 是一种描述 JSON 数据结构的通用格式（JSON Schema 是 JSON 写的 schema）。Fastify 5 默认用 **Ajv** 做入参校验、用 **fast-json-stringify** 做响应序列化——两者都由同一份 schema 驱动，所以「校验规则」和「序列化规则」永远不会走偏。

⭐ **schema → TS 类型的官方路径**是配一个 **type provider**（类型提供器）：`@fastify/type-provider-typebox`。它配合 **TypeBox**（一个「用 TS 写 JSON Schema」的库）——**同一份 schema 同时是运行时校验器和编译期类型来源**：

```ts
// 用 TypeBox + type provider：schema 和类型同源，改一处两边都变
import Fastify from "fastify";
import { Type, type Static } from "@sinclair/typebox";
import { TypeBoxTypeProvider } from "@fastify/type-provider-typebox";

const app = Fastify({ logger: true }).withTypeProvider<TypeBoxTypeProvider>();

const UserBody = Type.Object({
  name: Type.String({ minLength: 1, maxLength: 50 }),
  age: Type.Integer({ minimum: 0, maximum: 150 }),
});
type UserBody = Static<typeof UserBody>;   // ⭐ 类型由 schema 推导，不用手写

app.post("/users", {
  schema: {
    body: UserBody,                      // ⭐ 运行时：不合规直接返回 400
    response: { 201: UserBody },         // 运行时：按 schema 裁剪响应字段
  },
}, async (request, reply) => {
  const body = request.body;             // ⭐ 编译期它就是 UserBody
  await reply.code(201).send({ name: body.name, age: body.age });
});

await app.listen({ port: 3000, host: "0.0.0.0" });   // 顶层 await 需要 ESM
```

⚠️ **别漏了 `.withTypeProvider<TypeBoxTypeProvider>()`**——少了这一句，`request.body` 的类型就退化成 `unknown` / `any`，schema 的类型收益完全拿不到。这是 Fastify 类型推断最常见的翻车点。

⭐ **选型建议**：

- **新项目、想要类型安全 + 高性能 → Fastify 5**（当前稳定线是 5.x）；尤其 AI/LLM 服务这类「入参结构固定、要严格校验」的接口，schema 的收益很大。
- **需要大量现成中间件（鉴权、日志、限流、Swagger）→ Express 5**；生态优势是压倒性的。
- **性能差多少？** 裸路由基准测试里 Fastify 通常是 Express 的 2–3 倍吞吐。但对绝大多数业务接口，**瓶颈在数据库和下游 HTTP，不在框架**——别为了跑分做决策。

## 五、优雅停机 ⭐

**优雅停机（graceful shutdown）**：收到「该走了」的信号后，**不再接新请求，把手上的活干完，然后干净地退出**。

### 5.1 为什么必须做

在容器化部署里这是**必需项**，不是加分项：

```
K8s 删除 Pod 的流程：
  1. K8s 从 Service 的 Endpoints 里摘掉这个 Pod  → 不再有新流量进来
  2. K8s 给容器发 SIGTERM                       → 应用应该开始优雅退出
  3. 等待 terminationGracePeriodSeconds（默认 30s）
  4. 超时还没退出 → 发 SIGKILL 强杀              → ⚠️ 内存里的状态全丢
```

⚠️ **不处理 `SIGTERM` 的后果**：Node 的默认行为是**收到 `SIGTERM` 立刻退出**——正在处理的请求被硬生生切断，客户端拿到 `ECONNRESET`；而且「摘掉 Endpoints」和「真正没有新连接进来」之间有**时间差**（连接池里的 keep-alive 连接还可能继续发请求），这期间直接退出就会丢请求。

### 5.2 四个步骤

```
① 收到 SIGTERM / SIGINT
        ↓
② 标记 shuttingDown = true  →  新请求返回 503（或直接拒绝新连接）
        ↓
③ server.close()  →  停止接受新连接，已有连接继续跑完
        ↓
④ 所有在途请求结束 + 定时器/DB 连接清理完毕
        ↓
   process.exit(0)
```

### 5.3 完整实现

```ts
// 挂在 server.listen 之后
let shuttingDown = false;

function shutdown(signal: string): void {
  if (shuttingDown) return;          // ⭐ 幂等：重复信号不能重复执行
  shuttingDown = true;
  console.log(`[${signal}] 开始优雅停机`);

  // ① 停止接受新连接；已有连接（keep-alive）会继续服务
  server.close((err) => {
    if (err) console.error("关闭 HTTP 服务出错:", err);
    console.log("[shutdown] 所有连接已关闭，进程退出");
    process.exit(err ? 1 : 0);
  });

  // ② Node 18.2+：立刻断开空闲的 keep-alive 连接，否则它们会拖住 close()
  server.closeIdleConnections();

  // ③ 兜底：最多等 10 秒，超时就强拆
  const timer = setTimeout(() => {
    console.warn("[shutdown] 超时，强制关闭剩余连接");
    server.closeAllConnections();
    process.exit(1);
  }, 10_000);
  timer.unref();                       // ⭐ unref：这个定时器不阻止进程退出
}

process.on("SIGTERM", () => shutdown("SIGTERM"));   // K8s / Docker stop 发的信号
process.on("SIGINT", () => shutdown("SIGINT"));     // Ctrl+C

// 兜底：没被捕获的异常和拒绝，记录后退出（别静默地继续跑在未知状态里）
process.on("uncaughtException", (err) => {
  console.error("[uncaughtException]", err);
  shutdown("uncaughtException");
});
process.on("unhandledRejection", (reason) => {
  console.error("[unhandledRejection]", reason);
  shutdown("unhandledRejection");
});
```

⭐ **四个容易忽略的细节**：

1. **要处理 `SIGINT`**——本地 `Ctrl+C` 和开发时的热重载也会发它。
2. **`shutdown` 必须幂等**——`SIGTERM` 之后紧接着可能又来一个 `SIGINT`，重复执行 `server.close()` 会抛错。
3. **`timer.unref()`**——不加的话，这个 10 秒定时器会让进程「明明没事了还赖着不走」。
4. **资源清理不只有 HTTP**：数据库连接池、Redis 连接、消息队列消费者、定时器都要一并关掉。标准做法是维护一个 `cleanup` 数组，在 `shutdown` 里依次执行。

### 5.4 和容器化的关系

- 容器里的进程是 **PID 1**，默认不处理 `SIGTERM` 的信号转发问题，所以**你的 Node 进程必须是 PID 1**（用 `docker run --init` 或 `tini` 做 init，更稳）。
- Dockerfile 里用 `CMD ["node", "dist/server.js"]`（exec 形式）而不是 `CMD npm start`（shell 形式）——shell 形式下 PID 1 是 shell，**信号传不到 Node**，`SIGTERM` 会被忽略，容器每次都要等到 `SIGKILL`。这正是 [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] 那一组里「容器生命周期」要配合的地方。
- `terminationGracePeriodSeconds` 要设得比你的停机耗时长（典型 10–30s）。

## 陷阱

1. **⚠️ 以为「Node 是多线程的，所以不怕阻塞」**——JS 代码只在**一个**线程上跑；一段 2 秒的同步计算会让**整个进程**停摆 2 秒，所有在途请求一起超时；
2. **⚠️ 忘了读请求体**——`req` 是流，不消费它会导致连接无法复用、客户端写入被阻塞、偶发 `ECONNRESET`；
3. **⚠️ 忘了 `res.end()`**——客户端一直挂到超时，看起来「有响应头没响应体」；
4. **⚠️ 把 `createServer(async (req, res) => ...)` 直接写成 async**——抛出的错误变成未处理的 Promise 拒绝，**进程直接崩溃**，所有在途请求被重置；
5. **⚠️ 把密钥硬编码在源码里**——会进 Git 历史、镜像层、CI 日志；一律走 `process.env`；
6. **⚠️ 以为 `process.env.PORT` 是数字**——它永远是 `string | undefined`，`Number()` 之后还要校验 `NaN` 和范围；
7. **⚠️ 以为 `process.cwd()` 是「脚本所在目录」**——它是「你敲命令时所在的目录」，Docker 里跟着 `WORKDIR` 变；要文件目录请用 `import.meta.dirname`；
8. **⚠️ 忘了 `.env` 要进 `.gitignore`**——`.env` 一旦提交，密钥就进了历史；
9. **⚠️ 以为 `.env` 会覆盖系统里已有的环境变量**——⭐ **反过来：已存在的 `process.env.X` 优先**，容器里注入的值不会被 `.env` 顶掉（`dotenv` 要显式 `override: true` 才会覆盖）；
10. **⚠️ 用内置 `--env-file` 却忘了文件可能不存在**——Node 20.6+ 的 `--env-file` 遇到缺文件会**直接退出**，`loadEnvFile()` 会**抛异常**；生产环境要用 `--env-file-if-exists`（Node 22.9+）或自己 try/catch 忽略 `ENOENT`；
11. **⚠️ 不处理 `SIGTERM`**——容器滚动更新时，正在处理的请求被硬切，客户端报 `ECONNRESET`；
12. **⚠️ `Dockerfile` 用 shell 形式的 `CMD`**——PID 1 是 shell，信号传不到 Node，`SIGTERM` 形同虚设；
13. **⚠️ 只做 `server.close()` 不调 `closeIdleConnections()`**——残留的 keep-alive 空闲连接会让 `close()` 一直等下去；
14. **⚠️ 以为「用了框架就自动优雅停机」**——Express / Fastify **都不会**替你监听 `SIGTERM`，这段逻辑永远要自己写；
15. **⚠️ 用 Fastify 的 TypeBox 却忘了 `.withTypeProvider()`**——`request.body` 退化成 `unknown`，schema 的类型收益完全拿不到。

## 小结

- **Node.js = V8（跑 JS）+ libuv（做 IO）+ 标准库**；核心机制是 ⭐**单线程事件循环 + 线程池**
- ⭐ **适合 IO 密集**：等 IO 时主线程不阻塞，一个线程能扛成千上万并发连接
- ⚠️ **CPU 密集会卡死整个事件循环**；解法：`worker_threads` / 独立进程 / 异步队列
- **`process`** 是全局进程对象：`process.env`（环境变量，`string | undefined`）、`process.argv`（命令行参数）、`process.cwd()`（工作目录，≠ 文件目录）
- **密钥只走环境变量**；开发期 `.env`，生产期由部署平台注入；⭐ **Node 20.6+ 内置 `--env-file`、20.12+ 内置 `process.loadEnvFile()`**，新项目不必装 `dotenv`；⭐ **已存在的环境变量优先于 `.env`**
- **`node:http` 四要素**：`createServer` 建服务、`req.url`/`req.method` 做路由、`for await` 读请求体、`res.end()` 结束响应 + `res.statusCode` 定状态码
- ⚠️ 三大高频坑：**忘读请求体**、**忘 `res.end()`**、**async 回调未捕获异常导致进程崩溃**
- **Express 5 vs Fastify 5**：Express 生态大、上手快；Fastify 内置 JSON Schema 校验并能 ⭐**反推 TS 类型**，性能更好
- ⭐ **优雅停机四步**：`SIGTERM` → 停接新请求 → `server.close()` 等存量跑完 → `process.exit`；必须**幂等**、必须配 `closeIdleConnections()`、兜底定时器要 `unref()`

## 练习

1. 用自己的话解释「事件循环 + 线程池」这张图，并回答：为什么「等待数据库 50ms」不会浪费线程，而「计算 50ms」会浪费整个进程？
2. 写一个脚本同时起两个 HTTP 服务：一个返回 `200`，一个先 `setTimeout` 阻塞 3 秒再返回。用 `curl` 并发请求第一个，观察它是否被拖慢——然后解释原因。
3. 把 3.2 的服务改造成支持 `DELETE /users/:id`；`id` 从 `url.pathname` 里切出来（提示：`pathname.split("/")`）。
4. 给服务加一个 `GET /echo` 路由，把请求的所有 header 原样返回成 JSON。
5. 实现一个「优雅停机」：收到 `SIGTERM` 后打印日志、新请求返回 503、等 2 秒后退出。用 `kill -TERM <pid>` 验证。
6. 用 `process.env` 读取 `PORT` 和 `LOG_LEVEL`，写一份 `.env` 和 `.env.example`，并解释为什么 `.env.example` 要提交、`.env` 不能提交。
7. 把 3.2 的裸 `node:http` 服务改成 Fastify 5 + TypeBox schema，让 `POST /users` 的 body 自动校验并自动推导出 TS 类型。
8. 查一下 `server.close()` 和 `server.closeAllConnections()`（Node 18.2+）的文档，说明「只 close 不 closeAll」和「只 closeAll 不 close」分别会怎样。
9. 解释为什么 Dockerfile 用 `CMD npm start` 会导致 `SIGTERM` 失效，并给出至少两种修法。
10. 调研 `worker_threads` 与 `child_process.fork()` 的取舍：什么场景选哪个？通信方式有什么本质区别？

## 导航

[[typescript/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：无（本专题第一篇） · 下一篇：[[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]]
