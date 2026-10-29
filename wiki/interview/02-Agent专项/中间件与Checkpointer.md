---
tags: [interview, Agent, 中间件, Checkpointer, Store, 面试]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# 中间件与 Checkpointer

> **什么时候该读**：⭐ 这是你项目里的**真实生产实践**（SummarizationMiddleware + PostgresSaver），面试极有说服力。
> 完整机制在 [[langchain/middleware/中间件概述]]。

## 面试定位

⚠️ **middleware 是 LangChain v2 的重要特性**，很多人还不知道。答好这题能体现「你在跟进新版本」。

---

## 一、Middleware：横切关注点怎么插进去

### 一句话

> Middleware 是在 **Agent 循环的特定钩子**上插入自定义逻辑的机制，不用改 Agent 本身。

### 三类钩子（⭐ 必答）

| 钩子 | 时机 | 典型用途 |
|------|------|---------|
| ⭐ **`before_model`** | 调 LLM **前** | 改 prompt、注入上下文、限流 |
| ⭐ **`after_model`** | 调 LLM **后** | 校验输出、脱敏、重试 |
| ⭐ **`wrap_tool_call`** | 工具调用**前后包裹** | 计费、审计、超时、降级 |

⚠️ **追问：`wrap_tool_call` 为什么是「包裹」而不是「前后」两个钩子？**
→ ⭐ **因为要能包住「整个工具执行过程」**——可以在前后都做事，还能在异常时兜底，甚至替代原调用。

### 内置中间件（你用过哪个说哪个）

| 中间件 | 作用 | 你是否用过 |
|--------|------|-----------|
| ⭐ **`SummarizationMiddleware`** | ⭐ **超阈值时压缩历史** | ✅ 项目里用了 |
| `HumanInTheLoopMiddleware` | 人工审批敏感操作 | |
| `ModelFallbackMiddleware` | 主模型失败降级到备用模型 | |
| `ModelRetryMiddleware` | 模型调用重试 | |
| `ToolCallLimitMiddleware` | 限制单个工具调用次数 | |
| `PIIMiddleware` | 敏感信息过滤 | |
| `TodoListMiddleware` | 任务清单管理 | |

📌 [[langchain/middleware/SummarizationMiddleware]] · [[langchain/middleware/钩子机制详解]]

### ⭐ 你项目里的实践（重点讲这段）

```
「我的 Agent 跑长任务时上下文会爆炸——每轮工具结果都追加，十几轮后
token 超限。

用了 SummarizationMiddleware：超过阈值时把早期的对话历史压缩成摘要，
保留最近几轮原文。这样长任务能持续跑，不会因为上下文满而中断。

两个注意点：
① 摘要会丢细节，所以关键信息（如用户偏好、任务目标）我外置到 Store，
   不依赖摘要保留。
② 压缩时机要 tune——太早丢上下文，太晚已经超限了。」
```

⭐ **「外置到 Store」这句是亮点**——它说明你理解「摘要不是银弹」。

---

## 二、Checkpointer：状态持久化

### 一句话

> Checkpointer 把图的状态按 `thread_id` 持久化，支持**多轮对话、中断恢复、时间旅行**。

### 后端选型（你两个都用过）

| 后端 | 特点 | 场景 |
|------|------|------|
| `InMemorySaver` | 内存，重启丢 | 开发/测试 |
| ⭐ `SqliteSaver` | 单机文件 | ⭐ 本地应用、轻量部署 |
| ⭐ `PostgresSaver` | 多进程共享 | ⭐ **生产** |

⚠️ **追问：`thread_id` 是什么？**
→ ⭐ **会话标识**。同一个 `thread_id` 的所有调用共享状态；不同 `thread_id` 互相隔离。**多用户场景靠它隔离。**

⚠️ **追问：多租户怎么隔离？**
→ ⭐ **`thread_id` 加租户前缀**（如 `tenant_123:thread_456`），并在检索时强制带上前缀。⚠️ 光靠 `thread_id` 不够——**要防跨租户猜测 ID**。

📌 [[langchain/modules/langgraph_checkpoints/多轮对话与thread_id]]

---

## 三、Store：长期记忆（和 Checkpointer 的区别 ⭐ 高频题）

⚠️ **这题极容易混，务必分清：**

| | Checkpointer | Store |
|---|---|---|
| 存什么 | ⭐ **会话状态**（当前对话的中间态） | ⭐ **跨会话的长期记忆** |
| 生命周期 | 随 `thread_id` | ⭐ **跨 thread 持久** |
| 典型内容 | messages、图状态 | 用户偏好、事实、学到的知识 |
| 访问方式 | 自动（框架恢复） | ⭐ **显式 get/put** |
| 类比 | 短期记忆 | 长期记忆 |

```python
from langgraph.store.base import BaseStore

# Store 的典型用法：命名空间隔离
store.put(("users", user_id), "preference", {"lang": "zh"})
store.get(("users", user_id), "preference")
```

⚠️ **追问：为什么不能把长期记忆放 Checkpointer？**
→ ⭐ **因为 Checkpointer 随 thread 走**——新会话（新 thread_id）读不到。而且状态会被摘要压缩，不适合存需要永久保留的东西。

📌 [[langchain/memory/长期记忆]] · [[langchain/modules/langgraph_stores/命名空间设计]]

---

## 四、长短记忆的设计（⭐ 你项目的核心，值得展开讲）

按 memory 记录，你的系统有「长短记忆设计」。建议这样组织回答：

```
「我的记忆分三层：

① 短期（Checkpointer）：当前会话的对话历史和图状态，用 PostgresSaver
   持久化。thread_id 做会话隔离。

② 压缩（SummarizationMiddleware）：短期记忆超限时摘要化，保证长任务
   不中断。

③ 长期（Store）：跨会话的用户偏好、任务历史、常用上下文。按
   (用户ID, 类型) 命名空间隔离。

关键设计：长期记忆不依赖摘要保留——摘要会丢细节，所以我把「必须
记住的」显式写进 Store，而不是指望压缩时留下。」
```

⚠️ **如果实际实现和这不同，请按实际改**——这段是模板，不是事实。

---

## 五、高频追问速查

| 追问 | 答法要点 |
|------|---------|
| middleware 有几类钩子？ | ⭐ `before_model` / `after_model` / `wrap_tool_call` |
| `wrap_tool_call` 能做什么？ | 包裹整个工具执行：计费、审计、超时、降级、替代调用 |
| 上下文一直涨怎么办？ | ⭐ SummarizationMiddleware + 关键信息外置 Store |
| Checkpointer 和 Store 的区别？ | ⭐ 短期会话状态 vs 长期跨会话记忆 |
| 为什么生产用 PostgresSaver？ | 多进程/多实例共享状态 |
| 多租户怎么隔离？ | ⭐ thread_id 加租户前缀 + 检索强制带前缀 |
| `interrupt()` 之后能退出进程吗？ | ⭐ 能，状态已持久化，同 thread_id 续跑 |
| 摘要压缩有什么风险？ | ⭐ 丢细节。关键信息必须外置，不能指望摘要保留 |
| 自定义中间件怎么写？ | 继承基类，实现对应钩子，注意 `wrap_tool_call` 要处理异常路径 |

---

## 六、自测

- [ ] 说出三类钩子及各自典型用途
- [ ] `wrap_tool_call` 为什么设计成「包裹」而不是前后两个钩子？
- [ ] ⭐ Checkpointer vs Store：四个维度对比
- [ ] 你项目里的 SummarizationMiddleware 解决什么问题？有什么代价？
- [ ] 多租户隔离怎么做？为什么光靠 thread_id 不够？
- [ ] 能完整讲出「长短记忆三层设计」

---

## 导航

[[interview/README|面试知识库首页]] · 上一篇：[[langgraph/interview/00-面试总纲与答题框架]] · 下一篇：[[interview/02-Agent专项/Skills渐进加载]] · 详解：[[langchain/middleware/中间件概述]]
