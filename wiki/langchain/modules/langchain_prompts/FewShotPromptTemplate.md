---
tags: [langchain, prompts, FewShotPromptTemplate, few-shot, completion]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# FewShotPromptTemplate

`FewShotPromptTemplate` 用于为 **text completion 模型**（如老式 `OpenAI("text-davinci-003")`）创建 few-shot 提示词。输出是**纯字符串**，通过 `prefix + 格式化示例 + suffix` 拼接而成。

> 当前主流 LLM 都是 chat 模型，实际开发中几乎都用 [[langchain/modules/langchain_prompts/FewShotChatMessagePromptTemplate|FewShotChatMessagePromptTemplate]]。`FewShotPromptTemplate` 仅在需要兼容老式 completion 接口时使用。

## 构造函数参数

| 参数 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `examples` | list[dict] | 二选一 | 静态示例列表 |
| `example_selector` | BaseExampleSelector | 二选一 | 动态示例选择器 |
| `example_prompt` | PromptTemplate | 必填 | 格式化每个示例的模板（注意是 `PromptTemplate`，不是 `ChatPromptTemplate`） |
| `prefix` | str | 必填 | 示例前的文本 |
| `suffix` | str | 必填 | 示例后的文本，通常包含输入变量 |
| `input_variables` | list[str] | 必填 | 运行时需提供的变量名 |
| `example_separator` | str | 可选 | 示例之间的分隔符，默认 `"\n\n"` |

## 基础用法

```python
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

# 定义示例
examples = [
    {"word": "happy", "antonym": "sad"},
    {"word": "tall", "antonym": "short"},
    {"word": "fast", "antonym": "slow"},
]

# 每个示例的格式化模板（必须是 PromptTemplate）
example_prompt = PromptTemplate(
    template="Input: {word}\nOutput: {antonym}",
    input_variables=["word", "antonym"],
)

# 创建 few-shot 模板
few_shot = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="给出每个输入的反义词：",
    suffix="Input: {adjective}\nOutput: ",
    input_variables=["adjective"],
    example_separator="\n",  # 默认 "\n\n"
)

# 输出是纯字符串
result = few_shot.format(adjective="big")
print(result)
# 给出每个输入的反义词：
# Input: happy
# Output: sad
# Input: tall
# Output: short
# Input: fast
# Output: slow
# Input: big
# Output: 
```

## 与 LLM 链式调用

```python
from langchain_openai import OpenAI  # 注意是 completion 模型
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

# 定义示例
examples = [
    {"word": "happy", "antonym": "sad"},
    {"word": "tall", "antonym": "short"},
    {"word": "fast", "antonym": "slow"},
]

example_prompt = PromptTemplate(
    template="Input: {word}\nOutput: {antonym}",
    input_variables=["word", "antonym"],
)

few_shot = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="给出每个输入的反义词：",
    suffix="Input: {adjective}\nOutput: ",
    input_variables=["adjective"],
)

# 与 completion 模型链式调用
llm = OpenAI(model="gpt-3.5-turbo-instruct")
chain = few_shot | llm

result = chain.invoke({"adjective": "big"})
print(result)  # "small"
```

## 动态示例选择

```python
from langchain_core.example_selectors import LengthBasedExampleSelector
from langchain_core.prompts import PromptTemplate, FewShotPromptTemplate

examples = [
    {"word": "happy", "antonym": "sad"},
    {"word": "tall", "antonym": "short"},
    {"word": "fast", "antonym": "slow"},
    {"word": "brilliant", "antonym": "dull"},
    {"word": "ancient", "antonym": "modern"},
]

example_prompt = PromptTemplate(
    template="Input: {word}\nOutput: {antonym}",
    input_variables=["word", "antonym"],
)

# 按长度选择示例，确保总长度不超过 max_length
example_selector = LengthBasedExampleSelector(
    examples=examples,
    example_prompt=example_prompt,
    max_length=50,  # 按字符数限制
)

few_shot = FewShotPromptTemplate(
    example_selector=example_selector,
    example_prompt=example_prompt,
    prefix="给出每个输入的反义词：",
    suffix="Input: {adjective}\nOutput: ",
    input_variables=["adjective"][[,
)

# 短输入会选更多示例，长输入会选更少示例
result = few_shot.format(adjective="big")
```

## 相关页面

- [[langchain/modules/langchain_prompts/概述|prompts 模块概述]]
- [[langchain/modules/langchain_prompts/FewShotChatMessagePromptTemplate|FewShotChatMessagePromptTemplate]] — 消息版 few-shot（推荐）
- [[langchain/modules/langchain_prompts/PromptTemplate|PromptTemplate]] — example_prompt 必须使用此类
- [[langchain/modules/langchain_prompts/from_template方法|from_template 方法]]
