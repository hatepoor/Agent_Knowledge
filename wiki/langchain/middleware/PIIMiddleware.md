---
tags: [langchain, middleware, safety, pii, privacy]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#pii-detection
langchain_version: ">=1.0"
---

# PIIMiddleware（个人身份信息脱敏中间件）

## 一、定义与作用

`PIIMiddleware` 自动**检测并处理对话中出现的个人身份信息**（PII, Personally Identifiable Information），如邮箱、手机号、IP 地址、信用卡号等。处理方式可以是**脱敏、遮蔽、哈希或阻断**。

它解决了两个问题：
1. **合规性**：GDPR、CCPA 等法规要求不能把用户 PII 发送给第三方 LLM
2. **安全性**：避免敏感数据意外写入日志或检索系统

**核心价值**：让 Agent 自动遵守隐私法规，开发者不用在每个工具调用前手动检查。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 客服 Agent | 用户输入含手机号，自动脱敏后发给 LLM |
| 医疗 Agent | 患者姓名/病历号被遮蔽后再处理 |
| 跨境业务 | 用户 IP 地址脱敏，符合 GDPR |
| 公开部署的 Agent | 防止用户在输入中"投毒"信用卡号 |

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `pii_type` | `string` | **必填** | 内置类型（`email`、`credit_card`、`ip`、`mac_address`、`url`）或自定义类型名 |
| `strategy` | `string` | `"redact"` | `'block'` / `'redact'` / `'mask'` / `'hash'` |
| `detector` | `function \| regex` | — | 自定义检测器（regex 字符串、编译后的 regex 或函数） |
| `apply_to_input` | `boolean` | `True` | 在模型调用前检查用户消息 |
| `apply_to_output` | `boolean` | `False` | 在模型调用后检查 AI 消息 |
| `apply_to_tool_results` | `boolean` | `False` | 检查工具结果消息 |

### 4 种处理策略

| 策略 | 效果 | 示例 |
|------|------|------|
| `block` | 抛异常，Agent 终止 | `"Found PII: email"` |
| `redact` | 替换为 `[REDACTED_{PII_TYPE}]` | `alice@example.com` → `[REDACTED_EMAIL]` |
| `mask` | 部分遮蔽 | `13800138000` → `138****8000` |
| `hash` | 确定性哈希（同样 PII 同样哈希） | `alice@example.com` → `a1b2c3d4...` |

## 四、使用方法

### 基础用法：邮箱脱敏

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[...],
    middleware=[
        PIIMiddleware(
            pii_type="email",
            strategy="redact",
            apply_to_input=True,    # 检查用户消息
        ),
    ],
)

# 用户输入："我的邮箱是 alice@example.com"
# → 模型看到的是："我的邮箱是 [REDACTED_EMAIL]"
```

### 多种 PII 同时脱敏

```python
middleware = [
    PIIMiddleware(pii_type="email", strategy="redact"),
    PIIMiddleware(pii_type="credit_card", strategy="mask"),
    PIIMiddleware(pii_type="ip", strategy="redact", apply_to_tool_results=True),
]
```

### 自定义 PII 类型（如中国手机号）

```python
import re

PIIMiddleware(
    pii_type="cn_phone",                            # 自定义类型名
    detector=r"1[3-9]\d{9}",                        # regex 字符串
    strategy="mask",
)
```

```python
# 或自定义函数
def phone_detector(content: str):
    matches = []
    for m in re.finditer(r"1[3-9]\d{9}", content):
        matches.append({"text": m.group(), "start": m.start(), "end": m.end()})
    return matches

PIIMiddleware(
    pii_type="cn_phone",
    detector=phone_detector,
    strategy="mask",
)
```

### 阻断模式

```python
PIIMiddleware(
    pii_type="credit_card",
    strategy="block",   # 一旦发现含信用卡号，整个 Agent 抛出异常
)
```

## 五、注意事项

1. **PII 检测基于 regex**：不是 AI 检测。对于复杂的 PII（如人名、地址）识别能力有限

2. **不同语言 PII 格式不同**：
   - 美国手机号：`\d{3}-\d{3}-\d{4}`
   - 中国手机号：`1[3-9]\d{9}`
   - 美国 SSN：`\d{3}-\d{2}-\d{4}`
   - 通用邮箱：`[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`

3. **apply_to_output 默认 False**：LLM 输出的内容不检查。开启会**显著降低性能**（每次模型响应都要扫描）

4. **`apply_to_tool_results=True` 可能误伤**：工具返回结果中如果含 IP、邮箱等正常信息（如查询 IP 归属地的工具），会被脱敏

5. **脱敏不是加密**：`strategy="hash"` 只是文本哈希，反向查表仍可能恢复。**不能**用 PIIMiddleware 替代真正的数据加密

6. **建议组合**：
   - 输入用 `redact`（保护用户隐私）
   - 工具结果用 `redact` 或不开启
   - 信用卡用 `block`（最严格）
   - 手机号用 `mask`（部分信息可读）

7. **langchain >= 1.3.2 才有流式输出脱敏**：老版本只在非流式输出上脱敏

## 六、相关页面

- [[langchain/middleware/HumanInTheLoopMiddleware]] — 关键操作前人工审批
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/最佳实践]] — 中间件组合策略
