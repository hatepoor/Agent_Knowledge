---
tags: [a2a, 协议, 术语, 索引, AI]
type: index
created: 2026-10-13
updated: 2026-10-13
---

# A2A 术语与关键字手册

> **本页定位：只给「这是什么 + 一句话含义 + 详解在哪」，不展开解释。**
> 解释与机制在正文，本页只做兜底索引。**看不懂某个名词时先来这里查，再决定要不要回正文精读。**
> 用法：Ctrl+F 搜名词 → 看「详解」列的链接 → 跳过去。

---

## 一、协议与治理类名词

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **A2A / Agent2Agent Protocol** | 智能体之间发现彼此、委派任务、交换结果的开放通信标准 | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.1-A2A是什么]] |
| **MCP / Model Context Protocol** | 智能体连接工具与数据的标准，Anthropic 2024-11 发布 | [[langchain/modules/langchain-mcp/MCP协议概述]] |
| **AAIF / Agentic AI Foundation** | Linux Foundation 下的中立基金会，2025-12-09 成立，同时托管 MCP 与 A2A | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.3-版本演进与治理历程]] |
| **Linux Foundation** | 非营利基金会，2025-06-23 接收 A2A 捐赠 | 同上 |
| **TSC / Technical Steering Committee** | 技术指导委员会，A2A 由 8 家公司代表组成 | 同上 |
| **Growth Stage project** | AAIF 对项目的成熟度分级，A2A 2026-08 获此级别 | 同上 |
| **ACP / Agent Communication Protocol** | IBM 的横向协议，2025-08 已并入 A2A，不再独立发展 | 同上 |
| **AP2 / Agent Payments Protocol** | Google 的 A2A 支付层扩展，agentic commerce 用 | [[a2a/02-中段阶段-生产实践/06-生态现状与厂商支持]] |
| **ANP / Agent Network Protocol** | 面向开放互联网的去中心化 agent 发现协议，比 A2A 更早期 | 同上 |
| **agentgateway** | AAIF 项目，agent 流量的路由/策略/可观测性代理 | 同上 |
| **goose** | Block 的开源 agent 运行时，AAIF 项目 | 同上 |
| **AGENTS.md** | OpenAI 的纯文本约定，告诉编码 agent 如何在仓库里工作 | 同上 |

---

## 二、核心数据对象（五个，必须掌握）

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **Agent Card** | Agent 的身份证与能力说明书，一个 JSON 文档 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.1-AgentCard与智能体发现]] |
| **Agent Skill** | Agent Card 里的一条能力声明，含 id、name、输入输出格式 | 同上 |
| **Task** | 有状态的工作单元，有唯一 ID 和生命周期 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.4-Task生命周期与状态机]] |
| **Message** | 客户端与远端之间的一轮通信，含 role 和若干 Part | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.2-Task-Message-Part-Artifact四件套]] |
| **Part** | 最小内容单元，三选一：text / file / data | 同上 |
| **Artifact** | Task 产出的实际交付物，由若干 Part 组成 | 同上 |
| **Context / contextId** | 服务端生成的标识符，把相关 Task 与 Message 归为同一次会话 | 同上 |

---

## 三、参与者与角色

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **A2A Client（客户端 Agent）** | 代表用户发起 A2A 交互的一方 | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.1-A2A是什么]] |
| **A2A Server（远端 Agent）** | 暴露 HTTP 端点、实现 A2A 协议的一方 | 同上 |
| **Orchestrator Agent（编排 Agent）** | 决定把任务交给谁的 Agent，A2A 场景里的常见角色 | [[a2a/02-中段阶段-生产实践/03-A2A与MCP选型决策树]] |
| **opaque（黑盒）** | 协议要求远端 Agent 不暴露内部记忆、工具、逻辑 | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.1-A2A是什么]] |

---

## 四、Task 状态（8 个）

| 状态 | 类型 | 一句话含义 | 详解 |
|------|------|-----------|------|
| `submitted` | 非终态 | 已接收，还没开始处理 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.4-Task生命周期与状态机]] |
| `working` | 非终态 | 正在处理 | 同上 |
| `input-required` | 非终态 | 需要调用方补充信息才能继续 | 同上 |
| `auth-required` | 非终态 | 需要调用方重新鉴权才能继续 | 同上 |
| `completed` | 终态 | 成功完成，Artifact 已挂载 | 同上 |
| `failed` | 终态 | 处理中途出错终止 | 同上 |
| `canceled` | 终态 | 客户端主动取消 | 同上 |
| `rejected` | 终态 | 服务端在开始处理前就拒绝（策略/不支持/限流） | 同上 |

---

## 五、交互机制与传输

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **Polling（轮询）** | 客户端周期性调 `tasks/get` 查状态 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.3-三种交互机制与协议绑定]] |
| **Streaming / SSE** | 打开长连接，实时接收增量结果 | 同上 |
| **Push Notification（推送通知）** | 服务端在状态变化时 POST 到客户端提供的 webhook | 同上 |
| **Webhook** | 客户端暴露给服务端回调的 HTTP 端点 | 同上 |
| **Protocol Binding（协议绑定）** | 数据模型落到具体传输的方式，v1.0 有三套 | 同上 |
| **JSON-RPC 2.0** | 默认绑定使用的报文格式，`method` + `params` + `id` | 同上 |
| **gRPC** | 基于 HTTP/2 + Protocol Buffers 的 RPC 框架，A2A 三套绑定之一 | 同上 |
| **HTTP+JSON / REST** | 传统 REST 风格绑定，A2A 三套绑定之一 | 同上 |
| **JSON Schema 2020-12** | 数据模型除 protobuf 外的第二份定义，由 proto 自动生成 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.3-三种交互机制与协议绑定]] |
| **Protocol Buffers（protobuf）** | Google 的二进制序列化格式，A2A 数据模型的权威定义 | 同上 |
| **SSE / Server-Sent Events** | 服务端单向推流的 HTTP 机制 | 同上 |

---

## 六、发现与安全

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **Agent Discovery（智能体发现）** | 客户端找到 Agent Card 并据此判断能否委派任务的过程 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.1-AgentCard与智能体发现]] |
| **well-known URI** | 固定路径约定 `/.well-known/agent-card.json`（RFC 8615） | 同上 |
| **extendedAgentCard（扩展卡）** | 需认证后才能拉取的完整版 Agent Card，用于藏内部端点 | 同上 |
| **Signed Agent Card（签名卡）** | 带 `signatures` 字段的卡，可验证发卡方身份，防冒名 | 同上 |
| **Agent Registry（智能体注册中心）** | 集中存放 Agent Card 目录的服务，如 Google Cloud Agent Registry | 同上 |
| **securitySchemes** | Agent Card 中声明本 Agent 接受哪些鉴权方式 | [[a2a/02-中段阶段-生产实践/01-安全模型与鉴权]] |
| **OAuth 2.0 / 2.1** | 委托授权的标准协议，跨组织场景主流选择 | 同上 |
| **DPoP** | OAuth 2.1 的令牌绑定机制，防止令牌被窃取后重放 | 同上 |
| **OpenID Connect** | 在 OAuth 2.0 上加身份层的标准 | 同上 |
| **mTLS / 双向 TLS** | 客户端与服务端互相验证证书 | 同上 |
| **API Key** | 最简单的静态凭证，安全性最低 | 同上 |
| **SSRF** | 服务端请求伪造，校验 webhook URL 的主要理由 | 同上 |

---

## 七、工程与生态

| 名词 | 一句话含义 | 详解 |
|------|-----------|------|
| **a2a-python** | A2A 官方 Python SDK | [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.1-Python-SDK最小可用实现]] |
| **a2a-samples** | 官方示例仓库，含与 LangGraph / CrewAI / Google ADK 的集成范例 | 同上 |
| **a2a-cli** | 官方命令行工具，可在终端发现、发消息、管理 A2A 智能体 | [[a2a/02-中段阶段-生产实践/06-生态现状与厂商支持]] |
| **ADK / Agent Development Kit** | Google 的 Agent 开发框架，可导出 A2A 端点 | 同上 |
| **Agent Engine / Vertex AI** | Google Cloud 的 agent 托管平台，已支持 A2A | 同上 |
| **Azure AI Foundry** | 微软的 agent 平台，可暴露/发现 A2A 端点 | 同上 |
| **Bedrock AgentCore** | AWS 的 agent 运行时，可托管 A2A 服务端 | 同上 |
| **agent-as-tool（智能体即工具）** | 把远端 A2A Agent 包成本地工具调用的模式 | [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.2-把远端Agent包装成本地工具]] |
| **breaking change（破坏性变更）** | 不兼容旧版本的改动，v0.3→v1.0 即是 | [[a2a/02-中段阶段-生产实践/02-版本演进与迁移陷阱]] |
| **semver / 语义化版本** | 版本号约定。**`0.y.z` 阶段不承诺向后兼容**，所以 0.x → 1.0 出现破坏性变更是正常的 | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.3-版本演进与治理历程]] |
| **preferredTransport** | Agent Card 中声明首选协议绑定的字段（v1.0） | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.1-AgentCard与智能体发现]] |
| **capabilities.streaming / pushNotifications** | Agent Card 的两个能力位，决定你用流式还是轮询、推送还是轮询 | 同上 |
| **a2a-sdk** | A2A 在 PyPI 上的包名。⚠️ **与 GitHub 仓库名 `a2a-python` 不同**，找资料时两个都要搜 | [[a2a/速查索引]] |
| **Starlette / Uvicorn / httpx** | Python 侧的轻量 ASGI 框架、ASGI 服务器、HTTP 客户端，官方示例用它们跑通服务端 | [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.1-Python-SDK最小可用实现]] |
| **FastAPI** | A2A 路由可**直接挂载**到 FastAPI，鉴权、限流、日志、OpenAPI 都能复用（官方明确支持） | [[a2a/01-入门阶段-协议基础/第02章-工程实践/2.1-Python-SDK最小可用实现]] |
| **DeepAgents** | 本知识库的另一主题目录。A2A 是边界层，DeepAgents 是 Agent 运行时层，**两者互补不冲突** | [[deepagents/overview/DeepAgents概述]] |
| **supportedInterfaces** | Agent Card 中列出可用接口与绑定的字段 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.1-AgentCard与智能体发现]] |
| **extensions** | 协议之外的扩展声明机制，核心规范保持稳定 | [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.3-三种交互机制与协议绑定]] |

---

## 八、A2A 明确「不是」什么

| 名词 | 为什么容易搞混 | 详解 |
|------|---------------|------|
| **A2A 不是 Agent 框架** | LangGraph / CrewAI / ADK 才是框架，A2A 是它们之间的通信层 | [[a2a/01-入门阶段-协议基础/第00章-学前准备/0.1-A2A是什么]] |
| **A2A 不是子 Agent 协议** | 你的 Agent 内部怎么调子 Agent，A2A 不管 | 同上 |
| **A2A 不是 MCP 的替代** | MCP 管 agent→tool，A2A 管 agent→agent | 同上 |
| **A2A 不是聊天应用** | 机器对机器，不是 Slack/微信 | 同上 |
