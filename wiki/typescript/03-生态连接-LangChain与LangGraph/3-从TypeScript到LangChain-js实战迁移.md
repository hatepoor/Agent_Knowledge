---
tags: [typescript, langchain, agent, 迁移, 测试]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# 从 TypeScript 到 LangChain.js 实战迁移

> **什么时候该读**：你已经会写 TS、读过前两篇，想把一个「Python 版 AI 服务」或一个「纯 TypeScript 想法」真正落成一个**可上线、可测试**的 LangChain.js 项目。
> **阅读门槛**：读完 [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]]（会写 `tool()` / `createAgent` / `createMiddleware`），以及 [[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]]（会用 Vitest 的 `describe` / `it` / `expect` / `vi.fn`）。

## 学习目标

读完本篇，你应当能够：

1. 从零写出一个**端到端完整**的天气 Agent 项目：工具定义、错误处理、结构化输出、测试
2. 让 Agent 的**输入输出类型端到端类型安全**（Zod schema → `z.infer` → 工具参数 → 结构化响应）
3. 把本库 Python 侧的 RAG / Agent 概念**逐一映射**到 TS 侧，并说出每处的 API 差异
4. 处理**工具抛错**的两条路线：让 Agent 继续 vs 让整个调用失败
5. 用 **Vitest + 假模型**（`FakeListChatModel`）测 Agent 行为，**不产生任何 API 费用**
6. 避开**从 Python 照搬时最容易踩的 5 个坑**
7. 接上 **LangSmith 追踪**，在 UI 里回放 Agent 的每一步

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]] | 本篇的所有 API 都建立在那篇之上 |
| [[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]] | 测试策略那一节要用 Vitest 和 mock |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]] | 错误处理那一节要用 `unknown` 收窄 |
| [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]] | ESM 下相对导入要不要写扩展名——这是第 5 节要讲的坑 |
| [[langchain/tools/工具]] | Python 侧的工具机制，用于第 3 节的概念映射 |

---

## 一、⭐ 一个完整的端到端项目

### 1.1 项目结构

```
weather-agent/
├── src/
│   ├── schemas.ts       # ① 所有 Zod schema（唯一真相源）
│   ├── errors.ts        # ② 自定义错误类型
│   ├── tools.ts         # ③ 工具定义
│   ├── agent.ts         # ④ Agent 装配（模型用依赖注入传入）
│   └── index.ts         # ⑤ 入口
├── tests/
│   ├── setup.ts         # ⑥ Vitest 全局设置（注册 langchainMatchers）
│   └── agent.test.ts    # ⑦ 测试
├── package.json
└── tsconfig.json
```

⭐ **这个结构的核心思想**：**所有 schema 集中在 `schemas.ts`**。工具、Agent、测试全部从它派生类型——改一个字段，全链路跟着变。

### 1.2 ① schemas.ts：唯一真相源

```ts
// src/schemas.ts
import * as z from "zod";

/** 工具入参 schema：模型看到的就是它 */
export const WeatherQuerySchema = z.object({
  city: z.string().describe("要查询天气的城市名，例如「北京」「Tokyo」"),
  unit: z.enum(["celsius", "fahrenheit"]).default("celsius").describe("温度单位"),
});

/** 从 schema 推导类型——不手写 interface */
export type WeatherQuery = z.infer<typeof WeatherQuerySchema>;

/** 工具返回值 schema */
export const WeatherReportSchema = z.object({
  city: z.string(),
  temperature: z.number(),
  unit: z.enum(["celsius", "fahrenheit"]),
  condition: z.string(),
  humidity: z.number().min(0).max(100),
});

export type WeatherReport = z.infer<typeof WeatherReportSchema>;

/** Agent 的结构化输出 schema */
export const AnswerSchema = z.object({
  answer: z.string().describe("对用户问题的最终回答"),
  city: z.string().describe("实际查询的城市"),
  usedTool: z.boolean().describe("是否调用了天气工具"),
  confidence: z.number().min(0).max(1),
});

export type Answer = z.infer<typeof AnswerSchema>;
```

### 1.3 ② tools.ts：工具定义 + 错误处理

```ts
// src/tools.ts
import { tool } from "langchain";
import * as z from "zod";
import { WeatherQuerySchema, type WeatherReport } from "./schemas.js";
import { WeatherServiceError } from "./errors.js";

/** 假的天气服务——真实项目里换成 fetch / 数据库 */
const FAKE_DB: Record<string, Omit<WeatherReport, "city" | "unit">> = {
  北京: { temperature: 22, condition: "晴", humidity: 45 },
  Tokyo: { temperature: 18, condition: "多云", humidity: 60 },
};

export const getWeather = tool(
  // ⭐ 参数类型直接由 schema 推导；框架已经用 schema 校验过并填好了默认值
  async ({ city, unit }: WeatherQuery): Promise<string> => {
    const record = FAKE_DB[city];
    if (!record) {
      // ⚠️ 关键：查不到不是「崩溃」，是一个模型可以理解并向用户解释的结果
      return JSON.stringify({ error: `没有找到城市「${city}」的天气数据` });
    }

    const report: WeatherReport = { city, unit, ...record };
    return JSON.stringify(report);
  },
  {
    name: "get_weather",
    description:
      "查询指定城市的当前天气（温度、天气状况、湿度）。" +
      "当用户询问天气、温度、湿度或'会不会下雨'时使用。" +
      "不支持历史天气查询和天气预报——那些问题应直接告知用户本工具无法处理。",
    schema: WeatherQuerySchema,
  },
);
```

⭐ **注意 `unit` 为什么不用自己兜底**：`tool()` 的内部实现会先对模型传来的参数跑一次 **Zod 校验**（`interopParseAsync`），校验通过后**把解析后的值**交给你的函数——所以：

- `city` 一定是 `string`（模型传了数字会被拦下）；
- `unit` 的 `.default("celsius")` **已经被填好**（模型不传就是 `"celsius"`）；
- 校验失败时框架抛 `ToolInputParsingException`，**不会**调到你的函数。

⭐ **这就是「Zod schema 兼作运行时校验器」的真正含义**——不是「你记得调 `parse`」，而是「**框架替你调**」。

> ⚠️ 但**你的函数也可能被别人直接调用**（比如单元测试里 `getWeather.invoke({...})`）。那种情况下没人替你校验，所以**不要把「参数一定合法」当成可以省略校验的理由**——真正的保证来自类型系统，而不是运行时的运气。

### 1.4 ③ errors.ts：自定义错误类型

```ts
// src/errors.ts
/** 业务错误：可预期的失败（城市不存在、服务不可用） */
export class WeatherServiceError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "WeatherServiceError";
    // ⭐ 修复 ES5 target 下 instanceof 失效的问题
    Object.setPrototypeOf(this, WeatherServiceError.prototype);
  }
}
```

### 1.5 ④ agent.ts：装配

```ts
// src/agent.ts
import { createAgent, type BaseChatModel } from "langchain";
import { getWeather } from "./tools.js";
import { AnswerSchema } from "./schemas.js";

export function createWeatherAgent(model: BaseChatModel) {
  return createAgent({
    model,                                  // ⭐ 由调用方注入——测试时注入假模型
    tools: [getWeather],
    systemPrompt:
      "你是一个天气助手。查询天气时必须调用 get_weather 工具，不要凭记忆作答。" +
      "工具返回错误时，把错误如实告诉用户并建议换个城市名。",
    responseFormat: AnswerSchema,            // ⭐ 结构化输出
  });
}
```

⭐ **为什么把 `model` 做成参数**：这是**依赖注入**（dependency injection）——生产环境传真模型，测试环境传假模型（`fakeModel()` 返回的就是一个 `BaseChatModel` 子类），**同一份 Agent 逻辑**。

> 📖 依赖注入相比模块 mock 的优势，见 [[typescript/02-中段阶段-专题精讲/Node生态与异步/3-测试与错误处理]]。

### 1.6 ⑤ index.ts：入口

```ts
// src/index.ts
import { initChatModel } from "langchain";
import { createWeatherAgent } from "./agent.js";

const model = await initChatModel("openai:gpt-4o", { temperature: 0.2 });
const agent = createWeatherAgent(model);

const result = await agent.invoke({
  messages: [{ role: "user", content: "北京现在多少度？" }],
});

console.log(result.structuredResponse);   // 类型是 Answer
```

### 1.7 完整跑通

```bash
npm install langchain@1.5 @langchain/core@1.2 @langchain/openai@1.6 zod@4
export OPENAI_API_KEY="sk-..."
npx tsx src/index.ts
```

⭐ **版本要配套**：`langchain@1.5.x` 的 peer 依赖是 `@langchain/core@^1.2.20`——所以 core 要锁在 **1.2.x**，不能随手升到 1.3.x（会 peer 冲突）。模型集成包（`@langchain/openai` 等）也要挑**要求同一个 core 版本线**的版本（`@langchain/openai@1.6.x` → `core ^1.2.14`）。

---

## 二、⭐ 类型贯通实践

### 2.1 完整链路

⭐ **这张图是本篇的核心价值**：

```
        WeatherQuerySchema                ← 唯一真相源（Zod）
                 │
     ┌───────────┴───────────┐
     │                       │
   z.infer              直接给 tool()
     │                       │
     ▼                       ▼
WeatherQuery            模型看到的 JSON Schema
(工具函数的参数类型)      (运行时还会用它校验模型传来的参数)
     │
     │ 工具返回 WeatherReport
     ▼
  WeatherReport
     │
     │ Agent 的 responseFormat: AnswerSchema
     ▼
       Answer  ← result.structuredResponse 的类型
```

⭐ **四个「不可能出错」的地方**：

| 环节 | 保证 |
|------|------|
| 模型生成的参数 | `WeatherQuerySchema` 由**框架在运行时校验过**（`tool()` 内部跑 Zod）；工具函数收到的 `city` 一定是 `string`，`unit` 的默认值已填好 |
| 工具函数的参数类型 | `z.infer` 推导，和 schema **永远一致** |
| 工具的返回值 | 显式标注 `: Promise<string>`，且内部构造的对象满足 `WeatherReport` |
| Agent 的最终输出 | `result.structuredResponse` 的类型是 `Answer`，**不是 `any`** |

⭐ **注意这个保证的边界**：框架校验的是「**从模型进来的参数**」。你的函数**被别的代码直接调用**时（单元测试、别的模块 import），没有人替你校验——那条路径要自己 `safeParse`（见 4.6）。

### 2.2 端到端安全的写法

```ts
// src/tools.ts
import { tool } from "langchain";
import { WeatherQuerySchema, type WeatherQuery, type WeatherReport } from "./schemas.js";

export const getWeather = tool(
  // ⭐ 参数类型直接来自 schema —— 不可能对不上
  async ({ city, unit }: WeatherQuery): Promise<string> => {
    const record = lookup(city);
    const report: WeatherReport = {
      city,
      unit,                                  // ⭐ TS 知道 unit 是 "celsius" | "fahrenheit"
      temperature: record.temperature,
      condition: record.condition,
      humidity: record.humidity,
    };
    return JSON.stringify(report);
  },
  { name: "get_weather", description: "…", schema: WeatherQuerySchema },
);
```

### 2.3 ⭐ 结构化输出的类型是「免费」的

```ts
const result = await agent.invoke({
  messages: [{ role: "user", content: "北京现在多少度？" }],
});

// ⭐ TS 知道 structuredResponse 的确切类型
const answer: Answer = result.structuredResponse;
console.log(answer.confidence.toFixed(2));   // ⭐ confidence 是 number，.toFixed 合法
```

⚠️ **对照 Python**：Python 侧 `result["structured_response"]` 拿到的是 `ContactInfo` **实例**（类型靠 `mypy` 保证，但运行时未必是）；TS 侧拿到的是**普通对象**，类型是**编译期**推导的、运行时由 Zod 校验过的。**TS 侧在运行时多了一层保险**。

### 2.4 一个类型贯通的反例

```ts
// ❌ 类型"看起来"对了，实际是 any 逃逸
export const getWeather = tool(
  async (args: any) => {                     // ⚠️ any 逃逸：整个链路的类型安全从这一行断掉
    return JSON.stringify({ city: args.city });   // ⚠️ args.city 是 any，编译器不再检查
  },
  { name: "get_weather", description: "…", schema: WeatherQuerySchema },
);
```

⭐ **判断规则**：**函数参数上出现 `any` 或手写 `interface`，就说明类型贯通断了。** 正确做法是让参数类型从 schema 推导出来。

---

## 三、⭐ 与本库 Python 项目的概念映射

### 3.1 逐项对照表

⭐ 这张表是「已有 Python 知识」的迁移地图。**Python 列是本库里的页面**，**TS 列是本篇/上两篇的写法**。

| 概念 | Python 版（本库页面） | LangChain.js（TS） | 迁移注意 |
|------|---------------------|-------------------|---------|
| **模型** | [[langchain/models/模型]] `ChatOpenAI(model=...)` | `new ChatOpenAI({ model, temperature })` | ⚠️ Python 是关键字参数，TS 是**对象字面量** |
| **消息** | [[langchain/modules/langchain_messages/消息类型详解]] `HumanMessage(content=...)` | `new HumanMessage("...")` | ⚠️ TS 是**位置参数** |
| **工具** | [[langchain/tools/工具]] `@tool` + docstring | `tool(fn, { name, description, schema })` | ⚠️ TS 的 `description` 不写会退化成 `"<工具名> tool"`，模型基本选不出来 |
| **工具 schema** | [[langchain/tools/工具]] `class WeatherInput(BaseModel)` | `z.object({...})` | ⚠️ TS 不需要额外定义类 |
| **Agent** | [[langchain/agents/Agent代理]] `create_agent(model=..., tools=...)` | `createAgent({ model, tools })` | ⚠️ TS 是**一个对象参数**，不是关键字参数 |
| **系统提示词** | `create_agent(prompt=...)` | `createAgent({ systemPrompt })` | ⚠️ **参数改名了**：`prompt` → `systemPrompt` |
| **检索 / RAG** | [[langchain/modules/langchain_core/概述]] Retriever / VectorStore | 同名概念，`@langchain/core` 提供基础抽象 | 概念一致，TS 侧包名不同 |
| **流式** | [[langchain/streaming/流式传输]] `agent.stream(...)` | `agent.stream(input, { streamMode })` | ⚠️ TS 版**必须传 `streamMode`** |
| **结构化输出** | [[langchain/structured-output/结构化输出]] `response_format=ContactInfo` | `responseFormat: ContactInfoSchema` | ⚠️ TS 侧传 **Zod schema**，不是类 |
| **持久化 / checkpointer** | [[langgraph/modules/langgraph_checkpoint/概述]] `InMemorySaver()` | `new MemorySaver()`（来自 `@langchain/langgraph`） | ⚠️ **JS 侧叫 `MemorySaver`，Python 叫 `InMemorySaver`** |
| **多轮对话** | `config={"configurable":{"thread_id":"..."}}` | `{ configurable: { thread_id: "..." } }` | 这一项两边**完全一致** |
| **运行时上下文** | `context=AppContext(...)` | `{ context: { userId: "..." } }` | ⚠️ TS 侧是**普通对象**，不是 dataclass |
| **中间件钩子** | [[langchain/middleware/钩子机制详解]] `@before_model` | `createMiddleware({ beforeModel })` | ⚠️ Python 是装饰器，TS 是**配置对象** |
| **提前跳转** | `{"jump_to": "end"}` + `@hook_config(can_jump_to=[...])` | `{ jumpTo: "end" }` + `canJumpTo: [...]` | ⚠️ **TS 是 camelCase** |
| **MCP 工具** | [[langchain/modules/langchain-mcp/概述]] `MultiServerMCPClient` | 同名类，包是 `@langchain/mcp-adapters` | 类名一致，**npm 包名不同** |

### 3.2 一个「Python → TS」的重写对照

```python
# Python 版（本库 langchain/tools/工具 的写法）
from langchain.tools import tool

@tool
def get_weather(city: str, units: str = "celsius") -> str:
    """Get current weather for a city."""
    return f"It's always sunny in {city}!"
```

```ts
// TS 版
import { tool } from "langchain";
import * as z from "zod";

const getWeather = tool(
  ({ city, units }: { city: string; units?: "celsius" | "fahrenheit" }) =>
    `It's always sunny in ${city}!`,
  {
    name: "get_weather",                                  // ⚠️ TS 必须显式给 name
    description: "Get current weather for a city.",        // ⭐ TS 建议显式给 description
    schema: z.object({
      city: z.string(),
      units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
    }),
  },
);
```

⭐ **四条硬差异**：

| # | Python | TS |
|---|--------|-----|
| 1 | `@tool` 装饰器 | `tool(fn, config)` |
| 2 | 描述来自 **docstring** | 描述来自 **`config.description`**（不写会退化成 `"<工具名> tool"`，模型基本选不出来） |
| 3 | 参数类型来自 **Python 注解**（运行时反射） | 参数类型来自 **Zod schema**（显式声明） |
| 4 | 「必填/可选」由**有无默认值**决定 | 「必填/可选」由 **`.optional()` / `.default()`** 决定 |

---

## 四、⭐ 错误处理模式

### 4.1 核心问题：工具抛错，Agent 该不该挂？

⭐ **默认答案是不挂。** 框架内置的工具错误处理器（`defaultHandleToolErrors`）会把工具抛出的错误转成一条 **`status: "error"` 的 `ToolMessage`** 回喂给模型，模型看到后可以：换个参数重试、换个工具、或者如实告诉用户「查不到」。

这正是 Agent 相对固定流程的优势——**错误是一次信息，不是一次终止**。

⭐ **两种错误消息的差别**：

| 错误类型 | 消息内容 |
|---------|---------|
| `ToolInvocationError`（工具执行失败） | 原始 `error.message` |
| 其他异常 | `${error}\n Please fix your mistakes.` |

⚠️ **一个重要例外**：`wrapToolCall` 中间件里抛出的错误**不走这个默认处理器**，会**向上冒泡**导致整个调用失败（这符合直觉——中间件是你自己写的，你希望出错时立刻暴露）。

### 4.2 三种错误等级

| 等级 | 例子 | 处理方式 |
|------|------|---------|
| **可自愈** | 城市名拼错、参数缺了 | ⭐ **返回错误字符串**（不是抛错），让模型自己纠正 |
| **可重试** | 网络超时、限流 | ⭐ **抛错**，配 `toolRetryMiddleware` 自动重试 |
| **不可恢复** | 鉴权失败、配置缺失 | **抛错并让整个调用失败**（`onFailure: "error"`） |

### 4.3 模式 A：返回错误对象（让 Agent 继续）

```ts
export const getWeather = tool(
  async ({ city }: WeatherQuery): Promise<string> => {
    const record = lookup(city);
    if (!record) {
      // ⭐ 不抛错——把「查不到」当成一个正常的业务结果告诉模型
      return JSON.stringify({
        error: `没有找到「${city}」的天气数据`,
        hint: "请确认城市名，或尝试使用英文城市名（如 Beijing / Tokyo）",
      });
    }
    return JSON.stringify(record);
  },
  { name: "get_weather", description: "…", schema: WeatherQuerySchema },
);
```

⭐ **为什么这样更好**：模型看到 `hint` 后，很可能自己就换个城市名重试了——**不需要写任何重试逻辑**。

### 4.4 模式 B：抛错 + 自动重试

```ts
import { createAgent, toolRetryMiddleware } from "langchain";

const agent = createAgent({
  model,
  tools: [fetchRemoteData],
  middleware: [
    toolRetryMiddleware({
      maxRetries: 3,
      backoffFactor: 2.0,        // 指数退避
      initialDelayMs: 1000,
      tools: ["fetch_remote_data"],   // 只对特定工具重试
      onFailure: "continue",     // ⭐ 重试耗尽后：把错误变成 ToolMessage 继续，而不是整体失败
    }),
  ],
});
```

⭐ **`onFailure` 的取值决定最终行为**：

| 值 | 行为 |
|----|------|
| `"continue"`（默认） | 重试耗尽后生成一条含错误详情的 `ToolMessage`，**Agent 继续跑**，模型能看到错误并自己想办法 |
| `"error"` | 重新抛出异常，**整个 Agent 调用失败** |
| 自定义函数 | 返回你指定的字符串作为错误消息内容 |

### 4.5 模式 C：让整个调用失败

```ts
try {
  const result = await agent.invoke({
    messages: [{ role: "user", content: "北京现在多少度？" }],
  });
  console.log(result.structuredResponse);
} catch (err: unknown) {
  // ⭐ 用 unknown 收窄——TS 的 catch 变量默认是 unknown
  if (err instanceof WeatherServiceError) {
    console.error(`天气服务不可用：${err.message}`);
  } else {
    console.error("Agent 执行失败：", err);
  }
  process.exitCode = 1;
}
```

### 4.6 ⭐ Zod 校验失败的处理

⭐ **先分清「谁在校验」**：

| 场景 | 谁来校验 | 失败时 |
|------|---------|--------|
| **模型调用工具** | ⭐ **框架自动校验**（`tool()` 内部跑 Zod） | 抛 `ToolInputParsingException`，**你的函数根本不会被调用**；错误被包成 `status: "error"` 的 `ToolMessage` 回喂模型 |
| **你的代码直接调工具**（单元测试、别的模块直接 import） | **没人替你校验** | 得自己 `safeParse` |

⭐ 所以「在自己的代码里显式校验」不是重复劳动，而是**补上框架管不到的那条路径**：

```ts
import * as z from "zod";

const parsed = WeatherQuerySchema.safeParse(rawInput);

if (!parsed.success) {
  // ⭐ Zod 的错误对象里有 issue 列表，每条有 path 和 message
  const issues = parsed.error.issues.map(
    (i) => `${i.path.join(".") || "(根)"}: ${i.message}`,
  );
  throw new WeatherServiceError(`参数校验失败：${issues.join("; ")}`);
}

// ⭐ 校验通过后，parsed.data 的类型已经被收窄成 WeatherQuery
const { city } = parsed.data;
```

⚠️ **`safeParse` vs `parse`**：`parse` 成功返回数据、失败**直接抛 ZodError**；`safeParse` 返回 `{ success, data } | { success, error }` 的**判别联合**，需要自己处理失败分支。业务代码用 `safeParse` 更可控。

> 📖 `safeParse` 返回的是判别联合，用 `success` 收窄的原理见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.2-联合类型与字面量类型]]；错误收窄见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.4-类型收窄与类型守卫]]。

### 4.7 给工具加超时（必做）

```ts
const res = await fetch(url, {
  signal: AbortSignal.timeout(10_000),   // ⭐ 10 秒没响应就放弃
});
```

⭐ **为什么必须有**：Node 侧的网络请求默认**没有超时**。下游挂住 → 工具永不返回 → **Agent 循环卡死**。配合 `toolCallLimitMiddleware` 可以再加一道保险。

---

## 五、⭐ 测试策略

### 5.1 分层

| 层 | 测什么 | 用什么 | 成本 |
|----|-------|-------|------|
| **单元测试** | 工具函数的纯逻辑、schema 校验 | 直接调用工具 + `expect` | 免费、毫秒级 |
| **Agent 行为测试** | Agent 的**编排逻辑**（调了哪个工具、传了什么参） | ⭐ **假模型**（`FakeListChatModel`） | 免费、毫秒级 |
| **集成测试** | 真模型 + 真工具 | 真 API（CI 里用小模型） | 花钱、秒级 |

⭐ **核心原则**：**能用假模型测的，绝不调真模型。**

### 5.2 LangChain.js v1 自带的测试工具

⭐ **v1 提供了现成的假模型，不需要自己写 mock**：

| 入口 | 位置 | 特点 |
|------|------|------|
| **`fakeModel()`** ⭐ 推荐 | `langchain` 主包（也来自 `@langchain/core/testing`） | **链式 builder**：排队响应、排队工具调用、设结构化输出、记录每次调用 |
| **`FakeToolCallingModel`** | `langchain` 主包 | 直接给一个「工具调用序列」的模型，带 `structuredResponse` |
| `FakeListChatModel` / `FakeChatModel` / `FakeStreamingChatModel` | `@langchain/core/utils/testing` | 更底层：按字符串列表依次返回；⚠️ **它们的 `responses` 只接受 `string[]`，不能塞 `AIMessage`** |
| `FakeEmbeddings` / `FakeVectorStore` / `FakeRetriever` | `@langchain/core/utils/testing` | RAG 链路各组件的假实现 |

⭐ **首选 `fakeModel()`**——它是 v1 专门为「测 Agent」设计的，工具调用、结构化输出、调用历史都开箱可用。

```ts
import { fakeModel, AIMessage } from "langchain";

const model = fakeModel()
  .respondWithTools([{ name: "get_weather", args: { city: "北京" } }])   // 第 1 次调用：要求调工具
  .respond(new AIMessage("北京现在 22 度，晴。"));                          // 第 2 次调用：最终答案
```

**`fakeModel()` 的 API 速查**：

| 方法 | 作用 |
|------|------|
| `.respond(message \| error \| factory)` | 排队一个响应（`BaseMessage` / `Error` / `(messages) => message` 函数） |
| `.respondWithTools([{ name, args, id? }])` | 排队一个「带工具调用」的 `AIMessage`（`id` 可省略，自动生成） |
| `.alwaysThrow(error)` | 让**每次**调用都抛这个错 |
| `.structuredResponse(value)` | 设 `withStructuredOutput()` 要返回的值 |
| `.bindTools(tools)` | 绑定工具（返回共享同一个响应队列的新实例） |
| `.calls` | **每次调用的 `{ messages, options }` 记录**（只读数组） |
| `.callCount` | 被调用的次数 |

⭐ **注意**：响应是 **FIFO（先进先出）**的——排队几个就按顺序返回几个；**队列空了之后再调用会抛错**。

### 5.3 ⭐ 测「Agent 会不会调工具」

⭐ **这是 Agent 测试的核心问题**：不是「回答对不对」（那是模型的活），而是「**流程走对了吗**」。

```ts
// tests/agent.test.ts
import { describe, it, expect } from "vitest";
import { fakeModel, AIMessage } from "langchain";
import { createWeatherAgent } from "../src/agent.js";

describe("天气 Agent", () => {
  it("用户问天气时应该调用 get_weather 工具", async () => {
    // ⭐ 假模型：第 1 次调用要求调工具，第 2 次调用给最终答案
    const model = fakeModel()
      .respondWithTools([{ name: "get_weather", args: { city: "北京" } }])
      .respond(new AIMessage("北京现在 22 度，晴。"));

    const agent = createWeatherAgent(model);
    const result = await agent.invoke({
      messages: [{ role: "user", content: "北京现在多少度？" }],
    });

    // ⭐ 断言：模型被调用了 2 次（一次要工具，一次收尾）
    expect(model.callCount).toBe(2);

    // ⭐ 断言：确实产生了工具消息，且内容里有工具的返回值
    const toolMessages = result.messages.filter((m) => m.getType() === "tool");
    expect(toolMessages.length).toBeGreaterThan(0);
    expect(toolMessages[0]!.text).toContain("22");   // 工具真的执行了
  });
});
```

### 5.4 ⭐ 用 `vi.fn()` 断言「工具被调用了、参数是什么」

⭐ **比上面更好的做法**：把工具本身 mock 掉，这样能精确断言参数。

```ts
import { describe, it, expect, vi } from "vitest";
import { fakeModel, tool, AIMessage, createAgent } from "langchain";
import * as z from "zod";

describe("Agent 编排行为", () => {
  it("应该用正确的城市参数调用天气工具", async () => {
    // ⭐ 造一个真的 LangChain 工具，但函数体是 spy
    const spy = vi.fn(async ({ city }: { city: string }) =>
      JSON.stringify({ city, temperature: 22, condition: "晴", humidity: 45 }),
    );

    const fakeWeatherTool = tool(spy, {
      name: "get_weather",
      description: "查询天气",
      schema: z.object({ city: z.string() }),
    });

    const model = fakeModel()
      .respondWithTools([{ name: "get_weather", args: { city: "北京" } }])
      .respond(new AIMessage("22 度，晴。"));

    const agent = createAgent({ model, tools: [fakeWeatherTool] });

    await agent.invoke({ messages: [{ role: "user", content: "北京天气" }] });

    // ⭐ 精确断言：调用了几次、传了什么参数
    expect(spy).toHaveBeenCalledTimes(1);
    expect(spy.mock.calls[0]![0]).toEqual({ city: "北京" });
  });
});
```

⭐ **`spy.mock.calls[0]![0]`** 取的是「第一次调用的第一个参数」——也就是工具收到的入参。用它比 `toHaveBeenCalledWith` 更稳（不用管后面还有没有运行时配置参数）。

### 5.5 ⭐ 断言结构化输出

```ts
it("应该返回符合 schema 的结构化响应", async () => {
  const model = fakeModel()
    .respondWithTools([{ name: "get_weather", args: { city: "北京" } }])
    .structuredResponse({                       // ⭐ 设好结构化输出要返回什么
      answer: "北京现在 22 度，晴。",
      city: "北京",
      usedTool: true,
      confidence: 0.95,
    });

  const result = await createWeatherAgent(model).invoke({
    messages: [{ role: "user", content: "北京天气" }],
  });

  expect(result.structuredResponse.city).toBe("北京");
  expect(result.structuredResponse.usedTool).toBe(true);
});
```

### 5.6 ⭐ 用 `langchainMatchers` 断言消息结构

v1 还提供了一组 **Vitest 断言扩展**（`expect.extend` 后可用）：

```ts
// tests/setup.ts
import { expect } from "vitest";
import { langchainMatchers } from "@langchain/core/testing";

expect.extend(langchainMatchers);
```

```ts
import { describe, it, expect } from "vitest";
import { fakeModel, AIMessage } from "langchain";
import { createWeatherAgent } from "../src/agent.js";

it("应该产生一次工具调用", async () => {
  const model = fakeModel()
    .respondWithTools([{ name: "get_weather", args: { city: "北京" } }])
    .respond(new AIMessage("22 度，晴。"));

  const result = await createWeatherAgent(model).invoke({
    messages: [{ role: "user", content: "北京天气" }],
  });

  expect(result.messages).toHaveToolCallCount(1);      // ⭐ 恰好 1 次工具调用
  expect(result.messages).toContainToolCall({ name: "get_weather" });
});
```

⭐ **可用 matcher 一览**：`toBeHumanMessage` / `toBeAIMessage` / `toBeSystemMessage` / `toBeToolMessage` / `toHaveToolCalls` / `toHaveToolCallCount` / `toContainToolCall` / `toHaveToolMessages` / `toHaveBeenInterrupted` / `toHaveStructuredResponse`，以及流式专用的 `toHaveStreamText` / `toHaveStreamToolCalls` / `toHaveStreamUsage` / `toHaveStreamOutput`。

### 5.7 测「工具出错时 Agent 怎么办」

⭐ **框架的默认行为**：工具抛错 → 被转成一条 `status: "error"` 的 `ToolMessage` 回喂给模型 → **Agent 继续跑**。

```ts
it("工具查不到城市时应该把错误如实返回而不是崩溃", async () => {
  const model = fakeModel()
    .respondWithTools([{ name: "get_weather", args: { city: "不存在的城市" } }])
    .respond(new AIMessage("抱歉，没有找到该城市的天气数据。"));

  const result = await createWeatherAgent(model).invoke({
    messages: [{ role: "user", content: "查一下火星天气" }],
  });

  // ⭐ 关键断言：调用**没有抛错**，且最终有回答
  expect(result.messages.at(-1)?.text).toBeTruthy();
});
```

### 5.8 测工具本身（纯单元测试）

```ts
import { describe, it, expect } from "vitest";
import { WeatherQuerySchema } from "../src/schemas.js";

describe("WeatherQuerySchema", () => {
  it("应该接受城市名 + 默认单位", () => {
    const r = WeatherQuerySchema.safeParse({ city: "北京" });
    expect(r.success).toBe(true);
    if (r.success) {
      expect(r.data.unit).toBe("celsius");   // ⭐ 默认值生效
    }
  });

  it("应该拒绝缺少 city 的输入", () => {
    const r = WeatherQuerySchema.safeParse({});
    expect(r.success).toBe(false);
  });

  it("应该拒绝非法的 unit", () => {
    const r = WeatherQuerySchema.safeParse({ city: "北京", unit: "kelvin" });
    expect(r.success).toBe(false);
  });
});
```

⭐ **注意 `if (r.success)` 这个收窄**：`safeParse` 返回判别联合，不收窄就拿不到 `r.data`。

### 5.9 测试跑起来

```bash
npx vitest run          # 跑一遍
npx vitest              # watch 模式
```

⚠️ **单元测试里绝不能出现真 API key 依赖**。用假模型，测试就完全不花钱、不联网、可离线跑。

---

## 六、⚠️ 五个最容易踩的坑（从 Python 照搬 TS 时）

### 坑 1：⚠️ 以为类型注解在运行时存在

**Python 里的现实**：

```python
def get_weather(city: str) -> str:
    return f"{city} 的天气"
# 调用方传个 int 进去，Python **完全不报错**——注解只是给 IDE 和 mypy 看的
```

⭐ **TypeScript 的现实**：TS 的类型**同样在编译后被完全擦除**。运行时没有任何类型信息。

```ts
function getWeather(city: string): string { return `${city} 的天气`; }
// 来自"网络"的脏数据
const dirty = JSON.parse('{"city": 42}') as any;
getWeather(dirty.city);   // ⭐ 运行时 city 是数字，模板字符串照样拼出 "42 的天气"
```

⭐ **结论**：**TS 的类型注解和 Python 的一样，都是编译期的。** 想要运行时保障，只有一条路——**Zod**。

> ⚠️ **这直接推翻了「TS 类型安全所以不用校验」的错觉**。模型输出的 JSON 就是典型的「来自网络的不受信任数据」，**必须过 Zod**。

### 坑 2：⚠️ 忘了顶层 await

```ts
// ✅ 顶层 await 要求 ESM（package.json 里 "type": "module"）
const model = await initChatModel("openai:gpt-4o");
const result = await agent.invoke({
  messages: [{ role: "user", content: "北京今天天气怎么样？" }],
});
```

⚠️ **在 CommonJS 里顶层 await 是语法错误**。如果你的项目还没切到 ESM，要么把代码放进 `async main()` 里，要么改用 ESM。

⭐ **另一个更隐蔽的版本**：把 `await` 忘在**函数里**——

```ts
// ❌ 危险：忘了 await，result 是 Promise
const result = agent.invoke({
  messages: [{ role: "user", content: "北京今天天气怎么样？" }],
});
console.log(result.messages);   // ⚠️ undefined，因为 Promise 上没有 messages
```

这正是 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] 讲的坑：**忘了 await → 错误被吞 + 时序错 + 可能崩进程**。

### 坑 3：⚠️ `any` 逃逸

```ts
// ❌ 断链：从这一行起，类型安全全部失效
const args: any = input;
```

⭐ **`any` 是「类型系统的逃生舱」**——一旦用上，编译器就停止追踪这一支的类型。

⚠️ **Agent 项目里 `any` 特别容易溜进来的地方**：

| 位置 | 典型写法 | 后果 |
|------|---------|------|
| 工具参数 | `async (args: any) => ...` | 参数类型完全不校验 |
| 工具返回值 | `return JSON.parse(...)` | 返回值是 `any` |
| `catch` | `catch (e: any)` | 失去 `unknown` 收窄的保护 |
| mock 工具 | `as any` 硬转 | 测试通过但类型对不上 |

⭐ **对策**：`catch` 里用 `unknown`（不是 `any`）；工具参数用 `z.infer`；JSON.parse 后立刻过 `schema.parse()`。

### 坑 4：⚠️ 误以为 TS 有 Python 的 `Protocol` 运行时检查

**Python 的 `Protocol`**：可以做**运行时**的结构检查——

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class WeatherSource(Protocol):
    def get(self, city: str) -> str: ...

def use(src: WeatherSource) -> None: ...
use("我不是 WeatherSource")   # ⭐ Python 会抛 TypeError
```

⭐ **TypeScript 没有等价物**。TS 的 `interface` 是**纯编译期**的：

```ts
interface WeatherSource {
  get(city: string): string;
}

function use(src: WeatherSource) {
  return src.get("北京");
}
use("我不是 WeatherSource" as any);   // ⭐ 运行时啥也不检查，照样执行
```

⚠️ **TS 里最接近的东西是 `z.custom()` + `safeParse`**，那是**显式**的运行时检查，不是隐式的：

```ts
const WeatherSourceSchema = z.object({
  get: z.function(z.tuple([z.string()]), z.string()),
});
const parsed = WeatherSourceSchema.safeParse(candidate);
if (!parsed.success) throw new Error("不是合法的 WeatherSource");
```

> 📖 TS 的结构化类型（interface 只是形状约束，编译期生效）见 [[typescript/01-入门阶段-基础教程/第01章-核心语法/1.4-类与接口]]。

### 坑 5：⚠️ ESM 下相对导入缺扩展名

```ts
// ❌ ESM 下运行时报错：ERR_MODULE_NOT_FOUND
import { getWeather } from "./tools";
import { WeatherQuerySchema } from "../schemas";

// ✅ ESM 要求写完整扩展名
import { getWeather } from "./tools.js";
import { WeatherQuerySchema } from "../schemas.js";
```

⭐ **为什么写 `.js` 而不是 `.ts`**：Node 运行时执行的是**编译后的 JS**，TS 源文件里的 `./tools.ts` 在产物里不存在。写 `./tools.js` 指的是**产物里的那个文件**。TS 编译器（`moduleResolution: "node16"` / `"nodenext"` 这类按产物解析的模式）会把 `./tools.js` 解析回源文件 `./tools.ts`。

⚠️ **这个坑只在 ESM 下出现**——CommonJS 的 `require` 不带扩展名也能跑。所以「本地能跑、上了生产报 404」的现象就是这么来的。

⚠️ **一个例外**：如果你的 tsconfig 用的是 **`"moduleResolution": "bundler"`**（配 vite / tsx / esbuild 这类打包器或运行器），TS **允许**省略扩展名——因为最终产物由打包器决定。但如果你要产出**纯 Node 运行的 ESM**（`tsc` 直接编译 + `node dist/index.js`），就必须写 `.js`。

> 📖 ESM vs CommonJS 的完整差异、`"type": "module"`、条件导出，见 [[typescript/02-中段阶段-专题精讲/工程化与工具链/2-模块系统ESM与CommonJS]]。

---

## 七、可观测性：LangSmith 追踪接入

### 7.1 接入（三步）

```bash
# ① 在 smith.langchain.com 的 Settings → API Keys 生成 key
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_pt_xxxxxxxx
export LANGSMITH_PROJECT=weather-agent

# ② 非 serverless 环境：后台上报，降低延迟
export LANGCHAIN_CALLBACKS_BACKGROUND=true
#    ⚠️ serverless 环境反而要设为 false，让追踪在函数结束前提交完
```

⭐ **不需要写任何代码**——设好环境变量就跑，`createAgent` 内部的每次模型调用、每次工具执行**全部自动被追踪**。

### 7.2 能看到什么

| 内容 | 说明 |
|------|------|
| **完整调用树** | Agent → model 节点 → tools 节点 → 每个工具的入参和返回值 |
| **每步的耗时** | 哪个工具慢、哪次模型调用慢 |
| **token 用量** | 成本分析的基础 |
| **完整输入输出** | 复现「模型为什么这么答」 |
| **工具调用参数** | 排查「为什么调错了工具」 |

⭐ **这解决的是 Agent 最难调的问题**：**非确定性**。同样的输入，模型这次调 A 工具、下次调 B 工具——只有 trace 能告诉你到底发生了什么。

### 7.3 ⚠️ 追踪没出现时的排查顺序

| 顺序 | 检查 | 说明 |
|------|------|------|
| 1 | `echo $LANGSMITH_TRACING` | 应该是 `true` |
| 2 | API Key 和项目名 | Key 正确；`LANGSMITH_PROJECT` 拼写（拼错不会报错，只是进了 `default` 项目） |
| 3 | 区域 endpoint | ⭐ **非美区必须设 `LANGSMITH_ENDPOINT`**，否则认证失败。比如 EU 区是 `https://eu.api.smith.langchain.com` |
| 4 | serverless 环境 | `LANGCHAIN_CALLBACKS_BACKGROUND=false`，否则函数先结束、追踪后丢 |
| 5 | 确认真的调了 LangChain | 普通 `console.log` 不会产生 trace，必须跑过 `invoke` / `stream` |

### 7.4 只追踪部分调用

```ts
import { LangChainTracer } from "@langchain/core/tracers/tracer_langchain";

const tracer = new LangChainTracer({ projectName: "weather-agent-debug" });

await agent.invoke(
  { messages: [{ role: "user", content: "北京天气" }] },
  { callbacks: [tracer] },   // ⭐ 只给这一次调用挂追踪
);
```

### 7.5 ⚠️ 追踪数据的敏感性

⭐ **开了追踪就意味着你的数据上了第三方服务**。在把追踪接进生产前必须确认：

- 用户输入里**有没有 PII**（身份证、手机号、地址）？→ 用 `piiRedactionMiddleware` 脱敏，**但注意它只保护发给模型的那一段**，追踪里仍然是原文；
- 有没有**合规要求**禁止数据出境？
- 追踪保留期是多久？

> 📖 脱敏中间件的能力边界见上一篇第九节第 16 条。

---

## 陷阱

1. **⚠️ 以为 TS 类型注解在运行时存在**——TS 类型编译后被完全擦除，和 Python 注解一样是编译期的；**运行时校验只能靠 Zod**；
2. **⚠️ 以为 TS 有 Python 的 `Protocol` 运行时检查**——TS 的 `interface` 纯编译期；想要运行时检查必须显式写 `z.custom()` / `safeParse`；
3. **⚠️ 忘了顶层 await**——顶层 await 要求 ESM（`"type": "module"`），CommonJS 下是语法错误；`initChatModel` / `agent.invoke` 都是异步的；
4. **⚠️ `any` 逃逸**——工具参数、工具返回值、`catch`、mock 硬转这四个地方最容易漏；`catch` 应该用 `unknown`；
5. **⚠️ ESM 下相对导入缺扩展名**——必须写 `./tools.js`；CommonJS 下能跑、ESM 下报 `ERR_MODULE_NOT_FOUND`；
6. **⚠️ 以为「有 Zod 就不用管运行时」**——**框架会替模型调用的那条路径校验**，但**你的代码直接调工具时没人校验**；那条路径要自己 `safeParse`；
7. **⚠️ 工具挂住导致 Agent 卡死**——Node 的 `fetch` 默认无超时；必须加 `AbortSignal.timeout(ms)`，再配 `toolCallLimitMiddleware` 兜底；
8. **⚠️ 工具一抛错就以为整个 Agent 挂了**——默认行为是错误变成 `status: "error"` 的 `ToolMessage` 回喂模型；「查不到」这类业务错误应该**返回错误对象**而不是抛错；
9. **⚠️ 在测试里调真模型**——用 `fakeModel()`（`langchain` 主包导出），测试免费、离线、毫秒级；
10. **⚠️ 断言模型的「回答内容」**——模型的回答是**非确定性的**；要断言的是**流程**（调了哪个工具、传了什么参数、返回结构对不对）；
11. **⚠️ 以为 `FakeListChatModel` 的 `responses` 能塞 `AIMessage`**——它的类型是 **`string[]`**；要返回带工具调用的消息，用 `fakeModel().respondWithTools(...)` 或 `FakeToolCallingModel`；
12. **⚠️ `safeParse` 后不收窄就取 `data`**——`safeParse` 返回判别联合，必须先判 `success`；
13. **⚠️ 开了 LangSmith 追踪就等于把用户数据传第三方**——含 PII 的场景要先脱敏并确认合规；
14. **⚠️ 非美区忘了设 `LANGSMITH_ENDPOINT`**——表现是「Key 无效」，但其实是区域 endpoint 不对；
15. **⚠️ serverless 环境用后台回调**——`LANGCHAIN_CALLBACKS_BACKGROUND` 在 serverless 下要设为 `false`，否则追踪数据随函数结束而丢；
16. **⚠️ 把 Python 的包名/类名直接搬过来**——`InMemorySaver`（Python）→ `MemorySaver`（TS）；`langchain-mcp-adapters`（Python）→ `@langchain/mcp-adapters`（TS）；`prompt=` → `systemPrompt`；`jump_to` → `jumpTo`；
17. **⚠️ 以为改了 schema 类型就等于改了 Agent 行为**——schema 的 `.describe()` 文本会发给模型，**改描述就是在改模型的行为**；改完要重跑提示词相关的测试；
18. **⚠️ 锁了 `langchain@1.5.x` 却把 `@langchain/core` 升到 1.3.x**——`langchain@1.5.x` 的 peer 依赖是 `core@^1.2.20`，升到 1.3 会 **peer 冲突**；模型集成包也要跟着挑同一条 core 版本线。

## 小结

- ⭐ **项目结构**：schema 集中一处 → 错误类型 → 工具 → Agent（模型用依赖注入）→ 入口 → 测试
- ⭐ **类型贯通的完整链路**：Zod schema → `z.infer` → 工具参数类型 → 工具返回值类型 → `responseFormat` → `result.structuredResponse` 的类型
- ⭐ **框架会替你校验工具入参**（`tool()` 内部跑 Zod，失败抛 `ToolInputParsingException`）；但**直接调工具的路径**要自己 `safeParse`
- ⭐ **判定类型贯通是否断裂**：函数参数上出现 `any` 或手写 `interface` 就是断了
- ⭐ **Python → TS 概念映射**：模型 / 消息 / 工具 / Agent / 检索 / 流式 / 结构化输出 / checkpointer / 上下文 / 中间件钩子 / MCP，各有对应但写法不同
- ⭐ **错误三等级**：可自愈 → 返回错误对象；可重试 → 抛错 + `toolRetryMiddleware`（`onFailure: "continue"`）；不可恢复 → 抛错 + `onFailure: "error"`；框架默认把工具错误变成 `status: "error"` 的 `ToolMessage`
- ⭐ **测试用 `fakeModel()`**（`langchain` 主包导出）+ `langchainMatchers`；断言**流程**（调了哪个工具、参数是什么），不断言模型回答文本
- ⭐ **五个坑**：类型注解运行时不存在 / 忘了顶层 await / `any` 逃逸 / 没有运行时 `Protocol` / ESM 相对导入缺扩展名
- ⭐ **LangSmith 追踪**：`LANGSMITH_TRACING=true` + `LANGSMITH_API_KEY` + `LANGSMITH_PROJECT`，零代码接入；⚠️ 非美区要设 `LANGSMITH_ENDPOINT`，serverless 要把 `LANGCHAIN_CALLBACKS_BACKGROUND` 设为 `false`

## 练习

1. 把第一篇的天气 Agent 完整实现出来（5 个源文件 + 测试），确保 `npx vitest run` 全绿且**不联网**。
2. 在 `schemas.ts` 里把 `WeatherQuerySchema.city` 从 `z.string()` 改成 `z.string().min(2)`，观察：TS 编译期报不报错？让模型传 `"X"` 时运行时校验会不会拦住？说明「TS 类型 vs Zod 运行时校验」各自负责什么。
3. 用 `fakeModel().respondWithTools([...])` 写一个测试，断言 Agent 在用户问「北京天气」时**确实**用 `{ city: "北京" }` 调用了工具。故意把 `args` 改成 `{ city: "上海" }`，看测试如何失败。
4. 实现「工具查不到城市 → Agent 继续 → 模型自己换个城市重试」的完整链路：需要假模型排队**两次**工具调用（两次 `respondWithTools`，两次不同的 `args`）。写出这个假模型怎么写。
5. 写一个 `wrapToolCall` 中间件，把所有工具调用的**耗时**记录下来，并统计每个工具的调用次数。说明为什么这个应该用 `wrapToolCall` 而不是 `afterModel`。
6. 故意在你的项目里写一处 `catch (e: any)`，然后改成 `catch (e: unknown)` + 类型守卫，观察 TS 给出的提示差异。
7. 把项目从 CommonJS 切到 ESM（或反过来），观察相对导入扩展名问题在哪种配置下会报错；再试试 `"moduleResolution": "bundler"`，看结论是否变化。写出你的判断标准。
8. 接入 LangSmith，跑一次 Agent，在 UI 里找到「工具调用」节点，截图标注出：模型发出的工具名、参数、工具的返回值。说明这份 trace 能回答哪些「模型为什么这么答」的问题。

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：[[typescript/03-生态连接-LangChain与LangGraph/2-LangChain-js核心API实战]] · 下一篇：下一专题：[[typescript/03-生态连接-LangChain与LangGraph/4-LangGraph-js状态图实战]]
