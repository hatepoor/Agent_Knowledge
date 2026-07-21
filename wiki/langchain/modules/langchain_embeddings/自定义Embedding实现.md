---
tags: [langchain, embeddings, custom, 自定义]
type: concept
created: 2026-07-13
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# 自定义 Embedding 实现

## 是什么

当使用 LangChain 未内置集成的第三方 embedding 服务时，需要自己实现 `Embeddings` 接口，将第三方 SDK 包装成 LangChain 兼容的嵌入模型。

## 接口规范

继承 `langchain_core.embeddings.Embeddings` 基类，只需实现 2 个必选方法：

| 方法 | 输入 | 输出 | 必选 |
|------|------|------|------|
| `embed_documents(texts)` | `list[str]` | `list[list[float]]` | ✅ |
| `embed_query(text)` | `str` | `list[float]` | ✅ |
| `aembed_documents(texts)` | `list[str]` | `list[list[float]]` | 可选 |
| `aembed_query(text)` | `str` | `list[float]` | 可选 |

> **注意**：`Embeddings` **不实现** `Runnable` 接口，也**不是** pydantic `BaseModel`。如果你的类需要参数校验，需额外继承 `BaseModel`（见下方"模式A"）。

## 两种实现模式

### 模式 A：BaseModel + Embeddings（带参数校验）

适用于需要 API Key、model 名称等配置参数管理的场景。

```python
from typing import Any, Dict, List, Optional
from langchain_core.embeddings import Embeddings
from langchain_core.pydantic_v1 import BaseModel, Field, root_validator
from langchain_core.utils import get_from_dict_or_env


class MyCustomEmbeddings(BaseModel, Embeddings):
    """自定义嵌入模型（带参数校验）"""

    # 配置参数（BaseModel 提供校验）
    model: str = Field(default="default-model")
    api_key: str = ""
    client: Any = Field(default=None, exclude=True)  # SDK 实例，不暴露给序列化

    @root_validator(pre=True)
    def validate_environment(cls, values: Dict) -> Dict:
        """初始化时校验环境并创建 SDK 客户端"""
        # 从环境变量读取 API Key（备选方案）
        values["api_key"] = get_from_dict_or_env(
            values, "api_key", "MY_API_KEY_ENV_VAR"
        )
        # 创建 SDK 客户端
        values["client"] = create_sdk_client(values["api_key"])
        return values

    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        """嵌入文档列表 — 批量调用 SDK"""
        response = self.client.embeddings.create(
            model=self.model,
            input=texts,
        )
        # 从 SDK 响应中提取向量列表
        return [item.embedding for item in response.data]

    def embed_query(self, text: str) -> List[float]:
        """嵌入单个查询 — 委托给 embed_documents"""
        return self.embed_documents([text])[0]
```

### 模式 B：纯 Embeddings（轻量封装）

适用于简单场景，参数直接通过 `__init__` 传入。

```python
from typing import List
from langchain_core.embeddings import Embeddings


class LightweightEmbeddings(Embeddings):
    """轻量嵌入模型"""

    def __init__(self, model: str = "default", api_key: str = ""):
        self.model = model
        self.client = create_sdk_client(api_key)

    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        response = self.client.embeddings.create(model=self.model, input=texts)
        return [item.embedding for item in response.data]

    def embed_query(self, text: str) -> List[float]:
        return self.embed_documents([text])[0]
```

## 以智谱 AI zai-sdk 为例

```python
# pip install zai-sdk==0.2.3
from typing import Any, Dict, List, Optional
from langchain_core.embeddings import Embeddings
from langchain_core.pydantic_v1 import BaseModel, Field, root_validator
from langchain_core.utils import get_from_dict_or_env
from zai import ZhipuAiClient


class ZhipuAIEmbeddings(BaseModel, Embeddings):
    """智谱 AI 嵌入模型（基于 zai-sdk）

    使用示例:
        embeddings = ZhipuAIEmbeddings(
            model="embedding-3",
            api_key="xxx",
            dimensions=1024,
        )
        vec = embeddings.embed_query("你好世界")
        vecs = embeddings.embed_documents(["文本1", "文本2"])
    """

    model: str = Field(default="embedding-3")
    """模型名称: embedding-3（推荐）或 embedding-2"""

    api_key: str = ""
    """API Key，默认从环境变量 ZAI_API_KEY 读取"""

    dimensions: Optional[int] = None
    """输出维度，仅 embedding-3 支持: 2048/1024/512/256"""

    client: Any = Field(default=None, exclude=True)

    @root_validator(pre=True)
    def validate_environment(cls, values: Dict) -> Dict:
        values["api_key"] = get_from_dict_or_env(
            values, "api_key", "ZAI_API_KEY"
        )
        values["client"] = ZhipuAiClient(api_key=values["api_key"])
        return values

    def embed_documents(self, texts: List[str]) -> List[List[float]]:
        kwargs = {"model": self.model, "input": texts}
        if self.dimensions is not None:
            kwargs["dimensions"] = self.dimensions

        response = self.client.embeddings.create(**kwargs)
        return [item.embedding for item in response.data]

    def embed_query(self, text: str) -> List[float]:
        return self.embed_documents([text])[0]
```

## 在 VectorStore 中使用

```python
from langchain_community.vectorstores import FAISS

embeddings = ZhipuAIEmbeddings(
    model="embedding-3",
    api_key="xxx",
    dimensions=1024,
)

# 直接传给 VectorStore，自动调用 embed_documents
vectorstore = FAISS.from_documents(documents, embeddings)

# 检索时自动调用 embed_query
results = vectorstore.similarity_search("查询文本", k=5)
```

## 常见错误

| 错误 | 原因 | 修正 |
|------|------|------|
| `'MyEmbeddings' object has no attribute 'embed_documents'` | 方法名拼写错误或未实现 | 必须实现 `embed_documents` 和 `embed_query` |
| `TypeError: dict is not a list[float]` | SDK 返回值提取错误 | 检查响应结构，如 `response.data[0].embedding` 而非 `response.data` |
| `AttributeError: 'list' object has no attribute 'embedding'` | 搞混了单个/批量返回 | `response.data` 是列表，每个元素有 `.embedding` 属性 |
| 异步方法报 `NotImplementedError` | 未实现异步却掉了基类默认实现 | 要么实现 `aembed_*`，要么不 override，基类会用线程池跑同步版本 |

## 子页面

- [[langchain/modules/langchain_embeddings/智谱AI_Embedding|智谱AI Embedding]] — 智谱 AI 完整集成（两种方案）
- [[langchain/modules/langchain_embeddings/OpenAIEmbeddings|OpenAIEmbeddings]]
- [[langchain/modules/langchain_embeddings/HuggingFaceEmbeddings|HuggingFaceEmbeddings]]

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embedding 模块总览]]
- [LangChain 官方：自定义嵌入](http://python.langchain.ac.cn/docs/how_to/custom_embeddings/)
- [LangChain 官方：嵌入模型贡献指南](https://docs.langchain.com/oss/python/contributing/implement-langchain)
