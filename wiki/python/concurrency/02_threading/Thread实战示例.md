---
tags: [concept, Python, threading, 实战, 批量请求, 并发控制]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Thread 实战示例

> 本页展示 threading 在实际场景中的应用：并发批量请求和并发数控制。

## 并发批量请求

```python
import threading
import time
import random

def fetch_url(url, results, index):
    """模拟网络请求"""
    delay = random.uniform(0.5, 2.0)
    time.sleep(delay)
    results[index] = f"{url} -> {delay:.2f}s"

urls = [f"https://example.com/api/{i}" for i in range(20)]
results = [None] * len(urls)

# 串行
start = time.time()
for i, url in enumerate(urls):
    fetch_url(url, results, i)
serial_time = time.time() - start
print(f"串行: {serial_time:.2f}s")

# 并行
threads = []
for i, url in enumerate(urls):
    t = threading.Thread(target=fetch_url, args=(url, results, i))
    threads.append(t)

start = time.time()
for t in threads: t.start()
for t in threads: t.join()
parallel_time = time.time() - start
print(f"并行: {parallel_time:.2f}s")
print(f"加速比: {serial_time/parallel_time:.1f}x")
```

## 控制并发数

```python
import threading
import time
import random

class ConcurrencyLimiter:
    """使用信号量控制最大并发线程数"""

    def __init__(self, max_concurrent):
        self.semaphore = threading.Semaphore(max_concurrent)

    def run(self, func, *args):
        def wrapper():
            with self.semaphore:
                return func(*args)
        t = threading.Thread(target=wrapper)
        t.start()
        return t

def fetch(url):
    delay = random.uniform(0.5, 1.5)
    time.sleep(delay)
    print(f"  完成: {url} ({delay:.2f}s)")

urls = [f"url_{i}" for i in range(50)]
limiter = ConcurrencyLimiter(max_concurrent=5)  # 最多 5 个并发

threads = [limiter.run(fetch, url) for url in urls]
for t in threads: t.join()
print("全部完成")
```

## 相关概念

- [[python/concurrency/02_threading/Thread创建方式]] — 线程创建方法
- [[python/concurrency/03_同步原语/Semaphore信号量]] — 信号量详解
- [[python/concurrency/05_concurrent_futures/ThreadPoolExecutor详解]] — 更推荐的并发控制方式
- [[python/concurrency/08_设计模式/线程池模式]] — 线程池设计模式

## 来源

- Python 官方文档 — threading：https://docs.python.org/3/library/threading.html
