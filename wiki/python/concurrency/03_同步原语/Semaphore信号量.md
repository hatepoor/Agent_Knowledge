---
tags: [concept, Python, threading, Semaphore, BoundedSemaphore, 信号量]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Semaphore 信号量

> Semaphore 是"许可计数器"——线程获取许可后才能执行，执行完归还许可。适用于限制并发连接数。

## 基本概念

```
Semaphore(3)：允许 3 个线程同时持有

许可数: 3
线程 A: acquire() → 许可数 2 → 执行...
线程 B: acquire() → 许可数 1 → 执行...
线程 C: acquire() → 许可数 0 → 执行...
线程 D: acquire() → ⏳ 等待...
线程 A: release() → 许可数 1 → 线程 D 获取许可 → 执行...
```

## 使用场景：限制并发连接数

```python
import threading
import time
import random

sem = threading.Semaphore(3)  # 最多 3 个并发

def fetch_with_limit(url):
    with sem:
        print(f"  开始: {url}")
        time.sleep(random.uniform(0.5, 1.5))
        print(f"  完成: {url}")

urls = [f"url_{i}" for i in range(10)]
threads = [threading.Thread(target=fetch_with_limit, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
```

## BoundedSemaphore

```python
import threading

# 普通 Semaphore：release 超过初始值不会报错
sem = threading.Semaphore(2)
sem.release()  # 值 = 3，超出但不报错

# BoundedSemaphore：release 超过初始值会报错
bsem = threading.BoundedSemaphore(2)
bsem.release()  # ValueError: released too many times
```

**推荐**：总是使用 `BoundedSemaphore`，可以帮你发现 acquire/release 不匹配的 bug。

## 相关概念

- [[python/concurrency/03_同步原语/Lock互斥锁]] — Lock 是 Semaphore(1) 的特例
- [[python/concurrency/03_同步原语/同步原语选择指南]] — 各原语对比
- [[python/concurrency/02_threading/Thread实战示例]] — 信号量控制并发数

## 来源

- Python 官方文档 — Semaphore Objects：https://docs.python.org/3/library/threading.html#semaphore-objects
