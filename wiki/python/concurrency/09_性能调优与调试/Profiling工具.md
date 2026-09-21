---
tags: [concept, Python, profiling, cProfile, line_profiler, memory_profiler]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Profiling 工具

> 性能分析工具帮助定位热点代码和内存瓶颈。

## cProfile

```python
import cProfile
import pstats

def run_tasks():
    from concurrent.futures import ProcessPoolExecutor
    def cpu_task(n):
        return sum(i * i for i in range(n))
    with ProcessPoolExecutor() as executor:
        list(executor.map(cpu_task, [2_000_000] * 8))

profiler = cProfile.Profile()
profiler.enable()
run_tasks()
profiler.disable()

stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)
```

## line_profiler（逐行分析）

```python
# pip install line_profiler
from line_profiler import LineProfiler

def process_batch(items):
    results = []
    for item in items:
        import time; time.sleep(0.001)
        results.append(item * 2)
    return results

lp = LineProfiler()
lp_wrapper = lp(process_batch)
lp_wrapper(list(range(1000)))
lp.print_stats()

# 输出每行代码的命中次数和耗时
```

## memory_profiler

```python
# pip install memory-profiler
from memory_profiler import profile

@profile
def process_large_data():
    import numpy as np
    data = np.random.rand(10000, 10000)
    result = data.mean()
    del data
    return result
```

## 工具选择

| 工具 | 分析维度 | 适用场景 |
|------|----------|----------|
| cProfile | 函数级调用 | 整体性能概览 |
| line_profiler | 逐行耗时 | 定位热点行 |
| memory_profiler | 内存使用 | 内存泄漏 |

## 相关概念

- [[python/concurrency/09_性能调优与调试/性能基准测试方法]] — 基准测试方法
- [[python/concurrency/09_性能调优与调试/调试工具与技巧]] — 调试工具

## 来源

- Python cProfile：https://docs.python.org/3/library/profile.html
- line_profiler：https://github.com/pyutils/line_profiler
- memory_profiler：https://github.com/pythonprofilers/memory_profiler
