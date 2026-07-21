---
tags: [langchain, mcp, prompt, 提示词, 执行流程]
type: concept
created: 2026-07-18
updated: 2026-07-18
---

# MCP 提示词编写与执行流程

在 MCP 服务器中编写提示词模板（`@mcp.prompt()`）的完整工作流程，包括参数匹配机制。

## 核心概念

MCP 的提示词涉及两个层面：

| 层面 | 装饰器 | 作用 | 返回值 |
|------|--------|------|--------|
| 工具描述 | `@mcp.tool()` | 执行实际操作，返回数据 | 函数返回值（JSON 可序列化） |
| 提示词模板 | `@mcp.prompt()` | 组装提示词，不执行操作 | 字符串或消息列表 |

**关键区别：** tool 是"干活"的（读文件、查数据库），prompt 是"出主意"的（返回写好的提示词让 LLM 按套路行动）。

## 完整执行流程

```
用户: "帮我审查这段代码 def add(a,b): return a+b"
  │
  ▼
┌─────────────────────────────────────────┐
│  Agent (LLM) 收到：                      │
│  1. 用户的消息                            │
│  2. 所有工具/prompt 的描述列表             │
│     （名称 + docstring + 参数 JSON Schema）│
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  LLM 判断：                              │
│  "用户要审查代码 → 有 code_review prompt" │
│  "从消息中提取 code 参数"                 │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  LLM 输出结构化调用指令：                 │
│  {                                       │
│    "tool": "code_review",                │
│    "arguments": {                        │
│      "code": "def add(a,b): return a+b" │
│    }                                     │
│  }                                       │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  MCP 客户端收到 JSON                     │
│  → 转发给 MCP 服务器                     │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  MCP 服务器执行 Python 函数：             │
│  code_review(code="def add(a,b)...")     │
│  → f-string 机械替换 {code}              │
│  → 返回组装好的完整提示词                 │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  提示词返回给 LLM                        │
│  → LLM 按照提示词执行任务                │
│  → 生成最终回复给用户                    │
└─────────────────────────────────────────┘
```

## 参数匹配机制

`{}` 中的占位符不是 LLM 直接替换的，而是两步配合完成。

### 第一步：函数签名 → JSON Schema（自动）

MCP 框架自动将 Python 函数签名转为 JSON Schema 发送给 LLM：

```python
@mcp.prompt()
def code_review(code: str, language: str = "Python") -> str:
    """生成代码审查提示词"""
    return f"请审查以下{language}代码：\n\n{code}"
```

LLM 实际看到的 schema：

```json
{
  "name": "code_review",
  "description": "生成代码审查提示词",
  "parameters": {
    "type": "object",
    "properties": {
      "code": { "type": "string", "description": "要审查的代码" },
      "language": { "type": "string", "default": "Python" }
    },
    "required": ["code"]
  }
}
```

### 第二步：LLM 决定参数值 → Python 函数调用（机械替换）

LLM 根据用户消息推断参数值，输出 JSON 格式的调用指令。MCP 客户端拿到后执行普通 Python 函数调用，f-string 中的 `{code}`、`{language}` 由 Python 解释器替换，与 LLM 无关。

```
LLM 推断参数  ──→  {"code": "def add(a,b): return a+b"}
                        │
                        ▼
Python 执行函数  ──→  code_review(code="def add(a,b): return a+b")
                        │
                        ▼
f-string 替换    ──→  "请审查以下Python代码：\n\ndef add(a,b): return a+b"
```

**总结：** LLM 只负责决定"调哪个 prompt、传什么参数值"，`{}` 的实际替换是 Python 函数执行时的机械操作。

## 编写规范

### 参数命名

参数名是 LLM 理解该传什么的唯一依据，必须语义清晰：

```python
# 差：LLM 不知道 data 该填什么
@mcp.prompt()
def analyze(data: str, n: int) -> str:
    ...

# 好：参数名自解释
@mcp.prompt()
def analyze_bug_report(description: str, max_suggestions: int = 3) -> str:
    ...
```

### docstring 写法

docstring 会作为工具描述发送给 LLM，需说明功能和适用场景：

```python
# 差：太模糊
@mcp.prompt()
def review(code: str) -> str:
    """审查"""
    ...

# 好：说明何时使用、返回什么
@mcp.prompt()
def code_review(code: str, language: str = "Python") -> str:
    """生成代码审查提示词。
    当用户要求审查代码、检查代码质量时使用。
    返回包含审查要点和输出格式要求的完整提示词。"""
    ...
```

### 提示词模板结构

模板内部应包含：角色设定、任务指令、输入占位、输出格式要求。

```python
@mcp.prompt()
def code_review(code: str, language: str = "Python") -> str:
    """生成代码审查提示词"""
    return f"""请作为资深{language}开发者审查以下代码。

审查要点：
1. 逻辑正确性
2. 安全隐患
3. 性能优化空间
4. 编码规范

代码：
```{language.lower()}
{code}
```

请逐项给出评价和改进建议。"""
```

### 条件参数

可选参数用默认值，模板中按需拼接：

```python
@mcp.prompt()
def analyze_bug(description: str, error_log: str = "", severity: str = "unknown") -> str:
    """生成 Bug 分析提示词"""
    prompt = f"分析以下 Bug：\n\n{description}"
    
    if error_log:
        prompt += f"\n\n错误日志：\n```\n{error_log}\n```"
    
    if severity != "unknown":
        prompt += f"\n\n严重程度：{severity}"
    
    prompt += "\n\n请给出根因分析和修复建议。"
    return prompt
```

## 与 LangChain Agent 集成

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

async def main():
    # 连接 MCP 服务器
    client = MultiServerMCPClient({
        "code-tools": {
            "command": "python",
            "args": ["code_tools_server.py"],
        },
    })

    # 获取所有工具（tool + prompt 都转为 LangChain Tool）
    tools = await client.get_tools()

    # 创建 Agent
    agent = create_agent("openai:gpt-4o", tools=tools)

    # 用户一句话触发完整流程
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "审查 main.py 的代码"}]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

## tool 与 prompt 的选择

| 场景 | 选择 | 原因 |
|------|------|------|
| 需要读文件、查数据库、调 API | `@mcp.tool()` | 需要执行实际操作获取数据 |
| 封装固定的分析套路、报告模板 | `@mcp.prompt()` | 只需组装提示词，LLM 自己执行 |
| 先拿数据再按模板分析 | 两者配合 | tool 拿数据，prompt 告诉 LLM 怎么用数据 |

## 常见陷阱

### 参数名模糊导致 LLM 传错值

```python
# 问题：LLM 可能把文件名传给 data
@mcp.prompt()
def process(data: str) -> str:
    ...

# 修复：明确参数含义
@mcp.prompt()
def process(source_code: str) -> str:
    ...
```

### 忘记给可选参数设默认值

```python
# 问题：language 变成必填，LLM 必须猜用户用的语言
@mcp.prompt()
def review(code: str, language: str) -> str:
    ...

# 修复：给合理默认值
@mcp.prompt()
def review(code: str, language: str = "Python") -> str:
    ...
```

### prompt 模板中写了太多固定内容

```python
# 问题：模板太长，限制了 LLM 的灵活性
@mcp.prompt()
def review(code: str) -> str:
    """生成审查提示词"""
    return """你是一个代码审查专家，有20年经验...（500字背景描述）
请按照以下10个维度...（详细规则）
首先...其次...最后...（固定流程）
代码：{code}"""

# 修复：只写核心结构，给 LLM 留空间
@mcp.prompt()
def review(code: str) -> str:
    """生成审查提示词"""
    return f"审查以下代码，指出问题和改进建议：\n\n{code}"
```

## 相关

- [[langchain/modules/langchain-mcp/MCP服务器开发|MCP 服务器开发]] — FastMCP 框架完整用法
- [[langchain/modules/langchain-mcp/MCP协议概述|MCP 协议概述]] — 协议设计理念
- [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]] — 客户端连接多个 MCP 服务器
- [[langchain/modules/langchain-mcp/MCP与Agent中间件集成|MCP 与 Agent 中间件集成]] — Agent 中间件配合使用
