---
tags: [concept, Python, threading, Condition, 条件变量, wait, notify]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Condition 条件变量

> Condition 是最灵活的同步原语——线程可以等待某个条件满足，其他线程在条件满足后通知等待的线程。是生产者-消费者模式的核心。

## 基本概念

```
生产者-消费者模型：
  消费者: 条件不满足 → wait() → 释放锁并阻塞
  生产者: 生产数据 → notify() → 唤醒一个等待的消费者
  消费者: 被唤醒 → 重新获取锁 → 检查条件 → 执行
```

## 完整实现：生产者-消费者

```python
import threading
import time
import random

buffer = []
MAX_SIZE = 5
condition = threading.Condition()

def producer():
    for i in range(20):
        with condition:
            while len(buffer) >= MAX_SIZE:
                print(f"  [生产者] 缓冲区满，等待...")
                condition.wait()

            item = f"产品-{i}"
            buffer.append(item)
            print(f"  [生产者] 生产 {item}，缓冲区大小: {len(buffer)}")
            condition.notify()

        time.sleep(random.uniform(0.01, 0.1))

def consumer(consumer_id):
    for _ in range(10):
        with condition:
            while not buffer:
                print(f"  [消费者-{consumer_id}] 缓冲区空，等待...")
                condition.wait()

            item = buffer.pop(0)
            print(f"  [消费者-{consumer_id}] 消费 {item}，缓冲区大小: {len(buffer)}")
            condition.notify()

        time.sleep(random.uniform(0.01, 0.1))

p = threading.Thread(target=producer)
c1 = threading.Thread(target=consumer, args=(1,))
c2 = threading.Thread(target=consumer, args=(2,))
p.start(); c1.start(); c2.start()
p.join(); c1.join(); c2.join()
```

## wait/notify 的细节

```python
cond = threading.Condition()

# wait 的完整行为
with cond:
    # 1. 释放锁
    # 2. 阻塞等待 notify/notify_all
    # 3. 被唤醒后重新获取锁
    # 4. 从 wait 返回
    cond.wait(timeout=5)

# notify vs notify_all
cond.notify()      # 唤醒一个等待线程（随机选择）
cond.notify_all()  # 唤醒所有等待线程
```

## 为什么用 while 检查条件

```python
with condition:
    while not condition_satisfied:
        condition.wait()
    # 执行操作
```

**为什么不用 if**：

1. **虚假唤醒（Spurious Wakeup）**：某些 OS 实现中，wait 可能被无理由唤醒
2. **竞争条件**：notify 唤醒两个线程，但只有一个能获取锁。第一个线程消费数据后，第二个线程条件已不满足

## 相关概念

- [[python/concurrency/03_同步原语/Lock互斥锁]] — Condition 内部包含一把锁
- [[python/concurrency/08_设计模式/生产者消费者模式]] — Condition 的经典应用
- [[python/concurrency/03_同步原语/同步原语选择指南]] — 各原语对比

## 来源

- Python 官方文档 — Condition Objects：https://docs.python.org/3/library/threading.html#condition-objects
