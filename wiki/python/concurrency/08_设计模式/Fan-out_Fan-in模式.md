---
tags: [concept, Python, 设计模式, Fan-out, Fan-in, as_completed]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Fan-out / Fan-in 模式

> 将任务分发到多个工作单元并行执行（Fan-out），然后收集所有结果（Fan-in）。

## 原理

```
            ┌──────────┐
            │ 任务分发器 │
            └─────┬────┘
         ┌────────┼────────┐
         ↓        ↓        ↓      Fan-out（分发）
    ┌──────┐ ┌──────┐ ┌──────┐
    │Worker1│ │Worker2│ │Worker3│
    └──┬───┘ └──┬───┘ └──┬───┘
         │        │        │
         └────────┼────────┘
                  ↓               Fan-in（汇总）
            ┌──────────┐
            │ 结果收集器 │
            └──────────┘
```

## 实现

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time
import random

def fan_out_fan_in(tasks, worker_func, num_workers=4):
    results = []
    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        # Fan-out
        future_map = {executor.submit(worker_func, task): task for task in tasks}
        # Fan-in
        for future in as_completed(future_map):
            try:
                results.append(future.result())
            except Exception as e:
                print(f"  任务失败: {e}")
    return results

def process_data(data_chunk):
    time.sleep(random.uniform(0.5, 1.5))
    return sum(data_chunk)

data_chunks = [list(range(i*100, (i+1)*100)) for i in range(8)]
results = fan_out_fan_in(data_chunks, process_data, num_workers=4)
print(f"总计: {sum(results)}")
```

## 进程版

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def cpu_intensive(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        futures = [executor.submit(cpu_intensive, 5_000_000) for _ in range(8)]
        results = [f.result() for f in as_completed(futures)]
```

## 相关概念

- [[python/concurrency/05_concurrent_futures/as_completed与wait]] — as_completed 实现 Fan-in
- [[python/concurrency/08_设计模式/线程池模式]] — Fan-out 的基础
- [[python/concurrency/08_设计模式/MapReduce模式]] — Fan-out/Fan-in 的特化

## 来源

- Java Concurrency in Practice：https://jcip.net/
