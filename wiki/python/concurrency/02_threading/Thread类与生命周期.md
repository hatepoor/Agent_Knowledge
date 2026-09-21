---
tags: [concept, Python, threading, Thread, 生命周期, 守护线程]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Thread 类与生命周期

> Thread 类是 `threading` 模块的核心。本页讲解线程的五种状态、守护线程概念和 join 机制。

## 五种状态

```
      ┌─────────┐
      │  Created │  Thread(...) 构造完成
      └────┬─────┘
           │ .start()
           ▼
      ┌─────────┐
      │ Started  │  run() 开始执行
      └────┬─────┘
           │
     ┌─────┼──────────┐
     │     │          │
     ▼     ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Blocked│ │ Running│ │ Ready  │
│ (I/O等)│ │(执行中)│ │(等待GIL)│
└────────┘ └────────┘ └────────┘
     │           │
     └─────┬─────┘
           │ run() 返回 / 异常
           ▼
      ┌─────────┐
      │  Dead    │  线程终止
      └─────────┘
```

### 状态验证

```python
import threading
import time

def demo_lifecycle():
    print("线程执行中...")

t = threading.Thread(target=demo_lifecycle)

print(f"1. 创建后: is_alive={t.is_alive()}")  # False

t.start()
print(f"2. 启动后: is_alive={t.is_alive()}")   # True

t.join()
print(f"3. 结束后: is_alive={t.is_alive()}")   # False
```

## Thread 构造参数

```python
threading.Thread(
    group=None,       # 保留参数，必须为 None
    target=None,       # 线程执行的函数
    name=None,        # 线程名（默认 "Thread-N"）
    args=(),          # target 的位置参数
    kwargs={},        # target 的关键字参数
    daemon=None,      # 是否为守护线程
)
```

## 实例方法与属性

| 方法/属性 | 说明 |
|-----------|------|
| `start()` | 启动线程（只能调用一次，否则 RuntimeError） |
| `run()` | 线程执行体（继承时重写此方法） |
| `join(timeout=None)` | 等待线程完成 |
| `is_alive()` | 返回线程是否仍在运行 |
| `name` | 线程名属性（可读写） |
| `daemon` | 守护标志（必须在 start 前设置） |
| `ident` | 线程 ID（只读，操作系统层面） |
| `native_id` | 操作系统原生线程 ID（Python 3.8+） |

## 守护线程（Daemon Thread）

守护线程在**主线程退出时自动终止**，不会阻止进程退出。适用于后台监控、心跳检测等。

```python
import threading
import time

def heartbeat():
    while True:
        print("心跳...")
        time.sleep(1)

# 方式 1：构造时指定
t = threading.Thread(target=heartbeat, daemon=True)

# 方式 2：属性设置（必须在 start() 之前）
t = threading.Thread(target=heartbeat)
t.daemon = True

t.start()

# 主线程 3 秒后退出，守护线程自动终止
time.sleep(3)
print("主线程退出，守护线程随之结束")
```

**注意事项**：
- 守护线程被突然终止时，不会执行 `finally` 块或清理资源
- 如果需要确保清理，不要使用守护线程
- 非守护线程会阻止进程退出

## join 与超时

```python
import threading
import time

def slow_task():
    time.sleep(10)

t = threading.Thread(target=slow_task)
t.start()

# 等待最多 2 秒
t.join(timeout=2)

if t.is_alive():
    print("线程仍在运行，超时放弃等待")
else:
    print("线程已完成")
```

## 常见陷阱

### 线程不能 start 两次

```python
t = threading.Thread(target=lambda: None)
t.start()
t.join()
t.start()  # RuntimeError: threads can only be started once
```

### 不要在 fork 后使用锁

```python
import os
import threading

lock = threading.Lock()
lock.acquire()

pid = os.fork()  # Linux 上可用
if pid == 0:
    # 子进程继承了锁的"已锁定"状态
    lock.acquire()  # 死锁！
```

## 相关概念

- [[python/concurrency/02_threading/Thread创建方式]] — 函数式 vs 继承式
- [[python/concurrency/02_threading/Thread异常处理]] — 异常传播与 Timer
- [[python/concurrency/03_同步原语/Lock互斥锁]] — 保护共享资源

## 来源

- Python 官方文档 — Thread Objects：https://docs.python.org/3/library/threading.html#thread-objects
