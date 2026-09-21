---
tags: [concept, Python, multiprocessing, Process, 退出码]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Process 类详解

> Process 是 `multiprocessing` 模块的核心类，API 与 `threading.Thread` 几乎一致。

## 创建进程

```python
from multiprocessing import Process
import os
import time

def worker(name, delay):
    print(f"[{name}] PID={os.getpid()}, PPID={os.getppid()}")
    time.sleep(delay)
    print(f"[{name}] 完成")

if __name__ == "__main__":
    p = Process(target=worker, args=("子进程", 2), name="worker-1")
    p.start()
    p.join()
    print(f"主进程 PID={os.getpid()} 结束")
```

**关键**：`if __name__ == "__main__":` 是**必须的**（Windows 上），因为 spawn 方式会重新导入主模块。

## Process vs Thread API

```python
from multiprocessing import Process
import threading

p = Process(target=func, args=(1,), name="proc-1", daemon=False)
p.start()
p.join(timeout=5)
p.is_alive()
p.name
p.daemon
p.pid           # 进程 ID（Thread 没有）
p.exitcode      # 退出码（Thread 没有）
```

## 退出码

| exitcode | 含义 |
|----------|------|
| `None` | 进程仍在运行 |
| `0` | 正常退出 |
| `> 0` | 进程异常退出 |
| `< 0` | 进程被信号 N 终止（如 -9 = SIGKILL） |

## 继承 Process

```python
from multiprocessing import Process

class WorkerProcess(Process):
    def __init__(self, task_queue, result_queue):
        super().__init__()
        self.task_queue = task_queue
        self.result_queue = result_queue

    def run(self):
        while True:
            task = self.task_queue.get()
            if task is None:
                break
            result = self.process_task(task)
            self.result_queue.put(result)

    def process_task(self, task):
        return task * 2
```

## 相关概念

- [[python/concurrency/04_multiprocessing/multiprocessing概览]] — 模块概览
- [[python/concurrency/04_multiprocessing/Pool进程池]] — 推荐使用 Pool 而非裸 Process
- [[python/concurrency/02_threading/Thread类与生命周期]] — 与 Thread API 对比

## 来源

- Python 官方文档 — Process Objects：https://docs.python.org/3/library/multiprocessing.html#process-objects
