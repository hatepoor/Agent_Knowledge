---
tags: [concept, Python, threading.local, 线程隔离, 线程局部存储]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# threading.local 线程隔离

> `threading.local` 实现线程局部存储——每个线程有独立的属性副本，无需加锁。

## 基本用法

```python
import threading
import time
import random

local_data = threading.local()

def worker(worker_id):
    local_data.worker_id = worker_id  # 每个线程有独立的属性
    local_data.counter = 0

    for i in range(5):
        local_data.counter += 1
        time.sleep(random.uniform(0, 0.1))
        print(f"  Worker-{local_data.worker_id}: counter={local_data.counter}")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(3)]
for t in threads: t.start()
for t in threads: t.join()

# 主线程没有 worker_id
print(hasattr(local_data, 'worker_id'))  # False
```

## 使用场景

### 场景 1: 请求上下文

```python
import threading

log_context = threading.local()

class RequestContext:
    def __init__(self, request_id):
        self.request_id = request_id

    def __enter__(self):
        log_context.request_id = self.request_id

    def __exit__(self, *args):
        if hasattr(log_context, 'request_id'):
            del log_context.request_id

def get_current_request_id():
    return getattr(log_context, 'request_id', 'unknown')
```

### 场景 2: 每线程独立数据库连接

```python
import threading

db_connections = threading.local()

def get_db_connection():
    if not hasattr(db_connections, 'conn'):
        db_connections.conn = create_db_connection()
    return db_connections.conn

# 每个线程有独立的连接，不需要加锁
```

## 线程安全的单例模式

```python
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        # 双重检查锁定（Double-Checked Locking）
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

# 更 Pythonic 的方式：使用模块级变量
# Python 模块在第一次导入时执行，GIL 保证线程安全
```

## 注意事项

```python
# threading.local 不会被子线程继承
import threading

local = threading.local()
local.value = "parent"

def child():
    print(hasattr(local, 'value'))  # False

t = threading.Thread(target=child)
t.start(); t.join()
```

## 相关概念

- [[python/concurrency/06_线程安全/线程安全与原子操作]] — 线程安全基础
- [[python/concurrency/03_同步原语/Lock互斥锁]] — 另一种保证线程安全的方式

## 来源

- Python 官方文档 — Thread-local data：https://docs.python.org/3/library/threading.html#thread-local-data
