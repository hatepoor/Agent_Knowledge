---
tags: [langchain, embeddings, huggingface, local]
type: concept
created: 2026-07-07
updated: 2026-07-11
---

# HuggingFaceEmbeddings

## 是什么

`HuggingFaceEmbeddings` 基于 **HuggingFace sentence-transformers** 库实现本地文本嵌入，无需 API Key，适合隐私敏感场景和离线使用。

## 安装与导入

```bash
pip install langchain-huggingface sentence-transformers
```

```python
from langchain_huggingface import HuggingFaceEmbeddings
```

## 基本用法

```python
from langchain_huggingface import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={"device": "cpu"},     # 或 "cuda"
    encode_kwargs={"normalize_embeddings": True},
)

vec = embeddings.embed_query("Hello world")
print(f"HF vector: {len(vec)} dims")  # 384
```

## 推荐模型

| 模型 | 维度 | 特点 |
|------|------|------|
| `all-MiniLM-L6-v2` | 384 | 轻量快速，适合原型开发 |
| `all-mpnet-base-v2` | 768 | 更高精度 |
| `BAAI/bge-large-zh-v1.5` | 1024 | 中文嵌入推荐 |
| `BAAI/bge-large-en-v1.5` | 1024 | 英文嵌入推荐 |

## 配置参数

```python
embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-large-zh-v1.5",     # 模型名称
    model_kwargs={
        "device": "cuda",                      # GPU 加速
        "trust_remote_code": True,             # 某些模型需要
    },
    encode_kwargs={
        "normalize_embeddings": True,          # 向量归一化
        "batch_size": 32,                      # 批处理大小
    },
)
```

## GPU 加速

```python
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={"device": "cuda"},  # 使用 GPU
)
```

## 与 OpenAI 的对比

| 特性 | OpenAIEmbeddings | HuggingFaceEmbeddings |
|------|-----------------|----------------------|
| API Key | 需要 | 不需要 |
| 运行位置 | 云端 | 本地 |
| 速度 | 受网络影响 | 取决于硬件 |
| 隐私 | 数据发送到 OpenAI | 完全本地 |
| 中文质量 | 好 | 取决于模型 |

## 注意事项

- 首次使用会自动下载模型（约 80MB-1.3GB）
- 模型下载后缓存在 `~/.cache/huggingface/`
- GPU 需要安装对应的 CUDA 和 PyTorch 版本
- 不同模型的向量维度不同，切换模型需重建向量库

## 相关

- [[langchain/modules/langchain_embeddings/概述|← 返回 embeddings 概述]]
- [[langchain/modules/langchain_embeddings/OpenAIEmbeddings|OpenAIEmbeddings - 云端替代]]
- [[langchain/modules/langchain_embeddings/CacheBackedEmbeddings|CacheBackedEmbeddings - 添加缓存]]
