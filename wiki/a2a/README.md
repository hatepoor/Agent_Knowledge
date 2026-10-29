---
tags: [a2a, 协议, 导航, AI, Agent]
type: moc
created: 2026-10-13
updated: 2026-10-13
---

# A2A 协议知识库

> **A2A（Agent2Agent Protocol，智能体对智能体协议）** 是一套让不同公司、不同框架、不同作者写的 AI 智能体互相**发现彼此、委派任务、交换结果**的开放通信标准。Google 于 2025-04-09 发布，现由 Linux Foundation 下的 Agentic AI Foundation（AAIF）治理。
>
> **一句话记法：MCP 让智能体拿到工具，A2A 让智能体找到同行。**

## 阅读前提与门槛声明

⚠️ **本页及整个 `a2a/` 目录的阅读顺序有强依赖，请按下表判断从哪进。**

| 你的情况 | 从哪开始 | 说明 |
|---------|---------|------|
| **完全没听说过 Agent / MCP** | 先读 [[langchain/modules/langchain-mcp/MCP协议概述]]，再回来读本页 | A2A 默认你已经理解「工具调用」「Agent 契约」这些概念 |
| **做过 LangChain / LangGraph 单体 Agent** | 直接读本页 → [[a2a/01-入门阶段-协议基础/00-章节目录\|章节目录]] | 第00章会补齐协议类概念 |
| **已经在做多智能体（LangGraph 扇出、DeepAgents 子 Agent）** | 直接读本页，然后重点读 [[a2a/01-入门阶段-协议基础/第01章-核心对象/1.4-Task生命周期与状态机]] | 你最关心的是 Task 状态机与本地/跨界的区别 |
| **只想快速判断"要不要用 A2A"** | 读 [[a2a/02-中段阶段-生产实践/03-A2A与MCP选型决策树]] 一页 | 决策树 + 判定规则，不需要读完全部 |
| **要做跨组织集成的架构决策** | 全读，按 [[a2a/学习路线图]] 的顺序 | 涉及鉴权、授权、版本迁移等生产问题 |

**看不懂时该退回哪里：**
- 协议类名词（Agent Card、Task、Artifact…）看不懂 → [[a2a/A2A术语与关键字手册]]
- 记不住方法名和状态名 → [[a2a/速查索引]]
- 觉得「A2A 太重了、我这个小项目用不上」 → 这不是错觉，[[a2a/02-中段阶段-生产实践/04-落地时机与不适用场景]] 会明确告诉你**什么时候不需要它**
- 想知道它和你已学的 LangGraph 多智能体什么关系 → [[a2a/02-中段阶段-生产实践/05-与LangGraph多智能体的关系]]

## 版本基线（2026-10-13 核实）

| 事实 | 内容 |
|------|------|
| 最新稳定规范 | **v1.0.1**（2026-05-28）；v1.0.0 首发 2026-03-12 |
| 之前的版本 | v0.3.0（2025-07-30）、v0.2.6、v0.1.0 |
| 治理 | Linux Foundation 技术指导委员会（TSC）8 席：AWS、Cisco、Google、IBM Research、Microsoft、Salesforce、SAP、ServiceNow |
| 归属 | 2025-06-23 捐赠给 Linux Foundation；2026-08-17 被 AAIF 接纳为 Growth Stage 项目（AAIF 同时托管 MCP） |
| 许可证 | Apache 2.0 |
| 官方 SDK | Python、JS、Java、C#/.NET、Go、Rust（6 种） |
| 协议绑定 | JSON-RPC 2.0 over HTTPS（默认）、gRPC、HTTP+JSON / REST |

⚠️ **v1.0 相对 v0.3 有破坏性变更**，基于 0.3 构建的 Agent 需要迁移。详见 [[a2a/02-中段阶段-生产实践/02-版本演进与迁移陷阱]]。

## 目录结构

```
a2a/
├── README.md                    ← 本页：导航 + 门槛声明 + 版本基线
├── 学习路线图.md                  ← 两条路径（概念优先 / 落地优先）+ 自测清单
├── A2A术语与关键字手册.md          ← ★ 专有名词总索引（看不懂名词先查这里）
├── 速查索引.md                    ← 方法名、状态名、字段名、JSON 骨架速查
│
├── 01-入门阶段-协议基础/           ← 零基础起点，按章节顺序推进
│   ├── 00-章节目录.md
│   ├── 第00章-学前准备/
│   │   ├── 0.1-A2A是什么.md
│   │   ├── 0.2-为什么需要A2A.md
│   │   └── 0.3-版本演进与治理历程.md
│   ├── 第01章-核心对象/
│   │   ├── 1.1-AgentCard与智能体发现.md
│   │   ├── 1.2-Task-Message-Part-Artifact四件套.md
│   │   ├── 1.3-三种交互机制与协议绑定.md
│   │   └── 1.4-Task生命周期与状态机.md
│   └── 第02章-工程实践/
│       ├── 2.1-Python-SDK最小可用实现.md
│       └── 2.2-把远端Agent包装成本地工具.md
│
└── 02-中段阶段-生产实践/           ← 平行专题，跳读
    ├── 00-专题导航.md
    ├── 01-安全模型与鉴权.md
    ├── 02-版本演进与迁移陷阱.md
    ├── 03-A2A与MCP选型决策树.md
    ├── 04-落地时机与不适用场景.md
    ├── 05-与LangGraph多智能体的关系.md
    └── 06-生态现状与厂商支持.md
```

## 与已有内容的衔接

| 相关页面 | 关系 |
|---------|------|
| [[langchain/modules/langchain-mcp/MCP协议概述]] | **前置知识**。A2A 是 MCP 的横向补充，不是替代 |
| [[langchain/modules/langchain-mcp/MCP与Agent中间件集成]] | MCP 侧实操，对照阅读 |
| [[langgraph/modules/langgraph_multi_agent/概述]] | A2A 与「进程内多智能体」的边界对比 |
| [[agent/patterns/MultiAgent协作]] | Supervisor/Network/Hierarchical 模式，A2A 的本地简化版 |
| [[deepagents/tools/MCP集成]] | DeepAgents 侧 MCP 接入，A2A 在此之上再叠一层 |
| [[agent/harness/Agent_Harness_Engineering]] | 该页提到「MCP → A2A 的标准化演进」，A2A 详情见本目录 |
