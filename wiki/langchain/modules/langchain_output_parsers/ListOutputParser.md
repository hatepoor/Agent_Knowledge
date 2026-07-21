---
tags: [langchain, output-parsers, list]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# ListOutputParser

## 是什么

`ListOutputParser` 系列将 LLM 的输出解析为 **Python 列表** (`list[str]`)。它提供了多种子类型，支持逗号分隔、编号列表等不同格式。

## 类层级

```
BaseOutputParser
  └── ListOutputParser            # 基类，支持自定义 separator
        ├── CommaSeparatedListOutputParser   # 逗号分隔
        └── NumberedListOutputParser         # 编号列表（1. 2. 3.）
```

## 导入

```python
from langchain_core.output_parsers import (
    ListOutputParser,
    CommaSeparatedListOutputParser,
)
from langchain.output_parsers import NumberedListOutputParser
```

---

## 基类：ListOutputParser

`ListOutputParser` 是所有列表解析器的基类，支持自定义分隔符。

### 构造函数

```python
ListOutputParser(separator: str = ",")
```

- `separator`：用于分割 LLM 输出文本的分隔字符串，默认为逗号 `","`

### 自定义分隔符示例

```python
from langchain_core.output_parsers import ListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# 使用换行符作为分隔符
newline_parser = ListOutputParser(separator="\n")

prompt = ChatPromptTemplate.from_messages([
    ("system", "列出5个项目，每行一个，不要编号。"),
    ("human", "列出5种{category}。"),
])

chain = prompt | model | newline_parser
items = chain.invoke({"category": "水果"})
print(items)
# ['苹果', '香蕉', '橙子', '葡萄', '西瓜']
```

### get_format_instructions()

```python
from langchain_core.output_parsers import ListOutputParser

parser = ListOutputParser(separator=",")
print(parser.get_format_instructions())
# Your response should be a list of comma separated values, eg: `foo, bar, baz`

parser2 = ListOutputParser(separator="\n")
print(parser2.get_format_instructions())
# Your response should be a list of  separated values, eg: `foo\n bar\n baz`
```

---

## CommaSeparatedListOutputParser

最常用的列表解析器，专门解析 **逗号分隔** 的输出。

### 完整链示例

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = CommaSeparatedListOutputParser()

# 将 format_instructions 注入到提示词中，引导 LLM 输出正确格式
format_instructions = parser.get_format_instructions()

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。{format_instructions}"),
    ("human", "列出5种{category}。"),
])

chain = prompt | model | parser

items = chain.invoke({
    "category": "编程语言",
    "format_instructions": format_instructions,
})

print(items)
# ['Python', 'JavaScript', 'Go', 'Rust', 'Java']
print(type(items))      # <class 'list'>
print(type(items[0]))   # <class 'str'>
```

### get_format_instructions() 输出

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

parser = CommaSeparatedListOutputParser()
print(parser.get_format_instructions())
# Your response should be a list of comma separated values, eg: `foo, bar, baz`
```

### 流式输出

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = CommaSeparatedListOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "列出项目，逗号分隔。"),
    ("human", "列出{category}。"),
])

chain = prompt | model | parser

# 流式输出时，每个 chunk 是一个部分列表
for chunk in chain.stream({"category": "颜色"}):
    print(chunk)
# ['红']
# ['红', '蓝']
# ['红', '蓝', '绿']
```

---

## NumberedListOutputParser

解析 **编号列表** 格式的 LLM 输出，例如 `"1. item\n2. item\n3. item"`。

### 完整链示例

```python
from langchain.output_parsers import NumberedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = NumberedListOutputParser()

format_instructions = parser.get_format_instructions()

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。{format_instructions}"),
    ("human", "给出学习{subject}的5个建议。"),
])

chain = prompt | model | parser

items = chain.invoke({
    "subject": "Python编程",
    "format_instructions": format_instructions,
})

print(items)
# ['先学基础语法', '多做练习题', '阅读优秀源码', '参与开源项目', '写博客总结']
```

### get_format_instructions() 输出

```python
from langchain.output_parsers import NumberedListOutputParser

parser = NumberedListOutputParser()
print(parser.get_format_instructions())
# Your response should be a numbered list of items, eg: `1. foo\n2. bar\n3. baz`
```

### 内部解析逻辑

`NumberedListOutputParser` 会按行分割文本，然后去掉行首的编号前缀：

```python
from langchain.output_parsers import NumberedListOutputParser

parser = NumberedListOutputParser()

# 模拟 LLM 输出
raw_text = """1. 学习Python基础
2. 练习数据结构
3. 学习Web框架
4. 做项目实战
5. 阅读官方文档"""

result = parser.parse(raw_text)
print(result)
# ['学习Python基础', '练习数据结构', '学习Web框架', '做项目实战', '阅读官方文档']
```

它会自动处理以下格式变体：

- `1. item` （点号后有空格）
- `1) item` （右括号分隔）
- `1、item` （中文顿号）

---

## 错误处理：LLM 输出格式不正确时

### 常见错误场景

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

parser = CommaSeparatedListOutputParser()

# 场景1：LLM 输出了多余的解释文字
bad_output = "好的，以下是5种编程语言：Python, JavaScript, Go, Rust, Java"
result = parser.parse(bad_output)
print(result)
# ['好的，以下是5种编程语言：Python', 'JavaScript', 'Go', 'Rust', 'Java']
# ⚠️ 第一个元素包含了多余的中文前缀！

# 场景2：LLM 使用了中文逗号
bad_output2 = "Python，JavaScript，Go，Rust，Java"
result2 = parser.parse(bad_output2)
print(result2)
# ['Python，JavaScript，Go，Rust，Java']
# ⚠️ 英文逗号解析器不识别中文逗号，整个字符串变成了一个元素！
```

### 防御策略

**策略1：在提示词中明确格式要求**

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = CommaSeparatedListOutputParser()

# ✅ 在系统消息中明确格式，并要求"只输出列表，不要任何说明"
prompt = ChatPromptTemplate.from_messages([
    ("system", "只输出逗号分隔的列表，不要任何说明文字。不要使用中文逗号。\n{format_instructions}"),
    ("human", "列出5种{category}。"),
])

chain = prompt | model | parser
items = chain.invoke({
    "category": "编程语言",
    "format_instructions": parser.get_format_instructions(),
})
```

**策略2：后处理清洗**

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = CommaSeparatedListOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "列出项目，逗号分隔，不要说明文字。"),
    ("human", "列出5种{category}。"),
])

chain = prompt | model | parser
raw_items = chain.invoke({"category": "水果"})

# 后处理：去除空白、过滤空字符串、处理中文逗号
cleaned_items = [][[
for item in raw_items:
    # 处理中文逗号
    item = item.replace("，", ",")
    # 分割后再添加
    for sub in item.split(","):
        cleaned = sub.strip()
        if cleaned:
            cleaned_items.append(cleaned)

print(cleaned_items)
```

**策略3：使用 OutputFixingParser 自动修正**

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain.output_parsers import OutputFixingParser
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
base_parser = CommaSeparatedListOutputParser()
fixing_parser = OutputFixingParser.from_llm(llm=model, parser=base_parser)

# 当解析失败时，会自动调用 LLM 重新生成正确格式
```

参见：[[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser]]

---

## 实用技巧

### 批量调用

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
parser = CommaSeparatedListOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "逗号分隔列出项目。{format_instructions}"),
    ("human", "列出3种{category}。"),
])

chain = prompt | model | parser

results = chain.batch([
    {"category": "水果", "format_instructions": parser.get_format_instructions()},
    {"category": "颜色", "format_instructions": parser.get_format_instructions()},
    {"category": "乐器", "format_instructions": parser.get_format_instructions()},
])

for r in results:
    print(r)
```

### 选择合适的列表解析器

| 解析器 | 适用场景 | 示例 LLM 输出 |
|--------|----------|----------------|
| `CommaSeparatedListOutputParser` | 简短项目、标签、关键词 | `A, B, C` |
| `NumberedListOutputParser` | 步骤、排名、有序列表 | `1. A\n2. B\n3. C` |
| `ListOutputParser(separator="\n")` | 长文本项、多行描述 | `A\nB\nC` |

### 何时该用 with_structured_output 代替

如果你需要的是**结构化数据**（如带有类型约束的列表），考虑使用更现代的 `with_structured_output`：

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class TagList(BaseModel):
    tags: list[str] = Field(description="标签列表")

model = ChatOpenAI(model="gpt-4o-mini").with_structured_output(TagList)
result = model.invoke("给'人工智能'列出5个相关标签")
print(result.tags)  # ['机器学习', '深度学习', '神经网络', 'NLP', '计算机视觉'][[
```

参见：[[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output]]

---

## 相关

- [[langchain/modules/langchain_output_parsers/概述|← 返回 output_parsers 概述]]
- [[langchain/modules/langchain_output_parsers/StrOutputParser|StrOutputParser - 纯文本解析]]
- [[langchain/modules/langchain_output_parsers/EnumOutputParser|EnumOutputParser - 离散选择]]
- [[langchain/modules/langchain_output_parsers/OutputFixingParser|OutputFixingParser - 自动修正]]
- [[langchain/modules/langchain_output_parsers/with_structured_output|with_structured_output - 现代替代方案]]
- [[langchain/modules/langchain_output_parsers/错误处理|错误处理]]
