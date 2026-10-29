---
tags: [interview, langgraph, 面试, 题库, 导航]
type: moc
created: 2026-10-18
updated: 2026-10-18
---

# LangGraph 面试题库

> **本目录的定位：把 LangGraph 的知识点变成「面试能说出口的答案」。**
>
> 与 `interview/`（跨主题面试总库）的分工：**总库讲答题策略与项目话术，本目录按知识主题归档具体题目。**
> 逐题格式统一为「先问题、后答案」，**每篇不超过 10 题，且同篇只讲一个主题**。

---

## 一、版本口径（2026-10）

| 项 | 说明 |
|---|---|
| 版本基线 | LangGraph **1.x 稳定线**（1.0 于 2025-10-22 发布，1.1 / 1.2 为后续minor） |
| 重要变化 | 1.0 起 `langgraph.prebuilt` 命名空间弃用，`create_react_agent` 等推荐改用 `langchain.agents` |
| 包结构 | `langgraph`（图引擎）/ `langgraph-checkpoint`（持久化抽象）/ `langgraph-checkpoint-postgres`（生产后端） |
| 注意 | 网上大量 0.x 教程仍能跑，但导入路径与中断 API 有差异，面试举例时要说清版本 |

⚠️ **不要背具体默认值**（如递归上限、连接池大小）。这些随版本变，说「有上限保护、可配置」比报错数字安全。

---

## 二、文件清单

| 文件 | 主题 | 题数 |
|------|------|------|
| [[langgraph/interview/00-面试总纲与答题框架]] | 答题方法论、项目话术、速查 | 非题库 |
| [[langgraph/interview/01-核心概念与运行模型]] | 定位、StateGraph、superstep、Channel | 9 |
| [[langgraph/interview/02-状态与Reducer]] | TypedDict、Annotated、并行写入合并 | 8 |
| [[langgraph/interview/03-控制流与动态编排]] | 条件边、Send、Command、循环、合流 | 9 |
| [[langgraph/interview/04-持久化与Checkpointer]] | checkpoint 粒度、thread_id、时间旅行 | 9 |
| [[langgraph/interview/05-人机协同与中断恢复]] | interrupt 原理、恢复、幂等 | 8 |
| [[langgraph/interview/06-流式输出与可观测性]] | stream_mode、astream_events、追踪 | 8 |
| [[langgraph/interview/07-多智能体与子图]] | Supervisor、handoff、层级、上下文隔离 | 9 |
| [[langgraph/interview/08-生产实践与高频追问]] | 部署、并发、超时重试、选型 | 9 |

**合计 8 篇、69 题。**

---

## 三、使用方式

⚠️ **题库不是用来「读」的，是用来「练」的。**

```
第一遍：只看问题，合上书口述答案，卡住就做个标记
第二遍：读答案，重点看「追问」——面试官真正拉开差距的是第二层
第三遍：把标红的题连追问一起复述，录音回听
```

**判断标准**：能脱稿讲 60 秒 + 接住一层追问 = 这题过了。看懂 ≠ 会说。

---

## 四、题目来源与筛选口径

参考（2026-10 核对）：

| 来源 | 用途 |
|------|------|
| 面试鸭（mianshiya.com） | 高频题分布、追问风格 |
| 牛客网 | 面经里的真实提问顺序 |
| CSDN / 掘金 | 社区整理的长尾题、踩坑题 |
| LangGraph 官方文档与 release notes | **事实核对基准**，社区文章与官方冲突时以官方为准 |

⚠️ **筛选原则：只收「能被追问三层」的题。** 纯 API 记忆题（某参数叫什么）不收，因为版本一变就废；机制题、选型题、踩坑题才是长期有效的。

---

## 五、和知识库其他部分的关系

本目录**不重复讲原理**，只给「答题版本」。要吃透机制，回到：

| 主题 | 详解页 |
|------|--------|
| 图计算范式 | [[langgraph/concepts/图计算范式]] |
| StateGraph / 节点 / 边 | [[langgraph/modules/langgraph_core/概述]] |
| 状态与 Reducer | [[langgraph/modules/langgraph_state/概述]] |
| Checkpointer | [[langgraph/modules/langgraph_checkpoint/概述]] |
| 中断机制 | [[langgraph/modules/langgraph_human_in_loop/interrupt机制]] |
| 流式模式 | [[langgraph/modules/langgraph_streaming/流式模式对比]] |
| 多智能体模式 | [[langgraph/modules/langgraph_multi_agent/概述]] |
| 子图 | [[langgraph/modules/langgraph_subgraph/概述]] |
| 项目话术 | [[interview/04-项目经验/项目一-终端多智能体系统]] |

---

## 六、答题总原则

| 原则 | 说明 |
|------|------|
| **先给结论，再展开** | 第一句必须是「它是什么/解决什么问题」，不要铺垫 |
| **机制 > API** | 说清「为什么这样设计」，比背方法名得分高 |
| **主动给权衡** | 「A 适合 X，B 适合 Y」比「A 更好」显得有工程判断 |
| **带一个真实坑** | 每个主题准备一个踩坑故事，证明写过而不是看过 |
| **不确定就说不确定** | 版本细节记不清时，说「这块不同版本有差异，我按 1.x 讲」 |
