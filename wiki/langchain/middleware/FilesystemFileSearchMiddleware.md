---
tags: [langchain, middleware, filesystem, glob, grep, search]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#file-search
langchain_version: ">=1.0"
---

# FilesystemFileSearchMiddleware（文件系统搜索中间件）

## 一、定义与作用

`FilesystemFileSearchMiddleware` 给 Agent 提供两个文件搜索工具：
- **`glob_search`**：基于文件名模式匹配（`*.py`、`test_*.md`）
- **`grep_search`**：基于 regex 搜索文件内容

**核心价值**：让 Agent 能在大规模代码库/文档库中**快速定位**目标文件，而不需要逐个 `read_file`。

与 `FilesystemMiddleware` 的关系：
- `FilesystemMiddleware`：提供 ls/read/write/edit（文件操作）
- `FilesystemFileSearchMiddleware`：提供 glob_search/grep_search（文件搜索）
- **两者可以同时使用**，搜索 + 操作完美配合

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 代码库探索 | "找所有用到 `vectorstore` 的 Python 文件" |
| 文档搜索 | "找包含 '中间件' 的 Markdown 文件" |
| 日志分析 | "grep 错误日志" |
| 项目结构 | "列出所有 src/ 目录下的 .py 文件" |

## 三、参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `root_path` | `str` | **必填** | 搜索的根目录，所有操作相对此路径 |
| `use_ripgrep` | `bool` | `True` | 是否使用 ripgrep（不可用时回退到 Python regex） |
| `max_file_size_mb` | `int` | `10` | 搜索的最大文件大小（MB） |

### 提供的 2 个工具

| 工具名 | 作用 | 典型调用 |
|--------|------|----------|
| `glob_search` | Glob 文件名模式匹配 | `glob_search(pattern="**/*.py")` |
| `grep_search` | regex 搜索文件内容 | `grep_search(pattern="class.*Agent", path="src/")` |

## 四、使用方法

### 基础用法

```python
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/path/to/project",   # 必填
            max_file_size_mb=5,
        ),
    ],
)

# Agent 现在可以：
# - glob_search(pattern="**/*.py")  → 找所有 Python 文件
# - grep_search(pattern="def create_agent")  → 找定义 create_agent 的位置
```

### 配合 FilesystemMiddleware

```python
from langchain.agents.middleware import (
    FilesystemFileSearchMiddleware,
    FilesystemMiddleware,
)

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[],
    middleware=[
        # 先搜索
        FilesystemFileSearchMiddleware(root_path="/project"),
        # 再操作
        FilesystemMiddleware(),
    ],
)

# 典型工作流：
# 1. grep_search("TODO") → 找所有 TODO 注释
# 2. read_file(path) → 读具体文件
# 3. edit_file(path, old, new) → 改文件
```

### 关闭 ripgrep

```python
FilesystemFileSearchMiddleware(
    root_path="/project",
    use_ripgrep=False,   # 用 Python regex（更慢但更可移植）
)
```

## 五、注意事项

1. **`root_path` 必填**：不填会报错

2. **`max_file_size_mb` 默认 10MB**：超过此大小的文件不搜索（避免 grep 卡住）

3. **ripgrep 推荐安装**：比 Python regex 快 10-100 倍。Linux 通常预装；macOS 用 `brew install ripgrep`；Windows 用 `choco install ripgrep`

4. **路径安全**：grep 搜索默认在 `root_path` 下。如果不限制，Agent 可以搜整个文件系统。生产环境要明确 `root_path`

5. **正则转义**：`grep_search` 用的是 regex，特殊字符需要转义。如搜索 `(`, 要传 `\(`

6. **返回结果可能很多**：如果搜出来 1000 个文件，context 又会爆。Agent 最好配合 `head` 或 limit 参数

7. **隐藏文件不搜**（默认）：以 `.` 开头的文件被排除。如需搜索，设置 `include_hidden=True`（在工具调用时）

8. **编码问题**：搜索时默认按 UTF-8 解析。Windows 系统的 GBK 文件搜不到

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/FilesystemMiddleware]] — 文件操作
- [[langchain/middleware/ShellToolMiddleware]] — 命令执行
- [[langchain/middleware/SubAgentMiddleware]] — 派给子 Agent 做搜索
