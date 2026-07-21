---
tags: [agent, skills, langchain, 实现, 集成, 渐进式加载]
type: tutorial
created: 2026-07-21
updated: 2026-07-21
---

# Skills 实现与集成

在 LangChain Agent 中实现 Skills 支持的完整代码方案：目录扫描、动态加载、渐进式披露、与 MCP/本地工具混合组装。

## 核心思路

Skills 是纯 Markdown 文件，Agent 通过**工具**来访问它们。LLM 自己判断何时需要加载哪个 Skill。

```
用户提问
  │
  ▼
LLM 判断需要某个技能
  → 调用 list_skills() 查看有哪些
  → 调用 load_skill("xxx") 加载完整指令
  │
  ▼
LLM 按照 Skill 内容执行任务
```

## 目录结构（配置驱动）

```
project/
├── skills/                     # 用户放 .md 文件即可，不用改代码
│   ├── code_review.md
│   ├── report_writing.md
│   └── data_analysis.md
├── config/
│   └── agent.yaml              # 可配置 skills 目录路径
└── src/
    └── loaders/
        └── skill_loader.py     # 扫描 + 生成工具
```

## skill_loader.py — 扫描目录，生成工具

```python
from pathlib import Path
from langchain.tools import tool

# 从配置读取路径，不写死
SKILLS_DIR = Path("skills")

def scan_skills() -> list[dict]:
    """扫描 skills 目录，返回所有 skill 的元信息"""
    skills = []
    if not SKILLS_DIR.exists():
        return skills
    for f in sorted(SKILLS_DIR.glob("*.md")):
        lines = f.read_text(encoding="utf-8").strip().split("\n")
        title = lines[0].strip("# ") if lines else f.stem
        skills.append({
            "name": f.stem,
            "title": title,
            "path": str(f),
        })
    return skills

@tool
def list_skills() -> str:
    """列出所有可用技能。不确定该用什么方法完成任务时先调用这个。"""
    skills = scan_skills()
    if not skills:
        return "暂无可用技能"
    return "\n".join(f"- {s['name']}: {s['title']}" for s in skills)

@tool
def load_skill(skill_name: str) -> str:
    """加载指定技能的完整指令，然后按照指令执行任务。"""
    path = SKILLS_DIR / f"{skill_name}.md"
    if not path.exists():
        available = [s["name"] for s in scan_skills()]
        return f"技能 '{skill_name}' 不存在。可用: {available}"
    return path.read_text(encoding="utf-8")

def build_skill_tools() -> list:
    """返回 skill 相关的工具列表，供 Agent 组装时使用"""
    return [list_skills, load_skill]
```

## 组装到 Agent

```python
from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient
from src.loaders.skill_loader import build_skill_tools

async def build_agent():
    # 1. MCP 工具（外部能力）
    mcp_client = MultiServerMCPClient(mcp_config)
    mcp_tools = await mcp_client.get_tools()

    # 2. Skills 工具（领域知识）
    skill_tools = build_skill_tools()

    # 3. 本地工具（简单函数）
    from src.tools.calculator import calculator
    local_tools = [calculator]

    # 4. 合并，传给 Agent
    all_tools = [*mcp_tools, *skill_tools, *local_tools]
    agent = create_agent("openai:gpt-4o", tools=all_tools)
    return agent
```

对 LLM 来说，MCP 工具、Skill 工具、本地工具没有任何区别，它只看名称和描述来决定调用哪个。

## 渐进式加载（技能多时省 token）

技能数量多时，不要一次性把所有 Skill 内容塞进上下文。分两步：先发现，再激活。

```python
@tool
def discover_skills(task_description: str) -> str:
    """第一步：发现可用技能。只返回名称和一行描述，不返回完整内容。"""
    skills = scan_skills()
    if not skills:
        return "暂无可用技能"
    return "\n".join(f"- {s['name']}: {s['title']}" for s in skills)

@tool
def activate_skill(skill_name: str) -> str:
    """第二步：激活技能。返回完整的技能指令，按照指令执行。"""
    path = SKILLS_DIR / f"{skill_name}.md"
    if not path.exists():
        available = [s["name"] for s in scan_skills()]
        return f"技能不存在。可用: {available}"
    return path.read_text(encoding="utf-8")
```

LLM 的调用链：

```
用户: "帮我写一份季度报告"
  │
  ▼
LLM: 调用 discover_skills("写季度报告")
  → 返回: "- report_writing: 报告写作"（几十个 token）
  │
  ▼
LLM: 调用 activate_skill("report_writing")
  → 返回: 完整的写作 SOP（几百个 token，只在需要时加载）
  │
  ▼
LLM: 按照 Skill 内容执行写作
```

## Skill 文件写法

```markdown
# skills/code_review.md
# 代码审查

## 角色
你是一位资深代码审查专家。

## 流程
1. 通读代码，理解整体逻辑
2. 检查安全隐患（注入、XSS、敏感信息）
3. 检查性能问题（N+1 查询、内存泄漏）
4. 检查代码规范（命名、注释、结构）
5. 给出改进建议，按优先级排序

## 输出格式
- 问题列表（严重程度：高/中/低）
- 每个问题附修改建议
- 总体评价（1-10 分）
```

## 用户扩展方式

| 操作 | 方式 | 改代码？ |
|------|------|---------|
| 添加新 Skill | 往 `skills/` 放 .md 文件 | 不用 |
| 修改 Skill | 编辑对应 .md 文件 | 不用 |
| 删除 Skill | 删除 .md 文件 | 不用 |
| 改 Skills 目录路径 | 改 `config/agent.yaml` | 不用 |

## 与 MCP Prompts 的配合

Skills 也可以放在 MCP 服务器里，通过 `@mcp.prompt()` 暴露：

```python
# MCP 服务器端
@mcp.prompt()
def code_review(code: str) -> str:
    """代码审查技能模板"""
    return Path("skills/code_review.md").read_text() + f"\n\n代码:\n{code}"
```

```python
# Agent 端：MCP 的 prompt 就是远程的 Skill
prompt = await client.get_prompt("my-server", "code_review", {"code": code})
```

这样 Skill 的管理和更新在 MCP 服务器侧，Agent 端不用改代码。

## 各方案选择

| 技能数量 | 推荐方案 | 原因 |
|---------|---------|------|
| 1-5 个 | `list_skills` + `load_skill` | 简单直接 |
| 5-20 个 | `discover_skills` + `activate_skill` | 渐进式省 token |
| 20+ 个 | RAG 检索匹配 Skill | 自动匹配，不用 LLM 浏览列表 |
| 远程/多 Agent 共享 | MCP `@mcp.prompt()` | 集中管理，多端复用 |

## 相关

- [[agent/skills/Skills|<- Skills 概念]] — Skills 的定义、与 MCP/Tools 的区别
- [[langchain/tools/工具来源与集成|工具来源与集成]] — Skills vs Tools 对比
- [[langchain/modules/langchain-mcp/MCP提示词编写与执行流程|MCP 提示词]] — MCP Prompts 作为远程 Skill
- [[langchain/modules/langchain-mcp/MCP服务器开发|MCP 服务器开发]] — @mcp.prompt() 用法
