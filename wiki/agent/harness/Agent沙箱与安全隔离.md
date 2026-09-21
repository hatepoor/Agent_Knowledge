---
tags: [concept, agent, 沙箱, sandbox, 安全隔离, e2b, 安全]
created: 2026-09-10
updated: 2026-09-10
type: concept
source_url: https://developers.openai.ac.cn/codex/concepts/sandboxing
---

# Agent 沙箱与安全隔离

> Harness 工程层话题。本文讲 **Agent 沙箱的为什么、有哪些层次、主流产品怎么做、怎么配置**；安全审核与合规视角见 [[rag/11_安全审核与日志/安全与合规|安全与合规]]；Agent 架构总览见 [[agent/harness/Agent_六种架构|Agent 六种架构]]。

## 概述

Agent 要干活就必须"动手"——写文件、跑命令、装依赖、访问网络。但 AI 生成的代码/命令**不能直接在生产环境或用户本机裸跑**，否则可能误删文件、越权访问、污染环境、外发数据、耗尽资源。沙箱给 Agent 一个"**受限的游乐场**"——它能在里面折腾，但折腾不到外面的真实系统。

沙箱是 Agent 工程中**不可或缺的基础设施**，和 [[agent/memory/Agent_Memory|记忆系统]]、[[agent/harness/Function_Call|工具调用]] 并列，构成 Agent 安全执行的三大基座。

---

## 一、为什么 Agent 必须有沙箱

| 风险 | 例子 | 严重程度 |
|------|------|----------|
| **误删文件** | Agent 想删临时文件，`rm -rf /tmp/*` 写错路径 → 删了整个项目 | 高 |
| **越权访问** | Agent 读取 SSH key、密码文件、浏览器历史、环境变量 | 高 |
| **依赖污染** | Agent `pip install` 一堆东西把系统环境搞崩 | 中 |
| **网络外发** | Agent 不小心把本地数据 POST 到外部服务 | 高 |
| **资源耗尽** | Agent 写了死循环把 CPU/内存占满 | 中 |
| **持久化残留** | Agent 创建的后门/定时任务/开机自启留在系统里 | 高 |
| **命令注入** | 用户输入里藏了恶意命令，Agent 拼进去直接执行 | 高 |

**一句话**：没有沙箱的 Agent = 给 AI 管理员权限——迟早出事。

---

## 二、五种隔离层次（从弱到强）

| 层级 | 隔离方式 | 启动速度 | 安全性 | 一句话理解 | 代表 |
|------|----------|----------|--------|-----------|------|
| 1. 策略/审批 | 白名单 + 人工审批 | 无额外延迟 | 低 | "每一步先问你同不同意" | Codex approval、Trae 审批 |
| 2. OS-level | Seatbelt / bubblewrap / seccomp / Landlock | 毫秒级 | 中 | "用操作系统的权限机制把它关起来" | Claude Code、Codex CLI、OpenInterpreter |
| 3. 容器 | Docker / containerd / 根文件系统隔离 | 较快 | 中-高 | "给它一个独立的小操作系统" | 本地开发、部分 sandbox client |
| 4. MicroVM | Firecracker / gVisor | ~200ms | 高 | "给它一个轻量虚拟机，内核都隔离" | E2B、Daytona |
| 5. 完整 VM | KVM / QEMU / Hyper-V | 秒级 | 最高 | "给它一台完整的虚拟电脑" | TraeWork Windows Work/Design |

**选型直觉**：越往下越安全，但越慢越重、成本越高。编码 Agent 大多用 **OS-level + 审批**（快、够用）；云端多租户 Agent 服务用 **MicroVM 或容器**（强隔离）。

---

## 三、两种核心架构模式

### 模式 1：Isolate the Tool（只隔离工具）

```
┌─────────────────────────────┐
│  宿主机（你的电脑/服务器）    │
│  ┌──────────┐               │
│  │ Agent 主进程│             │  ← Agent 本身在宿主机
│  └────┬──────┘               │
│       │ 调用工具              │
│  ┌────▼──────┐               │
│  │ OS-level  │               │  ← 只有代码执行被沙箱包着
│  │  沙箱      │               │
│  └───────────┘               │
└─────────────────────────────┘
```

- Agent 主进程运行在宿主机，只有**工具调用**（代码执行、文件读写、命令执行）被沙箱包着
- 启动快、开销小、适合本地开发环境
- 代表：Codex CLI、Claude Code、TraeWork 本地模式、OpenInterpreter

### 模式 2：Isolate the Agent（整个 Agent 隔离）

```
┌─────────────────────────────┐
│  宿主机 / 云端               │
│  ┌───────────────────────┐  │
│  │  MicroVM / 容器        │  │  ← 整个 Agent 环境在里面
│  │  ┌─────────────────┐  │  │
│  │  │ Agent + 工具 + 文件 │  │  │
│  │  └─────────────────┘  │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

- 整个 Agent 运行环境（文件、命令、网络、依赖、状态）都在隔离计算单元内
- 安全强、隔离彻底、适合云端多租户 / 不可信代码执行
- 代表：E2B、TraeWork 云端、Codex 云端、OpenAI Agents SDK sandbox

**选型口诀**：
- 本地编码 Agent → 模式 1（OS-level + 审批），快且够用
- 云端 SaaS / 多租户 / 不可信代码 → 模式 2（MicroVM/容器），安全第一

---

## 四、主流产品沙箱配置对比

### Codex CLI（OpenAI 官方编码 Agent）

**三层防御**：沙盒模式 + 审批策略 + OS 级隔离。

**沙盒模式**（技术边界）：

| 模式 | 文件权限 | 网络 | 适用场景 |
|------|----------|------|----------|
| `read-only` | 只读检查文件 | 关 | 纯查询场景 |
| `workspace-write`（默认） | 可写工作区目录 | 默认关 | 常规编码 |
| `danger-full-access` | 完全放开 | 开 | 仅信任环境 |

**OS 级隔离实现**：
- **macOS**：Apple Seatbelt / `sandbox-exec`（苹果系统内置沙箱框架）
- **Linux / WSL2**：bubblewrap（用户命名空间隔离）+ seccomp（系统调用过滤）
- **Windows**：Windows 原生沙箱机制

**审批策略**：Agent 想做沙箱外的操作（访问外部文件、联网、执行高危命令）→ 暂停 → 问用户 → 用户批准/拒绝/加白名单。

### TraeWork

**双模式架构**：本地任务用 OS-level，云端任务用容器/虚拟化。

| 平台 | 模式 | 沙箱形态 |
|------|------|----------|
| macOS Code | 本地 | `sandbox-exec` 受限环境 |
| Windows Code | 本地 | 受限执行环境 |
| Windows Work / Design | 云端 | **完全隔离的虚拟化运行环境** |
| 云端任务 | 云端 | 独立沙箱容器 |

**核心规则**：
- 可写：项目目录 / 临时目录 / 缓存目录
- 只读/受保护：根目录、`.vscode`、系统目录
- 高风险命令（如 `rm -rf`）会被拦截，用户可选跳过/白名单/本次沙箱内运行

### Claude Code（Anthropic 官方编码 Agent）

思路和 Codex 几乎一致：OS-level sandbox，不搞 Docker/VM。

- **macOS**：Seatbelt / `sandbox-exec`
- **Linux / WSL2**：bubblewrap + 用户命名空间
- **网络**：所有流量走代理 Unix socket，默认黑名单拦截，仅允许批准的域名
- **权限模式**：`default` / `acceptEdits` / `plan` / `dontAsk` / `bypassPermissions`
- **`PreToolUse` hook**：可写脚本程序化决定批准还是拒绝每个工具调用

### OpenInterpreter（开源编码 Agent）

和 Codex 同架构：三种沙盒模式 + 平台级 OS 隔离 + 审批策略。额外支持 **OpenSandbox 后端**（可选），把 Python/Shell/JS 放进隔离容器执行。

### E2B（沙箱即服务）

**重量级方案——Firecracker MicroVM**，每个 sandbox 是一个独立 Linux 虚拟机：

| 组件 | 说明 |
|------|------|
| 内核 | 每个 sandbox 独立 Linux kernel |
| 内存 | 与宿主机及其他 sandbox 完全隔离 |
| 文件系统 | 基于 snapshot 的 copy-on-write rootfs |
| 网络 | 独立虚拟网络接口 |
| 启动速度 | 从 snapshot 恢复 < 200ms |
| 内部进程 | `envd` 守护进程 + Jupyter kernel |

**适合场景**：云端多租户 Agent 服务、需要强隔离的 AI 代码执行。

**Snapshot 架构**（为什么快）：
1. 先构建 template，安装好所有依赖
2. 把完整内存+磁盘状态保存为 snapshot
3. 新 sandbox 从 snapshot 恢复，不是从零启动
4. 基础 snapshot 只读共享，每个 sandbox 只保存自身变更

### OpenAI Code Interpreter（高级数据分析）

- OpenAI 托管的隔离环境（底层技术未公开，推测容器级）
- 只有 Python 运行时，100MB 文件限制
- **默认无外网访问**（要连数据库得先上传数据文件）
- 模型生成代码 → OpenAI 执行 → 返回 stdout/stderr/文件 → 模型继续

### 对比总表

| 方案 | 隔离机制 | 启动 | 网络控制 | 文件系统 | 典型使用 |
|------|----------|------|----------|----------|----------|
| OpenAI Code Interpreter | 托管隔离容器 | API 延迟 | 无外网 | OpenAI 托管文件 | 数据分析、图表、Python |
| Codex CLI | Seatbelt / bubblewrap + seccomp | 毫秒级 | 默认关 | 工作区受限 | 本地编码 Agent |
| Claude Code | Seatbelt / bubblewrap | 毫秒级 | 代理白名单 | 工作区写入 | 本地编码 Agent |
| OpenInterpreter | Seatbelt / bubblewrap / seccomp | 毫秒级 | 默认关，可配 | workspace-write | 开源编码 Agent |
| TraeWork 本地 | sandbox-exec / 受限环境 | 毫秒级 | 平台管理 | 项目目录可写 | 本地辅助开发 |
| TraeWork 云端/Work | 容器或虚拟化环境 | 秒级 | 平台管理 | 会话工作目录 | 安全执行 |
| E2B | Firecracker microVM + snapshot | <200ms | 独立虚拟网络 | COW rootfs | Agent 沙箱即服务 |

---

## 五、真实开发中怎么给 Agent 配沙箱

### 方案 A：本地开发用 OS-level（最快上手）

```python
# 伪代码：沙箱执行器
import subprocess
import os

class SandboxedExecutor:
    def __init__(self, workspace_dir: str, sandbox_mode: str = "workspace-write"):
        self.workspace = os.path.abspath(workspace_dir)
        self.mode = sandbox_mode

    def _get_sandbox_prefix(self) -> str:
        """根据平台选择沙箱前缀命令"""
        import sys
        if sys.platform == "darwin":
            # macOS: sandbox-exec + profile
            profile = self._build_seatbelt_profile()
            return f"sandbox-exec -f {profile}"
        elif sys.platform.startswith("linux"):
            # Linux: bubblewrap
            return f"bwrap --dev-bind / / --bind {self.workspace} {self.workspace} --ro-bind /usr /usr"
        else:
            # Windows: 受限执行环境
            return ""

    def run_command(self, cmd: str, timeout: int = 30) -> dict:
        sandbox_prefix = self._get_sandbox_prefix()
        full_cmd = f"{sandbox_prefix} {cmd}"

        result = subprocess.run(
            full_cmd, shell=True,
            capture_output=True, text=True,
            cwd=self.workspace,
            timeout=timeout,
        )

        return {
            "stdout": result.stdout,
            "stderr": result.stderr,
            "exit_code": result.returncode,
        }
```

### 方案 B：生产服务用 E2B（强隔离）

```python
# pip install e2b
from e2b import Sandbox

# 从 template 创建沙箱（从 snapshot 恢复，<200ms）
sandbox = Sandbox(template="python-data-science")

try:
    # 往沙箱里写文件
    sandbox.files.write("/home/user/analyze.py", code_content)
    sandbox.files.write("/home/user/data.csv", csv_content)

    # 在沙箱里执行代码，带超时
    output = sandbox.process.start_and_wait(
        "python /home/user/analyze.py",
        timeout=60,  # 秒级超时，防止死循环
    )

    print("stdout:", output.stdout)
    print("stderr:", output.stderr)

    # 读取生成的文件
    result_file = sandbox.files.read("/home/user/result.json")
finally:
    sandbox.close()  # 关闭后沙箱完全销毁
```

### 方案 C：LangGraph Agent 接入沙箱工具

```python
from langchain_core.tools import tool
from e2b import Sandbox

@tool
def run_python_code(code: str) -> str:
    """在隔离沙箱中执行 Python 代码，返回 stdout 和 stderr。
    所有代码执行都必须通过此工具，禁止直接在本地运行。
    """
    with Sandbox() as sb:
        sb.files.write("/tmp/run.py", code)
        result = sb.process.start_and_wait(
            "python /tmp/run.py",
            timeout=30,
        )
        return f"stdout:\n{result.stdout}\nstderr:\n{result.stderr}"

# 把这个工具注册给 Agent
tools = [run_python_code, search_tool, ...]
agent = create_react_agent(llm, tools)
```

---

## 六、Windows 平台沙箱实现

Windows 上有**两种不同量级**的沙箱，用途完全不同。编码 Agent 常用的是进程级方案，而不是完整 VM。

### 6.1 Windows 沙箱的两种形态

| 形态 | 级别 | 隔离强度 | 启动速度 | 适合什么 |
|------|------|----------|----------|----------|
| **Windows Sandbox**（系统自带） | 完整轻量 VM | 高（独立内核） | 秒级 | 跑不可信软件、完整测试环境 |
| **AppContainer / Job Object**（进程级） | 进程级隔离 | 中 | 毫秒级 | Agent 工具调用、单进程限制 |

### 6.2 Windows Sandbox（系统自带轻量 VM）

基于 Hyper-V 的**一次性微型虚拟机**——打开是干净的 Windows 桌面，关闭后全部销毁，下次打开又是全新的。

**前置条件**：
- Windows 10 Pro/Enterprise 1903+ 或 Windows 11 Pro/Enterprise
- CPU 开启虚拟化（BIOS 中 VT-x / AMD-V）
- 至少 4GB 内存（建议 8GB+）

**启用方式**：

```powershell
# 以管理员身份运行 PowerShell
Enable-WindowsOptionalFeature -FeatureName "Containers-DisposableClientVM" -All -Online
```

或图形化：`Win + R` → 输入 `optionalfeatures` → 勾选"Windows 沙箱" → 重启。

**自定义配置（.wsb 文件）**：

双击 `.wsb` 文件即可按配置启动沙箱。

```xml
<!-- StrictSandbox.wsb -->
<Configuration>
  <vGPU>Disable</vGPU>                  <!-- 禁用 GPU 共享 -->
  <Networking>Disable</Networking>       <!-- 禁用网络 -->
  <MappedFolders>
    <MappedFolder>
      <HostFolder>C:\Projects\my-app</HostFolder>
      <SandboxFolder>C:\workspace</SandboxFolder>
      <ReadOnly>true</ReadOnly>          <!-- 只读挂载项目目录 -->
    </MappedFolder>
  </MappedFolders>
  <ClipboardRedirection>Disable</ClipboardRedirection>
  <PrinterRedirection>Disable</PrinterRedirection>
</Configuration>
```

**适合场景**：跑完全不可信的代码、高风险操作；**不适合**编码 Agent 日常工具调用（太慢太重）。

### 6.3 进程级沙箱：AppContainer + Job Object

这是**编码 Agent 在 Windows 上的实际方案**——给单个进程套上限制，而非启动完整 VM。

#### AppContainer

微软为 UWP 设计的安全沙箱，每个 AppContainer 进程有独立 SID（安全标识符），默认几乎没有权限——不能读写文件、不能联网、不能访问硬件，要什么权限得显式声明。

| 特性 | 说明 |
|------|------|
| 粒度 | 进程级 |
| 启动速度 | 毫秒级 |
| 默认权限 | 最小权限，按需授予 |
| 系统要求 | Windows 8+ 自带 |

#### Job Object

Windows 内核对象，用来**管理一组进程的资源限制**：

| 能限制什么 | 作用 |
|-----------|------|
| 内存上限 | 防止进程吃爆内存 |
| CPU 时间 / 亲和性 | 限制 CPU 使用 |
| 进程数 | 防止 fork 炸弹 |
| 作业关闭时自动终止所有子进程 | 防止残留进程 |
| 磁盘 IO 速率 | 限制磁盘占用 |

**AppContainer 管"能访问什么"，Job Object 管"能用多少资源"**，两者配合就是完整的进程沙箱。

#### C# 实现示例（Job Object 封装）

```csharp
using System;
using System.Runtime.InteropServices;
using System.Diagnostics;

class SandboxedProcess
{
    [DllImport("kernel32.dll", SetLastError = true)]
    static extern IntPtr CreateJobObject(IntPtr lpJobAttributes, string lpName);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool SetInformationJobObject(
        IntPtr hJob,
        JOBOBJECTINFOCLASS JobObjectInfoClass,
        IntPtr lpJobObjectInfo,
        uint cbJobObjectInfoLength);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool AssignProcessToJobObject(IntPtr hJob, IntPtr hProcess);

    public static Process StartSandboxed(string exe, string args, string workingDir)
    {
        // 1. 创建 Job Object
        IntPtr job = CreateJobObject(IntPtr.Zero, "AgentSandbox_" + Guid.NewGuid());

        // 2. 设置限制：512MB 内存上限 + 60s CPU 时间限制 + 关闭时杀死子进程
        var info = new JOBOBJECT_EXTENDED_LIMIT_INFORMATION
        {
            BasicLimitInformation = new JOBOBJECT_BASIC_LIMIT_INFORMATION
            {
                LimitFlags = JOB_OBJECT_LIMIT_PROCESS_MEMORY |
                             JOB_OBJECT_LIMIT_JOB_TIME |
                             JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE,
                PerJobUserTimeLimit = 60 * 1000 * 10000, // 60秒
            },
            ProcessMemoryLimit = new UIntPtr(512 * 1024 * 1024)  // 512 MB
        };

        // 3. 启动进程
        var psi = new ProcessStartInfo(exe, args)
        {
            WorkingDirectory = workingDir,
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
        };
        var proc = Process.Start(psi);

        // 4. 把进程放进 Job（进程及其所有子进程都会被限制）
        AssignProcessToJobObject(job, proc.Handle);

        return proc;
    }
}
```

### 6.4 Windows 上的实际主流方案：WSL2 + Linux 沙箱

**Codex CLI、OpenInterpreter、Claude Code 等产品在 Windows 上的默认路线**——不在 Windows 原生沙箱上折腾，直接用 WSL2 里的 Linux 工具链（bubblewrap + seccomp）。

```
Windows 主机
  └── WSL2（轻量 VM）
        └── Linux 沙箱（bubblewrap + seccomp）
              └── Agent 执行的命令
```

**优点**：
- Linux 沙箱生态成熟（bubblewrap 广泛使用）
- 一套代码跑 macOS / Linux / Windows（WSL2），跨平台一致
- WSL2 本身就是隔离的，相当于双层防护

**缺点**：
- 需要用户安装 WSL2
- 跨文件系统访问有性能损耗

**启用 WSL2**：

```powershell
# 以管理员身份运行 PowerShell
wsl --install
# 安装完成后重启电脑
wsl --status   # 验证
```

### 6.5 Windows Agent 沙箱选型指南

按推荐优先级排序：

| 优先级 | 方案 | 适合场景 | 实现难度 |
|--------|------|----------|----------|
| 1 | **WSL2 + bubblewrap** | 编码 Agent、日常工具调用 | 中（依赖 WSL2） |
| 2 | **Job Object + 权限审批** | 纯 Windows 原生、不能装 WSL2 | 高（要自己写封装） |
| 3 | **Windows Sandbox（VM）** | 跑完全不可信的代码 | 低（系统自带） |
| 4 | **Docker Desktop** | 服务端、容器化工作流 | 中 |

**经验**：如果你在做本地编码 Agent，优先走 WSL2 路线（生态成熟、跨平台一致）；纯原生 Windows 方案只有在用户不能装 WSL2 时才做备选。

### 6.6 快速环境检查

```powershell
# 检查 Windows 版本（需要 Pro/Enterprise）
(Get-ComputerInfo).OsName
(Get-ComputerInfo).OsVersion

# 检查虚拟化是否开启
Get-ComputerInfo -Property "HyperV*"

# 检查 Windows Sandbox 是否已启用
Get-WindowsOptionalFeature -FeatureName "Containers-DisposableClientVM" -Online

# 检查 WSL2 是否安装
wsl --status
```

### 6.7 工程设计：四层防御架构

实际产品中，沙箱从来不是"一个东西"，而是**多层叠加的防御体系**。任何一层被绕过，下一层还能拦住。

```
┌─────────────────────────────────────────┐
│  Layer 4: 审批层（用户/策略）             │  越界操作先问用户
│  - 权限分级策略                           │
│  - 高危命令拦截 + 白名单                   │
├─────────────────────────────────────────┤
│  Layer 3: 文件/网络边界                  │  只能看/改允许的东西
│  - 文件系统白名单（工作区目录）            │
│  - 网络代理/域名白名单                    │
├─────────────────────────────────────────┤
│  Layer 2: 资源限制（Job Object）         │  用多少资源有上限
│  - 内存 / CPU / 进程数 / 超时             │
├─────────────────────────────────────────┤
│  Layer 1: 执行器（进程隔离）             │  在哪跑
│  - WSL2 + bubblewrap（首选）              │
│  - 或 Win32 Job Object（原生备选）        │
└─────────────────────────────────────────┘
```

#### 核心组件 1：执行器抽象（Executor）

执行器是沙箱的核心抽象——对外暴露统一接口，内部可以切换不同后端。Agent 工具层只认识接口，不认识具体后端。

```python
from abc import ABC, abstractmethod

class SandboxExecutor(ABC):
    """沙箱执行器抽象基类——所有后端都实现这个接口"""

    @abstractmethod
    def run_command(self, cmd: str, cwd: str, timeout: int = 30) -> dict:
        """在沙箱中执行命令，返回 {stdout, stderr, exit_code}"""
        ...

    @abstractmethod
    def read_file(self, path: str) -> str:
        """读取沙箱内文件（受权限策略约束）"""
        ...

    @abstractmethod
    def write_file(self, path: str, content: str) -> None:
        """写入沙箱内文件（受权限策略约束）"""
        ...
```

**两个后端实现**：

| 后端 | 适用场景 | 实现方式 |
|------|----------|----------|
| **WslExecutor** | 用户装了 WSL2（大多数情况） | `wsl -d agent-sandbox -- bwrap ... cmd` |
| **Win32Executor** | 纯 Windows 原生、不能装 WSL2 | Job Object + 权限审批 + 文件白名单 |

**为什么要抽象成接口**：
- 用户环境不一样，有的装了 WSL2 有的没装
- 可以按用户环境自动选择最佳后端
- 以后加 Docker / Windows Sandbox 后端也不用改上层代码

#### 核心组件 2：权限策略（Policy）

```python
from enum import Enum

class SandboxMode(Enum):
    READ_ONLY = "read_only"          # 只能读
    WORKSPACE_WRITE = "workspace"    # 只能写工作区
    FULL_ACCESS = "full"             # 完全放开（信任环境）

class PermissionPolicy:
    def __init__(self, mode: SandboxMode, workspace_dir: str):
        self.mode = mode
        self.workspace = os.path.abspath(workspace_dir)

    def can_read(self, path: str) -> bool:
        return self._is_inside_workspace(path) or self._is_temp_dir(path)

    def can_write(self, path: str) -> bool:
        if self.mode == SandboxMode.READ_ONLY:
            return False
        if self.mode == SandboxMode.WORKSPACE_WRITE:
            return self._is_inside_workspace(path)
        return True  # full

    def can_access_network(self) -> bool:
        return self.mode == SandboxMode.FULL_ACCESS

    def _is_inside_workspace(self, path: str) -> bool:
        abs_path = os.path.abspath(path)
        return abs_path.startswith(self.workspace + os.sep)
```

#### 核心组件 3：审批引擎（Approval）

```python
class ApprovalResult(Enum):
    ALLOW = "allow"           # 允许
    DENY = "deny"             # 拒绝
    ASK_USER = "ask_user"     # 需要问用户

class ApprovalEngine:
    def __init__(self, policy: PermissionPolicy):
        self.policy = policy
        self.allowed_commands = set()   # 用户本次会话已批准的命令

    def check_command(self, cmd: str) -> ApprovalResult:
        # 1. 先看白名单（本次已批准）
        if cmd in self.allowed_commands:
            return ApprovalResult.ALLOW

        # 2. 高危命令强制拦截并询问
        if self._is_dangerous(cmd):
            return ApprovalResult.ASK_USER

        # 3. 按权限模式判断
        if self.policy.mode == SandboxMode.FULL_ACCESS:
            return ApprovalResult.ALLOW

        # 4. 写操作需要审批
        if self._is_write_operation(cmd):
            return ApprovalResult.ASK_USER

        return ApprovalResult.ALLOW

    def _is_dangerous(self, cmd: str) -> bool:
        patterns = [
            r"rm\s+-rf\s+/",              # 删根目录
            r"format\s+",                  # 格式化
            r"reg\s+delete",               # 删注册表
            r"curl.*\|.*bash",             # 管道执行远程脚本
            r"powershell.*Remove-Item.*-Recurse",
        ]
        return any(re.search(p, cmd, re.IGNORECASE) for p in patterns)
```

#### 核心组件 4：资源限制

Windows 上用 **Job Object** 管资源（最稳）；WSL2 后端也可以用 cgroup / systemd-run 限制。

| 限制项 | 作用 | 默认值 |
|--------|------|--------|
| `ProcessMemoryLimit` | 单个进程最大内存 | 512 MB |
| `PerJobUserTimeLimit` | CPU 时间上限（防死循环） | 60 秒 |
| `ActiveProcessLimit` | 最多子进程数（防 fork 炸弹） | 50 个 |
| `KILL_ON_JOB_CLOSE` | 关闭时自动杀掉所有子进程 | 必开 |

### 6.8 WSL2 后端工程设计（主流方案）

这是目前实际产品中**最常用、最成熟**的方案。

#### 架构

```
Windows 宿主（Agent 主进程）
  │
  ├── 调用：wsl -d agent-sandbox -- /usr/local/bin/sandboxed-run <cmd>
  │
  └── WSL2 发行版（轻量 VM，专用沙箱）
        │
        ├── bubblewrap（进程级隔离）
        │     ├── 文件系统：只挂载工作区 + /tmp + /usr
        │     ├── 网络：默认关闭，需要时走代理
        │     └── 用户：非 root 用户运行
        │
        └── seccomp（系统调用过滤）
              └── 禁用危险 syscall（mount, ptrace, etc.）
```

#### 关键设计点

**1. 专用 WSL 发行版**
- 不是用用户默认的 Ubuntu，而是**导入一个专用的、最小化的发行版**（命名如 `agent-sandbox`）
- 里面预装 Python、Node、git 等 Agent 需要的工具
- 坏了可以直接卸载重装，不影响用户自己的 WSL 环境

```powershell
# 导入专用沙箱发行版
wsl --import agent-sandbox C:\WSL\agent-sandbox .\sandbox-rootfs.tar.gz
```

**2. 文件系统挂载策略**
- 工作区目录：从 Windows 挂载进去，**读写**（`/mnt/workspace`）
- 系统目录：只读挂载（`/usr`、`/lib`）
- 临时目录：沙箱内独立的 `/tmp`，用完清理
- **禁止挂载**：Windows 根目录、用户家目录、`C:\Windows`

**3. 网络控制**
- 默认**完全关闭网络**（bubblewrap `--unshare-net`）
- 需要联网时，走白名单代理（只允许 API 域名）
- 包管理、大文件下载走专用代理节点

**为什么这是最优方案**：

| 优势 | 说明 |
|------|------|
| 跨平台一致 | macOS/Linux/Windows 同一套 bubblewrap 代码 |
| 生态成熟 | Linux 沙箱工具链比 Windows 原生丰富得多 |
| 隔离足够 | WSL2 本身是 VM，再加 bubblewrap，双层防护 |
| 性能可接受 | WSL2 文件系统性能不错；跨系统访问有损耗但可接受 |

### 6.9 Windows 原生后端设计（备选方案）

当用户不能装 WSL2 时才走这个。实现复杂度高很多。

#### 架构

```
Windows 宿主（Agent 主进程）
  │
  ├── 审批层（ApprovalEngine）—— 先判断能不能执行
  │
  ├── Job Object —— 资源限制（内存/CPU/进程数/超时）
  │     └── 目标进程 + 所有子进程
  │
  ├── 文件系统边界 —— NTFS 权限 / 完整性级别限制
  │
  └── 网络控制 —— WFP / WinDivert 过滤（可选）
```

#### 三个难点（为什么原生方案难做）

**1. 文件系统边界难搞**
- macOS 的 Seatbelt 原生支持"只能读写某目录"的声明式配置
- Windows 没有等价的简洁方案，可选路线：
  - **低完整性级别**（Low Integrity Level）—— 粒度粗
  - **NTFS 权限 + 重解析点** —— 配置复杂
  - **Minifilter 驱动** —— 最稳但要内核开发，成本极高
- 大多数产品选择：**不强依赖 OS 级文件隔离，靠审批层兜底**

**2. 网络控制难搞**
- 原生 Windows 做进程级网络过滤需要 WFP 或 WinDivert
- 实现复杂度远高于 Linux 的 `iptables` / `--unshare-net`
- 常见做法：**不做进程级网络隔离，靠域名白名单代理 + 审批层**

**3. AppContainer 限制多**
- AppContainer 是为 UWP 设计的，和传统 Win32 程序兼容性不好
- 很多命令行工具在 AppContainer 里跑不起来
- 实际产品很少用，更多用 Job Object + 审批的组合

### 6.10 与 Agent 系统的集成

```
LangGraph Agent
    │
    ▼
  Tool Layer（工具层）
    │
    ├── run_shell_command(cmd)   ← Agent 调用这个工具
    │     │
    │     ▼
    │   SandboxExecutor  ← 统一接口
    │     │
    │     ├── WslExecutor（优先）  ← 检测到 WSL2 就用这个
    │     └── Win32Executor（备选） ← 没装 WSL2 就用这个
    │
    ├── read_file(path)
    ├── write_file(path, content)
    └── run_python(code)
```

**集成要点**：
1. **工具层只认识接口，不认识具体后端**——换后端不影响 Agent 逻辑
2. **审批在工具入口处做**——还没进沙箱就先判断要不要问用户
3. **沙箱错误向上抛给 Agent**——沙箱拦截了、命令超时了，都返回给 Agent 让它自己决定怎么处理
4. **日志完整记录**——每次执行的命令、参数、结果、审批状态，全部记审计日志

### 6.11 生产环境的三个额外要求

**1. 热切换后端**
- 启动时检测用户环境 → 自动选择最佳后端
- WSL2 不可用时自动降级到 Win32Executor
- 用户可以在设置里手动切换

**2. 沙箱健康检查**
- 启动时跑一个测试命令，确认沙箱正常工作
- WSL2 后端要检查发行版是否存在、是否损坏
- 异常时给出清晰的修复指引

**3. 清理机制**
- 每次会话结束清理临时文件
- WSL2 后端定期 `wsl --shutdown` 释放资源
- Job Object 确保进程树完全终止，不留孤儿进程

### 6.12 设计决策树

```
你在做 Windows Agent 沙箱吗？
    │
    ├─→ 用户可以装 WSL2 吗？
    │     ├─ 可以 → 【WSL2 + bubblewrap + seccomp】（推荐，生态成熟）
    │     └─ 不行 → 看下面
    │
    ├─→ 需要多强的隔离？
    │     ├─ 很强（跑不可信代码） → Windows Sandbox VM / Docker Desktop
    │     ├─ 中等（编码 Agent） → Job Object + 权限审批 + 文件白名单
    │     └─ 很弱（信任环境） → 只做审批 + 日志审计
    │
    └─→ 要不要跨平台一致？
          ├─ 要 → WSL2 方案（和 macOS/Linux 共享代码）
          └─ 不需要 → 原生 Win32 方案（但实现成本高）
```

---

## 七、沙箱配置清单（必配 6 项）

不管用哪种方案，这 6 项一定要配：

| 配置项 | 作用 | 建议默认值 | 调整方向 |
|--------|------|-----------|----------|
| **文件系统边界** | Agent 只能读写哪些目录 | 仅限工作区 + 临时目录 | 信任度越低，边界越紧 |
| **网络控制** | 能不能联网、能连哪些 | 默认关闭 | 必要时开域名白名单 |
| **执行超时** | 防止死循环 / runaway process | 30 秒 | 大数据任务可调大 |
| **资源限制** | CPU / 内存 / 磁盘上限 | 2 vCPU / 512MB / 1GB | 按任务类型调 |
| **高危命令拦截** | `rm -rf /`、格式化等直接拦 | 正则匹配 + 拦截 | 拦截后进入审批 |
| **审批策略** | 越界操作先问用户 | 读文件免审 / 写文件问一次 / 联网必问 | 按信任度分级 |

---

## 八、常见问题

**Q1：沙箱会不会影响 Agent 的能力？**
会限制，但这是故意的——Agent 不该能访问沙箱外的东西。能力不够时应该**扩展沙箱内的能力**（装更多依赖、加更多工具），而不是放宽沙箱边界。

**Q2：Agent 需要访问本地项目文件怎么办？**
把项目目录**挂载**进沙箱（只读或读写），而不是让 Agent 直接访问宿主机文件系统。挂载时指定精确路径，不要挂整个家目录。

**Q3：Docker 不就是沙箱吗？为什么还要 OS-level / MicroVM？**
Docker 是容器级隔离，**共享宿主机内核**——容器逃逸漏洞是真实风险。OS-level 更轻更快但隔离更弱；MicroVM 有独立内核，安全性接近 VM 但启动速度接近容器。选哪个取决于你的威胁模型。

**Q4：沙箱和安全审核是什么关系？**
沙箱是**事前隔离**（防止出事），安全审核是**事后检查**（查有没有越界）。两者配合：沙箱拦住大多数风险，审核做兜底审计和合规验证。

**Q5：怎么测试沙箱配置对不对？**
做"越狱测试"：故意让 Agent 尝试访问沙箱外的文件、联网、执行高危命令，看沙箱能不能拦住。定期做回归测试，防止配置变更导致隔离失效。

---

## 九、最佳实践

1. **默认最小权限**：能不给的权限就不给，需要时再加。
2. **分层防御**：沙箱 + 审批 + 审核日志，三层一起上。别只靠审批——用户可能点快了没看清。
3. **无状态即安全**：沙箱用完就销毁，不要复用。E2B 那种每次新建、用完即焚的模式最安全。
4. **审批要分级**：读文件可以免审，写文件问一次，联网/装依赖必问，高危操作强制二次确认。
5. **日志完整**：记录 Agent 在沙箱里执行的每一条命令、读写的每一个文件、每一次网络请求。出问题时能回溯。
6. **和 [[rag/11_安全审核与日志/安全与合规|安全与合规]] 联动**：沙箱是技术隔离，合规是制度隔离，两者互补。

---

## 相关导航

- 安全审核视角：[[rag/11_安全审核与日志/安全与合规|安全与合规]]
- 架构总览：[[agent/harness/Agent_六种架构|Agent 六种架构]]
- 工具调用底层：[[agent/harness/Function_Call|Function Call]]
- 反直觉原则：[[agent/harness/四条反直觉铁律|四条反直觉铁律]]
- 工程模式：[[agent/harness/六大工程模式|六大工程模式]]
- Agent 模式总览：[[agent/patterns/Agent模式总览|Agent模式总览]]

## 相关资料

- [Codex Sandboxing 官方文档](https://developers.openai.ac.cn/codex/concepts/sandboxing)
- [Claude Code Sandboxing 文档](https://code.claude.com/docs/en/sandboxing)
- [TraeWork 沙箱文档](https://docs.trae.cn/work_sandbox)
- [OpenInterpreter Sandbox & Approvals](https://www.openinterpreter.com/docs/terminal/sandbox)
- [E2B Sandbox 架构](https://e2b.dev/)
- [OpenAI Code Interpreter 文档](https://developers.openai.ac.cn/api/docs/guides/tools-code-interpreter)
- [Malkov & Yashunin, Efficient and robust ANN using HNSW, 2018](https://arxiv.org/abs/1603.09320) — Firecracker 论文参考
