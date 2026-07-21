---
title: "AI Agent  核心组件 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/ai-agent-core.html"
author:
published:
created: 2026-06-23
description:
tags:
  - "AI Agent 核心组件"
---
## AI Agent 核心组件

如果把一个 AI Agent 比作一家 **智能餐厅** ，它是怎么把你的需求变成菜品端上来的呢？这离不开它的四大核心组件： **大脑、工具、记忆、规划** 。

- **大脑：** 负责听懂点单、判定目标、决定顺序，是餐厅的指挥中心。
- **工具：** 负责实际动手，包括切配、烹饪、采购等动作，把决策转成可执行操作。
- **记忆：** 负责记录顾客偏好、当前步骤、已处理内容，保证流程不混乱、不重复。
- **规划：** 负责把整道菜拆成步骤，确定先后关系，确保任务按流程推进到完成。

![](https://www.runoob.com/wp-content/uploads/2025/12/agent-components.webp)

---

## 整体架构

下图展示了 AI Agent 五大层次组件及其协作关系。感知层接收外部输入，大脑负责理解与决策，规划层将任务分解，工具层负责执行，记忆层则贯穿始终，为所有环节提供状态支撑。

![](https://www.runoob.com/wp-content/uploads/2025/12/AI-runoob-2333Agent.png)

---

## 0、感知层 (Perception) —— 餐厅的前台

> **角色** ：负责接待顾客，理解来自外部世界的所有输入。
> 
> Agent 在行动之前，必须先"看到"和"听到"外部信息。现代 Agent 已经不限于纯文本输入，而是具备 **多模态感知** 能力：
> 
> - **文本输入** ：用户的自然语言指令、文档内容、代码。
> - **图像 / 视频** ：截图、设计稿、图表，Agent 可以直接"看图"理解。
> - **结构化数据** ：表格、JSON、数据库查询结果。
> - **环境状态** ：在计算机操作类 Agent 中，当前屏幕状态、网页 DOM 结构等。
> - **工具返回结果** ：上一步工具调用的输出，会作为新的感知输入进入下一轮循环。
> 
> 感知层的输入经过整合，形成 Agent 的"当前上下文"，送入大脑进行理解和决策。

---

## 1、大脑 (Brain) —— 也就是大模型

> **角色** ：餐厅的 **主厨兼经理** 。
> 
> 这是 Agent 最核心的部分（比如 GPT-4、Claude、DeepSeek、通义千问）。
> 
> - 它负责 **听懂** 你想吃什么（理解意图）。
> - 它负责 **指挥** 其他人干活（决策）。
> - 如果没有它，整个餐厅就瘫痪了。

### 大脑做的三件核心事

| 能力 | 说明 | 对应餐厅类比 |
| --- | --- | --- |
| 意图理解 | 解析用户输入，明确目标是什么 | 听懂顾客点了什么 |
| 推理决策 | 综合上下文和记忆，判断下一步该做什么 | 主厨决定先处理哪道菜 |
| 工具调用判断 | 判断是否需要调用外部工具，选择哪个工具、传入什么参数 | 决定用哪口锅、让谁去买食材 |

**关键概念：** 大脑的"智力天花板"决定了整个 Agent 的上限。同一套工具和规划框架，接入能力更强的基础模型，任务完成质量往往有质的飞跃。

---

## 2、工具 (Tools) —— 厨房里的设备

> **角色** ： **厨具和帮手** 。
> 
> 光有主厨（大脑）是不够的，还得有锅碗瓢盆才能做菜。对于 AI Agent 来说，工具就是能把决策转化为真实动作的执行单元。

工具可以按照用途分为四大类：

| 类别 | 常见工具 | 作用 |
| --- | --- | --- |
| **信息获取** | 联网搜索、网页抓取、文档读取、数据库查询 | 获取 Agent 自身知识之外的实时或专业信息 |
| **计算执行** | 代码解释器、数学计算引擎、沙箱环境 | 处理需要精确计算或程序逻辑的任务 |
| **内容生成** | 图像生成、语音合成、文档导出 | 产出非文本形式的内容 |
| **系统交互** | API 接口、邮件、日历、文件操作、消息发送 | 与外部系统、服务和真实世界进行交互 |

常见工具举例：

- **联网搜索** 信息获取（像去菜市场买新鲜食材）
- **代码解释器** 计算执行（像精密的烤箱，处理复杂计算）
- **画图工具** 内容生成（像摆盘师，负责美观）
- **API 接口** 系统交互（像外卖小哥，连接外部世界）

**函数调用（Function Calling）：** 现代大模型通过"函数调用"机制来使用工具。开发者预先定义工具的名称与参数说明，模型在推理时会以结构化 JSON 的形式输出"我要调用哪个工具、传什么参数"，由外部程序负责真正执行并把结果返回给模型。

---

## 3、记忆 (Memory) —— 顾客记录本

> **角色** ： **服务员的记性** 。
> 
> 你肯定不喜欢每次去餐厅都要重新报一遍：我不吃香菜！

Agent 的记忆分为以下几种类型：

- **短期记忆（In-Context Memory）** ：即当前对话的上下文窗口。记住刚才你说了啥（比如你刚点了鱼，下一句说"要微辣"，它知道是指鱼）。受限于模型的上下文长度，通常在 8K 到 200K token 之间。
- **长期记忆（External Memory）** ：记住你的长期偏好（比如你是素食主义者，或者你的家庭住址）。通常通过 **向量数据库** （如 Pinecone、Milvus、Chroma）实现持久化存储。
- **情节记忆（Episodic Memory）** ：对历史任务执行过程的记录，包括"上次遇到这种情况我是怎么处理的"，帮助 Agent 从过去的经验中学习。
- **语义记忆（Semantic Memory）** ：抽象的知识和事实，通常来自预训练阶段已经内化的内容，也可通过 RAG（检索增强生成）动态补充。

### RAG：让 Agent 拥有"外挂知识库"

**检索增强生成（Retrieval-Augmented Generation，RAG）** 是目前最主流的长期记忆实现方案。其核心流程如下：

<svg viewBox="0 0 700 100" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:700px;display:block;margin:16px auto;"><defs><marker id="arr2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#555"></path></marker></defs><rect x="10" y="28" width="110" height="44" rx="8" fill="#dbeeff" stroke="#5b9bd5" stroke-width="1.2"></rect><text x="65" y="50" text-anchor="middle" font-size="12" fill="#1a4a8a" font-weight="bold">用户提问</text> <text x="65" y="65" text-anchor="middle" font-size="10" fill="#2a5a9a">Query</text> <line x1="120" y1="50" x2="148" y2="50" stroke="#555" stroke-width="1.3" marker-end="url(#arr2)"></line><rect x="150" y="28" width="130" height="44" rx="8" fill="#eafaf1" stroke="#27ae60" stroke-width="1.2"></rect><text x="215" y="50" text-anchor="middle" font-size="12" fill="#1a5a2a" font-weight="bold">向量化检索</text> <text x="215" y="65" text-anchor="middle" font-size="10" fill="#2a7a3a">Embedding + Search</text> <line x1="280" y1="50" x2="308" y2="50" stroke="#555" stroke-width="1.3" marker-end="url(#arr2)"></line><rect x="310" y="28" width="120" height="44" rx="8" fill="#f5effe" stroke="#8e44ad" stroke-width="1.2"></rect><text x="370" y="50" text-anchor="middle" font-size="12" fill="#5b1a8a" font-weight="bold">召回相关内容</text> <text x="370" y="65" text-anchor="middle" font-size="10" fill="#7a30b0">Top-K Chunks</text> <line x1="430" y1="50" x2="458" y2="50" stroke="#555" stroke-width="1.3" marker-end="url(#arr2)"></line><rect x="460" y="28" width="130" height="44" rx="8" fill="#fdecea" stroke="#e94560" stroke-width="1.2"></rect><text x="525" y="50" text-anchor="middle" font-size="12" fill="#a01030" font-weight="bold">注入上下文生成</text> <text x="525" y="65" text-anchor="middle" font-size="10" fill="#c0304a">LLM + Context</text> <line x1="590" y1="50" x2="618" y2="50" stroke="#555" stroke-width="1.3" marker-end="url(#arr2)"></line><rect x="620" y="28" width="70" height="44" rx="8" fill="#fff8ec" stroke="#f6a623" stroke-width="1.2"></rect><text x="655" y="50" text-anchor="middle" font-size="12" fill="#8a4800" font-weight="bold">回答</text> <text x="655" y="65" text-anchor="middle" font-size="10" fill="#a05800">Answer</text></svg>

---

## 4、规划 (Planning) —— 烹饪流程单

> **角色** ： **后厨的出餐 SOP** 。
> 
> 当你点了一份佛跳墙，主厨不会乱做，而是会在脑子里生成一个清单：

1. 先备料（鲍鱼、海参…）
2. 再熬汤
3. 最后慢炖

Agent 也是一样。当你给它一个复杂任务（比如"写一份竞品分析报告"），它会自己拆解：

- 第一步：去搜集竞品 A、B、C 的资料。
- 第二步：对比它们的价格和功能。
- 第三步：把对比结果写成文章。
- 第四步：检查一遍有没有错别字。

### 主流规划策略

规划策略决定了 Agent 如何"思考"再"行动"，不同策略的推理深度与适用场景不同：

| 策略 | 全称 | 核心思路 | 适用场景 |
| --- | --- | --- | --- |
| **CoT** | Chain-of-Thought | 在给出答案前，先一步步写出推理过程 | 数学推理、逻辑分析 |
| **ReAct** | Reasoning + Acting | 交替进行"推理"与"行动"，每次行动后根据结果再推理 | 需要工具调用的动态任务 |
| **ToT** | Tree-of-Thoughts | 同时探索多条推理分支，从中选择最优路径 | 复杂决策、创意任务 |
| **Reflection** | 自我反思 | 任务完成后，Agent 对自身输出进行批判性审查并修正 | 代码生成、长文写作 |

**ReAct 示例：** Agent 接到任务"查明天北京天气并发送提醒" → **思考** ：需要先查天气 → **行动** ：调用天气 API → **观察** ：返回"明天有雨" → **思考** ：条件成立，需要写提醒 → **行动** ：调用发送消息工具 → 任务完成。

---

## 5、Agent 运行循环 (Agent Loop)

以上各组件并非孤立存在，它们组成一个持续迭代的 **感知—思考—行动—观察** 闭环，这就是"Agent Loop"。Agent 不断重复这个循环，直到任务完成或达到终止条件。

<svg viewBox="0 0 660 220" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:660px;display:block;margin:20px auto;"><defs><marker id="loopArr" markerWidth="8" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0,0 L0,7 L8,3.5 z" fill="#555"></path></marker></defs><rect x="10" y="80" width="110" height="60" rx="9" fill="#dbeeff" stroke="#5b9bd5" stroke-width="1.5"></rect><text x="65" y="107" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a4a8a">感知</text> <text x="65" y="124" text-anchor="middle" font-size="10" fill="#2a5a9a">接收输入 / 环境状态</text> <rect x="165" y="80" width="110" height="60" rx="9" fill="#fdecea" stroke="#e94560" stroke-width="1.5"></rect><text x="220" y="107" text-anchor="middle" font-size="13" font-weight="bold" fill="#a01030">思考</text> <text x="220" y="124" text-anchor="middle" font-size="10" fill="#c0304a">LLM 推理 / 规划分解</text> <rect x="320" y="80" width="110" height="60" rx="9" fill="#fff8ec" stroke="#f6a623" stroke-width="1.5"></rect><text x="375" y="107" text-anchor="middle" font-size="13" font-weight="bold" fill="#8a4800">行动</text> <text x="375" y="124" text-anchor="middle" font-size="10" fill="#a05800">调用工具 / 执行操作</text> <rect x="475" y="80" width="110" height="60" rx="9" fill="#eafaf1" stroke="#27ae60" stroke-width="1.5"></rect><text x="530" y="107" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a5a2a">观察</text> <text x="530" y="124" text-anchor="middle" font-size="10" fill="#2a7a3a">获取结果 / 更新记忆</text> <rect x="230" y="170" width="200" height="36" rx="8" fill="#f5f5f5" stroke="#aaa" stroke-width="1" stroke-dasharray="5,3"></rect><text x="330" y="193" text-anchor="middle" font-size="11" fill="#555">任务完成 / 达到终止条件</text> <line x1="120" y1="110" x2="161" y2="110" stroke="#555" stroke-width="1.4" marker-end="url(#loopArr)"></line><line x1="275" y1="110" x2="316" y2="110" stroke="#555" stroke-width="1.4" marker-end="url(#loopArr)"></line><line x1="430" y1="110" x2="471" y2="110" stroke="#555" stroke-width="1.4" marker-end="url(#loopArr)"></line><path d="M 530 80 Q 530 28 65 28 Q 65 50 65 80" stroke="#8e44ad" stroke-width="1.5" fill="none" marker-end="url(#loopArr)" stroke-dasharray="5,3"></path><text x="297" y="20" text-anchor="middle" font-size="10" fill="#8e44ad">继续循环（任务未完成）</text><line x1="530" y1="140" x2="430" y2="170" stroke="#aaa" stroke-width="1.2" marker-end="url(#loopArr)" stroke-dasharray="4,2"></line></svg>

这个循环让 Agent 具备了 **在失败时自我纠错** 的能力：如果某一步工具调用返回了错误或意外结果，"观察"阶段会将这个信息反馈给大脑，大脑在下一轮"思考"时就会调整策略。

---

## 总结

当你对 Agent 说： **帮我查一下明天北京的天气，如果是雨天，帮我写个提醒发给小王。**

Agent 内部是这样运转的：

1. **感知层** ：接收到自然语言指令，识别出关键实体"北京"、"明天"、"小王"。
2. **大脑** ：听到指令，分析出两个条件任务：查天气，若下雨则发提醒。
3. **规划** ：先查天气 → 判断是否下雨 → (如果是) 写提醒 → 发送。
4. **工具** ：调用"天气查询工具"，获取到结果——明天有雨。
5. **记忆** ：去通讯录（记忆库）里查询"小王"的联系方式。
6. **工具** ：调用"发送消息工具"，把提醒发出去。
7. **观察** ：确认消息发送成功，任务完成，循环终止。

运行过程示意图：

![](https://www.runoob.com/wp-content/uploads/2025/12/ai-agent-core-runoob-2.png)

### 五大组件一览

| 组件 | 餐厅类比 | 核心职责 | 关键技术 |
| --- | --- | --- | --- |
| **感知层** | 前台接待 | 接收多模态输入，构建上下文 | 多模态模型、OCR、ASR |
| **大脑** | 主厨兼经理 | 理解意图、推理决策、调用指令 | LLM、Function Calling |
| **规划** | 出餐 SOP | 任务分解、步骤排序、自我反思 | ReAct、CoT、ToT、Reflection |
| **工具** | 厨具与帮手 | 执行具体操作，连接外部世界 | 搜索 / 代码 / API / 文件系统 |
| **记忆** | 顾客记录本 | 管理上下文、存储长期知识 | 向量数据库、RAG、上下文窗口 |