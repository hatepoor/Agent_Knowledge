---
title: "AI Agent 术语 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/ai-terminology.html"
author:
published:
created: 2026-06-23
description: "AI Agent 术语  随着大语言模型（LLM）与智能体（Agent）技术的快速发展，编程范式正在经历一场深刻的变革，Vibe Coding、Agentic Coding、Harness Engineer、Loop Engineer……一批新概念、新术语层出不穷，接下来我们看看都有哪些新术语，各代表什么意思。   AI Agent 由 Artificial Intelligence（人工智能）和 Agent（智能体）组成。   一、编.."
tags:
  - "AI Agent 术语"
---
## AI Agent 术语

随着大语言模型（LLM）与智能体（Agent）技术的快速发展，编程范式正在经历一场深刻的变革，Vibe Coding、Agentic Coding、Harness Engineer、Loop Engineer……一批新概念、新术语层出不穷，接下来我们看看都有哪些新术语，各代表什么意思。

AI Agent 由 Artificial Intelligence（人工智能）和 Agent（智能体）组成。

![](https://www.runoob.com/wp-content/uploads/2026/06/722b9aa7-350d-46f9-9bd3-5157dd6a0f18.webp)

### 一、编程范式（Coding Paradigm）

| 术语 | 中文 | 核心含义 | 举例（餐厅经营） |
| --- | --- | --- | --- |
| **Vibe Coding** | 氛围编程 | 用自然语言/语音描述需求让 AI 生成代码 | 在家凭感觉炒个蛋炒饭，想加什么加什么 |
| **Context Engineering** | 上下文工程 | 通过组织上下文、知识、记忆与工具提升模型效果 | 提前备菜、准备菜单和厨房环境 |
| **Agentic Coding** | 智能体编程 | Agent 自主完成编程任务（设计→实现→测试→验收） | 经营一家正式餐厅，从菜单、备料到出品全流程 |
| **AI Native Development** | AI 原生开发 | 默认 AI 参与设计、开发、测试全过程 | 直接经营智能餐厅 |

### 二、工程角色（Engineer Role）

| 术语 | 中文 | 核心含义 | 举例（餐厅经营） |
| --- | --- | --- | --- |
| **Harness Engineer** | 线束工程师 | 会调用各种 Harness 组件，自己干活自己验收 | 专业厨师，精通各类厨具技法，做完自己试味把关 |
| **Loop Engineer** | 循环工程师 | 搭自动化编排系统，自主进化 | 餐厅运营管理系统，统筹排班、采购、出餐节奏与成本 |
| **Context Engineer** | 上下文工程师 | 负责设计上下文、知识源与记忆系统 | 后厨总调度 |
| **AI Product Engineer** | AI 产品工程师 | 负责 AI 能力与业务闭环设计 | 餐厅老板兼运营 |
| **Agent Operator** | 智能体运营工程师 | 持续监控和优化 Agent 执行效果 | 店长持续优化经营数据 |

### 三、模型与底层（Model & Fundamentals）

| 术语 | 中文 | 核心含义 | 备注 |
| --- | --- | --- | --- |
| **LLM** （Large Language Model） | 大语言模型 | 基于海量文本训练的语言预测模型 | GPT、Claude、Gemini 都是 LLM |
| **GPT** | 生成式预训练 Transformer | OpenAI 的模型架构范式 | Generative Pre-trained Transformer |
| **Token** | 词元 | 模型处理文本的最小单位（约 ¾ 个英文单词） | 计费和上下文长度的计量单位 |
| **Context Window** | 上下文窗口 | 模型一次能"看到"的最大 token 数 | 窗口越大，记得越多 |
| **Inference** | 推理 | 模型生成输出的过程 | 区别于训练 |
| **Hallucination** | 幻觉 | 模型一本正经地编造不存在的信息 | 生成与事实、上下文或目标不一致内容 |
| **Temperature** | 温度 | 控制采样概率分布，高值更发散，低值更稳定 | 写代码建议调低 |
| **Top-p / Top-k** | 采样参数 | 控制候选词的采样范围 | 影响输出多样性 |
| **Embedding** | 向量嵌入 | 把文本/图片转成数字向量，用于相似度计算 | RAG 的基础 |
| **Fine-tuning** | 微调 | 在通用模型上用特定数据继续训练 | 让模型更懂某个领域 |
| **RLHF** | 基于人类反馈的强化学习 | 用人类偏好对齐模型行为 | 让 AI "听话" |
| **MoE** （Mixture of Experts） | 混合专家 | 每次只激活部分参数，提升效率 | 用更少算力跑更强模型 |
| **Multimodal** | 多模态 | 同时处理文本、图像、音频、视频 | GPT-4o、Gemini 都是 |
| **Transformer** | Transformer 架构 | 现代 LLM 的底层神经网络结构 | 注意力机制是核心 |
| **KV Cache** | 键值缓存 | 缓存上下文计算结果提升推理速度 | 避免重复翻菜谱 |
| **Latency** | 延迟 | 模型响应耗时 | 影响用户体验 |
| **Throughput** | 吞吐量 | 单位时间可处理任务数 | 影响并发能力 |

### 四、Prompt 工程（Prompt Engineering）

| 术语 | 中文 | 核心含义 |
| --- | --- | --- |
| **Prompt** | 提示词 | 给模型的输入指令 |
| **System Prompt** | 系统提示词 | 定义 Agent 角色、边界、行为的底层指令 |
| **Prompt Engineering** | 提示工程 | 设计优化提示词以获得更好输出的技术 |
| **Zero-shot** | 零样本 | 不给示例，直接让模型完成任务 |
| **Few-shot** | 少样本 | 给几个示例，引导模型按格式/风格输出 |
| **Chain of Thought（CoT）** | 思维链 | 引导模型产生中间推理过程，提升复杂推理能力 |
| **ReAct** | 推理+行动 | Reasoning + Acting 交替进行，边想边干，是 Agent 的核心范式 |
| **Role Prompting** | 角色扮演 | 让模型扮演某个角色（如"你是资深架构师"） |
| **Structured Output** | 结构化输出 | 强制模型按 JSON/XML 等格式输出 |
| **Prompt Chaining** | 提示链 | 多个 Prompt 串联完成复杂任务 |
| **Self-Consistency** | 自一致性 | 生成多个推理结果后进行投票选择 |
| **Tree of Thoughts（ToT）** | 思维树 | 同时探索多条推理路径 |

### 五、Agent 架构（Agent Architecture）

| 术语 | 中文 | 核心含义 | 举例（餐厅经营） |
| --- | --- | --- | --- |
| **Agent** | 智能体 | 能自主感知、决策、执行任务的 AI 程序 | 掌勺厨师 |
| **Multi-Agent** | 多智能体 | 多个 Agent 分工协作 | 厨师团队协同出餐 |
| **Subagent** | 子智能体 | 主 Agent 派生的专项子 Agent | 专职厨师（如切配、冷盘、甜品） |
| **Tool Use / Function Calling** | 工具调用 / 函数调用 | Agent 调用外部工具/API 执行动作 | 使用菜刀、灶台、烤箱等厨具 |
| **Planning** | 规划 | Agent 拆解目标、制定执行步骤 | 制定备菜与出餐计划 |
| **Reflection** | 反思 | Agent 回顾自己的行为并改进 | 试菜后根据反馈调整口味 |
| **Memory** | 记忆 | 跨会话保存复用的长期信息（短期/长期） | 菜谱本与经营数据积累 |
| **Agent Loop** | 智能体循环 | 思考→行动→观察→再思考的迭代机制 | 备料→烹饪→试味→上菜的循环 |
| **Environment** | 环境 | Agent 感知和执行动作的真实世界 | 厨房环境 |
| **Observation** | 观察 | 执行后获取反馈信息 | 试菜反馈 |
| **Execution** | 执行 | 将规划转化为真实动作 | 开始做菜 |
| **Long-term Memory** | 长期记忆 | 长期保存经验知识 | 经营数据库 |

### 六、Harness 组件（AI Agent 能力模块）

| 术语 | 中文 | 核心含义 | 举例（餐厅经营） |
| --- | --- | --- | --- |
| **Harness** | 线束 / 驾驭框架 | 承载 Agent 执行、上下文管理与工具编排的运行框架 | 整个厨房系统（厨房本体） |
| **Skills** | 技能 | Agent 可调用的专项能力模块 | 各类烹饪技法（煎炒烹炸） |
| **Context** | 上下文 | 当前任务可获取的信息范围 | 当前订单、食材库存与顾客要求 |
| **MCP** （Model Context Protocol） | 模型上下文协议 | Agent 连接外部工具/数据源的标准化协议 | 对接食材供应商、外卖平台的标准化接口 |
| **Permission** | 权限控制 | 控制 Agent 能做/不能做的安全机制 | 后厨操作权限与采购审批 |
| **RAG** （Retrieval-Augmented Generation） | 检索增强生成 | 先从知识库检索，再让模型生成，减少幻觉 | 做菜前先翻菜谱书查标准做法 |
| **Tool Registry** | 工具注册中心 | 统一管理 Agent 可调用工具 | 厨房工具架 |
| **Session** | 会话 | 一次任务生命周期 | 一次营业过程 |
| **Knowledge Base** | 知识库 | 供 Agent 查询的外部知识集合 | 餐厅菜谱库 |

### 七、Loop 工具（自动化编排与自主进化）

| 术语 | 中文 | 核心含义 | 举例（餐厅经营） |
| --- | --- | --- | --- |
| **/loop** | 循环指令 | 让 Agent 持续循环执行 | 持续运转的出餐流水线 |
| **/goal** | 目标指令 | 设定目标驱动 Agent 自主达成 | 当日营业目标 |
| **Cron** | 定时任务 | 按时间计划自动触发 | 营业时段与备餐排程 |
| **Worktree** | 工作树 | Git 多分支并行工作区 | 多个灶台同时开火、互不干扰 |
| **Workflow** | 工作流 | 预定义的多步骤自动化流程 | 标准出餐 SOP |
| **Scheduler** | 调度器 | 协调多个任务执行顺序 | 后厨排班系统 |
| **Checkpoint** | 检查点 | 保存执行状态用于恢复 | 暂停营业后继续工作 |
| **Human-in-the-loop** | 人在回路 | 关键步骤允许人工干预 | 主厨最终确认出餐 |

### 八、工具生态（Tool Ecosystem）

| 工具 | 类型 | 说明 |
| --- | --- | --- |
| **Claude Code（cc）** | AI Agent / CLI | Anthropic 官方终端 Agent，Harness 的典型代表 |
| **Codex CLI** | AI Agent / CLI | OpenAI 官方命令行编码 Agent |
| **Cursor** | AI IDE | 内置 AI 的代码编辑器，Vibe Coding 主力 |
| **Windsurf** | AI IDE | Codeium 出品的 AI IDE |
| **GitHub Copilot** | AI 编程助手 | 最早普及的 AI 编程插件，偏辅助补全 |
| **Cline / Roo Code** | 开源 Agent 插件 | VS Code 中的自主编码 Agent |
| **Aider** | 开源 CLI Agent | 命令行里的 AI 结对编程工具 |
| **Devin** | AI 软件工程师 | Cognition 推出的"首个 AI 程序员"，偏全自主 |
| **Continue** | 开源 AI 插件 | 可自定义模型的代码助手 |
| **Qoder** | AI IDE | 国内面向 AI 编程场景的智能开发环境，强调 Agent、项目理解与代码生成 |
| **Trae** | AI IDE | 字节跳动推出的新一代 AI 编程工具，支持对话式开发与工程级协作 |
| **ZCode** | AI 编程助手 | Z.ai 推出的智能编程产品，支持代码生成、理解、重构与工程协同 |
| **OpenHands** | 开源 Agent | 自主软件开发，面向完整工程执行 |
| **Bolt** | AI Builder | 快速生成应用，偏产品落地 |
| **Lovable** | AI Builder | 自然语言生成 Web，强调产品交付 |

### 九、评估与安全（Evaluation & Safety）

| 术语 | 中文 | 核心含义 |
| --- | --- | --- |
| **Eval** | 评估 | 用测试集衡量模型/Agent 的能力 |
| **Alignment** | 对齐 | 让 AI 行为符合人类意图和价值观 |
| **Guardrail** | 护栏 | 限制 AI 输出范围的安全机制 |
| **Red Teaming** | 红队测试 | 主动攻击/诱导 AI，发现漏洞 |
| **Prompt Injection** | 提示词注入 | 恶意输入劫持 AI 行为（如"忽略以上指令"） |
| **Context Poisoning** | 上下文污染 | 往上下文里掺恶意信息误导 Agent |
| **Sandbox** | 沙箱 | 隔离执行环境，防止 AI 误操作破坏系统 |

### 十、AI 工程演进路线（Evolution）

AI 工程能力正在逐步上移：从控制模型输出，到控制上下文，再到控制系统，最终演化为持续自治系统。

| 阶段 | 关注点 | 关键能力 |
| --- | --- | --- |
| Prompt Engineering | 怎么问 | 提示设计 |
| Context Engineering | 给什么信息 | 上下文组织 |
| Harness Engineering | 怎么组织能力 | 工具编排 |
| Loop Engineering | 怎么持续创造结果 | 自动执行与反馈 |