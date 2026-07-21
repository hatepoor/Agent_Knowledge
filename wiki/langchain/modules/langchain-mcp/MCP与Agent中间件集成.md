---
tags: [langchain, mcp, middleware, agent, integration, patterns]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# MCP 与 Agent 中间件集成

LangChain Agent 的中间件机制为 MCP 工具调用提供了强大的拦截、增强和控制能力。本文档介绍常见的集成模式。

## 中间件概念

LangChain Agent 中间件可以在以下时机介入 MCP 工具调用：

| 中间件类型 | 时机 | 用途 |
|-----------|------|------|
| `@before_agent` | Agent 运行前 | 动态加载工具、注入上下文 |
| `@wrap_tool_call` | 工具调用前后 | 监控、审批、缓存、校验 |

这些中间件可以链式组合，形成完整的 MCP 工具管理管线。

## 动态加载 MCP 工具

根据用户配置或运行时条件，动态决定加载哪些 MCP 服务器：

```python
from langchain.agents.middleware import before_agent
from langchain-mcp.client import MultiServerMCPClient

# 所有可用的 MCP 配置
MCP_CONFIGS = {
    "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "transport": "stdio",
    },
    "filesystem": {
        "url": "http://localhost:3000/sse",
        "transport": "sse",
    },
    "database": {
        "command": "python",
        "args": ["db_server.py"],
        "transport": "stdio",
    },
}

@before_agent
async def load_user_mcp_tools(state, runtime):
    """根据用户配置动态加载 MCP 工具"""
    # 从上下文获取用户选择的 MCP 服务器
    user_mcp = runtime.context.get("mcp_servers", ["github"])
    configs = {k: MCP_CONFIGS[k] for k in user_mcp if k in MCP_CONFIGS}

    client = MultiServerMCPClient(configs)
    mcp_tools = await client.get_tools()

    # 将 MCP 工具与内置工具合并
    from langchain.tools import tool

    @tool
    def search(query: str) -> str:
        """搜索知识库"""
        return f"搜索结果: {query}"

    @tool
    def calculate(expression: str) -> str:
        """计算数学表达式"""
        return str(eval(expression))

    builtin_tools = [search, calculate]
    all_tools = builtin_tools + mcp_tools

    return {"available_tools": all_tools}
```

### 按角色加载

```python
@before_agent
async def load_role_based_tools(state, runtime):
    """根据用户角色加载不同的 MCP 工具"""
    role = runtime.context.get("user_role", "viewer")

    role_configs = {
        "viewer": ["search"],                    # 只读工具
        "developer": ["search", "github", "filesystem"],  # 开发工具
        "admin": ["search", "github", "filesystem", "database"],  # 全部工具
    }

    server_names = role_configs.get(role, ["search"])
    configs = {k: MCP_CONFIGS[k] for k in server_names if k in MCP_CONFIGS}

    client = MultiServerMCPClient(configs)
    mcp_tools = await client.get_tools()
    return {"available_tools": mcp_tools}
```

## MCP 工具监控中间件

监控所有 MCP 工具调用，记录日志和性能指标：

```python
from langchain.agents.middleware import wrap_tool_call
import time
import logging

logger = logging.getLogger("mcp.monitor")

# MCP 工具前缀（用于识别 MCP 工具）
MCP_PREFIXES = ["github_", "fs_", "db_", "search_", "slack_"]

@wrap_tool_call
def mcp_tool_monitor(request, handler):
    """监控所有 MCP 工具调用"""
    tool_name = request.tool_call["name"]

    # 检测是否是 MCP 工具
    is_mcp = any(tool_name.startswith(prefix) for prefix in MCP_PREFIXES)

    if is_mcp:
        start_time = time.time()
        logger.info(f"[MCP] 调用: {tool_name}, 参数: {request.tool_call['args']}")

    result = handler(request)

    if is_mcp:
        duration = time.time() - start_time
        output_preview = str(result.output)[:100]
        logger.info(f"[MCP] 返回: {output_preview} (耗时: {duration:.2f}s)")

    return result
```

### 调用频率统计

```python
from collections import defaultdict
import time

# 全局统计
call_stats = defaultdict(lambda: {"count": 0, "total_time": 0, "errors": 0})

@wrap_tool_call
def mcp_stats_collector(request, handler):
    """收集 MCP 工具调用统计"""
    tool_name = request.tool_call["name"]
    start = time.time()

    try:
        result = handler(request)
        duration = time.time() - start
        call_stats[tool_name]["count"] += 1
        call_stats[tool_name]["total_time"] += duration
        return result
    except Exception:
        call_stats[tool_name]["errors"] += 1
        raise

def get_stats_report():
    """生成统计报告"""
    report = []
    for name, stats in call_stats.items():
        avg = stats["total_time"] / max(stats["count"], 1)
        report.append(
            f"{name}: 调用 {stats['count']} 次, "
            f"平均耗时 {avg:.2f}s, "
            f"错误 {stats['errors']} 次"
        )
    return "\n".join(report)
```

## MCP 工具缓存中间件

缓存只读 MCP 工具的结果，减少重复调用：

```python
from langchain.agents.middleware import wrap_tool_call

# 缓存存储
cache = {}

# 只读工具前缀（这些工具的相同输入总是返回相同输出）
READONLY_PREFIXES = ["get_", "list_", "search_", "query_", "fetch_", "read_"]

@wrap_tool_call
def mcp_tool_cache(request, handler):
    """缓存只读 MCP 工具的结果"""
    tool_name = request.tool_call["name"]
    args_key = str(sorted(request.tool_call["args"].items()))
    cache_key = f"{tool_name}:{args_key}"

    # 只缓存读取类工具
    is_readonly = any(tool_name.startswith(p) for p in READONLY_PREFIXES)

    if is_readonly:
        cached = cache.get(cache_key)
        if cached is not None:
            print(f"[缓存命中] {tool_name}")
            from langchain.agents.middleware import ToolResponse
            return ToolResponse(output=cached)

    result = handler(request)

    # 存入缓存
    if is_readonly:
        cache[cache_key] = result.output

    return result
```

### 带过期时间的缓存

```python
import time

class TimedCache:
    """带过期时间的缓存"""

    def __init__(self, ttl_seconds=300):
        self.ttl = ttl_seconds
        self._cache = {}

    def get(self, key):
        if key in self._cache:
            value, expiry = self._cache[key]
            if time.time() < expiry:
                return value
            else:
                del self._cache[key]  # 过期删除
        return None

    def set(self, key, value):
        self._cache[key] = (value, time.time() + self.ttl)

    def clear(self):
        self._cache.clear()

# 5 分钟 TTL 的缓存
timed_cache = TimedCache(ttl_seconds=300)

@wrap_tool_call
def mcp_timed_cache(request, handler):
    """带过期时间的 MCP 工具缓存"""
    tool_name = request.tool_call["name"]
    args_key = str(sorted(request.tool_call["args"].items()))
    cache_key = f"{tool_name}:{args_key}"

    is_readonly = any(tool_name.startswith(p) for p in READONLY_PREFIXES)

    if is_readonly:
        cached = timed_cache.get(cache_key)
        if cached is not None:
            from langchain.agents.middleware import ToolResponse
            return ToolResponse(output=cached)

    result = handler(request)

    if is_readonly:
        timed_cache.set(cache_key, result.output)

    return result
```

## MCP 工具审批中间件

对危险操作要求人工审批：

```python
from langchain.agents.middleware import wrap_tool_call

# 需要审批的工具列表
DANGEROUS_TOOLS = {
    "github_delete_file",
    "github_create_release",
    "db_execute_sql",
    "fs_write_file",
    "fs_delete_file",
    "slack_send_message",
}

@wrap_tool_call
def mcp_approval_gate(request, handler):
    """MCP 工具审批门控"""
    tool_name = request.tool_call["name"]

    if tool_name in DANGEROUS_TOOLS:
        # 检查是否已获审批
        approved = request.runtime.state.get("approved_tools", set())
        if tool_name not in approved:
            # 返回审批提示，Agent 会看到并可以请求人工审批
            return ToolResponse(
                output=(
                    f"⚠️ {tool_name} 是危险操作，需要人工审批。\n"
                    f"参数: {request.tool_call['args']}\n"
                    f"请联系管理员确认后重试。"
                )
            )

    return handler(request)
```

## 完整集成示例

将多种中间件组合，构建完整的 MCP 工具管理管线：

```python
import asyncio
import os
import time
import logging
from langchain.agents import create_agent
from langchain.agents.middleware import before_agent, wrap_tool_call
from langchain.agents.middleware import ToolResponse
from langchain-mcp.client import MultiServerMCPClient
from langgraph.checkpoint.memory import InMemorySaver

logger = logging.getLogger("mcp")

# ========== 配置 ==========

MCP_CONFIGS = {
    "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {"GITHUB_TOKEN": os.environ.get("GITHUB_TOKEN", "")},
        "transport": "stdio",
    },
    "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
        "transport": "stdio",
    },
    "database": {
        "command": "python",
        "args": ["db_server.py"],
        "transport": "stdio",
    },
}

DANGEROUS_TOOLS = {"github_delete_file", "db_execute_sql", "fs_delete_file"}
READONLY_PREFIXES = ["get_", "list_", "search_", "query_", "read_"]

# ========== 中间件 ==========

@before_agent
async def load_mcp_tools(state, runtime):
    """动态加载 MCP 工具"""
    user_servers = runtime.context.get("mcp_servers", ["github"])
    configs = {k: MCP_CONFIGS[k] for k in user_servers if k in MCP_CONFIGS}

    client = MultiServerMCPClient(configs)
    mcp_tools = await client.get_tools()
    return {"available_tools": mcp_tools}


@wrap_tool_call
def mcp_approval_and_monitor(request, handler):
    """MCP 工具审批 + 监控"""
    tool_name = request.tool_call["name"]
    args = request.tool_call["args"]
    start = time.time()

    # 1. 危险操作审批
    if tool_name in DANGEROUS_TOOLS:
        approved = request.runtime.state.get("approved", set())
        if tool_name not in approved:
            return ToolResponse(output=f"⚠️ {tool_name} 需要人工审批")

    # 2. 调用前日志
    logger.info(f"[MCP] 调用: {tool_name}({args})")

    # 3. 执行调用
    result = handler(request)
    duration = time.time() - start

    # 4. 调用后日志
    output_preview = str(result.output)[:80]
    logger.info(f"[MCP] → {output_preview} ({duration:.2f}s)")

    return result


@wrap_tool_call
def mcp_readonly_cache(request, handler):
    """只读 MCP 工具缓存"""
    tool_name = request.tool_call["name"]
    args_key = str(sorted(request.tool_call["args"].items()))
    cache_key = f"{tool_name}:{args_key}"

    is_readonly = any(tool_name.startswith(p) for p in READONLY_PREFIXES)

    if is_readonly and cache_key in _cache:
        return ToolResponse(output=_cache[cache_key])

    result = handler(request)

    if is_readonly:
        _cache[cache_key] = result.output

    return result

_cache = {}

# ========== 创建 Agent ==========

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[],  # 由 before_agent 中间件动态加载
    middleware=[
        load_mcp_tools,           # 1. 动态加载 MCP 工具
        mcp_readonly_cache,       # 2. 只读工具缓存
        mcp_approval_and_monitor, # 3. 审批 + 监控
    ],
    checkpointer=InMemorySaver(),
)

# ========== 运行 ==========

async def main():
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "列出我的 GitHub 仓库"}]},
        context={"mcp_servers": ["github"]},
    )
    print(result["messages"][-1].content)

asyncio.run(main())
```

## 中间件执行顺序

中间件的执行顺序很重要：

```
用户请求
  │
  ▼
① before_agent: 动态加载 MCP 工具
  │
  ▼
② wrap_tool_call: 缓存检查（只读工具直接返回）
  │
  ▼
③ wrap_tool_call: 审批检查（危险操作拦截）
  │
  ▼
④ wrap_tool_call: 监控日志（记录调用）
  │
  ▼
⑤ 实际 MCP 工具调用
  │
  ▼
④ 监控日志（记录返回结果）
  │
  ▼
③ 审批通过
  │
  ▼
② 缓存结果（只读工具）
  │
  ▼
返回给 Agent
```

**顺序原则：**

- 缓存放在最外层（最先检查，避免不必要的调用）
- 审批放在缓存之后（避免缓存审批结果）
- 监控紧贴实际调用（准确记录耗时）

## 错误处理与重试

```python
@wrap_tool_call
def mcp_retry_handler(request, handler):
    """MCP 工具调用失败时重试"""
    tool_name = request.tool_call["name"]
    max_retries = 3

    for attempt in range(max_retries):
        try:
            result = handler(request)
            return result
        except ConnectionError as e:
            if attempt < max_retries - 1:
                logger.warning(
                    f"[MCP] {tool_name} 连接失败，重试 {attempt + 1}/{max_retries}: {e}"
                )
                continue
            return ToolResponse(output=f"⚠️ {tool_name} 连接失败（已重试 {max_retries} 次）: {e}")
        except Exception as e:
            # 非连接错误不重试
            return ToolResponse(output=f"⚠️ {tool_name} 调用失败: {e}")
```

## 超时控制

```python
import signal

@wrap_tool_call
def mcp_timeout_handler(request, handler):
    """MCP 工具调用超时控制"""
    tool_name = request.tool_call["name"]
    timeout_seconds = 30  # 默认 30 秒超时

    # 危险操作给更长超时
    if tool_name.startswith("db_"):
        timeout_seconds = 60

    try:
        # 使用 signal 设置超时（仅 Unix）
        old_handler = signal.signal(signal.SIGALRM, lambda *_: None)
        signal.alarm(timeout_seconds)
        try:
            result = handler(request)
            return result
        finally:
            signal.alarm(0)
            signal.signal(signal.SIGALRM, old_handler)
    except Exception:
        return ToolResponse(output=f"⚠️ {tool_name} 调用超时（>{timeout_seconds}s）")
```

## 相关

- [[langchain/modules/langchain-mcp/概述|<- 返回 mcp-adapters 概述]]
- [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]]
- [[langchain/modules/langchain-mcp/MCP服务器开发|MCP 服务器开发]]
- [[langchain/modules/langchain-mcp/MCP安全模型|MCP 安全模型]]
- [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]
- [[langchain/modules/langchain_agents/概述|agents 模块]]
- [[langchain/modules/langchain-mcp/MCP协议概述|MCP 协议]]
