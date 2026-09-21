---
tags: [concept, Python, multiprocessing, Pool, map, starmap, imap, chunksize]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Pool 进程池

> 进程池预创建一组工作进程并复用，避免频繁创建销毁进程的开销。是 multiprocessing 最常用的接口。

## 基本用法

```python
from multiprocessing import Pool
import time
import os

def cpu_bound_task(n):
    total = sum(i * i for i in range(n))
    return total

if __name__ == "__main__":
    NUM_WORKERS = os.cpu_count()

    start = time.time()
    with Pool(NUM_WORKERS) as pool:
        results = pool.map(cpu_bound_task, [5_000_000] * 8)
    print(f"并行 8 次: {time.time()-start:.2f}s")
```

## Pool 方法对比

| 方法 | 说明 | 返回值 | 阻塞 |
|------|------|--------|------|
| `map(func, iterable)` | 对每个元素执行 func | 结果列表 | 阻塞 |
| `map_async(func, iterable)` | 异步版本 | AsyncResult | 非阻塞 |
| `starmap(func, iterable)` | 多参数 map | 结果列表 | 阻塞 |
| `apply(func, args)` | 执行单个任务 | 结果 | 阻塞 |
| `apply_async(func, args)` | 异步执行单个任务 | AsyncResult | 非阻塞 |
| `imap(func, iterable, chunksize)` | 惰性 map | 迭代器 | 逐个返回 |
| `imap_unordered(...)` | 无序惰性 map | 迭代器 | 顺序不定 |

## starmap（多参数）

```python
from multiprocessing import Pool

def process_data(data, multiplier, offset):
    return data * multiplier + offset

if __name__ == "__main__":
    args_list = [(10, 2, 1), (20, 3, 5), (30, 4, 10)]
    with Pool() as pool:
        results = pool.starmap(process_data, args_list)
        print(results)  # [21, 65, 130]
```

## apply_async 与回调

```python
from multiprocessing import Pool

def slow_task(x):
    import time; time.sleep(1)
    return x * x

def callback(result):
    print(f"  回调: 结果 = {result}")

if __name__ == "__main__":
    with Pool(4) as pool:
        pool.apply_async(slow_task, (5,), callback=callback)
        import time; time.sleep(2)
```

## imap（惰性求值）

```python
# map: 全部完成后一次性返回
# imap: 结果逐个返回，任务完成一个就 yield 一个
# imap_unordered: 返回顺序不确定，但更快

with Pool(4) as pool:
    for result in pool.imap(process, items, chunksize=2):
        print(result, end=" ")

with Pool(4) as pool:
    for result in pool.imap_unordered(process, items):
        print(result, end=" ")
```

## chunksize 参数

```python
# chunksize 控制：将 iterable 分成多大的一块分给每个进程
# 任务多但单个任务快 → 增大 chunksize
# 任务少但单个任务慢 → chunksize=1，确保负载均衡

pool.map(func, range(10000), chunksize=100)
```

## 相关概念

- [[python/concurrency/04_multiprocessing/Process类详解]] — 裸 Process 对比
- [[python/concurrency/05_concurrent_futures/ProcessPoolExecutor详解]] — 更高层抽象
- [[python/concurrency/09_性能调优与调试/性能基准测试方法]] — 性能测试

## 来源

- Python 官方文档 — Pool Objects：https://docs.python.org/3/library/multiprocessing.html#module-multiprocessing.pool
