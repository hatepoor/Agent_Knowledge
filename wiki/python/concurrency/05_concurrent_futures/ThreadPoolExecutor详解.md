---
tags: [concept, Python, concurrent_futures, ThreadPoolExecutor, 线程池]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# ThreadPoolExecutor 详解

> ThreadPoolExecutor 是 I/O 密集型任务的推荐方式，提供池化线程管理和统一的结果获取接口。

## 基本用法

```python
from concurrent.futures import ThreadPoolExecutor
import time
import random

def fetch_url(url):
    delay = random.uniform(0.5, 2.0)
    time.sleep(delay)
    return f"{url} -> {delay:.2f}s"

urls = [f"https://example.com/{i}" for i in range(10)]

with ThreadPoolExecutor(max_workers=5) as executor:
    # 方式 1: map（按提交顺序返回）
    results = executor.map(fetch_url, urls)
    for result in results:
        print(result)

    # 方式 2: submit（返回 Future 对象）
    futures = [executor.submit(fetch_url, url) for url in urls]
    for future in futures:
        print(future.result())
```

## max_workers 默认值

```python
import os

# ThreadPoolExecutor 默认: min(32, os.cpu_count() + 4)
# 原因：I/O 密集型任务线程数可远超 CPU 核数
# 但超过 32 个线程可能导致上下文切换开销过大

# 显式设置
with ThreadPoolExecutor(max_workers=10) as ex:
    ...
```

## 上下文管理器

```python
# with 语句确保退出时关闭池并等待所有任务完成
with ThreadPoolExecutor(max_workers=4) as executor:
    for i in range(10):
        executor.submit(task, i)
# with 块结束时自动 shutdown(wait=True)

# 等价于
executor = ThreadPoolExecutor(max_workers=4)
try:
    for i in range(10):
        executor.submit(task, i)
finally:
    executor.shutdown(wait=True)
```

## 相关概念

- [[python/concurrency/05_concurrent_futures/Future对象详解]] — Future API
- [[python/concurrency/05_concurrent_futures/as_completed与wait]] — 批量等待
- [[python/concurrency/05_concurrent_futures/ProcessPoolExecutor详解]] — 对比进程池

## 来源

- Python 官方文档 — ThreadPoolExecutor：https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor
