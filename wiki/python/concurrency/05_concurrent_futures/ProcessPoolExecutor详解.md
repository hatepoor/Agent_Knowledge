---
tags: [concept, Python, concurrent_futures, ProcessPoolExecutor, 进程池]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# ProcessPoolExecutor 详解

> ProcessPoolExecutor 是 CPU 密集型任务的推荐方式，API 与 ThreadPoolExecutor 完全一致，只需替换类名。

## 基本用法

```python
from concurrent.futures import ProcessPoolExecutor
import os

def cpu_bound(n):
    total = sum(i * i for i in range(n))
    return total

if __name__ == "__main__":
    NUM_WORKERS = os.cpu_count()

    with ProcessPoolExecutor(max_workers=NUM_WORKERS) as executor:
        futures = [executor.submit(cpu_bound, 5_000_000) for _ in range(8)]
        results = [f.result() for f in futures]
        print(f"完成 {len(results)} 个计算")
```

## ThreadPoolExecutor vs ProcessPoolExecutor

| 维度 | ThreadPoolExecutor | ProcessPoolExecutor |
|------|-------------------|--------------------|
| 底层 | threading.Thread | multiprocessing.Process |
| GIL | 受限 | 不受限 |
| max_workers 默认 | min(32, cpu+4) | os.cpu_count() |
| 任务序列化 | 不需要 | 需要 pickle |
| 共享内存 | 天然共享 | 需要特殊机制 |
| 启动开销 | 低 | 高 |
| 适用 | I/O 密集 | CPU 密集 |

## 混合使用

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def download(url):
    import time; time.sleep(1)
    return f"data_from_{url}"

def process_data(data):
    return sum(ord(c) for c in data * 10000)

if __name__ == "__main__":
    urls = [f"url_{i}" for i in range(8)]

    with ThreadPoolExecutor(max_workers=4) as io_pool, \
         ProcessPoolExecutor() as cpu_pool:

        # 阶段 1: 并行下载（I/O 密集用线程池）
        all_data = list(io_pool.map(download, urls))

        # 阶段 2: 并行处理（CPU 密集用进程池）
        results = list(cpu_pool.map(process_data, all_data))
        print(f"处理完成: {len(results)} 个结果")
```

## 相关概念

- [[python/concurrency/05_concurrent_futures/ThreadPoolExecutor详解]] — 线程池对比
- [[python/concurrency/04_multiprocessing/Pool进程池]] — 底层 multiprocessing.Pool
- [[python/concurrency/05_concurrent_futures/Future对象详解]] — Future API

## 来源

- Python 官方文档 — ProcessPoolExecutor：https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor
