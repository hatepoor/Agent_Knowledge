---
tags: [interview, Agent, ReAct, 面试, 专项]
type: guide
created: 2026-10-13
updated: 2026-10-13
---

# ReAct 与 Agent 循环

> **什么时候该读**：⭐ Agent 岗面试**必问的第一题**。答不好这题，后面的都不用聊了。
> 完整机制在 [[langchain/agents/Agent循环实现]]，本页给「面试怎么答」。

## 面试定位

⚠️ **这题考察三件事**：① 你懂不懂原理；② 知不知道边界；③ 有没有踩过坑。
**只答「Reason + Act 交替执行」是及格线以下**——那是背定义。

---

## 一、标准答法（30 秒版）

> ReAct 是「**推理（Reason）+ 行动（Act）交替**」的范式：模型先思考「我现在需要什么」，再决定调用哪个工具，拿到观察结果后继续思考，直到能给出最终答案。

**接着必须展开的三点**（否则显得只会背）：

| 点 | 内容 |
|---|---|
| ① 为什么需要 | ⭐ **纯 CoT 只思考不行动，无法获取外部信息；纯工具调用不会规划步骤**。ReAct 把两者接起来 |
| ② 循环终止 | 模型输出**不再请求工具**（直接给最终答案）或达到最大步数 |
| ③ 观察怎么进上下文 | ⭐ 工具结果作为新消息追加，下一轮模型能看到 |

---

## 二、展开版（2 分钟）

### 一次完整的循环长什么样

```
用户：「北京明天天气怎么样，需要带伞吗？」

── 第 1 轮 ──────────────────────────────
Thought: 我需要先查北京明天的天气
Action:   get_weather(city="北京", date="明天")

── 第 2 轮 ──────────────────────────────
Observation: 小雨，18~22°C，降水概率 80%
Thought: 有小雨且降水概率高，需要带伞
Final Answer: 北京明天小雨，18~22°C，降水概率 80%，建议带伞
```

⭐ **关键观察**：`Observation` 不是模型「幻想」的，是**工具真实返回**后由框架拼进上下文的。**这是 Agent 和「让模型直接答」的本质区别。**

### ReAct 的四种变体（被追问时用）

| 变体 | 区别 |
|------|------|
| **标准 ReAct** | Thought → Action → Observation |
| **ReAct + CoT** | 推理过程更长更细 |
| **Plan-and-Execute** | ⭐ **先出完整计划再逐步执行**，适合长任务（详见 [[agent/patterns/PlanAndExecute]]） |
| **Reflexion** | ⭐ 失败后把教训写进记忆，重试时复用 |

⚠️ **追问「ReAct 和 Plan-and-Execute 怎么选？」**
→ ⭐ **ReAct 灵活但可能跑偏（每步独立决策，误差累积）；Plan-and-Execute 有全局视野但计划可能过时。** 实践中常混合：先粗规划，再 ReAct 执行，中途允许修正计划。

---

## 三、LangChain 里的实现（你项目里就是这么做的）

### `create_agent` 的循环

```python
from langchain.agents import create_agent

agent = create_agent(
    model="deepseek-chat",
    tools=[search_tool, calculator],
    # ⚠️ 关键：这就是 ReAct 循环的载体
)
```

**内部发生什么**（⭐ 这是最容易被追问的）：

```
① 组装 system prompt + 用户消息 + 工具描述（function schema）
② 调 LLM → 模型返回 tool_calls 或直接返回文本
③ 有 tool_calls → 执行工具，结果作为 ToolMessage 追加
④ 回到 ②，直到模型不再请求工具 或 达到 recursion_limit
```

⚠️ **追问：`recursion_limit` 是什么？**
→ LangGraph 里默认 25。⚠️ **它是防「无限循环烧钱」的保险丝**——模型可能反复请求同一个工具。生产必须设。

### 工具是怎么被模型「看见」的

⭐ **这是高频深挖点**：

| 环节 | 内容 |
|------|------|
| 工具定义 | 函数签名 + docstring → 转成 JSON Schema（`name`/`description`/`parameters`） |
| 注入位置 | system prompt 或专门的 tools 字段 |
| 模型决策 | ⭐ **模型只输出「调用哪个工具 + 参数」**，不真正执行 |
| 真正执行 | ⭐ **框架负责调用，结果回填上下文** |

⚠️ **追问：工具描述写得不好会怎样？**
→ ⭐ **模型选错工具或参数错。** 我项目里的实践：`description` 写清「什么时候**不**该用这个工具」+ 给 2~3 个 `examples`。

📌 [[langchain/tools/工具]]

---

## 四、生产环境的四个坑（⭐ 这是区分「用过」和「玩过」的地方）

### 坑 1：无限循环

**现象**：模型反复请求同一个工具，或两个工具互相调用。

**根因**：① 工具返回了模型不理解的内容；② 任务本身无法完成但模型不肯放弃。

**解法**：
- ⭐ `recursion_limit` 硬限制
- ⭐ **检测「连续 N 次相同 tool_call」主动中断**
- 工具报错时返回**可读的错误信息**（而不是抛异常中断）

### 坑 2：上下文爆炸

**现象**：跑十几轮后 token 超限或成本飙升。

**根因**：每轮的 Observation 都追加进上下文，只增不减。

**解法**：⭐ **`SummarizationMiddleware`**（我在项目里用了）——超过阈值时把早期历史压缩成摘要。

📌 [[langchain/memory/上下文压缩]]

### 坑 3：工具失败的处理

**现象**：工具抛异常 → 整个 Agent 崩了。

**解法**：⭐ **工具内部兜底，返回可读错误**，让模型自己决定重试还是换方案：
```python
@tool
def query_db(sql: str) -> str:
    """查询数据库。"""
    try:
        return run(sql)
    except Exception as e:
        return f"查询失败：{e}。请检查 SQL 语法或换一种问法。"
        # ⭐ 不要 raise，要让模型看到错误
```

### 坑 4：可观测性缺失

**现象**：Agent 答错了，但不知道它在哪一步开始错的。

**解法**：⭐ **流式输出 + 结构化日志**。我在项目里用 loguru 记录每一轮的 Thought / Action / Observation，排查时能直接定位。

---

## 五、高频追问速查

| 追问 | 答法要点 |
|------|---------|
| ReAct 和 Function Calling 什么关系？ | ⭐ **Function Calling 是底层机制，ReAct 是上层范式。** ReAct 用 FC 实现「请求工具」这一步 |
| 为什么模型不会自己执行工具？ | ⭐ **模型只生成文本。执行必须由框架做**——这是安全边界 |
| 怎么防止 Agent 跑偏？ | 强 system prompt + recursion_limit + 相同调用检测 + 人工介入点 |
| 多轮之后性能下降怎么办？ | 上下文压缩（摘要化）+ 关键信息外置到 Store |
| 怎么评估 Agent 好不好？ | ⭐ **不能只看最终答案**，要看：任务完成率、步数效率、工具选择准确率、失败恢复能力 |
| 流式有什么用？ | ⭐ **用户体验（实时看到思考过程）+ 长任务不超时** |
| Thought 一定要显式输出吗？ | 不必须。但显式输出**可解释、可调试**，生产中建议保留 |

---

## 六、自测

⚠️ **合上页面脱稿说：**

- [ ] 用 30 秒说清 ReAct，并说出「为什么需要它」
- [ ] 画出一次完整循环，标出 Observation 是谁生成的
- [ ] ReAct vs Plan-and-Execute：怎么选，为什么
- [ ] LangChain 的 `create_agent` 内部循环有哪四步？
- [ ] 工具是怎么被模型「看见」的？描述写得不好会怎样？
- [ ] ⭐ 说出生产环境的四个坑及解法
- [ ] ReAct 和 Function Calling 的关系？
- [ ] 为什么模型不会自己执行工具？（安全边界）

---

## 导航

[[interview/README|面试知识库首页]] · 上一篇：[[interview/01-八股基础/网络与操作系统八股]] · 下一篇：[[langgraph/interview/00-面试总纲与答题框架]] · 详解：[[langchain/agents/Agent循环实现]]
