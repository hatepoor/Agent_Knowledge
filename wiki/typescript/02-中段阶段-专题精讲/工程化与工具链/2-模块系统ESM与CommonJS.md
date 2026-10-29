---
tags: [typescript, 中段, 工程化]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# 模块系统：ESM 与 CommonJS

> **什么时候该读**：你写了 `import`，报 `Cannot find module` 或 `ERR_MODULE_NOT_FOUND`；或者要发布一个库，不确定该用 `require` 还是 `import`；或者看到别人的项目里 `package.json` 有一行 `"type": "module"`，不知道它是干什么的。
> **阅读门槛**：需要知道 TypeScript 会被编译成 JS、且产物里没有类型信息（见上一篇 [[typescript/02-中段阶段-专题精讲/工程化与工具链/1-编译流程与tsconfig]] 第一节）；需要会写 `import` / `export` 语法。运行时原理部分（事件循环）见第 4 篇 [[typescript/02-中段阶段-专题精讲/Node生态与异步/1-Node运行时与HTTP服务]]。

## 学习目标

读完本篇，你应当能够：

1. 说清 ESM 与 CommonJS 两套模块系统的**语法差异**与**运行时差异**
2. 解释 `package.json` 里 `"type": "module"` 这一行的确切作用范围
3. 读懂 `exports` 字段与**条件导出**，说清「一个包同时支持两种模块系统」的标准做法
4. 用动态 `import()` 实现按需加载，并解释它返回什么
5. 解释 `esModuleInterop` 为什么能让 `import x from "cjs-module"` 工作
6. 说出 Node ESM 的硬规则（相对导入必须带扩展名）并解释它为什么存在
7. 判断「顶层 `await` 在 CJS 里为什么不能用」

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/02-中段阶段-专题精讲/工程化与工具链/1-编译流程与tsconfig]] | `module` / `moduleResolution` 决定产物模块格式；`esModuleInterop` 的出处 |
| [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.3-对象与数组]] | 模块的导出对象本质是一个对象，解构/展开语法直接用得上 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]] | `import type` 与普通 `import` 的区别 |
| [[typescript/术语与关键字手册]] | 术语速查 |

---

## 一、两套模块系统：它们到底是什么

一个程序要拆成多个文件，就必须有一套规矩说明「A 文件要用 B 文件的东西，去哪里拿」。这套规矩就是**模块系统**。

现在有两套并行存在：

| | **ESM**（ECMAScript Modules） | **CommonJS**（简称 CJS） |
|---|---|---|
| 地位 | **语言标准**（ECMAScript 规范的一部分） | Node.js 的历史传统，**不是**语言标准 |
| 语法 | `import` / `export` | `require()` / `module.exports` |
| 运行环境 | 浏览器原生支持、Node.js、bun、Deno | 主要是 Node.js；浏览器**不能**直接跑 |
| 文件判定 | 由 `package.json` 的 `type` 或扩展名决定 | 同左 |

### 1.1 CJS 怎么写

```js
// utils.js —— CommonJS
function add(a, b) {
  return a + b;
}
module.exports = { add };
```

```js
// main.js —— CommonJS
const { add } = require("./utils");
console.log(add(1, 2)); // 3
```

要点：
- `require("./utils")` 是**函数调用**，返回一个对象（**导出对象**）。
- `module.exports` 是那个对象的引用；`exports` 是它的别名。
- `require` 可以写在函数里、条件里 → **运行时**决定加载谁。

### 1.2 ESM 怎么写

```js
// utils.js —— ESM
export function add(a, b) {
  return a + b;
}
```

```js
// main.js —— ESM
import { add } from "./utils.js";
console.log(add(1, 2)); // 3
```

要点：
- `import` 是**语句**，不是函数调用，**不能**写在函数体里（但 `import()` 动态形式可以，见第四节）。
- `export` 有多种形态：`export function`、`export default`、`export { a, b }`、`export * from "./x.js"`。

### 1.3 在 TypeScript 里写

```ts
// src/utils.ts
export function add(a: number, b: number): number {
  return a + b;
}
export default { version: "1.0.0" };
```

```ts
// src/main.ts
import { add } from "./utils.js";
//                  ↑ 补 .js 是必须的，见第五节
console.log(add(1, 2));
```

⚠️ **注意最后那个 `.js`**。TypeScript 源文件是 `.ts`，但 **import 路径要写运行时真实存在的文件名**（`.js`），因为编译产物就是 `.js`，Node 运行时按 `.js` 去找。这是最容易踩的坑，第五节专门讲。

---

## 二、关键差异表

这是本篇的核心。ESM 和 CJS 不只是语法不同，**加载时机、绑定语义、可用能力都不一样**。

| 维度 | ESM | CommonJS |
|---|---|---|
| **加载时机** | **静态**：编译期就能确定依赖图，加载前全部解析、链接 | **动态**：运行时逐个 `require`，走到才加载 |
| **解析时机** | 编译期（TS 里就是 `tsc` 那一步） | 运行时 |
| **导入提升（hoisting）** | ⭐ `import` 会被**提升**到模块顶部，无论写在哪 | `require` 不会提升，写在哪就在哪执行 |
| **绑定语义** | ⭐ **活绑定**（live binding）：导出变量变了，导入方看到的是新值 | **快照拷贝**：导出时复制一份值，之后原变量改了不影响已导入的 |
| **顶层 `await`** | ⭐ 支持 | ⛔ 不支持 |
| **`__dirname` / `__filename`** | ⛔ 不存在，要自己用 `import.meta.url` 算 | ⭐ 直接可用 |
| **`import.meta.url`** | ⭐ 可用 | ⛔ 不可用 |
| **浏览器直接跑** | ⭐ 原生支持（`<script type="module">`） | ⛔ 不能，需打包 |
| **`this`（模块顶层）** | `undefined` | `module.exports` |

### 2.1 「静态 vs 动态」到底差在哪

**静态分析**（static analysis）指的是：编译器在**不看运行结果**的前提下，只读源码就能分析出程序结构。

```ts
// ESM：编译器看得懂每一行 import
import { a } from "./a.js";
import { b } from "./b.js";
```

因为是静态的，ESM 能做到两件 CJS 做不到的事：

1. **提前发现用错的导入名**。`import { notExist } from "./a.js"` 在编译期就报错 `Module '"./a.js"' has no exported member 'notExist'`。
2. **「链接」阶段先建好整张依赖图**，模块之间可以**循环引用**（虽然要小心「暂时性死区」——即变量已绑定但尚未初始化的那段区间，访问它会抛 `ReferenceError`，见 [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.1-变量与基本类型]]）。

```ts
// CJS：编译器看不见，写错了要到运行时才炸
const { notExist } = require("./a.js");
```

⚠️ 运行时才炸意味着：**CJS 里导入名打错、导出不存在，程序会在执行到那一行时才崩**。这在 Web 服务里意味着「半夜某个低频接口被调用时才崩」。

### 2.2 「导入提升」是什么意思

```ts
// ESM
console.log(x);      // ⛔ ReferenceError: Cannot access 'x' before initialization
import { x } from "./x.js";
```

`import` 语句虽然写在最后，但被**提升**到模块最顶部执行。所以模块还没执行完，`x` 就已经绑定了（只是它的值还是「暂时性死区」状态，访问会抛错）。

CJS 则相反：

```js
// CJS
console.log(y);      // ✅ 不报错（y 是 undefined）
const y = require("./y.js");
```

`require` 是普通函数调用，就在那行执行。

### 2.3 「活绑定 vs 快照拷贝」

```ts
// counter.ts (ESM)
export let count = 0;
export function increment() {
  count++;
}
```

```ts
// main.ts (ESM)
import { count, increment } from "./counter.js";
console.log(count);   // 0
increment();
console.log(count);   // ⭐ 1 —— 活绑定，导入方看到的是最新值
```

CJS 则是快照：

```js
// counter.js (CJS)
let count = 0;
exports.getCount = () => count;      // 用函数绕开快照问题
exports.count = count;
```

⚠️ 纯值导出在 CJS 里是**拷贝**，导出后再改原变量，导入方看到的还是旧值。ESM 的 `import` 是**绑定**（引用），永远看到最新值。

⚠️ 另外 ESM 的 `import` **是只读的**：给导入的变量赋值会在编译期直接报错 `Cannot assign to 'x' because it is an import`。

### 2.4 `__dirname` 与 `import.meta.url`

```js
// CJS
console.log(__dirname);   // /home/me/project/src
```

```js
// ESM：__dirname 不存在，用 import.meta.url 换算
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

**`import.meta`** 是 ESM 特有的**元数据对象**，`import.meta.url` 是当前模块的完整文件 URL（`file:///home/me/project/src/main.js`）。

---

## 三、`"type": "module"` 开关

⭐ **这是本篇最该记住的一个字段。**

**`.js` 文件到底按 ESM 还是 CJS 处理，Node.js 看 `package.json` 里的 `type` 字段**：

| `package.json` | `.js` 文件被当作 | 备注 |
|---|---|---|
| 有 `"type": "module"` | **ESM** | 整个包的 `.js` 都是 ESM |
| 有 `"type": "commonjs"` | **CommonJS** | 显式声明 CJS |
| **没有 `type` 字段** | **CommonJS** | Node 的历史默认 |

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js"
}
```

### 3.1 作用域：一个包一份设置

⚠️ **`type` 是按「最近的 `package.json`」逐级向上查找的**——即 **Node 的「包边界」**规则。`node_modules` 里每个包各有自己的 `package.json`，所以**依赖包和你的代码可以各用各的模块系统，互不干扰**。

### 3.2 扩展名可以覆盖 `type`

不管 `type` 怎么设，两个扩展名是**硬规则**：

| 扩展名 | 始终按 |
|---|---|
| `.mjs` | ESM |
| `.cjs` | CommonJS |
| `.js` | 看 `package.json` 的 `type` |

⭐ 这个「逃生舱」很重要：**同一目录里可以混放**——`main.mjs`（ESM）和 `legacy.cjs`（CJS） coexisting。

### 3.3 ⚠️ 双包陷阱（dual package hazard）

⚠️ **最经典的事故**：一个库同时以 ESM 和 CJS 两种形态发布，用户装到 `node_modules` 里的**其实是两份独立的代码副本**。

```text
node_modules/
└── my-lib/
    ├── dist/esm/index.js     ← 副本 A（ESM）
    └── dist/cjs/index.cjs    ← 副本 B（CJS）
```

后果：ESM 侧和 CJS 侧**内存不共享**。典型症状：

- 单例失效：`new Date()`、配置对象、事件总线各有两份
- `instanceof` 失效：`a instanceof B` 突然变成 `false`（两个不同类的构造器）
- 内存泄漏：模块级缓存永远清不掉

⭐ **规避方式**（`exports` 条件导出能做到）：

```json
{
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs"
    }
  }
}
```

⚠️ 条件导出解决的是「**同一个包**对外提供两种入口」，但**不能**解决「两个不同版本的同一个包同时被装进来」（那是 npm 依赖树扁平化的问题，见第 3 篇）。

---

## 四、`exports` 字段与条件导出

### 4.1 `exports`：现代包的标准入口声明

老式 `package.json` 用 `main` + `module` 两个字段分别指向 CJS 和 ESM 入口。新式做法是**用一个 `exports` 字段**统一描述「这个包对外暴露什么」。

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./package.json": "./package.json"
  }
}
```

各字段含义：

| 字段 | 含义 |
|---|---|
| `main` | 老式入口（CommonJS），给老工具看 |
| `module` | 老式入口（ESM），给 bundler 看（**非标准**但约定俗成） |
| `types` | 类型声明文件入口（`.d.ts`） |
| `exports` | ⭐ 现代入口声明，**同时也是「封装边界」** |

⭐ **`exports` 最大的价值是「封装」**：一旦声明了 `exports`，**包内部的文件路径默认全部被封锁**，只有 `exports` 里显式列出的路径能被外部 `import`。

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./internal/helper.js": "./dist/internal/helper.js"
  }
}
```

上面这个包里，外部 `import "my-lib/dist/index.js"` 会报 `ERR_PACKAGE_PATH_NOT_EXPORTED`。

⚠️ **副作用**：因为 `exports` 会屏蔽深层路径，**有些工具依赖深层导入时会失效**——典型的是 `import pkg from "my-lib/package.json"`。解法是在 `exports` 里显式放行（如上面的 `"./package.json"` 条目）。

### 4.2 条件导出（conditional exports）

**条件导出**是在 `exports` 里用「条件 → 路径」的映射，让**运行时根据调用方的环境选不同的文件**。

```json
{
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs"
    }
  }
}
```

上面的含义：

- 调用方用 `import` 语法 → 给它 ESM 文件
- 调用方用 `require()` → 给它 CJS 文件

⭐ **这就是「同一个包同时支持两种模块系统」的标准做法**：一次发布，两个入口，用户无须关心。

### 4.3 条件不止两种

条件是可以自定义的，Node 按**从上到下**的顺序匹配，**第一个匹配的生效**：

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "node": {
        "import": "./dist/node.mjs",
        "require": "./dist/node.cjs"
      },
      "browser": "./dist/browser.js",
      "default": "./dist/index.js"
    }
  }
}
```

| 条件 | 什么时候命中 |
|---|---|
| `types` | ⚠️ **必须放最前面**，TypeScript 靠它找类型声明 |
| `import` | 调用方用了 ESM `import` |
| `require` | 调用方用了 CJS `require()` |
| `node` | 在 Node.js 环境里 |
| `browser` | 在浏览器 / bundler 的 browser 字段环境 |
| `default` | 兜底，放最后 |

⚠️ **`types` 必须放第一位**。TypeScript 匹配 `exports` 时按同样顺序遍历，如果 `types` 排在后面且前面某条命中了，TS 就找不到类型声明了——症状是「编辑器里这个包没有类型」。

### 4.4 为什么类型放 `exports` 里而不是只放 `types` 字段

因为 `exports` 可以**按条件区分类型文件**：

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/esm/index.d.mts",
        "default": "./dist/esm/index.js"
      },
      "require": {
        "types": "./dist/cjs/index.d.cts",
        "default": "./dist/cjs/index.cjs"
      }
    }
  }
}
```

这样 ESM 消费方拿 `.d.mts`、CJS 消费方拿 `.d.cts`，类型完全对得上。**TypeScript 4.7+** 支持这套写法（`.d.mts` / `.d.cts` 这两种声明文件扩展名）。

---

## 五、动态 `import()`：按需加载

静态 `import` 是语句，不能写在函数里。但**动态 `import()`** 是一个**函数式的表达式**，返回一个 **Promise**。

```ts
// src/loader.ts
async function loadHeavyModule() {
  // ⚠️ 注意：路径仍然是相对于当前模块的，且要带扩展名
  const mod = await import("./heavy.js");
  mod.run();
}
```

### 5.1 三个要点

1. **返回 Promise**：里面是模块的**命名空间对象**（namespace object），用法 `mod.函数名()`。
2. **路径是字面量字符串**：不能是变量拼接（除非用 bundler 的能力），因为 bundler 靠静态分析找依赖。
3. **CJS 里也能用**：`import()` 在两种系统里都合法（TS 会按 `module` 设置决定编译成 `import()` 还是 `Promise.resolve().then(() => require(...))`）。

### 5.2 用途：代码分割

⭐ 动态 `import()` 最主要的用途是**按需加载**（代码分割，code splitting）：把一个不常用的大模块从主包拆出去，用户用到时才下载。

```ts
// 只有用户点了「导出报表」才加载这一大坨
async function exportReport(id: string) {
  const { exportToExcel } = await import("./exporters/excel.js");
  return exportToExcel(id);
}
```

没有动态 `import()` 的话，要么全量打进主包（首屏变慢），要么手动拆多个文件 + 自己写加载逻辑。

### 5.3 对照 Python

Python 的对应物是 `importlib.import_module("x")`，同样是**运行时**导入，返回一个模块对象。

⚠️ 但 Python 有一个 ESM 完全没有的东西：**Python 的 `import` 语句在函数里是合法的，且有「只导入一次」的全局缓存**（`sys.modules`）。JS 的 `import()` 每次调用都会返回一个 Promise，但**底层模块只会被真正求值一次**（ESM 有模块缓存，CJS 有 `require.cache`）。所以「动态 import 会不会重复加载」的答案是：**不会重复执行模块体**。

---

## 六、⚠️ 默认导入与命名导入的互操作

这是「为什么 `import x from "cjs-module"` 能工作」的故事。

### 6.1 问题是什么

一个 CJS 库这样导出：

```js
// node_modules/legacy-lib/index.js
module.exports = { hello: () => "hi" };
```

你用 ESM 语法导入它：

```ts
import legacy from "legacy-lib";   // ⛔ error TS1259: Module can only be default-imported using the 'esModuleInterop' flag
```

⚠️ **为什么报错？** 因为 `module.exports = {...}` 这个对象里**没有名为 `default` 的属性**。ESM 规范要求默认导入对应导出对象上的 `default` 键，取不到就该报错。

### 6.2 `esModuleInterop` 做了什么

```json
{
  "compilerOptions": {
    "esModuleInterop": true
  }
}
```

开启后，TypeScript 会做两件事：

1. **类型层面**：允许你把 CJS 模块默认导入，类型就是那个导出对象的类型。
2. **产物层面**：插入一层**互操作垫片**（interop helper），大致等价于：

```js
// esModuleInterop 开启后的产物（简化）
import { __importDefault } from "tslib";  // 或内联的等价实现

const legacy_1 = __importDefault(require("legacy-lib"));
// legacy_1 = { default: { hello: ... } }  ← legacy-lib 没有 __esModule 标记，被包了一层
legacy_1.default.hello();   // ✅ 拿到 { hello: ... } 本身
```

垫片的逻辑是：如果被导入的模块**没有** `__esModule` 标记，就把它整个包一层 `{ default: 模块 }`；如果有标记（说明它本来就是 ESM 风格），就直接用。

```js
// 垫片的实际行为（简化）
function __importDefault(mod) {
    return (mod && mod.__esModule) ? mod : { default: mod };
}
```

所以 `import legacy from "legacy-lib"` 拿到的是 `{ hello: ... }` 这个对象本身（被垫片包成了 `{ default: { hello: ... } }`，再由 `default` 取回来）。

### 6.3 命名导入为什么也能工作

```ts
import { hello } from "legacy-lib";
```

CJS 模块的导出对象上**恰好有** `hello` 这个属性，运行时能取到——但 TypeScript 不知道「这个 CJS 包导出了什么名字」（没有 `.d.ts` 描述时，它只能把这个模块整体推断成 `any`）。开了 `esModuleInterop` 之后，TS 对 CJS 模块的**命名导入**也放宽了：命名导入按「导出对象的属性」来解析，而不是按 ESM 的静态导出列表。

### 6.4 `esModuleInterop` 与 `allowSyntheticDefaultImports`

| 字段 | 作用 |
|---|---|
| `allowSyntheticDefaultImports` | 只影响**类型检查**，允许你写默认导入（不改产物） |
| `esModuleInterop` | 影响**产物**，插入垫片；**并且隐含开启** `allowSyntheticDefaultImports` |

⭐ **版本要点**：`esModuleInterop` 从 **TypeScript 5.0** 起**默认值就是 `true`**，所以新项目不用手写也会开着（TS 6.0 起 `strict: true` 是默认，更是必开）。只有显式关掉、或沿用很老的 `tsconfig.json`，才会遇到默认导入报错。

⚠️ **反过来的坑**：如果你把 `esModuleInterop` 关掉又开了 `allowSyntheticDefaultImports`，类型检查能过，但**产物运行时会报错**——因为没有垫片。这是「类型过了但运行炸」的典型。

### 6.5 怎么选：优先用命名导入

⭐ **实践建议**：写库给别人用时，**优先导出命名成员**（`export function foo`），而不是 `export default`。理由：

- 命名导入让使用者可以**按需取**，还能在 ESM/CJS 之间自由混用；
- 默认导入 + 互操作垫片是**兼容性 hack**，只在对接老 CJS 库时才需要；
- 一些打包工具（bundler）对 `export default` 的**树摇优化**（tree shaking，摇掉没用到的代码）支持较差。

---

## 七、⚠️ Node 的 ESM 硬规则：相对导入必须带扩展名

### 7.1 规则

在 Node 的 ESM 里：

```ts
import { a } from "./utils";        // ⛔ ERR_MODULE_NOT_FOUND
import { a } from "./utils.js";     // ✅ 正确
import { a } from "./utils/index.js"; // ✅ 正确（要写到具体文件）
```

⚠️ **三条硬规则**：

1. **相对导入必须带完整扩展名**（`./utils.js`），不能省略成 `./utils`；
2. **不能靠目录索引**：`./utils` 不会自动去找 `./utils/index.js`；
3. **不能省扩展名**：`./utils.mjs`、`./utils.cjs` 也不能省成 `./utils`。

### 7.2 为什么 Node 要这么定

因为 **ESM 是给浏览器设计的**。浏览器加载模块时靠 **URL** 去网络请求，服务器不保证给你 MIME 类型推断后帮你补扩展名——浏览器**必须**知道确切文件名。所以 ESM 规范要求完整路径，Node 继承了这一点。

CJS 则是运行时在文件系统里逐级找（`./utils` → `./utils.js` → `./utils/index.js`），所以 CJS 能省扩展名。

### 7.3 TS 里为什么写 `.js` 而不是 `.ts`

因为 **import 路径要指向运行时真实存在的文件**。你的产物是 `dist/utils.js`，所以源码里就写 `"./utils.js"`。

TypeScript 有一套**后缀替换**规则专门处理这件事：

| 你写的 | TS 查找的文件 |
|---|---|
| `./utils.js` | 优先找 `utils.ts` → `utils.tsx` → `utils.d.ts`，找不到再看真的 `utils.js` |
| `./utils.mjs` | `utils.mts` → `utils.d.mts` → `utils.mjs` |
| `./utils.cjs` | `utils.cts` → `utils.d.cts` → `utils.cjs` |

⭐ **所以标准做法是：源码里统一写 `.js` 扩展名**（即使源文件是 `.ts`）。这样 `tsc` 产出的 JS 可以直接在 Node 里跑，路径天然对得上。

### 7.4 ⚠️ 这是最常见的报错来源

```text
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/home/me/project/dist/utils'
    imported from /home/me/project/dist/index.js
```

排查顺序：

1. 是不是漏写了扩展名；
2. 是不是写了 `.ts`（运行时没有 `.ts` 文件）；
3. 是不是路径大小写不对（Linux 区分大小写，Windows 不区分 → **本地能跑、服务器炸**）。

⚠️ 顺带一提：`moduleResolution: "bundler"` 模式下 TS **允许**省略扩展名，但产物跑在 Node 上还是会挂——`bundler` 解析策略意味着你的代码**最终要经过打包器**，不经过的话就该用 `nodenext`。

---

## 八、顶层 `await`

### 8.1 什么是顶层 await

**顶层 await** 指在**模块最外层**（不在任何函数里）直接使用 `await`：

```ts
// src/init.ts —— 顶层 await
const db = await connectDatabase();
const config = await loadConfig();
startServer(db, config);
```

### 8.2 为什么只有 ESM 能有

因为 ESM 的加载模型**天然支持异步求值**：模块的依赖图先链接完，再异步求值，求值过程中允许 `await`。整个模块系统的规范里就有「async module」这个概念。

CJS 的加载模型是**完全同步**的：`require()` 必须立刻返回一个值。要支持顶层 await，就得把整个 CJS 加载模型改成异步的——这是破坏性变更，Node 不会做。

⚠️ **所以在 `.cjs` 文件（或 `"type": "commonjs"` 的包里）写顶层 await 会直接语法报错**：

```text
SyntaxError: await is only valid in async functions and the top level bodies of modules
```

### 8.3 顶层 await 的代价

⚠️ **顶层 await 会阻塞整个依赖图的求值**。如果模块 A 顶层 await 了，而模块 B 依赖 A，那么 B 的代码也要等 A 的 await 完成才能跑。

在**库代码**里用顶层 await 是危险动作——你的库一旦有顶层 await，所有依赖它的模块都会被拖慢。所以：

- ✅ 应用入口（`index.ts`）用顶层 await 没问题；
- ⛔ 发布给别人用的库，别在顶层 await。

### 8.4 TS 里怎么开启

顶层 await 需要：

```json
{
  "compilerOptions": {
    "target": "es2017",
    "module": "esnext",
    "moduleResolution": "bundler"
  }
}
```

⚠️ **`module` 必须是 `esnext` / `es2022` / `nodenext` / `preserve` 这些 ESM 形态**，且 **`target` 至少 `es2017`**。用 `module: "commonjs"` 时 TS 会报 `TS1378: Top-level 'await' expressions are only allowed when the 'module' option is set to 'es2022', 'esnext', 'system', 'node16', 'node18', 'nodenext', or 'preserve', and the 'target' option is set to 'es2017' or higher`。

### 8.5 对照 Python

Python **没有**顶层 await。模块顶层写 `await` 是语法错误。Python 的做法是：

```python
# ❌ Python 模块顶层不能 await
# ✅ 用 asyncio.run 在入口同步等待
if __name__ == "__main__":
    asyncio.run(main())
```

TS 有顶层 await，Python 得靠 `asyncio.run` 包一层——这是 ESM 相对 Python 的一处便利。

---

## 九、陷阱

1. **⚠️ ESM 里相对导入漏了扩展名**——`import "./utils"` 直接 `ERR_MODULE_NOT_FOUND`。**规则：写 `./utils.js`**，即使源文件是 `.ts`。
2. **⚠️ 在 ESM 文件里用 `__dirname`**——`__dirname is not defined`。改用 `fileURLToPath(import.meta.url)`。
3. **⚠️ 给 ESM 文件写顶层 await，然后 `require()` 它**——CJS 加载器同步拿结果，ESM 模块有顶层 await 时只能动态 `import()`。
4. **⚠️ 以为 `"type": "module"` 只影响入口文件**——它影响**整个包**的所有 `.js`。想混用就用 `.cjs` / `.mjs` 逃生舱。
5. **⚠️ 双包陷阱**——同一库同时发 ESM 和 CJS 两份产物，两个副本内存不共享，单例和 `instanceof` 都会出问题。
6. **⚠️ `exports` 里 `types` 没放第一位**——TS 匹配不到类型声明，编辑器里「这个包没有类型」。
7. **⚠️ `exports` 封锁了深层路径**——`import "pkg/dist/xxx"` 会报 `ERR_PACKAGE_PATH_NOT_EXPORTED`，需要显式放行。
8. **⚠️ 以为 `esModuleInterop` 只是类型层面的事**——它还改产物。只开 `allowSyntheticDefaultImports` 不开它，类型过了运行时报错。
9. **⚠️ 以为 CJS 也能写 `import`**——CJS 里 `import` 是语法错误；反过来 ESM 里 `require`/`module.exports` 也不行。
10. **⚠️ 库代码里用顶层 await**——会把所有依赖你的模块的求值一起阻塞。

## 小结

- ESM 是**语言标准**（`import`/`export`，浏览器原生支持），CJS 是 Node 的**历史传统**（`require`/`module.exports`，浏览器不能直接跑）。
- 核心差异：**静态 vs 动态**加载、**编译期 vs 运行期**解析、**导入提升**（ESM 提升，CJS 不提升）、**活绑定 vs 快照拷贝**、**顶层 await**（ESM 独有）、**`__dirname` vs `import.meta.url`**。
- `"type": "module"` 决定**整个包**的 `.js` 按 ESM 处理；`.mjs` 恒为 ESM、`.cjs` 恒为 CJS。
- `exports` 字段是现代包入口声明，也是**封装边界**（深层路径默认被封锁）；**条件导出**（`import`/`require`/`node`/`browser`/`default`）是「一个包支持两种模块系统」的标准做法；⚠️ `types` 必须排第一。
- **动态 `import()`** 返回 Promise，用于按需加载与代码分割；CJS 里也可用。
- `esModuleInterop` 让默认导入能对接 CJS 模块：类型上放宽，产物上插垫片；**库作者应优先用命名导出**。
- Node ESM 硬规则：**相对导入必须带完整扩展名**（`./a.js`），TS 里用后缀替换规则映射到 `.ts` 源文件。
- 顶层 await 只有 ESM 有；代价是阻塞依赖图求值，**库代码别用**。

## 练习

1. 建一个 `type: "module"` 的项目，把下面两行分别写进 `main.ts`，观察报错差异并解释原因：
   ```ts
   const x = require("./utils.js");
   console.log(__dirname);
   ```
2. 在 ESM 文件里写 `import { add } from "./utils"`（漏扩展名），运行 `node dist/main.js` 看报错原文；然后改成 `"./utils.js"`，再跑一次。解释为什么 TS 允许你写 `.js` 却能编译到 `utils.ts`。
3. 写一个 CJS 模块 `module.exports = { hello: () => "hi" }` 和一个 ESM 模块 `export function hello() { return "hi" }`，用 ESM 分别默认导入和命名导入它们。观察哪种写法在哪种模块上直接报错、哪种需要 `esModuleInterop`。
4. 动手搭一个带条件导出的包：`exports` 里配 `types` / `import` / `require` 三个条件，然后分别在 ESM 和 CJS 两个消费方项目里 `import` 它，打印两种情况下实际加载的是哪个文件。
5. 把 `exports` 里 `types` 挪到 `import` 后面，重新在 TS 项目里引入这个包，观察编辑器里类型是否消失。解释「条件匹配按顺序、第一个命中即生效」这句话在 `exports` 解析中的含义。
6. 用动态 `import()` 写一个「首次调用时才加载」的模块，并在加载前后各打一条日志，验证模块体确实只执行了一次。
7. 查一下 `moduleResolution: "bundler"` 和 `"nodenext"` 下，分别写 `import "./utils"` 和 `import "./utils.js"` 各自是否通过类型检查，并预测产物直接交给 Node 跑时哪个会挂。

## 导航

[[typescript/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[typescript/02-中段阶段-专题精讲/工程化与工具链/1-编译流程与tsconfig]] · 下一篇：[[typescript/02-中段阶段-专题精讲/工程化与工具链/3-构建工具与包管理]]
