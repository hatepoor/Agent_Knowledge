---
tags: [concept, Agent, Harness-Engineering, 工程范式]
type: concept
created: 2026-07-01
updated: 2026-07-08
---

# Agent Harness Engineering

> Harness Engineering 是 AI Agent 工程的第三代范式，核心是"给野马套上缰绳"——通过工程化手段让 Agent 可控地工作。

## 定义

**Agent = Model + Harness**

- **Model**：负责推理（Reasoning）
- **Harness**：负责"剩下的所有事情"——工具系统、上下文管理、权限控制、反馈回路、记忆与协作

这个公式的颠覆性在于它彻底改变了优化方向：过去几年大量精力放在"换更强的模型"上，但真正的杠杆位置一直在模型之外。

## 三代范式演进

| 代际 | 范式 | 核心问题 | 形象对比 |
|------|------|----------|----------|
| 第一代 | [[agent/prompt/提示词工程|Prompt Engineering]] | 怎么把话说清楚 | 对马喊话的技巧 |
| 第二代 | [[agent/prompt/上下文工程|Context Engineering]] | 怎么给 AI 喂对信息 | 给马看的地图 |
| 第三代 | Harness Engineering | 怎么让 Agent 可控地工作 | 给马造高速公路，配护栏、限速牌、加油站 |

## 核心知识体系

Harness Engineering 的完整知识分为三个层次：

### 反直觉认知（为什么）

四条铁律揭示了 Harness 思维与工程师直觉的根本差异 → 详见 [[agent/harness/四条反直觉铁律]]

### 工程模式（怎么做）

六种经生产验证的设计模式，构成 Harness 的"设计模式手册" → 详见 [[agent/harness/六大工程模式]]

### 实战案例（谁在做）

从阿里云悟空招聘 Agent 的真实落地案例，展示全能 Agent → 专才架构的改造全过程 → 详见 [[agent/harness/Agent_Harness_工程实践_阿里云开发者]]

## 行业速览

| 团队 / 产品 | 最有辨识度的 Harness 选择 |
|--------------|--------------------------|
| Anthropic / Claude Code | Initializer + Executor 双阶段架构 |
| LangChain / Deep Agents | 自我验证 + 工具签名优化 |
| Cursor / Cline | 内置反馈回路（Linter / Test 自动闭环） |
| 悟空 AI 招聘 | 2 Agent + N Skill + 硬护栏 |

> 各团队的具体做法和启示，详见上述各子页面。

## 未来趋势

1. **从"Prompt 工程师"到"Agent 工程师"**：岗位正在快速转型
2. **从"工具调用"到"Agent 协作协议"**：MCP → A2A 的标准化演进
3. **从"单 Agent"到"Agent 操作系统"**：Shell + Scheduler + System Calls
4. **模型与 Harness 接口标准化**：Agent 工程进入工业化阶段

## 相关页面

- [[agent/harness/四条反直觉铁律]] — Harness 的认知基础
- [[agent/harness/六大工程模式]] — Harness 的实践手册
- [[agent/harness/Agent_Harness_工程实践_阿里云开发者]] — 真实落地案例
- [[agent/harness/Agent_三层架构]] — Agent 的通用三层架构
- [[agent/harness/Agent_六种架构]] — 六种 Agent 组织方式
- [[agent/prompt/上下文工程]] — 第二代范式详解
- [[agent/prompt/提示词工程]] — 第一代范式详解

## 来源

- ima 知识库：「AI Agent」
- 阿里云开发者：《给野马套上缰绳：Agent Harness 工程实践》
- Mitchell Hashimoto - My AI Adoption Journey
- LangChain - Improving Deep Agents with Harness Engineering
- HuggingFace - Agent 是昂贵的，Skill 是廉价的
