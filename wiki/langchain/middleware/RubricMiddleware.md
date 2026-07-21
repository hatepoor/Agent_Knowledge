---
tags: [langchain, middleware, rubric, grading, deepagents, beta]
created: 2026-07-15
updated: 2026-07-15
type: source
source_url: https://docs.langchain.com/oss/python/langchain/middleware/built-in#rubric
langchain_version: ">=1.0"
deepagents_version: ">=0.6.5"
status: beta
---

# RubricMiddleware（评分量规中间件）

## 一、定义与作用

`RubricMiddleware`（**Beta 版本**）让 Agent 在声明"任务完成"前**根据预设的评分量规自评**，如果不满足则继续迭代。

它把"完成"从一个**主观判断**变成一个**可量化的标准**：
- Agent 完成任务后提出："我满足了这些标准"
- RubricMiddleware 用 LLM 评估
- 不满足则强制 Agent 继续

**核心价值**：自动化质量控制。无需人类评估，Agent 自己把好"质量关"。

## 二、使用场景

| 场景 | 案例 |
|------|------|
| 内容创作 | "必须含 3 个数据点、2 个例子、1 个反例" |
| 代码生成 | "必须通过 pytest、有类型注解、< 100 行" |
| 客户服务 | "必须确认用户问题、提供解决方案、给出时间预期" |
| 报告生成 | "必须有摘要、详细数据、结论" |

**不适合**：
- 开放性任务（无明确"对"的标准）
- 实时性要求高的任务（自评会拖慢响应）

## 三、参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `BaseChatModel` | 用于评分的模型（**不能是字符串**） |
| `max_iterations` | `number` | 默认 3，最大迭代次数 |

## 四、使用方法

```python
# pip install "deepagents>=0.6.5"
from langchain.agents import create_agent
from deepagents.middleware import RubricMiddleware
from langchain_deepseek import ChatDeepSeek

# 用于评分的模型
grading_llm = ChatDeepSeek(model="deepseek-v4-flash", temperature=0.0)

agent = create_agent(
    model="deepseek-v4-flash",
    tools=[...],
    system_prompt=(
        "你是一个研究报告生成助手。\n\n"
        "完成报告时必须满足以下标准：\n"
        "1. 报告包含至少 3 个具体数据点\n"
        "2. 至少引用 2 个权威来源\n"
        "3. 提供 1 个反面观点\n"
        "4. 全文不少于 500 字"
    ),
    middleware=[
        RubricMiddleware(
            model=grading_llm,
            max_iterations=3,  # 最多自评 3 轮
        ),
    ],
)
```

执行流程：
```
1. Agent 生成报告
2. Agent 声明"完成"
3. RubricMiddleware 评分 → 不满足标准
4. 把评分反馈给 Agent
5. Agent 改进
6. 再次评分 → 满足 → 结束
```

## 五、注意事项

1. **Beta 阶段**：API 可能变化，生产环境慎用

2. **必须有清晰的评分标准**：`system_prompt` 中要明确写出"完成"的定义。如果标准模糊，评分模型也会跟着模糊

3. **额外的 LLM 调用成本**：每次评分都要调一次 LLM。`max_iterations=3` 意味着最坏情况多 3 次 LLM 调用

4. **`max_iterations` 太少可能不收敛**：如果评分标准太严格，3 次不够。需要平衡标准严格度与迭代次数

5. **评分模型建议用 temperature=0**：评分要稳定，不要有随机性

6. **不替代人工评估**：自动评分只能评估"是否满足形式标准"，无法评估内容质量。建议用于"形式检查"（"是否引用了来源"）+ 人工抽样

7. **可以配合 HITL**：评分通过后再走 HITL 审批，是企业级质控的常见组合

## 六、相关页面

- [[langchain/middleware/中间件概述]] — 总览
- [[langchain/middleware/SubAgentMiddleware]] — 子代理
- [[langchain/middleware/HumanInTheLoopMiddleware]] — 评分通过后人工审批
