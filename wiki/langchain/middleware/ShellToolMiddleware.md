---
tags: [langchain, middleware, shell, sandbox, code-execution]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#shell-tool
langchain_version: ">=1.0"
---

# ShellToolMiddleware（Shell 工具中间件）

## 一、定义与作用

`ShellToolMiddleware` 给 Agent 提供一个**持久 shell 会话**作为工具——Agent 可以像开发者一样在命令行执行命令、读输出、跑脚本。

与一次性 `subprocess.run` 不同，**持久 shell** 保留了：
- 当前目录
- 环境变量
- 已定义的 shell 变量/函数
- 后台进程

**核心价值**：让 Agent 执行真实的代码（Python 脚本、Node、Shell 命令、CLI 工具）。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 代码 Agent | 跑测试、安装依赖、git 操作 |
| DevOps Agent | 执行 kubectl、docker、terraform 命令 |
| 数据分析 Agent | 跑 Python 脚本做计算 |
| 文件处理 Agent | 用 grep、sed、awk 处理文本 |

**安全风险**：
- Agent 可以执行 `rm -rf /`（如果不加限制）
- 必须用 `execution_policy` 限制环境

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `workspace_root` | `str \| Path \| None` | shell 会话的基础目录（省略则使用临时目录） |
| `startup_commands` | `tuple \| list \| str \| None` | 会话启动后顺序执行的命令 |
| `shutdown_commands` | `tuple \| list \| str \| None` | 会话关闭前执行的命令 |
| `execution_policy` | `BaseExecutionPolicy \| None` | 执行策略：`HostExecutionPolicy`（默认）/ `DockerExecutionPolicy` / `CodexSandboxExecutionPolicy` |
| `redaction_rules` | `tuple \| list[RedactionRule] \| None` | 在返回到模型前对命令输出进行脱敏的规则 |
| `tool_description` | `str \| None` | shell 工具描述的自定义覆盖 |
| `shell_command` | `Sequence[str] \| str \| None` | 用于启动持久会话的 shell 可执行文件（默认 `/bin/bash`） |
| `env` | `Mapping[str, Any] \| None` | shell 会话的环境变量 |

### 三种 execution_policy 对比

| 策略 | 隔离度 | 性能 | 适用场景 |
|------|--------|------|----------|
| `HostExecutionPolicy` | ❌ 无隔离 | 最高 | 信任的本地 Agent |
| `DockerExecutionPolicy` | ✅ 容器级 | 中等 | 通用生产环境 |
| `CodexSandboxExecutionPolicy` | ✅ OS 级沙箱 | 较高 | 高安全要求 |

## 四、使用方法

### 基础用法（宿主机执行）

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ShellToolMiddleware

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/tmp/agent_workspace",
        ),
    ],
)
```

### Docker 隔离执行

```python
from langchain_community.execution_policies import DockerExecutionPolicy

ShellToolMiddleware(
    workspace_root="/workspace",
    execution_policy=DockerExecutionPolicy(
        image="python:3.12-slim",
        # 限制资源
        mem_limit="512m",
        cpu_quota=50000,
    ),
)
```

### 启动命令 + 脱敏规则

```python
ShellToolMiddleware(
    startup_commands=["source venv/bin/activate", "export PYTHONPATH=/workspace"],
    redaction_rules=[
        # 匹配 API Key 模式并脱敏
        RedactionRule(pattern=r"sk-[a-zA-Z0-9]{32}", replacement="[API_KEY]"),
    ],
)
```

## 五、注意事项

1. **持久 shell 不能与 HITL 一起使用**：这是官方限制。HITL 需要中断恢复，但 shell 的"当前状态"很难跨中断保持

2. **HostExecutionPolicy 没有隔离**：Agent 可以执行 `rm -rf /` 等危险命令。生产环境**必须**用 Docker 或沙箱策略

3. **状态可能泄漏**：持久 shell 内的环境变量、文件等会持续存在。注意 `redaction_rules` 脱敏敏感输出

4. **大输出会撑爆上下文**：如果命令返回 1GB 日志，整个上下文会爆。用 `head` / `tail` 限制

5. **跨平台兼容**：`/bin/bash` 是默认；Windows 需要装 WSL 或用 `cmd` 路径

6. **适合执行**：
   - 跑测试（`pytest`）
   - 装包（`pip install`）
   - 跑 lint（`ruff check`）
   - 文件操作（`grep`、`sed`）

7. **不适合执行**：
   - 长任务（> 5 分钟）— Agent 等不了
   - 交互式命令（`vim`、`less`）— 无 TTY
   - GUI 程序

8. **强烈建议配合 PIIMiddleware**：脱敏命令输出中的密码、token

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/FilesystemMiddleware]] — 文件系统（不执行命令）
- [[langchain/middleware/FilesystemFileSearchMiddleware]] — 文件搜索
- [[langchain/middleware/PIIMiddleware]] — 输出脱敏
