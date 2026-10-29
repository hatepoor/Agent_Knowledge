---
tags: [typescript, 中段, 工程化]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# 编译流程与 tsconfig

> **什么时候该读**：你写了第一份 `.ts` 文件，现在要弄明白「`tsc` 到底干了什么」「为什么改一行代码要重跑一次全量编译」「`package.json` 里那个 `tsconfig.json` 每个字段是干嘛的」。
> **阅读门槛**：会写基本 TS 语法、装好了 Node.js 与 `tsc`。看不懂工具链概念时，先退回 [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.2-环境搭建与第一个程序]]；类型注解本身不清楚的，见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]]。

## 学习目标

读完本篇，你应当能够：

1. 把「编译」拆成**类型检查 → 转译 → 执行**三个独立阶段，并说清每一阶段的输入输出
2. 手写一份能跑的 `tsconfig.json`，并解释其中每个关键字段的取值含义
3. 列出 `strict` 家族的全部子开关，说清 `strictNullChecks` 为什么是地基
4. 在 CI 里用 `tsc --noEmit` 卡住类型错误，在本地用 `tsc --watch` / `incremental` 提效
5. 判断一个库项目该不该开 `declaration`，并看懂产出的 `.d.ts` 是什么意思
6. 说出 TypeScript **6.0** 改了哪些默认值、**7.0** 把哪些配置变成了硬错误

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.2-环境搭建与第一个程序]] | 装好 `tsc`、跑通第一个 TS 程序 |
| [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]] | 理解「TS 是先编译再执行」，这是本文所有讨论的前提 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]] | `strict` 系列开关管的就是类型注解的严格程度 |
| [[typescript/术语与关键字手册]] | 术语速查，本文出现的编译器相关术语均在此登记 |

---

## 一、把「编译」拆成三个阶段

新手最常见的误解是：以为 TypeScript 有个「编译器」把 TS 变成另一种语言。这一句没错，但不够用。真正决定你日常体验的是：**这个流程其实是三个可以拆开使用的能力**。

```
        src/order.ts  （你写的源文件）
              │
              ▼
   ① 类型检查阶段   tsc --noEmit
      读源文件 + 类型注解 + 依赖的 .d.ts
      输出：一串诊断信息（错误 / 警告），不产出任何文件
              │
              │  ← 这一步通过（或你用 --noEmit 绕开）
              ▼
   ② 转译阶段       tsc
      擦除所有类型信息，把 TS 语法降级/翻译成 JS
      输出：dist/order.js（+ 可选 .d.ts / .js.map）
              │
              ▼
   ③ 执行阶段       node dist/order.js
      普通的 JS 引擎执行，和执行你手写的 .js 没有任何区别
```

⭐ **核心认知：① 和 ② 是两个独立的能力。**
- `tsc --noEmit` = 只做 ①，永远不做 ②。
- `esbuild` / `tsx` 这类工具 = 只做 ②，**完全不做 ①**。
- `tsc` 一个命令 = ① + ② 一起做。

这条分界线决定了后面所有工具选型（第 3 篇的「构建工具与包管理」整篇都在用这条线）。

### 1.1 阶段一：类型检查——**只检查，不产出**

**类型检查**（type checking）指的是：编译器逐行读你的 `.ts` 文件，把类型注解当成额外信息，推导出每个表达式的类型，然后检查这些推导结果彼此是否自洽。

它**不产出任何文件**。跑完只有一个「有没有错」的结论。

```ts
// src/order.ts
interface Order {
  id: string;
  total: number;
}

export function summarize(o: Order): string {
  return o.total.toUpperCase();
  //     ^^^^^^^^^^^ ⛔ error TS2339: Property 'toUpperCase' does not exist on type 'number'.
}
```

注意 `interface Order` 这个类型**只活在编译期**，它既不会变成对象，也不会在运行时做任何校验。

### 1.2 阶段二：转译——**擦除类型，产出 JS**

**转译**（transpile，也叫「降级」downlevel）是把 TS 源码改写成纯 JavaScript 的过程。

**类型擦除**（type erasure）是转译的核心规则：**所有只服务于类型系统的语法，在产物里被整段删掉**。不是「翻译成另一种形式」，是「消失」。

```ts
// 源码 src/order.ts
interface Order { id: string; total: number }
type Summary = string;
const TAX_RATE: number = 0.06;
export function summarize(o: Order): string {
  return `${o.id}: ${o.total}`;
}
```

```js
// 产物 dist/order.js —— interface / type / : number 全部消失
const TAX_RATE = 0.06;
export function summarize(o) {
    return `${o.id}: ${o.total}`;
}
```

⚠️ 由此得出三个必须记住的推论：

1. **运行时没有「类型」这回事**。`o.total` 是数字还是字符串，编译出来的 JS 完全不在乎，错了就是运行时才炸。
2. **类型检查器无法保证运行时正确**。它只保证「形状自洽」，不保证业务逻辑对。
3. **`const` / `interface` / `type` 这些语法不会拖慢运行时**——因为它们根本不在产物里。

### 1.3 阶段三：执行

产物就是普通 JS，用 `node` 跑、用浏览器 `<script>` 跑、用打包器跑都行。

### 1.4 对照 Python：差别在哪

| | TypeScript | Python |
|---|---|---|
| 执行模型 | 先编译成 JS，再执行 | 解释执行（`.pyc` 只是字节码缓存，不改变语义） |
| 类型检查 | **编译器的一个阶段**，跑 `tsc` 就会检查 | 运行时**完全不看**注解；`x: int = "a"` 照样能跑 |
| 检查工具 | 内建在 `tsc` 里 | 要另装 `mypy` / `pyright` |
| 注解的运行时价值 | 零（被擦除） | 存进 `__annotations__`，框架（如 Pydantic）会读它 |

⭐ **最容易照搬的错**：以为 Python 里 `def f(x: int)` 一样会拦下 `f("a")`。TS 里 `f("a")` 确实编译期就报错，但前提是 **`strict` 开着且参数没被推断成 `any`**——这正是第三节要讲的。

---

## 二、`tsconfig.json` 关键字段逐个解释

**`tsconfig.json`** 是 TypeScript 项目的配置文件，放在项目根目录。它告诉编译器「怎么编译这个项目」。

**编译器选项**（`compilerOptions`）里字段很多，但真正必懂的就是下面这 8 组。

| 字段 | 含义 | 建议值 |
|---|---|---|
| `target` | 产物 JS 版本 | `es2022`+ |
| `module` | 产物模块格式 | `nodenext` / `esnext` |
| `moduleResolution` | 模块解析策略 | `bundler` / `nodenext` |
| `strict` | 一组严格检查的总开关 | `true` |
| `outDir` / `rootDir` | 输出目录 / 源码根 | `dist` / `src` |
| `esModuleInterop` | 默认导入互操作 | `true` |
| `skipLibCheck` | 跳过 `.d.ts` 检查 | `true`（提速） |
| `include` / `exclude` | 纳入编译的文件 | `["src"]` |

### 2.1 一份最小可用的配置

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src"]
}
```

### 2.2 `target`：产物 JS 写到哪个语言版本

`target` 决定**语法降级**到什么程度：编译器会把新语法改写成老语法，让老引擎也能跑。

| `target` | 大致对应 | 说明 |
|---|---|---|
| `es5` | ES5（2009） | ⚠️ **TS 6.0 起已废弃，TS 7.0 变成硬错误** |
| `es2017` | async/await 原生 | 老项目常用 |
| `es2022` | 顶层 `await`、类字段、`at()` | ⭐ 现在的安全推荐 |
| `es2025` | 最新 | ⭐ **TS 6.0 起的新默认值** |

⚠️ **`target` 只管语法，不管 API。** 把 `target` 设成 `es2017` 并不会让 `Array.prototype.at()` 这种新 API 自动出现在老 Node 里——那是 **polyfill**（运行时打补丁，给老环境补上新 API）该干的活，通常靠 core-js 之类的库。`target` 降的是语法，polyfill 补的是 API，两者别混。

### 2.3 `module` 与 `moduleResolution`：产物格式 vs 找文件的方式

这两个字段名字像，但管的是**完全不同**的事：

- **`module`**：决定**产物文件用什么模块语法**（`import`/`export` 还是 `require`/`module.exports`）。
- **`moduleResolution`**：决定**编译时怎么找到 `import "./foo"` 里的那个文件**。

| `module` | 产物里的模块写法 | 适合谁 |
|---|---|---|
| `commonjs` | `require()` / `exports.x` | 老项目、只发 CJS 的库 |
| `esnext` | `import` / `export` 原样保留 | 交给打包器处理的前端项目 |
| `nodenext` | 由 `package.json` 的 `type` 字段决定 | ⭐ Node 库、后端服务 |
| `preserve` | 原样保留 `import`，但**不**擅自改写成 CJS | 用 bundler 的 TS 源码库 |

| `moduleResolution` | 解析规则 | 注意 |
|---|---|---|
| `node10`（旧称 `node`） | 经典 node_modules 逐级上溯 | ⚠️ **TS 6.0 废弃，7.0 硬错误** |
| `bundler` | 按打包器的方式找：走 `exports` 字段、允许无扩展名 | ⭐ 配 `module: esnext`/`preserve` |
| `nodenext` | 严格按 Node.js 现行规则找 | ⭐ 配 `module: nodenext`；**相对导入必须带扩展名** |

⚠️ **`module: nodenext` 会隐含把 `moduleResolution` 也设成 `nodenext`**，而 `nodenext` 带来的「相对导入必须写全扩展名」是最常见的报错来源（`ERR_MODULE_NOT_FOUND`）。模块系统的完整讨论见第 2 篇 [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]]。

### 2.4 `outDir` / `rootDir`：产物放哪、源码从哪开始

- **`outDir`**：产物输出目录。设了 `outDir: "dist"` 后，`src/order.ts` → `dist/order.js`，目录结构一一对应。
- **`rootDir`**：源码的根目录。**它决定了产物目录的形状**：设 `rootDir: "src"`，`src/a/b.ts` 就变成 `dist/a/b.js`；不设的话，编译器会推断一个「所有输入文件的公共父目录」作为根，产物结构可能跟你预期的不一样。

### 2.5 `esModuleInterop`：默认导入能不能用

`esModuleInterop: true` 开启后，编译器会在产物里插入一层「互操作垫片」，让 `import x from "cjs-module"` 这种**默认导入**能正常工作。原理与常见报错在第 2 篇展开讲，这里只记住结论：**新项目一律开**。

### 2.6 `skipLibCheck`：跳过第三方类型文件的检查

**`.d.ts`** 是**声明文件**（declaration file）——只含类型、不含实现的文件，用来描述一个 JS 库长什么样。

`skipLibCheck: true` 的作用是：检查你的代码时，**不检查 `node_modules` 里那些 `.d.ts` 自身的类型是否自洽**。

- 收益：类型检查速度常常快 2–3 倍（第三方类型文件动辄上万行）。
- 代价：第三方库之间如果存在类型冲突，**你看不到那条报错**，但用到冲突的地方可能会连带报错。

⚠️ 它跳过的只是「第三方 `.d.ts` 内部自检」，**你自己代码里的类型检查一点不少**。这是本篇少数「建议默认开」的宽松选项。

### 2.7 `include` / `exclude`：哪些文件参与编译

`include` 是**白名单**（要编译哪些），`exclude` 是**黑名单**（要排除哪些）。

```json
{
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

⚠️ 两个常见后果：
- `include: ["src"]` 意味着**根目录的 `vite.config.ts`、`scripts/*.ts` 都不在编译范围内**——它们要单独跑或另写一份配置。
- `exclude` 不写 `node_modules` / `dist` 时，编译器可能把**产物也当输入**，引发奇怪的循环错误。

### 2.8 其他几个高频字段（简表）

| 字段 | 作用 |
|---|---|
| `lib` | 指定可用的内置 API 集合（默认跟随 `target`）；⚠️ 手动调小 `lib` 会让某些 API 报「不存在」 |
| `declaration` | 产出 `.d.ts` 类型声明（发库必备，见第五节） |
| `declarationMap` | 产出 `.d.ts.map`，让 IDE 能从声明跳回源码 |
| `sourceMap` | 产出 `.js.map`，让运行时堆栈能映射回 TS 源码 |
| `noEmit` | 只检查不产出（等价于命令行 `--noEmit`） |
| `incremental` | 增量编译，只重编改动的文件 |
| `forceConsistentCasingInFileNames` | 文件名大小写必须与导入一致（跨 Linux/macOS 必开） |
| `noUncheckedIndexedAccess` | 数组/索引访问结果也可能是 `undefined`（更严格，默认关） |
| `verbatimModuleSyntax` | 类型导入必须用 `import type` 显式标注 |

---

## 三、`strict` 家族：一组严格检查的总开关

`strict` 是个**总开关**，它本身不检查任何东西，只是一次性打开下面 7 个子开关。

| 子开关 | 管什么 | 关闭后会出现什么 |
|---|---|---|
| `strictNullChecks` | `null` / `undefined` 不能随便赋给别的类型 | ⚠️ 整个空值安全体系失效（见 3.2） |
| `noImplicitAny` | 参数/变量推不出类型时**禁止**隐式变成 `any` | 拼错参数名也不报错 |
| `strictFunctionTypes` | 函数参数按**逆变**（contravariant）严格检查 | 子类回调能悄悄收窄参数类型 |
| `strictBindCallApply` | `bind`/`call`/`apply` 的参数也参与严格检查 | 这三个方法传错类型不报错 |
| `strictPropertyInitialization` | 类的属性必须在声明处或构造函数里初始化 | `this.x` 可能是 `undefined` 却不报 |
| `noImplicitThis` | `this` 没有明确类型时报错 | `this` 悄悄变成 `any` |
| `alwaysStrict` | 产物始终带 `"use strict"` | 输出不再自动启用严格模式 |

补充：从 **TypeScript 4.4** 起，`strict` 还会顺带打开 `useUnknownInCatchVariables`——`catch (e)` 里的 `e` 类型是 `unknown` 而不是 `any`，强制你先收窄再使用。

### 3.1 `noImplicitAny`：别让参数悄悄变成 `any`

```ts
function add(a, b) {
  return a + b;
}
//    ^  ⛔ error TS7006: Parameter 'a' implicitly has an 'any' type.
```

⚠️ **`any` 会「传染」**：只要表达式里出现一个 `any`，整个推导结果就是 `any`，类型检查在这里彻底失效。**`any` 是把类型系统整段短路的开关**，不是「暂时先这样」的省事选项。真正想表达「我还没想好」的时候，用 `unknown`（见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]]）。

### 3.2 ⚠️ `strictNullChecks` 关掉会怎样

**`strictNullChecks` 是 `strict` 家族里最重要的一个**，因为它决定了「空值安全」这套体系在不在。

默认（开启）时，`null` 和 `undefined` **不能**赋给普通类型：

```ts
function len(s: string): number {
  return s.length;
}
const r = len(undefined);
//          ^^^^^^^^^  ⛔ error TS2345: Argument of type 'undefined' is not assignable to parameter of type 'string'.
```

⚠️ **一旦把它关掉**（`"strictNullChecks": false`），上面全部合法：

```ts
const s: string = undefined;  // ✅ 不报错
const n: number = null;       // ✅ 不报错
function len(s: string) { return s.length; }
len(undefined);               // ✅ 不报错 → 💥 运行时 TypeError: Cannot read properties of undefined
```

关掉之后的连锁反应：

| 现象 | 后果 |
|---|---|
| `string` 可以接 `undefined` | 几乎所有函数的返回值都「可能」是 `undefined`，调用方不再被提醒 |
| 可选属性 `a?: string` 的 `a` 推断不出 `undefined` | 读 `obj.a.length` 不报错，运行时可能崩 |
| `??`、`?.` 的存在意义被削弱 | 到处都要写防御代码 |
| `strictPropertyInitialization` 跟着失去约束 | 类的属性可以不给初值 |

⭐ **一句话**：`strictNullChecks` 关掉，等于把「类型系统给了你安全感」这件事的主要来源拿掉了。剩下的严格开关只是补边角。

### 3.3 对照 Python

Python 生态里等价的东西是 **mypy 的 `--strict-optional`（默认开启）**——它管的就是「`None` 能不能传给 `str` 参数」。TS 把这个能力做成了 `strict` 的默认组成部分，**没有单独的等价物，也不需要额外配置**。

所以从 Python 迁过来最容易犯的错是：看到 `strictNullChecks` 报错太多（尤其是从 JS 老代码迁过来），就图省事关掉它。**关掉它，等于把 Python 里 mypy 默认帮你做的那件事关掉了。**

### 3.4 实践建议：分级开

如果历史包袱重，可以分两步走，但**最终目标必须是 `strict: true`**：

1. **第一步**：只开 `strictNullChecks` + `noImplicitAny`（收益最大的两个），其他按需。
2. **第二步**：补齐剩余子开关。`strictPropertyInitialization` 通常和类的字段设计有关，`strictFunctionTypes` 会在改回调签名时冒出来。

⚠️ 不要一上来就 `strict: false` 然后「以后再说」——`strict` 家族的问题会一路累积到运行时。

---

## 四、三种常用编译姿势

### 4.1 `tsc --noEmit`：只检查，不产出（CI 里用）

```bash
npx tsc --noEmit
```

- **作用**：完整跑一遍类型检查，**不写出任何 `.js` 文件**。
- **用在哪**：CI 流水线、pre-commit 钩子、编辑器之外的独立校验步骤。
- **为什么要有它**：构建产物通常是交给 `vite` / `tsup` 之类工具生成的（见第 3 篇），如果你在 CI 里跑 `tsc` 只会得到一堆没人用的 `.js` 文件，还会掩盖「构建工具到底有没有做类型检查」这个问题。

```yaml
# CI 里的一行
- run: npx tsc --noEmit
```

### 4.2 `tsc --watch`：边写边检查

```bash
npx tsc --watch
```

- 启动后常驻，**每次保存文件就重新检查**，并在终端打印错误。
- 按 `Ctrl+C` 退出。
- 用途：老项目没有 bundler 的开发循环；或者你就是想用 `tsc` 直接出产物跑。

⚠️ `tsc --watch` **会真的产出文件**（除非额外加 `--noEmit`）。想边看错误边跑，用 `npx tsc --watch --noEmit`；或用 `--watchOnError` 只在有错时编译。

### 4.3 `incremental`：增量编译

```bash
npx tsc --incremental
```

**增量编译**指编译器记下「上一次编译到哪儿了」，下次只重新编译**受改动影响的那部分文件**，其余直接复用上次的产物。

- 开启方式：命令行 `--incremental`，或在 `tsconfig.json` 里写 `"incremental": true`。
- 编译器会把状态写进一个 **`.tsbuildinfo`** 文件（默认放在 `outDir` 或项目根）。
- ⚠️ **`--incremental` 和 `--noEmit` 在 TypeScript 5.0 之前是互斥的**（会报 `Option '--incremental' cannot be specified with option '--noEmit'`）；**TS 5.0 起**可以同时用，于是「CI 增量检查 + 不产出文件」成立了。
- 配套概念：`composite`（项目引用，`references`）用于多包仓库，把大仓拆成若干互相引用的子项目分别编译。

⚠️ **增量编译省的是时间，不是正确性**。它依赖 `.tsbuildinfo` 准确反映现状；如果这个文件被错误地保留或拷贝到别的机器上，可能出现「改了代码却没重新编译」的诡异现象。CI 里要么删掉它，要么别开增量。

---

## 五、声明产物：`declaration` / `declarationMap` / `sourceMap`

这三个字段都控制「除了 `.js` 之外还要额外产出什么」。

| 字段 | 产出 | 用途 |
|---|---|---|
| `declaration: true` | `dist/order.d.ts` | **类型声明**：告诉使用方「这个函数接收什么、返回什么」 |
| `declarationMap: true` | `dist/order.d.ts.map` | 让 IDE 从 `.d.ts` 跳回 `src/*.ts` 源码 |
| `sourceMap: true` | `dist/order.js.map` | 让运行时堆栈映射回 TS 源码 |

### 5.1 为什么发库必须开 `declaration`

你的库是 `.ts` 写的，但使用者装到 `node_modules` 里的通常是 `.js`（或者 `.d.ts` + `.js`）。**没有 `.d.ts`，使用方就完全不知道你的 API 签名**，只能用 `any` 糊过去，等于你的类型工作全白做。

开启后的产物长这样：

```ts
// dist/order.d.ts —— 只有类型，没有实现
export interface Order {
    id: string;
    total: number;
}
export declare function summarize(o: Order): string;
```

⭐ **判断标准**：这个项目**只是给自己用**（应用、私有服务）→ `declaration` 可以不开；**要发布给别人用**（发到 npm 的库）→ `declaration` 必开，而且通常还要开 `declarationMap` + `sourceMap`，让使用者能点进源码。

### 5.2 `declarationMap` 的额外前提

`declarationMap` 依赖 `declaration`，也依赖**源码文件能被找到**。发布时如果只发 `dist/`，声明映射就指不到 `src/`——要么把 `src` 一起发出去，要么就别开 `declarationMap`。

### 5.3 `sourceMap` 与运行时堆栈

开了 `sourceMap` 后，`node dist/order.js` 崩栈时显示的是 `src/order.ts:42` 而不是 `dist/order.js:12`。⚠️ **生产环境通常不上传 `.map` 文件到公开 CDN**，但要在服务器日志里保留，否则线上排障只能看到一串压缩后的行号。

---

## 六、版本要点：TypeScript 6.0 与 7.0

> 这一节的内容**只适用于 TypeScript 6.0 及以上**。5.x 已经是维护线，别把 5.x 的默认值当现行标准。

### 6.1 TS 6.0（2026-03-23）改了默认值

TS 6.0 是**最后一个基于 JavaScript 实现的编译器版本**（下一个 7.0 换成了 Go 重写）。

| 配置项 | TS 5.x 默认 | **TS 6.0 默认** |
|---|---|---|
| `strict` | `false` | ⭐ **`true`** |
| `module` | `commonjs` | ⭐ **`esnext`** |
| `target` | `es5` | ⭐ **`es2025`** |
| `types` | 自动引入全部 `@types/*` | ⭐ **`[]`（空数组）** |
| `noUncheckedSideEffectImports` | `false` | ⭐ `true` |

前三条是好消息：开箱即用的项目自动获得现代默认值。

### 6.2 ⚠️ `types: []` 意味着 `@types/node` 不再自动引入

这是升级 TS 6.0 后最常见的现象：**代码一片「Cannot find name 'process'」「Cannot find name 'Buffer'」「Cannot find name 'NodeJS'」**，但这些明明是 Node 自带的。

原因：`types` 默认为空数组，代表**不再自动把 `node_modules/@types/*` 下的所有全局类型声明拉进来**。以前 TS 会扫描 `node_modules/@types` 目录全部引入，现在一个都不引，必须点名。

两种修法：

```json
// 方案 A：在 tsconfig.json 里显式点名（需先 npm i -D @types/node）
{
  "compilerOptions": {
    "types": ["node"]
  }
}
```

```ts
// 方案 B：在具体文件顶部用三斜线指令引用（适合只想局部引入的情况）
/// <reference types="node" />
```

⚠️ **只在需要 Node 全局类型时才加 `node`**。前端项目加了会白白把 Node 类型带进来，可能反而让「用了浏览器环境没有的 API」这类错误漏过去。

### 6.3 ⚠️ TS 6.0 废弃的项 —— TS 7.0 会变成硬错误

| 废弃项 | 6.0 行为 | **7.0 行为** |
|---|---|---|
| `target: "es5"` | 警告 | ⛔ **硬错误** |
| `moduleResolution: "node"` / `"node10"` | 警告 | ⛔ **硬错误** |
| AMD / UMD / SystemJS 模块格式 | 警告 | ⛔ **硬错误** |
| `baseUrl` 当作模块查找根 | 警告 | ⛔ **硬错误** |

**迁移对照表**：

| 老配置 | 新配置 |
|---|---|
| `"target": "es5"` | `"target": "es2022"`+ |
| `"moduleResolution": "node"` | `"moduleResolution": "bundler"`（前端）或 `"nodenext"`（Node 库） |
| `"module": "amd"` / `"umd"` / `"system"` | `"module": "esnext"` + 打包器处理 |
| 依赖 `baseUrl` 解析裸模块名 | 删掉 `baseUrl`，改用 `paths`（`paths` 不依赖 `baseUrl`，相对 `tsconfig.json` 解析） |

⚠️ **升级前必须处理干净**：这些配置在 6.0 只是警告，很容易被忽略，等到 7.0 直接编译失败。**升级前先用 6.0 跑一遍 `tsc --noEmit`，把警告全清掉。**

### 6.4 TS 7.0（2026-07-08 GA）：Go 原生编译器

- **编译器用 Go 重写**（代号 `tsgo`），编译速度比 6.0 快一个数量级。
- ⚠️ **不提供 compiler API**（没有 `typescript` 包的编程接口）。这意味着 **`typescript-eslint` 这类直接 import TS API 的工具必须留在 TS 6.0 上**，要等 7.1 才会补上。
- 所以现实中的节奏是：**类型检查跑 7.0，lint 工具链留在 6.0**。

---

## 七、陷阱

1. **⚠️ 以为 `tsc` 会打包**——`tsc` 只转译不打包：一个 `import` 对应一个 `.js` 文件，产物里全是相对路径导入。直接 `node dist/index.js` 在 ESM 下常报 `ERR_MODULE_NOT_FOUND`（因为相对导入没带扩展名）。要用打包工具，见第 3 篇 [[typescript/02-中段阶段-专题精讲/工程化与工具链/3-构建工具与包管理]]。
2. **⚠️ 以为类型检查能挡住运行时错误**——TS 只保证「形状自洽」。`JSON.parse` 返回 `null`、单位写错、并发竞态、浮点精度，TS 全都拦不住。类型是护栏，不是安全带。
3. **⚠️ 用 `any` 消掉报错**——`any` 会沿着推导链传染，把一片代码的类型检查全部短路。想表达「不确定」用 `unknown`。
4. **⚠️ 关掉 `strictNullChecks` 图省事**——这是空值安全的地基，关掉后编译器不再拦 `undefined` 传入，错误全部推迟到运行时。
5. **⚠️ 升级 TS 6.0/7.0 后满屏 `Cannot find name 'process'`**——`types` 默认值变成 `[]` 了，显式写 `"types": ["node"]` 或用 `/// <reference types="node" />`。
6. **⚠️ 忘了 6.0 已废弃 `target: es5` / `moduleResolution: node`**——6.0 只是警告，**7.0 直接硬错误**。升级前先在 6.0 上把警告清零。
7. **⚠️ `tsc --watch` 悄悄产出文件**——想只看不写，加 `--noEmit`；CI 里一律用 `--noEmit`。
8. **⚠️ `include` 没覆盖到配置文件**——`include: ["src"]` 时根目录的 `vite.config.ts`、`scripts/build.ts` 不参与编译，容易漏检。
9. **⚠️ `.tsbuildinfo` 被提交进 git 或跨机器复用**——增量状态与文件系统强相关，失效会导致「改了没重新编译」。把它加进 `.gitignore`。

## 小结

- 编译流程是**三段独立能力**：① 类型检查（只检查不产出）② 转译（擦除类型产出 JS）③ 执行。`tsc --noEmit` 只要 ①，`esbuild` 只要 ②。
- **类型擦除**意味着运行时不存在类型信息，类型检查器不保证运行时正确性。
- `tsconfig.json` 必懂 8 组字段：`target`（语法降级，不管 API）、`module`（产物模块格式）、`moduleResolution`（找文件的方式）、`strict`、`outDir`/`rootDir`、`esModuleInterop`、`skipLibCheck`、`include`/`exclude`。
- `strict` 家族 7 个子开关，**`strictNullChecks` 是地基**——关掉它，「类型带来的安全感」就没了。
- `tsc --noEmit` 进 CI，`tsc --watch` 进本地，`incremental` 提效；⚠️ TS 5.0 前 `--incremental` 与 `--noEmit` 互斥。
- 发库必开 `declaration`（出 `.d.ts`）；`declarationMap` 和 `sourceMap` 提升可导航性与排障能力。
- **TS 6.0（2026-03）**默认改为 `strict: true` / `module: esnext` / `target: es2025` / `types: []`；⚠️ **`types: []` 导致 `@types/node` 不再自动引入**。**TS 7.0（2026-07 GA）**把 `target: es5`、`moduleResolution: node`、AMD/UMD、`baseUrl` 查找根变成硬错误，且不提供 compiler API（lint 工具链需留在 6.0）。

## 练习

1. 建一个空目录，`npm init -y`，写入本篇第二节那份 `tsconfig.json`，新建 `src/order.ts` 放一个明显的类型错误，分别执行 `npx tsc --noEmit` 和 `npx tsc`，观察两者的**产物差异**（后者生成了什么文件？前者生成了吗？）。
2. 把 `strict` 设为 `false`，重做练习 1 的错误代码，再逐个打开 `strictNullChecks`、`noImplicitAny`，记录每一步多出来的报错。哪一步的报错最「值钱」？
3. 同一个项目分别用 `"target": "es2017"` 和 `"target": "es2022"` 编译，比较 `dist` 里的 JS 有何不同；再写一句代码用到 `Array.prototype.at()`，说明为什么**改 `target` 并没有**让你在旧 Node 上能用它。
4. 给项目加 `"incremental": true` 和 `"noEmit": true`，在 TypeScript 5.0 以下与 5.0 以上各会看到什么现象？（提示：TS 5.0 的 release notes 提到过一次「允许 `--noEmit` 与 `--incremental` 共存」的变更）
5. 用 `"types": ["node"]` 消除 `Cannot find name 'process'`；然后把它改回 `[]`，在文件顶部加 `/// <reference types="node" />`，对比两种方式的生效范围。
6. 查一下你项目里现存的 `tsconfig.json`，列出它用到的废弃项（`es5` / `node` / `amd` / `baseUrl`），并给出对应的现代替代配置。

## 导航

[[typescript/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：无（本专题第一篇） · 下一篇：[[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]]
