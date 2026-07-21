---
tags: [langchain, middleware, tool-selection, token-saving]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#llm-tool-selector
langchain_version: ">=1.0"
---

# LLMToolSelectorMiddleware（LLM 工具选择器中间件）

## 一、定义与作用

`LLMToolSelectorMiddleware` 在**主模型调用前**，先用**一个小模型**（通常是便宜模型）分析用户问题，**智能筛选出相关的工具**喂给主模型。

它解决了一个关键问题：**当 Agent 有几十上百个工具时，每个 tool 的 JSON schema 都喂给主模型会消耗大量 token**。

**核心价值**：让 Agent 能管理大量工具，同时保持主模型的高准确率。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 100+ 工具的复杂 Agent | 工具市场里挑出 5 个最相关的 |
| 多租户 SaaS | 不同用户看到不同工具集（筛选后再喂主模型） |
| 动态工具加载 | 不同任务加载不同工具 |
| 权限控制 | 把用户无权调用的工具排除掉 |

**不适合**：
- 工具数量 < 10（筛选开销反而浪费）
- 工具之间语义差别不大（筛选可能误判）

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `string \| BaseChatModel` | 用于工具选择的模型（默认用 Agent 的主模型） |
| `system_prompt` | `string` | 选择模型的指令 |
| `max_tools` | `number` | 选择的最大工具数（超出仅取前 N 个） |
| `always_include` | `list[string]` | 始终包含的工具名（不计入 `max_tools`） |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

# 假设有 100 个工具
all_tools = [tool_1, tool_2, ..., tool_100]

agent = create_agent(
    model="deepseek-v4-flash",  # 主模型
    tools=all_tools,
    middleware=[
        LLMToolSelectorMiddleware(
            model="deepseek-v4-flash",  # 用便宜模型筛选
            max_tools=5,                # 最多选 5 个
            always_include=["search", "calculator"],  # 这两个始终包含
        ),
    ],
)
```

执行流程：
```
1. 用户问："今天北京天气？"
2. 筛选模型分析 → 选出 [search, weather_api, get_date] 3 个工具
3. 主模型看到 3 个工具（不是 100 个）→ 准确选择 weather_api
4. 节省 95% 的工具 schema token
```

### 自定义选择指令

```python
LLMToolSelectorMiddleware(
    model="deepseek-v4-flash",
    max_tools=5,
    system_prompt=(
        "根据用户问题，从候选工具中选择最相关的 5 个。\n"
        "规则：\n"
        "- 必须包含查询类工具\n"
        "- 必须包含执行类工具\n"
        "- 不要选超过 2 个相似功能的工具"
    ),
)
```

## 五、注意事项

1. **额外 LLM 调用成本**：每次用户输入都多一次"筛选 LLM 调用"。但这个调用非常短（输入只有工具列表和用户问题），通常 < 500 tokens，比直接喂 100 个工具的 schema 便宜得多

2. **筛选模型的选择**：
   - 用 `gpt-4o-mini` / `claude-haiku` 等便宜模型
   - **不建议**用主模型（重复劳动，浪费钱）

3. **`max_tools` 的选择**：
   - 太低（< 3）：可能漏掉正确工具
   - 太高（> 10）：失去筛选意义
   - 经验值：5-8

4. **`always_include` 的使用**：核心工具（如搜索、计算器）始终包含，避免筛选模型判断失误

5. **筛选模型的能力**：筛选模型要能"理解"每个工具的描述。如果工具描述写得不清楚，筛选会失败

6. **延迟增加**：每次请求多 1 次 LLM 调用，延迟增加 200-500ms

7. **首次调用延迟更明显**：warm-up 后会好

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/ProviderToolSearchMiddleware]] — 更激进的工具发现策略
- [[langchain/middleware/LLMToolEmulator]] — 测试用工具模拟
- [[langchain/tools/工具来源与集成]] — 工具管理
