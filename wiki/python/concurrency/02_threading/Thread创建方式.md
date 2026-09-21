---
tags: [concept, Python, threading, Thread创建, 函数式, 继承式]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# Thread 创建方式

> 创建线程有两种方式：函数式（target 参数）和继承式（重写 run）。本页对比两种方式并给出选择指南。

## 方式一：函数式（target 参数）

```python
import threading
import time

def worker(name, delay):
    print(f"[{name}] 开始工作")
    time.sleep(delay)
    print(f"[{name}] 完成，耗时 {delay}s")

# 创建线程
t = threading.Thread(
    target=worker,
    args=("Thread-A", 2),
    name="worker-1"
)

t.start()
t.join()
print("主线程结束")
```

## 方式二：继承 Thread 类

```python
import threading

class DataProcessor(threading.Thread):
    def __init__(self, data_source, result_queue):
        super().__init__()  # 必须调用父类 __init__
        self.data_source = data_source
        self.result_queue = result_queue
        self._stop_event = threading.Event()

    def run(self):
        """重写 run()，start() 会调用此方法"""
        for item in self.data_source:
            if self._stop_event.is_set():
                print(f"[{self.name}] 收到停止信号，退出")
                return
            processed = self.process(item)
            self.result_queue.put(processed)

    def process(self, item):
        import time
        time.sleep(0.1)
        return item * 2

    def stop(self):
        self._stop_event.set()

# 使用
import queue
results = queue.Queue()
processor = DataProcessor(range(10), results)
processor.start()
processor.join()

while not results.empty():
    print(results.get())
```

## 选择指南

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 简单一次性任务 | 函数式 | 简洁直接 |
| 需要封装状态 | 继承式 | 状态封装在对象内 |
| 需要自定义生命周期 | 继承式 | 可重写 run/stop |
| 快速并行批处理 | 函数式 + ThreadPoolExecutor | 详见 [[python/concurrency/05_concurrent_futures/concurrent_futures概览]] |

## 闭包参数陷阱

```python
import threading

# 错误：闭包延迟绑定
threads = []
for i in range(5):
    t = threading.Thread(target=lambda: print(i))
    threads.append(t)

for t in threads: t.start()
# 可能全部输出 4

# 正确：通过 args 传递
for i in range(5):
    t = threading.Thread(target=lambda x: print(x), args=(i,))
    t.start()
```

## 相关概念

- [[python/concurrency/02_threading/Thread类与生命周期]] — Thread 类 API 与状态
- [[python/concurrency/05_concurrent_futures/ThreadPoolExecutor详解]] — 更高层抽象

## 来源

- Python 官方文档 — Thread Objects：https://docs.python.org/3/library/threading.html#thread-objects
