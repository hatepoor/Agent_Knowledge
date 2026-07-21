---
tags: [source, Agent, Harness-Engineering, 工程实践, 阿里云]
type: source
created: 2026-06-30
updated: 2026-07-08
---

# Agent Harness 工程实践

> 原文：给野马套上缰绳：Agent Harness 工程实践 ——从范式理论到钉钉AI招聘的真实落地
> 作者：阿里云开发者
> 发布时间：2026年6月30日
> 来源：ima「AI Agent」知识库

## 核心观点

Agent = Model + Harness。瓶颈不在模型够不够聪明，而在我们有没有把它"装"好。

> 理论体系（三代范式、四条铁律、六大模式）详见 [[agent/harness/Agent_Harness_Engineering]] 及其子页面。本文聚焦**真实落地案例**。

## 实战案例：悟空 AI 招聘 Agent

### 第一版：全能招聘 Agent（失败）

- 一个超级 Agent 包办所有事情
- Prompt 写了 600+ 行
- 工具列表 13 个
- **问题**：上下文炸了、工具选错、状态丢失、错误难复现

### 第二版：2 Agent + N Skill 的专才架构

- **Orchestrator（悟空）**：拆任务、分发、维护 Workspace
- **人岗匹配 Agent**：职责聚焦、Prompt 80 行、Tools 4 个
- **招聘沟通 Agent**：职责聚焦、Prompt 90 行、Tools 5 个
- **N 个原子化 Skill**：parse_resume、score_match、query_calendar 等

### 改造前后对比

| 维度 | 全能 Agent | 专才架构 |
|------|-----------|------------|
| 工具选择 | n 工具混选，错率明显偏高 | 每 Agent 只看 4-5 工具，错率显著下降 |
| 端到端准确率 | 达不到上线门槛 | 稳定超过上线门槛，可生产运行 |
| 可调试性 | "不知道哪步错了"，回滚要查数小时日志 | 日志按 Agent 分桶，分钟级定位 |
| 可复用性 | 换场景就废 | 人岗匹配 Agent 已复用到内推系统 |
| 上下文消耗 | 数量级偏高，长期接近模型上限 | 显著下降，长期处于安全区 |

## 六句心法

1. 你优化的不是 Agent，是 Agent 的工作环境
2. 上下文是稀缺资源，不是无限仓库
3. 状态写在文件里，不在脑子里
4. 能写成 Linter 的约束，别写成文档
5. Agent 是昂贵的，Skill 是廉价的，护栏是最便宜的
6. 对外说话和动用户数据的地方，硬护栏要早一步到位

## 相关页面

- [[agent/harness/Agent_Harness_Engineering]] — 概念总览与知识导航
- [[agent/harness/四条反直觉铁律]] — 铁律详解（本文的理论基础）
- [[agent/harness/六大工程模式]] — 六大模式详解（本文的工程方法论）
- [[agent/skills/Skills]] — Skill 管理最佳实践
- [[agent/memory/Agent_Memory]] — Agent 状态管理

## 来源

- ima 知识库：「AI Agent」- 阿里云开发者
- Mitchell Hashimoto - My AI Adoption Journey
- LangChain - Improving Deep Agents with Harness Engineering
- HuggingFace - Agent 是昂贵的，Skill 是廉价的
