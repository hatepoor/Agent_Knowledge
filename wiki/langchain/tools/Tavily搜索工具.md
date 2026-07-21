---
tags: [langchain, tools, tavily, search, integration]
created: 2026-07-07
updated: 2026-07-07
type: concept
---

# Tavily 搜索工具

## 概述

Tavily Search API 是**专为 LLM 和 AI 应用设计**的搜索引擎。LangChain 目前提供两种集成方式：新版 `langchain-tavily` 包（推荐）和旧版 `langchain-community` 工具。

[Tavily 官方文档](https://docs.tavily.com/documentation/api-reference/endpoint/search)

---

## 一、Tavily 的几种调用方式

### 方式 1：新版 `langchain-tavily` 包（推荐）

这是官方最新推荐的集成方式，包名独立，功能更完整：

```bash
pip install -qU langchain-tavily
```

```python
from langchain_tavily import TavilySearch
import os

os.environ["TAVILY_API_KEY"] = "your_key"

tool = TavilySearch(
    max_results=5,              # 结果数量
    topic="general",            # 搜索类别："general" / "news" / "finance"
    include_answer=False,       # 是否包含直接答案
    include_raw_content=False,  # 是否包含清洗后的 HTML 全文
    include_images=False,       # 是否包含图片
    include_image_descriptions=False,  # 是否包含图片描述
    search_depth="basic",       # "basic" / "advanced"
    time_range=None,            # "day" / "week" / "month" / "year"
    include_domains=None,       # 白名单
    exclude_domains=None,       # 黑名单
)

# 直接调用
result = tool.invoke({"query": "2025年诺贝尔经济学奖获得者"})

# ToolCall 模式
tool_call = {"args": {"query": "euro 2024 host nation"}, "id": "1", "name": "tavily", "type": "tool_call"}
tool_msg = tool.invoke(tool_call)
```

> **注意**：`include_answer` 和 `include_raw_content` 只能在实例化时设置，不能在 `invoke()` 时覆盖——这是 Tavily 为防止意外上下文窗口溢出做的限制。

### 方式 2：旧版 `langchain-community` 工具

```bash
pip install -qU "langchain-community>=0.2.11" tavily-python
```

```python
from langchain_community.tools import TavilySearchResults

tool = TavilySearchResults(
    max_results=5,
    search_depth="advanced",
    include_answer=True,
    include_raw_content=True,
    include_domains=None,
    exclude_domains=None,
)

result = tool.invoke({"query": "2025年AI Agent框架对比"})
```

### 方式 3：原生 Python SDK（轻量方案）

不依赖 LangChain，直接与 Tavily API 交互：

```python
from tavily import TavilyClient

client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))

response = client.search(
    query="AI Agent 框架对比",
    search_depth="advanced",
    topic="general",
    max_results=5,
    include_answer=True,
    include_raw_content=False,
    include_images=False,
    time_range="month",
)

print(response["answer"])         # 预生成摘要
for r in response["results"]:     # 结果列表
    print(r["title"], r["url"], r["content"][:200])
```

### 方式 4：REST API 直接调用

最低层的方式，直接调用 Tavily 的 HTTP 端点：

```python
import httpx

response = httpx.post(
    "https://api.tavily.com/search",
    json={
        "api_key": os.getenv("TAVILY_API_KEY"),
        "query": "2025年AI进展",
        "search_depth": "basic",
        "max_results": 3,
        "include_answer": False,
    },
)
data = response.json()
```

### 方式 5：Agent 内动态参数调用

Agent 可以根据用户问题**动态设置 Tavily 参数**。例如只搜索特定域名：

```python
from langchain_tavily import TavilySearch
from langchain.agents import create_agent

tool = TavilySearch(max_results=5, topic="general")

agent = create_agent(model="openai:gpt-5.5", tools=[tool])

# Agent 会根据 system prompt 动态设置 include_domains 等参数
stream = agent.stream_events(
    {"messages": "Euro 2024在哪个国家举办的？只搜索wikipedia。"},
    version="v3",
)
# 实际调用时 Agent 会填充：{"query": "Euro 2024 host nation", "include_domains": ["wikipedia.org"]}
```

### 方式对比

| 方式 | 依赖 | 灵活性 | 适用场景 |
|------|------|--------|----------|
| `langchain-tavily`（新版） | `langchain-tavily` | 高 | Agent 集成，推荐方式 |
| `TavilySearchResults`（旧版） | `langchain-community` | 中 | 旧项目兼容 |
| `TavilyClient` SDK | `tavily-python` | 最高 | 无需 Agent 的轻量场景 |
| REST API 直接调用 | `httpx` / `requests` | 最高 | 非 Python 环境或自定义封装 |
| Agent 动态参数 | `langchain-tavily` | 智能 | Agent 自主决定搜索策略 |

---

## 二、Token 消耗问题与优化策略

Tavily 返回的搜索结果会被送入模型上下文，其结果数量和内容长度直接影响 token 消耗。以下是系统性的优化策略。

### 策略 1：缩减结果数量（效果最直接）

```python
# ❌ token 高
tool = TavilySearch(max_results=10, search_depth="advanced")

# ✅ token 低
tool = TavilySearch(max_results=3, search_depth="basic")
```

| max_results | 估算 token 消耗 | 场景 |
|-------------|----------------|------|
| 3 | ~500-800 tokens | 快速问答 |
| 5 | ~800-1500 tokens | 通用场景（默认值） |
| 8-10 | ~1500-3000 tokens | 深度调研 |

### 策略 2：关闭 `include_raw_content`（最关键的优化）

`raw_content` 返回的是经过清洗的完整 HTML 内容，**这是最大的 token 消耗源**。

```python
# ❌ token 极高 — 每个结果可能带入数千 token 的 HTML 正文
tool = TavilySearch(max_results=5, include_raw_content=True)

# ✅ token 低 — 只返回简短的内容摘要（约 100-300 tokens/条）
tool = TavilySearch(max_results=5, include_raw_content=False)
```

> **经验值**：开启 `include_raw_content` 后，单次搜索结果的 token 消耗可能是关闭时的 **5-10 倍**。非必须场景建议保持关闭。

### 策略 3：使用 `search_depth="basic"`

```python
# ❌ advanced 模式会深入抓取更多细节
tool = TavilySearch(search_depth="advanced")

# ✅ basic 模式只返回摘要级内容
tool = TavilySearch(search_depth="basic")
```

`basic` vs `advanced` 的 token 差异通常在 **2-3 倍**之间。

### 策略 4：关闭 `include_answer`

```python
# ❌ answer 字段本身是 LLM 生成的摘要，额外消耗 token
tool = TavilySearch(include_answer=True)

# ✅ 只返回原始结果列表，由主 Agent 自行综合分析
tool = TavilySearch(include_answer=False)
```

如果 Agent 本身具备较强的总结能力，关闭 `include_answer` 是安全的。

### 策略 5：使用 `topic` 限定搜索类别

新版 `TavilySearch` 支持 `topic` 参数，缩小搜索范围可减少无关结果的 token 浪费：

```python
# general：搜索所有内容（token 最广）
tool = TavilySearch(topic="general")

# news：只搜索新闻来源，结果更聚焦
tool = TavilySearch(topic="news")

# finance：只搜索财经来源
tool = TavilySearch(topic="finance")
```

### 策略 6：使用 `time_range` 限定时间窗口

搭配 `time_range` 避免搜索到大量过期内容：

```python
tool = TavilySearch(time_range="week")  # 只看本周
tool = TavilySearch(time_range="month") # 只看本月
tool = TavilySearch(time_range="year")  # 只看本年
```

### 策略 7：域名限定

用白名单精确控制来源，从源头减少低质量结果：

```python
# 只从高质量源搜索，结果更少但更精准
tool = TavilySearch(
    include_domains=["arxiv.org", "wikipedia.org", "github.com"],
    max_results=3,
)
```

### 策略 8：缓存重复查询

相同或相似的查询在短时间内重复执行时，缓存可完全消除 token 消耗：

```python
from functools import lru_cache

@lru_cache(max_size=128)
def cached_search(query: str) -> dict:
    tool = TavilySearch(max_results=3, search_depth="basic")
    return tool.invoke({"query": query})

result = cached_search("2025年AI Agent 框架对比")
```

也可以使用磁盘缓存（如 `diskcache`）在会话间持久化。

### 策略 9：Agent 提示词优化（减少不必要调用）

在 system prompt 中明确限定 Tavily 的调用条件，让模型只在该搜索时搜索：

```python
SYSTEM_PROMPT = """你是信息助手。

## Tavily 搜索使用规则
- 仅当问题涉及最新信息、实时数据或超出你知识范围时，才调用 tavily
- 常识性、定义性问题直接回答，不要调用搜索
- 每次搜索前先判断是否已有足够信息
- 搜索时优先设置 include_domains 以聚焦高质量源
"""
```

### 策略 10：分层搜索策略

先快速搜索获取概览，只有需要深度信息时才进行深度搜索：

```python
# 第一步：快速搜索（低 token）
quick = TavilySearch(max_results=2, search_depth="basic", include_answer=True)
overview = quick.invoke({"query": query})

# 仅在需要时：深度搜索（高 token）
if needs_deep_dive(overview):
    deep = TavilySearch(max_results=5, search_depth="advanced")
    detail = deep.invoke({"query": query})
```

### 优化效果对比

| 策略 | 预期 token 缩减 | 实现难度 | 对效果的影响 |
|------|----------------|----------|-------------|
| max_results=3 | 40-60% | 低 | 信息量减少 |
| 关闭 raw_content | 70-90% | 低 | 缺少全文细节 |
| basic 深度 | 50-70% | 低 | 搜索深度降低 |
| 关闭 answer | 10-20% | 低 | 需 Agent 自行总结 |
| 域名限定 | 20-40% | 中 | 提升精准度 |
| 缓存 | 取决于命中率 | 中 | 无影响 |
| 提示词控制 | 30-50% | 低 | 减少无效调用 |

### 推荐配置模板

```python
# === Token 敏感场景（如高频问答）===
tool = TavilySearch(
    max_results=3,
    search_depth="basic",
    include_answer=False,
    include_raw_content=False,
    include_images=False,
    topic="general",
    time_range="month",
)

# === 平衡场景（通用推荐）===
tool = TavilySearch(
    max_results=5,
    search_depth="basic",
    include_answer=True,
    include_raw_content=False,
    include_images=False,
    topic="general",
)

# === 高质量场景（深度调研）===
tool = TavilySearch(
    max_results=5,
    search_depth="advanced",
    include_answer=True,
    include_raw_content=True,
    include_images=False,
    include_domains=["arxiv.org", "wikipedia.org"],
)
```

---

## 三、自定义搜索工具（进阶方案）

当 Tavily 官方工具的 token 消耗仍然超出预期时，可以用 `@tool` 装饰器自行包装搜索逻辑，精确控制返回给模型的内容格式和长度。

### 核心思路

使用 `TavilyClient` 底层 SDK 做搜索，然后用 `@tool` 包装，只返回最小必需信息。

### 方案 A：标题 + URL 仅（最精简）

```python
from langchain.tools import tool
from tavily import TavilyClient
import os

client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))

@tool
def quick_search(query: str, max_results: int = 3) -> str:
    """Search the web for current information.

    Use this when you need real-time data, recent events, or facts
    that may be outside your training knowledge.

    Args:
        query: The search query
        max_results: Number of results (1-5)
    """
    response = client.search(
        query=query,
        search_depth="basic",
        max_results=max_results,
        include_answer=False,
        include_raw_content=False,
    )

    # 只提取标题和 URL，完全不返回 content
    lines = []
    for i, r in enumerate(response["results"], 1):
        lines.append(f"{i}. {r['title']} — {r['url']}")

    return "\n".join(lines)
```

**token 消耗**：每个结果约 50-100 tokens，相比 `TavilySearch(raw_content=False)` 再减少约 50%。

### 方案 B：分级内容 + 可配置长度

```python
@tool
def concise_search(query: str, detail_level: str = "low") -> str:
    """Search the web. Use detail_level 'low' for quick lookup, 'high' for depth.

    Args:
        query: The search query (required)
        detail_level: 'low' (title+URL only), 'medium' (brief snippet), 'high' (full snippet)
    """
    max_results = 3 if detail_level == "low" else 5
    depth = "basic" if detail_level in ("low", "medium") else "advanced"

    response = client.search(
        query=query,
        search_depth=depth,
        max_results=max_results,
        include_answer=False,
        include_raw_content=False,
    )

    parts = []
    for i, r in enumerate(response["results"], 1):
        if detail_level == "low":
            parts.append(f"{i}. {r['title']} — {r['url']}")
        elif detail_level == "medium":
            snippet = r['content'][:150]  # 硬截断 150 字
            parts.append(f"{i}. {r['title']}\n   {snippet}...\n   {r['url']}")
        else:
            parts.append(f"{i}. {r['title']}\n   {r['content'][:500]}\n   {r['url']}")

    return "\n\n".join(parts)
```

### 方案 C：结构化返回（Pydantic 精确控制）

```python
from pydantic import BaseModel, Field
from typing import Optional

class SearchResult(BaseModel):
    title: str
    url: str
    snippet: Optional[str] = None

class SearchResponse(BaseModel):
    results: list[SearchResult] = Field(description="Search results, count controlled by max_results")
    total_found: int

@tool
def structured_search(query: str, max_results: int = 3) -> SearchResponse:
    """Search the web and return structured results.

    Args:
        query: The search query
        max_results: Number of results (1-5)
    """
    response = client.search(
        query=query,
        search_depth="basic",
        max_results=max_results,
        include_raw_content=False,
    )

    results = [
        SearchResult(title=r["title"], url=r["url"], snippet=r.get("content", "")[:200])
        for r in response["results"]
    ][[
    return SearchResponse(results=results, total_found=len(results))
```

### token 消耗对比

| 方案 | 每个结果 token | 3 个结果总计 | 相比 Tavily(raw_content=True) |
|------|---------------|-------------|-------------------------------|
| Tavily 默认 (raw_content=True) | ~2000-5000 | ~6000-15000 | 基线 |
| Tavily (raw_content=False) | ~100-300 | ~300-900 | 减 90%+ |
| **自定义：标题+URL 仅** | **~50-100** | **~150-300** | **减 97%+** |
| **自定义：截断 150 字** | **~70-150** | **~200-450** | **减 95%+** |

### 适用场景判断

| 适合自定义工具的场景 | 不适合自定义工具的场景 |
|---------------------|---------------------|
| 高频问答，只需要确认有相关信息 | 深度调研，需要逐篇阅读全文 |
| token 成本敏感，按量计费 | 需要 HTML 全文做精细分析 |
| 搜索源需要混合（多引擎/内部库） | Tavily raw_content 已满足需求 |
| 返回格式需与下游系统对接 | 快速原型，不想多写代码 |

### 与 Tavily 官方工具的互补关系

自定义工具不是替代 Tavily，而是在 Tavily 之上加一层**控制面**。Tavily（或 `TavilyClient`）负责搜索和数据获取，`@tool` 负责格式化输出。两者组合使用可以达到最佳效果：Tavily 保证了搜索质量，`@tool` 保证了 token 效率。

---

## 相关页面

- [[langchain/tools/工具]] — LangChain 工具系统基础
- [[langchain/agents/Agent代理]] — Agent 核心概念
- [[langchain/agents/系统提示词]] — System Prompt 编写指南
- [[langchain/modules/langchain-mcp/概述]] — MCP 工具集成对比
