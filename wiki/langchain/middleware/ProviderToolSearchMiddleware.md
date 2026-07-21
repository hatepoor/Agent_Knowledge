---
tags: [langchain, middleware, tool-search, provider, anthropic, openai]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#provider-tool-search
langchain_version: ">=1.0"
---

# ProviderToolSearchMiddleware（Provider 工具搜索中间件）

## 一、定义与作用

`ProviderToolSearchMiddleware` 把工具**延迟到模型服务端按需发现**——不是把所有工具 schema 一次性塞给主模型，而是让模型在需要时**主动搜索**工具。

它比 `LLMToolSelectorMiddleware` 更激进：
- 后者：先用小模型筛选 → 把筛选结果喂给主模型（仍然是"全部 schema 一次性"）
- 前者：主模型看到**工具目录**，自己按需请求某个工具的完整 schema

**核心价值**：让 Agent 能管理**海量工具**（数千个），同时不浪费主模型的 token。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 工具市场平台 | 平台有 1000+ 工具，Agent 按需加载 |
| 企业内部工具库 | 几千个内部 API，按需调用 |
| 多团队共用 Agent | 各团队注册自己的工具，主 Agent 按需发现 |
| 工具频繁更新 | 工具集动态变化，避免每次启动重新打包 schema |

**前置条件**：
- 使用 **Anthropic**（Claude Sonnet 4+ / Opus 4+ / Haiku 4.5+）或 **OpenAI**（gpt-5.5+）等支持的模型
- 其他模型会抛 `ValueError`

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `searchable_tools` | `list[str \| BaseTool]` | 延迟到提供方工具搜索的工具（按名称或实例）。使用 `extras={"defer_loading": True}` 构建的工具自动延迟 |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ProviderToolSearchMiddleware

# 创建 100 个工具
all_tools = [tool_1, tool_2, ..., tool_100]

agent = create_agent(
    model="claude-sonnet-4-6",  # 必须用支持的模型
    tools=all_tools,
    middleware=[
        ProviderToolSearchMiddleware(
            searchable_tools=[t.name for t in all_tools],  # 100 个工具都延迟
        ),
    ],
)
```

执行流程：
```
1. 用户问："今天北京天气？"
2. 主模型收到一个"工具目录"（不包含完整 schema，只含名称+简介）
3. 主模型："我需要 weather_api 这个工具" → 请求完整 schema
4. 完整 schema 加载进来 → 主模型调用
```

### 工具声明延迟加载

```python
from langchain_core.tools import tool

@tool(extras={"defer_loading": True})  # 自动延迟，无需在中间件列出
def web_search(query: str) -> str:
    """Search the web for information."""
    return ...
```

```python
# 中间件自动识别所有 defer_loading=True 的工具
ProviderToolSearchMiddleware()
# 不需要传 searchable_tools
```

## 五、注意事项

1. **必须用支持的模型**：
   - Anthropic：Claude Sonnet 4+, Opus 4+, Haiku 4.5+
   - OpenAI：gpt-5.5+
   - 其他模型（DeepSeek、Qwen、Llama）暂不支持，会抛 `ValueError`

2. **第一次调用有额外延迟**：模型需要先"搜索"工具，再获取 schema，延迟可能增加 500ms-2s

3. **工具描述要写好**：模型只看到"简介"决定要不要搜索。简介写不清楚，模型找不到正确工具

4. **不适合频繁使用的工具**：如果某个工具每次都用到（如 `search`），延迟加载反而拖慢。建议核心工具**不**延迟

5. **token 成本 vs 延迟的权衡**：
   - 全部 schema 喂给主模型：token 成本高、延迟低
   - 延迟加载：token 成本低、首次调用延迟高

6. **与 `LLMToolSelectorMiddleware` 是不同方案**：
   - `LLMToolSelectorMiddleware`：用小模型筛（仍然一次性喂）
   - `ProviderToolSearchMiddleware`：主模型自己搜（按需喂）

## 六、相关页面

- [[langchain/middleware/LLMToolSelectorMiddleware]] — 工具筛选（用小模型）
- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/tools/工具来源与集成]] — 工具管理
