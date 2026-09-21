---
tags: [concept, Python, threading, Event, 事件, 信号通知]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Event 事件

> Event 是一个布尔标志，线程可以等待标志变为 True，也可以设置/清除标志。适用于启动信号和优雅停止。

## 基本概念

```
Event 初始状态: False

线程 A: event.wait()  ⏳ 阻塞等待...
线程 B: event.set()   标志 → True
线程 A: wait() 返回，继续执行

event.clear()  标志 → False（重置）
```

## 使用场景：启动信号

```python
import threading
import time

start_event = threading.Event()

def worker(worker_id):
    print(f"  Worker-{worker_id} 等待启动信号...")
    start_event.wait()
    print(f"  Worker-{worker_id} 启动！")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(5)]
for t in threads: t.start()

time.sleep(2)
print("发出启动信号！")
start_event.set()

for t in threads: t.join()
```

## 使用场景：优雅停止

```python
import threading

stop_event = threading.Event()

def monitor():
    while not stop_event.is_set():
        print("监控中...")
        stop_event.wait(timeout=1)  # 用 wait 代替 sleep
    print("监控停止")

t = threading.Thread(target=monitor, daemon=True)
t.start()

time.sleep(5)
stop_event.set()
t.join(timeout=2)
```

## Event API

| 方法 | 说明 |
|------|------|
| `set()` | 设置标志为 True，唤醒所有 wait 中的线程 |
| `clear()` | 设置标志为 False |
| `is_set()` | 返回标志当前状态 |
| `wait(timeout=None)` | 阻塞直到标志为 True 或超时 |

**关键区别**：`event.wait()` vs `time.sleep()` — `wait()` 可以被 `set()` 唤醒，`sleep()` 不能。

## 相关概念

- [[python/concurrency/03_同步原语/Condition条件变量]] — 比 Event 更灵活的条件变量
- [[python/concurrency/03_同步原语/同步原语选择指南]] — 各原语对比

## 来源

- Python 官方文档 — Event Objects：https://docs.python.org/3/library/threading.html#event-objects
