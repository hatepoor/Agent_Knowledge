---
tags: [concept, Python, threading, 异常处理, Timer, ThreadWithResult]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Thread 异常处理与 Timer

> 线程中的异常不会自动传播到主线程。本页讲解如何正确捕获线程异常、ThreadWithResult 模式和 Timer 延迟执行。

## 线程异常不自动传播

```python
import threading

def risky_task(x):
    if x < 0:
        raise ValueError("x 不能为负数")
    return x * 2

# 异常在线程中抛出，但不会传播到主线程
t = threading.Thread(target=risky_task, args=(-1,))
t.start()
t.join()
print("主线程正常结束")  # 会打印！异常被吞掉了
```

## ThreadWithResult：捕获异常并返回结果

```python
import threading
import traceback

class ThreadWithResult(threading.Thread):
    """捕获线程异常，存储返回值"""

    def __init__(self, target=None, args=(), kwargs=None):
        super().__init__(target=target, args=args, kwargs=kwargs or {})
        self._exception = None
        self._result = None

    def run(self):
        try:
            if self._target:
                self._result = self._target(*self._args, **self._kwargs)
        except Exception as e:
            self._exception = e
            traceback.print_exc()

    def result(self):
        """获取结果（join 后调用）"""
        if self._exception:
            raise self._exception
        return self._result

# 使用
def risky_task(x):
    if x < 0:
        raise ValueError("x 不能为负数")
    return x * 2

# 正常情况
t1 = ThreadWithResult(target=risky_task, args=(5,))
t1.start(); t1.join()
print(f"结果: {t1.result()}")  # 10

# 异常情况
t2 = ThreadWithResult(target=risky_task, args=(-1,))
t2.start(); t2.join()
try:
    print(t2.result())
except ValueError as e:
    print(f"捕获线程异常: {e}")  # "x 不能为负数"
```

## Timer 延迟执行

```python
from threading import Timer

def delayed_action():
    print("延迟 3 秒后执行")

# 3 秒后执行一次
timer = Timer(3.0, delayed_action)
timer.start()

# 取消（如果还未触发）
timer.cancel()
```

## 相关概念

- [[python/concurrency/02_threading/Thread类与生命周期]] — Thread 基础 API
- [[python/concurrency/09_性能调优与调试/调试工具与技巧]] — 线程调试
- [[python/concurrency/05_concurrent_futures/Future对象详解]] — Future 提供更优雅的异常处理

## 来源

- Python 官方文档 — Timer Objects：https://docs.python.org/3/library/threading.html#timer-objects
