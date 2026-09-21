---
tags: [concept, Python, queue, Queue, LifoQueue, PriorityQueue, SimpleQueue]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# queue 模块详解

> `queue` 模块提供线程安全的数据结构，是多线程编程中最常用的通信方式。

## 四种队列类型

```python
from queue import Queue, LifoQueue, PriorityQueue, SimpleQueue

# Queue: FIFO（先进先出）—— 最常用
q = Queue(maxsize=10)
q.put('item1')
q.get()  # 'item1'

# LifoQueue: LIFO（后进先出，栈）
lq = LifoQueue()
lq.put(1); lq.put(2); lq.put(3)
lq.get()  # 3

# PriorityQueue: 优先级队列
pq = PriorityQueue()
pq.put((3, "低优先级"))
pq.put((1, "高优先级"))
pq.get()  # (1, "高优先级")

# SimpleQueue: 轻量队列（无 maxsize、无 task_done/join）
sq = SimpleQueue()
sq.put('item')
sq.get()
```

## Queue API

```python
q = Queue(maxsize=10)
q.put('item')                # 入队（满时阻塞）
q.put('item', block=False)   # 入队（满时抛 queue.Full）
q.put('item', timeout=2)     # 入队（超时抛 queue.Full）
q.get()                      # 出队（空时阻塞）
q.get(block=False)           # 出队（空时抛 queue.Empty）
q.get(timeout=2)             # 出队（超时抛 queue.Empty）
q.qsize()                    # 当前大小
q.empty()                    # 是否为空
q.full()                     # 是否已满
q.task_done()                # 标记一个任务完成
q.join()                     # 等待所有任务完成
```

## 类型对比

| 类型 | 顺序 | maxsize | task_done/join | 适用场景 |
|------|------|---------|----------------|----------|
| `Queue` | FIFO | 支持 | 支持 | 通用生产者-消费者 |
| `LifoQueue` | LIFO | 支持 | 支持 | 栈式处理 |
| `PriorityQueue` | 优先级 | 支持 | 支持 | 任务调度 |
| `SimpleQueue` | FIFO | 不支持 | 不支持 | 高性能简单队列 |

## task_done 与 join

```python
from queue import Queue
import threading
import time

q = Queue()

def worker():
    while True:
        item = q.get()
        if item is None:
            break
        time.sleep(0.1)
        print(f"  完成: {item}")
        q.task_done()  # 标记完成

t = threading.Thread(target=worker, daemon=True)
t.start()

for i in range(5):
    q.put(f"任务-{i}")

q.join()  # 等待所有 task_done 被调用
print("所有任务完成")
```

## 优先级队列实战

```python
from queue import PriorityQueue

class Task:
    def __init__(self, priority, data):
        self.priority = priority
        self.data = data

    def __lt__(self, other):
        return self.priority < other.priority

pq = PriorityQueue()
pq.put(Task(3, "普通任务"))
pq.put(Task(1, "紧急任务"))
pq.put(Task(2, "中等任务"))

while not pq.empty():
    print(pq.get().data)  # 紧急 → 中等 → 普通
```

## multiprocessing.Queue vs queue.Queue

| 维度 | `queue.Queue` | `multiprocessing.Queue` |
|------|---------------|------------------------|
| 通信范围 | 同一进程内的线程 | 不同进程 |
| 数据传递 | 直接内存引用 | pickle 序列化 + 管道 |
| 性能 | 高 | 较低 |
| 可 pickle | 不需要 | 需要 |

## 相关概念

- [[python/concurrency/08_设计模式/生产者消费者模式]] — Queue 的经典应用
- [[python/concurrency/04_multiprocessing/进程间通信]] — 进程间 Queue
- [[python/concurrency/06_线程安全/线程安全与原子操作]] — Queue 为什么线程安全

## 来源

- Python 官方文档 — `queue`：https://docs.python.org/3/library/queue.html
