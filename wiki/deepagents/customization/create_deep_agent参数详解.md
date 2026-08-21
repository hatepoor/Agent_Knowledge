---
tags: [deepagents, create_deep_agent, API参考, 参数详解]
created: 2026-08-01
updated: 2026-08-01
type: concept
source_url: https://docs.langchain.com/oss/python/deepagents/customization
deepagents_version: ">=0.7"
---

# create_deep_agent 参数详解

> `create_deep_agent` 是 DeepAgents 的唯一入口工厂函数，共 **18 个参数**，返回一个 LangGraph 编译后的 `CompiledStateGraph`。除 `model`、`tools` 外，其余参数均须用关键字传递。

## 完整签名

```python
create_deep_agent(
    model: str | BaseChatModel | None = None,
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    subagents: Sequence[SubAgent | CompiledSubAgent | AsyncSubAgent] | None = None,
    skills: list[str] | None = None,
    memory: list[str] | None = None,
    permissions: list[FilesystemPermission] | None = None,
    backend: BackendProtocol | None = None,
    interrupt_on: dict[str, bool | InterruptOnConfig] | None = None,
    response_format: ResponseFormat | type | dict | None = None,
    state_schema: type[DeepAgentState] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph[AgentState, ContextT, ...]
```

> `*` 表示除 `model`、`tools` 外的参数必须用关键字传。返回值是 LangGraph 的 CompiledStateGraph——可流式、可中断、可持久化。

## 参数速查总表

| 分组 | 参数 | 作用 |
|------|------|------|
| 基本配置 | `model` | 模型（字符串 `"供应商:模型名"` 或实例） |
| | `tools` | 自定义工具（@tool/函数/MCP 工具） |
| | `system_prompt` | 系统指令，追加到默认指令之后 |
| 四能力 | `subagents` | 自定义子代理列表 |
| | `skills` | 技能目录路径，按需加载 |
| | `memory` | AGENTS.md 路径，启动即加载 |
| | `backend` | 文件系统后端（6 种） |
| 持久化/安全 | `checkpointer` | 检查点器（HITL 必需） |
| | `store` | 长期记忆存储 |
| | `permissions` | 文件系统路径级访问控制 |
| | `interrupt_on` | 工具执行前人工审批 |
| 高级定制 | `response_format` | 结构化输出 schema |
| | `state_schema` | 自定义图状态 schema |
| | `context_schema` | 运行时上下文 schema |
| | `middleware` | 额外中间件（按 name 替换默认栈） |
| 辅助 | `debug` | 调试模式 |
| | `name` | Agent 名称 |
| | `cache` | LLM 结果缓存 |

---

## 一、基本配置（3 个）

### model — 模型

```python
# 字符串：一行切换供应商
agent = create_deep_agent(model="deepseek:deepseek-v4-flash")
agent = create_deep_agent(model="anthropic:claude-sonnet-4-6")
agent = create_deep_agent(model="openai:gpt-5.4")

# 模型实例：可传 temperature/max_tokens
from langchain.chat_models import init_chat_model
llm = init_chat_model("deepseek:deepseek-v4-flash", temperature=0.3)
agent = create_deep_agent(model=llm)
```

| 注意 | 说明 |
|------|------|
| 格式 | `"供应商:模型名"`，支持 OpenAI/Anthropic/Gemini/Bedrock/HF 等 |
| 要求 | 模型必须支持工具调用（function calling） |
| 默认 | 官方默认 `claude-sonnet-4-5` |

### tools — 自定义工具

```python
from deepagents import create_deep_agent

# 普通函数（自动转工具）
def internet_search(query: str) -> str:
    """网络搜索"""
    return tavily_client.search(query)

# MCP 工具（langchain-mcp-adapters 加载）
from langchain_mcp_adapters.client import MultiServerMCPClient
async with MultiServerMCPClient({"server": {"url": "..."}}) as client:
    mcp_tools = await client.get_tools()

agent = create_deep_agent(model=llm, tools=[internet_search, *mcp_tools])
```

| 类型 | 说明 |
|------|------|
| `BaseTool` | `@tool` 装饰的对象 |
| `Callable` | 普通 Python 函数，自动转工具 |
| `dict` | MCP 工具等结构化定义 |

### system_prompt — 系统指令

```python
agent = create_deep_agent(
    model=llm,
    system_prompt="你是研究助手，先 write_todos 规划，再执行。",
)
```

> 追加到默认指令之后。不要重复解释内置工具（中间件已覆盖），只写领域工作流和约束。

---

## 二、四能力（subagents / skills / memory / backend）

### subagents — 子代理

```python
from deepagents import create_deep_agent, SubAgent

agent = create_deep_agent(
    model=llm,
    subagents=[
        SubAgent(
            name="researcher",
            description="深度调研，返回带引用的总结",
            system_prompt="你是研究员，先搜索再总结。",
            model="deepseek:deepseek-v4-flash",   # 可选，默认继承主模型
            tools=[internet_search],               # 可选，默认继承主工具
        ),
        # 也支持 CompiledSubAgent（打包自定义图）/ AsyncSubAgent（并行）
    ],
)
```

### skills — 技能（按需加载）

```python
agent = create_deep_agent(model=llm, skills=["./skills"])
# skills/ 下每个 SKILL.md 是一个技能，LLM 按需 load_skill 加载
# 渐进式披露：只给名字+描述，需要时才加载正文 → 省 token
```

### memory — 常驻记忆（AGENTS.md）

```python
agent = create_deep_agent(model=llm, memory=["./AGENTS.md"])
# 启动即加载为常驻上下文，每次请求都携带
```

**skills vs memory 对比**：

| | `skills` | `memory` |
|---|---|---|
| 加载时机 | 按需（LLM 调 load_skill） | 启动即加载 |
| token 占用 | 只有名字+描述 | 全文常驻 |
| 适合内容 | 任务型流程（代码审查 SOP） | 稳定领域知识（项目说明） |

### backend — 文件系统后端

```python
from deepagents.backends import (
    StateBackend, FilesystemBackend, StoreBackend, CompositeBackend,
)

# 默认：StateBackend（内存文件系统，配 checkpointer 跨轮持久）
agent = create_deep_agent(model=llm)

# 本地磁盘
agent = create_deep_agent(model=llm, backend=FilesystemBackend(root_dir="./workspace"))

# 混合：工作文件临时，记忆路径长期
agent = create_deep_agent(
    model=llm,
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": StoreBackend(store=store)},
    ),
)
```

| 后端 | 说明 | 安全 |
|------|------|------|
| `StateBackend`（默认） | 线程内内存文件系统 | 安全 |
| `FilesystemBackend` | 真实磁盘操作 | ⚠️ 慎用 |
| `LocalShellBackend` | 磁盘 + execute 命令 | ⚠️⚠️ 极度谨慎 |
| `StoreBackend` | 跨线程长期存储 | 安全 |
| `ContextHubBackend` | LangSmith Hub 持久文件 | 安全 |
| `CompositeBackend` | 按路径路由到不同后端 | 视配置 |

---

## 三、持久化与安全（checkpointer / store / permissions / interrupt_on）

### checkpointer — 检查点器

```python
from langgraph.checkpoint.sqlite import SqliteSaver

agent = create_deep_agent(
    model=llm,
    checkpointer=SqliteSaver(conn),   # HITL 必需
)
```

### store — 长期记忆

```python
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    model=llm,
    store=InMemoryStore(),   # 生产换 PostgresStore
)
```

### permissions — 文件权限

```python
from deepagents import FilesystemPermission

agent = create_deep_agent(
    model=llm,
    permissions=[
        FilesystemPermission("/workspace", allow=True),   # 白名单
        FilesystemPermission("/etc", allow=False),         # 黑名单
    ],
)
```

### interrupt_on — 人工审批

```python
agent = create_deep_agent(
    model=llm,
    checkpointer=SqliteSaver(conn),   # 必须配！
    interrupt_on={
        "send_email": True,   # 发邮件前暂停等审批
        "delete_file": {"allowed_decisions": ["approve", "reject", "edit"]},
    },
)
```

| 注意 | 说明 |
|------|------|
| 必需配合 | 不配 `checkpointer` 会报错 |
| 默认决策 | `approve, edit, reject, respond` |
| 机制 | 工具调用前暂停图执行，等人工输入 |

---

## 四、高级定制（response_format / state_schema / context_schema / middleware）

### response_format — 结构化输出

```python
from pydantic import BaseModel

class ResearchReport(BaseModel):
    title: str
    findings: list[str]
    sources: list[str]

agent = create_deep_agent(
    model=llm,
    response_format=ResearchReport,   # 结果存 state["structured_response"]
)
```

### state_schema — 自定义图状态

```python
from typing_extensions import NotRequired
from deepagents import DeepAgentState

class MyState(DeepAgentState):
    model_call_count: NotRequired[int]   # 扩展状态字段
    user_id: NotRequired[str]

agent = create_deep_agent(model=llm, state_schema=MyState)
```

### context_schema — 运行时上下文

```python
from pydantic import BaseModel

class RuntimeContext(BaseModel):
    user_id: str
    api_key: str          # 每次运行时传入，不进对话历史
    feature_flags: dict[str, bool] = {}

agent = create_deep_agent(model=llm, context_schema=RuntimeContext)

# 运行时传入
agent.invoke(
    {"messages": [...]},
    config={"configurable": {"context": {"user_id": "u1", "api_key": "sk-..."}}},
)
```

### middleware — 中间件

```python
from langchain.agents.middleware import ModelCallLimitMiddleware

agent = create_deep_agent(
    model=llm,
    middleware=[ModelCallLimitMiddleware(thread_limit=20)],
)
# 按 .name 匹配默认栈：同名 → 替换默认实例；不同名 → 插入 PatchToolCallsMiddleware 之后
```

| 注意 | 说明 |
|------|------|
| 覆盖 FilesystemMiddleware | 必须手动传 `backend` + `permissions` |
| 版本要求 | `deepagents>=0.7` 支持覆盖默认中间件 |
| 并发安全 | 不要在中间件中改实例属性，用图状态 |

---

## 五、辅助（debug / name / cache）

```python
agent = create_deep_agent(
    model=llm,
    debug=True,          # 打印中间过程
    name="research-agent",  # 图命名
    cache=BaseCache(),   # LLM 结果缓存
)
```

## 六、易错点汇总

```python
# ❌ interrupt_on 不配 checkpointer
agent = create_deep_agent(model=llm, interrupt_on={"send_email": True})
# 报错：HITL 需要 checkpointer

# ✅ 正确
agent = create_deep_agent(
    model=llm,
    checkpointer=SqliteSaver(conn),
    interrupt_on={"send_email": True},
)

# ❌ StateBackend 虚拟路径不以 / 开头
backend=StateBackend()   # 写文件到 "memories/x.txt"
# 报错：路径必须以 / 开头

# ✅ 正确："/memories/x.txt"

# ❌ 覆盖 FilesystemMiddleware 不传 backend/permissions
middleware=[FilesystemMiddleware()]
# 报错：覆盖时需完整配置

# ❌ 用 dict 传核心参数（model 是位置参数，但 * 后必须关键字）
create_deep_agent("deepseek:deepseek-v4-flash", system_prompt="...")  # ❌
create_deep_agent(model="deepseek:deepseek-v4-flash", system_prompt="...")  # ✅
```

## 相关页面

- [[deepagents/overview/DeepAgents概述]] — 架构定位与核心能力
- [[deepagents/subagents/子代理概述]] — subagents 参数详解
- [[deepagents/skills/Skills系统]] — skills 参数详解
- [[deepagents/backend/后端概述]] — backend 六种后端选型
- [[deepagents/customization/HarnessProfile深度用法]] — 按模型配置包
- [[deepagents/customization/State扩展与DeltaChannel]] — state_schema/context_schema
- [[deepagents/middleware/中间件栈概述]] — middleware 默认栈与合并规则
