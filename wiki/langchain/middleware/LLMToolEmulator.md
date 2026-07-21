---
tags: [langchain, middleware, testing, tool-emulator]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#llm-tool-emulator
langchain_version: ">=1.0"
---

# LLMToolEmulator（LLM 工具模拟器中间件）

## 一、定义与作用

`LLMToolEmulator` 在**测试环境**用 **LLM 模拟工具的返回结果**——Agent 调用工具时，实际上不会真正执行工具，而是 LLM 生成一个"看起来合理"的响应。

它解决的是测试中的两个痛点：
1. **真实工具调用成本高**（付费 API、消耗资源）
2. **真实工具调用结果不确定**（每次返回不同，难以回归测试）

**核心价值**：让 Agent 的开发/测试**不依赖真实工具**，快速验证 Agent 决策逻辑。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 单元测试 | 测试 Agent 决策逻辑，不希望真实调搜索 API |
| 离线开发 | 没有网络/没有 API Key 时仍能调试 Agent |
| 演示 | 给客户演示 Agent 流程时避免真实操作 |
| 训练数据生成 | 用模拟工具生成"假设场景"下的对话数据 |

**不适合**：
- 生产环境（必须用真实工具）
- 集成测试（需要测试真实工具集成）

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `tools` | `list[str \| BaseTool]` | `None` | 要模拟的工具名或实例列表（`None` 表示全部模拟；`[]` 表示不模拟任何工具） |
| `model` | `string \| BaseChatModel` | Agent 模型 | 用于生成模拟响应的模型 |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[web_search, send_email, calculator],
    middleware=[
        LLMToolEmulator(),  # 所有工具都被模拟
    ],
)

# 测试时调用
result = agent.invoke({"messages": [HumanMessage("查北京天气")]})
# 实际上 web_search 没有真正调用，LLM 生成了一个"看起来合理"的天气响应
```

### 只模拟部分工具

```python
LLMToolEmulator(
    tools=["web_search", "database_query"],  # 只模拟这两个，send_email 真实执行
)
```

### 用便宜的模型模拟

```python
from langchain_deepseek import ChatDeepSeek

LLMToolEmulator(
    model=ChatDeepSeek(model="deepseek-v4-flash", temperature=1.0),  # 便宜模型
)
```

## 五、注意事项

1. **仅限测试**：**绝对不要**在生产环境用 `LLMToolEmulator`，否则 Agent 永远得不到真实数据

2. **结果不可预测**：每次调用 LLM 模拟的结果可能不同。如要稳定的测试结果：
   - 用 `temperature=0` 的模型
   - 或改用 mock 库（`unittest.mock`）

3. **模拟的真实性取决于模型能力**：用 GPT-5.5 模拟比用 Haiku 模拟更像真实结果

4. **适合"端到端流程测试"**：测试 Agent 决策路径（"调了几个工具"），不测试工具行为本身

5. **不能完全替代集成测试**：真实工具可能有特殊行为（异常、超时、格式错误），模拟器不会触发

6. **与 `wrap_tool_call` 装饰器配合**：可以写一个简单的"返回固定结果"版本：

```python
from langchain.agents.middleware import wrap_tool_call

@wrap_tool_call
def mock_tool(request, handler):
    """返回固定响应"""
    return ToolMessage(content="模拟响应", tool_call_id=request.tool_call["id"])

# 比 LLMToolEmulator 更可控
```

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/LLMToolSelectorMiddleware]] — 工具筛选
- [[langchain/middleware/自定义中间件开发]] — 写测试中间件
