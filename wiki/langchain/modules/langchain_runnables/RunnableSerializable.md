---
tags: [langchain, module, runnables, serialization, RunnableSerializable]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# RunnableSerializable

> `RunnableSerializable` 是 LangChain 中的关键基类，将**可执行**（Runnable）和**可序列化**（Serializable）两种能力合为一体。几乎所有 LangChain 组件都继承自它。

## 是什么

`RunnableSerializable` 位于 `langchain_core.runnables.base`，同时继承了 `Runnable` 和 `Serializable` 两个基类：

- **Runnable** — 提供 `invoke()` / `stream()` / `batch()` 等执行接口
- **Serializable** — 提供 `to_json()` 方法，支持序列化为 JSON/dict

这意味着你构建的 Prompt、Model、Parser、Chain 等不仅能执行，还能**存成 JSON 再加载回来**。

## 类继承关系

```
Serializable          # 可序列化（to_json, to_json_not_implemented）
    ↑
Runnable              # 可执行（invoke, stream, batch）
    ↑
RunnableSerializable  # = Runnable + Serializable（大多数组件的基类）
    ↑
├── BaseChatModel     # ChatOpenAI, ChatAnthropic 等
├── BasePromptTemplate # PromptTemplate, ChatPromptTemplate 等
├── BaseOutputParser  # StrOutputParser, PydanticOutputParser 等
├── BaseTool          # @tool 装饰器创建的工具
├── BaseRetriever     # 各种检索器
├── RunnableSequence  # 用 | 创建的链
├── RunnableLambda    # 自定义函数包装
└── ...
```

## 序列化 API

序列化功能由 `langchain_core.load` 模块提供，有四个核心函数：

| 函数 | 方向 | 输入 → 输出 |
|------|------|------------|
| `dumps(obj)` | 序列化 | 对象 → JSON 字符串 |
| `loads(text)` | 反序列化 | JSON 字符串 → 对象 |
| `dumpd(obj)` | 序列化 | 对象 → Python dict |
| `load(dct)` | 反序列化 | Python dict → 对象 |

### 基础用法

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.load import dumps, loads, dumpd, load

# 构建一个链
prompt = ChatPromptTemplate.from_template("翻译成英文：{text}")
model = ChatOpenAI(model="gpt-4o-mini")
chain = prompt | model | StrOutputParser()

# ========== 序列化 ==========

# 1. dumps() → JSON 字符串
json_str = dumps(chain, pretty=True)
print(json_str)
# {
#   "lc": 1,
#   "type": "constructor",
#   "id": ["langchain", "schema", "runnable", "RunnableSequence"],
#   "kwargs": {
#     "first": { ... prompt template ... },
#     "middle": [ ... ],
#     "last": { ... output parser ... }
#   }
# }

# 2. dumpd() → Python dict
chain_dict = dumpd(chain)
print(chain_dict["type"])  # "constructor"

# ========== 反序列化 ==========

# 3. loads() → 从 JSON 字符串还原
restored = loads(json_str)

# 4. load() → 从 dict 还原
restored2 = load(chain_dict)

# 还原后的链和原链功能相同
result = restored.invoke({"text": "你好"})
```

### 序列化格式

LangChain 的序列化 JSON 有固定结构：

```json
{
  "lc": 1,
  "type": "constructor",
  "id": ["langchain_core", "prompts", "chat", "ChatPromptTemplate"],
  "kwargs": {
    "messages": [ ... ],
    "input_variables": ["text"]
  }
}
```

| 字段 | 含义 |
|------|------|
| `lc` | 序列化协议版本（固定为 1） |
| `type` | 对象类型（`"constructor"` 表示可重建的类） |
| `id` | 类的完整路径（模块 + 类名） |
| `kwargs` | 构造函数参数 |

### to_json() 实例方法

任何 `RunnableSerializable` 对象都可以直接调用 `to_json()`：

```python
from langchain_core.prompts import PromptTemplate

template = PromptTemplate.from_template("你好，{name}")

# 实例方法
json_data = template.to_json()
print(json_data)
# {
#   "lc": 1,
#   "type": "constructor",
#   "id": ["langchain", "prompts", "prompt", "PromptTemplate"],
#   "kwargs": {
#     "template": "你好，{name}",
#     "input_variables": ["name"],
#     ...
#   }
# }
```

## 实际用途

| 场景 | 说明 |
|------|------|
| 持久化存储 | 把链保存到文件或数据库，下次加载直接用 |
| 跨进程传输 | 把链的 JSON 发给另一个进程/服务去执行 |
| LangSmith 追踪 | LangSmith 用序列化记录链的结构用于可观测性 |
| 配置管理 | 把不同版本的链存为 JSON 配置，按场景加载 |
| LangChain Hub | 分享和加载社区模板 |

### 保存到文件

```python
from langchain_core.load import dumps, loads

# 保存
json_str = dumps(chain, pretty=True)
with open("chain.json", "w", encoding="utf-8") as f:
    f.write(json_str)

# 加载
with open("chain.json", "r", encoding="utf-8") as f:
    restored = loads(f.read())

result = restored.invoke({"text": "你好"})
```

### 保存单个组件

```python
from langchain_core.load import dumps, loads
from langchain_core.prompts import ChatPromptTemplate

# 保存 prompt 模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家"),
    ("human", "{question}"),
])

# 序列化
prompt_json = dumps(prompt)

# 反序列化
restored_prompt = loads(prompt_json)
messages = restored_prompt.format_messages(role="物理", question="什么是量子？")
```

## 安全注意事项

`loads()` 会**实际实例化对象**（包括执行构造函数参数），因此：

### 不要对不可信输入调用 loads()

```python
# ❌ 危险：恶意 JSON 可能实例化带有害 base_url 的模型
obj = loads(untrusted_json)
# 攻击者可构造 base_url 指向恶意服务器 → SSRF 攻击
```

### 使用 allowed_objects 限制反序列化范围

```python
# ✅ 安全：只允许消息类
safe_obj = loads(untrusted_json, allowed_objects='messages')

# ✅ 较安全：只允许 langchain-core 中的类
obj = loads(json_str, allowed_objects='core')

# ❌ 不安全：允许所有类（包括第三方集成包）
obj = loads(json_str, allowed_objects='all')
```

### allowed_objects 选项

| 值 | 说明 | 安全性 |
|---|------|--------|
| `'messages'` | 仅消息类（HumanMessage 等） | 安全，适合不可信输入 |
| `'core'`（默认） | langchain-core 中的类 | 中等 |
| `'all'` | 所有映射中的类（含第三方） | 不安全 |
| 显式列表 | 只允许指定的类 | 最安全 |

## 常见陷阱

### 陷阱 1：lambda 函数无法序列化

```python
from langchain_core.runnables import RunnableLambda

# ❌ lambda 无法序列化
step = RunnableLambda(lambda x: x.upper())
json_str = dumps(step)  # 报错或丢失函数体

# ✅ 使用命名函数
def to_upper(x):
    return x.upper()

step = RunnableLambda(to_upper)
json_str = dumps(step)  # 可以序列化（引用函数路径）
```

### 陷阱 2：API Key 序列化行为

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini", api_key="sk-...")

# API Key 默认不会被序列化（安全设计）
json_str = dumps(model)
# json 中 api_key 字段为 SecretStr 占位符，不是明文

# 反序列化时需要通过 secrets_map 或环境变量重新提供
restored = loads(json_str, secrets_map={"OPENAI_API_KEY": "sk-..."})
# 或
import os
os.environ["OPENAI_API_KEY"][[ = "sk-..."
restored = loads(json_str, secrets_from_env=True)
```

### 陷阱 3：自定义类需要注册映射

```python
# 自定义的 Runnable 需要被 LangChain 序列化映射识别
# 否则 loads() 无法还原

# ✅ 确保类在 langchain_core.load.mapping 中有注册
# 或在 loads 时提供 additional_import_mappings
restored = loads(
    json_str,
    additional_import_mappings={
        ("my_package", "my_module"): ("my_package", "my_module")
    }
)
```

## 相关页面

- [[langchain/modules/langchain_runnables/概述|← 返回 runnables 模块]]
- [[langchain/modules/langchain_runnables/LCEL链式组合|LCEL 链式组合]]
- [[langchain/modules/langchain_core/概述|langchain-core 概述]]
