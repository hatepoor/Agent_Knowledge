---
tags: [concept, Python, concurrent_futures, as_completed, wait, FIRST_COMPLETED]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# as_completed 与 wait

> `as_completed` 和 `wait` 是批量管理 Future 的两个核心函数。`as_completed` 按完成顺序迭代，`wait` 等待一组 Future 达到条件。

## as_completed（先完成先处理）

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time
import random

def task(task_id):
    delay = random.uniform(0.5, 3.0)
    time.sleep(delay)
    return f"任务-{task_id} ({delay:.2f}s)"

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {executor.submit(task, i): i for i in range(10)}

    for future in as_completed(futures):
        task_id = futures[future]
        try:
            result = future.result()
            print(f"  完成: {result}")
        except Exception as e:
            print(f"  失败: 任务-{task_id}, 错误: {e}")
```

**`as_completed` vs `map` 的区别**：

```
map:                 as_completed:
任务1 ████ 3.0s      任务3 █ 0.5s → 先返回
任务2 ███ 2.5s       任务7 ██ 1.0s → 第二个返回
任务3 █ 0.5s         任务1 ████ 3.0s → 最后返回
按提交顺序返回        按完成顺序返回
```

## wait（等待多个 Future）

```python
from concurrent.futures import ThreadPoolExecutor, wait, FIRST_COMPLETED
import time

def slow_task(n):
    time.sleep(n)
    return n

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(slow_task, n) for n in [3, 1, 2, 5, 4]]

    # 等到至少一个完成
    done, not_done = wait(futures, return_when=FIRST_COMPLETED)
    print(f"第一个完成: {[f.result() for f in done]}")
    print(f"还在执行: {len(not_done)}")

    # 带超时
    done, not_done = wait(futures, timeout=2.0)
    print(f"2 秒内完成: {len(done)}")
```

## return_when 选项

| 值 | 含义 |
|------|------|
| `ALL_COMPLETED`（默认） | 等待所有 Future 完成 |
| `FIRST_COMPLETED` | 等到第一个 Future 完成就返回 |
| `FIRST_EXCEPTION` | 等到第一个异常或全部完成 |

## 实战：带进度条的任务批处理

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def process_item(item):
    time.sleep(0.1)
    return f"processed_{item}"

items = list(range(50))

with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(process_item, item) for item in items]
    completed = 0
    total = len(futures)

    for future in as_completed(futures):
        completed += 1
        bar_len = 30
        filled = int(bar_len * completed / total)
        bar = '█' * filled + '░' * (bar_len - filled)
        print(f"\r[{bar}] {completed}/{total}", end='', flush=True)
    print()
```

## 相关概念

- [[python/concurrency/05_concurrent_futures/Future对象详解]] — Future API
- [[python/concurrency/08_设计模式/Fan-out_Fan-in模式]] — as_completed 实现 Fan-in
- [[python/advanced/异步编程]] — asyncio.gather 的对比

## 来源

- Python 官方文档 — wait 和 as_completed：https://docs.python.org/3/library/concurrent.futures.html#functions
