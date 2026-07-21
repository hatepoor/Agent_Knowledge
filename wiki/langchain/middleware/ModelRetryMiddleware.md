---
tags: [langchain, middleware, reliability, retry, model]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#model-retry
langchain_version: ">=1.0"
---

# ModelRetryMiddleware（模型重试中间件）

## 一、定义与作用

`ModelRetryMiddleware` 在**模型调用失败**时（网络错误、速率限制、临时服务异常），以**指数退避**策略自动重试。

它针对的是 **LLM API 层的瞬时失败**——比如 OpenAI 偶尔返回 500、TLS 握手超时、负载均衡节点故障等。

**核心价值**：让 Agent 对"LLM 供应商小毛病"免疫。短暂网络问题不再让用户看到错误。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| OpenAI/Claude 限流 | 429 错误，重试几秒后通过 |
| 网络抖动 | TLS 握手偶尔失败 |
| 供应商负载过高 | 5xx 错误，重试后正常 |
| 跨地域部署 | 节点偶发不可用 |

**不适合重试**：
- 鉴权失败（401）— 不会因为重试通过
- 输入过长（400）— 重试也还是长
- 内容审核拒绝（403）— 永久性拒绝

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_retries` | `number` | `2` | 最大重试次数（不含首次） |
| `retry_on` | `tuple \| callable` | `(Exception,)` | 触发重试的异常类型或判断函数 |
| `on_failure` | `string \| callable` | `"continue"` | `'continue'`（返回含错误的 AIMessage）/ `'error'`（抛出）/ 自定义 callable |
| `backoff_factor` | `number` | `2.0` | 退避乘数 |
| `initial_delay` | `number` | `1.0` | 初始延迟（秒） |
| `max_delay` | `number` | `60.0` | 最大延迟（秒） |
| `jitter` | `boolean` | `true` | 添加随机抖动 |

### on_failure 行为差异（与 ToolRetryMiddleware 不同）

| 模式 | 行为 |
|------|------|
| `"continue"` | 返回一个**含错误信息的 `AIMessage`**，Agent 把错误当作文本处理 |
| `"error"` | 重新抛出异常，Agent 终止 |

> **注意**：模型重试的默认行为是 `"continue"`（不是 `"return_message"`），因为模型调用是同步的，错误可以包装为 AIMessage。

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[...],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            initial_delay=1.0,
            backoff_factor=2.0,
        ),
    ],
)
```

### 自定义重试异常

```python
from openai import RateLimitError, APIConnectionError

ModelRetryMiddleware(
    retry_on=(RateLimitError, APIConnectionError),
    max_retries=5,
    initial_delay=2.0,
)
```

### 错误传播模式

```python
ModelRetryMiddleware(
    max_retries=2,
    on_failure="continue",   # 把错误当 AIMessage 返回（默认）
)

# 或严格模式
ModelRetryMiddleware(
    max_retries=2,
    on_failure="error",      # 重试失败后抛异常
)
```

## 五、注意事项

1. **与 ModelFallbackMiddleware 配合**：
   - `ModelRetryMiddleware` 处理"瞬时失败"（重试就好）
   - `ModelFallbackMiddleware` 处理"长期失败"（换模型）
   - 通常先重试再回退

2. **不要重试鉴权错误**：`retry_on=(Exception,)` 会重试所有异常，包括 401、403 等**永久性**错误。建议明确指定异常类型

3. **重试会增加延迟**：用户能看到的就是"卡了几秒"。`max_retries=3` 最坏情况要等 `1+2+4 = 7` 秒

4. **某些 LLM 库自带重试**：如 `langchain-openai` 内部已经有重试，再叠加 `ModelRetryMiddleware` 会双重重试，行为可能不符合预期。`max_retries=0` 关闭库内置重试

5. **on_failure="continue" 的副作用**：Agent 把"调用失败"当作文本，**可能继续思考错误的方案**。需要配 `ToolCallLimitMiddleware` 等保护机制

6. **与流式输出配合**：重试会中断流（用户看到消息发了一半又从头开始）。如要避免，需要在 `stream` 层处理

## 六、相关页面

- [[langchain/middleware/ToolRetryMiddleware]] — 工具重试（参数结构类似）
- [[langchain/middleware/ModelFallbackMiddleware]] — 模型回退
- [[langchain/middleware/中间件概述]] — 总览
