---
tags: [langchain, overview, structured, reference]
created: 2026-07-07
updated: 2026-07-07
type: overview
---

# LangChain 使用方法知识库（章节结构总览）

> 本文档作为 LangChain 知识库的**总目录与查询入口**，按"安装 → 核心概念 → 进阶能力 → 高级编排 → 实战"组织。所有子页面内容均已与官方文档对齐，保留完整代码与参数说明。

---

## 章节地图

| 章节 | 核心主题 | 关键 API | 文档 |
|------|---------|---------|------|
| 第 1 章 | 安装与环境 | `pip install langchain`、`langchain-openai` | [[langchain/install/安装LangChain]] |
| 第 2 章 | 框架总览 | `create_agent` 设计哲学 | [[langchain/overview/LangChain概述]] |
| 第 3 章 | 快速入门 | 第一个 Agent 完整流程 | [[langchain/quickstart/快速入门]] |
| 第 4 章 | 模型调用 | `init_chat_model`、参数、工具调用 | [[langchain/models/模型]] |
| 第 5 章 | 工具集成 | `@tool` 装饰器、`ToolRuntime` | [[langchain/tools/工具]] |
| 第 6 章 | 记忆管理 | Checkpointer、Store、裁剪/摘要 | [[langchain/memory/短期记忆]] |
| 第 7 章 | 运行时上下文 | `Runtime`、`context_schema`、依赖注入 | [[langchain/runtime/运行时上下文]] |
| 第 8 章 | 代理（Agent） | `create_agent` 核心组件、调用、流式 | [[langchain/agents/Agent代理]] |
| 第 9 章 | 流式传输 | `updates`/`messages`/`custom` 三种模式 | [[langchain/streaming/流式传输]] |
| 第 10 章 | 结构化输出 | `ProviderStrategy`、`ToolStrategy` | [[langchain/structured-output/结构化输出]] |
| 第 11 章 | 中间件 | 钩子系统、自定义与内置中间件 | [[langchain/middleware/中间件概述]] |
| 第 12 章 | 多 Agent 系统 | 子 Agent、交接、技能、路由器 | [[langchain/multi-agent/多Agent系统]] |

---

## 第 1 章：安装与环境

**功能说明**：安装 LangChain 核心包及各提供商独立集成包。

**关键参数**：
- `langchain`：核心库（Python 3.10+）
- `langchain-openai`、`langchain-anthropic`、`langchain-google-genai` 等：提供商独立包
- `langgraph-checkpoint-postgres`：生产级 checkpointer

**使用场景**：
- 最小安装：`pip install -U langchain`
- 多提供商：`pip install -U langchain-openai langchain-anthropic`

详见 [[langchain/install/安装LangChain]]。

---

## 第 2 章：框架总览

**功能说明**：理解 LangChain 的设计定位（Agent = Model + Harness）。

**关键概念**：
- **Deep Agents**：开箱即用，含上下文压缩、虚拟文件系统、子 Agent
- **LangChain**（`create_agent`）：高度可定制框架
- **LangGraph**：低级编排框架
- **LangSmith**：追踪、调试、评估

详见 [[langchain/overview/LangChain概述]]。

---

## 第 3 章：快速入门

**功能说明**：从零构建一个真实可用的文学研究 Agent。

**核心流程**：
1. 定义系统提示词
2. 创建工具（`@tool` 装饰器）
3. 配置模型（`init_chat_model`）
4. 添加 Checkpointer（`InMemorySaver`）
5. 调用 Agent

**关键参数**：`model`、`tools`、`system_prompt`、`checkpointer`

详见 [[langchain/quickstart/快速入门]]。

---

## 第 4 章：模型调用

**功能说明**：通过统一接口访问数百个 LLM 提供商。

### 关键参数

| 参数 | 类型 | 描述 |
|------|------|------|
| `model` | string | `provider:model` 格式 |
| `temperature` | number | 0~1 控制随机性 |
| `max_tokens` | number | 最大输出 token |
| `timeout` | number | 超时（秒） |
| `max_retries` | number | 最大重试次数 |

### 主要方法

- `invoke(messages)`：单次调用
- `stream(messages)`：流式输出 token
- `batch(messages_list)`：批量调用
- `bind_tools([...][[)`：绑定工具
- `with_structured_output(Schema)`：结构化输出

详见 [[langchain/models/模型]]。

### 使用场景
- 单轮问答：`invoke`
- 实时聊天 UI：`stream`
- 数据处理批：`batch`
- 工具增强对话：`bind_tools`

---

## 第 5 章：工具集成

**功能说明**：让 Agent 与外部世界交互（数据库、API、文件系统等）。

### 关键参数

- `@tool` 装饰器：默认用 docstring 作为描述
- `name` 参数：自定义工具名
- `description` 参数：自定义工具描述
- `args_schema` 参数：使用 Pydantic 模型定义复杂入参
- `return_direct=True`：工具直接返回结果，跳过模型再处理
- `runtime: ToolRuntime`：访问 State、Context、Store、Stream Writer

### 核心使用场景

1. **基础工具定义**：`@tool` 装饰器
2. **运行时信息访问**：`runtime.state`、`runtime.context`、`runtime.store`
3. **状态更新**：返回 `Command(update={...})`
4. **动态工具选择**：通过 middleware 过滤 `request.tools`
5. **运行时工具注册**：在 middleware 中动态添加
6. **多模态返回**：返回标准内容块列表

详见 [[langchain/tools/工具]]。

---

## 第 6 章：记忆管理

**功能说明**：管理对话历史与跨会话知识。

### 关键参数

| 组件 | 描述 | 关键 API |
|------|------|----------|
| **Checkpointer** | 短期记忆（对话历史） | `InMemorySaver`、`PostgresSaver` |
| **Store** | 长期记忆（持久知识） | `InMemoryStore`、`PostgresStore` |
| **SummarizationMiddleware** | 历史摘要 | `trigger`、`keep` |
| **RemoveMessage** | 删除消息 | `REMOVE_ALL_MESSAGES` |

### 使用场景

- **多轮对话**：`checkpointer + thread_id`
- **裁剪过长历史**：`@before_model` 中间件
- **摘要压缩**：`SummarizationMiddleware`
- **用户偏好存储**：`Store` + namespace
- **选择性删除**：`RemoveMessage`

详见 [[langchain/memory/短期记忆]]。

---

## 第 7 章：运行时上下文

**功能说明**：通过依赖注入将配置传递到工具和中间件。

### 关键参数

- `context_schema`：定义 `Context` dataclass
- `runtime.context`：工具/中间件中读取
- `runtime.store`：访问长期记忆
- `runtime.stream_writer`：发送自定义流式事件
- `runtime.execution_info`：thread_id、run_id 等
- `runtime.server_info`：LangGraph Server 元数据

### 使用场景
- 用户级配置：user_id、API key
- 跨调用数据：会话信息、特性开关
- 多租户隔离：每个用户独立的 Store namespace

详见 [[langchain/runtime/运行时上下文]]。

---

## 第 8 章：代理（Agent）

**功能说明**：`create_agent` 是 LangChain v1 的核心工厂函数。

### 关键参数

| 参数 | 描述 |
|------|------|
| `model` | 模型标识符或实例 |
| `tools` | 工具列表 |
| `system_prompt` | 字符串或 `SystemMessage` |
| `state_schema` | 自定义 `AgentState` 子类 |
| `context_schema` | 运行时 Context schema |
| `checkpointer` | 短期记忆后端 |
| `store` | 长期记忆后端 |
| `middleware` | 中间件列表 |
| `response_format` | 结构化输出策略 |
| `name` | Agent 标识（多 Agent 场景） |

### 核心使用场景

1. **基础 Agent**：model + tools + system_prompt
2. **多轮对话**：加 checkpointer
3. **个性化**：加 context_schema
4. **自定义状态**：加 state_schema
5. **结构化输出**：加 response_format
6. **可观察性**：用 LangSmith 追踪

详见 [[langchain/agents/Agent代理]]。

---

## 第 9 章：流式传输

**功能说明**：实时呈现 Agent 运行状态，提升用户体验。

### 三种核心流模式

| 模式 | 数据 | 适用场景 |
|------|------|----------|
| `updates` | 每步状态更新 | 显示工具调用、AI 消息 |
| `messages` | LLM token 流 | 实时打字机效果 |
| `custom` | 自定义数据 | 进度、状态、调试信息 |

### 关键 API

- `agent.stream(input, stream_mode="...")`
- `agent.stream_events(input, version="v3")`：v2 统一格式
- `get_stream_writer()`：工具内发自定义事件
- 多模式并行：`stream_mode=["updates", "custom"][[`

### 使用场景
- 聊天 UI：`messages` 模式
- 调试追踪：`updates` 模式
- 长时间任务进度：`custom` 模式
- 子 Agent 区分：`subgraphs=True` + `lc_agent_name`

详见 [[langchain/streaming/流式传输]]。

---

## 第 10 章：结构化输出

**功能说明**：让 Agent 返回符合预定义 schema 的数据（替代自然语言解析）。

### 两种策略

| 策略 | 适用场景 | 优点 |
|------|---------|------|
| `ProviderStrategy` | OpenAI、Anthropic、Gemini 等 | 高可靠性、严格验证 |
| `ToolStrategy` | 所有支持 tool calling 的模型 | 通用性 |

### 关键参数

- `schema`：Pydantic / dataclass / TypedDict / JSON Schema
- `strict`：开启严格模式（部分提供商）
- `tool_message_content`：自定义工具消息内容
- `handle_errors`：错误处理策略（bool / str / Exception / Callable）

### 使用场景
- 抽取任务：返回 Pydantic 实例
- 多类型选择：`Union[A, B][[`
- 错误重试：自定义 `handle_errors` 回调
- 严格模式：财务、法律等高精度场景

详见 [[langchain/structured-output/结构化输出]]。

---

## 第 11 章：中间件

**功能说明**：通过钩子在 Agent 循环各阶段注入自定义逻辑。

### 钩子类型

- `before_agent` / `after_agent`：Agent 级别
- `before_model` / `after_model`：模型调用级别
- `before_tool` / `after_tool`：工具执行级别

### 关键 API

- `wrap_model_call`：包装模型调用
- `wrap_tool_call`：包装工具调用
- `@before_model`、`@after_model`：装饰器
- `@dynamic_prompt`：动态生成系统提示词

### 内置中间件

- `SummarizationMiddleware`：摘要对话历史
- `HumanInTheLoopMiddleware`：高危操作人工审批
- `ModelRetryMiddleware`：模型重试
- `ToolRetryMiddleware`：工具重试
- `PIIMiddleware`：PII 检测脱敏
- `TodoListMiddleware`：任务规划

### 使用场景
- 日志与监控：`wrap_model_call`
- 动态提示词：`@dynamic_prompt`
- 错误恢复：`ModelRetryMiddleware`
- 安全合规：`PIIMiddleware` + `HumanInTheLoopMiddleware`
- 上下文压缩：`SummarizationMiddleware`

详见 [[langchain/middleware/中间件概述]]。

---

## 第 12 章：多 Agent 系统

**功能说明**：通过专业化分工处理复杂任务。

### 四种核心模式

| 模式 | 特点 | 适用 |
|------|------|------|
| **子 Agent** | 主 Agent 协调多个子 Agent | 分布式开发、并行任务 |
| **交接** | 行为基于状态动态变化 | 多轮路由 |
| **技能** | 知识按需加载 | 单一 Agent + 大量领域知识 |
| **路由器** | 分类后分发 | 多领域并行 |

### 性能对比

| 场景 | 子 Agent | 交接 | 技能 | 路由器 |
|------|----------|------|------|--------|
| 单次请求 | 4 calls | 3 calls ✅ | 3 calls ✅ | 3 calls ✅ |
| 重复请求 | 8 calls | 5 calls ✅ | 5 calls ✅ | 6 calls |
| 多领域 | 5 calls | 7+ calls | 3 calls | 5 calls ✅ |

详见 [[langchain/multi-agent/多Agent系统]]。

---

## 关键 API 速查

### 创建相关

| API | 用途 | 关键参数 |
|-----|------|----------|
| `create_agent` | 创建 Agent | model、tools、system_prompt、middleware |
| `init_chat_model` | 初始化模型 | model、temperature、max_tokens |
| `@tool` | 定义工具 | name、description、args_schema |
| `add_messages` | 状态 reducer | - |

### 记忆与状态

| API | 用途 |
|-----|------|
| `InMemorySaver` | 内存 checkpointer |
| `PostgresSaver` | 生产 checkpointer |
| `InMemoryStore` | 内存 store |
| `PostgresStore` | 生产 store |
| `RemoveMessage` | 删除消息 |

### 中间件

| API | 用途 |
|-----|------|
| `@before_model` | 模型前钩子 |
| `@after_model` | 模型后钩子 |
| `@dynamic_prompt` | 动态提示词 |
| `wrap_model_call` | 包装模型调用 |
| `wrap_tool_call` | 包装工具调用 |

### 工具运行时

| API | 用途 |
|-----|------|
| `ToolRuntime` | 工具运行时参数 |
| `runtime.state` | 访问 State |
| `runtime.context` | 访问 Context |
| `runtime.store` | 访问 Store |
| `runtime.stream_writer` | 自定义流 |
| `runtime.tool_call_id` | 当前工具调用 ID |

### 响应处理

| API | 用途 |
|-----|------|
| `response_format` | 结构化输出 |
| `ProviderStrategy` | 提供商原生结构化 |
| `ToolStrategy` | 工具调用结构化 |
| `Command` | 状态更新指令 |

---

## 与已有知识库的关联

### Agent 框架层
- [[agent/orchestration/LangChain]] — 框架级实体介绍
- [[agent/RAG/LlamaIndex]] — 同类框架对比
- [[agent/orchestration/AutoGen]] — 多 Agent 框架

### 工程实践层
- [[agent/harness/Agent_三层架构]] — 感知/思考/执行
- [[agent/harness/Agent_Harness_Engineering]] — Harness 工程实践
- [[agent/harness/Function_Call]] — 工具调用机制

### 记忆与 RAG
- [[agent/memory/Agent_Memory]] — 记忆系统总览
- [[agent/RAG/RAG]] — 检索增强生成
- [[agent/RAG/向量数据库]] — 向量存储

### 高级能力
- [[agent/patterns/ReAct]] — 推理+行动
- [[langchain/modules/langchain-mcp/MCP协议概述]] — 模型上下文协议
- [[agent/skills/Skills]] — 技能机制

---

## 学习路径建议

### 入门级（1-2 天）
1. [[langchain/install/安装LangChain]]
2. [[langchain/overview/LangChain概述]]
3. [[langchain/quickstart/快速入门]]

### 核心级（3-5 天）
4. [[langchain/models/模型]]
5. [[langchain/tools/工具]]
6. [[langchain/agents/Agent代理]]

### 进阶级（5-7 天）
7. [[langchain/memory/短期记忆]]
8. [[langchain/runtime/运行时上下文]]
9. [[langchain/streaming/流式传输]]
10. [[langchain/structured-output/结构化输出]]

### 高级级（1-2 周）
11. [[langchain/middleware/中间件概述]]
12. [[langchain/multi-agent/多Agent系统]]

### 实战（持续）
- [[langchain/tutorial/智能体核心技术教程/智能体核心技术教程]] — 4 大模块综合实战

---

*文档基于 LangChain v1.x 官方文档（2026-07 同步）。*
*使用过程中如遇 API 变更，请参考 [官方文档](https://docs.langchain.com/oss/python/)。*
