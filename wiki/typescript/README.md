---
tags: [typescript, 导航, 生态连接, langchain, langgraph]
type: moc
created: 2026-10-22
updated: 2026-10-22
---

# TypeScript 知识库

> **TypeScript 是 JavaScript 的超集**——给 JS 加一层编译期类型检查，编译后产出纯 JS，在 Node.js、浏览器、Deno、Bun 上都能跑。
>
> **本库的三条主线**：
> 1. **语言本身**——语法 → 类型系统 → 工程化 → Node 异步（这是根）；
> 2. **AI 生态连接**——⭐ **LangChain.js / LangGraph.js 的 TypeScript 实战**（这是本库最有价值的一部分，直接对接你的 Agent 项目）；
> 3. **面试八股**——把上面两块变成「能说出口的答案」。

## 阅读前提与门槛声明

⚠️ **本目录默认读者是零基础**，且**假定你有 Python 基础**（本库读者画像）——凡是能类比 Python 的地方都直接对照，学习成本显著低于纯零基础。

| 你的情况 | 从哪开始 | 说明 |
|---------|---------|------|
| **完全没写过 TS/JS** | [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.1-TypeScript是什么与生态定位]] | 不要求任何 TS 前置知识 |
| **会 Python，想快速上手 TS** | ⭐ [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]] | 逐项对照 + ⚠️ 五个最容易照搬踩坑的点 |
| **想用 TS 写 AI Agent** | ⭐ [[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]] | **本库的核心路径**，先做选型再上手 |
| **已用 Python 写过 LangChain，想迁 TS** | [[typescript/03-生态连接-LangChain与LangGraph/6-Python与TypeScript双栈对照]] | 完整 API 对照表 + 双栈协作模式 |
| **要准备 TS 面试** | [[typescript/interview/README|TypeScript 面试题库]]（4 篇 40 题） | 题库给答案，教程给机制 |
| **只想查名词** | [[typescript/术语与关键字手册]] | 专有名词兜底索引 |

**看不懂时该退回哪里：**
- 名词看不懂 → [[typescript/术语与关键字手册]]
- 「Python 里这么写，TS 怎么写」 → [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]]
- 「TS 的类型为什么这么写」 → [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.1-类型推断与注解]]
- 卡在 Agent 写法上 → [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]]
- 卡在版本选择上 → 本页下方「版本基线」

## 版本基线（2026-10-22 核实）

⚠️ **TS 的版本节奏与 Python 完全不同**：一年一个大版本，且正处于**编译器实现切换期**——这一点直接影响你的技术选型。

| 版本 | 时间 | 状态（2026-10） | 关键点 |
|------|------|----------------|--------|
| ⭐ **7.0** | 2026-07-08 | ⭐ **当前最新** | ⭐ **Go 原生编译器**（`tsgo`），编译速度大幅提升；⚠️ **不提供 compiler API** |
| **6.0** | 2026-03-23 | 维护中 | **最后一个纯 JavaScript 实现版本**；改了默认值、废弃了一批旧配置 |
| 5.x | 2023–2025 | 维护线 | 存量项目大量停留于此，⚠️ **新项目不建议从这里起步** |

### TS 6.0 的默认值变更（升级必看）

| 配置项 | 旧默认 | 6.0 默认 | 影响 |
|--------|--------|---------|------|
| `strict` | `false` | **`true`** | 严格检查默认开启（好事，但老项目会冒出一批错误） |
| `module` | 跟 `target` 走 | **`esnext`** | 影响产物模块格式 |
| `target` | `es5` | **`es2025`** | 产物可用新语法 |
| `types` | 自动引入全部 `@types/*` | **`[]`（空）** | ⚠️ **症状是满屏「Cannot find name 'process'」**——需显式写 `"types": ["node"]` |

⚠️ **TS 6.0 废弃的项**：`target: es5`、`moduleResolution: node/node10`、AMD/UMD/SystemJS 模块、`baseUrl` 作为查找根。**TS 7.0 会把它们变成硬错误**——迁移前必须处理干净，别用 `"ignoreDeprecations": "6.0"` 掩盖。

⚠️ **TS 7.0 不提供 compiler API**：`typescript-eslint` 等直接调用 TS 内部 API 的工具需暂时留在 6.0（等 7.1 恢复 API）。这不影响业务代码，但影响你的 lint 工具链选型。

### AI 生态版本

| 包 | 版本线 | 关键事实 |
|---|-------|---------|
| `langchain` | **v1**（1.5.x） | ⭐ 入口是 `createAgent` / `tool` / `createMiddleware`；⚠️ **`createOpenAIFunctionsAgent` 与 `AgentExecutor` 已移除**（旧教程会导入失败）；遗留功能在 `@langchain/classic` |
| `@langchain/core` | v1（1.2.x） | 与 `langchain` 有 peer 版本约束，⚠️ **显式 pin 版本**，别用 `latest` |
| `@langchain/langgraph` | v1（1.4.x） | ⭐ 状态定义用 **`StateSchema`（基于 Zod）**，`Annotation.Root` 是遗留别名 |
| `zod` | v3 / v4 均可 | `langchain` 依赖 `zod ^3.25.76 \|\| ^4` |
| Node.js | **≥20**（22/24 LTS 推荐） | `langchain@1.5.x` 的 `engines.node` 是 `>=20`，不是 22 |

⚠️ **版本配套必须显式锁**——LangChain.js 系列发版频繁，`langchain` / `@langchain/core` / `@langchain/openai` 三者的 peer 约束不同步就会装出一堆 peer 警告。

## 目录结构

```
typescript/
├── README.md                       ← 本页：导航 + 门槛 + 版本基线
├── 术语与关键字手册.md              ← ★ 名词兜底索引
│
├── 01-入门阶段-基础教程/            ← 语言基础（零基础可读）
│   ├── 00-章节目录.md
│   ├── 第00章-学前准备/  3 篇（TS 是什么 / 环境搭建 / 与 Python 的差异）
│   ├── 第01章-核心语法/  4 篇（变量与基本类型 / 函数与参数 / 对象与数组 / 类与接口）
│   └── 第02章-类型系统/  4 篇（推断与注解 / 联合与字面量 / 泛型与约束 / 收窄与守卫）
│
├── 02-中段阶段-专题精讲/            ← 工程化与运行时
│   ├── 00-专题导航.md
│   ├── 工程化与工具链/  3 篇（编译流程与 tsconfig / ESM 与 CommonJS / 构建工具与包管理）
│   └── Node生态与异步/   3 篇（Node 运行时与 HTTP / Promise 与异步流 / 测试与错误处理）
│
├── 03-生态连接-LangChain与LangGraph/  ← ⭐ 本库核心：7 篇
│   ├── 00-专题导航.md
│   ├── 1-TypeScript生态定位与选型.md
│   ├── 2-LangChain-js核心API实战.md
│   ├── 3-从TypeScript到LangChain-js实战迁移.md
│   ├── 4-LangGraph-js状态图实战.md
│   ├── 5-LangGraph-js持久化与人工介入.md
│   └── 6-Python与TypeScript双栈对照.md
│
└── interview/                      ← ★ 面试八股题库（4 篇、40 题、82 条追问）
    ├── README.md
    ├── 01-核心语法与类型系统.md
    ├── 02-泛型与类型体操.md
    ├── 03-工程化与工具链.md
    └── 04-异步编程与Node.md
```

**规模**：教程正文 17 篇 + 生态连接 7 篇 + 题库 4 篇 = **28 篇**，加本页、术语手册、两级导航、题库首页共 **33 篇**。

## ⭐ 一条主线：TS → LangChain.js → LangGraph.js

这是本库与你的实际项目衔接最紧密的路径，建议按这个顺序读：

```
第 00 章 环境搭建  →  第 01 章 核心语法  →  第 02 章 类型系统  →  Node 异步  →  ⭐ 生态连接 7 篇
                                                                    ↓
                                                          实战项目：TS 写 Agent
```

⭐ **为什么 TypeScript 与 LangChain.js 是「天生一对」**：

| 维度 | 说明 |
|------|------|
| **Zod 一套 schema 两用** | ⭐ 同一份 Zod schema **既是运行时校验器，又是静态类型来源**（`z.infer`）——这是 TS 侧相对 Python Pydantic 的核心优势 |
| **端到端类型贯通** | 工具参数 → Agent 输入 → 结构化输出，全链路无 `any` |
| **同构部署** | 前端、Node 服务端、边缘运行时（Cloudflare Workers）一套代码 |
| **`StateSchema` 天然契合泛型** | `typeof State.State` / `typeof State.Update` 直接推导节点签名 |

⚠️ **但要诚实**：**Python 版生态更全**——LangGraph 的新特性与第三方 checkpointer 后端总是 Python 先出，TS 侧跟进。详见 [[typescript/03-生态连接-LangChain与LangGraph/6-Python与TypeScript双栈对照]]。

## 与已有内容的衔接

| 相关页面 | 关系 |
|---------|------|
| [[langchain/README\|langchain]] | ⭐ **Python 版 LangChain 知识库（143 篇）**——本库 TS 侧的对偶内容 |
| [[langgraph/README\|langgraph]] | ⭐ **Python 版 LangGraph 知识库（51 篇 + 题库）**——同上 |
| [[langchain/modules/langchain-mcp/概述\|MCP 协议]] | 双栈协作的桥梁：Python 起 MCP Server，TS 接 MCP Client |
| [[python/basics/Python_基础概览\|Python 知识库]] | **并列的姊妹库**。差异对照见 [[typescript/01-入门阶段-基础教程/第00章-学前准备/0.3-TypeScript与Python的差异]] |
| [[agent/interview/README\|Agent 面试题库]] | 跨主题 Agent 题库；本库 TS 题库是语言侧补充 |
| [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] | Node 服务容器化（JVM 容器内存参数是常见坑，Node 有自己的坑） |
| [[fastapi/项目结构]] | 对照：Python 后端分层 vs Node/TS 后端分层 |

## 一句话技术画像

| 维度 | 特点 |
|------|------|
| 定位 | JavaScript 的**编译期超集**；类型在编译后被完全擦除 |
| 类型系统 | **结构化类型**（形状匹配即兼容）；静态强类型 |
| 运行方式 | `tsc` → JS → Node/浏览器；或 `tsx`/Deno/Bun 直接跑 TS |
| 模块系统 | ESM（`import`/`export`）为主，CJS 兼容 |
| 包管理 | npm / pnpm / yarn / bun |
| 构建工具 | `tsc`（类型检查权威）、esbuild（转译）、vite（开发）、tsup（库打包） |
| AI 生态 | ⭐ **LangChain.js v1 / LangGraph.js v1 / Zod / MCP** |
| 典型岗位 | 前端工程、全栈、Node 后端、AI 应用工程、SDK/库开发 |
