---
tags: [concept, Python, threading, Lock, 互斥锁, 临界区]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Lock 互斥锁

> Lock 是最基础的同步原语——一把互斥锁，同一时刻只有一个线程能持有。

## 基本概念

```
线程 A:  acquire() ✓ → 执行临界区 → release()
线程 B:  acquire() ⏳ 等待...                     → acquire() ✓ → 执行
线程 C:  acquire() ⏳ 等待...                                    → acquire() ✓ → 执行
```

## 基本用法

```python
import threading

lock = threading.Lock()
shared_data = []

def safe_append(item):
    lock.acquire()
    try:
        shared_data.append(item)  # 临界区
    finally:
        lock.release()

# 推荐使用 context manager
def safe_append_v2(item):
    with lock:
        shared_data.append(item)
```

## 为什么需要 Lock

```python
import threading

counter = 0

def unsafe_increment(n):
    global counter
    for _ in range(n):
        counter += 1  # 非原子操作！

threads = [threading.Thread(target=unsafe_increment, args=(100000,)) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"期望: {5*100000}, 实际: {counter}")
# 输出可能是 387421 而非 500000
```

## 非阻塞获取

```python
lock = threading.Lock()

# 非阻塞获取
if lock.acquire(blocking=False):
    try:
        print("获取成功")
    finally:
        lock.release()
else:
    print("锁被占用，跳过")

# 带超时获取
if lock.acquire(timeout=2.0):
    try:
        print("2 秒内获取成功")
    finally:
        lock.release()
else:
    print("2 秒内未获取到锁")
```

## Lock API

| 方法 | 说明 |
|------|------|
| `acquire(blocking=True, timeout=-1)` | 获取锁 |
| `release()` | 释放锁 |
| `locked()` | 返回锁是否被持有 |
| `__enter__` / `__exit__` | 上下文管理器支持 |

**注意**：Lock 不可重入——同一线程第二次 acquire 会死锁：

```python
lock = threading.Lock()
lock.acquire()
lock.acquire()  # 死锁！
```

## 相关概念

- [[python/concurrency/03_同步原语/RLock可重入锁]] — 可重入锁解决同线程多次获取
- [[python/concurrency/06_线程安全/竞态条件详解]] — 为什么需要锁
- [[python/concurrency/03_同步原语/同步原语选择指南]] — 各原语对比

## 来源

- Python 官方文档 — Lock Objects：https://docs.python.org/3/library/threading.html#lock-objects
