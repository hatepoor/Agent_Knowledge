---
tags: [langchain, mcp, server, fastmcp, development]
type: concept
created: 2026-07-11
updated: 2026-07-11
---

# MCP 服务器开发

使用 FastMCP 框架从零开发 MCP 服务器，并将其接入 LangChain Agent。

## 安装

```bash
pip install "mcp[cli]" httpx
# 或用 uv（更快）
uv add "mcp[cli]" httpx
```

`mcp[cli]` 包含 FastMCP 高层 API 和命令行工具（`mcp dev`、`mcp run`）。`httpx` 用于 HTTP 传输方式的客户端支持。

## 最简服务器

```python
from mcp.server.fastmcp import FastMCP

# 创建服务器实例，名称用于客户端标识
mcp = FastMCP("my-server")

@mcp.tool()
async def add(a: int, b: int) -> int:
    """两数相加"""
    return a + b

@mcp.tool()
async def get_weather(city: str) -> str:
    """获取城市天气信息"""
    # 实际项目中调用天气 API
    return f"{city}: 晴天, 28°C"

if __name__ == "__main__":
    mcp.run(transport="stdio")  # 或 "sse" / "streamable_http"
```

运行方式：

```bash
# stdio 模式（由客户端启动）
python my_server.py

# 或用 mcp 命令行工具
mcp run my_server.py
```

**关键点：**

- `FastMCP` 是 MCP 官方 Python SDK 的高层 API，极大简化服务器开发
- 每个工具函数必须有 **docstring**（作为工具描述发送给 LLM）
- **类型注解必须明确**（用于自动生成 JSON Schema，LLM 据此构造参数）
- 工具函数可以是 `async` 或同步的

## 三种能力

MCP 服务器提供三种核心能力：

| 能力 | 说明 | 类比 | 使用频率 |
|------|------|------|----------|
| **Tools** | 可调用的函数 | API 端点 | 最常用 |
| **Resources** | 类文件的数据源 | GET 接口 | 中等 |
| **Prompts** | 提示词模板 | 模板引擎 | 较少 |

### Tools（工具）

工具是 MCP 最常用的能力，LLM 可以像调用函数一样使用它们。

```python
@mcp.tool()
async def search_documents(query: str, max_results: int = 10) -> list[dict]:
    """搜索文档库并返回匹配结果"""
    # 实际项目中连接搜索引擎或数据库
    results = [
        {"title": f"文档 {i}", "score": 0.9 - i * 0.1}
        for i in range(min(max_results, 3))
    ]
    return results

@mcp.tool()
async def create_task(title: str, priority: str = "medium") -> dict:
    """创建新任务并返回任务信息"""
    return {
        "id": "task-001",
        "title": title,
        "priority": priority,
        "status": "created",
    }
```

**工具设计原则：**

- 函数名用动词开头（`get_`、`list_`、`create_`、`delete_`、`search_`）
- docstring 要清晰描述功能（这是 LLM 理解工具的唯一依据）
- 参数类型用 Python 标准类型（`str`、`int`、`bool`、`list`、`dict`）
- 返回值用可 JSON 序列化的类型

### @mcp.tool() 装饰器参数

```python
@mcp.tool(
    name="find_products",                          # 自定义工具名（覆盖函数名）
    description="搜索产品目录，带有可选的分类过滤。",  # 自定义描述（覆盖 docstring）
    tags={"catalog", "search"},                    # 分类标签（LLM 不可见）
    meta={"version": "1.2", "author": "team-a"},   # 元数据（LLM 不可见）
)
async def search_products_in_catalog(query: str) -> str:
    """这个 docstring 会被 description 参数覆盖"""
    ...
```

| 参数 | 谁可见 | 作用 | 默认值 |
|------|--------|------|--------|
| `name` | LLM | 工具名称，覆盖函数名 | 函数名 |
| `description` | LLM | 工具描述，覆盖 docstring | 函数 docstring |
| `tags` | 开发者/客户端 | 分类标签，用于过滤和组织 | 无 |
| `meta` | 开发者/客户端 | 附加元数据（版本、作者等） | 无 |

**tags 的用途：客户端按标签过滤加载工具**

一个 MCP 服务器可能有几十个工具，但某个 Agent 只需要其中一部分。tags 让客户端可以按需加载：

```python
# 服务器端：给工具打标签
@mcp.tool(tags={"catalog", "read"})
async def find_products(query: str) -> str: ...

@mcp.tool(tags={"catalog", "admin"})
async def delete_product(product_id: str) -> str: ...

@mcp.tool(tags={"user", "read"})
async def get_profile(user_id: str) -> str: ...
```

```python
# 客户端：按标签过滤
all_tools = await client.get_tools()

# 普通用户只加载 read 类工具
safe_tools = [t for t in all_tools if "read" in (t.tags or set())]

# 管理员加载 admin 类工具
admin_tools = [t for t in all_tools if "admin" in (t.tags or set())]

# 只加载产品目录相关
catalog_tools = [t for t in all_tools if "catalog" in (t.tags or set())]
```

> **注意：** 过滤是客户端自己做的。MCP 协议会把所有工具定义都发给客户端，客户端筛选后再注入给 Agent。tags 不会出现在 LLM 的上下文中。

**好处：**

- 减少 token 消耗（工具描述也占 LLM 上下文）
- 降低 LLM 选错工具的概率（选项越少越精准）
- 权限隔离（不同角色加载不同工具集）

### Resources（资源）

资源是只读的数据源，类似 REST API 的 GET 接口，客户端可以读取但不能修改。

```python
@mcp.resource("config://app")
def get_config() -> str:
    """返回应用配置"""
    return "debug=true\nport=8080"

@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """获取用户资料"""
    return f"User {user_id}: name=Alice, role=admin"

@mcp.resource("data://products")
def list_products() -> str:
    """返回产品列表"""
    return "Product A, Product B, Product C"
```

**资源特点：**

- URI 格式支持模板参数（`{user_id}`）
- 返回字符串或字节数据
- 适合暴露配置、状态、元数据等只读信息
- 客户端通过 `read_resource` 读取

### Prompts（提示词模板）

提示词模板是预定义的提示词生成器，帮助用户快速构造标准化的提示词。

```python
@mcp.prompt()
def code_review(code: str) -> str:
    """生成代码审查提示词"""
    return f"请审查以下代码并给出改进建议：\n\n{code}"

@mcp.prompt()
def bug_report(description: str, steps: str) -> str:
    """生成 Bug 报告提示词"""
    return f"""请根据以下信息生成一份完整的 Bug 报告：

问题描述：{description}
复现步骤：{steps}

请包含：预期行为、实际行为、严重程度评估。"""
```

> 完整的提示词编写规范、参数匹配机制和执行流程详见 [[langchain/modules/langchain-mcp/MCP提示词编写与执行流程|MCP 提示词编写与执行流程]]。

## 传输方式

MCP 服务器支持三种传输方式，详见 [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]：

### stdio -- 本地进程通信

```python
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

- 最常用，由客户端（如 [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]]）启动子进程
- 通过标准输入/输出通信，不暴露网络端口
- 生命周期由客户端管理

### sse -- Server-Sent Events

```python
if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

- 基于 HTTP 的服务端推送
- 服务器需要独立启动并保持运行
- 适合内部微服务

### streamable_http -- 新版 HTTP 传输

```python
if __name__ == "__main__":
    mcp.run(transport="streamable_http", host="0.0.0.0", port=8000)
```

- MCP 协议推荐的新传输方式
- 基于标准 HTTP，支持流式响应
- 适合云服务和远程 API

## 与 LangChain Agent 集成

开发好的 MCP 服务器可以通过 [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]] 接入 LangChain Agent：

```python
import asyncio
from langchain-mcp.client import MultiServerMCPClient
from langchain.agents import create_agent

async def main():
    # 连接自定义 MCP 服务器
    client = MultiServerMCPClient({
        "my-server": {
            "command": "python",
            "args": ["/path/to/my_server.py"],
            "transport": "stdio",
        },
    })

    # 加载所有工具
    tools = await client.get_tools()
    print(f"已加载 {len(tools)} 个工具")

    # 创建 Agent
    agent = create_agent("openai:gpt-4o-mini", tools=tools)
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "1+1=?"}]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

混合使用自定义服务器和第三方服务器：

```python
import os

async def multi_server_agent():
    client = MultiServerMCPClient({
        # 自定义服务器
        "my-tools": {
            "command": "python",
            "args": ["my_server.py"],
            "transport": "stdio",
        },
        # 第三方 GitHub 服务器
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ.get("GITHUB_TOKEN", "")},
            "transport": "stdio",
        },
        # 远程 HTTP 服务器
        "weather": {
            "url": "https://weather-mcp.example.com/mcp",
            "transport": "streamable_http",
        },
    })

    tools = await client.get_tools()
    agent = create_agent("openai:gpt-4o-mini", tools=tools)
    return agent
```

更多服务器配置示例参见 [[langchain/modules/langchain-mcp/MCP服务器示例|MCP 服务器示例]]。

## 高级特性

### 带状态的工具

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("stateful-server")

# 使用闭包或类维护状态
class TaskManager:
    def __init__(self):
        self.tasks = []
        self.counter = 0

    def register_tools(self, mcp: FastMCP):
        @mcp.tool()
        def create_task(title: str) -> dict:
            """创建新任务"""
            self.counter += 1
            task = {"id": self.counter, "title": title, "done": False}
            self.tasks.append(task)
            return task

        @mcp.tool()
        def list_tasks() -> list[dict]:
            """列出所有任务"""
            return self.tasks

        @mcp.tool()
        def complete_task(task_id: int) -> dict:
            """标记任务为完成"""
            for task in self.tasks:
                if task["id"] == task_id:
                    task["done"] = True
                    return task
            return {"error": f"Task {task_id} not found"}

manager = TaskManager()
manager.register_tools(mcp)
```

### 错误处理

```python
@mcp.tool()
async def divide(a: float, b: float) -> float:
    """两数相除"""
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

@mcp.tool()
async def fetch_url(url: str) -> str:
    """获取网页内容"""
    import httpx
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(url, timeout=10)
            resp.raise_for_status()
            return resp.text[:5000]  # 截断避免过长
    except httpx.HTTPError as e:
        return f"请求失败: {e}"
```

### 文件操作工具

```python
import os
from pathlib import Path

@mcp.tool()
async def read_file(path: str) -> str:
    """读取文件内容"""
    file_path = Path(path).resolve()
    # 安全检查：限制可访问的目录
    allowed_dir = Path("/safe/directory").resolve()
    if not str(file_path).startswith(str(allowed_dir)):
        return "错误：不允许访问该路径"
    if not file_path.exists():
        return f"文件不存在: {path}"
    return file_path.read_text(encoding="utf-8")

@mcp.tool()
async def write_file(path: str, content: str) -> str:
    """写入文件内容"""
    file_path = Path(path).resolve()
    allowed_dir = Path("/safe/directory").resolve()
    if not str(file_path).startswith(str(allowed_dir)):
        return "错误：不允许访问该路径"
    file_path.parent.mkdir(parents=True, exist_ok=True)
    file_path.write_text(content, encoding="utf-8")
    return f"已写入 {path}（{len(content)} 字节）"
```

## 调试

### MCP Inspector（可视化调试）

```bash
# 启动 MCP Inspector，提供 Web UI 进行可视化调试
mcp dev my_server.py
```

MCP Inspector 可以：

- 浏览所有注册的工具、资源、提示词
- 手动调用工具并查看返回结果
- 检查 JSON Schema 是否正确生成
- 模拟客户端请求

### 命令行运行

```bash
# 直接运行服务器（stdio 模式）
mcp run my_server.py

# 指定传输方式
mcp run my_server.py --transport sse

# 指定端口
mcp run my_server.py --transport sse --port 8000
```

### 环境变量调试

```bash
# 启用详细日志
export MCP_DEBUG=1
python my_server.py
```

## 常见陷阱

### stdio 模式下不要 print 到 stdout

```python
# 错误！print 输出会破坏 MCP 协议
@mcp.tool()
async def bad_tool(x: int) -> int:
    print(f"调试信息: x={x}")  # 这会破坏 stdio 通信！
    return x * 2

# 正确：使用 stderr 或 logging
import logging
logger = logging.getLogger(__name__)

@mcp.tool()
async def good_tool(x: int) -> int:
    logger.debug(f"调试信息: x={x}")  # logging 默认输出到 stderr
    return x * 2
```

### 工具函数必须有 docstring

```python
# 错误：没有 docstring，LLM 不知道这个工具做什么
@mcp.tool()
async def calc(x: int, y: int) -> int:
    return x + y

# 正确：清晰的 docstring
@mcp.tool()
async def calculate_sum(x: int, y: int) -> int:
    """计算两个整数的和并返回结果"""
    return x + y
```

### 类型注解必须明确

```python
# 错误：缺少类型注解，无法生成 JSON Schema
@mcp.tool()
def bad_tool(data):  # 没有类型注解
    """处理数据"""
    return data

# 正确：明确的类型注解
@mcp.tool()
def process_data(data: list[str], count: int = 10) -> dict:
    """处理字符串列表并返回统计结果"""
    return {"total": len(data), "showing": data[:count]}
```

### 避免返回过大内容

```python
# 错误：可能返回数 MB 的内容，LLM 上下文放不下
@mcp.tool()
async def read_large_file(path: str) -> str:
    """读取文件"""
    return Path(path).read_text()  # 可能非常大！

# 正确：截断或分页
@mcp.tool()
async def read_file_preview(path: str, lines: int = 50) -> str:
    """读取文件的前 N 行"""
    content = Path(path).read_text().splitlines()
    preview = "\n".join(content[:lines])
    total = len(content)
    return f"{preview}\n\n... 共 {total} 行，已显示前 {lines} 行"
```

## 完整项目结构

```
my-mcp-server/
├── pyproject.toml        # 项目配置和依赖
├── src/
│   └── my_server/
│       ├── __init__.py
│       ├── server.py     # FastMCP 服务器定义
│       ├── tools.py      # 工具实现
│       ├── resources.py  # 资源定义
│       └── prompts.py    # 提示词模板
└── tests/
    └── test_tools.py     # 工具测试
```

`pyproject.toml` 配置：

```toml
[project]
name = "my-mcp-server"
version = "0.1.0"
dependencies = [
    "mcp[cli]>=1.0.0",
    "httpx>=0.27.0",
]

[project.scripts]
my-mcp-server = "my_server.server:main"
```

## 相关

- [[langchain/modules/langchain-mcp/概述|<- 返回 mcp-adapters 概述]]
- [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]]
- [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]
- [[langchain/modules/langchain-mcp/MCP服务器示例|MCP 服务器示例]]
- [[langchain/modules/langchain-mcp/MCP安全模型|MCP 安全模型]]
- [[langchain/modules/langchain-mcp/MCP与Agent中间件集成|MCP 与 Agent 中间件集成]]
- [[langchain/modules/langchain-mcp/MCP提示词编写与执行流程|MCP 提示词编写与执行流程]]
- [[langchain/modules/langchain-mcp/MCP协议概述|MCP 协议]]
