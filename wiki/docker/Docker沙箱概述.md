---
tags: [docker, sandbox, agent, concept]
created: 2026-07-28
updated: 2026-07-28
type: concept
---

# Docker 沙箱概述

生产环境中 Agent 执行代码的风险在于：模型生成的代码不可信任、权限边界模糊、资源无限制。Docker 沙箱是自建方案的最优解——容器化隔离 + Python SDK 控制。

## 一、核心问题

Agent 在执行 skill 或工具时面临三类风险：

| 风险类别 | 具体场景 | 沙箱对策 |
|----------|---------|---------|
| **文件系统** | `rm -rf /`、读取 `/etc/passwd`、覆盖宿主配置文件 | 只读根文件系统 + 选择性挂载 |
| **网络** | 反弹 shell、DDoS 攻击、数据外泄 | `network_mode="none"` 或白名单 |
| **资源** | 挖矿、内存泄漏、fork 炸弹 | CPU/内存/超时三重限制 |

## 二、整体架构

```
宿主机（Agent 进程）
  │
  ├── sandbox_execute(code)    ──→ 创建容器 → 执行 → 销毁
  ├── sandbox_read_file(path)  ──→ 挂载只读 /skills → 读取 → 销毁
  └── sandbox_run_skill(name)  ──→ 加载 skill 文件 → 执行 → 销毁
         │
         ▼ 所有 tool 通过 docker-py 管理
     ┌──────────────────────────────────┐
     │  Docker 容器（每次调用新建销毁）    │
     │  ├── /workspace（临时读写）        │
     │  ├── /skills（宿主挂载，只读）      │
     │  ├── CPU: 1 核 / 内存: 256M       │
     │  ├── 网络: 禁用（按需开启）         │
     │  └── 超时: 30s 自动 kill           │
     └──────────────────────────────────┘
```

## 三、三种实现方案

### 方案 A：ShellToolMiddleware + DockerExecutionPolicy

```python
from langchain.agents.middleware import ShellToolMiddleware, DockerExecutionPolicy

agent = create_agent(
    model="deepseek-v4-flash",
    middleware=[ShellToolMiddleware(
        execution_policy=DockerExecutionPolicy(image="python:3.11-slim"),
    )],
)
```

- **优势**：一行配置，Agent 自动获得 `execute` 工具
- **局限**：只能执行 shell 命令，不能自定义挂载 skills 目录，不能执行 Python 函数

### 方案 B：自建 DockerSandbox + @tool

```python
from sandbox_manager import sandbox

@tool
def sandbox_execute(code: str) -> str:
    return sandbox.execute(code).stdout
```

- **优势**：完全控制容器生命周期、自定义挂载点、限制 CPU/内存/网络、可执行 Python 函数
- **局限**：需要写 200 行管理代码

> 详见：[[docker/Docker_Sandbox核心类]]

### 方案 C：Deep Agents Sandbox Backend（托管）

```python
from deepagents import create_deep_agent
from deepagents.backends.langsmith import LangSmithSandbox

agent = create_deep_agent(backend=LangSmithSandbox(sandbox=ls_sandbox))
```

- **优势**：microVM 级别隔离、认证代理、快照/分叉、< 1s 启动
- **局限**：依赖 LangSmith 平台，非自建

## 四、选择决策

```
你的场景：
  ├── 个人开发/原型                     → 方案 A（ShellToolMiddleware）
  ├── 团队项目/需要控制技能目录          → 方案 B（自建 DockerSandbox）
  ├── SaaS 产品/多租户/需要审计日志      → 方案 B + 安全加固
  └── 企业级/合规要求/不想运维基础设施    → 方案 C（LangSmith Sandboxes）
```

## 五、方案 B 核心组件

自建方案由四个模块构成：

| 模块 | 文件 | 职责 |
|------|------|------|
| 配置 | `SandboxConfig` | 镜像、内存、CPU、超时、网络、挂载路径 |
| 执行 | `DockerSandbox.execute()` | 创建容器、执行代码、返回结果、销毁容器 |
| 封装 | `sandbox_execute` 等 `@tool` | 把沙箱暴露为 LangChain tool 供 Agent 调用 |
| 组装 | `create_agent(tools=[...])` | 把 tool 注入 Agent + system prompt 约束 |

> 详见：[[docker/Docker_Sandbox核心类]]

## 六、生产级关注点

| 关注点 | 说明 | 参考 |
|--------|------|------|
| 安全加固 | 网络隔离、镜像扫描、密钥管理、只读文件系统 | [[docker/沙箱安全加固]] |
| 性能优化 | 镜像优化、预热池、缓存策略、并发控制 | [[docker/沙箱性能优化]] |
| 监控日志 | 结构化日志、审计追踪、metrics | [[docker/沙箱监控与日志]] |
| 容错 | 重试策略、超时处理、graceful degradation | [[docker/沙箱错误处理与容错]] |
| 部署 | 生产 checklist、Dockerfile 模板、CI/CD | [[docker/沙箱生产部署清单]] |
| 集成 | 与 Agent 的完整生产级代码 | [[docker/沙箱与Agent集成实践]] |

## 七、相关页面

- [[docker/Docker_Sandbox核心类]] — SandboxConfig/DockerSandbox 完整实现
- [[docker/沙箱安全加固]] — 网络隔离、资源限制、密钥管理
- [[docker/沙箱与Agent集成实践]] — 完整生产级 Agent 组装
- [[langchain/middleware/ShellToolMiddleware]] — 内置 Shell 工具
- [[agent/skills/Skills实现与集成]] — Skills 在沙箱中的运行
