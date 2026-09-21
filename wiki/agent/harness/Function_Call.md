---
tags: [concept, AI, LLM, 工具调用, Agent]
created: 2026-06-23
updated: 2026-06-23
type: concept
---

# Function Call（函数调用）

## 定义

Function Call 是大语言模型（LLM）与外部工具/服务交互的核心机制。它使 LLM 能够**自主判断**"是否需要调用工具"以及"调用哪个工具"，然后以结构化格式（通常是 JSON）输出工具调用请求，由外部系统执行后将结果返回给 LLM。

## 核心流程

```
用户请求 → LLM 判断是否需要工具 → 输出函数调用 JSON → 外部系统执行 → 结果返回 LLM → LLM 整合生成最终回答
```

## 关键特性

- **自主决策**：LLM 自行判断何时需要调用工具，无需硬编码规则
- **结构化输出**：以标准化 JSON 格式输出函数名和参数，便于程序解析
- **多工具支持**：一次推理中可选择调用多个工具
- **闭环反馈**：工具执行结果回传 LLM，形成"思考→行动→观察"循环

## 典型应用场景

| 场景 | 工具示例 |
|------|----------|
| 联网搜索 | SerpAPI、Google Search API |
| 文档解析 | PyPDF、Unstructured |
| 数学计算 | Python REPL、Wolfram Alpha |
| 数据库查询 | SQL 执行器 |
| 企业系统对接 | 飞书/钉钉 API、订单查询接口 |

## 在 Agent 架构中的角色

Function Call 是 [[agent/harness/Agent_三层架构]] 中**思考层驱动执行层**的关键机制。思考层通过 Function Call 决定调用哪些工具，执行层负责实际执行并将结果返回。

## 设计原则

- **场景必需**：只挂载当前场景必需的工具，过多工具会导致误调用
- **描述清晰**：工具的名称和描述要让 LLM 能准确理解其用途
- **参数明确**：函数参数需要有清晰的类型和说明

## 相关概念
- [[agent/harness/Agent_三层架构]] — Function Call 连接思考层和执行层
- [[agent/memory/Agent_Memory]] — 记忆检索本身也是一种 Function Call
- [[agent/harness/Agent沙箱与安全隔离]] — 工具调用的安全执行必须在沙箱内进行
- [[agent/orchestration/LangChain]] — 提供了完善的 Function Call 封装
- [[agent/orchestration/开发Agent实操指南]] — 原文中"加工具调用能力"的核心机制

## 来源
- [[raw/Clippings/如何开发一个Agent，来手把手教你]] — 原文"加工具调用能力"章节
