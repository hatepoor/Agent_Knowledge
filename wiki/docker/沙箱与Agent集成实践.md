---
tags: [docker, sandbox, production, integration, agent]
created: 2026-07-28
updated: 2026-07-28
type: concept
---

# 沙箱与 Agent 集成实践

将 Docker 沙箱、Skills 系统、LangChain Agent 整合为一个生产就绪的完整应用。

## 一、项目结构

```
project/
├── main.py                      # FastAPI + Agent 入口
├── config.py                    # 配置管理（Pydantic Settings）
├── sandbox/
│   ├── __init__.py
│   ├── manager.py               # DockerSandbox + SandboxConfig + SandboxResult
│   └── tools.py                 # @tool 封装
├── skills/
│   ├── loader.py                # Skill 扫描 + 解析
│   └── tools.py                 # list/load/search skill tool
├── skills_data/                 # Skill 文件（挂载到沙箱 /skills）
│   ├── code_review.md
│   ├── data_analysis.py
│   └── report_writing.md
├── middleware/
│   └── audit.py                 # 自定义审计中间件
├── Dockerfile                   # 自定义 Agent 镜像
├── Dockerfile.sandbox           # 沙箱镜像
├── .env
├── requirements.txt
└── pyproject.toml
```

## 二、配置管理（config.py）

用 Pydantic Settings 管理所有配置，支持环境变量覆盖：

```python
from pydantic_settings import BaseSettings
from pathlib import Path

class Settings(BaseSettings):
    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

    # Agent
    model_name: str = "deepseek-v4-flash"
    api_key: str = ""

    # Sandbox
    sandbox_image: str = "python:3.11-slim"
    sandbox_memory_mb: int = 256
    sandbox_cpu: float = 1.0
    sandbox_timeout: int = 30
    sandbox_network_default: bool = False

    # Skills
    skills_dir: str = "./skills_data"

    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 2

settings = Settings()
```

## 三、沙箱管理器（sandbox/manager.py）

```python
import docker
import uuid
import time
from pathlib import Path
from dataclasses import dataclass
from config import settings


@dataclass
class SandboxConfig:
    image: str = settings.sandbox_image
    memory_limit: str = f"{settings.sandbox_memory_mb}m"
    cpu_limit: float = settings.sandbox_cpu
    timeout: int = settings.sandbox_timeout
    network_enabled: bool = settings.sandbox_network_default
    skills_mount: str = settings.skills_dir


@dataclass
class SandboxResult:
    stdout: str = ""
    stderr: str = ""
    exit_code: int = 0
    container_id: str = ""
    duration_ms: float = 0.0


class DockerSandbox:
    def __init__(self, config: SandboxConfig | None = None):
        self.config = config or SandboxConfig()
        self.client = docker.from_env()
        self._ensure_image()

    def _ensure_image(self):
        try:
            self.client.images.get(self.config.image)
        except docker.errors.ImageNotFound:
            self.client.images.pull(self.config.image)

    def execute(self, code: str, *, language: str = "python",
                timeout: int | None = None, env: dict | None = None,
                network_enabled: bool | None = None) -> SandboxResult:
        ...

    def _build_mounts(self) -> list:
        import docker.types
        mounts = []
        skills_dir = Path(self.config.skills_mount)
        if skills_dir.exists():
            mounts.append(docker.types.Mount(
                target="/skills", source=str(skills_dir.absolute()),
                type="bind", read_only=True))
        return mounts

    def _build_command(self, code: str, language: str) -> list[str]:
        if language == "python": return ["python", "-c", code]
        elif language == "bash": return ["bash", "-c", code]
        raise ValueError(f"unsupported language: {language}")


sandbox = DockerSandbox(SandboxConfig())
```

## 四、Tool 封装（sandbox/tools.py + skills/tools.py）

```python
# sandbox/tools.py
from langchain.tools import tool
from sandbox.manager import sandbox

@tool
def sandbox_execute(code: str, language: str = "python") -> str:
    """在 Docker 沙箱中安全执行代码。"""
    result = sandbox.execute(code, language=language)
    if result.exit_code != 0:
        return f"[失败 {result.exit_code}] stderr: {result.stderr}\nstdout: {result.stdout}"
    return result.stdout or "[完成]"

@tool
def sandbox_read_file(path: str) -> str:
    """在沙箱中读取 /skills/ 下的文件。参数: path 如 'code_review.md'"""
    code = f"""
import pathlib
p = pathlib.Path('/skills/{path.strip("/")}')
print(p.read_text('utf-8') if p.exists() else f'文件不存在: {p}')
"""
    return sandbox.execute(code).stdout

@tool
def sandbox_run_skill(skill_name: str, input_text: str) -> str:
    """在沙箱中加载并执行技能。"""
    code = f"""
import pathlib
md = pathlib.Path('/skills/{skill_name}.md')
py = pathlib.Path('/skills/{skill_name}.py')
if py.exists(): exec(py.read_text('utf-8'))
elif md.exists(): print(md.read_text('utf-8'))
else: print(f'技能不存在: {skill_name}')
"""
    return sandbox.execute(code, timeout=60).stdout

# skills/tools.py
from skills.loader import SkillRegistry

registry = SkillRegistry(Path(settings.skills_dir))

@tool
def list_skills() -> str:
    """列出所有可用技能（名称+描述）。"""
    return registry.format_list()

@tool
def load_skill(skill_name: str) -> str:
    """加载技能完整指令。名称从 list_skills 获取。"""
    skill = registry.get_skill(skill_name)
    return skill["body"] if skill else f"不存在: {skill_name}"
```

## 五、自定义中间件（middleware/audit.py）

```python
import structlog
from langchain.agents.middleware import AgentMiddleware, AgentState
from langgraph.runtime import Runtime

logger = structlog.get_logger()

class AuditMiddleware(AgentMiddleware):
    """每次工具调用记录审计日志"""

    def wrap_tool_call(self, request, handler):
        start = time.time()
        tool_name = request.tool_call["name"]
        tool_args = request.tool_call.get("args", {})

        try:
            result = handler(request)
            duration_ms = (time.time() - start) * 1000
            logger.info("tool_call.success",
                tool=tool_name, duration_ms=duration_ms,
                args_preview=str(tool_args)[:200])
            return result
        except Exception as e:
            duration_ms = (time.time() - start) * 1000
            logger.error("tool_call.failed",
                tool=tool_name, duration_ms=duration_ms, error=str(e))
            raise
```

## 六、Agent + FastAPI 组装（main.py）

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ToolCallLimitMiddleware, ModelCallLimitMiddleware,
    SummarizationMiddleware, PIIMiddleware,
)
from langchain_deepseek import ChatDeepSeek
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

from config import settings
from sandbox.tools import sandbox_execute, sandbox_read_file, sandbox_run_skill
from skills.tools import list_skills, load_skill
from middleware.audit import AuditMiddleware

# ── 生命周期 ──
checkpointer = None
agent = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global checkpointer, agent
    conn = sqlite3.connect("sessions.db", check_same_thread=False)
    checkpointer = SqliteSaver(conn)
    checkpointer.setup()

    llm = ChatDeepSeek(model=settings.model_name, api_key=settings.api_key)
    summary_llm = ChatDeepSeek(model=settings.model_name, api_key=settings.api_key, temperature=0)

    agent = create_agent(
        model=llm,
        tools=[
            # 沙箱工具
            sandbox_execute, sandbox_read_file, sandbox_run_skill,
            # Skills 工具
            list_skills, load_skill,
        ],
        checkpointer=checkpointer,
        system_prompt="""你是沙箱安全助手。重要规则：
1. 所有代码执行通过 sandbox_execute
2. 读取技能使用 sandbox_read_file 或 load_skill
3. 绝不在本地执行代码
4. 不符合任何技能的任务，用自己的能力完成""",
        middleware=[
            AuditMiddleware(),
            ToolCallLimitMiddleware(run_limit=15),
            ModelCallLimitMiddleware(thread_limit=30),
            SummarizationMiddleware(model=summary_llm, trigger=("tokens", 4000)),
            PIIMiddleware(pii_type="email", strategy="redact"),
        ],
    )
    yield
    conn.close()

# ── FastAPI ──
app = FastAPI(title="Agent Sandbox API", lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

class ChatRequest(BaseModel):
    message: str
    thread_id: str = "default"

class ChatResponse(BaseModel):
    reply: str
    thread_id: str

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": req.message}]},
        config={"configurable": {"thread_id": req.thread_id}},
    )
    return ChatResponse(
        reply=result["messages"][-1].content,
        thread_id=req.thread_id,
    )

@app.get("/health")
async def health():
    return {"status": "ok", "agent": agent is not None}
```

## 七、启动

```bash
# 确保 Docker daemon 运行
docker info > /dev/null 2>&1 || exit 1

# 预拉镜像
docker pull python:3.11-slim

# 启动
python -m uvicorn main:app --host 0.0.0.0 --port 8000 --workers 2
```

## 八、相关页面

- [[docker/Docker沙箱概述]] — 架构总览与方案选择
- [[docker/Docker_Sandbox核心类]] — 核心类实现详解
- [[docker/沙箱安全加固]] — 安全配置全维度
- [[docker/沙箱生产部署清单]] — 生产 checklist
- [[agent/skills/Skills实现与集成]] — Skills 系统
