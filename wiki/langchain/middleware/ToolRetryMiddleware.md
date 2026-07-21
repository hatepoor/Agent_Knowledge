---
tags: [langchain, middleware, reliability, retry, exponential-backoff]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#tool-retry
langchain_version: ">=1.0"
---

# ToolRetryMiddleware（工具重试中间件）

## 一、定义与作用

`ToolRetryMiddleware` 在**工具调用失败**时，以**指数退避**（exponential backoff）策略自动重试。默认最多重试 2 次（加上首次共 3 次尝试）。

它解决的是工具的"暂时性失败"——网络抖动、第三方 API 限流、数据库瞬时不可用等。这些问题重试 1-2 次往往就恢复了。

**核心价值**：把"工具调用失败"从"Agent 终止"降级为"自动重试成功"，业务稳定性大幅提升。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 第三方 API 限流 | Tavily 返回 429，重试 5 秒后恢复 |
| 数据库瞬时不可用 | Redis 偶尔 timeout，重试一次就成功 |
| 网络抖动 | 偶尔的连接错误 |
| 瞬时资源不足 | 计算服务短暂繁忙 |

**不适合重试**：
- 参数错误（重试也会失败）
- 权限错误（不会因为重试变好）
- 业务逻辑错误（如"用户余额不足"）

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_retries` | `number` | `2` | 初次调用后的最大重试次数（默认共 3 次尝试） |
| `tools` | `list[BaseTool \| str]` | `None` | 仅对这些工具应用重试（`None` 表示全部） |
| `retry_on` | `tuple[type[Exception], ...] \| callable` | `(Exception,)` | 重试的异常类型元组或判断函数 |
| `on_failure` | `string \| callable` | `"return_message"` | `'return_message'`（返回错误消息）/ `'raise'`（重新抛出）/ 自定义 callable |
| `backoff_factor` | `number` | `2.0` | 指数退避乘数（`0.0` 表示恒定延迟） |
| `initial_delay` | `number` | `1.0` | 首次重试前初始延迟（秒） |
| `max_delay` | `number` | `60.0` | 重试间最大延迟（秒） |
| `jitter` | `boolean` | `true` | 是否添加随机抖动（±25%） |

### 指数退避时序

```
第 1 次尝试: 立即
[失败] 等待 initial_delay * (backoff_factor^0) = 1.0s + 抖动
第 2 次尝试: ~1.0s
[失败] 等待 initial_delay * (backoff_factor^1) = 2.0s + 抖动
第 3 次尝试: ~3.0s
[失败] 等待 initial_delay * (backoff_factor^2) = 4.0s + 抖动
[达到 max_retries] → on_failure 行为
```

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[web_search, read_file, send_email],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,            # 初次 + 3 次重试 = 4 次尝试
            initial_delay=1.0,
            backoff_factor=2.0,
        ),
    ],
)
```

### 只对特定工具重试

```python
ToolRetryMiddleware(
    tools=["web_search", "database_query"],  # 只对这两个工具重试
    max_retries=3,
)
```

### 自定义重试异常类型

```python
import requests

ToolRetryMiddleware(
    retry_on=(
        requests.exceptions.Timeout,
        requests.exceptions.ConnectionError,
    ),
    # 其他异常（参数错误、权限错误）不重试
    max_retries=3,
)
```

### 自定义失败处理

```python
ToolRetryMiddleware(
    max_retries=2,
    on_failure="raise",  # 重试耗尽后重新抛异常（而非返回错误消息）
)
```

```python
# 自定义 callable 失败处理
def custom_failure(tool_call, exception):
    return f"工具 {tool_call['name']} 调用失败：{str(exception)[:100]}"

ToolRetryMiddleware(
    max_retries=2,
    on_failure=custom_failure,
)
```

## 五、注意事项

1. **on_failure 默认行为是 "return_message"**：Agent 看到错误消息后**自己决定**下一步。这通常是正确的——Agent 可能改用其他工具。但**不保证** Agent 一定能恢复

2. **不重试业务异常**：默认 `retry_on=(Exception,)` 会重试**所有**异常。生产环境应指定具体异常类型，避免对"参数错误"等永久性错误做无谓重试

3. **退避因子不要设太大**：`backoff_factor=10` 会让第 3 次重试等待 100 秒，整个 Agent 卡住。一般 1.5-3.0 即可

4. **max_delay 是上限**：实际等待时间不会超过 `max_delay` 秒

5. **jitter 必须开启**：避免"惊群效应"——多个 Agent 同时重试同一服务

6. **与 ModelRetryMiddleware 是不同东西**：
   - `ToolRetryMiddleware`：包在工具调用层（处理工具失败）
   - `ModelRetryMiddleware`：包在模型调用层（处理 LLM API 失败）

7. **调试重试**：开启日志中间件，看清楚什么时候触发了重试

## 六、相关页面

- [[langchain/middleware/ModelRetryMiddleware]] — 模型重试
- [[langchain/middleware/ModelFallbackMiddleware]] — 模型回退
- [[langchain/middleware/中间件概述]] — 总览
