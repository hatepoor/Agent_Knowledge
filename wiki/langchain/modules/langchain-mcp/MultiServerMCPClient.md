---
tags: [langchain, mcp, multi-server, client]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# MultiServerMCPClient

`MultiServerMCPClient` 是 langchain-mcp-adapters 的核心类，支持同时连接**多个 MCP 服务器**并统一加载工具。

## 导入

```python
from langchain-mcp.client import MultiServerMCPClient
```

## 多服务器配置

```python
import os

mcp_config = {
    # 本地 stdio 服务器：GitHub
    "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {"GITHUB_TOKEN": os.environ.get("GITHUB_TOKEN", "")},
        "transport": "stdio",
    },
    # 远程 SSE 服务器：内部文件系统
    "filesystem": {
        "url": "http://localhost:3000/sse",
        "transport": "sse",
    },
    # WebSocket 服务器：数据库
    "database": {
        "url": "ws://localhost:8000/ws",
        "transport": "websocket",
    },
    # 新版 streamable_http
    "weather": {
        "url": "https://weather-mcp.example.com/mcp",
        "transport": "streamable_http",
    },
}
```

## 加载工具并创建 Agent

```python
import asyncio
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

async def build_agent():
    client = MultiServerMCPClient(mcp_config)
    mcp_tools = await client.get_tools()
    # mcp_tools 是 LangChain 工具列表，可直接用

    print(f"Loaded {len(mcp_tools)} MCP tools:")
    for tool in mcp_tools:
        print(f"  - {tool.name}: {tool.description[:60]}")

    model = init_chat_model("openai:gpt-5.5")
    agent = create_agent(
        model,
        tools=mcp_tools,  # MCP 工具与本地工具可混合
    )
    return agent

async def main():
    agent = await build_agent()

    # Agent 会自动选择合适的 MCP 工具
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "List my GitHub repos"}]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

## 单独使用某个 MCP 工具

```python
async def use_single_tool():
    client = MultiServerMCPClient({"github": mcp_config["github"]})
    tools = await client.get_tools()

    # 找到指定工具
    list_repos = next(t for t in tools if t.name == "list_repositories")
    result = await list_repos.ainvoke({})
    print(result)

asyncio.run(use_single_tool())
```

## 与本地工具混合

```python
from langchain.tools import tool

@tool
def local_calculator(expression: str) -> str:
    """Calculate a math expression."""
    return str(eval(expression))

async def mixed_agent():
    client = MultiServerMCPClient({"github": mcp_config["github"]})
    mcp_tools = await client.get_tools()

    agent = create_agent(
        "openai:gpt-5.5",
        tools=[*mcp_tools, local_calculator],  # 混合 MCP 和本地工具
    )
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "Calculate 2^10 and list repos"}]
    })
    print(result["messages"][-1].content)

asyncio.run(mixed_agent())
```

## 在 LangGraph 节点中使用

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    messages: list
    data: dict

async def call_mcp_node(state: State):
    client = MultiServerMCPClient({"github": mcp_config["github"]})
    tools = await client.get_tools()
    search_tool = tools[0]
    result = await search_tool.ainvoke({"query": state["messages"][-1]["content"][[})
    return {"data": {"mcp_result": result}}

# graph = StateGraph(State).add_node("mcp", call_mcp_node)...
```

## 注意事项

- `get_tools()` 是异步方法，需要 `await`
- MCP 工具与 LangChain 本地工具在使用上无差别
- 工具是动态加载的，服务器更新工具定义后无需重启客户端
- stdio 类型的服务器会在 `get_tools()` 时启动子进程

## 相关

- [[langchain/modules/langchain-mcp/概述|<- 返回 mcp-adapters 概述]]
- [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]
