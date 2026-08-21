---
tags: [deepagents, tools, task, subagent]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# Task 子代理工具详解

`task` 工具是 DeepAgents 子代理系统的入口——主 Agent 通过它派发独立任务。

## 一、调用格式

```python
# Agent 内部调用
task(
    subagent_type="general-purpose",   # 或自定义子代理名
    description="搜索 LangGraph 最新版本并返回变更摘要",
)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `subagent_type` | `str` | 子代理类型名（`general-purpose` 或自定义名） |
| `description` | `str` | 任务描述（越具体，结果越精准） |

## 二、执行流程

```
主 Agent 调 task(subagent_type="researcher", description="研究 X")
  │
  ▼
SubAgentMiddleware 拦截
  │
  ├── 查找名为 "researcher" 的子代理定义
  ├── 创建子 Agent 实例（独立上下文、独立工具、独立 model）
  ├── 子 Agent 执行: 搜 → 读 → 分析 → 总结
  ├── 子 Agent 销毁
  └── 返回一份最终报告给主 Agent
```

## 三、description 的质量决定结果

```python
# ❌ 模糊 description
task("researcher", description="研究 LangChain")
# 子代理: "LangChain 是什么？有哪些功能？历史版本？..."   → 结果宽泛

# ✅ 精确 description
task("researcher", description="搜索 LangChain 中间件系统的官方文档，整理18个中间件的分类，按场景排列")
# 子代理: 搜 LangChain middleware built-in → 读文档 → 按场景分类 → 返回结构化表格
```

## 四、通用子代理的限制

| 限制 | 说明 |
|------|------|
| 无状态 | 每次调用从头开始，不记忆之前的调用 |
| 单向通信 | 只返回一份报告，不能多轮回传 |
| 不能嵌套 | 通用子代理不能再派发子代理（同步栈无 SubAgentMiddleware） |
| 超时不重试 | 子代理超时 = task 工具返回超时错误，由主 Agent 决定是否重试 |

## 五、何时用 task vs 手动分解

| 场景 | 推荐 |
|------|------|
| 子任务独立、结果可总结为一段话 | `task` 派发 |
| 子任务需要实时交互（如"你搜一页，回来报告，再搜"） | 主 Agent 手动分解 |
| 子任务结果很大（50 页搜索） | `task` 派发（子代理隔离上下文） |
| 子任务需要访问主 Agent 的上下文 | 主 Agent 手动分解 |

## 六、相关页面

- [[deepagents/subagents/子代理概述]] — 子代理系统基础
- [[deepagents/subagents/子代理配置实战]] — 生产级子代理配置
- [[deepagents/tools/内置工具概述]] — 所有内置工具
