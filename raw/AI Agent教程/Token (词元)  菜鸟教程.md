---
title: "Token (词元) | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/token-intro.html"
author:
published:
created: 2026-06-23
description:
tags:
  - "Token (词元)"
---
## Token (词元)

> 你有没有好奇过，当你向 Deepseek、豆包、千问、ChatGPT 或 Claude 提问时，AI 是怎么读懂你的文字的？
> 
> 答案的核心，就是一个叫做 Token，中文称之为 词元的概念。理解 Token，是走进 AI 世界的第一步。

### 什么是 Token (词元)？

```
Token (词元) = AI 能理解的最小文本单位
```

我们可以把它理解成：AI 世界里的文字碎片或者 AI 世界的燃料。

就像人类阅读时以词语为单位来理解意思，AI 也不会逐个字母地处理文字——它会先把文本切割成一块一块的碎片，每一块就叫做一个 Token（词元）。

<svg viewBox="0 0 460 280" width="100%" xmlns="http://www.w3.org/2000/svg"><defs><linearGradient id="cutGrad" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" stop-color="#ff8906" stop-opacity="0.3"></stop><stop offset="100%" stop-color="#0abfbc" stop-opacity="0.3"></stop></linearGradient></defs><rect width="460" height="280" rx="20" fill="#f5f3ee"></rect><text x="30" y="36" font-family="monospace" font-size="11" fill="#a7a9be" letter-spacing="2">INPUT TEXT</text> <rect x="30" y="52" width="400" height="52" rx="12" fill="#e0ddd6"></rect><text x="230" y="83" text-anchor="middle" font-family="monospace" font-size="16" fill="#555">"ChatGPT is amazing"</text> <line x1="230" y1="114" x2="230" y2="142" stroke="#c0bdb6" stroke-width="2" stroke-dasharray="4 3"></line><polygon points="230,148 224,138 236,138" fill="#c0bdb6"></polygon><text x="30" y="172" font-family="monospace" font-size="11" fill="#a7a9be" letter-spacing="2">TOKENS</text> <rect x="30" y="186" width="66" height="36" rx="9" fill="rgba(255,137,6,0.18)" stroke="rgba(255,137,6,0.5)" stroke-width="1.5"></rect><text x="63" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#cc6d00">Chat</text> <rect x="106" y="186" width="40" height="36" rx="9" fill="rgba(10,191,188,0.15)" stroke="rgba(10,191,188,0.5)" stroke-width="1.5"></rect><text x="126" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#07908e">G</text> <rect x="156" y="186" width="40" height="36" rx="9" fill="rgba(242,95,76,0.15)" stroke="rgba(242,95,76,0.5)" stroke-width="1.5"></rect><text x="176" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#c03020">PT</text> <rect x="206" y="186" width="30" height="36" rx="9" fill="rgba(167,120,218,0.15)" stroke="rgba(167,120,218,0.5)" stroke-width="1.5"></rect><text x="221" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#8060b0">is</text> <rect x="246" y="186" width="80" height="36" rx="9" fill="rgba(80,200,120,0.15)" stroke="rgba(80,200,120,0.5)" stroke-width="1.5"></rect><text x="286" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#30904f">amaz</text> <rect x="336" y="186" width="60" height="36" rx="9" fill="rgba(255,180,50,0.18)" stroke="rgba(255,180,50,0.5)" stroke-width="1.5"></rect><text x="366" y="209" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#a07010">ing</text> <text x="30" y="258" font-family="monospace" font-size="13" fill="#888">共 <tspan fill="#ff8906" font-size="18" font-weight="700">6</tspan> 个 Token</text></svg>

**一个好用的比喻：**

把一篇文章想象成一块比萨，你不会整块吞下去，也不会把它碾成粉末——而是切成一块一块，每块就是一个 Token。AI 就是这样吃进文字的：切块、理解、消化。

Token 比单词更细，比字母更粗，是一种灵活的中间单位。

---

## Token 长什么样？

不同的文本，切分方式大相径庭。

常见词是一个 Token，复杂词会被拆开，标点也算 Token。

英文句子

"Hello, world!"

Hello →, → world →!

共 4 个 Token

复杂英文单词

"unbelievable"

un believ able

共 3 个 Token —— 长词会被拆分

中文句子

"你好，世界！"

你好 ， 世界 ！

共 4 个 Token —— 中文每字约消耗更多 Token

---

## Token 和 字、 词 有什么区别？

很多人会以为 Token = 单词，但其实 **并不完全是** 。

Token 的划分遵循一套特殊的算法（比如 BPE，字节对编码），结果有时候很直觉，有时候却出人意料：

| 文本 | Token 数量 | 说明 |
| --- | --- | --- |
| `cat` | 1 | 常见词，直接一个 Token |
| `unbelievable` | 4 | `un` + `believ` + `able` + … |
| `ChatGPT` | 3 | `Chat` + `G` + `PT` |
| `你好` | 约 2～3 | 中文通常比英文消耗更多 Token |

> **结论：** Token 比单词更细，比字母更粗，是一种灵活的中间单位。

粒度

单个字母或汉字

示例

H, e, l, l, o

AI 是否使用

❌ 太细，信息密度低

类比

把比萨碾成粉末

粒度

词根、短词、字符组合

示例

Hello / Chat·G·PT / un·believ·able

AI 是否使用

✅ 高效，平衡信息密度

类比

把比萨切成合适的一口大小

粒度

完整单词或句子

示例

Hello, unbelievable

AI 是否使用

❌ 太粗，处理不灵活

类比

整块比萨一口塞进去

---

## Token 为什么重要？

### 1\. 它决定了 AI 能读多少

每个 AI 模型都有一个 **上下文窗口（Context Window）** ，也就是它一次能处理的最大 Token 数量。比如：

- GPT-3.5：约 4,000 Token
- GPT-4 Turbo：约 128,000 Token
- Claude 3.5：约 200,000 Token

超过这个限制，AI 就会记不住之前说的话，就像一个人的短期记忆装满了一样。

### 2\. 它影响 API 的使用费用

调用 AI 接口（API）时，费用通常按 Token 数量计费。 **输入的文字 + AI 回复的文字** ，都会消耗 Token，所以写得越长，花费越多。

### 3\. 它影响 AI 的回复速度

AI 生成文字时，是 **一个 Token 一个 Token 地输出** 的，所以回复越长，等待时间越久。

---

## Token 的小知识

- **1,000 个 Token ≈ 750 个英文单词** ，是一个常用的估算比例。
- **中文、日文、韩文** 等亚洲语言通常比英文消耗更多 Token，因为这些字符在编码上更复杂。
- **空格和标点** 也算 Token！不要以为只有文字才计数。
- **代码** 的 Token 消耗一般比自然语言少，因为编程语言结构紧凑。

---

## 总结

| 概念 | 一句话解释 |
| --- | --- |
| Token | AI 处理文本的最小单位，介于字符和单词之间 |
| 上下文窗口 | AI 一次能处理的最大 Token 数 |
| Token 计费 | API 调用按输入+输出的 Token 总量收费 |
| Token 生成 | AI 回复是逐 Token 产生的，越长越慢 |