---
tags: [RAG, multimodal, 多模态, 图像检索, embedding]
type: howto
created: 2026-07-26
updated: 2026-07-26
---

# 多模态 RAG

> 📌 **历史遗留页面**：权威内容见 [[rag/14_范式与前沿/多模态与长上下文RAG]]。对照见 [[agent/RAG/README|RAG 迁移对照表]]。

## 概述

传统 RAG 只处理文本：文档切块 → 文本向量化 → 检索 → 塞进 prompt。**多模态 RAG** 将图片、音频等非文本内容也纳入检索和生成流程。核心变化在三个环节：**多模态 Embedding**、**图文混合入库**、**检索结果的多模态组装**。

## 架构差异

```
传统 RAG：
  文本文档 → 切块 → 文本 Embedding → 向量库
  文本查询 → 文本 Embedding → 检索 → 文本上下文 → LLM → 文本回答

多模态 RAG：
  文本文档 ─→ 切块 ─→ 文本 Embedding ──┐
  文档中的图片 → 图片 Embedding ─────┤→ 向量库（统一空间）
                                     │
  文本查询 → 文本 Embedding → 检索 ──┘
       ↓
  命中图片 → base64 编码 → 多模态消息 ──┐
  命中文本 → 文本上下文 ──────────────┤→ 多模态 LLM → 回答
                                      ┘
```

关键区别：文本和图片被映射到**同一个向量空间**，所以一个文本 query 可以同时检索到文本和图片。

## 1. 多模态 Embedding 模型

普通 Embedding 模型只接受文本。多模态 RAG 需要能把文本和图片映射到同一向量空间的模型：

| 模型 | 来源 | 维度 | 特点 |
|------|------|------|------|
| `clip-ViT-B-32` | 开源 (SentenceTransformers) | 512 | 经典图文对齐，本地部署 |
| `text-embedding-3-large` | OpenAI | 3072 | 云端 API，支持图文 |
| `jina-clip-v2` | Jina AI | 1024 | 开源，多语言+多模态 |
| `Cohere embed-v3` | Cohere | 1024 | 支持图文混合检索 |
| `bge-visualized` | BAAI | 768 | 开源，中英文双语多模态 |

### CLIP 本地部署

```python
from sentence_transformers import SentenceTransformer
from PIL import Image

clip = SentenceTransformer("clip-ViT-B-32")

# 文本向量化
text_vector = clip.encode(["一只猫坐在沙发上", "系统架构图"])

# 图片向量化
images = [Image.open("cat.jpg"), Image.open("arch_diagram.png")]
image_vectors = clip.encode(images)

# 文本和图片在同一向量空间，可以计算相似度
from sentence_transformers import util
scores = util.cos_sim(text_vector, image_vectors)
```

### OpenAI API

```python
from openai import OpenAI

client = OpenAI()

# 文本 embedding
text_response = client.embeddings.create(
    model="text-embedding-3-large",
    input="一只猫坐在沙发上"
)

# 图片 embedding（通过 base64）
import base64
with open("cat.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

image_response = client.embeddings.create(
    model="text-embedding-3-large",
    input=[{"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}}]
)
```

## 2. 文档解析：提取文本 + 图片

### PDF 中的图片提取

```python
import fitz  # PyMuPDF
from langchain_core.documents import Document

def parse_pdf_with_images(pdf_path: str) -> list[Document]:
    """解析 PDF，同时提取文本块和图片"""
    docs = []
    pdf = fitz.open(pdf_path)

    for page_num, page in enumerate(pdf):
        # 提取文本
        text = page.get_text()
        if text.strip():
            docs.append(Document(
                page_content=text,
                metadata={"source": pdf_path, "page": page_num, "type": "text"}
            ))

        # 提取图片
        for img_index, img in enumerate(page.get_images()):
            xref = img[0]
            base_image = pdf.extract_image(xref)
            image_bytes = base_image["image"]
            image_ext = base_image["ext"]

            # 保存图片到本地
            img_path = f"data/images/{pdf_path}_p{page_num}_{img_index}.{image_ext}"
            with open(img_path, "wb") as f:
                f.write(image_bytes)

            docs.append(Document(
                page_content=img_path,  # 存路径，检索到时再读取
                metadata={
                    "source": pdf_path,
                    "page": page_num,
                    "type": "image",
                    "caption": "",  # 可后续用多模态 LLM 自动生成
                }
            ))

    return docs
```

### 自动生成图片描述 (Caption)

给图片生成文本描述，后续检索时可以先用 caption 做文本匹配（快且便宜），命中了再把原图发给模型：

```python
from langchain_core.messages import HumanMessage

async def generate_caption(image_path: str, llm) -> str:
    """用多模态 LLM 给图片生成描述"""
    import base64
    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()

    result = await llm.ainvoke([
        HumanMessage(content=[
            {"type": "text", "text": "用一句话简洁描述这张图片的内容"},
            {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}},
        ])
    ])
    return result.content

# 入库时自动生成
caption = await generate_caption("screenshot.png", llm)
doc.metadata["caption"] = caption
```

## 3. 入库：图文混合存储

文本和图片使用**同一个向量库**，通过 metadata 中的 `type` 字段区分：

```python
from langchain_chroma import Chroma

vectorstore = Chroma(
    collection_name="multimodal_rag",
    embedding_function=clip_embeddings,  # 多模态 embedding 函数
    persist_directory="./chroma_mm"
)

# 文本和图片一起入库
vectorstore.add_documents(all_docs)  # 包含 type=text 和 type=image 的 Document
```

两种检索策略：

```python
# 策略 1：统一检索，不区分类型
results = vectorstore.similarity_search("系统架构是什么", k=5)
# 可能返回文本块和图片块

# 策略 2：分开检索再合并
text_results = vectorstore.similarity_search(
    "系统架构是什么", k=3, filter={"type": "text"}
)
image_results = vectorstore.similarity_search(
    "系统架构是什么", k=2, filter={"type": "image"}
)
all_results = text_results + image_results
```

## 4. 检索结果组装：发给多模态 LLM

检索到图片时，需要把图片数据读出来转 base64，组装成多模态消息：

```python
import base64
from langchain_core.messages import HumanMessage, SystemMessage

def build_multimodal_prompt(query: str, docs: list) -> list:
    """将检索结果组装成多模态 prompt"""
    content = [{"type": "text", "text": f"根据以下参考资料回答问题。\n问题：{query}\n\n参考资料：\n"}]

    for i, doc in enumerate(docs):
        if doc.metadata.get("type") == "image":
            img_path = doc.page_content
            caption = doc.metadata.get("caption", "图片")
            with open(img_path, "rb") as f:
                img_b64 = base64.b64encode(f.read()).decode()
            suffix = img_path.rsplit(".", 1)[-1].lower()
            mime = {"png": "image/png", "jpg": "image/jpeg",
                    "jpeg": "image/jpeg"}.get(suffix, "image/png")
            content.append({"type": "text", "text": f"[图片 {i+1}：{caption}]"})
            content.append({
                "type": "image_url",
                "image_url": {"url": f"data:{mime};base64,{img_b64}"},
            })
        else:
            content.append({"type": "text", "text": f"[文档 {i+1}]\n{doc.page_content}\n"})

    return [
        SystemMessage(content="你是一个智能助手，可以根据文本和图片回答问题。"),
        HumanMessage(content=content),
    ]
```

## 5. 完整 Pipeline

```python
from langchain.agents import create_agent
from langchain_chroma import Chroma
from langchain_openai import ChatOpenAI

# 1. 初始化
llm = ChatOpenAI(model="gpt-4o")  # 多模态模型
vectorstore = Chroma(embedding_function=clip_embeddings, persist_directory="./chroma_mm")
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 2. 定义检索工具
@tool
def search_knowledge_base(query: str) -> str:
    """搜索知识库，返回相关文本和图片引用"""
    docs = retriever.invoke(query)
    text_parts = []
    for doc in docs:
        if doc.metadata.get("type") == "image":
            text_parts.append(f"[图片：{doc.metadata.get('caption', '图片')}] (路径: {doc.page_content})")
        else:
            text_parts.append(doc.page_content)
    return "\n\n".join(text_parts)

# 3. 创建 Agent
agent = create_agent(llm, tools=[search_knowledge_base])

# 4. 高级用法：检索后自动注入图片到消息
# 这需要自定义 Agent 的 prompt 组装逻辑，
# 在检索工具返回后，把图片以 base64 形式附加到下一轮的 HumanMessage 中
```

## 注意事项

| 问题 | 说明 | 解决方案 |
|------|------|----------|
| 向量库膨胀 | 图片向量 + 图片文件占用大 | 图片文件单独存，向量库只存路径和 caption |
| Token 成本 | 图片转 token 很贵 | 先用 caption 做文本检索，命中再传图 |
| 模型兼容性 | 不是所有模型都支持多模态输入 | 确认 LLM 支持图片输入（GPT-4o/Claude 3） |
| 检索精度 | 图片和文本的向量空间对齐质量影响检索 | 选择高质量多模态 Embedding（CLIP v2+） |
| Checkpointer 膨胀 | 历史消息中的 base64 图片占大量存储 | 对历史消息做图片裁剪，只保留文本占位 |

## 两阶段检索（推荐模式）

为了平衡效果和成本，实际项目通常采用两阶段检索：

```python
async def two_stage_retrieve(query: str, vectorstore, top_k=5, image_top_k=2):
    """
    第一阶段：用文本 embedding 检索文本块（快、便宜）
    第二阶段：只对命中的图片做 base64 编码传给模型（按需、贵）
    """
    # 第一阶段：纯文本检索
    text_docs = vectorstore.similarity_search(
        query, k=top_k, filter={"type": "text"}
    )

    # 同时检索图片，但只拿 caption
    image_docs = vectorstore.similarity_search(
        query, k=image_top_k, filter={"type": "image"}
    )

    return text_docs, image_docs

# 使用时，先只传文本和 caption
# 如果模型判断需要看图，再传完整图片
```

## 相关页面

- [[agent/RAG/RAG|← 返回 RAG 概述]]
- [[agent/RAG/RAG_LangChain完整实现|LangChain RAG 完整实现]]
- [[agent/RAG/RAG关键模块解析|RAG 关键模块解析]]
- [[agent/RAG/向量数据库|向量数据库]]
- [[agent/perception/多模态|多模态概念]]
- [[agent/perception/Agent多模态集成|Agent 多模态集成]]
- [[langchain/modules/langchain_messages/多模态消息|多模态消息格式]]
