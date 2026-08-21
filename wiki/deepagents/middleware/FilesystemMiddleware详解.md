---
tags: [deepagents, middleware, filesystem]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# DeepAgents FilesystemMiddleware

DeepAgents 的 `FilesystemMiddleware` 比 LangChain 内置版本多了**权限控制**和**后端路由**，是生产文件系统的核心。

## 一、与 LangChain FilesystemMiddleware 的差异

| | LangChain 内置 | DeepAgents |
|---|---|---|
| 包来源 | `langchain` | `deepagents.middleware.filesystem` |
| Backend 支持 | StateBackend | 6 种 Backend（State/Filesystem/Store/Composite/Sandbox） |
| 权限控制 | ❌ | ✅ `FilesystemPermission` |
| 工具裁剪 | ❌ | ✅ `tools=["read_file", "ls"]` |
| 多模态读取 | ❌ | ✅ 图片/PDF/音频直接读取 |

## 二、带权限的文件系统

```python
from deepagents import create_deep_agent
from deepagents.middleware import FilesystemPermission

agent = create_deep_agent(
    model=llm,
    backend=my_backend,
    permissions=[
        # 精确路径 + 访问级别
        FilesystemPermission(path="/workspace", access="read_write"),

        # 全局只读
        FilesystemPermission(path="/docs", access="read"),

        # 通配符
        FilesystemPermission(path="/tmp/*", access="read_write"),

        # 禁止访问
        FilesystemPermission(path="/secrets", access="deny"),
    ],
    # 其他路径默认不可访问
)
```

### 权限解析优先级

```
请求: write_file("/workspace/report.md", content="...")
  │
  ├── 匹配 /workspace (read_write) → ✅ 允许
  └── 未匹配任何规则 → ❌ 拒绝
```

## 三、限制暴露的工具

```python
from deepagents.middleware import FilesystemMiddleware

agent = create_deep_agent(
    model=llm,
    middleware=[
        FilesystemMiddleware(
            backend=my_backend,
            tools=["read_file", "ls", "glob", "grep"],
            # write_file 和 edit_file 不可用
        ),
    ],
)
```

## 四、多模态文件读取

```python
# Agent 调用 read_file("/report.pdf")
# → 自动返回文本内容（PDF 解析）
# 支持扩展名：.png .jpg .jpeg .gif .webp .heic .heif（图片）
#            .mp4 .mov .avi 等（视频）
#            .wav .mp3 .aiff 等（音频）
#            .pdf .ppt .pptx（文档）
```

## 五、相关页面

- [[deepagents/backend/后端概述]] — 6 种 Backend 选型
- [[deepagents/customization/自定义概述]] — 中间件覆盖
- [[langchain/middleware/FilesystemMiddleware]] — LangChain 基础版
