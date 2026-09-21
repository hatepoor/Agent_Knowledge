---
tags: [concept, Python, multiprocessing, 概览, 多进程]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# multiprocessing 模块概览

> `multiprocessing` 模块通过创建独立进程实现真正的 CPU 并行计算，每个进程有自己的 Python 解释器和内存空间，不受 GIL 约束。

## 多进程 vs 多线程

| 维度 | multiprocessing | threading |
|------|----------------|-----------|
| GIL 限制 | 不受限 | 受限 |
| 并行能力 | 真正并行 | 仅 I/O 并发 |
| 内存 | 独立空间，不共享 | 共享进程内存 |
| 通信成本 | 高（需要 IPC） | 低（直接共享变量） |
| 启动开销 | ~10-100ms | ~0.01ms |
| 稳定性 | 进程崩溃不影响主进程 | 线程崩溃可能导致整个进程崩溃 |
| 适用场景 | CPU 密集型 | I/O 密集型 |

## 子页导航

- [[python/concurrency/04_multiprocessing/Process类详解]] — Process 创建、API、退出码
- [[python/concurrency/04_multiprocessing/Pool进程池]] — Pool、map/starmap/apply/imap
- [[python/concurrency/04_multiprocessing/进程间通信]] — Queue vs Pipe
- [[python/concurrency/04_multiprocessing/共享内存]] — Value/Array/Manager/SharedMemory
- [[python/concurrency/04_multiprocessing/进程启动方式]] — spawn/fork/forkserver

## 相关概念

- [[python/concurrency/01_GIL/GIL原理与工作机制]] — GIL 是使用多进程的根源
- [[python/concurrency/05_concurrent_futures/ProcessPoolExecutor详解]] — 更高层抽象
- [[python/concurrency/07_选型对比/选型决策树]] — 何时使用多进程

## 来源

- Python 官方文档 — `multiprocessing`：https://docs.python.org/3/library/multiprocessing.html
