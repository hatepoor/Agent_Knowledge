---
tags: [meta, 规范, 约定, schema, 操作规范]
type: meta
created: 2026-09-19
updated: 2026-09-19
aliases:
  - SCHEMA
  - 知识库操作规范
---

# SCHEMA · 知识库操作规范

> 本页是全库的**统一操作规范（Schema）**。所有页面都应遵守，新增/修订内容前先读本页。
> 结构总览另见 [[meta/知识库三层架构]] 与 [[meta/概览]]。

## 一、通用规则

- **集中式组织**：同一主题的所有正文收敛到一个权威目录，错误/拼写目录不建；多处并存时以导航声明的权威目录为准。
- **避免断链**：移动/重命名后须用 wikilink 在两处留转发指引，确保旧链接可达。
- **新增内容归位**：新页面必须放进对应分层/子目录，不零散堆放。
- **版本信息**：每个页面 frontmatter 维护 `created` / `updated`（`YYYY-MM-DD`）。

## 二、Frontmatter（YAML 头）

每篇都以 `---` 包裹的 YAML frontmatter 开头，字段约定：

| 字段 | 说明 | 示例 |
|------|------|------|
| `tags` | 主题标签，含 `type`、领域、关键词 | `tags: [concept, RAG, 分块]` |
| `type` | 页面类型 | `concept` / `howto` / `entity` / `source` / `reference` / `moc` / `navigation` / `meta` |
| `created` / `updated` | 创建/更新时间 | `created: 2026-09-19` |
| `aliases` | 别名（用于旧名/断链兼容） | `aliases: [SCHEMA]` |
| `source_url` | 搬运来源时记录原地址 | 仅 `type: source` 页面 |

## 三、交叉引用（wikilink）

- 一律使用 Obsidian `[[路径/无扩展名|显示名]]` 撰写，不用裸 URL。
- 相对根目录写全路径：`[[rag/RAG_学习路径导航]]`。
- **孤儿页禁止**：每篇要有被引关系；概览/导航页负责把下属页面连通。
- 匿名旧路径以转发页 + 对照表承接（见 [[agent/RAG/README|RAG 迁移对照表]] 示例）。

## 四、文档模板约定

- **入门/概念类**：`学习目标 → 前置知识 → 正文 → 小结 → 练习 → 相关页面`。
- **Agent 模式族**：`定义 → 核心循环 → 对比 → 优劣 → 适用场景`。
- **RAG 主题**：倾向 `一问一答` 形式，增强入门关联性（见 [[rag/00_导览与基础/RAG零基础一问一答]]）。
- **专题类**：`专题概览 → 语法速查 → 实战案例 → 常见陷阱`，可在单文件内聚。

## 五、目录组织

```
wiki/
├─ meta/           # 全库规范、方法论、总览
├─ agent/          # Agent 领域（patterns/orchestration/memory/...）
├─ rag/            # RAG 领域（00_导览与基础 … 14_范式与前沿）
├─ langchain/      # 框架
├─ postgresql/     # 数据库（入门→中段→进阶三阶段）
├─ python/         # 语言（basics/concurrency/...）
└─ tools/          # 各类工具
```

> 子目录统一小写、用 `-` 连接；导航文件用 `00-` 前缀保持排序；不建空目录。

## 相关页面

- [[meta/知识库三层架构]] — 层级与归属总览
- [[meta/Welcome]] — 最顶层入口
- [[meta/概览]] — 全局概览
- [[postgresql/README]] — 阶段导航示例（含本页引用）
- [[agent/RAG/README]] — 目录级转发/迁移对照示例