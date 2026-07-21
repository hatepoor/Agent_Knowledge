---
title: "大模型多模态（Multimodal） | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/llm-multimodal.html"
author:
published:
created: 2026-06-23
description: "大模型多模态（Multimodal） 多模态（Multimodal）大模型让 AI 同时处理文本、图像、音频、视频等多种信息形式，是当前大模型最核心的发展方向之一。 本文从零开始讲解多模态的概念、发展过程、内部原理与典型能力，并在文末给出 5 分钟搭建第一个多模态应用 的可运行示例。    一、什么是多模态？ 普通大模型通常只能处理一种信息——文本，如果给传统 AI 一张图片，并问它'图片里有几只猫？'，如果不支持多.."
tags:
  - "大模型多模态（Multimodal）"
---
## 大模型多模态（Multimodal）

多模态（Multimodal）大模型让 AI 同时处理文本、图像、音频、视频等多种信息形式，是当前大模型最核心的发展方向之一。

本文从零开始讲解多模态的概念、发展过程、内部原理与典型能力，并在文末给出 5 分钟搭建第一个多模态应用 的可运行示例。

---

## 一、什么是多模态？

普通大模型通常只能处理一种信息——文本，如果给传统 AI 一张图片，并问它"图片里有几只猫？"，如果不支持多模态，该模型就做不到。

而现实世界不是由文字组成的，而是由多种感官信息共同构成：

| 感官 | 对应信息 |
| --- | --- |
| 眼睛 | 图像 |
| 耳朵 | 声音 |
| 嘴巴 | 语言 |
| 手 | 动作 |
| 视频 | 图像 + 时间 |
| 网页 | 文本 + 图片 + 结构 |

因此，多模态（Multimodal） 这一概念应运而生。

用一句话理解：

> 多模态 = 一个模型同时理解和生成多种信息形式。

看两个典型例子：

- 1、输入图片加问题： **"这张图里的食物热量高吗？"** ，模型回答： **"这是炸鸡和薯条，预计约 1200 大卡。"**
- 2、输入语音： **"帮我总结这段会议"** ，模型输出文字摘要并附带行动建议。

这种看图说话、听音总结的能力，就是多模态最直接的体现。

---

## 二、什么是模态（Modal）？

模态（Modality）就是信息的表现形式。

不同的模态对应不同的输入与输出，下表列出常见模态及其典型应用：

| 模态 | 输入示例 | 输出示例 | 典型应用 |
| --- | --- | --- | --- |
| Text（文本） | 小说、聊天记录 | 回答、文章 | 问答、写作、翻译 |
| Image（图像） | 照片、截图 | 图片理解、生成图 | OCR、视觉问答 |
| Audio（音频） | 语音、音乐 | 转写文字、合成语音 | 会议摘要、语音助手 |
| Video（视频） | 视频流 | 视频分析、时间轴 | 教学总结、内容检索 |
| Sensor（传感器） | GPS、温度 | 控制信号 | 机器人、自动驾驶 |
| Action（动作） | 鼠标点击、键盘输入 | 执行动作 | GUI Agent、自动化 |

模型支持的模态越多，能力就越接近人，这也是为什么所有头部实验室都在把文字模型升级为多模态模型。

![](https://www.runoob.com/wp-content/uploads/2026/06/7229d673-75b7-489d-a30e-bcaca789bf80.webp)

---

## 三、多模态大模型的发展过程

从只能处理文字，到能看、能听、能输出视频，多模态大模型经历了三个清晰的阶段。

<svg viewBox="0 0 880 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;"><defs><linearGradient id="stage1" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#3498db"></stop><stop offset="100%" stop-color="#2c3e50"></stop></linearGradient><linearGradient id="stage2" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#1abc9c"></stop><stop offset="100%" stop-color="#16a085"></stop></linearGradient><linearGradient id="stage3" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e67e22"></stop><stop offset="100%" stop-color="#d35400"></stop></linearGradient><marker id="arrow" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#7f8c8d"></polygon></marker></defs><text x="440" y="32" text-anchor="middle" fill="#2c3e50" font-size="18" font-family="sans-serif" font-weight="bold">多模态大模型发展三阶段</text> <text x="440" y="52" text-anchor="middle" fill="#7f8c8d" font-size="12" font-family="sans-serif">从单一文字到原生多模态，每一步都让模型更接近"人"的理解方式</text> <line x1="80" y1="200" x2="800" y2="200" stroke="#bdc3c7" stroke-width="3" stroke-dasharray="6,4"></line><circle cx="160" cy="200" r="28" fill="url(#stage1)" stroke="#1c3d5a" stroke-width="2"></circle><text x="160" y="206" text-anchor="middle" fill="#fff" font-size="16" font-family="sans-serif" font-weight="bold">1</text> <rect x="60" y="240" width="200" height="170" rx="8" fill="#ecf0f1" stroke="#bdc3c7" stroke-width="1"></rect><text x="160" y="262" text-anchor="middle" fill="#2c3e50" font-size="14" font-family="sans-serif" font-weight="bold">第一阶段：LLM</text> <text x="160" y="280" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif">（语言模型）</text> <text x="80" y="305" fill="#2c3e50" font-size="11" font-family="sans-serif" font-weight="bold">输入：文字</text> <text x="80" y="322" fill="#2c3e50" font-size="11" font-family="sans-serif" font-weight="bold">输出：文字</text> <text x="80" y="345" fill="#2c3e50" font-size="11" font-family="sans-serif" font-weight="bold">能力：</text> <text x="80" y="362" fill="#34495e" font-size="10" font-family="sans-serif">写代码 / 写文章</text> <text x="80" y="378" fill="#34495e" font-size="10" font-family="sans-serif">翻译 / 推理 / 问答</text> <text x="80" y="396" fill="#c0392b" font-size="10" font-family="sans-serif" font-weight="bold">限制：看不见世界</text> <text x="160" y="135" text-anchor="middle" fill="#2c3e50" font-size="10" font-family="sans-serif" font-style="italic">代表：GPT、Claude、Gemini</text> <circle cx="440" cy="200" r="28" fill="url(#stage2)" stroke="#0e6655" stroke-width="2"></circle><text x="440" y="206" text-anchor="middle" fill="#fff" font-size="16" font-family="sans-serif" font-weight="bold">2</text> <rect x="340" y="240" width="200" height="170" rx="8" fill="#e8f6f3" stroke="#1abc9c" stroke-width="1"></rect><text x="440" y="262" text-anchor="middle" fill="#16a085" font-size="14" font-family="sans-serif" font-weight="bold">第二阶段：VLM</text> <text x="440" y="280" text-anchor="middle" fill="#16a085" font-size="12" font-family="sans-serif">（视觉语言模型）</text> <text x="360" y="305" fill="#16a085" font-size="11" font-family="sans-serif" font-weight="bold">输入：图片 + 文字</text> <text x="360" y="322" fill="#16a085" font-size="11" font-family="sans-serif" font-weight="bold">输出：文字</text> <text x="360" y="345" fill="#16a085" font-size="11" font-family="sans-serif" font-weight="bold">新增能力：</text> <text x="360" y="362" fill="#34495e" font-size="10" font-family="sans-serif">看图 / OCR</text> <text x="360" y="378" fill="#34495e" font-size="10" font-family="sans-serif">图表理解 / 截图问答</text> <text x="360" y="396" fill="#c0392b" font-size="10" font-family="sans-serif" font-weight="bold">限制：仅"看"图</text> <text x="440" y="135" text-anchor="middle" fill="#16a085" font-size="10" font-family="sans-serif" font-style="italic">代表：GPT-4V、Gemini 1.5、Qwen-VL</text> <circle cx="720" cy="200" r="28" fill="url(#stage3)" stroke="#a04000" stroke-width="2"></circle><text x="720" y="206" text-anchor="middle" fill="#fff" font-size="16" font-family="sans-serif" font-weight="bold">3</text> <rect x="620" y="240" width="200" height="170" rx="8" fill="#fdebd0" stroke="#e67e22" stroke-width="1"></rect><text x="720" y="262" text-anchor="middle" fill="#d35400" font-size="14" font-family="sans-serif" font-weight="bold">第三阶段：原生多模态</text> <text x="720" y="280" text-anchor="middle" fill="#d35400" font-size="12" font-family="sans-serif">（统一模型）</text> <text x="640" y="305" fill="#d35400" font-size="11" font-family="sans-serif" font-weight="bold">输入：任意模态组合</text> <text x="640" y="322" fill="#d35400" font-size="11" font-family="sans-serif" font-weight="bold">输出：任意模态</text> <text x="640" y="345" fill="#d35400" font-size="11" font-family="sans-serif" font-weight="bold">完整能力：</text> <text x="640" y="362" fill="#34495e" font-size="10" font-family="sans-serif">看 / 听 / 说 / 推理</text> <text x="640" y="378" fill="#34495e" font-size="10" font-family="sans-serif">视频 / 实时交互 / Agent</text> <text x="640" y="396" fill="#16a085" font-size="10" font-family="sans-serif" font-weight="bold">现状：主流方向</text> <text x="720" y="135" text-anchor="middle" fill="#d35400" font-size="10" font-family="sans-serif" font-style="italic">代表：GPT-5、Gemini 2.5、Claude 4.6</text><line x1="188" y1="200" x2="412" y2="200" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrow)"></line><line x1="468" y1="200" x2="692" y2="200" stroke="#7f8c8d" stroke-width="2" marker-end="url(#arrow)"></line></svg>

### 第一阶段：语言模型（LLM）

这一阶段模型的输入和输出都是文字，代表产品包括 OpenAI GPT 系列、Anthropic Claude、Google Gemini。

它们能写代码、写文章、做翻译、回答问题，能力很强，但有一个根本限制：看不见。

### 第二阶段：视觉语言模型（VLM）

VLM 在 LLM 基础上增加了"看图"能力，典型用法是上传一张网页截图，问"为什么布局错了"，模型回答"CSS 冲突"。

这一阶段模型开始具备：图像理解、OCR、图表理解等能力。

### 第三阶段：原生多模态模型

原生多模态模型不再把"图"和"文"当作两个拼接的模块，而是从训练开始就统一处理图像、声音、视频、文本。

它具备完整能力：看、听、说、推理、执行，这也是当前业界的主流方向。

---

## 四、多模态模型内部是怎么工作的？

多模态模型的内部工作可以概括为一句话：把所有数据转成统一向量空间。

例如：

> "一只猫"、"cat"、cat.png 三种表达，虽然来自不同模态，编码后在向量空间里会落在接近区域。

整体流程如下图所示：

<svg viewBox="0 0 880 540" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;"><defs><marker id="arrow2" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#555"></polygon></marker><linearGradient id="inputGrad" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#3498db"></stop><stop offset="100%" stop-color="#2980b9"></stop></linearGradient><linearGradient id="encoderGrad" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#1abc9c"></stop><stop offset="100%" stop-color="#16a085"></stop></linearGradient><linearGradient id="tfGrad" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e67e22"></stop><stop offset="100%" stop-color="#d35400"></stop></linearGradient><linearGradient id="outputGrad" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#9b59b6"></stop><stop offset="100%" stop-color="#8e44ad"></stop></linearGradient><radialGradient id="vecGrad" cx="0.5" cy="0.5" r="0.5"><stop offset="0%" stop-color="#fff5d6"></stop><stop offset="100%" stop-color="#f1c40f"></stop></radialGradient></defs><text x="440" y="32" text-anchor="middle" fill="#2c3e50" font-size="18" font-family="sans-serif" font-weight="bold">多模态模型内部工作流程</text> <text x="440" y="52" text-anchor="middle" fill="#7f8c8d" font-size="12" font-family="sans-serif">把不同模态映射到统一向量空间，再由 Transformer 统一推理</text> <rect x="30" y="80" width="820" height="380" rx="10" fill="#fafbfc" stroke="#ecf0f1" stroke-width="1"></rect><text x="100" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">输入层</text> <rect x="50" y="115" width="100" height="36" rx="6" fill="url(#inputGrad)"></rect><text x="100" y="138" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Text 文字</text> <rect x="50" y="160" width="100" height="36" rx="6" fill="url(#inputGrad)"></rect><text x="100" y="183" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Image 图像</text> <rect x="50" y="205" width="100" height="36" rx="6" fill="url(#inputGrad)"></rect><text x="100" y="228" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Audio 音频</text> <rect x="50" y="250" width="100" height="36" rx="6" fill="url(#inputGrad)"></rect><text x="100" y="273" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Video 视频</text> <text x="240" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">编码器层</text> <rect x="190" y="115" width="100" height="36" rx="6" fill="url(#encoderGrad)"></rect><text x="240" y="138" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Text Encoder</text> <rect x="190" y="160" width="100" height="36" rx="6" fill="url(#encoderGrad)"></rect><text x="240" y="183" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Vision Encoder</text> <rect x="190" y="205" width="100" height="36" rx="6" fill="url(#encoderGrad)"></rect><text x="240" y="228" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Audio Encoder</text> <rect x="190" y="250" width="100" height="36" rx="6" fill="url(#encoderGrad)"></rect><text x="240" y="273" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">Video Encoder</text> <line x1="150" y1="133" x2="188" y2="133" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="150" y1="178" x2="188" y2="178" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="150" y1="223" x2="188" y2="223" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="150" y1="268" x2="188" y2="268" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><text x="400" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">统一语义空间</text> <ellipse cx="400" cy="200" rx="80" ry="90" fill="url(#vecGrad)" stroke="#e67e22" stroke-width="2" opacity="0.85"></ellipse><text x="400" y="180" text-anchor="middle" fill="#7d6608" font-size="11" font-family="sans-serif" font-weight="bold">Embedding</text> <text x="400" y="195" text-anchor="middle" fill="#7d6608" font-size="11" font-family="sans-serif" font-weight="bold">Vector Space</text> <circle cx="380" cy="220" r="4" fill="#e74c3c"></circle><circle cx="395" cy="225" r="4" fill="#e74c3c"></circle><circle cx="410" cy="218" r="4" fill="#e74c3c"></circle><circle cx="420" cy="232" r="4" fill="#e74c3c"></circle><circle cx="385" cy="240" r="4" fill="#e74c3c"></circle><text x="400" y="265" text-anchor="middle" fill="#7d6608" font-size="10" font-family="sans-serif" font-style="italic">"猫" / "cat" / 图像 在这里彼此接近</text> <line x1="290" y1="133" x2="335" y2="170" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="290" y1="178" x2="335" y2="190" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="290" y1="223" x2="335" y2="210" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="290" y1="268" x2="335" y2="230" stroke="#555" stroke-width="1.5" marker-end="url(#arrow2)"></line><text x="570" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">Transformer</text> <rect x="500" y="130" width="140" height="140" rx="8" fill="url(#tfGrad)" stroke="#a04000" stroke-width="1"></rect><text x="570" y="170" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">Self-Attention</text> <text x="570" y="190" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">+ Feed Forward</text> <text x="570" y="215" text-anchor="middle" fill="#fde2cf" font-size="11" font-family="sans-serif">统一推理</text> <text x="570" y="232" text-anchor="middle" fill="#fde2cf" font-size="11" font-family="sans-serif">跨模态关联</text> <text x="570" y="252" text-anchor="middle" fill="#fde2cf" font-size="11" font-family="sans-serif">Attention(Q, K, V)</text> <line x1="480" y1="200" x2="498" y2="200" stroke="#555" stroke-width="1.8" marker-end="url(#arrow2)"></line><text x="700" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">解码器</text> <rect x="650" y="160" width="100" height="80" rx="6" fill="url(#encoderGrad)"></rect><text x="700" y="190" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif" font-weight="bold">Decoder</text> <text x="700" y="210" text-anchor="middle" fill="#d6f5ee" font-size="10" font-family="sans-serif">按需生成</text> <text x="700" y="225" text-anchor="middle" fill="#d6f5ee" font-size="10" font-family="sans-serif">任意模态</text> <line x1="640" y1="200" x2="648" y2="200" stroke="#555" stroke-width="1.8" marker-end="url(#arrow2)"></line><text x="820" y="100" text-anchor="middle" fill="#2c3e50" font-size="12" font-family="sans-serif" font-weight="bold">输出层</text> <rect x="770" y="115" width="80" height="30" rx="5" fill="url(#outputGrad)"></rect><text x="810" y="135" text-anchor="middle" fill="#fff" font-size="11" font-family="sans-serif" font-weight="bold">文字</text> <rect x="770" y="155" width="80" height="30" rx="5" fill="url(#outputGrad)"></rect><text x="810" y="175" text-anchor="middle" fill="#fff" font-size="11" font-family="sans-serif" font-weight="bold">图像</text> <rect x="770" y="195" width="80" height="30" rx="5" fill="url(#outputGrad)"></rect><text x="810" y="215" text-anchor="middle" fill="#fff" font-size="11" font-family="sans-serif" font-weight="bold">音频</text> <rect x="770" y="235" width="80" height="30" rx="5" fill="url(#outputGrad)"></rect><text x="810" y="255" text-anchor="middle" fill="#fff" font-size="11" font-family="sans-serif" font-weight="bold">视频</text> <line x1="750" y1="180" x2="768" y2="130" stroke="#555" stroke-width="1.2" marker-end="url(#arrow2)"></line><line x1="750" y1="200" x2="768" y2="170" stroke="#555" stroke-width="1.2" marker-end="url(#arrow2)"></line><line x1="750" y1="220" x2="768" y2="210" stroke="#555" stroke-width="1.2" marker-end="url(#arrow2)"></line><line x1="750" y1="240" x2="768" y2="250" stroke="#555" stroke-width="1.2" marker-end="url(#arrow2)"></line><rect x="60" y="490" width="760" height="36" rx="6" fill="#fef9e7" stroke="#f1c40f" stroke-width="1"></rect><text x="440" y="513" text-anchor="middle" fill="#7d6608" font-size="12" font-family="sans-serif" font-weight="bold">核心思想：把任何模态都映射到同一个高维向量空间，让 Transformer 用统一的方式理解它们</text></svg>

上图展示了从输入到输出的完整链路，下面拆解每一步。

### 第一步：编码（Encoding）

不同模态要先转成数字，模型才能处理。

| 模态 | 原始数据 | 编码后 |
| --- | --- | --- |
| 文本 | "你好" | \[2034, 789\] 一组 token id |
| 图像 | 像素矩阵 | 视觉特征向量 |
| 音频 | 波形 | 频谱特征向量 |

### 第二步：统一语义空间

编码后的关键一步，是让"猫"、"cat"、cat.png 在向量空间里彼此接近。

这就是"跨模态理解"的本质：把不同模态的特征投影到同一个空间里。

### 第三步：生成输出

在统一空间里推理之后，模型按需把结果"还原"成不同模态。

输出可以是：文本、图像、音频、视频中的任意一种。

例如：上传一张图片 → 描述 → 再生成一段介绍视频，这是完整的多模态生成链。

---

## 五、为什么 Transformer 能统一多模态？

核心原因只有一个：Attention 机制。

Transformer 的核心公式是 Attention(Q, K, V)，含义是"模型自己判断该关注什么"。

举个例子：当用户问"图片里谁在打篮球？"时，模型会：

1. 先看图，识别出图中所有人物
2. 识别出图中的篮球
3. 判断"谁"和"篮球"之间的关系
4. 综合上下文输出答案

这种"自主决定关注点"的能力，让 Transformer 天然适合处理多模态输入：

| 输入 | 模型在关注什么 |
| --- | --- |
| 纯文字 | 句子之间的语义关联 |
| 图片 + 文字 | 图像区域和文字描述的对应 |
| 视频 + 文字 | 时间轴片段和问题的关联 |

> 这也是为什么 Attention 几乎成了所有现代大模型的"标配"——它天生就是为跨模态设计的。

---

## 六、多模态典型能力

多模态模型具备五大典型能力，下图展示了它们之间的关系。

<svg viewBox="0 0 880 540" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;"><defs><radialGradient id="coreGrad" cx="0.5" cy="0.5" r="0.5"><stop offset="0%" stop-color="#3498db"></stop><stop offset="100%" stop-color="#2c3e50"></stop></radialGradient><linearGradient id="cap1" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#1abc9c"></stop><stop offset="100%" stop-color="#16a085"></stop></linearGradient><linearGradient id="cap2" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e67e22"></stop><stop offset="100%" stop-color="#d35400"></stop></linearGradient><linearGradient id="cap3" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#9b59b6"></stop><stop offset="100%" stop-color="#8e44ad"></stop></linearGradient><linearGradient id="cap4" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e74c3c"></stop><stop offset="100%" stop-color="#c0392b"></stop></linearGradient><linearGradient id="cap5" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#f1c40f"></stop><stop offset="100%" stop-color="#f39c12"></stop></linearGradient></defs><text x="440" y="32" text-anchor="middle" fill="#2c3e50" font-size="18" font-family="sans-serif" font-weight="bold">多模态大模型的五大典型能力</text> <text x="440" y="52" text-anchor="middle" fill="#7f8c8d" font-size="12" font-family="sans-serif">一个统一模型，覆盖"看、听、说、做"的完整链路</text> <circle cx="440" cy="280" r="62" fill="url(#coreGrad)" stroke="#1c3d5a" stroke-width="2"></circle><text x="440" y="275" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">多模态</text> <text x="440" y="293" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">大模型</text> <text x="440" y="312" text-anchor="middle" fill="#d6eaf8" font-size="10" font-family="sans-serif">Unified Model</text> <line x1="395" y1="240" x2="270" y2="135" stroke="#1abc9c" stroke-width="2"></line><rect x="120" y="85" width="180" height="110" rx="8" fill="url(#cap1)" stroke="#0e6655" stroke-width="1"></rect><text x="210" y="112" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">图像理解</text> <text x="210" y="130" text-anchor="middle" fill="#d6f5ee" font-size="10" font-family="sans-serif">OCR / 图表 / 截图问答</text> <text x="135" y="152" fill="#d6f5ee" font-size="10" font-family="sans-serif">典型应用：</text> <text x="135" y="168" fill="#fff" font-size="10" font-family="sans-serif">· 报错截图 → AI 定位问题</text> <text x="135" y="184" fill="#fff" font-size="10" font-family="sans-serif">· 论文图表自动解读</text> <line x1="485" y1="240" x2="610" y2="135" stroke="#e67e22" stroke-width="2"></line><rect x="580" y="85" width="180" height="110" rx="8" fill="url(#cap2)" stroke="#a04000" stroke-width="1"></rect><text x="670" y="112" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">图像生成</text> <text x="670" y="130" text-anchor="middle" fill="#fde2cf" font-size="10" font-family="sans-serif">文生图 / 图生图</text> <text x="595" y="152" fill="#fde2cf" font-size="10" font-family="sans-serif">典型应用：</text> <text x="595" y="168" fill="#fff" font-size="10" font-family="sans-serif">· 海报 / 封面 / 插画</text> <text x="595" y="184" fill="#fff" font-size="10" font-family="sans-serif">· 游戏素材 / UI 草图</text> <line x1="380" y1="290" x2="210" y2="290" stroke="#9b59b6" stroke-width="2"></line><rect x="50" y="250" width="180" height="110" rx="8" fill="url(#cap3)" stroke="#5b2c6f" stroke-width="1"></rect><text x="140" y="277" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">语音交互</text> <text x="140" y="295" text-anchor="middle" fill="#e8daef" font-size="10" font-family="sans-serif">ASR + TTS + 对话</text> <text x="65" y="317" fill="#e8daef" font-size="10" font-family="sans-serif">典型应用：</text> <text x="65" y="333" fill="#fff" font-size="10" font-family="sans-serif">· 实时语音助手</text> <text x="65" y="349" fill="#fff" font-size="10" font-family="sans-serif">· 会议转写与摘要</text> <line x1="500" y1="290" x2="670" y2="290" stroke="#e74c3c" stroke-width="2"></line><rect x="650" y="250" width="180" height="110" rx="8" fill="url(#cap4)" stroke="#922b21" stroke-width="1"></rect><text x="740" y="277" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">视频理解</text> <text x="740" y="295" text-anchor="middle" fill="#fadbd8" font-size="10" font-family="sans-serif">长视频 / 时间轴 / 问答</text> <text x="665" y="317" fill="#fadbd8" font-size="10" font-family="sans-serif">典型应用：</text> <text x="665" y="333" fill="#fff" font-size="10" font-family="sans-serif">· 教学视频自动总结</text> <text x="665" y="349" fill="#fff" font-size="10" font-family="sans-serif">· 监控关键事件检索</text> <line x1="440" y1="342" x2="440" y2="430" stroke="#f1c40f" stroke-width="2"></line><rect x="320" y="430" width="240" height="90" rx="8" fill="url(#cap5)" stroke="#9c7a08" stroke-width="1"></rect><text x="440" y="457" text-anchor="middle" fill="#fff" font-size="14" font-family="sans-serif" font-weight="bold">Agent 智能体</text> <text x="440" y="475" text-anchor="middle" fill="#fef9e7" font-size="10" font-family="sans-serif">搜索 / 生成 / 执行 / 反馈</text> <text x="335" y="497" fill="#fef9e7" font-size="10" font-family="sans-serif">· 一句话完成 PPT / 海报 / 调研</text> <text x="335" y="513" fill="#fef9e7" font-size="10" font-family="sans-serif">· 串联多个工具形成"行动链"</text> <rect x="60" y="500" width="200" height="32" rx="6" fill="#ecf0f1" stroke="#bdc3c7" stroke-width="1"></rect><text x="160" y="521" text-anchor="middle" fill="#2c3e50" font-size="11" font-family="sans-serif" font-weight="bold">五种能力可自由组合</text></svg>

### 1\. 图像理解

输入截图，模型输出页面问题、UI 分析或 OCR 结果。

程序员最常用的场景是：截图报错，让 AI 定位问题。

### 2\. 图像生成

输入"生成未来城市"这样的文字描述，模型返回 AI 图片。

典型应用：海报、封面、游戏素材、产品草图。

### 3\. 语音交互

说话即输入，模型实时回答，能力覆盖 ASR（语音转文字）、TTS（文字转语音）、对话。

### 4\. 视频理解

上传一段教学视频，模型输出总结、时间轴、可问答的关键片段。

### 5\. Agent 智能体

输入"帮我做 PPT"，模型开始执行完整行动链：搜索资料 → 生成大纲 → 制作幻灯片 → 修改 → 导出。

Agent 是把"看、听、说"和"动手执行"结合的最终形态。

---

## 七、初学者必须理解的五个术语

阅读多模态相关文档时，下表中的术语会反复出现，先记住它们：

| 术语 | 含义 | 举例 |
| --- | --- | --- |
| Token（词元） | 模型处理的最小单位，文本按词切分 | "你好世界" → \[你\]\[好\]\[世界\] |
| Embedding（嵌入） | 把信息转成数字坐标的过程 | "猫" → \[0.12, 0.78, 0.33,...\] |
| Context（上下文） | 当前对话的历史与背景信息 | 用户说"苹果"+"上下文：手机"= iPhone，而非水果 |
| Attention（注意力） | 决定模型关注输入哪一部分 | 看图时模型先看"人"再看"篮球" |
| Inference（推理） | 模型根据输入生成输出的过程 | 用户输入问题后，模型"思考并回答" |

> 这五个词贯穿所有大模型文档，先建立直觉，遇到细节再回查。

---

## 八、动手体验：5 分钟搭建你的第一个多模态应用

本节目标：上传一张图片，AI 输出文字描述。

整个流程不需要任何深度学习基础，只要会 Python 即可。

### 8.1 准备项目

先创建一个项目 multimodal-demo，并使用 openai 库来测试：

## 实例

\# 创建项目  
mkdir multimodal-demo  
  
\# 进入项目目录  
cd multimodal-demo  
  
\# 安装 OpenAI 官方 SDK（兼容多种多模态模型）  
pip install openai

在 multimodal-demo 目录下上次一张图片 cat-cartoon.webp，图片如下（可以右击保存下来测试）：

[![](https://www.runoob.com/wp-content/uploads/2026/06/cat-cartoon.webp)](https://www.runoob.com/wp-content/uploads/2026/06/cat-cartoon.webp)

### 8.2 编写调用代码

本实例采用的是 [节方舟的 Coding Plan](https://www.volcengine.com/activity/ai618?utm_campaign=hw&utm_content=hw&utm_medium=devrel_tool_web&utm_source=OWO&utm_term=runoob) ，他们的 doubao-seed-2.0-pro 模型支持多模态。

把下面这段代码保存为 test.py：

## 实例

import base64  
import os  
from io import BytesIO  
from openai import OpenAI  
from PIL import Image  
  
\# ==================== 方舟配置 ====================  
ARK\_API\_KEY = "ark-xxxx" # 这里填写你的 Key  
ARK\_BASE\_URL = "https://ark.cn-beijing.volces.com/api/coding/v3" # 推荐使用通用 v3 接口  
MODEL\_ENDPOINT = "doubao-seed-2.0-pro" # 支持多模态的模型  
  
\# 初始化客户端  
client = OpenAI(  
api\_key=ARK\_API\_KEY,  
base\_url=ARK\_BASE\_URL,  
timeout=60  
)  
  
def prepare\_image\_base64(image\_path: str, max\_size: int = 1024) -> str:  
"""  
读取本地图片，处理透明通道，缩放并转换为 base64 字符串  
"""  
if not os.path.exists(image\_path):  
raise FileNotFoundError(f"找不到测试图片: {image\_path}")  
  
with Image.open(image\_path) as img:  
\# 1. 处理 RGBA 透明背景  
if img.mode == "RGBA":  
background = Image.new("RGB", img.size, (255, 255, 255))  
background.paste(img, mask=img.split()\[-1\])  
img = background  
elif img.mode!= "RGB":  
img = img.convert("RGB")  
  
\# 2. 等比例缩放（防大图撑爆 Payload/Token）  
img.thumbnail((max\_size, max\_size))  
  
\# 3. 转 Base64  
buf = BytesIO()  
img.save(buf, format="JPEG", quality=85) # 适当压缩质量  
b64 = base64.b64encode(buf.getvalue()).decode("utf-8")  
  
return f"data:image/jpeg;base64,{b64}"  
  
def multimodal\_chat(img\_path: str, prompt: str):  
"""多模态对话测试"""  
try:  
print(f"正在处理图片: {img\_path}...")  
img\_b64 = prepare\_image\_base64(img\_path)  
  
print("正在发送请求至火山引擎方舟平台...")  
res = client.chat.completions.create(  
model=MODEL\_ENDPOINT,  
messages=\[  
{  
"role": "user",  
"content": \[  
{"type": "text", "text": prompt},  
{"type": "image\_url", "image\_url": {"url": img\_b64}}  
\]  
}  
\],  
temperature=0.7,  
max\_tokens=1024  
)  
  
print("\\n" + "="\*10 + " 识别结果 " + "="\*10)  
print(res.choices\[0\].message.content)  
print("="\*30)  
return res  
  
except FileNotFoundError as e:  
print(f"【错误】{e}")  
except Exception as e:  
print(f"【API 请求失败】请检查网络或 Endpoint/Key 是否正确。错误信息:\\n{e}")  
return None  
  
if \_\_name\_\_ == "\_\_main\_\_":  
\# 测试配置  
image\_file = "./cat-cartoon.webp"  
question = "详细描述这张图片里的内容，分析画面主体、色彩和场景"  
  
\# 模拟创建一个临时测试图（若本地没有相应图片，方便快速肉眼 debug）  
if not os.path.exists(image\_file):  
print(f"未检测到 {image\_file}，正在生成一张临时测试图...")  
Image.new('RGB', (200, 200), color = (73, 109, 137)).save(image\_file)  
  
multimodal\_chat(image\_file, question)

运行 Pyhton，输出结果如下：

```
在发送请求至火山引擎方舟平台...

========== 识别结果 ==========
这是一张**Q版萌系卡通风格的数字角色插画**，整体背景为纯净的纯白色，没有任何环境元素，所有视觉重心都集中在中心的小猫主体上，是典型的独立角色立绘/吉祥物设计。
---
### 一、画面主体详细描述
主体是一只端正坐姿的卡通幼橘猫，是经过萌化设计的家养橘白虎斑猫形象，整体气质乖巧天真、亲人友好：
1.  **比例与姿态**：采用萌系设计特有的大头短身比例，头部占整体高度的60%左右，
...
```

打开浏览器控制台，就能看到模型对图片的文字描述。整个链路是：

> 用户上传图片 → 浏览器把图片 URL 传给 OpenAI SDK → 多模态模型理解图片 → 返回文字描述 → 打印到控制台。

### 8.3 常见扩展

完成上面这段最小可运行代码后，你可以继续扩展：

| 想做什么 | 怎么改 |
| --- | --- |
| 改成上传本地文件 | 用户上传图片后读取后转为 base64，再赋给 image\_file |
| 支持连续多轮对话 | 把历史问答追加到 messages 数组 |
| 换成其它厂商模型 | 构造 OpenAI 客户端时指定 baseURL，如 Qwen、OpenAI、Anthropic 兼容端点 |
| 处理语音 | 把 type: "image\_url" 换成 type: "input\_audio"，并提供 base64 音频 |

---

## 九、学习路线（推荐）

如果你刚接触大模型，下图是一条经过验证的 6 周学习路径。

<svg viewBox="0 0 880 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;"><defs><linearGradient id="week1" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#3498db"></stop><stop offset="100%" stop-color="#2980b9"></stop></linearGradient><linearGradient id="week2" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#1abc9c"></stop><stop offset="100%" stop-color="#16a085"></stop></linearGradient><linearGradient id="week3" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#9b59b6"></stop><stop offset="100%" stop-color="#8e44ad"></stop></linearGradient><linearGradient id="week4" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e67e22"></stop><stop offset="100%" stop-color="#d35400"></stop></linearGradient><linearGradient id="week5" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#e74c3c"></stop><stop offset="100%" stop-color="#c0392b"></stop></linearGradient><linearGradient id="week6" x1="0" y1="0" x2="0" y2="1"><stop offset="0%" stop-color="#f1c40f"></stop><stop offset="100%" stop-color="#f39c12"></stop></linearGradient><marker id="arrow" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#7f8c8d"></polygon></marker></defs><text x="440" y="32" text-anchor="middle" fill="#2c3e50" font-size="18" font-family="sans-serif" font-weight="bold">多模态学习路线（6 周）</text> <text x="440" y="52" text-anchor="middle" fill="#7f8c8d" font-size="12" font-family="sans-serif">从基础概念到完整项目，循序渐进建立能力</text> <line x1="60" y1="180" x2="820" y2="180" stroke="#bdc3c7" stroke-width="3"></line><circle cx="50" cy="180" r="10" fill="#7f8c8d"></circle><text x="50" y="155" text-anchor="middle" fill="#7f8c8d" font-size="11" font-family="sans-serif" font-weight="bold">起点</text> <circle cx="160" cy="180" r="22" fill="url(#week1)" stroke="#1c3d5a" stroke-width="1.5"></circle><text x="160" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W1</text> <rect x="110" y="210" width="100" height="70" rx="6" fill="#ebf5fb" stroke="#3498db" stroke-width="1"></rect><text x="160" y="232" text-anchor="middle" fill="#2c3e50" font-size="11" font-family="sans-serif" font-weight="bold">基础</text> <text x="160" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">Transformer</text> <text x="160" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">Token</text> <circle cx="280" cy="180" r="22" fill="url(#week2)" stroke="#0e6655" stroke-width="1.5"></circle><text x="280" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W2</text> <rect x="230" y="210" width="100" height="70" rx="6" fill="#e8f6f3" stroke="#1abc9c" stroke-width="1"></rect><text x="280" y="232" text-anchor="middle" fill="#16a085" font-size="11" font-family="sans-serif" font-weight="bold">上手</text> <text x="280" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">Prompt 工程</text> <text x="280" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">调用 API</text> <circle cx="400" cy="180" r="22" fill="url(#week3)" stroke="#5b2c6f" stroke-width="1.5"></circle><text x="400" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W3</text> <rect x="350" y="210" width="100" height="70" rx="6" fill="#f4ecf7" stroke="#9b59b6" stroke-width="1"></rect><text x="400" y="232" text-anchor="middle" fill="#8e44ad" font-size="11" font-family="sans-serif" font-weight="bold">数据</text> <text x="400" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">Embedding</text> <text x="400" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">向量数据库</text> <circle cx="520" cy="180" r="22" fill="url(#week4)" stroke="#a04000" stroke-width="1.5"></circle><text x="520" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W4</text> <rect x="470" y="210" width="100" height="70" rx="6" fill="#fdebd0" stroke="#e67e22" stroke-width="1"></rect><text x="520" y="232" text-anchor="middle" fill="#d35400" font-size="11" font-family="sans-serif" font-weight="bold">检索</text> <text x="520" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">RAG 原理</text> <text x="520" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">文档问答</text> <circle cx="640" cy="180" r="22" fill="url(#week5)" stroke="#922b21" stroke-width="1.5"></circle><text x="640" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W5</text> <rect x="590" y="210" width="100" height="70" rx="6" fill="#fadbd8" stroke="#e74c3c" stroke-width="1"></rect><text x="640" y="232" text-anchor="middle" fill="#c0392b" font-size="11" font-family="sans-serif" font-weight="bold">行动</text> <text x="640" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">Agent 编排</text> <text x="640" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">工具调用</text> <circle cx="760" cy="180" r="22" fill="url(#week6)" stroke="#9c7a08" stroke-width="1.5"></circle><text x="760" y="184" text-anchor="middle" fill="#fff" font-size="12" font-family="sans-serif" font-weight="bold">W6</text> <rect x="710" y="210" width="100" height="70" rx="6" fill="#fef9e7" stroke="#f1c40f" stroke-width="1"></rect><text x="760" y="232" text-anchor="middle" fill="#9c7a08" font-size="11" font-family="sans-serif" font-weight="bold">整合</text> <text x="760" y="250" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">完整多模态</text> <text x="760" y="266" text-anchor="middle" fill="#34495e" font-size="10" font-family="sans-serif">项目实战</text> <line x1="60" y1="180" x2="138" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><line x1="182" y1="180" x2="258" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><line x1="302" y1="180" x2="378" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><line x1="422" y1="180" x2="498" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><line x1="542" y1="180" x2="618" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><line x1="662" y1="180" x2="738" y2="180" stroke="#7f8c8d" stroke-width="1.5" marker-end="url(#arrow)"></line><text x="440" y="320" text-anchor="middle" fill="#2c3e50" font-size="14" font-family="sans-serif" font-weight="bold">推荐实战项目</text> <rect x="60" y="340" width="155" height="110" rx="6" fill="#ffffff" stroke="#3498db" stroke-width="1.5"></rect><text x="137" y="365" text-anchor="middle" fill="#2980b9" font-size="12" font-family="sans-serif" font-weight="bold">1. 图片问答</text> <text x="75" y="388" fill="#34495e" font-size="10" font-family="sans-serif">上传图片 + 问题</text> <text x="75" y="404" fill="#34495e" font-size="10" font-family="sans-serif">模型输出文字答案</text> <text x="75" y="425" fill="#7f8c8d" font-size="10" font-family="sans-serif" font-style="italic">覆盖：图像理解</text> <rect x="225" y="340" width="155" height="110" rx="6" fill="#ffffff" stroke="#1abc9c" stroke-width="1.5"></rect><text x="302" y="365" text-anchor="middle" fill="#16a085" font-size="12" font-family="sans-serif" font-weight="bold">2. AI OCR</text> <text x="240" y="388" fill="#34495e" font-size="10" font-family="sans-serif">截图 → 文字 + 结构化</text> <text x="240" y="404" fill="#34495e" font-size="10" font-family="sans-serif">自动整理到表格</text> <text x="240" y="425" fill="#7f8c8d" font-size="10" font-family="sans-serif" font-style="italic">覆盖：图像理解</text> <rect x="390" y="340" width="155" height="110" rx="6" fill="#ffffff" stroke="#9b59b6" stroke-width="1.5"></rect><text x="467" y="365" text-anchor="middle" fill="#8e44ad" font-size="12" font-family="sans-serif" font-weight="bold">3. 文档助手</text> <text x="405" y="388" fill="#34495e" font-size="10" font-family="sans-serif">PDF 上传 → 问答</text> <text x="405" y="404" fill="#34495e" font-size="10" font-family="sans-serif">RAG 检索增强</text> <text x="405" y="425" fill="#7f8c8d" font-size="10" font-family="sans-serif" font-style="italic">覆盖：RAG</text> <rect x="555" y="340" width="155" height="110" rx="6" fill="#ffffff" stroke="#e67e22" stroke-width="1.5"></rect><text x="632" y="365" text-anchor="middle" fill="#d35400" font-size="12" font-family="sans-serif" font-weight="bold">4. 视频总结</text> <text x="570" y="388" fill="#34495e" font-size="10" font-family="sans-serif">长视频 → 时间轴</text> <text x="570" y="404" fill="#34495e" font-size="10" font-family="sans-serif">关键片段问答</text> <text x="570" y="425" fill="#7f8c8d" font-size="10" font-family="sans-serif" font-style="italic">覆盖：视频理解</text> <rect x="720" y="340" width="155" height="110" rx="6" fill="#ffffff" stroke="#e74c3c" stroke-width="1.5"></rect><text x="797" y="365" text-anchor="middle" fill="#c0392b" font-size="12" font-family="sans-serif" font-weight="bold">5. AI 编程助手</text> <text x="735" y="388" fill="#34495e" font-size="10" font-family="sans-serif">截图报错 → 修复</text> <text x="735" y="404" fill="#34495e" font-size="10" font-family="sans-serif">调用工具写代码</text> <text x="735" y="425" fill="#7f8c8d" font-size="10" font-family="sans-serif" font-style="italic">覆盖：Agent</text></svg>

---

## 十、未来会发生什么？

回顾软件形态的演变，可以看到清晰的趋势：

> 过去：软件 = 功能。
> 
> 现在：软件 = 模型。
> 
> 未来：软件 = 多模态智能体。

这意味着未来用户只需说一句话，例如"做个网站"，模型就会自动完成设计、编码、部署、运营的全流程。

多模态并不是"给模型加一个看图功能"那么简单。

它真正改变的是：

> 让计算机开始直接理解现实世界。

这正是当前 AI 行业最重要的范式转变，也是所有开发者值得投入的方向。

---

## 常见问题

### 多模态模型和普通 LLM 哪个更好用？

取决于场景：纯文本写作、代码生成、翻译，普通 LLM 性价比更高；需要看图、听音频、理解视频时，多模态模型是唯一选择。

### 本地能跑多模态模型吗？

可以，但成本较高。小参数 VLM（如 Qwen2-VL-7B）可在 16GB 显存的消费级显卡上运行；更大模型通常需要云端 GPU 或调用 API。

### 多模态模型的输出一定是文字吗？

不一定。原生多模态模型可以输出文字、图像、音频甚至视频；纯 VLM 通常只输出文字。

### 如何选择多模态 API？

如果只做图像理解：OpenAI gpt-4o、Gemini 2.5 Flash、Qwen2-VL 都是性价比高的选择；如果要"看 + 听 + 实时语音"，建议直接使用 OpenAI Realtime API 或 Gemini Live。