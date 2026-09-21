---
tags: [concept, Python, GIL, multiprocessing, C扩展, NumPy]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# 绕过 GIL 的策略

> 在 PEP 703 尚未稳定的时代（Python 3.12 及以下），以下方法可以绕过 GIL 实现真正的 CPU 并行。

## 策略一：multiprocessing 多进程

每个进程有独立的 Python 解释器和 GIL，可以真正并行。

```python
from multiprocessing import Pool

def cpu_bound(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with Pool(4) as p:
        results = p.map(cpu_bound, [30_000_000] * 4)
    # 每个进程有独立 GIL，可真正并行
```

详见 [[python/concurrency/04_multiprocessing/multiprocessing概览]]。

## 策略二：C 扩展释放 GIL

NumPy 等库在内部通过 `Py_BEGIN_ALLOW_THREADS` 释放 GIL，利用多核并行。

```python
# NumPy 内部已释放 GIL
import numpy as np
import threading

# 大型矩阵运算在 C 层面释放 GIL，可利用多核
a = np.random.rand(10000, 10000)
threads = [threading.Thread(target=lambda: np.dot(a, a)) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
# NumPy 使用 BLAS 多线程后端，无需 Python 层面并行
```

## 策略三：ctypes / subprocess

```python
import ctypes
import threading

# ctypes 调用 C 库时，可以释放 GIL
lib = ctypes.CDLL("./mycompute.so")
threads = [threading.Thread(target=lib.heavy_compute) for _ in range(4)]
```

## 策略四：run_in_executor

在 asyncio 中将 CPU 密集型任务委托给进程池：

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def main():
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10_000_000)

asyncio.run(main())
```

## 策略选择指南

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| multiprocessing | 通用 CPU 并行 | 标准库，无依赖 | 进程开销大，IPC 复杂 |
| C 扩展（NumPy） | 数值计算 | 性能最优 | 需要专用库 |
| ctypes | 已有 C 库 | 直接调用 | 需要编译 C 代码 |
| run_in_executor | asyncio + CPU | 与异步集成 | 本质还是进程池 |

## 相关概念

- [[python/concurrency/01_GIL/GIL原理与工作机制]] — GIL 的基本原理
- [[python/concurrency/04_multiprocessing/multiprocessing概览]] — 多进程方案
- [[python/concurrency/05_concurrent_futures/concurrent_futures概览]] — Executor 方案

## 来源

- Python 官方文档 — GIL：https://docs.python.org/3/glossary.html#term-GIL
- Python 并发编程 HOWTO：https://docs.python.org/3/howto/sockets.html
