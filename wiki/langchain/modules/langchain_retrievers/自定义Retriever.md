---
tags: [langchain, retrievers, custom, base-retriever]
type: concept
created: 2026-07-07
updated: 2026-07-13
langchain_version: ">=0.3.0"
---

# 自定义 Retriever

## 是什么

当内置的检索器都不满足需求（如：调用外部搜索 API、数据库查询、混合多个数据源），继承 `BaseRetriever` 实现自定义检索逻辑，然后直接塞进 LCEL 链。

## 必须实现的方法

| 方法 | 说明 | 必选 |
|------|------|:---:|
| `_get_relevant_documents(query, *, run_manager)` | 同步检索 | ✅ |
| `_aget_relevant_documents(query, *, run_manager)` | 异步检索 | 可选（不实现会用线程池跑同步版） |

---

## 示例 1：关键词匹配检索器

```python
from typing import List
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun


class KeywordRetriever(BaseRetriever):
    """基于关键词匹配的简单检索器"""

    documents: List[Document] = []

    def _get_relevant_documents(
        self,
        query: str,
        *,
        run_manager: CallbackManagerForRetrieverRun,
    ) -> List[Document]:
        keywords = set(query.lower().split())
        results = []
        for doc in self.documents:
            doc_words = set(doc.page_content.lower().split())
            if keywords & doc_words:  # 有交集
                results.append(doc)
        return results

# 使用
retriever = KeywordRetriever(documents=[
    Document(page_content="LangChain is a framework"),
    Document(page_content="Python is a language"),
])
retriever.invoke("framework")  # → 返回第 1 篇
```

---

## 示例 2：外部 API 搜索检索器

```python
import requests
from typing import List, Optional
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun


class SearchAPIRetriever(BaseRetriever):
    """调用外部搜索 API 的检索器"""

    api_url: str = "https://api.example.com/search"
    api_key: Optional[str] = None
    top_k: int = 5

    def _get_relevant_documents(
        self,
        query: str,
        *,
        run_manager: CallbackManagerForRetrieverRun,
    ) -> List[Document]:
        response = requests.get(
            self.api_url,
            params={"q": query, "limit": self.top_k},
            headers={"Authorization": f"Bearer {self.api_key}"},
        )
        response.raise_for_status()

        return [
            Document(
                page_content=item["content"],
                metadata={"source": item.get("url"), "score": item.get("score")},
            )
            for item in response.json()["results"]
        ]

    async def _aget_relevant_documents(
        self,
        query: str,
        *,
        run_manager: CallbackManagerForRetrieverRun,
    ) -> List[Document]:
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(
                self.api_url,
                params={"q": query, "limit": self.top_k},
                headers={"Authorization": f"Bearer {self.api_key}"},
            ) as resp:
                data = await resp.json()
                return [
                    Document(
                        page_content=item["content"],
                        metadata={"source": item.get("url")},
                    )
                    for item in data["results"]
                ]
```

---

## 示例 3：数据库查询检索器

```python
import sqlite3
from typing import List
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun


class SQLiteRetriever(BaseRetriever):
    """从 SQLite 数据库检索文档"""

    db_path: str = ":memory:"
    table_name: str = "documents"

    def _get_relevant_documents(
        self,
        query: str,
        *,
        run_manager: CallbackManagerForRetrieverRun,
    ) -> List[Document]:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.execute(
            f"SELECT content, metadata FROM {self.table_name} "
            "WHERE content LIKE ? LIMIT 5",
            (f"%{query}%",),
        )
        import json
        results = [
            Document(page_content=row[0], metadata=json.loads(row[1]))
            for row in cursor.fetchall()
        ]
        conn.close()
        return results
```

---

## 核心约定

1. **返回类型必须是 `List[Document]`**——这是 LangChain 所有 Retriever 的契约
2. **自定义 Retriever 自动继承 `invoke` / `ainvoke` / `batch`**——不需要手动实现
3. **异步方法如果不实现**，基类会用 `asyncio.to_thread` 跑同步版本，但会阻塞事件循环——生产环境务必实现 `_aget_relevant_documents`
4. **可以用 Pydantic Field 定义配置参数**——但不要继承 `BaseModel`（`BaseRetriever` 已经继承了）

---

## 如何链入 LCEL

自定义 Retriever 是标准 Runnable，直接 `|` 进去：

```python
retriever = SearchAPIRetriever(api_key="xxx", top_k=3)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

rag_chain.invoke("最新 AI 动态")
```

---

## 常见错误

| 错误 | 原因 | 修正 |
|------|------|------|
| `AttributeError: 'list' object has no attribute 'page_content'` | 返回了字符串列表而非 Document 列表 | 包装为 `Document(page_content=item)` |
| `NotImplementedError` 异步调用 | 只实现了 `_get_relevant_documents` 未实现异步版 | 实现 `_aget_relevant_documents` 或接受线程池 |
| 检索结果为空 | `_get_relevant_documents` 中的逻辑有 bug | 加日志，检查 query 和 documents |

---

## 子页面

- [[langchain/modules/langchain_retrievers/VectorStoreRetriever|VectorStoreRetriever]] — 参照实现
- [[langchain/modules/langchain_retrievers/概述|← 返回 retrievers 概述]]
