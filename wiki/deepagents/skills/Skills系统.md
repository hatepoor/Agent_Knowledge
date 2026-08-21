---
tags: [deepagents, skills, middleware]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# DeepAgents Skills 系统

DeepAgents 内置的 Skills 系统——通过 `SkillsMiddleware` 和 `skills/` 目录，Agent 按需加载 Markdown 技能文件。

## 一、与手动 Skills 的区别

| 维度 | 手动 Skills（agent/skills 方案） | DeepAgents SkillsMiddleware |
|------|--------------------------------|---------------------------|
| 加载方式 | 自定义 `@tool` (list/load/search) | `SkillsMiddleware` 自动注入 |
| 文件格式 | Markdown + YAML frontmatter | 同，但支持 Agent Skills 规范 |
| 工具暴露 | 3 个自定义 tool | 零额外 tool（内嵌到中间件） |
| 配置方式 | 手写 loader 代码 | `skills=["code_review"]` 参数 |

## 二、使用方法

```python
from deepagents import create_deep_agent

# 指定要加载的技能
agent = create_deep_agent(
    model=llm,
    skills=["code_review", "report_writing", "data_analysis"],
    # ↑ 自动从 skills/ 目录加载对应 .md 文件
)

# skills/ 目录结构
# skills/
# ├── code_review.md
# ├── report_writing.md
# └── data_analysis.md
```

**也支持路径指定**：

```python
agent = create_deep_agent(
    model=llm,
    skills=[
        "./my_skills/code_review.md",
        "./my_skills/report_writing.md",
        "https://example.com/shared_skill.md",  # 支持远程
    ],
)
```

## 三、Skill 文件格式（Agent Skills 规范）

遵循 [agentskills.io](https://agentskills.io/specification) 规范：

```markdown
---
name: code_review
description: 按安全/性能/规范三维度审查代码
triggers: [审查代码, review, code review]
tags: [代码, 审查, 开发]
version: 1.0
---

# 代码审查

## 角色
你是一位资深代码审查专家。

## 工作流程
1. 通读代码，理解整体逻辑
2. 检查安全隐患（SQL注入、XSS、敏感信息）
3. 检查性能问题（N+1查询、内存泄漏）
4. 检查代码规范（命名、注释、函数长度）

## 输出格式
| # | 严重程度 | 维度 | 问题 | 建议 |
|---|---------|------|------|------|
```

## 四、SkillsMiddleware 工作原理

`SkillsMiddleware` 在 Agent 初始化时：

1. 扫描指定的 `skills` 列表
2. 解析每个 skill 的 frontmatter（name/description/triggers）
3. 把 skill 元信息注入 Agent 的 system prompt
4. 把 skill 正文注入 Agent 的上下文（需要时才完整加载）

**中间件栈中的位置**：在 `TodoListMiddleware` 之后、`FilesystemMiddleware` 之前（第 2 层），确保技能元数据在文件工具运行前可用。

## 五、v0.5.0 CLI Skills

```bash
# 启动时加载技能
deepagents --skill code_review --skill report_writing

# CLI 内按需调用
/skill:code_review
```

## 六、相关页面

- [[deepagents/middleware/中间件栈概述]] — SkillsMiddleware 在栈中的位置
- [[agent/skills/Skills实现与集成]] — 手动 Skills 实现方案
- [[agent/skills/Skills]] — Skills 概念定义
