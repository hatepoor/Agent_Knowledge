---
tags: [fastapi, background-tasks, agent, 后台任务, 异步]
type: tutorial
created: 2026-07-22
updated: 2026-07-22
---

# 后台任务与长时间 Agent 作业

FastAPI 的 `BackgroundTasks` 让服务器先返回响应，再在后台执行耗时操作。Agent 场景中常用于：长时间研究任务、文档索引、批量处理。

## 一、为什么需要后台任务

```
同步模式（用户等 30 秒）：
  用户 POST /research → Agent 研究 30 秒 → 返回结果
  用户体验：页面转圈 30 秒，可能超时

后台模式（立即响应）：
  用户 POST /research → 立即返回 {"task_id": "abc", "status": "running"}
  后台：Agent 研究 30 秒 → 结果存入数据库
  用户 GET /research/abc → 查询进度/结果
```

## 二、BackgroundTasks 基础

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, content: str):
    """后台执行：发送邮件（不阻塞响应）"""
    import time
    time.sleep(3)  # 模拟耗时操作
    print(f"已发送邮件到 {email}")

@app.post("/register")
async def register(email: str, background_tasks: BackgroundTasks):
    # 先返回响应给用户
    # 后台任务在响应发送后执行
    background_tasks.add_task(send_email, email, "欢迎注册！")
    return {"message": "注册成功，邮件稍后发送"}
```

### 执行顺序

```
请求进入 → 执行路由函数 → 返回响应给用户 → 执行 background_tasks
                              ↑                      ↑
                         用户已经收到            后台静默执行
```

## 三、Agent 长时间任务（完整实现）

```python
import asyncio
import uuid
import time
from fastapi import FastAPI, BackgroundTasks, HTTPException
from pydantic import BaseModel
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage

app = FastAPI()
agent = create_agent("openai:gpt-4o", tools=[])

# 任务存储（生产环境用 Redis/数据库）
tasks_db: dict[str, dict] = {}

class ResearchRequest(BaseModel):
    topic: str
    depth: str = "normal"  # normal | deep

class TaskResponse(BaseModel):
    task_id: str
    status: str  # running | done | failed
    message: str

# ===== 后台执行函数 =====
async def run_research(task_id: str, topic: str, depth: str):
    """后台执行 Agent 研究任务"""
    try:
        tasks_db[task_id]["status"] = "running"
        tasks_db[task_id]["started_at"] = time.time()
        
        prompt = f"深入研究以下主题，给出详细报告：{topic}"
        if depth == "deep":
            prompt += "。要求：至少 2000 字，包含数据支撑和参考来源。"
        
        result = await agent.ainvoke({
            "messages": [HumanMessage(content=prompt)]
        })
        
        tasks_db[task_id]["status"] = "done"
        tasks_db[task_id]["result"] = result["messages"][-1].content
        tasks_db[task_id]["finished_at"] = time.time()
    
    except Exception as e:
        tasks_db[task_id]["status"] = "failed"
        tasks_db[task_id]["error"] = str(e)

# ===== API 端点 =====
@app.post("/research", response_model=TaskResponse)
async def start_research(req: ResearchRequest, background_tasks: BackgroundTasks):
    """提交研究任务（立即返回，后台执行）"""
    task_id = str(uuid.uuid4())[:8]
    tasks_db[task_id] = {"status": "pending", "topic": req.topic}
    
    # 添加到后台任务
    background_tasks.add_task(run_research, task_id, req.topic, req.depth)
    
    return TaskResponse(
        task_id=task_id,
        status="running",
        message=f"研究任务已提交，主题: {req.topic}",
    )

@app.get("/research/{task_id}")
async def get_research_status(task_id: str):
    """查询任务进度/结果"""
    task = tasks_db.get(task_id)
    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")
    
    response = {"task_id": task_id, "status": task["status"]}
    
    if task["status"] == "done":
        response["result"] = task["result"]
        duration = task["finished_at"] - task["started_at"]
        response["duration_seconds"] = round(duration, 1)
    elif task["status"] == "failed":
        response["error"] = task["error"]
    
    return response

@app.get("/research")
async def list_research_tasks():
    """列出所有任务"""
    return [
        {"task_id": tid, "status": t["status"], "topic": t.get("topic", "")}
        for tid, t in tasks_db.items()
    ]
```

### 使用流程

```
1. POST /research {"topic": "LangGraph 最新特性"}
   → 立即返回: {"task_id": "a1b2c3", "status": "running"}

2. GET /research/a1b2c3（轮询）
   → {"task_id": "a1b2c3", "status": "running"}  （还在跑）

3. GET /research/a1b2c3（再查）
   → {"task_id": "a1b2c3", "status": "done", "result": "...", "duration_seconds": 25.3}
```

## 四、BackgroundTasks vs asyncio.create_task

| 维度 | BackgroundTasks | asyncio.create_task |
|------|----------------|---------------------|
| 执行时机 | 响应发送**后** | 立即（不等响应） |
| 生命周期 | 跟随请求 | 跟随事件循环 |
| 错误处理 | 静默失败（需自己 try） | 可以 await 获取异常 |
| 适用场景 | 发邮件、写日志、简单后处理 | 长时间 Agent 任务、需要追踪状态 |
| 能否查询进度 | 不能（fire-and-forget） | 能（配合任务存储） |

**Agent 场景建议：** 简单的后处理（保存日志、发通知）用 `BackgroundTasks`。长时间 Agent 任务用 `asyncio.create_task` + 任务存储 + 查询端点。

## 五、带进度上报的后台任务

```python
@app.post("/research/progress", response_model=TaskResponse)
async def start_research_with_progress(req: ResearchRequest):
    """带进度上报的研究任务"""
    task_id = str(uuid.uuid4())[:8]
    tasks_db[task_id] = {"status": "running", "progress": 0, "topic": req.topic}
    
    # 用 create_task 而不是 BackgroundTasks（需要追踪进度）
    asyncio.create_task(run_research_with_progress(task_id, req.topic))
    
    return TaskResponse(task_id=task_id, status="running", message="已提交")

async def run_research_with_progress(task_id: str, topic: str):
    """分步骤执行，每步更新进度"""
    steps = ["搜索相关资料", "分析整理", "生成报告", "质量检查"]
    
    for i, step in enumerate(steps):
        tasks_db[task_id]["progress"] = int((i / len(steps)) * 100)
        tasks_db[task_id]["current_step"] = step
        
        # 每步调用 Agent
        result = await agent.ainvoke({
            "messages": [HumanMessage(content=f"{step}: {topic}")]
        })
        await asyncio.sleep(1)  # 模拟处理时间
    
    tasks_db[task_id]["status"] = "done"
    tasks_db[task_id]["progress"] = 100
    tasks_db[task_id]["result"] = result["messages"][-1].content

@app.get("/research/{task_id}/progress")
async def get_progress(task_id: str):
    """查询进度"""
    task = tasks_db.get(task_id)
    if not task:
        raise HTTPException(404, "任务不存在")
    return {
        "task_id": task_id,
        "status": task["status"],
        "progress": task.get("progress", 0),
        "current_step": task.get("current_step", ""),
    }
```

## 六、常见陷阱

### 1. BackgroundTasks 中用 async 函数

```python
# BackgroundTasks 支持 async 函数，但要注意：
# 它在响应发送后执行，此时请求上下文已关闭

# 正确：async 函数可以直接用
background_tasks.add_task(run_research, task_id, topic)

# 错误：不要在里面访问 request 对象
async def bad_task(request: Request):  # request 已经无效了！
    ...
```

### 2. 任务丢失（进程重启）

```python
# 错误：任务存在内存字典里，重启就没了
tasks_db = {}  # 重启后清空

# 正确：用持久化存储
# 开发：SqliteSaver
# 生产：Redis / PostgreSQL
```

### 3. 没有超时控制

```python
# 错误：Agent 可能跑 5 分钟，一直占资源
result = await agent.ainvoke(...)

# 正确：设置超时
try:
    result = await asyncio.wait_for(
        agent.ainvoke({"messages": [HumanMessage(content=prompt)]}),
        timeout=120,  # 最多 2 分钟
    )
except asyncio.TimeoutError:
    tasks_db[task_id]["status"] = "failed"
    tasks_db[task_id]["error"] = "任务超时（>120秒）"
```

## 相关

- [[fastapi/异步处理|异步处理]] — async/await 基础
- [[fastapi/与LangChain集成|与 LangChain 集成]] — Agent API 基础
- [[fastapi/WebSocket与Agent实时通信|WebSocket]] — 实时推送进度（替代轮询）
- [[fastapi/FastAPI概述|<- 返回 FastAPI 概述]]
