---
tags: [concept, Python, GIL, 性能, 基准测试]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# GIL 性能影响与量化实验

> GIL 对 Python 并发性能的影响是决定性的。本页通过量化实验展示 GIL 在 CPU 密集型和 I/O 密集型任务下的实际表现。

## CPU 密集型：多线程几乎无加速

```python
import threading
import time
import os

def cpu_bound(n):
    """纯 CPU 计算"""
    total = 0
    for i in range(n):
        total += i * i
    return total

N = 30_000_000
NUM_CORES = os.cpu_count()

# === 单线程 ===
start = time.time()
for _ in range(4):
    cpu_bound(N)
single = time.time() - start
print(f"单线程 4 次: {single:.2f}s")

# === 4 线程（4 核机器）===
start = time.time()
threads = [threading.Thread(target=cpu_bound, args=(N,)) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
multi = time.time() - start
print(f"4 线程: {multi:.2f}s")
print(f"加速比: {single/multi:.2f}x (理论: {4}x)")
```

典型输出（8 核机器）：
```
单线程 4 次: 8.21s
4 线程: 7.95s
加速比: 1.03x (理论: 4x)
```

**结论**：在 GIL 限制下，多线程在 CPU 密集型任务上几乎没有加速效果，甚至可能因线程切换开销而变慢。

## I/O 密集型：多线程接近线性加速

```python
import threading
import time

def io_bound(url):
    """模拟网络 I/O"""
    time.sleep(0.5)  # 模拟网络延迟

# === 单线程 ===
start = time.time()
for i in range(10):
    io_bound(f"url_{i}")
single = time.time() - start
print(f"单线程 10 次: {single:.2f}s")  # ~5s

# === 10 线程 ===
start = time.time()
threads = [threading.Thread(target=io_bound, args=(f"url_{i}",)) for i in range(10)]
for t in threads: t.start()
for t in threads: t.join()
multi = time.time() - start
print(f"10 线程: {multi:.2f}s")  # ~0.5s

print(f"加速比: {single/multi:.1f}x")
```

**结论**：I/O 密集型任务下，多线程可以获得接近线性的加速比。

## 性能影响总结

| 任务类型 | GIL 影响 | 多线程加速比 | 推荐 |
|----------|----------|-------------|------|
| 纯 CPU 计算 | 严重受限 | ~1x | multiprocessing |
| I/O 密集（网络/文件） | I/O 时释放 GIL | 接近线性 | threading |
| 混合型 | 计算部分受限 | 部分加速 | 视比例而定 |
| C 扩展计算 | 扩展内释放 GIL | 可并行 | NumPy 等专用库 |

## 相关概念

- [[python/concurrency/01_GIL/GIL原理与工作机制]] — GIL 的工作原理
- [[python/concurrency/09_性能调优与调试/性能基准测试方法]] — 系统化的基准测试方法
- [[python/concurrency/07_选型对比/性能基准对比]] — 三种模型的性能对比

## 来源

- Python 官方文档 — GIL：https://docs.python.org/3/glossary.html#term-GIL
- Python 并发编程 HOWTO：https://docs.python.org/3/howto/sockets.html
