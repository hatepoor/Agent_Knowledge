---
tags: [deepagents, context, memory, AGENTS.md]
created: 2026-07-29
updated: 2026-08-01
type: concept
deepagents_version: ">=0.7"
---

# 长期记忆与 AGENTS.md

DeepAgents 通过 `MemoryMiddleware` 从 `AGENTS.md` 文件加载跨会话持久记忆。不同于 Skills 的按需加载——memory 文件**始终加载**。

## 一、AGENTS.md 格式

```markdown
# Agent Name: Research Assistant

## 记忆
- 用户偏好：深度研究报告，至少 2000 字，引用权威来源
- 上次研究主题：LangChain 中间件系统（2026-07-20）
- 常用信息源：官方文档 > 学术论文 > 社区博客
- 用户是 Python 开发者，偏好中文回答
- 禁止使用 emoji

## 项目约定
- 代码风格：PEP 8，类型提示必选
- 文档格式：Markdown，含目录
- 测试覆盖率：> 80%

## 最近工作
- 创建了 agent_middleware_report.md（2026-07-20）
- 审查了 docker_sandbox.py 安全问题（2026-07-21）
```

## 二、配置方式

```python
# 加载一个 memory 文件
agent = create_deep_agent(
    model=llm,
    memory=["AGENTS.md"],
)

# 加载多个
agent = create_deep_agent(
    model=llm,
    memory=[
        "AGENTS.md",               # 项目级
        "~/.deepagents/memory.md", # 用户级
    ],
)
```

## 三、Agent 自动更新记忆

MemoryMiddleware 不仅加载记忆——Agent 可以**主动更新**自己的 AGENTS.md：

```
Agent 完成任务 → 判断有值得记录的信息
  → write_file("/AGENTS.md", updated_content)
  → 下次启动时自动加载更新后的记忆

Agent 不能直接修改 memory 文件?
  → MemoryMiddleware 管理 /AGENTS.md 路径
  → Agent 用 write_file 写入即可
```

## 四、Memory vs Skills

| | Memory（AGENTS.md） | Skills（SKILL.md） |
|---|---|---|
| 加载时机 | **始终加载**（启动时注入 prompt） | **按需加载**（Agent 主动触发） |
| 内容类型 | 偏好、约定、历史 | 工作流 SOP、领域知识 |
| Agent 可更新 | ✅ 可以 write_file 修改 | ❌ 只读 |
| 典型大小 | 200-500 字 | 500-2000 字 |

## 五、AGENTS.md vs system_prompt

### 5.1 一句话区别

> `system_prompt` 是开发者**写死在代码里**的人设指令；`AGENTS.md` 是**独立文件形式的活的长期记忆**。两者最终都注入 SystemMessage，但来源、可变性、管理者完全不同。

### 5.2 执行流程对比

```
系统提示词 (system_prompt):
    create_deep_agent(system_prompt="你是一个研究助手")
        ↓
    create_deep_agent 内部：默认指令 + 你的 system_prompt
        ↓
    每次请求都作为 SystemMessage 注入
        ↓
    ← 写死在代码里，改 prompt 要改代码

AGENTS.md (memory):
    create_deep_agent(memory=["AGENTS.md"])
        ↓
    MemoryMiddleware 读取文件内容
        ↓
    同样注入 SystemMessage
        ↓
    ← 独立文件，改内容不用改代码，Agent 还能自己改
```

### 5.3 七维对比表

| 维度 | system_prompt | AGENTS.md |
|------|--------------|-----------|
| **本质** | 代码参数，写死的指令 | 独立文件，活的记忆 |
| **注入时机** | 每次请求，静态 | 每次请求，但内容可动态变化 |
| **谁定义** | 开发者（代码里） | 开发者 + **Agent 自己** |
| **可变性** | 改代码才生效 | 改文件即生效，Agent 可 write_file 自更新 |
| **作用范围** | 该 Agent 的人设/工作方式 | 项目级/用户级长期事实 |
| **更新频率** | 低频（代码迭代） | 高频（每次任务后可能沉淀） |
| **典型内容** | "你是研究助手，先规划再执行" | "用户偏好中文，上次做到 XX" |

### 5.4 核心差异：谁在管理它

**system_prompt 是"死"的，AGENTS.md 是"活"的**——这是最关键的区分：

```python
# system_prompt：只能开发者改
agent = create_deep_agent(
    system_prompt="你是研究助手",   # 想改？改代码重新部署
)

# AGENTS.md：Agent 自己也能改
# Agent 完成任务 → 判断有值得记录的信息
#   → write_file("/AGENTS.md", updated_content)
#   → 下次启动自动加载新记忆 → 自我进化
```

**自进化流程**：

```
用户告诉 Agent "我偏好中文回答"
    │
    ▼
Agent 判断这是值得长期记忆的信息
    │
    ▼
write_file("/AGENTS.md", 追加"用户偏好中文回答")
    │
    ▼
下次会话启动 → MemoryMiddleware 加载更新后的 AGENTS.md
    │
    ▼
Agent 自动记住了 → 跨会话不失忆
```

### 5.5 生产中的三层 prompt 体系

```
① system_prompt（代码层）：人设与行为准则
    "你是研究助手，先 write_todos 规划再执行，引用来源"

② AGENTS.md（记忆层）：项目与用户长期事实
    "用户偏好中文，报告 ≥2000 字，引用官方文档"

③ SKILL.md（技能层）：任务型 SOP，按需加载
    "代码审查：通读→安全→性能→输出表格"
```

| 层 | 更新者 | 加载方式 | 典型内容 |
|----|--------|---------|---------|
| system_prompt | 开发者 | 常驻（每次） | 人设、工作流约束 |
| AGENTS.md | 开发者 + Agent | 常驻（每次） | 偏好、约定、历史 |
| SKILL.md | 开发者 | 按需（触发才读） | 流程、步骤、格式 |

### 5.6 生产实践建议

```python
agent = create_deep_agent(
    model="deepseek:deepseek-v4-flash",
    system_prompt="""你是一个专业研究助手。
工作方式：先规划，再执行，引用来源。""",          # ① 人设（代码里）
    memory=["AGENTS.md"],                          # ② 记忆（文件里）
    skills=["./skills/code_review.md"],            # ③ 技能（按需）
)
```

| 要点 | 说明 |
|------|------|
| 常驻 = 每次占 token | 只放"每次都要"的稳定知识，别放大段流程（流程放 SKILL.md） |
| 控制大小 | 200-500 字为宜，大了反而稀释注意力 |
| 可设置只读 | 不想让 Agent 乱改 → 放只读后端（FilesystemBackend 只读挂载） |
| 多级文件 | 项目级 AGENTS.md + 用户级 ~/.deepagents/memory.md 分开维护 |

## 六、相关页面

- [[deepagents/context/上下文管理概述]] — 四层上下文
- [[deepagents/skills/Skills系统]] — Skills 系统
- [[deepagents/middleware/中间件栈概述]] — MemoryMiddleware 位置
