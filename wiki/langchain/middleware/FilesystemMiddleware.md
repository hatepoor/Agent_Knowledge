---
tags: [langchain, middleware, filesystem, deepagents]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#filesystem
langchain_version: ">=1.0"
deepagents_version: ">=0.6"
---

# FilesystemMiddleware（文件系统中间件）

## 一、定义与作用

`FilesystemMiddleware` 来自 `deepagents` 包（**不是** `langchain` 主包），它**自动给 Agent 配备 4 个文件系统工具**：`ls`、`read_file`、`write_file`、`edit_file`。

让 Agent 像一个开发者一样操作文件系统——能读文件、能写文件、能改文件。这是"Deep Agent"模式的基础能力。

**核心价值**：让 Agent 处理"需要多文件协作"的任务（如代码生成、长文档编辑、项目管理）成为可能。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 代码 Agent | 读源文件 → 改源文件 → 写测试文件 |
| 文档 Agent | 读参考资料 → 写新文档 → 编辑现有文档 |
| 数据处理 Agent | 读 CSV → 处理数据 → 写结果文件 |
| 长期项目 Agent | 跨多个会话维护一个项目目录 |

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `backend` | `Backend` | 可选，自定义后端（默认 `StateBackend`） |
| `system_prompt` | `string` | 可选，自定义系统提示 |
| `custom_tool_descriptions` | `dict` | 可选，文件系统工具的自定义描述 |
| `tools` | `list` | 可选，限制暴露的文件系统工具白名单 |

### 提供的 4 个工具

| 工具名 | 作用 | 典型调用 |
|--------|------|----------|
| `ls` | 列出目录 | `ls(directory="/")` |
| `read_file` | 读取文件 | `read_file(path="/notes.md")` |
| `write_file` | 写入文件 | `write_file(path="/output.txt", content="...")` |
| `edit_file` | 编辑文件（字符串替换） | `edit_file(path="/x.py", old_string="...", new_string="...")` |

### 短期 vs 长期文件系统

通过 `CompositeBackend` 将特定路径（如 `/memories/`）路由到 `StoreBackend` 可实现跨线程持久化：

```python
from deepagents.middleware import FilesystemMiddleware
from deepagents.backends import StateBackend, StoreBackend, CompositeBackend

agent = create_agent(
    model=llm,
    tools=[],
    middleware=[
        FilesystemMiddleware(
            backend=CompositeBackend(
                default=StateBackend(),           # / 路径用 StateBackend（短期）
                routes={
                    "/memories/": StoreBackend(), # /memories/ 用 StoreBackend（长期）
                },
            ),
        ),
    ],
)
```

## 四、使用方法

### 基础用法

```python
# pip install deepagents
from langchain.agents import create_agent
from deepagents.middleware import FilesystemMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[],   # 这里不用传 tools，FilesystemMiddleware 自带
    middleware=[FilesystemMiddleware()],
)

# Agent 现在可以用 ls/read_file/write_file/edit_file
result = agent.invoke({
    "messages": [{"role": "user", "content": "在 /notes.txt 写一段关于 Agent 的笔记"}],
})
```

### 自定义工具描述

```python
middleware = FilesystemMiddleware(
    custom_tool_descriptions={
        "write_file": "将内容写入项目内的文件。注意：只能写当前工作目录下的文件。",
        "edit_file": "编辑项目内文件。支持精确字符串替换。",
    }
)
```

## 五、注意事项

1. **来自 `deepagents` 包**，需要单独安装：`pip install deepagents`

2. **`StateBackend` 数据不持久**：默认状态下，文件存在 LangGraph state 里，线程结束就没了。需要持久化必须用 `StoreBackend` 或自定义后端

3. **路径安全**：Agent 可能会尝试访问系统敏感文件（`/etc/passwd`）。生产环境务必用 `backend` 限制根目录

4. **edit_file 是字符串替换**：如果 `old_string` 在文件中多处出现会失败。复杂编辑建议让 Agent 先 `read_file`，再生成完整新内容用 `write_file`

5. **不能与 ShellToolMiddleware 同时用**会带来安全风险——Agent 既能写文件又能执行命令

6. **大文件性能问题**：`read_file` 一次性读全文，超大文件（> 1MB）会撑爆上下文。建议先做分块

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/ShellToolMiddleware]] — 持久 shell 会话
- [[langchain/middleware/FilesystemFileSearchMiddleware]] — 文件搜索（Glob + Grep）
- [[langchain/middleware/SubAgentMiddleware]] — 子代理派发
