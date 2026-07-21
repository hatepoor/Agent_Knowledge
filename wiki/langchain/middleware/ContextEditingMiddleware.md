---
tags: [langchain, middleware, context, token-management]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#context-editing
langchain_version: ">=1.0"
---

# ContextEditingMiddleware（上下文编辑中间件）

## 一、定义与作用

`ContextEditingMiddleware` 在上下文**即将超出 token 限制**时，按预设规则**编辑（删除/替换）旧消息**以回收 token。

它与 `SummarizationMiddleware` 的核心区别：
- **Summarization**：把旧消息**压缩**成一段摘要（信息保留）
- **ContextEditing**：把旧消息**直接清除**或**替换为占位符**（信息丢失但更快、更省）

最常用的配套策略是 `ClearToolUsesEdit`——清除较早的工具调用输出（用占位符替代），但保留工具调用的"事实记录"（AI 知道曾经调过这个工具，但不再把完整结果塞给模型）。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 工具调用密集型 Agent | 搜索 Agent 一次返回 20 条结果，后续对话不必每次都重看 |
| 长期运行的代码 Agent | 一次会话读了几十个文件，旧文件内容可清除 |
| 调试/开发场景 | 想要控制 token 用量但不需要保留所有历史 |
| 与 Summarization 配合 | 摘要前先清空工具输出，进一步压缩 |

## 三、参数详解

### ContextEditingMiddleware

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `edits` | `list[ContextEdit]` | `[ClearToolUsesEdit()]` | 应用的编辑策略列表 |
| `token_count_method` | `string` | `"approximate"` | `'approximate'`（快速但粗略）或 `'model'`（精确但慢） |

### ClearToolUsesEdit

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `trigger` | `number` | `100000` | 触发编辑的 token 计数 |
| `clear_at_least` | `number` | `0` | 至少回收的 token 数 |
| `keep` | `number` | `3` | 必须保留的最近工具结果数 |
| `clear_tool_inputs` | `boolean` | `False` | 是否清空 AI 消息中工具调用参数 |
| `exclude_tools` | `list[string]` | `()` | 不清除其输出的工具名列表 |
| `placeholder` | `string` | `"[cleared]"` | 清除后插入的占位符文本 |

## 四、使用方法

### 基础用法：清除旧工具输出

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ContextEditingMiddleware,
    ClearToolUsesEdit,
)

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[search_web, read_file],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=10000,          # 累计 10000 tokens 触发
                    keep=3,                  # 保留最近 3 个工具结果
                    clear_tool_inputs=False, # 保留工具调用的入参
                    placeholder="[已清除历史工具输出]",
                ),
            ],
        ),
    ],
)
```

### 多个编辑策略组合

```python
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

# 自定义一个编辑策略：保留最近 5 个工具结果
class CustomEdit(ClearToolUsesEdit):
    def __init__(self):
        super().__init__(trigger=8000, keep=5, exclude_tools=["calculator"])

agent = create_agent(
    model=llm,
    tools=tools,
    middleware=[ContextEditingMiddleware(edits=[CustomEdit()])],
)
```

## 五、注意事项

1. **与 Summarization 的选择**：
   - 想要"信息保留"——用 `SummarizationMiddleware`
   - 想要"快速控制 token"——用 `ContextEditingMiddleware`
   - 想要"既要又要"——可以同时使用（`edits` 先清工具输出，节省的 token 留给后续摘要）

2. **`keep` 值不能太小**：保留的最近工具结果数太少会让 AI 失去当前任务上下文

3. **`exclude_tools` 谨慎使用**：如果排除了关键工具（如数据库查询），AI 可能会基于占位符做错误决策

4. **`placeholder` 会让 AI 困惑**：要选能表达"信息已清除"的占位符，避免 AI 误以为是真实数据

5. **`token_count_method="model"` 性能差**：会调用模型计算 token，频繁触发时严重影响性能。除非必要，用 `approximate` 即可

## 六、相关页面

- [[langchain/middleware/SummarizationMiddleware]] — 摘要压缩（信息保留）
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/memory/上下文压缩]] — 传统压缩方案
