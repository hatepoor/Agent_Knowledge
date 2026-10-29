---
tags: [typescript, langchain, agent, 中间件]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# LangChain.js 核心 API 实战

> **什么时候该读**：你已经决定用 LangChain.js v1，需要动手写出第一个能跑的 Agent，并搞清 `tool()` / `createAgent` / `createMiddleware` 这三个入口到底怎么用。
> **阅读门槛**：读完 [[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]]（知道 v1 的整体格局和版本基线），以及 [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]]（会写 `async/await`）。Zod 的 `z.object` / `z.infer` 在本篇第一次出现时都会解释。

## 学习目标

读完本篇，你应当能够：

1. 用 `tool()` 定义一个带 **Zod schema** 的工具，并说清 `description` 为什么决定模型会不会调它
2. 解释 **Zod schema 如何同时充当运行时校验器和静态类型来源**，说明工具参数为什么不用手写 `interface`
3. 用 `createAgent` 写出一个**完整可运行**的 Agent，并从结果里读出内容
4. 在**字符串 ID / `initChatModel` / 模型实例**三种模型接法之间做出选择并说明理由
5. 用 `responseFormat` 实现**结构化输出**，从 `result.structuredResponse` 取到类型安全的结果
6. 用 `agent.stream()` 做流式输出，并说清 `streamMode` 各取值的差别
7. 说清**中间件的 6 个钩子**各自在什么时候触发、适合干什么，并写出一个自定义中间件
8. 用 `contextSchema` + `runtime.context` 把**应用级上下文**（用户等级、租户 ID）传进 Agent
9. 避开本篇列出的**常见坑清单**里的每一条

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]] | v1 的包结构、版本基线、`@langchain/classic` 的来历 |
| [[typescript/02-中段阶段-专题精讲/Node生态与异步/2-Promise与异步流]] | Agent 的每次调用都是异步的；`await` / `for await` 是本篇的基本功 |
| [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]] | `z.infer<typeof Schema>` 用到了条件类型 + `infer` |
| [[langchain/middleware/钩子机制详解]] | Python 版的钩子机制讲解——本篇讲的是 TS 版的对应物，**概念相通、写法不同** |

---

## 一、⭐ `tool()`：定义工具

### 1.1 工具是什么、为什么模型需要「描述」

工具就是**一个有名字、有描述、有参数 schema 的普通函数**。模型看到的是它的「说明书」，不是它的代码。

流程是这样的：

```
用户："北京今天天气怎么样？"
   ↓
模型看到工具清单：
   · get_weather
     description: "查询指定城市的当前天气。当用户问天气、温度、湿度、下不下雨时使用。"
     参数: { city: string }
   ↓
模型判断："这个需求匹配 get_weather，city = 北京"
   ↓
框架调用你的函数 → 把结果回填给模型 → 模型组织语言回答
```

⭐ **关键机制**：模型**不会**去看你的函数体。它只看 `name`、`description` 和 `schema`。**这三个东西就是你和模型之间唯一的契约。**

### 1.2 最小可运行示例

```ts
import { tool } from "langchain";
import * as z from "zod";

const getWeather = tool(
  ({ city }) => `It's always sunny in ${city}!`,   // ← 第一个参数：函数本体
  {
    name: "get_weather",                          // ← 工具名
    description: "Get the weather for a given city.",
    schema: z.object({ city: z.string() }),       // ← Zod schema
  },
);
```

⚠️ **注意参数顺序**：`tool(fn, config)`——**函数在前、配置在后**。这和 Python 的 `@tool` 装饰器（配置在前）是相反的，迁移时容易写错。

⭐ **框架会替你校验工具入参**：`tool()` 的内部实现会先对「模型传来的参数」跑一次 **Zod 校验**，通过后**把解析后的值**交给你的函数。所以：

- 类型不对的参数（模型传了 `city: 42`）会被拦下，抛 `ToolInputParsingException`，**你的函数根本不会被调用**；
- `.default(...)` 声明的默认值**已经被填好**（模型不传就是默认值）；
- 校验失败时错误会被包成 `status: "error"` 的 `ToolMessage` 回喂给模型，模型能据此改参数重试。

⭐ 这就是「Zod schema 兼作运行时校验器」的真正含义——**不是「你记得调 `parse`」，而是「框架替你调」**。

**Zod 基础速查**（本篇用到的全部）：

| 写法 | 含义 |
|------|------|
| `z.object({ ... })` | 一个对象结构 |
| `z.string()` | 字符串 |
| `z.number()` | 数字 |
| `z.boolean()` | 布尔 |
| `z.enum(["a", "b"])` | 只能是列举值之一（字面量联合） |
| `.optional()` | 该字段可省略 |
| `.default(x)` | 该字段可省略，缺省时取 `x` |
| `.describe("...")` | 给字段写说明——**这个说明会一起发给模型** |

### 1.3 ⭐ `description` 必须写清楚

⭐ **这是本篇最重要的实践建议**。`description` 是模型选择工具的**唯一依据**——它决定了「这个工具什么时候该被调用」。

| 写法 | 效果 |
|------|------|
| ❌ `description: "天气工具"` | 模型不知道啥时候用；也可能该用的时候不用 |
| ⚠️ `description: "Get weather"` | 太模糊：查历史天气？查预报？查空气质量？ |
| ✅ `description: "查询指定城市的**当前**天气（温度、体感温度、湿度、天气状况）。当用户问『天气怎么样』『多少度』『会不会下雨』时使用。不用于历史天气查询——那要用另一个工具。"` | 模型能准确判断边界 |

⭐ **写 `description` 的三个要点**：

1. **说清「做什么」**——动词开头，写具体动作，不要写名词短语；
2. **说清「什么时候用」**——给出触发场景，最好带上用户可能的问法；
3. **说清「什么时候不用」**——这一条最容易被忽略，但对工具多的 Agent 极其重要（模型会挑错工具）。

### 1.4 工具函数可以做什么

工具函数**可以**是同步的，也可以是 `async` 的（绝大多数真实场景是异步的：要调 HTTP、要查库）。

```ts
const searchOrders = tool(
  async ({ userId, keyword }) => {
    // 第二个参数是运行时配置，能拿到上下文、写入流式进度
    const res = await fetch(`https://api.example.com/orders?u=${userId}&q=${keyword}`);
    if (!res.ok) throw new Error(`订单查询失败：${res.status}`);
    return JSON.stringify(await res.json());
  },
  {
    name: "search_orders",
    description: "按关键词搜索当前用户的订单记录。用户问『我的订单』『买过什么』时使用。",
    schema: z.object({
      userId: z.string().describe("当前用户 ID，由系统提供"),
      keyword: z.string().describe("订单关键词，如商品名或订单号"),
    }),
  },
);
```

⭐ **工具函数的第二个参数（运行时配置 `ToolRuntime`）** 能拿到这些：

| 属性 | 用途 |
|------|------|
| `runtime.context` | 本次运行的上下文（就是 `runtime.context`，见第七节） |
| `runtime.writer` | 往流里写自定义进度（配合 `streamMode: "custom"`） |
| `runtime.toolCallId` | 本次工具调用的 ID（写 `ToolMessage` 时要用） |
| `runtime.state` | 当前 Agent 状态 |
| `runtime.store` | 长期记忆存储（跨会话） |
| `runtime.config` | LangGraph 的 `RunnableConfig` |

类型可以显式标注成 `ToolRuntime`（从 `langchain` 导入）：

```ts
import { tool, type ToolRuntime } from "langchain";

const getUserName = tool(
  (_input, runtime: ToolRuntime) => runtime.context.userName,   // ⭐ 直接读上下文
  {
    name: "get_user_name",
    description: "获取当前用户的名字。用户问『我是谁』时使用。",
    schema: z.object({}),
  },
);
```

---

## 二、⭐ Zod schema 与 TS 类型的双向关系

### 2.1 一份声明，两个用途

```ts
import * as z from "zod";

const WeatherArgs = z.object({
  city: z.string().describe("城市名，如「北京」"),
  units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
});

// ⭐ 从 schema 反推类型——一个泛型参数搞定
type WeatherArgs = z.infer<typeof WeatherArgs>;
```

`z.infer<typeof X>` 展开后大致等价于：

```ts
type WeatherArgs = {
  city: string;
  units: "celsius" | "fahrenheit";   // .default() 让它变成可选输入 + 有默认输出
};
```

⭐ 但**你不需要手写这段**。它的价值在于：**类型永远和 schema 一致**。改了 schema，类型自动跟着变，不可能对不上。

### 2.2 为什么工具参数不用手写 `interface`

对比一下两种写法：

```ts
// ❌ 手写 interface：两份声明，可能对不上
interface WeatherArgs {
  city: string;
  units: "celsius" | "fahrenheit";
}
const weatherSchema = z.object({
  city: z.string(),
  units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
});
// ⚠️ 隐患：改了 schema 忘了改 interface → 模型按新 schema 发参数，你的函数按旧类型处理 → 静默的 bug
```

```ts
// ✅ 只写 schema，类型由它推导
const weatherSchema = z.object({
  city: z.string(),
  units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
});
type WeatherArgs = z.infer<typeof weatherSchema>;   // ⭐ 永远一致
```

⭐ **三个层面的好处**：

| 层面 | 说明 |
|------|------|
| **静态类型** | TS 知道 `city` 是 `string`，不用 `as string` 强转 |
| **运行时校验** | `schema.parse(input)` 会真的检查；框架会用它校验模型传来的参数 |
| **发给模型的契约** | schema 会被**转成 JSON Schema** 发给模型，模型据此生成参数 |

⭐ **一句话**：**Zod schema 是「唯一真相源」，同时喂饱了编译器、运行时和模型三方。** 这在 Python 里做不到（见上一篇第五节）。

### 2.3 schema 直接给 `tool()` 用

`tool()` 的 `schema` 参数接受的就是这个 Zod 对象，**框架负责把它转成 JSON Schema 发给模型**：

```ts
const getWeather = tool(
  ({ city, units }: z.infer<typeof weatherSchema>) => {
    //                       ^^^^^^^^^^^^^^^^^^^^^^^^ 参数类型也自动有了
    return `${city}: ${units}`;
  },
  {
    name: "get_weather",
    description: "查询指定城市的当前天气。",
    schema: weatherSchema,
  },
);
```

> 📖 `z.infer` 的底层原理（条件类型 + `infer` 关键字）见 [[typescript/01-入门阶段-基础教程/第02章-类型系统/2.3-泛型与类型约束]]。

---

## 三、⭐ `createAgent`：完整可运行示例

### 3.1 一个能跑的 Agent

```ts
// agent.ts
import { createAgent, tool } from "langchain";
import * as z from "zod";

const getWeather = tool(
  ({ city }) => `It's always sunny in ${city}!`,
  {
    name: "get_weather",
    description: "Get the weather for a given city.",
    schema: z.object({ city: z.string().describe("The city to get the weather for") }),
  },
);

const agent = createAgent({
  model: "openai:gpt-4o",                       // 模型：字符串 ID
  tools: [getWeather],                          // 工具清单
  systemPrompt: "You are a helpful weather assistant.",
});

const result = await agent.invoke({
  messages: [{ role: "user", content: "What's the weather in San Francisco?" }],
});

console.log(result.messages.at(-1)?.text);
```

### 3.2 `createAgent` 的参数

| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `model` | ✅ **是** | `string \| BaseChatModel` | 模型。字符串 ID 走 `initChatModel` 自动选提供商 |
| `tools` | 否 | `Array<Tool>` | 工具清单。空 Agent 也可以（纯对话） |
| `systemPrompt` | 否 | `string \| SystemMessage` | 系统提示词。用 `SystemMessage` 可以做多内容块、缓存控制等高级操作 |
| `responseFormat` | 否 | Zod schema \| 策略对象 | 结构化输出，见第五节 |
| `middleware` | 否 | `Array<Middleware>` | 中间件清单，见第八节 |
| `checkpointer` | 否 | `BaseCheckpointSaver \| boolean` | 持久化后端。传了才能用 `thread_id` 多轮对话 |
| `contextSchema` | 否 | Zod schema | 运行时上下文的结构定义，见第七节 |
| `stateSchema` | 否 | `StateSchema`（来自 `@langchain/langgraph`） | 扩展 Agent 状态（**更常见的做法是放在中间件的 `stateSchema` 里**） |
| `store` | 否 | `BaseStore` | 长期记忆存储（跨会话） |
| `name` | 否 | `string` | Agent 名字，便于在 trace 里区分 |
| `description` | 否 | `string` | Agent 描述 |
| `signal` | 否 | `AbortSignal` | 取消信号 |

⚠️ **`systemPrompt` 不是 `prompt`**：v0.x 的 `prompt` 参数在 v1 改名为 `systemPrompt`。写 `prompt` 会被静默忽略或报错。

### 3.3 调用与结果

`invoke` 的输入是 `{ messages: [...] }`，输出是一个**状态对象**：

```ts
const result = await agent.invoke({
  messages: [{ role: "user", content: "你好" }],
});

result.messages              // 完整的消息列表（用户消息 + AI 消息 + 工具消息…）
result.structuredResponse    // 有 responseFormat 时才有
```

**取最终回复**的惯用写法是 `result.messages.at(-1)?.text`：

- `.at(-1)` 取最后一个元素（比 `[length - 1]` 更安全，数组为空时返回 `undefined` 而不是抛错）；
- `?.` 是可选链——结果为空时不崩；
- ⭐ **用 `.text` 而不是 `.content`**：v1 的**标准内容块（content blocks）**让 `content` 可能是 `string`，也可能是内容块数组（取决于 `outputVersion` 和模型）。**`.text` 永远返回 `string`**，是安全的选择。

```ts
const last = result.messages.at(-1);
console.log(last?.text);          // ⭐ 一定是 string
console.log(last?.contentBlocks); // ⭐ 标准内容块数组（多模态统一访问）
```

⚠️ **`.at()` 需要 ES2022 lib**。本库基线 TS 6/7 的 `target: es2025` 天然支持。

### 3.4 消息格式

`messages` 数组里的元素可以是**普通对象**，也可以是 LangChain 的消息类：

```ts
// 简写：普通对象（role + content）
{ messages: [{ role: "user", content: "你好" }] }

// 完整：消息类（需要更多控制时用）
import { HumanMessage, SystemMessage } from "langchain";
{ messages: [new HumanMessage("你好")] }
```

> 📖 消息类型体系（`HumanMessage` / `AIMessage` / `ToolMessage` / `SystemMessage`）见 [[langchain/modules/langchain_messages/消息类型详解]]（以 Python 讲解，类型名与 TS 一致）。

---

## 四、⭐ 模型接入的三种写法

### 4.1 写法一：字符串 ID（最省事）

```ts
const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [],
});
```

⭐ **机制**：字符串会走 `initChatModel`（`initChatModel` 也在 `langchain` 包里导出），格式是 **`{provider}:{model}`**：

| 写法 | 效果 |
|------|------|
| `"openai:gpt-4o"` | 显式指定提供商 |
| `"anthropic:claude-sonnet-4-6"` | Anthropic |
| `"openrouter:anthropic/claude-sonnet-4-6"` | OpenRouter（多供应商聚合） |
| `"google:gemini-3.6-flash"` | Google |

⚠️ **provider 前缀是必需的**（除非环境里只装了一个提供商集成包）。写 `"gpt-4o"` 会因为猜不出提供商而报错。

⚠️ **对应的 npm 包必须装上**：`"openai:gpt-4o"` 需要 `npm install @langchain/openai`。

### 4.2 写法二：`initChatModel`（需要传参数时）

```ts
import { initChatModel } from "langchain";

const model = await initChatModel("anthropic:claude-sonnet-4-6", {
  temperature: 0.7,
  timeout: 30,
  maxTokens: 1000,
  maxRetries: 6,
});

const agent = createAgent({ model, tools: [] });
```

⭐ **注意 `initChatModel` 是异步的**——必须 `await`。它返回的是一个聊天模型实例。

接 OpenAI 兼容网关（自建代理 / vLLM / 各类中转）：

```ts
const model = await initChatModel("my-model", {
  modelProvider: "openai",
  baseUrl: "https://your-gateway.example.com/v1",
  apiKey: process.env.GATEWAY_API_KEY,
});
```

### 4.3 写法三：模型实例（最精细）

```ts
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  model: "gpt-4o",
  apiKey: process.env.OPENAI_API_KEY,
  temperature: 0.2,          // ⭐ 低温 = 输出更确定，适合工具调用
  maxTokens: 2048,
});
```

用实例接自建网关时，base URL 走 OpenAI SDK 的 `ClientOptions`：

```ts
const model = new ChatOpenAI({
  model: "my-model",
  apiKey: process.env.GATEWAY_API_KEY,
  configuration: { baseURL: "https://your-gateway.example.com/v1" },   // ⚠️ 实例用 baseURL（SDK 命名）
});
```

⚠️ **注意大小写不一样**：`initChatModel` 的选项是 **`baseUrl`**（小写 u），而 `ChatOpenAI` 实例的 `configuration.baseURL` 是 **OpenAI SDK 的 `baseURL`**（大写 URL）。写错不会报错，只会「连不上」。

### 4.4 ⭐ 什么时候该用哪一种

| 场景 | 选择 | 理由 |
|------|------|------|
| 快速原型、模型只有一两个 | **字符串 ID** | 最少代码；换模型只改一个字符串 |
| 需要统一设 `temperature` / `timeout` / `maxRetries` | **`initChatModel`** | 参数集中在一处，不用每个 Agent 重复写 |
| 需要**精细控制**：用某厂商特有的参数（比如 Anthropic 的 `thinking`）、跑多个模型做对比、单独设置 OpenAI SDK 的 `configuration` | **实例** | 只有实例化能拿到提供商特有选项 |
| 需要在**运行时动态换模型**（比如按负载路由） | **实例 + 中间件** | 换模型在 v1 里通过 `wrapModelCall` 钩子做，见第八节 |

⭐ **实战建议**：Agent 类应用里 `temperature` 应该压低（`0`～`0.3`）——高温会让模型乱调工具。

> ⚠️ v1 **不支持**在 `model` 位置传函数来动态选模型（v0 的 `createReactAgent` 支持）。v1 里这能力已迁移到中间件的 `wrapModelCall` 钩子。

---

## 五、⭐ 结构化输出

### 5.1 需求：让 Agent 返回「数据」而不是「一段话」

默认情况下 Agent 返回自然语言。但业务代码往往需要**确定的结构**——比如「把这段评论解析成 `{ sentiment, score }`」。

### 5.2 用法：`responseFormat` + `structuredResponse`

```ts
import { createAgent } from "langchain";
import * as z from "zod";

const ContactInfo = z.object({
  name: z.string().describe("姓名"),
  email: z.string().describe("邮箱"),
  phone: z.string().describe("电话"),
});

type ContactInfo = z.infer<typeof ContactInfo>;   // ⭐ 类型自动推导

const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [],
  responseFormat: ContactInfo,          // ← 传 Zod schema
});

const result = await agent.invoke({
  messages: [{ role: "user", content: "Extract: John Doe, john@example.com, (555) 123-4567" }],
});

console.log(result.structuredResponse);
// { name: 'John Doe', email: 'john@example.com', phone: '(555) 123-4567' }
//                  ^^^^^^^^^^^ 类型是 ContactInfo，不是 any
```

⭐ **三个要点**：

1. **结果在 `result.structuredResponse`**，不是 `result.messages` 里；
2. **类型是推导出来的**——`result.structuredResponse` 的 TS 类型就是 `ContactInfo`；
3. **v1 的结构化输出在主循环内生成**——不像旧版那样额外再发一次 LLM 调用，因此**更省 token、延迟更低**。

### 5.3 底层策略：工具策略 vs 提供商原生

`responseFormat` 直接传 schema 时，LangChain 会**自动选策略**：

| 策略 | 条件 | 特点 |
|------|------|------|
| **提供商原生**（`providerStrategy`） | 模型和提供商支持原生 JSON Schema 输出 | 快、准、token 省 |
| **工具策略**（`toolStrategy`） | 其他所有模型 | 把 schema 变成一个「工具」让模型调用 |

⭐ **需要手动指定时**：

```ts
import { toolStrategy, providerStrategy, createAgent } from "langchain";

const agent = createAgent({
  model: "openai:gpt-4o",
  responseFormat: toolStrategy(ContactInfo),     // 强制走工具策略
});
```

### 5.4 ⚠️⚠️ 不要预先 `bindTools()`

⭐ **这是本篇最要命的坑。**

```ts
// ❌❌❌ 错误：预先绑定工具，createAgent 会直接抛错
const model = new ChatOpenAI({ model: "gpt-4o" }).bindTools([getWeather]);
const agent = createAgent({ model, tools: [getWeather], responseFormat: ContactInfo });
// ⭐ 抛 MultipleToolsBoundError：
//   "The provided LLM already has bound tools. Please provide an LLM without bound
//    tools to createAgent. The agent will bind the tools provided in the 'tools' parameter."
```

**为什么坏**：

- `createAgent` 内部**本来就会**把 `tools` 参数里的工具绑定到模型上；
- 你再手动 `bindTools` 一次，框架在**构造 Agent 时**就检测到「模型上已经有绑定的工具」，**直接抛 `MultipleToolsBoundError`**；
- 换句话说：**v1 把「重复绑定」从「静默的行为异常」升级成了「明确的启动期错误」**——这其实是好事，至少你不会在生产里才发现。

⭐ **正确做法：把「模型」和「工具」分开传，不要自己绑。**

```ts
// ✅ 正确
const model = new ChatOpenAI({ model: "gpt-4o" });     // 干净的模型
const agent = createAgent({ model, tools: [getWeather], responseFormat: ContactInfo });
//                                          ^^^^^^^^^ createAgent 内部自动绑定
```

> ⚠️ **v1 明确不支持「预绑定模型」**（迁移指南里写的是 “pre-bound models not supported”）。

---

## 六、⭐ 流式输出

### 6.1 `agent.stream()`

```ts
for await (const chunk of await agent.stream(
  { messages: [{ role: "user", content: "讲个笑话" }] },
  { streamMode: "updates" },
)) {
  console.log(chunk);
}
```

⭐ **注意这里有两层 `await`**：`agent.stream(...)` **本身返回一个 Promise**（要先拿到流对象），拿到之后才用 `for await` 消费。写成 `for await (const chunk of agent.stream(...))` 会报错。

### 6.2 `streamMode` 取值

⭐ `streamMode` 是 LangGraph 层面的概念（Agent 底层就是一张图），取值来自 LangGraph 的 `StreamMode` 联合类型：

| `streamMode` | 产出什么 | 典型用途 |
|-------------|---------|---------|
| `"values"` | 每一步之后的**完整状态**快照 | 做审计、记录每一步的完整 state |
| `"updates"` | 每一步的**增量更新**（哪个节点改了什么） | ⭐ 最常用；看清「模型说了啥、调了啥工具」 |
| `"messages"` | **token 级**的模型输出流 | ⭐ 做打字机效果的前端 |
| `"custom"` | 工具里用 `config.writer` 写入的自定义数据 | 展示工具执行进度 |
| `"debug"` | 详细的执行事件（含中间状态） | 调试用，信息量最大 |
| `"checkpoints"` | 每一步的检查点事件 | 观察持久化状态 |
| `"tasks"` | 每个任务的开始/结束事件 | 观察任务调度 |

⭐ **LangChain.js 的流式文档重点讲前三个**（`updates` / `messages` / `custom`），其余属于 LangGraph 的通用能力。

⭐ **可以传数组同时开多个模式**：

```ts
for await (const [streamMode, chunk] of await agent.stream(input, {
  streamMode: ["updates", "messages", "custom"],
})) {
  console.log(streamMode, chunk);
}
```

### 6.3 token 级流（打字机效果）

```ts
for await (const [token, metadata] of await agent.stream(
  { messages: [{ role: "user", content: "你好" }] },
  { streamMode: "messages" },
)) {
  console.log(`node: ${metadata.langgraph_node}`);
  process.stdout.write(token.text);      // ⭐ token.text 一定是 string
}
```

### 6.4 ⚠️ 节点名从 `"agent"` 改成了 `"model"`

⭐ **v1 的破坏性变更**：流式事件里的**节点名**变了。

| 版本 | 节点名 |
|------|--------|
| v0.x | `"agent"` |
| **v1** | **`"model"`** |

⚠️ **后果**：如果你在代码里按 `metadata.langgraph_node === "agent"` 做过滤判断，**升级后这段逻辑会静默失效**（永远不匹配，表现为「流式输出没有内容」）。

```ts
// ❌ v0.x 写法
if (metadata.langgraph_node === "agent") { ... }

// ✅ v1 写法
if (metadata.langgraph_node === "model") { ... }
```

### 6.5 ⭐ `contentBlocks`：跨提供商统一的多模态访问

⭐ **v1 的标准内容块（standard content blocks）**解决了「不同模型返回的消息结构不一样」这个长期痛点。

**问题**：OpenAI 返回 `{ type: "text", text: "..." }`，Anthropic 返回 `{ type: "text", text: "..." }`，图像的格式又各不相同——你如果直接读 `message.content`，每接一个新提供商就要改一遍。

**解法**：`contentBlocks` 提供**统一的结构**：

```ts
const last = result.messages.at(-1);

for (const block of last?.contentBlocks ?? []) {
  if (block.type === "text") {
    console.log("文本：", block.text);
  } else if (block.type === "image") {
    console.log("图片：", block.source_type);   // ⭐ 统一的 source_type
  }
}
```

⭐ **三个要点**：

| 要点 | 说明 |
|------|------|
| **统一结构** | 所有提供商的多模态内容都归一成同一套 block 类型 |
| **类型安全** | `contentBlocks` 有完整的 TS 类型（`Standard[]`），`block.type` 可以直接收窄 |
| **向后兼容** | 标准内容块支持**延迟加载**，所以从 `content` 切到 `contentBlocks` 不会引入破坏性变更 |

⭐ **什么时候用**：

- 只处理纯文本 → 用 `.text`（最省事）；
- 需要处理图片 / 音频 / 文件 → 用 `.contentBlocks`（跨提供商统一）；
- 需要把内容转发给另一个模型 → 用 `.contentBlocks`（**不会因为提供商不同而丢信息**）。

> 📖 Python 版的内容块概念见 [[langchain/modules/langchain_messages/消息类型详解]]。

### 6.6 `streamEvents`：类型化事件流

```ts
const stream = await agent.streamEvents(
  { messages: [{ role: "user", content: "北京天气" }] },
  { version: "v3" },        // ⭐ v1 推荐 v3
);

await Promise.all([
  (async () => {
    for await (const message of stream.messages) {
      for await (const token of message.text) {
        process.stdout.write(token);
      }
    }
  })(),
  (async () => {
    for await (const call of stream.toolCalls) {
      console.log(`\nTool call: ${call.name}(${JSON.stringify(call.input)})`);
      console.log(`Tool result: ${await call.output}`);
    }
  })(),
]);

const finalState = await stream.output;
```

> 📖 流式模式的完整对比（Python 视角，模式语义一致）见 [[langgraph/modules/langgraph_streaming/流式模式对比]] 与 [[langchain/streaming/流式传输]]。

---

## 七、⭐ `contextSchema` + `runtime.context`：传应用级上下文

### 7.1 为什么需要它

Agent 常常需要知道一些「**本次请求级别的信息**」：

- 当前用户的**等级**（决定用哪个模型、给多详细）；
- **租户 ID**（决定查哪个数据库）；
- **API Key**（换供应商时用）；
- 功能开关（灰度）。

⭐ **这些不属于「对话状态」**——它们不该被存进 checkpoint、也不该被模型当消息看到。LangChain.js v1 用 **runtime context** 承载它们。

### 7.2 三个步骤

```ts
import { createAgent } from "langchain";
import * as z from "zod";

// ① 声明上下文的结构
const contextSchema = z.object({
  userId: z.string(),
  userRole: z.enum(["free", "pro"]).default("free"),
  tenantId: z.string(),
});

const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [],
  contextSchema,          // ← ② 挂到 agent 上
});

// ③ 调用时传 context（不是 configurable）
const result = await agent.invoke(
  { messages: [{ role: "user", content: "介绍一下你的功能" }] },
  {
    context: { userId: "u-123", tenantId: "acme" },
  },
);
```

⭐ **注意 `context` 的类型是「必填字段全填」**：`contextSchema` 里 `userRole` 有 `.default("free")`，所以可以省略；`userId` / `tenantId` 必填。

### 7.3 读上下文：`runtime.context`

⭐ **在中间件里**通过 `request.runtime.context` 读：

```ts
import { createMiddleware } from "langchain";
import * as z from "zod";

const contextSchema = z.object({
  userId: z.string(),
  tenantId: z.string(),
});

const userContextMiddleware = createMiddleware({
  name: "UserContextMiddleware",
  contextSchema,
  wrapModelCall: (request, handler) => {
    const { userId, tenantId } = request.runtime.context;   // ⭐ 读上下文
    const contextText = `User ID: ${userId}, Tenant: ${tenantId}`;
    const newSystemMessage = request.systemMessage.concat(contextText);
    return handler({ ...request, systemMessage: newSystemMessage });
  },
});
```

⭐ **在工具里**通过 `runtime.context` 读（见第一节 1.4）。

### 7.4 ⚠️ `context` vs `configurable`

⭐ **v1 把这两件事彻底分开了**：

| 字段 | 装什么 | 生命周期 |
|------|--------|---------|
| **`context`** | **你的应用数据**（用户 ID、等级、租户） | 只在本次运行有效，**不入 checkpoint** |
| **`configurable`** | **框架的图配置**（`thread_id`、`recursion_limit` 等） | 框架自己用，你一般只关心 `thread_id` |

> ⚠️ **v0.x 的写法是把两者混在一起**：把 `userRole` 塞进 `config.configurable`。v1 迁移指南明确写了这条变更：**「Runtime context: `context` property instead of `config.configurable`」**。

⭐ **判断口诀**：**「这是我的业务数据吗？」** 是 → `context`；否 → `configurable`。

---

## 八、⭐ 中间件的 6 个钩子

### 8.1 两类钩子

中间件的钩子分两类，**这个区分是理解一切的前提**：

| 风格 | 钩子 | 特点 |
|------|------|------|
| **节点式（Node-style）** | `beforeAgent` / `beforeModel` / `afterModel` / `afterAgent` | **顺序执行**，在某个时间点触发一次 |
| **包装式（Wrap-style）** | `wrapModelCall` / `wrapToolCall` | **包裹执行**，你决定 handler 调 0 次 / 1 次 / N 次 |

⭐ **关键区别**：节点式钩子**不能拦截「要不要调模型」**——模型总是被调用。包装式可以：handler 调 **0 次**（短路）、**1 次**（正常）、**N 次**（重试）。

### 8.2 六个钩子一览

| 钩子 | 触发时机 | 典型用途 | 签名 |
|------|---------|---------|------|
| **`beforeAgent`** | Agent 启动前，每次 `invoke` 一次 | 初始化资源、加载用户偏好、记录开始时间、输入校验 | `(state, runtime)` |
| **`beforeModel`** | **每次**模型调用前 | 摘要压缩、改写系统提示、脱敏、按权限筛工具 | `(state, runtime)` |
| **`wrapModelCall`** | **包裹**每次模型调用 | 重试、模型回退、动态选模型、提示缓存、动态换工具集 | `(request, handler)` |
| **`wrapToolCall`** | **包裹**每次工具调用 | 工具重试、人工审批、参数校验、工具日志 | `(request, handler)` |
| **`afterModel`** | **每次**模型响应后 | 校验输出、记录 token、拦截危险工具调用 | `(state, runtime)` |
| **`afterAgent`** | Agent 完成后，每次 `invoke` 一次 | 清理资源、持久化最终结果、记录总耗时 | `(state, runtime)` |

⭐ **四个节点式钩子的签名完全一致**：`（state, runtime) => 结果`。`runtime` 里带 `context`（应用上下文）、`store`、`writer`、`interrupt`、`signal`。

⭐ **每个节点式钩子都可以写成两种形式**：

```ts
// createMiddleware({ ... }) 的两个属性各有一种写法
const middlewareConfig = {
  // 形式一：直接给函数（不能提前跳转）
  beforeModel: (state, runtime) => {
    // 在这里改写 state.messages 或直接返回更新
  },

  // 形式二：给对象（可以用 canJumpTo 提前跳转）
  beforeModel: {
    canJumpTo: ["end"],
    hook: (state, runtime) => {
      // 同上
    },
  },
};
```

### 8.3 节点式钩子：读 state、改 state

```ts
import { createMiddleware, AIMessage } from "langchain";
import * as z from "zod";

const trackingStateSchema = z.object({
  modelCallCount: z.number().default(0),
});

const incrementAfterModel = createMiddleware({
  name: "incrementAfterModel",
  stateSchema: trackingStateSchema,          // ⭐ 扩展 Agent 状态
  afterModel: (state) => {
    return { modelCallCount: state.modelCallCount + 1 };   // ⭐ 返回要合并进 state 的字段
  },
});
```

⭐ **节点式钩子返回「要更新的 state 片段」**（`undefined` 表示不改）。`state` 是**完整的当前状态快照**。

⭐ **提前终止**：需要提前跳出时，返回 `jumpTo`。这要求用**对象形式**声明钩子，并显式声明可跳转目标：

```ts
const messageLimitMiddleware = createMiddleware({
  name: "MessageLimitMiddleware",
  beforeModel: {
    canJumpTo: ["end"],                       // ⭐ 必须显式声明
    hook: (state) => {
      if (state.messages.length >= 50) {
        return {
          messages: [new AIMessage("Conversation limit reached.")],
          jumpTo: "end",                      // ⭐ 提前结束
        };
      }
      return;                                 // 不返回 = 不改状态
    },
  },
});
```

⚠️ **JS 版是 `camelCase`**：`jumpTo`（不是 Python 的 `jump_to`）、`canJumpTo`（不是 `can_jump_to`）。

### 8.4 包装式钩子：改 request、控 handler

```ts
import { createMiddleware } from "langchain";

const retryMiddleware = createMiddleware({
  name: "RetryMiddleware",
  wrapModelCall: (request, handler) => {
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        return handler(request);        // ⭐ 调 handler = 真正调模型
      } catch (e) {
        if (attempt === 2) throw e;    // 最后一次失败才抛出
        console.log(`第 ${attempt + 1} 次重试：${e}`);
      }
    }
    throw new Error("unreachable");
  },
});
```

⭐ **`request` 上有什么**（`ModelRequest` 类型）：

| 属性 | 内容 |
|------|------|
| `request.state` | 当前状态快照 |
| `request.runtime` | 运行时对象（`.context` 是应用上下文） |
| `request.messages` | 消息列表 |
| `request.systemMessage` | 系统消息（`SystemMessage` 对象）。⚠️ `request.systemPrompt`（字符串）已标记 `@deprecated` |
| `request.model` | 当前模型（可以换） |
| `request.tools` | 当前工具集（可以换） |
| `request.toolChoice` | 工具选择策略（`"auto"` / `"none"` / `"required"` / 指定某个工具） |
| `request.responseFormat` | 结构化输出配置（中间件可以覆盖它） |
| `request.modelSettings` | 额外绑定到模型的设置（如 `headers`），每次调用重新绑定 |

⭐ **改 request 的两种方式**：

```ts
// 方式一：展开后改字段
return handler({ ...request, systemMessage: request.systemMessage.concat("额外上下文。") });

// 方式二：直接改副本的字段
const modified = { ...request };
modified.model = otherModel;      // ⭐ 动态换模型
return handler(modified);
```

### 8.5 `wrapToolCall`：工具级拦截

```ts
const toolMonitoringMiddleware = createMiddleware({
  name: "ToolMonitoringMiddleware",
  wrapToolCall: (request, handler) => {
    console.log(`执行工具：${request.toolCall.name}`);
    console.log(`参数：${JSON.stringify(request.toolCall.args)}`);
    const result = handler(request);      // ⭐ 真正执行工具
    console.log("工具执行成功");
    return result;
  },
});
```

⭐ **`request` 上有什么**（`ToolCallRequest` 类型）：

| 属性 | 内容 |
|------|------|
| `request.toolCall` | 这次调用的信息（`.name` 工具名、`.args` 参数、`.id` 调用 ID） |
| `request.tool` | 正在执行的工具实例（动态注册的工具这里可能是 `undefined`） |
| `request.state` | 当前状态快照 |
| `request.runtime` | 运行时对象 |

### 8.6 预置中间件

v1 自带一批常用中间件，**从 `langchain` 直接导入**：

```ts
import {
  createAgent,
  summarizationMiddleware,
  humanInTheLoopMiddleware,
  piiMiddleware,
} from "langchain";

const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [readEmail, sendEmail],
  middleware: [
    piiMiddleware("email", { strategy: "redact", applyToInput: true }),
    summarizationMiddleware({
      model: "openai:gpt-4o-mini",
      trigger: { tokens: 500 },        // 超过 500 token 就触发摘要
      keep: { messages: 20 },         // 摘要后保留最近 20 条
    }),
    humanInTheLoopMiddleware({
      interruptOn: {
        sendEmail: { allowedDecisions: ["approve", "edit", "reject"] },
      },
    }),
  ],
});
```

⭐ **预置中间件速查**：

| 名称 | 作用 | 关键选项 |
|------|------|---------|
| `summarizationMiddleware` | 对话太长时压缩历史 | `model`（必填）、`trigger`、`keep` |
| `humanInTheLoopMiddleware` | 敏感工具调用前要人工批 | `interruptOn`（工具名 → 允许的决策）、`interruptMode` |
| `piiMiddleware` | 发给模型前脱敏 PII | 第一个参数是 PII 类型（`"email"` / `"credit_card"` / `"ip"` / 自定义），第二个参数是 `{ strategy, detector, applyToInput, applyToOutput, applyToToolResults }` |
| `modelRetryMiddleware` | 模型调用失败自动重试 | `maxRetries`、`backoffFactor`、`initialDelayMs`、`onFailure` |
| `toolRetryMiddleware` | 工具失败自动重试 | 同上，外加 `tools`（限定哪些工具） |
| `modelFallbackMiddleware` | 主模型挂了换备用模型 | 直接传模型 ID 字符串（可变参数） |
| `modelCallLimitMiddleware` | 限制模型调用次数 | `threadLimit`、`runLimit`、`exitBehavior` |
| `toolCallLimitMiddleware` | 限制工具调用次数 | `toolName`、`threadLimit`、`runLimit`、`exitBehavior` |
| `contextEditingMiddleware` | 上下文超长时清理旧工具结果 | `edits`（如 `new ClearToolUsesEdit({...})`） |
| `dynamicSystemPromptMiddleware` | 按 state / context 动态生成系统提示词 | 接收 `(state, runtime)` 返回提示词字符串 |
| `toolErrorMiddleware` | 统一处理工具错误 | 自定义错误处理函数 |

⚠️ **`piiRedactionMiddleware` 已经废弃**：`langchain@1.5.x` 里它仍然导出，但类型上标了 `@deprecated`。**新代码用 `piiMiddleware`**（签名不同：前者是 `piiRedactionMiddleware({ rules: { ssn: /.../ } })`，后者是 `piiMiddleware("email", { strategy: "redact" })`）。

⭐ **不确定当前版本到底导出了什么时，直接查**：

```ts
// 在 node 里直接查实际导出，最可靠
import * as lc from "langchain";
console.log(Object.keys(lc).filter((k) => k.toLowerCase().includes("pii")));
```

> 📖 Python 版的钩子机制全景（6 个钩子的执行时序、`before_*` 正序 / `after_*` 逆序 / `wrap_*` 嵌套）见 [[langchain/middleware/钩子机制详解]]；Python 版中间件总览见 [[langchain/middleware/中间件概述]]。**顺序规则两边一致**：`before_*` 正序、`after_*` 逆序、`wrap_*` 嵌套（第一个中间件包裹所有其他）。

---

## 陷阱

⭐ 以下是本篇最容易踩的坑清单，按「版本 / API / 用法 / 概念」四类排列。

1. **⚠️ 包版本不匹配导致旧 API 导入失败**——v1 里 `createOpenAIFunctionsAgent`、`AgentExecutor` **不再从 `langchain` 导出**；旧教程的 `import { createOpenAIFunctionsAgent } from "langchain/agents"` 会直接报「找不到导出」。用 `createAgent`；实在要用旧 API 就装 `@langchain/classic` 并改 import 路径；
2. **⚠️ 忘记装 `zod`**——`zod` 是 **peer 依赖**，不装会报 `Cannot find module 'zod'`；`npm install zod` 即可；
3. **⚠️ 预先 `.bindTools()`**——`createAgent` 内部会自动绑定；手动预绑定会让 `createAgent` **在构造时直接抛 `MultipleToolsBoundError`**。传「干净的模型 + 工具清单」就够；
4. **⚠️ `description` 写太笼统**——模型选不出工具，表现为「Agent 明明有工具却直接文字回答」。要把「什么时候用 / 什么时候不用」写清楚；
5. **⚠️ 用 `prompt` 而不是 `systemPrompt`**——v1 已改名；v0.x 教程里的 `prompt:` 参数在 v1 里不存在；
6. **⚠️ 把应用上下文塞进 `configurable`**——v1 里业务数据走 `context` + `contextSchema`；`configurable` 留给 `thread_id` 这类图配置；
7. **⚠️ 流式里按 `"agent"` 过滤节点**——v1 的节点名改成了 `"model"`，旧过滤条件会静默失效；
8. **⚠️ `for await (const chunk of agent.stream(...))` 少写 `await`**——`stream()` 返回的是 Promise，要先 `await` 拿到流对象；
9. **⚠️ 忘了 `await agent.invoke(...)`**——拿到的是 Promise 对象不是结果；配合 `void` 或 `.catch()` 避免变成未处理的 rejection；
10. **⚠️ 忘了 `await initChatModel(...)`**——它同样是异步的；
11. **⚠️ 模型 ID 没写 provider 前缀**——`"gpt-4o"` 猜不出提供商；写 `"openai:gpt-4o"`；
12. **⚠️ 用了 `^` 版本范围**——LangChain.js minor 升级就可能改行为；用 `~` 并提交 lock 文件；而且 `langchain` / `@langchain/core` / 模型集成包**必须锁在同一条版本线上**（`langchain@1.5.x` 要求 `core@^1.2.20`）；
13. **⚠️ 以为 `beforeModel` 等钩子能拦截模型调用**——节点式钩子拦不住；想短路/重试必须用 `wrapModelCall`；
14. **⚠️ 在 `beforeModel` 里返回 `jumpTo` 却没声明 `canJumpTo`**——会抛错；且 JS 版是 `canJumpTo` / `jumpTo`（camelCase），不是 Python 的 `can_jump_to` / `jump_to`；
15. **⚠️ 以为 `humanInTheLoopMiddleware` 不用 `checkpointer`**——人工介入靠「中断 + 从检查点恢复」，**必须配 `checkpointer`**，否则中断后无法恢复；
16. **⚠️ 以为 `piiMiddleware` 能保护全链路**——它只保证**发给模型提供商**的那一段是脱敏的；日志、checkpoint、工具入参、最终输出里**仍然有原文**；
17. **⚠️ 工具抛错就以为整个 Agent 挂了**——工具抛错默认会被框架包成 `status: "error"` 的 `ToolMessage` 回给模型，让模型自己换策略重试；要「整个 Agent 失败」得自己配 `onFailure: "error"` 之类的策略；
18. **⚠️ 用 `message.content` 当字符串用**——v1 的标准内容块让 `content` 可能是字符串、也可能是内容块数组；**用 `.text`**（永远是 string）和 `.contentBlocks`（统一多模态访问）；
19. **⚠️ base URL 大小写写错**——`initChatModel` 用 **`baseUrl`**，而 `ChatOpenAI` 实例的 `configuration.baseURL` 是 OpenAI SDK 的 **`baseURL`**（大写 URL）。写错不报错，只会「连不上」；
20. **⚠️ 用已废弃的 `piiRedactionMiddleware`**——`langchain@1.5.x` 里它仍导出但已标 `@deprecated`；新代码用 `piiMiddleware("email", { strategy: "redact" })`。

---

## 小结

- ⭐ **`tool(fn, { name, description, schema })`**——函数在前配置在后；`description` 是模型选工具的**唯一依据**，要写清「什么时候用 / 什么时候不用」
- ⭐ **Zod schema 一份声明两用**：运行时校验（**框架替你调 `parse`**，校验失败抛 `ToolInputParsingException`）+ `z.infer` 静态类型；工具参数**不用手写 interface**
- ⭐ **`createAgent({ model, tools, systemPrompt, responseFormat, middleware, checkpointer, contextSchema, stateSchema, name })`**——`model` 必填；取结果用 `result.messages.at(-1)?.text`
- ⭐ **模型三写法**：字符串 ID（`"openai:gpt-4o"`，最省事）/ `initChatModel`（要统一传参，**记得 await**）/ 实例（要厂商特有参数、SDK `configuration` 时才用）
- ⭐ **结构化输出**：`responseFormat` 传 Zod schema → 结果在 `result.structuredResponse`；v1 在**主循环内**生成，不再多一次 LLM 调用
- ⭐ **⚠️ 绝不预先 `.bindTools()`**——v1 内部自动绑定；预绑定会让 `createAgent` 在构造时抛 `MultipleToolsBoundError`
- ⭐ **流式**：`for await (const chunk of await agent.stream(input, { streamMode }))`；`streamMode` 有 `values` / `updates` / `messages` / `custom` / `debug` / `checkpoints` / `tasks`，可传数组同时开多个；⚠️ **节点名 `"agent"` → `"model"`**
- ⭐ **`contentBlocks`**——v1 的标准内容块，提供跨提供商统一的多模态访问；纯文本用 `.text`
- ⭐ **6 个钩子**：节点式 `beforeAgent` / `beforeModel` / `afterModel` / `afterAgent`（顺序执行，拦不住模型调用）；包装式 `wrapModelCall` / `wrapToolCall`（能短路、能重试）；顺序规则 `before_*` 正序、`after_*` 逆序、`wrap_*` 嵌套
- ⭐ **`contextSchema` + `runtime.context`** 传应用级上下文；⚠️ **业务数据走 `context`，图配置走 `configurable`**
- ⭐ **JS 版是 camelCase**：`jumpTo` / `canJumpTo`（Python 是 `jump_to` / `can_jump_to`）

## 练习

1. 写一个 `searchKnowledgeBase` 工具：schema 有 `query: string` 和 `topK: number`（带 `.default(5)` 和 `.describe`），并写出一段**能让模型准确选用它**的 `description`（要求同时写清「什么时候不用」）。
2. 用 `z.infer` 推出上面工具的参数类型，并写一个函数接收该类型。故意在 schema 里把 `topK` 的类型改成 `z.string()`，观察 TS 报什么错——说明这个错误**在编译期**还是**运行期**能被抓住。
3. 写一个完整 Agent：2 个工具 + `responseFormat`（Zod schema，字段含 `summary: string` 和 `confidence: number`），并打印 `result.structuredResponse`。然后**故意**把模型改成 `.bindTools()` 过的实例，观察 `createAgent` 抛什么错（提示：看错误消息里有没有 `MultipleToolsBoundError`）。
4. 写一段 `agent.stream(..., { streamMode: "messages" })` 的代码，把 token 逐字写到 stdout。再改成 `streamMode: "updates"`，说明两者的输出**结构上**有什么不同。
5. 实现一个 `wrapModelCall` 中间件：调用失败后**最多重试 3 次**，每次间隔 1 秒，最后一次仍失败则抛出。说明为什么这个必须用 `wrapModelCall` 而不是 `afterModel`。
6. 用 `beforeModel` 钩子实现「消息数超过 20 条就提前结束」，写清 `canJumpTo` 和 `jumpTo` 的写法。再解释：为什么它**不能**用来「阻止模型调用」？
7. 用 `contextSchema` 传 `{ userRole: "free" | "pro" }`，写一个 `wrapModelCall` 中间件：`pro` 用户用 `gpt-4o`、`free` 用户用 `gpt-4o-mini`。
8. 列出你项目里所有会用到模型的 API 调用点，检查：有没有忘记 `await`？有没有地方需要 `context` 却写成了 `configurable`？把检查结果写成一页 checklist。

## 导航

[[typescript/03-生态连接-LangChain与LangGraph/00-专题导航|返回专题导航]] · 上一篇：[[typescript/03-生态连接-LangChain与LangGraph/1-TypeScript生态定位与选型]] · 下一篇：[[typescript/03-生态连接-LangChain与LangGraph/3-从TypeScript到LangChain-js实战迁移]]
