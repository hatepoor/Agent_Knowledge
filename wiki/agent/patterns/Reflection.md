---
tags: [agent, patterns, reflection, self-critique, 自我反思]
type: concept
created: 2026-09-03
updated: 2026-09-03
---

# Reflection（反思 / 评估-优化）

## 定义
智能体在产出结果后，先由一个 **Critic（评审者/评估者）** 评估质量；不达标则带着反馈**重新生成**，直到满足标准或达到最大迭代次数。对应 Anthropic《Building Effective Agents》中的 **evaluator-optimizer** 工作流 [$TRAE_REF](https://www.anthropic.com/engineering/building-effective-agents)。

```
生成(Generator) → 评估(Evaluator/Critic)
     ▲                │ 未通过
     └──────── 反馈 ◄──┘
     （迭代到合格或达到上限）
```

## 两种评审方式
1. **自我反思**：同一个模型、同一 prompt，自己审自己的输出。
2. **独立 Critic**：用另一个模型（或同模型不同 prompt/更强角色）专门评估——在 Anthropic 建议中，**独立评估器通常优于自我评估**，质量要求高时应使用。

## 与其他模式的区别
- **vs [[PlanAndExecute]]**：Reflection 把关「输出质量」；Plan-and-Execute 把关「行动计划」。两者常组合：计划生成后由 Critic 审查再执行。
- **vs [[ReAct]]**：ReAct 的循环由「外部工具反馈」推动；Reflection 的循环由「自我/独立评估」推动。
- **vs [[Reflexion]]**：Reflection 每轮从零重做、不保存经验；Reflexion 把失败/反思**写进记忆**供后续复用。
- **vs [[ToT]]**：Reflection 沿单条解反复精修；ToT 分叉多条并回溯比较。

## 关键设计：评估标准与停止条件
- **必须给出可判定的评估指标**（打分、检查清单、单元测试是否通过）——否则反思只是空转白耗 token。
- **必须设置最大迭代上限**，防止死循环。
- 评估结果要能转成**可执行的改进指令**（具体错在哪、怎么改）。

## 优点 / 局限
| 优点 | 局限 |
|------|------|
| 显著提升输出质量（尤其代码、文案） | 多轮迭代增加延迟与成本 |
| 对「一次成型难」的任务稳定收敛 | 依赖模型的自我评估能力，若无法识别错误只是空转 |
| 流程简单，可叠加到任意范式上 | 独立 Critic 增加一次模型调用 |
| 适用于生成-检查含明确标准 | 开放、无标准任务难定「合格」 |

## 最适合的场景
- 代码生成（写测试→跑→失败→修复→再跑）
- 文案润色、论文/报告打磨
- 生成物有**客观或较明确评价标准**的精修任务

## 相关导航
- [[Agent模式总览]] — 模式全集与选型
- [[Reflexion]] — 带记忆的进阶版反思
- [[PlanAndExecute]] — 常见的组合搭配
- [[agent/harness/Agent_六种架构]] §4 反思与自我修正

## 来源与权威参考
- Anthropic《Building Effective Agents》：evaluator-optimizer 工作流，及独立评估器优于自我评估的建议。
- Shinn et al.《Reflexion》(2023) 中对 Reflection 与 Reflexion 的界定。