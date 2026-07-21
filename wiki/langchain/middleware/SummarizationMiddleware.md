---
tags: [langchain, middleware, context, summary]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#summarization
langchain_version: ">=1.0"
---

# SummarizationMiddleware（对话摘要中间件）

## 一、定义与作用

`SummarizationMiddleware` 在对话历史接近 token 限制时，**自动用 LLM 总结较早的对话**，只保留最近若干条消息，从而避免上下文窗口溢出。

它是 LangChain 处理长对话的"标准答案"——比"无脑截断前面 100 条"更聪明，保留了关键信息。

**核心价值**：当用户跟 Agent 聊了几百轮后，旧消息被压缩成一段摘要，LLM 仍然知道早期讨论了什么，但 token 占用大幅下降。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 客服 Agent | 用户反馈 50 轮后，AI 仍记得最初的问题描述 |
| 代码助手 | 跨多个文件讨论几小时后，AI 仍记得之前重构的决策 |
| 教学 Agent | 长程辅导，AI 记得学生最开始的薄弱知识点 |
| 多任务助理 | 用户穿插多个话题，AI 不会把"帮我订机票"和"解释 Python 闭包"混淆 |

**不适用的场景**：
- 短对话（< 10 轮）——增加额外 LLM 调用反而浪费钱
- 需要**精确**回溯每一句话（如法律/医疗场景）——摘要会丢失细节

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `model` | `string \| BaseChatModel` | **必填** | 用于生成摘要的 LLM 实例（**不能是字符串**） |
| `trigger` | `ContextSize \| TriggerClause \| list \| None` | — | 触发摘要的条件，详见下方 |
| `keep` | `ContextSize` | `('messages', 20)` | 摘要后保留多少最近的上下文 |
| `token_counter` | `function` | 字符计数（粗略） | 自定义 token 计数函数 |
| `summary_prompt` | `string` | 内置模板 | 自定义摘要提示模板（需含 `{messages}`） |
| `trim_tokens_to_summarize` | `number` | `4000` | 摘要时包含的最大 token 数 |

### trigger 参数格式

```python
# 三种触发粒度
trigger=("fraction", 0.8)    # 上下文达到 80% 时触发（按模型窗口算）
trigger=("tokens", 4000)     # 总 token 数达到 4000 时触发
trigger=("messages", 50)     # 消息数达到 50 时触发

# 多个条件（任一满足即触发）
trigger=[("tokens", 4000), ("messages", 30)]
```

### 已弃用参数

| 弃用参数 | 替代方案 |
|----------|----------|
| `summary_prefix` | `summary_prompt` |
| `max_tokens_before_summary` | `trigger=("tokens", value)` |
| `messages_to_keep` | `keep=("messages", value)` |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langchain_deepseek import ChatDeepSeek

# ⚠️ 必须是模型实例，不能是字符串
summary_llm = ChatDeepSeek(model="deepseek-v4-flash", temperature=0.0)

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model=summary_llm,
            trigger=("tokens", 4000),     # 超过 4000 tokens 触发摘要
            keep=("messages", 20),         # 摘要后保留最近 20 条
        ),
    ],
)
```

### 自定义摘要 prompt

```python
middleware = SummarizationMiddleware(
    model=summary_llm,
    trigger=("tokens", 4000),
    keep=("messages", 20),
    summary_prompt=(
        "请用中文总结以下对话的关键信息、用户偏好和决策。"
        "保留事实细节、忽略寒暄。\n\n{messages}"
    ),
)
```

### 精确 token 计数

```python
from langchain_deepseek import ChatDeepSeek

llm = ChatDeepSeek(model="deepseek-v4-flash")

middleware = SummarizationMiddleware(
    model=summary_llm,
    trigger=("tokens", 4000),
    token_counter=llm.get_num_tokens_from_messages,  # 精确计数
)
```

## 五、注意事项

1. **model 参数必须是实例，不是字符串**
   ```python
   # ❌ 错误
   SummarizationMiddleware(model="deepseek-v4-flash")
   # ✅ 正确
   SummarizationMiddleware(model=ChatDeepSeek(model="deepseek-v4-flash"))
   ```

2. **只压缩文本消息**：不压缩图片、音频、视频内容

3. **额外 LLM 开销**：每次摘要会调用一次 LLM，成本约 = 触发频率 × 当前历史长度。`trigger=("tokens", 4000)` 对一个 128K 窗口的模型触发频率约为每 30 轮一次

4. **不能与流式输出配合使用时的中间步骤观测**（stream_events）——摘要会改变消息流结构

5. **注意 trigger 阈值**：设得太低（< 1000）会让 Agent 频繁摘要浪费钱；设得太高（> 模型窗口 0.9）可能来不及

6. **keep 值不要太小**：保留消息太少（< 5）会丢失当前对话上下文连贯性

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/ContextEditingMiddleware]] — 另一种上下文压缩策略（清除旧工具输出）
- [[langchain/memory/上下文压缩]] — 传统 LCEL 时代的压缩方案
- [[langchain/agents/Agent循环实现]] — Agent 在中间件中如何被调用
