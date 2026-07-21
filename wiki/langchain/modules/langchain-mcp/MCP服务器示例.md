---
tags: [langchain, mcp, server, examples, ecosystem]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# MCP 服务器示例

MCP 生态中已有大量官方和社区维护的服务器，可以直接集成到 LangChain Agent 中。

## 常见 MCP 服务器

| 服务器 | 功能 | 启动方式 |
|--------|------|----------|
| GitHub | 仓库管理、Issue、PR 操作 | `npx @modelcontextprotocol/server-github` |
| Filesystem | 文件系统读写 | `npx @modelcontextprotocol/server-filesystem` |
| Postgres | 数据库查询 | `npx @modelcontextprotocol/server-postgres` |
| Puppeteer | 浏览器自动化 | `npx @modelcontextprotocol/server-puppeteer` |
| Slack | Slack 消息与频道 | `npx @modelcontextprotocol/server-slack` |
| Google Maps | 地图与地点搜索 | `npx @modelcontextprotocol/server-google-maps` |
| Brave Search | 网络搜索 | `npx @modelcontextprotocol/server-brave-search` |

## 配置示例

### GitHub

```python
import os

config = {
    "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {
            "GITHUB_TOKEN": os.environ.get("GITHUB_TOKEN", ""),
        },
        "transport": "stdio",
    },
}
```

### Filesystem

```python
config = {
    "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
        "transport": "stdio",
    },
}
```

### Postgres

```python
config = {
    "database": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres",
                 "postgresql://user:pass@localhost:5432/mydb"],
        "transport": "stdio",
    },
}
```

### Brave Search

```python
config = {
    "search": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-brave-search"],
        "env": {
            "BRAVE_API_KEY": os.environ.get("BRAVE_API_KEY", ""),
        },
        "transport": "stdio",
    },
}
```

## 多服务器组合使用

```python
import asyncio
from langchain-mcp.client import MultiServerMCPClient
from langchain.agents import create_agent

async def create_powered_agent():
    client = MultiServerMCPClient({
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
            "transport": "stdio",
        },
        "search": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-brave-search"],
            "env": {"BRAVE_API_KEY": os.environ["BRAVE_API_KEY"]},
            "transport": "stdio",
        },
        "filesystem": {
            "url": "http://localhost:3000/sse",
            "transport": "sse",
        },
    })

    tools = await client.get_tools()
    agent = create_agent("openai:gpt-5.5", tools=tools)
    return agent

async def main():
    agent = await create_powered_agent()
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "Search for LangChain news and summarize"}]
    })
    print(result["messages"][-1][[.content)

asyncio.run(main())
```

## 注意事项

- 所有 `npx` 命令需要本地已安装 Node.js 和 npm
- 部分服务器需要 API key（通过 `env` 传递）
- 首次运行 `npx` 会自动下载包，可能需要等待
- 服务器启动后由客户端管理生命周期
- 生产环境建议固定版本号（如 `@modelcontextprotocol/server-github@1.0.0`）

## 相关

- [[langchain/modules/langchain-mcp/概述|<- 返回 mcp-adapters 概述]]
- [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]]
- [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]
- [[langchain/modules/langchain-mcp/MCP协议概述|MCP 协议]]
