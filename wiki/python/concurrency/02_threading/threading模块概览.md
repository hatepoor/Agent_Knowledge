---
tags: [concept, Python, threading, 概览]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# threading 模块概览

> `threading` 模块是 Python 标准库中基于线程的并发实现。本页提供模块全貌和核心组件索引。

## threading vs _thread

| 维度 | `threading`（推荐） | `_thread`（低层） |
|------|---------------------|-------------------|
| 抽象层级 | 高层 OO 接口 | C 层映射 |
| Thread 类 | 有 | 无 |
| 同步原语 | Lock/RLock/Semaphore/Event/Condition/Barrier | 仅 Lock |
| 守护线程 | 支持 | 不支持 |
| 异常处理 | 原生捕获 | 崩溃进程 |
| 用途 | 99% 的多线程需求 | 极少直接使用 |

## 核心组件

```
threading 模块
├── Thread          — 线程对象
├── Lock / RLock    — 互斥锁 / 可重入锁
├── Semaphore       — 信号量（限制并发数）
├── Event           — 事件标志（线程间通知）
├── Condition       — 条件变量（等待/通知）
├── Barrier         — 屏障（线程同步点）
├── Timer           — 延迟执行线程
├── local           — 线程局部存储
├── enumerate()     — 列出所有活跃线程
├── active_count()  — 活跃线程数
├── current_thread() — 当前线程对象
└── main_thread()   — 主线程对象
```

## 子页导航

- [[python/concurrency/02_threading/Thread类与生命周期]] — Thread 类、五种状态、守护线程
- [[python/concurrency/02_threading/Thread创建方式]] — 函数式 vs 继承式、选择指南
- [[python/concurrency/02_threading/Thread异常处理]] — 异常传播、ThreadWithResult、Timer
- [[python/concurrency/02_threading/模块级API]] — enumerate/active_count/current_thread 等
- [[python/concurrency/02_threading/Thread实战示例]] — 并发批量请求、并发数控制

## 相关概念

- [[python/concurrency/01_GIL/GIL原理与工作机制]] — GIL 对多线程的影响
- [[python/concurrency/03_同步原语/同步原语选择指南]] — Lock/RLock/Semaphore 等
- [[python/concurrency/06_线程安全/queue模块详解]] — 线程安全队列

## 来源

- Python 官方文档 — `threading`：https://docs.python.org/3/library/threading.html
