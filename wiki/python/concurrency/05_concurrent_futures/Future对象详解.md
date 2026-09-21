---
tags: [concept, Python, concurrent_futures, Future, 回调]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Future 对象详解

> Future 是异步结果的占位对象，提供结果获取、异常处理和回调机制。

## 基本用法

```python
from concurrent.futures import ThreadPoolExecutor

def compute(x):
    import time; time.sleep(1)
    if x < 0:
        raise ValueError(f"x 不能为负: {x}")
    return x ** 2

with ThreadPoolExecutor(max_workers=3) as executor:
    f1 = executor.submit(compute, 5)
    f2 = executor.submit(compute, -1)

    print(f1.done())        # False（可能还没完成）
    print(f1.result())      # 25（阻塞等待）

    try:
        print(f2.result())
    except ValueError as e:
        print(f"捕获异常: {e}")

    print(f2.exception())  # ValueError(...)
```

## Future 全量 API

```python
future.done()        # 是否完成
future.running()    # 是否正在执行
future.cancelled()  # 是否被取消
future.cancel()     # 尝试取消（未开始时可成功）

future.result(timeout=None)     # 获取结果（阻塞）
future.exception(timeout=None)  # 获取异常（阻塞）
future.add_done_callback(fn)    # 添加完成回调
```

## 回调机制

```python
def on_complete(future):
    try:
        result = future.result()
        print(f"  回调: {result}")
    except Exception as e:
        print(f"  回调错误: {e}")

f = executor.submit(compute, 42)
f.add_done_callback(on_complete)  # 完成后自动调用
```

## 常见陷阱

### 异常不会自动传播

```python
from concurrent.futures import ThreadPoolExecutor

def risky():
    raise RuntimeError("something went wrong")

with ThreadPoolExecutor() as ex:
    future = ex.submit(risky)
    # 异常不会自动传播，必须显式调用 result()
    try:
        future.result()
    except RuntimeError as e:
        print(f"捕获: {e}")
```

### 不要在 lambda 中捕获循环变量

```python
# 错误：闭包延迟绑定，可能全部打印 4
futures = [ex.submit(lambda: print(i)) for i in range(5)]

# 正确：通过参数传递
futures = [ex.submit(lambda x: print(x), i) for i in range(5)]
```

## 相关概念

- [[python/concurrency/05_concurrent_futures/concurrent_futures概览]] — 模块概览
- [[python/concurrency/05_concurrent_futures/as_completed与wait]] — 批量等待 Future
- [[python/concurrency/02_threading/Thread异常处理]] — 与裸线程的异常处理对比

## 来源

- Python 官方文档 — Future Objects：https://docs.python.org/3/library/concurrent.futures.html#future-objects
