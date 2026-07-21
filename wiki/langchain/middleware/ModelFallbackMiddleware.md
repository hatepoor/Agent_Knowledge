---
tags: [langchain, middleware, reliability, fallback]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#model-fallback
langchain_version: ">=1.0"
---

# ModelFallbackMiddleware（模型回退中间件）

## 一、定义与作用

`ModelFallbackMiddleware` 在**主模型调用失败**时，自动按顺序尝试**备用模型**，直到有一个成功为止。

它解决了"主模型供应商挂掉"或"主模型触发速率限制"的场景——业务不会因为单一供应商故障而停摆。

**核心价值**：提升系统**可用性**（availability）。一个配置得当的 Agent 可以在主模型宕机时透明地切换到备用模型，用户毫无感知。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 主备供应商容灾 | 主用 OpenAI GPT-5.5，挂了自动切 Anthropic Claude |
| 速率限制 | 触发了 OpenAI 的 RPM 上限，自动切到 DeepSeek |
| 区域可用性 | 国内服务优先用 DeepSeek，国际服务用 Claude |
| 成本优化 | 平时用便宜模型（Haiku），失败切旗舰模型 |

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `first_model` | `string \| BaseChatModel` | **必填**，第一个备用模型（不是主模型） |
| `*additional_models` | `string \| BaseChatModel` | 后续备用模型（按顺序尝试，可传多个） |

> **重要**：`first_model` 不是"主模型"而是"**第一个备用**"——主模型仍然是 `create_agent(model=...)` 中指定的那个。

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="deepseek-v4-flash",  # 主模型
    tools=[...],
    middleware=[
        ModelFallbackMiddleware(
            first_model="claude-sonnet-4-6",    # 第一个备用
            "gpt-5.5",                          # 第二个备用
            "claude-haiku-4-5",                 # 第三个备用
        ),
    ],
)
```

### 使用模型实例

```python
from langchain_deepseek import ChatDeepSeek
from langchain_anthropic import ChatAnthropic

# 主模型
main_model = ChatDeepSeek(model="deepseek-v4-flash")

# 备用模型
fallback_1 = ChatAnthropic(model="claude-sonnet-4-6")
fallback_2 = ChatAnthropic(model="claude-haiku-4-5")

agent = create_agent(
    model=main_model,
    tools=[...],
    middleware=[
        ModelFallbackMiddleware(
            first_model=fallback_1,
            fallback_2,
        ),
    ],
)
```

### 配合重试中间件

```python
from langchain.agents.middleware import ModelFallbackMiddleware, ModelRetryMiddleware

middleware = [
    # 先重试 3 次
    ModelRetryMiddleware(max_retries=3, initial_delay=1.0),
    # 重试都失败再切换模型
    ModelFallbackMiddleware(
        first_model="claude-sonnet-4-6",
        "deepseek-v4-pro",
    ),
]
```

## 五、注意事项

1. **与 `ModelRetryMiddleware` 的配合**：
   - `ModelRetryMiddleware`：同一模型失败时**重试**（如网络抖动）
   - `ModelFallbackMiddleware`：所有重试都失败时**换模型**（如供应商宕机）
   - 通常先重试，再回退

2. **不传递成本到主模型**：如果主模型是按 token 计费的，回退到更贵的模型可能产生意外费用。建议主备都是相似价位的模型

3. **能力差异问题**：备用模型能力可能比主模型弱。配置时要考虑"降级体验"是否可接受
   - 例：平时用 GPT-5.5 写代码，回退到 Haiku 可能写不出高质量代码

4. **不处理工具调用格式差异**：不同模型的 tool_call 格式可能不同。如果一个工具调用在 Claude 中成功但 DeepSeek 解析失败，回退不能解决

5. **不处理上下文长度差异**：如果主模型支持 128K 上下文，备用模型只支持 8K，会触发长度错误

6. **最佳实践**：
   - 选择**能力相当**的备用模型
   - 备用模型建议**多供应商**（不要 OpenAI 挂了切 Anthropic，要分散风险）
   - 监控回退发生频率——频繁回退说明主模型服务不稳定

## 六、相关页面

- [[langchain/middleware/ModelRetryMiddleware]] — 同模型重试
- [[langchain/middleware/ToolRetryMiddleware]] — 工具重试
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/最佳实践]] — 中间件组合策略
