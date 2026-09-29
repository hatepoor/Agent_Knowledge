---
tags: [docker, sandbox, agent, tool, 实现]
created: 2026-07-28
updated: 2026-07-28
type: concept
---

# Docker Sandbox 核心类

自建 Docker 沙箱的核心组件：配置类 → 结果类 → 管理器类 → tool 封装 → Agent 组装。逐层拆解。

## 一、配置类（SandboxConfig）

```python
import docker
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class SandboxConfig:
    """沙箱配置 —— 控制每次执行的隔离策略"""

    image: str = "python:3.11-slim"
    # Docker 镜像名。slim 比 full 小 ~100MB，启动快。如需 pandas/numpy，
    # 用自定义镜像 "agent-sandbox:latest"

    memory_limit: str = "256m"
    # 内存硬上限。容器使用超限立即被 OOM killer 杀掉。
    # 经验值：纯 Python 计算 128m，含 pandas 512m

    cpu_limit: float = 1.0
    # CPU 核心数。设为 1.0 表示最多用 1 个核，设为 0.5 表示半核。
    # Docker 内部用 nano_cpus（纳秒），1 核 = 1e9 纳秒

    timeout: int = 30
    # 单次执行的绝对超时秒数。到时间不管代码是否执行完，容器被 kill。
    # 生产经验：简单计算 10s，数据分析 60s，LLM 调 API 不设限（靠 Agent 层控制）

    network_enabled: bool = False
    # 默认禁网。启用后容器可访问外网——skill 需要调 API 时才开启。
    # 生产建议：默认禁用，需要时按 tool 粒度开启（见沙箱安全加固）

    skills_mount: str = "./skills"
    # 宿主机 skills 目录路径，挂载到容器的 /skills，只读
```

**与 Agent 启动时初始化**：整个应用启动时创建一次 `SandboxConfig`，所有 Agent 的 tool 共享同一个配置。

## 二、结果类（SandboxResult）

```python
@dataclass
class SandboxResult:
    """沙箱执行结果 —— 用于 tool 封装中判断成功/失败"""

    stdout: str          # 标准输出，正常结果从这里取
    stderr: str          # 标准错误，异常信息从这里取
    exit_code: int       # 0=成功，非0=失败，-1=容器本身异常
    container_id: str    # 容器 ID 前 12 位，用于日志追踪
    duration_ms: float   # 执行耗时，用于性能监控
```

**tool 中判断结果**：

```python
result = sandbox.execute(code)
if result.exit_code != 0:
    return f"[失败 {result.exit_code}] {result.stderr}"
return result.stdout
```

## 三、管理器类（DockerSandbox）

完整类分为 4 个职责块：

### 3.1 初始化 + 镜像管理

```python
class DockerSandbox:
    def __init__(self, config: SandboxConfig | None = None):
        self.config = config or SandboxConfig()
        self.client = docker.from_env()   # 从环境变量/Docker socket 连接
        self._ensure_image()              # 应用启动时拉镜像

    def _ensure_image(self):
        """如果镜像不存在就拉取。启动阶段完成，不影响运行时性能。"""
        try:
            self.client.images.get(self.config.image)
        except docker.errors.ImageNotFound:
            print(f"[Sandbox] 拉取镜像 {self.config.image}...")
            self.client.images.pull(self.config.image)
```

### 3.2 核心执行

```python
    def execute(
        self,
        code: str,
        *,
        language: str = "python",
        timeout: int | None = None,
        env: dict[str, str] | None = None,
        network_enabled: bool | None = None,
    ) -> SandboxResult:
        """在 Docker 容器中执行代码并返回结果。

        执行流程：
          1. 生成唯一容器 ID（用于日志追踪）
          2. 构建挂载点（skills 目录 → /skills，只读）
          3. docker run：限制 CPU/内存/网络/超时，执行代码
          4. 容器自动销毁（remove=True），消除残留风险
          5. 返回 stdout、stderr、exit_code、耗时
        """
        import uuid, time

        container_id = str(uuid.uuid4())[:12]
        timeout = timeout or self.config.timeout
        net_enabled = (
            network_enabled if network_enabled is not None
            else self.config.network_enabled
        )

        # ── 构建挂载 ──
        mounts = self._build_mounts()

        # ── 构建命令 ──
        cmd = self._build_command(code, language)

        start = time.time()
        try:
            container = self.client.containers.run(
                image=self.config.image,
                command=cmd,
                mem_limit=self.config.memory_limit,
                nano_cpus=int(self.config.cpu_limit * 1e9),
                network_mode="none" if not net_enabled else "bridge",
                mounts=mounts,
                environment=env or {},
                working_dir="/workspace",
                read_only=True,             # 根文件系统只读
                tmpfs={"/workspace": "size=100m"},  # /workspace 临时读写
                cap_drop=["ALL"],           # 丢弃所有 Linux capabilities
                security_opt=["no-new-privileges:true"],  # 禁止提权
                timeout=timeout,
                remove=True,                # 执行完自动删除
                detach=False,
                stdout=True,
                stderr=True,
            )
            exit_code = 0
            stdout = container.decode("utf-8") if container else ""
            stderr = ""

        except docker.errors.ContainerError as e:
            exit_code = e.exit_status
            stdout = e.stdout.decode("utf-8") if e.stdout else ""
            stderr = e.stderr.decode("utf-8") if e.stderr else str(e)
        except Exception as e:
            exit_code = -1
            stdout = ""
            stderr = f"沙箱异常: {str(e)}"

        duration_ms = (time.time() - start) * 1000

        return SandboxResult(
            stdout=stdout.strip(),
            stderr=stderr.strip(),
            exit_code=exit_code,
            container_id=container_id,
            duration_ms=duration_ms,
        )
```

**关键安全参数说明**：

| 参数 | 值 | 作用 |
|------|-----|------|
| `read_only` | `True` | 根文件系统只读，防止写 `/etc`、`/bin` |
| `tmpfs` | `{"/workspace": "size=100m"}` | 只有 `/workspace` 可写，上限 100MB |
| `cap_drop` | `["ALL"]` | 禁用所有 Linux 内核能力 |
| `security_opt` | `"no-new-privileges"` | 禁止 setuid/setgid 提权 |
| `remove` | `True` | 执行完立即删除，无容器残留 |

### 3.3 挂载构建

```python
    def _build_mounts(self) -> list:
        """构建容器挂载点：skills 目录只读挂载"""
        import docker.types

        mounts = []
        skills_dir = Path(self.config.skills_mount)
        if skills_dir.exists():
            mounts.append(docker.types.Mount(
                target="/skills",
                source=str(skills_dir.absolute()),
                type="bind",
                read_only=True,
            ))
        return mounts
```

### 3.4 命令构建

```python
    def _build_command(self, code: str, language: str) -> list[str]:
        """根据语言构建 Docker CMD"""
        if language == "python":
            return ["python", "-c", code]
        elif language == "bash":
            return ["bash", "-c", code]
        else:
            raise ValueError(f"不支持的语言: {language}")
```

### 3.5 全局单例

```python
# 应用启动时创建一次，所有 tool 共享
sandbox = DockerSandbox(SandboxConfig(
    image="python:3.11-slim",
    memory_limit="256m",
    cpu_limit=1.0,
    timeout=30,
    network_enabled=False,
    skills_mount="./skills",
))
```

> **为什么用全局单例**：`docker.from_env()` 会建立与 Docker daemon 的持久连接。每次新建 `DockerSandbox` 实例就建一次连接是对资源的浪费。单例保证连接复用。

## 四、Tool 封装（@tool）

```python
from langchain.tools import tool
from sandbox_manager import sandbox

@tool
def sandbox_execute(code: str, language: str = "python") -> str:
    """在 Docker 沙箱中执行代码。所有代码执行必须通过此工具。

    参数:
        code: 要执行的代码
        language: python 或 bash
    """
    result = sandbox.execute(code, language=language)
    if result.exit_code != 0:
        return (
            f"[沙箱执行失败，退出码 {result.exit_code}]\n"
            f"stderr:\n{result.stderr}\n"
            f"stdout:\n{result.stdout}"
        )
    return result.stdout or "[执行完成，无输出]"

@tool
def sandbox_read_file(path: str) -> str:
    """在沙箱中读取 /skills/ 目录下的文件。

    参数:
        path: 文件路径，相对 /skills/，如 "code_review.md"
    """
    code = f"""
import pathlib
p = pathlib.Path('/skills/{path.strip("/")}')
if not p.exists():
    print(f'文件不存在: {{p}}')
else:
    print(p.read_text(encoding='utf-8'))
"""
    result = sandbox.execute(code)
    return result.stdout or result.stderr

@tool
def sandbox_run_skill(skill_name: str, input_text: str) -> str:
    """在沙箱中加载并执行技能。

    参数:
        skill_name: 技能名称（/skills/ 目录下 .md 或 .py）
        input_text: 输入数据
    """
    code = f"""
import pathlib, sys
md = pathlib.Path('/skills/{skill_name}.md')
py = pathlib.Path('/skills/{skill_name}.py')
if py.exists():
    exec(py.read_text('utf-8'))
elif md.exists():
    print(md.read_text('utf-8'))
else:
    print(f'技能不存在: {skill_name}')
"""
    result = sandbox.execute(code, timeout=60)
    return result.stdout or result.stderr
```

## 五、组装到 Agent

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware
from langchain_deepseek import ChatDeepSeek

agent = create_agent(
    model=ChatDeepSeek(model="deepseek-v4-flash"),
    tools=[sandbox_execute, sandbox_read_file, sandbox_run_skill],
    system_prompt="""你是沙箱安全助手。重要规则：
1. 所有代码执行通过 sandbox_execute
2. 读取文件使用 sandbox_read_file
3. 执行技能使用 sandbox_run_skill
4. 绝不在本地执行代码或读文件""",
    middleware=[
        ToolCallLimitMiddleware(run_limit=10),
    ],
)
```

## 六、相关页面

- [[docker/沙箱/Docker沙箱概述]] — 架构总览与方案选择
- [[docker/沙箱/沙箱安全加固]] — 网络隔离、镜像扫描、密钥管理
- [[docker/沙箱/沙箱性能优化]] — 镜像优化、预热池
- [[docker/沙箱/沙箱与Agent集成实践]] — 完整生产级代码
