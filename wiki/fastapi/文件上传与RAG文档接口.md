---
tags: [fastapi, file-upload, rag, 文件上传, 文档处理]
type: tutorial
created: 2026-07-22
updated: 2026-07-22
---

# 文件上传与 RAG 文档接口

FastAPI 原生支持文件上传（`UploadFile`），在 Agent 项目中常用于：RAG 知识库文档上传、用户附件处理、批量导入。

## 一、基础文件上传

```python
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    # UploadFile 的属性
    # file.filename  → "report.pdf"
    # file.content_type → "application/pdf"
    # file.size → 文件大小（字节）
    
    # 读取内容
    content = await file.read()  # bytes
    
    # 保存到磁盘
    save_path = f"uploads/{file.filename}"
    with open(save_path, "wb") as f:
        f.write(content)
    
    return {
        "filename": file.filename,
        "size": len(content),
        "content_type": file.content_type,
    }
```

### UploadFile vs bytes

| 参数类型 | 适用场景 | 内存占用 |
|---------|---------|---------|
| `file: UploadFile` | 大文件、需要文件名/类型 | 低（流式读取） |
| `file: bytes = File(...)` | 小文件、只关心内容 | 高（全部加载到内存） |

## 二、RAG 文档上传接口（完整实现）

```python
import os
import uuid
from pathlib import Path
from fastapi import FastAPI, UploadFile, File, HTTPException, BackgroundTasks
from pydantic import BaseModel
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import init_embeddings
from langchain.vectorstores import Chroma

app = FastAPI()

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)
VECTORSTORE_DIR = "data/vectorstore"

# 初始化向量数据库
embeddings = init_embeddings("openai:text-embedding-3-small")
vectorstore = Chroma(
    persist_directory=VECTORSTORE_DIR,
    embedding_function=embeddings,
)

# 文档处理状态
processing_tasks: dict[str, dict] = {}

# ===== 文件类型处理 =====
def extract_text(file_path: Path, content_type: str) -> str:
    """根据文件类型提取文本"""
    if content_type == "application/pdf":
        from pypdf import PdfReader
        reader = PdfReader(str(file_path))
        return "\n".join(page.extract_text() or "" for page in reader.pages)
    
    elif content_type in ("text/plain", "text/markdown"):
        return file_path.read_text(encoding="utf-8")
    
    elif content_type == "application/vnd.openxmlformats-officedocument.wordprocessingml.document":
        # .docx
        import docx
        doc = docx.Document(str(file_path))
        return "\n".join(p.text for p in doc.paragraphs)
    
    else:
        raise ValueError(f"不支持的文件类型: {content_type}")

# ===== 后台索引任务 =====
async def index_document(task_id: str, file_path: Path, content_type: str):
    """后台执行：提取文本 → 分块 → 向量化 → 存储"""
    try:
        processing_tasks[task_id]["status"] = "extracting"
        
        # 1. 提取文本
        text = extract_text(file_path, content_type)
        if not text.strip():
            processing_tasks[task_id]["status"] = "failed"
            processing_tasks[task_id]["error"] = "文件内容为空"
            return
        
        # 2. 分块
        processing_tasks[task_id]["status"] = "chunking"
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=500,
            chunk_overlap=50,
        )
        chunks = splitter.split_text(text)
        
        # 3. 向量化 + 存储
        processing_tasks[task_id]["status"] = "embedding"
        metadatas = [{"source": file_path.name, "chunk_id": i} for i in range(len(chunks))]
        vectorstore.add_texts(chunks, metadatas=metadatas)
        
        # 4. 完成
        processing_tasks[task_id]["status"] = "done"
        processing_tasks[task_id]["chunks"] = len(chunks)
        processing_tasks[task_id]["chars"] = len(text)
    
    except Exception as e:
        processing_tasks[task_id]["status"] = "failed"
        processing_tasks[task_id]["error"] = str(e)

# ===== API 端点 =====
@app.post("/documents/upload")
async def upload_document(
    file: UploadFile = File(...),
    background_tasks: BackgroundTasks = None,
):
    """上传文档并后台索引到知识库"""
    # 校验文件类型
    allowed_types = {
        "application/pdf", "text/plain", "text/markdown",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    }
    if file.content_type not in allowed_types:
        raise HTTPException(400, f"不支持的文件类型: {file.content_type}")
    
    # 校验文件大小（最大 20MB）
    content = await file.read()
    if len(content) > 20 * 1024 * 1024:
        raise HTTPException(400, "文件过大（最大 20MB）")
    
    # 保存文件
    file_id = str(uuid.uuid4())[:8]
    save_path = UPLOAD_DIR / f"{file_id}_{file.filename}"
    save_path.write_bytes(content)
    
    # 后台索引
    task_id = file_id
    processing_tasks[task_id] = {"status": "pending", "filename": file.filename}
    background_tasks.add_task(index_document, task_id, save_path, file.content_type)
    
    return {
        "file_id": file_id,
        "task_id": task_id,
        "filename": file.filename,
        "status": "processing",
        "message": "文件已上传，正在后台索引",
    }

@app.get("/documents/{task_id}/status")
async def get_indexing_status(task_id: str):
    """查询索引进度"""
    task = processing_tasks.get(task_id)
    if not task:
        raise HTTPException(404, "任务不存在")
    return {"task_id": task_id, **task}

@app.post("/documents/search")
async def search_documents(query: str, top_k: int = 5):
    """在已索引的文档中搜索"""
    results = vectorstore.similarity_search_with_score(query, k=top_k)
    return [
        {
            "content": doc.page_content,
            "source": doc.metadata.get("source", "unknown"),
            "score": round(score, 3),
        }
        for doc, score in results
    ]

@app.get("/documents")
async def list_documents():
    """列出所有已上传的文档"""
    return [
        {"task_id": tid, "filename": t.get("filename"), "status": t["status"]}
        for tid, t in processing_tasks.items()
    ]
```

### 使用流程

```
1. POST /documents/upload（上传 PDF）
   → {"file_id": "a1b2", "task_id": "a1b2", "status": "processing"}

2. GET /documents/a1b2/status（查询进度）
   → {"status": "embedding", ...}  （正在向量化）

3. GET /documents/a1b2/status（再查）
   → {"status": "done", "chunks": 42, "chars": 15000}

4. POST /documents/search?query=TCP三次握手
   → [{"content": "...", "source": "network.pdf", "score": 0.92}]
```

## 三、多文件批量上传

```python
from fastapi import UploadFile, File

@app.post("/documents/batch-upload")
async def batch_upload(
    files: list[UploadFile] = File(...),
    background_tasks: BackgroundTasks = None,
):
    """批量上传多个文档"""
    results = []
    
    for file in files:
        content = await file.read()
        file_id = str(uuid.uuid4())[:8]
        save_path = UPLOAD_DIR / f"{file_id}_{file.filename}"
        save_path.write_bytes(content)
        
        processing_tasks[file_id] = {"status": "pending", "filename": file.filename}
        background_tasks.add_task(index_document, file_id, save_path, file.content_type)
        
        results.append({"file_id": file_id, "filename": file.filename})
    
    return {"uploaded": len(results), "files": results}
```

## 四、与 Agent 集成（上传后直接问答）

```python
from langchain.tools import tool
from langchain.agents import create_agent

@tool
def search_knowledge_base(query: str, top_k: int = 5) -> str:
    """在用户上传的知识库文档中搜索相关内容。"""
    results = vectorstore.similarity_search(query, k=top_k)
    if not results:
        return "知识库中未找到相关内容"
    return "\n\n---\n\n".join(
        f"[来源: {doc.metadata.get('source')}]\n{doc.page_content}"
        for doc in results
    )

# Agent 可以使用这个工具
agent = create_agent("openai:gpt-4o", tools=[search_knowledge_base])

@app.post("/chat-with-docs")
async def chat_with_docs(message: str, thread_id: str = "default"):
    """基于已上传文档的对话"""
    result = await agent.ainvoke(
        {"messages": [HumanMessage(content=message)]},
        config={"configurable": {"thread_id": thread_id}},
    )
    return {"reply": result["messages"][-1].content}
```

## 五、常见陷阱

### 1. 大文件一次性读入内存

```python
# 错误：100MB 文件全部加载到内存
content = await file.read()  # 内存爆炸

# 正确：分块读取
chunk_size = 1024 * 1024  # 1MB
with open(save_path, "wb") as f:
    while chunk := await file.read(chunk_size):
        f.write(chunk)
```

### 2. 文件名安全问题

```python
# 错误：直接用用户文件名（可能有路径遍历攻击）
save_path = f"uploads/{file.filename}"  # filename 可能是 "../../etc/passwd"

# 正确：用 UUID 重命名
safe_name = f"{uuid.uuid4().hex}_{Path(file.filename).suffix}"
save_path = UPLOAD_DIR / safe_name
```

### 3. 忘记校验文件类型

```python
# 错误：任何文件都接受
@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    ...  # 用户可能上传 .exe

# 正确：白名单校验
ALLOWED = {"application/pdf", "text/plain", "text/markdown"}
if file.content_type not in ALLOWED:
    raise HTTPException(400, "不支持的文件类型")
```

## 相关

- [[fastapi/后台任务与长时间Agent作业|后台任务]] — 后台索引机制
- [[fastapi/与LangChain集成|与 LangChain 集成]] — Agent API 基础
- [[langchain/modules/langchain_documents/文本分割器|文本分割器]] — 分块策略
- [[langchain/modules/langchain_vectorstores/概述|向量数据库]] — 向量存储
- [[fastapi/FastAPI概述|<- 返回 FastAPI 概述]]
