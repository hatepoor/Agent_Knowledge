---
tags: [concept, Python, GIL, PEP703, 自由线程, 历史]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# GIL 历史演进与 PEP 703

> GIL 从 1992 年诞生到 PEP 703 提出自由线程，经历了三十年的演进。本页梳理 GIL 的历史脉络和自由线程的未来。

## GIL 历史年表

| 时间 | 版本/PEP | 事件 | 意义 |
|------|----------|------|------|
| 1992 | Python 0.9 | GIL 引入 | CPython 获得多线程支持 |
| 2003 | Python 2.3 | `sys.setcheckinterval()` | 基于字节码计数切换 GIL |
| 2010 | Python 3.2 | PEP 3144 | GIL 切换从 tick 改为时间片（5ms），减少竞争 |
| 2015 | — | PEP 518 / Gilectomy 实验 | Larry Hastings 尝试移除 GIL，性能下降约 40% |
| 2023 | Python 3.13 | PEP 703 | 实验性自由线程（No-GIL），`--disable-gil` |
| 2026+ | Python 3.15+ | 自由线程稳定化 | 预期成为正式特性 |

## PEP 703：自由线程（No-GIL）

### 核心改动

PEP 703 由 Sam Gross 于 2023 年提出，使 GIL 成为**可选**特性：

```
关键改动：
1. 引用计数改为线程安全的原子操作（atomic refcount）
2. 对象的 ob_refcnt 从 Py_ssize_t 改为 uint32_t（原子操作需要 32 位）
3. 偏序垃圾回收器替代部分分代回收
4. 容器类型（dict/list/set）的内部操作加细粒度锁
5. C 扩展需要显式标记 GIL 兼容性（Py_mod_gil slot）
```

### 使用方式（Python 3.13+）

```python
import sys

# 检查是否启用了自由线程
print(sys._is_gil_enabled())  # True = GIL 开启, False = 自由线程模式

# Python 3.13 启动自由线程（需要特殊构建版）
# 命令行: python -X gil=0  (在自由线程构建版上)
# 或环境变量: PYTHON_GIL=0
```

### 自由线程下的性能

```python
# 同样的 CPU 密集型测试（Python 3.13 No-GIL）
import threading
import time

def cpu_bound(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

# 自由线程模式下，多线程可以真正并行
threads = [threading.Thread(target=cpu_bound, args=(30_000_000,)) for _ in range(4)]
start = time.time()
for t in threads: t.start()
for t in threads: t.join()
print(f"No-GIL 4 线程: {time.time()-start:.2f}s")
# 现在可以看到接近 4x 的加速比
```

### 注意事项

- 线程安全变得更复杂：以前隐式安全的操作现在需要显式同步
- C 扩展兼容性：未标记 GIL 兼容的扩展会强制启用 GIL
- 单线程性能略有下降（约 3-5%），因为原子操作开销
- 当前仍是实验性特性，生产环境需谨慎

## 相关概念

- [[python/concurrency/01_GIL/GIL原理与工作机制]] — GIL 的基本原理
- [[python/concurrency/01_GIL/GIL性能影响与量化]] — GIL 对性能的影响
- [[python/concurrency/06_线程安全/线程安全与原子操作]] — 自由线程下的线程安全

## 来源

- PEP 703 — Making the GIL Optional：https://peps.python.org/pep-0703/
- Sam Gross — Lessons from PEP 703 (PyCon 2024)：https://www.youtube.com/watch?v=1O1mJfRPvQU
- Larry Hastings — Removing the GIL (PyCon 2016)：https://www.youtube.com/watch?v=P3AyCYgnaTc
