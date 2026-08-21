---
tags: [deepagents, sandbox, 沙箱, backend, 安全]
created: 2026-08-03
updated: 2026-08-03
type: concept
source_url: https://docs.langchain.com/oss/python/deepagents/customization
deepagents_version: ">=0.7"
---

# Sandbox 沙箱后端详解

Sandbox backend 让 Agent 在**隔离的容器/VM 环境**中执行代码和命令，Agent 自动获得 `execute` 工具。是"让 Agent 写代码、跑代码"的安全方案。

## 一、为什么需要沙箱

Agent 执行代码的场景（数据分析、脚本编写、测试运行）存在巨大风险：

| 风险 | 无沙箱 | 有沙箱 |
|------|--------|--------|
| 读取敏感文件（.env、SSH Key） | 可以 | 隔离环境内没有 |
| 执行 rm -rf / 等破坏命令 | 可以 | 只影响容器 |
| 向外网发送数据（数据泄露） | 可以 | 网络受限 |
| 挖矿/恶意软件 | 可以 | 容器销毁即消失 |

## 二、沙箱后端全览

| 后端 | 安装 | 环境类型 | 适用 |
|------|------|----------|------|
| `LangSmithSandbox` | `langsmith[sandbox]` | 托管容器 | LangSmith 生态用户 |
| `DaytonaSandbox` | `langchain-daytona` | 云容器 | 需要远程隔离环境 |
| `ModalSandbox` | `langchain-modal` | 云 VM | 已有 Modal 账户 |
| E2B 后端 | `langchain-e2b` | 云沙箱 | E2B 用户 |
| Runloop 后端 | `langchain-runloop` | 云沙箱 | Runloop 用户 |
| `LocalShellBackend` | 内置 | 本机 shell | ⚠️ 仅开发/CLI，**禁止生产** |

## 三、核心用法

### LangSmith Sandbox

```python
from deepagents.backends.langsmith import LangSmithSandbox
from langsmith.sandbox import SandboxClient

client = SandboxClient()
sandbox = client.create_sandbox()

agent = create_deep_agent(
    model=llm,
    backend=LangSmithSandbox(sandbox=sandbox),
    # Agent 自动获得 execute 工具 + 文件系统工具（ls/read/write 等）
)
```

### Daytona Sandbox

```python
from daytona import Daytona
from langchain_daytona import DaytonaSandbox

sandbox = Daytona().create()
backend = DaytonaSandbox(
    sandbox=sandbox,
    timeout=300,                 # 单条命令超时 300s
    sync_polling_interval=0.25,  # 轮询间隔
)

# 直接执行命令
result = backend.execute("echo hello")
print(result.output)  # "hello"

agent = create_deep_agent(model=llm, backend=backend)
```

### Modal Sandbox

```python
import modal
from langchain_modal import ModalSandbox

sandbox = ModalSandbox(modal.Sandbox.create(app=modal.App.lookup("your-app")))
result = sandbox.execute("echo hello")
print(result.output)

agent = create_deep_agent(model=llm, backend=sandbox)
```

## 四、execute 工具的行为

使用 Sandbox backend 后，Agent 的工具集中自动多出 `execute`：

```python
@tool
def execute(command: str, timeout: int = 120) -> str:
    """在沙箱环境中执行 shell 命令。命令在隔离容器中运行，不会影响宿主机器。"""
    ...
```

Agent 的典型用法：

```
用户: 帮我算一下这个数据集的统计量
Agent: write_file("data.csv", ...) → execute("python analyze.py") → 读结果回答
```

特点：
- 命令在容器/VM 中执行，**宿主机零风险**
- 每次会话沙箱独立（或按需复用）
- 支持超时控制（默认 120s，可配置）
- 输出和退出码返回给 Agent，供其判断

## 五、LocalShellBackend —— 禁止用于生产

```python
from deepagents.backends import LocalShellBackend

# ⚠️ 极度谨慎！不要在生产环境使用！
agent = create_deep_agent(
    model=llm,
    backend=LocalShellBackend(),
)
```

官方安全警告：

| 风险 | 说明 |
|------|------|
| 任意命令执行 | Agent 可以执行任何 shell 命令，权限等同你的用户 |
| 敏感文件读取 | 可读取 API Key、凭据、.env、SSH Key |
| 数据外泄 | 配合网络工具可 SSRF 外传数据 |
| 不可逆操作 | 文件修改和命令执行永久生效 |

**官方明确的使用场景**：本地开发 CLI（编程助手）、可信的个人开发环境、CI/CD。**禁止**：生产 Web 服务、多租户系统、处理不可信输入。

## 六、自建沙箱 vs 托管沙箱

| 方案 | 优点 | 缺点 |
|------|------|------|
| 托管沙箱（LangSmith/Daytona/E2B/Modal） | 零运维、安全可靠 | 付费、数据出网 |
| Docker 沙箱（自建） | 可控、免费 | 需自己写隔离逻辑、维护镜像 |
| LocalShellBackend | 零成本 | 不安全，禁止生产 |

知识库中 [[docker/Docker沙箱概述]] 介绍了自建 Docker 沙箱方案。

## 七、选择建议

```
你的场景：
  ├── 已有 LangSmith 生态        → LangSmithSandbox
  ├── 已有 Modal/Daytona/E2B 账户 → 对应后端
  ├── 本地开发 CLI（自己用）      → LocalShellBackend（可接受风险）
  ├── 自建 Docker 隔离            → 自定义 BaseSandbox 子类
  └── 生产服务/多租户             → 必须托管沙箱，禁止 LocalShellBackend
```

## 相关页面

- [[deepagents/backend/后端概述]] — 6 种后端总览
- [[deepagents/backend/Backend实战配置]] — 生产配置代码
- [[docker/Docker沙箱概述]] — 自建 Docker 沙箱方案
- [[docker/沙箱生产部署清单]] — 沙箱部署 checklist
- [[deepagents/tools/内置工具概述]] — execute 工具位置
