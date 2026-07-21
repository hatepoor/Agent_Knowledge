---
title: "AI Agent 工作原理 | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/ai-agent-working-principle.html"
author:
published:
created: 2026-06-23
description: "AI Agent 工作原理 在开始深入了解 AI Agent 之前，我们先来回答一个根本问题：既然有 LLM，为什么还需要 Agent？  想象一下，你有一位非常博学的朋友——他读过海量的书，能回答几乎所有问题，但他在 2024 年不再接触新信息，也没有手机、不能上网、不能帮你订外卖、不能查今天的天气。这就是大型语言模型（LLM）——知识丰富，但知识截止于训练数据，无法主动获取实时信息，也无法执行具体的操作。  而 AI Agent 就.."
tags:
  - "AI Agent 工作原理"
---
## AI Agent 工作原理

在开始深入了解 AI Agent 之前，我们先来回答一个根本问题： **既然有 LLM，为什么还需要 Agent？**

想象一下，你有一位非常博学的朋友——他读过海量的书，能回答几乎所有问题，但他在 2024 年不再接触新信息，也没有手机、不能上网、不能帮你订外卖、不能查今天的天气。这就是大型语言模型（LLM）——知识丰富，但 **知识截止于训练数据** ，无法主动获取实时信息，也无法执行具体的操作。

而 **AI Agent** 就像是给这位朋友配备了：一部可以上网的手机、一台计算器、一个日历……让他不仅能思考，还能 **真正动手做事** 。AI Agent 通过将 LLM 与工具和记忆结合，突破了这一限制，实现了 **思考与行动的统一** 。

---

## AI Agent 的三大核心组成

一个典型的 AI Agent 由三个关键部分协同工作，我们继续用上面的比喻来理解：

**1、大脑 (The Brain) - 大型语言模型 (LLM)**

- **角色** ：Agent 的决策中心和推理引擎。
- **功能** ：理解用户输入的 **目标** 和 **上下文** ，分析当前状况，然后决定下一步该做什么——是直接回答问题，还是调用某个工具？它负责规划和分解复杂任务。
- **比喻** ：就像公司的 **CEO 或指挥官** ，负责战略思考、任务规划和下达指令。它知道要做什么，但需要借助工具来真正做到。

**2、工具 (Tools) - 可执行的动作**

- **角色** ：Agent 的手和脚，是其能力的延伸。
- **功能** ：一个个具体的函数或 API，让 Agent 能够与外部世界互动。例如： `search_web` （搜索网页）、 `execute_python_code` （运行代码）、 `read_file` （读取文件）、 `send_email` （发送邮件）等。
- **比喻** ：就像员工桌上的 **各种办公软件和设备** ，如 Excel、浏览器、电话、打印机。CEO（大脑）发出指令，员工（工具）负责执行。
- **关键理解** ：没有工具，LLM 只能说，有了工具，Agent 才能真正做。

**3、记忆 (Memory) - 对话与经验的存储**

- **角色** ：记录工作过程，保证任务的连贯性。
- **功能** ：
	- **短期记忆** ：保存当前对话的历史，让 Agent 记得之前说过什么、做过什么。就好比你跟朋友聊天，不需要每句话都重新自我介绍。
		- **长期记忆** ：可以存储更持久的信息（例如用户偏好、历史任务结果），供未来任务参考。就像你有一本专属于你的用户档案。
- **比喻** ：就像员工的 **工作笔记和项目档案** ，避免重复劳动，让每次工作都能基于之前的经验继续推进。

![](https://www.runoob.com/wp-content/uploads/2025/12/ai-agent-core-runoob-3-scaled.png)

💡 **一句话总结** ：大脑负责思考，工具负责行动，记忆保证连贯，三者缺一不可。

---

## AI Agent 的类型

根据不同的设计目标和复杂度，AI Agent 可以分为多种类型：

| Agent 类型 | 特点 | 典型应用 |
| --- | --- | --- |
| **反应式 Agent (Reactive Agent)** | 基于当前感知做出即时响应，不维护内部状态。就像一个只看眼前、不记过去的客服。 | 简单问答、游戏 AI |
| **目标导向 Agent (Goal-based Agent)** | 围绕特定目标规划行动，能评估目标是否达成。就像有 KPI 的员工，知道自己做什么是为了完成什么目标。 | 任务助手、自动化工作流 |
| **实用型 Agent (Utility-based Agent)** | 通过"效用函数"给多种可能的行动打分，选择最优方案。就像一个会权衡利弊、追求最优解的决策者。 | 资源优化、路径规划 |
| **学习型 Agent (Learning Agent)** | 能从经验中学习，不断优化决策策略。越用越聪明，像一个勤于复盘的员工。 | 推荐系统、个性化助手 |
| **多智能体系统 (Multi-Agent)** | 多个 Agent 协作分工，各司其职。就像一个分工明确的团队，每人负责一块。 | 复杂任务分解、团队协作 |

初学者只需先掌握前两种，它们是最常见的基础形态。

---

## 工作流程：ReAct 循环

了解了 Agent 的组成，下一个问题是：它是 **怎么运转起来** 的？

AI Agent 通常遵循一个名为 **ReAct = Reasoning（推理）+ Acting（行动）** 的经典思维范式。顾名思义，就是 **先想、再做、再想、再做** ……这个循环不断重复，直到任务完成。

你可以把它想象成一个认真负责的新员工接到一个任务时的工作方式： *"我先想想该怎么做 → 去查一下资料 → 看看查到了什么 → 再想下一步 → 继续行动……"* ，而不是一拍脑袋就乱做。

<svg width="100%" viewBox="0 0 800 320" xmlns="http://www.w3.org/2000/svg" style="background:#fafafa;border-radius:8px;margin:20px 0;"><defs><marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#3498db"></polygon></marker></defs><ellipse cx="400" cy="160" rx="340" ry="130" fill="none" stroke="#e0e0e0" stroke-width="2" stroke-dasharray="8,4"></ellipse><circle cx="400" cy="160" r="40" fill="#2c3e50" filter="url(#shadow)"></circle><text x="400" y="155" text-anchor="middle" fill="white" font-size="11" font-weight="bold">用户</text> <text x="400" y="170" text-anchor="middle" fill="white" font-size="10">User</text> <circle cx="90" cy="130" r="45" fill="#3498db" filter="url(#shadow)"></circle><text x="90" y="125" text-anchor="middle" fill="white" font-size="13" font-weight="bold">思考</text> <text x="90" y="142" text-anchor="middle" fill="white" font-size="10">Reason</text> <text x="90" y="157" text-anchor="middle" fill="#aed6f1" font-size="9">LLM 推理</text> <circle cx="280" cy="55" r="45" fill="#27ae60" filter="url(#shadow)"></circle><text x="280" y="50" text-anchor="middle" fill="white" font-size="13" font-weight="bold">行动</text> <text x="280" y="67" text-anchor="middle" fill="white" font-size="10">Action</text> <text x="280" y="82" text-anchor="middle" fill="#a9dfbf" font-size="9">调用工具</text> <circle cx="520" cy="55" r="45" fill="#e67e22" filter="url(#shadow)"></circle><text x="520" y="50" text-anchor="middle" fill="white" font-size="13" font-weight="bold">观察</text> <text x="520" y="67" text-anchor="middle" fill="white" font-size="10">Observe</text> <text x="520" y="82" text-anchor="middle" fill="#f5cba7" font-size="9">获取结果</text> <circle cx="710" cy="130" r="45" fill="#9b59b6" filter="url(#shadow)"></circle><text x="710" y="125" text-anchor="middle" fill="white" font-size="13" font-weight="bold">记忆</text> <text x="710" y="142" text-anchor="middle" fill="white" font-size="10">Memory</text> <text x="710" y="157" text-anchor="middle" fill="#d2b4de" font-size="9">存储上下文</text> <path d="M 135 130 Q 180 90 235 55" stroke="#3498db" stroke-width="2" fill="none" marker-end="url(#arrowhead)"></path><path d="M 325 55 Q 380 40 475 55" stroke="#27ae60" stroke-width="2" fill="none" marker-end="url(#arrowhead)"></path><path d="M 565 55 Q 620 90 665 130" stroke="#e67e22" stroke-width="2" fill="none" marker-end="url(#arrowhead)"></path><path d="M 710 175 L 710 240 Q 600 290 400 290 Q 200 290 135 200 Q 100 170 90 175" stroke="#9b59b6" stroke-width="2" fill="none" marker-end="url(#arrowhead)"></path><text x="170" y="95" fill="#555" font-size="10">1. 生成计划</text> <text x="380" y="40" fill="#555" font-size="10">2. 执行动作</text> <text x="600" y="95" fill="#555" font-size="10">3. 接收反馈</text> <text x="400" y="305" fill="#555" font-size="10" text-anchor="middle">4. 更新记忆，继续推理</text></svg>

### 用一个完整例子来跟踪这个循环

用户向 Agent 发出指令： **"帮我在北京找一家评分高于 4.5 的意大利餐厅，并告诉我它的地址和招牌菜。"**

下面我们一步一步跟着 Agent 的"思路"走：

**第 1 步：思考 (Think/Reason)**

- **大脑分析目标** ：Agent 的大脑（LLM）拿到这个任务，首先在脑子里梳理： *"这是一个信息查询任务，需要两类信息：① 找到符合条件的餐厅；② 获取地址和招牌菜。我目前什么都不知道，应该先去搜索。"*
- **生成行动指令** ：大脑决定： *"调用 `search_web` 工具，关键词设为'北京意大利餐厅评分 4.5 以上'。"*

💡 **注意** ：这一步 Agent 还没有"做任何事"，它只是在"想"。LLM 的价值就体现在这里——能够理解意图、拆解任务、制定计划。

**第 2 步：行动 (Act)**

- **调用工具** ：Agent 根据上一步的决策，实际调用 `search_web` 工具，并传入关键词。
- **工具执行** ：搜索工具在互联网上执行搜索，返回一批原始的搜索结果（网页标题、摘要、链接等）。

💡 **注意** ：工具本身不会"思考"，它只是忠实执行。思考是大脑的工作，执行是工具的工作，分工明确。

**第 3 步：观察 (Observe)**

- **接收反馈** ：Agent 获取到工具返回的搜索结果，并将结果存入 **记忆** 中，供下一轮推理使用。
- 假设结果包含了若干家餐厅：Bottega 意库（评分 4.7）、Da Vittorio、Le Marche 等。

**第 4 步：再思考 (Think Again)**

- **大脑再分析** ：看到搜索结果后，大脑继续推理： *"好，我找到了几家候选餐厅。但用户要的是地址和招牌菜，这些在搜索摘要里没有。我需要进一步查询 Bottega 意库的详细信息。"*
- **生成新指令** ： *"调用 `get_restaurant_details` 工具，查询'Bottega 意库'的详细信息。"*

**第 5 步：再行动 → 再观察（循环继续）**

- 这个 思考 → 行动 → 观察 的循环会不断重复，每一轮都让 Agent 离目标更近一步，直到大脑判断"我已经有了足够的信息来回答用户"。

**第 6 步：输出最终答案 (Final Answer)**

- 当大脑确认任务完成，它将所有收集到的信息（存储在记忆中）整合起来，生成一个结构化、人性化的回复：
- *"找到一家符合要求的餐厅： **Bottega 意库** （评分 4.7）。地址：北京市朝阳区三里屯路 XX 号。招牌菜：黑松露披萨、手工提拉米苏。"*

这个 **思考 → 行动 → 观察 → 再思考…** 的循环，就是 AI Agent 自主完成复杂任务的核心动力机制。它不是一次性给出答案，而是像人一样 **逐步推进、边做边想** 。

---

## Python 中实现 AI Agent

理论讲完了，现在来看代码。一个 AI Agent 系统通常由几个核心模块协同工作，理解这个架构有助于我们明白它是如何思考和行动的。

![](https://www.runoob.com/wp-content/uploads/2026/02/68040279-910a-4bf9-8f0b-8b7bd88138c2.webp)

我们把每个模块拆开来，用简单的 Python 代码演示它的职责。初学者不必深究每一行代码，重点是理解 **每个模块在做什么** 。

### 1\. 感知模块 —— Agent 的"眼睛和耳朵"

感知模块负责从 **环境** 中获取信息，即"接收到了什么输入"。环境可以是：

- **数字世界** ：一段文本、一个网页、数据库中的记录、API 返回的数据。
- **物理世界** （通过硬件）：摄像头图像、麦克风音频、传感器数据。

对于最常见的文本型 Agent，感知就是"接收用户输入的一句话"。

## 实例

\# 感知模块：负责"接收"外部信息  
def perceive\_from\_environment():  
"""  
从环境中感知信息。  
在此示例中，环境就是命令行里用户输入的文字。  
"""  
user\_input = input("请输入您的指令或问题：")  
print(f"\[感知模块\] 接收到信息：'{user\_input}'")  
return user\_input # 将感知到的内容传递给下一个模块（决策模块）  
  
\# 获取感知信息  
current\_observation = perceive\_from\_environment()

### 2\. 决策模块（大脑）—— Agent 的"指挥官"

这是 Agent 的核心，通常由一个 **AI 模型（如大语言模型 LLM）** 驱动。它拿到感知模块传来的信息，负责做三件事：

- **理解** ：这条信息是什么意思？用户想要什么？
- **推理** ：当前情况下，我该怎么做？
- **规划** ：下一步（或接下来几步）应该调用哪个工具、做什么操作？

下面的代码用最简单的"关键词匹配"来模拟决策过程。真实的 Agent 里，这一步由 LLM 完成，能处理复杂得多的语义理解。

## 实例

\# 决策模块：负责"思考"和"规划"  
def make\_decision(observation):  
"""  
根据感知信息做出简单决策。  
这里用关键词匹配来模拟，实际中由 LLM 完成更复杂的推理。  
"""  
print(f"\[决策模块\] 正在分析信息：'{observation}'")  
  
\# 根据用户输入中的关键词，判断该调用哪个工具  
if "天气" in observation:  
decision = "调用天气查询工具"  
elif "计算" in observation:  
decision = "调用计算器工具"  
elif "结束" in observation:  
decision = "执行终止动作"  
else:  
decision = "进行通用对话回应" # 没有匹配到工具，就直接对话  
  
print(f"\[决策模块\] 决策结果：{decision}")  
return decision # 将决策结果传递给行动模块  
  
\# 基于感知做出决策  
current\_decision = make\_decision(current\_observation)

### 3\. 行动模块 —— Agent 的"执行者"

决策模块输出的是"想法"（要做什么），行动模块则负责将想法变成"现实"（真的去做）。它执行具体的操作，从而影响外部环境，常见的行动包括：

- **数字行动** ：在屏幕上输出答案、调用某个函数、发起 API 请求、写入文件。
- **物理行动** （通过控制硬件）：控制机械臂移动、让音箱播放声音。

## 实例

\# 行动模块：负责"执行"决策模块给出的指令  
def execute\_action(decision):  
"""  
执行决策模块给出的指令，并返回执行结果。  
执行结果会被存入记忆，供下一轮推理使用。  
"""  
print(f"\[行动模块\] 正在执行：{decision}")  
  
if decision == "调用天气查询工具":  
\# 这里可以替换为真实的天气 API 调用  
result = "北京：晴，25℃。"  
elif decision == "调用计算器工具":  
result = "1+1=2"  
elif decision == "执行终止动作":  
result = "任务结束。"  
print(result)  
exit() # 结束整个程序  
else:  
result = f"我已理解您的意思：'{decision}'"  
  
print(f"\[行动模块\] 行动结果：{result}")  
return result # 返回结果，进入"观察"阶段  
  
\# 执行决策，得到结果  
action\_result = execute\_action(current\_decision)

### 4\. 记忆模块 —— Agent 的"工作笔记"

如果没有记忆，Agent 每次回复都是"失忆"状态——忘了你之前说过什么，忘了它自己做过什么。记忆模块解决了这个问题，它分两类：

- **短期记忆 / 对话历史** ：记录本次对话中说过的话，让 Agent 在多轮对话中保持连贯。就像你和朋友聊天，你不需要每句话都重新解释背景。
- **长期记忆 / 知识库** ：通过向量数据库等技术存储的专属知识（例如公司内部文档、用户偏好），用于增强模型的能力。初学者可暂时忽略这部分，先掌握短期记忆即可。

在下面的完整示例中，我们用一个 Python 列表来模拟短期记忆。

### 5\. 工具模块 —— Agent 的"瑞士军刀"

模型本身的能力是有限的（比如不知道实时天气、不能做复杂计算、不能操作文件）。工具模块为 Agent 提供了一套"外挂技能"，极大地扩展了其能力边界。工具可以是一个 Python 函数、一个第三方 API，或者一个完整的外部软件。

下面是一个最简单的工具示例：

## 实例

\# 工具模块示例：一个简单的计算器工具  
def calculator\_tool(expression):  
"""  
接收一个数学表达式字符串，返回计算结果。  
这就是一个"工具"——等待被 Agent 的大脑按需调用。  
"""  
try:  
\# 注意：eval() 在生产环境中有安全风险，此处仅用于演示。  
result = eval(expression)  
return f"计算结果：{expression} = {result}"  
except Exception as e:  
return f"计算错误：{e}"  
  
\# 模拟：大脑决策后，调用这个工具  
tool\_result = calculator\_tool("3 + 5 \* 2")  
print(tool\_result) # 输出：计算结果：3 + 5 \* 2 = 13  
\# 注意：Python 先算乘除后算加减，所以是 3 + 10 = 13，而不是 8 \* 2 = 16

---

## 实践练习：构建一个简单的命令行 AI Agent

上面我们分别介绍了每个模块。现在，让我们把它们 **组合** 起来，创建一个能进行持续对话和工具调用的完整迷你 Agent。

运行前，请先在脑海中过一遍这个结构：

- 用户输入 → **感知** → **决策** （关键词匹配）→ **行动** （调用工具或对话）→ 输出结果 → **记忆** （存入历史）→ 等待下一轮输入

## 实例

\# 完整迷你 AI Agent 示例  
import random  
  
\# ==================== 工具定义 ====================  
def get\_weather(city):  
"""工具1：模拟天气查询（实际可替换为真实天气 API）"""  
weather\_options = \["晴", "多云", "小雨", "大风"\]  
temperature = random.randint(15, 35)  
return f"{city}的天气是{random.choice(weather\_options)}，气温{temperature}℃。"  
  
def simple\_calculator(a, b, operator):  
"""工具2：简单计算器"""  
if operator == '+':  
return f"{a} + {b} = {a + b}"  
elif operator == '-':  
return f"{a} - {b} = {a - b}"  
else:  
return "暂不支持此运算。"  
  
\# ==================== 记忆（短期）====================  
\# 用列表模拟对话历史，每一条都是一个字符串  
conversation\_history = \[\]  
  
\# ==================== Agent 主循环 ====================  
def run\_simple\_agent():  
print("【简单AI Agent已启动】输入'退出'来结束对话。")  
print("提示：你可以问天气（如'北京今天天气'）或做计算（如'计算1+1'）\\n")  
  
while True:  
\# ---- 感知阶段：获取用户输入 ----  
user\_input = input("您：")  
conversation\_history.append(f"用户：{user\_input}")  
  
\# 特殊指令：退出  
if user\_input.lower() in \["退出", "exit", "quit"\]:  
print("Agent：再见！")  
break  
  
\# ---- 决策 + 行动阶段：根据输入选择工具或对话 ----  
response = ""  
  
if "天气" in user\_input:  
\# 尝试从输入中识别城市名，默认用北京  
city = "北京"  
for c in \["北京", "上海", "广州", "深圳"\]:  
if c in user\_input:  
city = c  
break  
\# 调用天气工具  
response = get\_weather(city)  
  
elif "计算" in user\_input or "+" in user\_input or "-" in user\_input:  
\# 简单的模式匹配，识别几个固定的计算表达式  
if "1+1" in user\_input:  
response = simple\_calculator(1, 1, '+')  
elif "10-5" in user\_input:  
response = simple\_calculator(10, 5, '-')  
else:  
response = "请尝试输入'计算1+1'或'计算10-5'。"  
  
else:  
\# 没有匹配到工具，退化为普通对话  
default\_responses = \[  
"我理解您的意思了。",  
"这是一个有趣的话题！",  
"我目前的技能有限，可以试试问我天气或做简单计算。",  
"嗯，请继续说。"  
\]  
response = random.choice(default\_responses)  
  
\# ---- 输出 + 记忆阶段：打印回复，并存入历史 ----  
print(f"Agent：{response}")  
conversation\_history.append(f"Agent：{response}")  
  
\# 对话结束后，打印完整的对话历史（模拟短期记忆的内容）  
print("\\n=== 本次对话历史（短期记忆内容） ===")  
for line in conversation\_history:  
print(line)  
  
\# 启动 Agent  
if \_\_name\_\_ == "\_\_main\_\_":  
run\_simple\_agent()

**运行这个程序，你将体验到：**

- 一个能 **持续多轮对话** 的循环（感知模块不断运转）。
- 根据你的输入关键词（如"天气"、"计算"） **自动触发不同的工具** （行动模块）。
- 对话结束后可以看到完整的 **对话历史记录** （记忆模块）。

💡 **思考一下** ：如果你想让这个 Agent 更强大，可以尝试：① 增加更多工具（如查汇率、翻译）；② 把决策模块中的关键词匹配替换为真实的 LLM API 调用，Agent 就能理解自然语言了！

---

## 总结与展望

通过本文，你应该已经掌握了 AI Agent 的基本概念：

- 它是由 **感知、决策（大脑）、行动、记忆、工具** 等模块组成的智能程序；
- 它通过 **ReAct 循环** （思考 → 行动 → 观察 → 再思考）自主追求目标；
- LLM 是它的大脑，工具是它的手脚，记忆让它保持连贯。

一句话记住核心： **LLM 只能"说"，Agent 能"做"。**

### 下一步学习建议

- **深入大语言模型（LLM）API** ：学习如何调用 OpenAI GPT、DeepSeek、通义千问等模型的 API，将它们作为你 Agent 的真正大脑，替换掉我们示例中简陋的关键词匹配。
- **学习 Agent 开发框架** ：探索 **LangChain** 、 **LlamaIndex** 或 **AutoGen** 等成熟框架。它们把记忆、工具链、任务编排等模块都封装好了，用它们可以事半功倍，避免重复造轮子。
- **接入真实工具** ：尝试将你的 Agent 连接到真实的 API，例如数据库、邮件系统或天气服务，解决实际问题。这一步会让你对"工具"的概念有全新的体会。
- **学习提示工程（Prompt Engineering）** ：如何给 LLM 写出高质量的"指令"，直接决定 Agent 大脑的发挥水平。这是开发高效 Agent 不可忽视的关键技能。

AI Agent 的世界广阔而充满可能，从自动化个人助手到企业级智能解决方案，它正在成为人机交互的新范式。希望你以此文为起点，开始构建属于自己的智能体。