---
tags: [concept, Python, concurrent_futures, Executor, 概览]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# concurrent.futures 概览

> `concurrent.futures` 提供线程和进程的统一高层接口，是 Python 3.2+ 推荐的并发执行方式。

## 设计理念

用统一的 API 封装线程池和进程池，让开发者专注于"做什么"而非"怎么做"。

```
concurrent.futures
├── Executor (ABC)           — 抽象基类
│   ├── ThreadPoolExecutor   — 线程池实现
│   └── ProcessPoolExecutor  — 进程池实现
├── Future                   — 异步结果的占位对象
├── wait(fs)                 — 等待一组 Future 完成
└── as_completed(fs)         — 按完成顺序迭代 Future
```

## 为什么用 Executor 而非裸 Thread/Process

| 维度 | 裸 Thread/Process | Executor |
|------|-------------------|----------|
| 生命周期管理 | 手动 start/join | 自动管理（with 语句） |
| 结果获取 | 需自己封装 | Future 对象自动返回 |
| 异常传播 | 不自动传播 | Future.exception() |
| 资源限制 | 无内置限制 | max_workers 控制并发 |
| 批量任务 | 手动管理 | map/submit 批量提交 |

## 子页导航

- [[python/concurrency/05_concurrent_futures/ThreadPoolExecutor详解]] — 线程池
- [[python/concurrency/05_concurrent_futures/ProcessPoolExecutor详解]] — 进程池
- [[python/concurrency/05_concurrent_futures/Future对象详解]] — Future API 与回调
- [[python/concurrency/05_concurrent_futures/as_completed与wait]] — 批量等待

## 相关概念

- [[python/concurrency/02_threading/threading模块概览]] — 底层 threading
- [[python/concurrency/04_multiprocessing/multiprocessing概览]] — 底层 multiprocessing
- [[python/advanced/异步编程]] — asyncio + Executor 集成

## 来源

- Python 官方文档 — `concurrent.futures`：https://docs.python.org/3/library/concurrent.futures.html
- PEP 3148 — futures：https://peps.python.org/pep-3148/
