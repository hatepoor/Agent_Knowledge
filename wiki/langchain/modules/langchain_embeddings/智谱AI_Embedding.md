---
tags: [langchain, embeddings, zhipuai, 智谱]
type: concept
created: 2026-07-13
updated: 2026-07-13
langchain_version: ">=0.3.0"
zai_sdk_version: "0.2.3"
---

# 智谱AI Embedding

## 概览

智谱AI 提供两代嵌入模型，当前推荐使用第三代 `embedding-3`。

| 模型 | 维度 | 最大 Token/条 | 最大批次 | 价格 |
|------|------|---------------|----------|------|
| `embedding-3` | 2048/1024/512/256（可自定义） | 3072 | 64 条 | 0.5 元/百万 Token |
| `embedding-2` | 1024（固定） | 512 | 不限（总量 ≤ 8K Token） | 0.5 元/百万 Token |

## 集成方式：zai-sdk 自定义封装

> **不使用 `langchain_community`**。直接继承 `langchain_core.embeddings.Embeddings`，用 zai-sdk 原生 API 实现，保持最新 SDK 支持和完全可控。

### 安装

```bash
pip install zai-sdk==0.2.3
```

### zai-sdk 原生 Embedding API

先了解 zai-sdk 的原始调用方式，才能正确封装：

```python
from zai import ZhipuAiClient

client = ZhipuAiClient(api_key="your-api-key")

# embedding-3，默认 2048 维
response = client.embeddings.create(
    model="embedding-3",
    input=["文本1", "文本2", "文本3"],
)

# 响应结构: response.data 是列表，每项有 .embedding 和 .index
for item in response.data:
    print(f"索引{item.index}: 维度{len(item.embedding)}, 前3值{item.embedding[:3]}")

# 自定义维度
response = client.embeddings.create(
    model="embedding-3",
    input=["文本"],
    dimensions=512,  # 可选: 256/512/1024/2048
)
```

### 封装为 LangChain Embeddings

```python
from typing import Any, Dict, List, Optional
from langchain_core.embeddings import Embeddings
from langchain_core.pydantic_v1 import BaseModel, Field, root_validator
from langchain_core.utils import get_from_dict_or_env
from zai import ZhipuAiClient


class ZhipuAIEmbeddings(BaseModel, Embeddings):
    """智谱 AI 嵌入模型 — 基于 zai-sdk v0.2.3

    使用:
        embeddings = ZhipuAIEmbeddings(
            model="embedding-3",
            api_key="xxx",
            dimensions=512,
        )
        vec = embeddings.embed_query("查询文本")
    """

    model: str = Field(default="embedding-3")
    """模型: embedding-3（推荐）或 embedding-2"""

    api_key: str = ""
    """API Key，默认读取环境变量 ZAI_API_KEY"""

    dimensions: Optional[int] = None
    """仅 embedding-3 支持: 2048/1024/512/256"""

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

### 在 Agent / RAG 中使用

```python
# 1. 直接用于 VectorStore
from langchain_community.vectorstores import FAISS

embeddings = ZhipuAIEmbeddings(
    model="embedding-3",
    api_key="your-key",
    dimensions=1024,
)
vectorstore = FAISS.from_documents(documents, embeddings)

# 2. 配合 CacheBackedEmbeddings 节省成本
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

cached = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=embeddings,
    document_embedding_cache=LocalFileStore("./.cache/embeddings"),
)

# 3. 用于 RAG Agent 的检索器
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
```

---

## 维度选择指南

| 维度 | 精度 | 存储成本 | 检索速度 | 推荐场景 |
|------|------|----------|----------|----------|
| 2048 | 最高 | 高 | 慢 | 法律文书、学术论文、高精度 Q&A |
| 1024 | 高 | 中 | 中 | **通用推荐**：大多数场景的首选 |
| 512 | 中 | 低 | 快 | 大规模语料库、粗粒度分类 |
| 256 | 较低 | 最低 | 最快 | 实时搜索、初筛、去重 |

> 向量存储成本公式：`文件数 × 维度 × 4 字节`。100 万条 1024 维向量 ≈ 4GB。

---

## 与 VectorStore 的维度兼容

切换 embedding 时必须**重建向量库**，因为不同维度的向量无法放在同一个索引中：

```python
# 从嵌入模型获取维度
embeddings = ZhipuAIEmbeddings(model="embedding-3", dimensions=1024)
test_vec = embeddings.embed_query("test")
dim = len(test_vec)  # 1024

# 创建向量库（自动使用正确的维度）
# FAISS 自动适配，PGVector 需要提前建索引
vectorstore = FAISS.from_documents(docs, embeddings)
```

---

## 子页面

- [[langchain/modules/langchain_embeddings/自定义Embedding实现|自定义 Embedding 实现]] — 通用封装方法
- [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings|CacheBackedEmbeddings]] — 嵌入缓存降低 API 成本

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embedding 模块总览]]
- [智谱 AI 开放平台](https://docs.bigmodel.cn/)
- [zai-sdk GitHub](https://github.com/zai-org/z-ai-sdk-python)
- [Embedding-3 模型文档](https://docs.bigmodel.cn/cn/guide/models/embedding/embedding-3)
