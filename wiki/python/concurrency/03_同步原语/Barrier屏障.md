---
tags: [concept, Python, threading, Barrier, 屏障, 同步点]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Barrier 屏障

> Barrier 让一组线程在同一个点同步——所有线程到达屏障后才能继续执行。适用于多阶段并行计算。

## 基本概念

```
阶段 1:  线程1 ████ → 到达屏障 ⏳
        线程2 █████ → 到达屏障 ⏳
        线程3 ██ → 到达屏障 ⏳
        全部到达 → 屏障放行
阶段 2:  线程1 ████ → ...
```

## 使用场景：多阶段计算

```python
import threading
import time

NUM_THREADS = 3
barrier = threading.Barrier(NUM_THREADS)

def multi_stage(worker_id):
    for stage in range(1, 4):
        print(f"  Worker-{worker_id} 阶段 {stage} 开始")
        time.sleep(worker_id * 0.3)
        print(f"  Worker-{worker_id} 阶段 {stage} 完成，等待同步...")
        barrier.wait()
        print(f"  Worker-{worker_id} 进入阶段 {stage + 1}")

threads = [threading.Thread(target=multi_stage, args=(i,)) for i in range(NUM_THREADS)]
for t in threads: t.start()
for t in threads: t.join()
```

## 超时与 abort

```python
import threading

barrier = threading.Barrier(3, timeout=5)

def worker(worker_id):
    try:
        barrier.wait()
        print(f"Worker-{worker_id} 通过屏障")
    except threading.BrokenBarrierError:
        print(f"Worker-{worker_id} 屏障超时，放弃")
```

## Barrier API

| 方法/属性 | 说明 |
|-----------|------|
| `wait(timeout=None)` | 等待其他线程到达 |
| `reset()` | 重置屏障到初始状态 |
| `abort()` | 使屏障进入破裂状态 |
| `parties` | 需要同步的线程数 |
| `n_waiting` | 当前等待中的线程数 |
| `broken` | 屏障是否已破裂 |

## 相关概念

- [[python/concurrency/03_同步原语/同步原语选择指南]] — 各原语对比
- [[python/concurrency/08_设计模式/流水线模式]] — 多阶段处理

## 来源

- Python 官方文档 — Barrier Objects：https://docs.python.org/3/library/threading.html#barrier-objects
