---
tags: [concept, Python, threading, API, 模块级函数]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# threading 模块级 API

> `threading` 模块提供了多个模块级函数用于查询和控制线程状态。本页逐个讲解这些 API。

## 模块级函数

```python
import threading

# 当前线程对象
t = threading.current_thread()
print(t.name)  # "MainThread" 或 "Thread-N"

# 主线程对象
main = threading.main_thread()

# 活跃线程数
print(threading.active_count())  # 包括主线程

# 列出所有活跃线程
for t in threading.enumerate():
    print(f"  {t.name} (daemon={t.daemon})")

# 获取/设置 GIL 切换间隔
print(threading.getswitchinterval())  # 0.005
threading.setswitchinterval(0.005)
```

## API 速查表

| 函数 | 说明 |
|------|------|
| `current_thread()` | 返回当前线程对象 |
| `main_thread()` | 返回主线程对象 |
| `active_count()` | 返回活跃线程数（含主线程） |
| `enumerate()` | 列出所有活跃线程列表 |
| `getswitchinterval()` | 获取 GIL 切换间隔（秒） |
| `setswitchinterval(s)` | 设置 GIL 切换间隔 |
| `get_ident()` | 获取当前线程 ID |

## 实用示例：线程状态监控

```python
import threading
import time

def monitor_threads():
    """监控所有活跃线程状态"""
    while True:
        print(f"\n活跃线程: {threading.active_count()}")
        for t in threading.enumerate():
            print(f"  {t.name} (daemon={t.daemon}, alive={t.is_alive()})")
        time.sleep(1)
```

## 相关概念

- [[python/concurrency/02_threading/Thread类与生命周期]] — Thread 实例方法
- [[python/concurrency/09_性能调优与调试/调试工具与技巧]] — 线程调试

## 来源

- Python 官方文档 — threading module functions：https://docs.python.org/3/library/threading.html#module-threading
