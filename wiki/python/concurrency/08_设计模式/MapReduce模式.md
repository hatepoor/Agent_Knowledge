---
tags: [concept, Python, 设计模式, MapReduce, 分而治之]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# MapReduce 模式

> 分而治之：Map 阶段并行处理数据块，Shuffle 按 key 分组，Reduce 阶段聚合结果。

## 原理

```
    输入数据
    ┌──────────────────┐
    │ [k1,v1] [k2,v2]  │
    └────────┬─────────┘
             │ Map
    ┌────────┼────────┐
    ↓        ↓        ↓
  [k1,v1]  [k2,v2]  [k3,v3]
  [k1,v4]
             │ Shuffle（按 key 分组）
    ┌────────┴─────────┐
    │ k1: [v1, v4]     │
    │ k2: [v2]         │
    │ k3: [v3]         │
    └────────┬─────────┘
             │ Reduce
             ↓
    ┌──────────────────┐
    │ k1: reduced_value│
    │ k2: reduced_value│
    └──────────────────┘
```

## 实现

```python
from collections import defaultdict
from concurrent.futures import ProcessPoolExecutor
import re

def map_function(text_chunk):
    words = re.findall(r'\w+', text_chunk.lower())
    return [(word, 1) for word in words]

def reduce_function(word, counts):
    return (word, sum(counts))

def map_reduce(text_chunks, num_workers=None):
    # Map 阶段
    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        map_results = list(executor.map(map_function, text_chunks))

    # Shuffle 阶段
    shuffled = defaultdict(list)
    for chunk_pairs in map_results:
        for word, count in chunk_pairs:
            shuffled[word].append(count)

    # Reduce 阶段
    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        reduce_futures = [
            executor.submit(reduce_function, word, counts)
            for word, counts in shuffled.items()
        ]
        results = [f.result() for f in reduce_futures]

    return dict(results)

# 使用：词频统计
texts = [
    "hello world python programming",
    "python is great for data science",
    "hello python world data",
    "data data everywhere in python",
]

word_counts = map_reduce(texts, num_workers=4)
for word, count in sorted(word_counts.items(), key=lambda x: -x[1]):
    print(f"  {word}: {count}")
```

## 模式选择指南

```
你的任务结构是？
├─ 生产和消费速率不同？ → 生产者-消费者
├─ 大量同类独立任务？ → 线程池
├─ 数据需要多阶段处理？ → 流水线
├─ 分发+汇总？ → Fan-out/Fan-in
└─ 大数据需要聚合？ → MapReduce
```

## 相关概念

- [[python/concurrency/08_设计模式/Fan-out_Fan-in模式]] — MapReduce 的简化版
- [[python/concurrency/04_multiprocessing/Pool进程池]] — Pool.map 实现 Map
- [[python/concurrency/05_concurrent_futures/concurrent_futures概览]] — Executor 实现

## 来源

- Google Research — MapReduce：https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/
