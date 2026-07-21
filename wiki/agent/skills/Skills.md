---
tags: [concept, AI, Agent, Skills, 工程, 能力模块]
created: 2026-07-01
updated: 2026-07-01
type: concept
---

# Skills（AI Agent 技能）

## 定义

Skills 是教 AI 按固定流程做事的**操作说明书**，以 Markdown 文件形式存在。一旦写好，就能像函数一样被反复调用。Skills 的本质是把"某类事情应该怎么专业做"封装成一个**可复用、可自动触发的能力模块**。

Skills 与传统 Prompt 最大的区别是**按需加载 + 渐进式披露**——只在需要时才把完整的 SOP 加载到上下文中，极大节省 token。

## 与 Prompt、MCP 的区别

| 对比项 | 普通 Prompt | Skills | MCP |
|--------|-------------|--------|-----|
| 用途 | 单次任务指令 | 知识复用（经验、SOP） | 能力扩展（API、数据库） |
| 上下文占用 | 每次全量塞入 | 渐进式加载 | 启动时加载全部工具定义 |
| 一致性 | 依赖每次 prompt 质量 | 高（固定 SOP + 模板） | N/A |
| 复用性 | 手动复制粘贴 | 自动匹配 / slash 命令 | 协议级标准化 |
| 维护方式 | 改一次 prompt 重新发 | 修改 SKILL.md 文件即生效 | 修改服务端代码 |
| 技术门槛 | 无 | 低（纯 Markdown） | 中（需编码+服务器配置） |

**一句话理解**：Prompt 是一次性指令，MCP 是装软件，Skills 是给 AI 一套岗位培训大礼包。

## 核心结构

### 最简结构
```
my-skill/
└── SKILL.md   （唯一必需）
```

### SKILL.md 格式

```yaml
---
name: pdf-processing
description: 从 PDF 中提取文本和表格，填写表单，并合并文档
---
# PDF 处理

## 使用场景
当需要对 PDF 文件进行操作时使用...

## 提取文本
- 使用 pdfplumber 提取文本型 PDF 内容
...
```

### 元数据字段

| 字段            | 必需  | 说明                       |
| ------------- | --- | ------------------------ |
| name          | 是   | 技能名，最长 64 字符，小写字母+数字+连字符 |
| description   | 是   | 功能与使用场景说明，最长 1024 字符     |
| license       | 否   | 许可证名称                    |
| compatibility | 否   | 环境与依赖说明                  |
| metadata      | 否   | 自定义键值对（作者、版本号等）          |
| allowed-tools | 否   | 允许使用的工具列表                |

### 进阶目录结构
```
my-skill/
├── SKILL.md       # 必需：指令 + 元数据
├── templates/     # 可选：常用模板
├── examples/      # 可选：优秀/反例
├── references/    # 可选：规范文档
└── scripts/       # 可选：可执行脚本
```

在 SKILL.md 中按需引用这些文件，AI 会渐进式加载而非一次性塞入上下文。

## 工作机制：渐进式加载

Skills 用三阶段高效管理上下文：

1. **发现（Discovery）**：启动时，AI 只加载每个 Skill 的**名称和描述**，保留最基本的识别信息
2. **激活（Activation）**：当任务匹配某个 Skill 的描述时，AI 才把完整的 SKILL.md 指令读入上下文
3. **执行（Execution）**：AI 按指令执行，按需加载参考文件或运行代码

这种设计让 AI 保持快速响应，同时能在需要时获取完整知识。

## 在 Claude Code 中的使用

### Skill 查找优先级

| 级别 | 路径 | 生效范围 |
|------|------|----------|
| 企业级 | 管理控制台配置 | 组织内所有用户 |
| 个人级 | `~/.claude/skills/<skill-name>/SKILL.md` | 所有项目 |
| 项目级 | `.claude/skills/<skill-name>/SKILL.md` | 仅当前项目 |
| 插件级 | `<plugin>/skills/<skill-name>/SKILL.md` | 启用该插件的环境 |

### 支持 Skills 的主流客户端

| 工具 | 免费 | 推荐人群 | 备注 |
|------|------|----------|------|
| Claude Code | 是 | 所有人 | 标准制定者，生态最全 |
| Cursor | 是 | 写代码最常用 | 几乎无缝兼容 |
| Trae / OpenCode | 是 | 追求性价比 | 国内用户较多 |
| VS Code + 插件 | 部分 | 深度 VS Code 用户 | 快速跟进中 |

## 与 Agent 架构的关系

在 [[agent/harness/Agent_六种架构]] 和 [[agent/prompt/上下文工程]] 的视角下，Skills 解决的是**上下文效率**问题：
- 传统方式：每次把所有规则和 SOP 塞进系统提示 → token 浪费严重
- Skills 方式：按需加载，只在触发时读入完整指令 → 上下文预算更充裕

Skills 是 [[agent/prompt/上下文工程]] 中**系统提示工程化**和**渐进式披露**模式的具体实现方案。

## 相关概念
- [[agent/skills/Skills实现与集成|Skills 实现与集成]] — 在 LangChain Agent 中的代码实现（目录扫描、渐进式加载、组装）
- [[langchain/modules/langchain-mcp/MCP协议概述]] — Skills 用于知识复用，MCP 用于能力扩展，二者互补
- [[agent/prompt/上下文工程]] — Skills 是上下文工程的具体实践方案
- [[agent/prompt/提示词工程]] — Skills 超越了单次提示词，实现了持久化复用
- [[agent/harness/Agent_六种架构]] — Skills 可增强任何架构中 Agent 的专业能力

## 来源
- [[agent/sources/菜鸟教程_AI_Agent系列]] — Skills 教程篇
