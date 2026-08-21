---
tags: [agent, skills, langchain, 实现, 集成, 渐进式加载]
type: tutorial
created: 2026-07-21
updated: 2026-07-28
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

**为什么用工具而不是直接塞进 system prompt：**

| 方式 | 优点 | 缺点 |
|------|------|------|
| 全部塞进 system prompt | 简单 | 10 个 Skill 就占几千 token，浪费 |
| 封装成工具按需加载 | 省 token，LLM 自主选择 | 多一轮工具调用 |
| RAG 检索匹配 | 技能极多时自动匹配 | 需要向量数据库 |

## 目录结构（配置驱动）

```
project/
├── skills/                     # 用户放 .md 文件即可，不用改代码
│   ├── code_review.md
│   ├── report_writing.md
│   ├── data_analysis.md
│   └── weekly_report.md
├── config/
│   └── agent.yaml              # 配置 skills 目录路径、加载策略
└── src/
    └── loaders/
        └── skill_loader.py     # 扫描 + 解析 + 生成工具
```

## Skill 文件格式规范

每个 Skill 文件建议带 YAML frontmatter，方便 loader 解析元信息：

```markdown
---
name: code_review
description: 代码审查技能，按照安全和性能维度审查代码
tags: [开发, 审查, 代码质量]
version: 1.2
author: team-lead
triggers: ["审查代码", "review", "代码质量"]
---

# 代码审查

## 角色
你是一位资深代码审查专家，有 10 年以上经验。

## 审查流程
1. 通读代码，理解整体逻辑和意图
2. 检查安全隐患（SQL 注入、XSS、敏感信息泄露、路径遍历）
3. 检查性能问题（N+1 查询、内存泄漏、不必要的循环）
4. 检查代码规范（命名、注释、函数长度、单一职责）
5. 检查错误处理（是否有未捕获异常、边界条件）
6. 给出改进建议，按优先级排序（高/中/低）

## 输出格式
### 问题列表
| # | 严重程度 | 位置 | 问题描述 | 修改建议 |
|---|---------|------|---------|---------|
| 1 | 高 | 第12行 | SQL 拼接有注入风险 | 使用参数化查询 |

### 总体评价
- 评分：X/10
- 一句话总结
```

**frontmatter 字段说明：**

| 字段 | 必须 | 作用 |
|------|------|------|
| `name` | 是 | 技能唯一标识（用于加载） |
| `description` | 是 | 一行描述（展示在列表中） |
| `tags` | 否 | 分类标签（用于过滤） |
| `version` | 否 | 版本号 |
| `triggers` | 否 | 触发关键词（辅助 LLM 匹配） |

## skill_loader.py — 完整实现

```python
"""Skills 动态加载器：扫描目录 → 解析 frontmatter → 生成 LangChain 工具"""
import yaml
from pathlib import Path
from langchain.tools import tool

# ===== 配置 =====
def load_skills_config() -> dict:
    """从 agent.yaml 读取 skills 配置"""
    config_path = Path("config/agent.yaml")
    if config_path.exists():
        config = yaml.safe_load(config_path.read_text(encoding="utf-8"))
        return config.get("skills", {})
    return {}

_config = load_skills_config()
SKILLS_DIR = Path(_config.get("directory", "skills"))
LOAD_STRATEGY = _config.get("strategy", "on_demand")  # on_demand | eager | rag

# ===== 解析 Skill 文件 =====
def parse_skill_file(file_path: Path) -> dict:
    """解析 Skill 文件，提取 frontmatter 和正文"""
    content = file_path.read_text(encoding="utf-8")
    
    # 解析 YAML frontmatter
    metadata = {}
    body = content
    if content.startswith("---"):
        parts = content.split("---", 2)
        if len(parts) >= 3:
            try:
                metadata = yaml.safe_load(parts[1]) or {}
            except yaml.YAMLError:
                pass
            body = parts[2].strip()
    
    # 如果没有 frontmatter，从第一行提取标题
    if not metadata.get("name"):
        lines = body.split("\n")
        metadata["name"] = file_path.stem
        metadata["description"] = lines[0].strip("# ") if lines else file_path.stem
    
    return {
        "name": metadata.get("name", file_path.stem),
        "description": metadata.get("description", ""),
        "tags": metadata.get("tags", []),
        "triggers": metadata.get("triggers", []),
        "version": metadata.get("version", "1.0"),
        "body": body,
        "path": str(file_path),
        "full_content": content,
    }

def scan_skills(tag_filter: str | None = None) -> list[dict]:
    """扫描 skills 目录，返回所有 skill 的元信息"""
    skills = []
    if not SKILLS_DIR.exists():
        return skills
    
    for f in sorted(SKILLS_DIR.glob("*.md")):
        skill = parse_skill_file(f)
        # 可选：按标签过滤
        if tag_filter and tag_filter not in skill.get("tags", []):
            continue
        skills.append(skill)
    
    return skills

# ===== 生成 LangChain 工具 =====
@tool
def list_skills(category: str | None = None) -> str:
    """列出所有可用技能的名称和描述。
    不确定该用什么方法完成任务时先调用这个。
    可选传入 category 按标签过滤。"""
    skills = scan_skills(tag_filter=category)
    if not skills:
        return "暂无可用技能" if not category else f"没有 '{category}' 分类的技能"
    
    lines = []
    for s in skills:
        line = f"- {s['name']}: {s['description']}"
        if s.get("tags"):
            line += f"  [{', '.join(s['tags'])}]"
        lines.append(line)
    return f"共 {len(skills)} 个技能:\n" + "\n".join(lines)

@tool
def load_skill(skill_name: str) -> str:
    """加载指定技能的完整指令。加载后严格按照指令中的流程和格式执行任务。
    
    参数:
        skill_name: 技能名称（用 list_skills 查看可用名称）
    """
    path = SKILLS_DIR / f"{skill_name}.md"
    if not path.exists():
        available = [s["name"] for s in scan_skills()]
        return f"技能 '{skill_name}' 不存在。可用: {available}"
    
    skill = parse_skill_file(path)
    return skill["body"]  # 只返回正文，不返回 frontmatter

@tool
def search_skills(query: str) -> str:
    """根据任务描述搜索最匹配的技能。当技能很多、不确定用哪个时使用。"""
    skills = scan_skills()
    if not skills:
        return "暂无可用技能"
    
    # 简单关键词匹配（生产环境可换成向量检索）
    query_lower = query.lower()
    scored = []
    for s in skills:
        score = 0
        # 匹配名称
        if s["name"] in query_lower:
            score += 3
        # 匹配描述
        if any(word in s.get("description", "").lower() for word in query_lower.split()):
            score += 2
        # 匹配触发词
        for trigger in s.get("triggers", []):
            if trigger.lower() in query_lower:
                score += 4
        # 匹配标签
        for tag in s.get("tags", []):
            if tag in query_lower:
                score += 1
        if score > 0:
            scored.append((score, s))
    
    scored.sort(key=lambda x: -x[0])
    if not scored:
        return f"没有找到与 '{query}' 匹配的技能。用 list_skills 查看全部。"
    
    # 返回 top 3
    results = [f"- {s['name']}: {s['description']} (匹配度: {score})" for score, s in scored[:3]]
    return "推荐技能:\n" + "\n".join(results)

def build_skill_tools() -> list:
    """根据配置的策略返回对应的工具列表"""
    if LOAD_STRATEGY == "eager":
        # 少量技能：直接 list + load
        return [list_skills, load_skill]
    elif LOAD_STRATEGY == "rag":
        # 大量技能：搜索 + 加载
        return [search_skills, load_skill, list_skills]
    else:
        # 默认：全部提供
        return [list_skills, load_skill, search_skills]
```

## config/agent.yaml 中的 Skills 配置

```yaml
skills:
  directory: "skills"           # Skill 文件目录
  strategy: "on_demand"         # 加载策略: on_demand | eager | rag
  
  # 可选：按 Agent 分配不同技能
  agent_assignment:
    coder: ["code_review", "debugging"]
    writer: ["report_writing", "weekly_report"]
    researcher: ["data_analysis"]
```

## 组装到 Agent

```python
import asyncio
from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient
from src.loaders.skill_loader import build_skill_tools
from src.loaders.mcp_loader import create_mcp_client

async def build_agent():
    # 1. MCP 工具（外部能力：GitHub、数据库、API）
    mcp_client = await create_mcp_client()
    mcp_tools = await mcp_client.get_tools() if mcp_client else []

    # 2. Skills 工具（领域知识：SOP、模板、最佳实践）
    skill_tools = build_skill_tools()

    # 3. 本地工具（简单函数：计算器、时间）
    from src.tools.calculator import calculator
    local_tools = [calculator]

    # 4. 合并所有工具
    all_tools = [*mcp_tools, *skill_tools, *local_tools]
    print(f"[Agent] 工具总数: {len(all_tools)}")
    print(f"  - MCP: {len(mcp_tools)}")
    print(f"  - Skills: {len(skill_tools)}")
    print(f"  - 本地: {len(local_tools)}")

    # 5. 创建 Agent
    agent = create_agent(
        "openai:gpt-4o",
        tools=all_tools,
        system_message="""你是一个智能助手。

使用技能的原则：
- 遇到特定类型任务时，先用 list_skills 或 search_skills 查找匹配技能
- 找到后用 load_skill 加载完整指令
- 严格按照技能中的流程和输出格式执行
- 如果没有匹配技能，用自己的能力完成""",
    )
    return agent, mcp_client

async def main():
    agent, _ = await build_agent()
    
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "帮我审查这段代码: def add(a,b): return a+b"}]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

## 渐进式加载详解

### 为什么需要渐进式

```
假设你有 15 个 Skill，每个平均 800 字：
- 全塞进 system prompt: 15 × 800 = 12000 字 ≈ 6000 token（每轮都消耗）
- 渐进式加载: 列表 15 × 30 = 450 字 ≈ 200 token + 需要时加载 1 个 800 字

节省: 6000 → 600 token（省 90%）
```

### 三阶段模式（Discovery → Activation → Execution）

```python
# 阶段1: Discovery — 只看名字和描述（极省 token）
@tool
def discover_skills(task: str) -> str:
    """发现可用技能。返回技能列表（名称+一行描述），不返回完整内容。"""
    skills = scan_skills()
    return "\n".join(f"- {s['name']}: {s['description']}" for s in skills)

# 阶段2: Activation — 加载完整指令（按需消耗 token）
@tool
def activate_skill(skill_name: str) -> str:
    """激活技能，返回完整指令。加载后必须严格按照指令执行。"""
    path = SKILLS_DIR / f"{skill_name}.md"
    if not path.exists():
        return f"技能不存在: {skill_name}"
    return parse_skill_file(path)["body"]

# 阶段3: Execution — LLM 按照 Skill 内容执行（不需要额外工具）
```

### LLM 的完整调用链

```
用户: "帮我写一份季度报告"
  │
  ▼
LLM 第1轮思考: 写报告 → 有没有对应技能？
  → 调用 discover_skills("写季度报告")
  → 返回: "- report_writing: 结构化报告写作\n- weekly_report: 周报生成"
  │  （消耗 ~50 token）
  │
  ▼
LLM 第2轮思考: report_writing 最匹配
  → 调用 activate_skill("report_writing")
  → 返回: 完整的写作 SOP（角色、结构、格式要求）
  │  （消耗 ~400 token，但只在需要时）
  │
  ▼
LLM 第3轮: 按照 Skill 中的格式和流程写报告
  → 直接输出结果给用户
```

### 底层消息流转（逐轮拆解）

渐进式加载的本质是**两个 tool 控制信息密度**。以下是 LLM 每一轮实际看到的消息内容。

假设 skills 目录含 `code_review`、`report_writing`、`data_analysis`，用户说"帮我审查这段代码"。

**第 1 轮 — LLM 触发 Discovery**：

```
User: "帮我审查这段代码: def add(a,b): return a+b"
```

LLM 判断这是"代码审查"类任务 → 不确定用哪个技能 → 调 `list_skills`。

```python
# LLM 返回的消息
AIMessage(
    content="",    # 本轮无文本
    tool_calls=[{"name": "list_skills", "args": {}}]
)
```

**第 2 轮 — ToolNode 执行 → LLM 看目录**：

`list_skills()` 执行后，ToolMessage 追加到消息列表：

```python
# 此时 messages 变为：
[
    HumanMessage("帮我审查这段代码: def add(a,b): return a+b"),
    AIMessage(tool_calls=[{"name": "list_skills", "args": {}}]),
    ToolMessage(
        content="共 3 个技能:\n"
                "- code_review: 代码审查，按安全/性能维度审查\n"
                "- report_writing: 结构化报告写作\n"
                "- data_analysis: 数据分析与可视化",
        tool_call_id="call_1",
    ),
]
# ↑ LLM 只看到名称+一行描述，token 消耗 ~50
```

**第 3 轮 — Activation：加载完整指令**：

LLM 看到 `code_review` 最匹配，调 `load_skill("code_review")`。ToolNode 执行后追加：

```python
[
    ...前 3 条,
    AIMessage(tool_calls=[{"name": "load_skill", "args": {"skill_name": "code_review"}}]),
    ToolMessage(
        content="# 代码审查\n\n## 角色\n你是一位资深代码审查专家...\n\n"
                "## 审查流程\n1. 通读代码，理解整体逻辑\n"
                "2. 检查安全隐患（SQL注入、XSS、敏感信息泄露）\n"
                "3. 检查性能问题（N+1查询、内存泄漏）\n"
                "4. 检查代码规范（命名、注释、函数长度）\n"
                "5. 给出改进建议，按优先级排序\n\n"
                "## 输出格式\n| # | 严重程度 | 问题 | 建议 |\n",
        tool_call_id="call_2",
    ),
]
# ↑ LLM 现在有了完整 SOP —— 知道角色、步骤、输出格式，token 消耗 ~400
```

**第 4 轮 — Execution：按 SOP 执行**：

LLM 不再调工具，直接按 skill 流程输出：

```python
AIMessage(
    content="## 代码审查结果\n\n"
            "### 问题列表\n"
            "| # | 严重程度 | 问题 | 建议 |\n"
            "|---|---------|------|------|\n"
            "| 1 | 低 | 缺少类型提示 | 改为 def add(a: int, b: int) -> int: |\n"
            "| 2 | 低 | 缺少文档字符串 | 添加 docstring |\n\n"
            "### 总体评价\n- 评分：7/10\n- 函数简单，无安全隐患，缺少工程规范"
)
```

**token 消耗对比**：

```
                    全量加载方式    渐进式加载
                    ──────────    ──────────
用户消息                30             30
list_skills 结果        —              50
load_skill 结果         —             400
全部技能塞 prompt     ~3000             —
AI 最终回复            150            150
                    ──────────    ──────────
总计                 ~3180          ~630  （省 80%）
```

差距来源：全量方式每次对话都带 3 个 skill × 500 字 = 1500 字，15 个 skill 差距更悬殊（~6000 vs ~600 token，省 90%）。

### 为什么 tool 的 docstring 决定是否生效

渐进式能否被 LLM 正确理解，取决于两个 tool 的 docstring：

```python
@tool
def list_skills(category: str | None = None) -> str:
    """列出所有可用技能的名称和描述。
    不确定该用什么方法完成任务时先调用这个。"""
    # ↑ 关键：docstring 暗示 "这是第一步，不确定就先调它"
    ...

@tool
def load_skill(skill_name: str) -> str:
    """加载指定技能的完整指令。加载后严格按照指令中的流程和格式执行任务。

    参数:
        skill_name: 技能名称（用 list_skills 查看可用名称）
    """
    # ↑ 关键：docstring 暗示 "先 list 再 load"，暴露参数获取方式
    ...
    # ⚠️ docstring 中不要写死技能名（如 "加载 code_review 技能"），
    #   否则 LLM 可能跳过 list_skills 直接用硬编码的名字调 load_skill
```

### 3 个 vs 15 个的策略差异

```
3 个以内的技能：
  可以直接在 load_skill 的 docstring 里写死名字，
  省一次 list_skills 调用。

5 个以上：
  必须走渐进式（list → load），
  且 docstring 不能暴露名字，强制 LLM 先查目录。
```

## Multi-Agent 场景：不同 Agent 分配不同 Skills

```python
import yaml
from pathlib import Path

def get_agent_skills(agent_name: str) -> list:
    """根据配置为特定 Agent 加载分配的技能"""
    config = yaml.safe_load(Path("config/agent.yaml").read_text(encoding="utf-8"))
    assignment = config.get("skills", {}).get("agent_assignment", {})
    allowed_skills = assignment.get(agent_name, [])
    
    if not allowed_skills:
        # 没配置就加载全部
        return build_skill_tools()
    
    # 只暴露分配的技能
    @tool
    def list_my_skills() -> str:
        """列出我可用的技能"""
        skills = [s for s in scan_skills() if s["name"] in allowed_skills]
        return "\n".join(f"- {s['name']}: {s['description']}" for s in skills)
    
    @tool
    def load_my_skill(skill_name: str) -> str:
        """加载指定技能"""
        if skill_name not in allowed_skills:
            return f"你无权使用 '{skill_name}'。可用: {allowed_skills}"
        path = SKILLS_DIR / f"{skill_name}.md"
        return parse_skill_file(path)["body"] if path.exists() else "不存在"
    
    return [list_my_skills, load_my_skill]

# 使用：代码 Agent 只有代码相关技能
coder_tools = [*github_mcp_tools, *get_agent_skills("coder")]
coder_agent = create_agent("openai:gpt-4o", tools=coder_tools)
```

## RAG 匹配（技能 20+ 时）

```python
from langchain.vectorstores import InMemoryVectorStore
from langchain.embeddings import init_embeddings

class SkillRAG:
    """用向量检索自动匹配最相关的技能"""
    
    def __init__(self):
        self.embeddings = init_embeddings("openai:text-embedding-3-small")
        self.vectorstore = None
        self._build_index()
    
    def _build_index(self):
        """把所有 Skill 的描述向量化"""
        skills = scan_skills()
        if not skills:
            return
        
        texts = []
        metadatas = []
        for s in skills:
            # 用 name + description + triggers 作为检索文本
            search_text = f"{s['name']} {s['description']} {' '.join(s.get('triggers', []))}"
            texts.append(search_text)
            metadatas.append({"skill_name": s["name"]})
        
        self.vectorstore = InMemoryVectorStore.from_texts(
            texts, self.embeddings, metadatas=metadatas
        )
    
    def match(self, query: str, top_k: int = 1) -> list[str]:
        """根据用户任务描述匹配最相关的技能名"""
        if not self.vectorstore:
            return []
        results = self.vectorstore.similarity_search(query, k=top_k)
        return [doc.metadata["skill_name"] for doc in results]

# 封装成工具
skill_rag = SkillRAG()

@tool
def auto_match_skill(task_description: str) -> str:
    """根据任务描述自动匹配最相关的技能并返回完整指令。"""
    matches = skill_rag.match(task_description, top_k=1)
    if not matches:
        return "没有找到匹配的技能"
    
    skill_name = matches[0]
    path = SKILLS_DIR / f"{skill_name}.md"
    skill = parse_skill_file(path)
    return f"[自动匹配: {skill_name}]\n\n{skill['body']}"
```

## 与 MCP Prompts 的配合（远程 Skill）

Skills 也可以放在 MCP 服务器里，通过 `@mcp.prompt()` 暴露：

```python
# ===== MCP 服务器端（集中管理 Skills）=====
from mcp.server.fastmcp import FastMCP
from pathlib import Path

mcp = FastMCP("skill-server")
SKILLS_DIR = Path("skills")

@mcp.prompt()
def code_review(code: str, language: str = "Python") -> str:
    """代码审查技能。传入代码，返回审查指令。"""
    template = (SKILLS_DIR / "code_review.md").read_text(encoding="utf-8")
    return f"{template}\n\n## 待审查代码（{language}）\n```{language.lower()}\n{code}\n```"

@mcp.prompt()
def report_writing(topic: str, style: str = "专业") -> str:
    """报告写作技能。传入主题，返回写作指令。"""
    template = (SKILLS_DIR / "report_writing.md").read_text(encoding="utf-8")
    return f"{template}\n\n## 写作任务\n主题: {topic}\n风格: {style}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```python
# ===== Agent 端（调用远程 Skill）=====
# 方式1：启动时获取，作为 system message
prompt_result = await client.get_prompt(
    server_name="skill-server",
    prompt_name="code_review",
    arguments={"code": "def add(a,b): return a+b"},
)
system_text = prompt_result[0].content

# 方式2：封装成工具，LLM 按需调用
@tool
async def get_remote_skill(skill_name: str, **kwargs) -> str:
    """从远程技能服务器获取技能模板"""
    result = await client.get_prompt("skill-server", skill_name, kwargs)
    return result[0].content
```

**本地 Skill vs 远程 Skill（MCP）选择：**

| 维度 | 本地 Skill（文件） | 远程 Skill（MCP） |
|------|-------------------|-------------------|
| 修改方式 | 编辑 .md 文件 | 改 MCP 服务器代码 |
| 多 Agent 共享 | 每个 Agent 各自一份 | 集中一份，多端调用 |
| 参数化 | 不支持（纯文本） | 支持（模板变量） |
| 部署复杂度 | 零（放文件） | 需要启动 MCP 服务器 |
| 适用场景 | 单 Agent、个人项目 | 多 Agent、团队、SaaS |

## 用户扩展方式

| 操作 | 方式 | 改代码？ |
|------|------|---------|
| 添加新 Skill | 往 `skills/` 放 .md 文件 | 不用 |
| 修改 Skill | 编辑对应 .md 文件 | 不用 |
| 删除 Skill | 删除 .md 文件 | 不用 |
| 改 Skills 目录路径 | 改 `config/agent.yaml` | 不用 |
| 给特定 Agent 分配 Skill | 改 `agent.yaml` 的 `agent_assignment` | 不用 |
| 切换加载策略 | 改 `agent.yaml` 的 `strategy` | 不用 |

## 常见陷阱

### 1. Skill 内容太长，超出上下文窗口

```python
# 错误：Skill 文件 5000 字，加载后占满上下文
# 正确：Skill 控制在 1000 字以内，只写核心 SOP
# 如果确实很长，拆分成多个子 Skill
```

### 2. LLM 不主动调用 list_skills

```python
# 解决：在 system prompt 中明确指示
system_message = """...
重要：遇到以下类型任务时，必须先调用 list_skills 查找匹配技能：
- 代码审查、Bug 分析
- 报告写作、文档生成
- 数据分析、可视化
不要跳过这一步直接回答。"""
```

### 3. Skill 文件名和 name 字段不一致

```python
# 错误：文件叫 code-review.md，frontmatter 里 name: code_review
# LLM 调用 load_skill("code_review") 会找不到文件

# 正确：文件名和 name 保持一致，都用下划线
# 文件名: code_review.md, name: code_review
```

### 4. 忘记处理文件不存在的情况

```python
# 错误：直接 read_text，文件不存在就崩溃
content = (SKILLS_DIR / f"{name}.md").read_text()

# 正确：先检查存在性
path = SKILLS_DIR / f"{name}.md"
if not path.exists():
    return f"技能不存在。可用: {[s['name'] for s in scan_skills()]}"
```

## 各方案选择总结

| 技能数量 | 推荐方案 | 工具组合 | 原因 |
|---------|---------|---------|------|
| 1-5 个 | 直接加载 | `list_skills` + `load_skill` | 简单直接 |
| 5-20 个 | 渐进式 | `discover_skills` + `activate_skill` | 省 token |
| 20+ 个 | RAG 匹配 | `auto_match_skill` | 自动匹配，不用浏览列表 |
| 远程/多 Agent | MCP Prompts | `get_remote_skill` | 集中管理，参数化 |
| 混合场景 | 本地 + MCP | 本地 list + 远程 get | 常用的本地，共享的远程 |

## 相关

- [[agent/skills/Skills|<- Skills 概念]] — Skills 的定义、与 MCP/Tools 的区别
- [[langchain/tools/工具来源与集成|工具来源与集成]] — Skills vs Tools 对比
- [[langchain/modules/langchain-mcp/MCP提示词编写与执行流程|MCP 提示词]] — MCP Prompts 作为远程 Skill
- [[langchain/modules/langchain-mcp/MCP服务器开发|MCP 服务器开发]] — @mcp.prompt() 用法
