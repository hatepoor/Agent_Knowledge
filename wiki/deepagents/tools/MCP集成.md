---
tags: [deepagents, MCP, tools, 集成]
created: 2026-08-03
updated: 2026-08-03
type: howto
source_url: https://docs.langchain.com/oss/python/deepagents/customization
deepagents_version: ">=0.7"
---

# DeepAgents 与 MCP 集成

MCP（Model Context Protocol）工具通过 `tools` 参数传入 `create_deep_agent`。**MCP 工具加载后就是标准的 LangChain `BaseTool`**，和本地 `@tool` 没有任何区别——Agent 的规划、子代理、HITL 机制对它们一视同仁。

## 一、核心用法

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

async def main():
    # 1. 加载 MCP 工具
    mcp_tools = []
    async with MultiServerMCPClient({
        "github": {"url": "http://localhost:8001/mcp"},   # HTTP 传输
        "database": {"command": "python", "args": ["db_server.py"]},  # stdio 传输
    }) as client:
        mcp_tools = await client.get_tools()
        # mcp_tools 是 List[BaseTool] —— 与 @tool 装饰的工具类型一致

        # 2. 与本地工具合并，传给 create_deep_agent
        agent = create_deep_agent(
            model="deepseek:deepseek-v4-flash",
            tools=[*mcp_tools, internet_search, calculator],
            system_prompt="你可以访问 GitHub 和数据库工具。",
        )

        # 3. 正常调用
        result = await agent.ainvoke({
            "messages": [{"role": "user", "content": "查询仓库里的 issue 列表"}]
        })
        print(result["messages"][-1].content)

asyncio.run(main())
```

关键点：**MCP 工具和本地工具在 Agent 内部完全等价**。`create_deep_agent` 不区分工具来源，中间件栈（TodoList/Filesystem/HITL 等）对所有工具统一生效。

## 二、配置驱动加载（生产推荐）

MCP 服务器配置写在 JSON/YAML 中，不写死在代码里：

```json
// mcp.json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "http://localhost:8001/mcp",
      "auth": "oauth",
      "allowedTools": ["github_get_file", "github_create_pull_request"]
    },
    "database": {
      "command": "python",
      "args": ["db_server.py"],
      "env": {
        "DB_URL": "${DB_URL}"      // 从环境变量解析
      }
    }
  }
}
```

```python
import json
from langchain_mcp_adapters.client import MultiServerMCPClient

def load_mcp_config(path: str = "mcp.json") -> dict:
    """读取 MCP 服务器配置文件"""
    with open(path, encoding="utf-8") as f:
        return json.load(f)["mcpServers"]

async def build_agent():
    # 从配置加载所有 MCP 服务器
    config = load_mcp_config()
    async with MultiServerMCPClient(config) as client:
        tools = await client.get_tools()
        agent = create_deep_agent(model=llm, tools=tools)
        return agent
```

## 三、与 MCP 服务器错误处理

单个 MCP 服务器挂了不应导致整个 Agent 不可用：

```python
async def load_mcp_tools_safe() -> list:
    """容错加载：失败的服务器只告警，不影响其他工具"""
    all_tools = []
    config = load_mcp_config()

    for server_name, server_config in config.items():
        try:
            async with MultiServerMCPClient({server_name: server_config}) as client:
                tools = await client.get_tools()
                all_tools.extend(tools)
                print(f"[MCP] {server_name}: 加载 {len(tools)} 个工具")
        except Exception as e:
            print(f"[MCP] {server_name} 加载失败: {e}，跳过")
            continue
    return all_tools
```

## 四、MCP 工具与中间件的配合

| 中间件 | 对 MCP 工具的影响 |
|--------|------------------|
| `TodoListMiddleware` | 无影响，规划工具与 MCP 工具独立 |
| `FilesystemMiddleware` | 无影响，文件操作走 backend，MCP 工具照常 |
| `SubAgentMiddleware` | 子代理继承主 Agent 的所有工具（含 MCP） |
| `HumanInTheLoopMiddleware` | **可以拦截 MCP 工具**：`interrupt_on={"github_create_pull_request": True}` |
| `SummarizationMiddleware` | 对工具调用历史统一摘要 |

子代理继承 MCP 工具：

```python
from deepagents import create_deep_agent, SubAgent

agent = create_deep_agent(
    model=llm,
    tools=[*mcp_tools, *local_tools],   # 主 Agent 工具
    subagents=[
        SubAgent(
            name="db_analyst",
            description="数据库查询分析",
            # 不传 tools → 自动继承主 Agent 的全部工具（含 MCP 的 database）
        ),
        SubAgent(
            name="clean_researcher",
            description="只做搜索，不碰数据库",
            tools=[internet_search],     # 显式指定 → 只继承这一部分
        ),
    ],
)
```

## 五、MCP Prompts 作为远程 Skills

MCP 服务器可以用 `@mcp.prompt()` 暴露技能模板，Agent 端封装成工具调用：

```python
# ===== MCP 服务器端 =====
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("skill-server")

@mcp.prompt()
def code_review(code: str) -> str:
    """代码审查技能"""
    return f"请按照以下流程审查代码:\n{code}"

# ===== Agent 端 =====
from langchain_core.tools import tool

@tool
async def get_remote_skill(skill_name: str, **kwargs) -> str:
    """从远程技能服务器获取技能模板"""
    result = await mcp_client.get_prompt(
        server_name="skill-server",
        prompt_name=skill_name,
        arguments=kwargs,
    )
    return result[0].content

agent = create_deep_agent(model=llm, tools=[*mcp_tools, get_remote_skill])
```

## 六、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| MCP 服务器启动失败 | 配置的 command/url 不对 | 单独测试服务器，再集成 |
| 工具未出现在 Agent 中 | `get_tools()` 在 `async with` 块外调用 | 确保在 client 生命周期内获取 |
| 认证失败 | 缺少 API Key | 用 env 变量注入，不要硬编码 |
| 工具调用超时 | MCP 服务器响应慢 | 配置更长的 timeout，或异步工具 |
| HITL 不拦截 MCP 工具 | interrupt_on 里没写工具名 | 工具名必须是 MCP 工具的实际名称（如 `github_create_pull_request`） |

## 相关页面

- [[deepagents/tools/内置工具概述]] — 内置工具 vs MCP 工具
- [[deepagents/customization/create_deep_agent参数详解]] — tools 参数支持 MCP 工具
- [[deepagents/middleware/HumanInTheLoop详解]] — 拦截 MCP 工具调用
- [[langchain/modules/langchain-mcp/MCP服务器开发]] — 开发 MCP 服务器
- [[langchain/modules/langchain-mcp/MultiServerMCPClient]] — MultiServerMCPClient 详解
