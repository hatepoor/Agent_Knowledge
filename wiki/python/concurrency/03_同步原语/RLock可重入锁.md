---
tags: [concept, Python, threading, RLock, 可重入锁]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# RLock 可重入锁

> RLock（Reentrant Lock）允许同一线程多次获取同一把锁，内部维护计数器。适用于递归调用和装饰器嵌套场景。

## 为什么需要 RLock

当同一个线程需要多次获取锁时（如递归调用、装饰器嵌套），Lock 会导致死锁。RLock 解决此问题。

```python
import threading

rlock = threading.RLock()

def recursive_factorial(n):
    with rlock:
        if n <= 1:
            return 1
        return n * recursive_factorial(n - 1)  # 同一线程再次获取 RLock

# RLock 计数：acquire -> 1 -> 2 -> 3 ... -> release -> 2 -> 1 -> 0
```

## RLock vs Lock

| 维度 | Lock | RLock |
|------|------|-------|
| 同线程多次 acquire | 死锁 | 允许（计数器 +1） |
| 不同线程 acquire | 阻塞等待 | 阻塞等待 |
| 性能 | 略高 | 略低（额外计数器） |
| 所有者跟踪 | 无 | 有（记录持有线程） |
| 释放 | 任意线程可 release | 只有持有者可 release |

## 实用模式：线程安全计数器

```python
import threading

class AtomicCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.RLock()

    def increment(self):
        with self._lock:
            self._value += 1

    def decrement(self):
        with self._lock:
            self._value -= 1

    def get_and_increment(self):
        with self._lock:
            val = self._value
            self.increment()  # RLock 允许在持有锁时再次获取
            return val

    @property
    def value(self):
        with self._lock:
            return self._value
```

## 相关概念

- [[python/concurrency/03_同步原语/Lock互斥锁]] — 基础互斥锁
- [[python/concurrency/03_同步原语/同步原语选择指南]] — 选择指南

## 来源

- Python 官方文档 — RLock Objects：https://docs.python.org/3/library/threading.html#rlock-objects
