---
tags: [concept, Python, GIL, CPython, 引用计数]
type: concept
created: 2026-09-16
updated: 2026-09-16
---

# GIL 原理与工作机制

> GIL（Global Interpreter Lock，全局解释器锁）是 CPython 中最核心的并发约束。本页讲解 GIL 的本质、为什么存在、以及它的工作机制。

## 什么是 GIL

GIL 是 CPython 解释器中的一把**进程级互斥锁**。任何线程在**执行 Python 字节码之前**，必须先获取 GIL。由于 GIL 同一时刻只允许一个线程持有，多线程无法在多核 CPU 上真正并行执行 Python 代码。

## 为什么存在 GIL

CPython 的内存管理基于**引用计数**（Reference Counting）：

```c
// CPython 内部（简化）
typedef struct _PyObject {
    Py_ssize_t ob_refcnt;  // 引用计数
    PyTypeObject *ob_type;
} PyObject;

#define Py_INCREF(op)  ((op)->ob_refcnt++)
#define Py_DECREF(op)  \
    if (--((op)->ob_refcnt) == 0) { \
        _Py_Dealloc(op);  // 引用计数归零，立即回收
    }
```

**核心问题**：如果多个线程同时对同一个对象做 `ob_refcnt++` 或 `ob_refcnt--`，会导致数据竞争：

```
线程 A: 读取 refcnt = 2
线程 B: 读取 refcnt = 2    ← 两个线程读到相同的值
线程 A: 写入 refcnt = 3
线程 B: 写入 refcnt = 3    ← 应该是 4，但写成了 3

更危险的场景：
线程 A: refcnt-- → 0 → 回收内存
线程 B: 还在使用这个对象 → 访问已释放内存 → 段错误（segfault）
```

GIL 的解决方案：**用一把全局锁串行化所有字节码执行**，彻底避免引用计数的竞争。这是 1992 年 GIL 诞生时的设计动机，当时单核 CPU 是主流。

## GIL 保护的边界

| GIL 保护 | GIL 不保护 |
|-----------|------------|
| Python 字节码执行 | I/O 操作（读写文件、网络请求） |
| 对象引用计数 | C 扩展中显式释放 GIL 的代码 |
| 字节码层面的操作原子性 | 底层 C 库函数调用 |
| 内置类型操作（list.append 等） | 第三方 C 扩展的内部逻辑 |

## GIL 的获取与释放

```python
# Python 层面的伪代码（实际在 C 层实现）
def run_thread(func):
    for each python_bytecode_instruction:
        acquire_gil()        # 获取 GIL
        execute_bytecode()   # 执行一条字节码
        release_gil()        # 释放 GIL（如果有其他线程等待）
```

**关键**：GIL 的获取/释放不是逐字节码进行的，而是按**时间片**轮转。

### GIL 切换策略的演进

#### Python 3.1 及以前：基于 tick 计数

```
- 每 100 条字节码（sys.setcheckinterval(100)）强制释放 GIL
- 问题：某些指令耗时长，tick 计数无法保证公平
```

#### Python 3.2 及以后：基于时间片

```
- 默认每 5ms（sys.getswitchinterval()）释放 GIL
- 如果没有其他线程等待 GIL，当前线程立即重新获取
- 改进：避免过于频繁的 GIL 竞争导致的"惊群"效应
```

```python
import sys

# 查看当前 GIL 切换间隔（秒）
print(sys.getswitchinterval())  # 0.005 (5ms)

# 设置切换间隔（不推荐在生产环境修改）
sys.setswitchinterval(0.005)
```

## I/O 操作释放 GIL

这是 GIL 设计中**最关键**的一点：当线程执行阻塞式 I/O 操作时，会**主动释放 GIL**，让其他线程执行。

```python
import threading
import time

def cpu_task():
    """纯计算——一直持有 GIL"""
    total = 0
    for i in range(100_000_000):
        total += i
    return total

def io_task():
    """I/O 操作——释放 GIL"""
    time.sleep(1)  # 模拟 I/O
    print("I/O done")

# 两个纯计算线程：GIL 串行化，无并行加速
start = time.time()
t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)
t1.start(); t2.start()
t1.join(); t2.join()
print(f"双线程 CPU: {time.time()-start:.2f}s")

# 单线程两次：同样串行，但少了线程切换开销
start = time.time()
cpu_task()
cpu_task()
print(f"单线程 CPU: {time.time()-start:.2f}s")

# 两个 I/O 线程：释放 GIL，真并发
start = time.time()
t1 = threading.Thread(target=io_task)
t2 = threading.Thread(target=io_task)
t1.start(); t2.start()
t1.join(); t2.join()
print(f"双线程 I/O: {time.time()-start:.2f}s")  # ~1s 而非 ~2s
```

## C 扩展释放 GIL

C 扩展函数可以通过 `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS` 宏显式释放 GIL：

```c
// C 扩展示例
#include <Python.h>

static PyObject* heavy_compute(PyObject* self, PyObject* args) {
    double result;

    Py_BEGIN_ALLOW_THREADS   // 释放 GIL
    // === 以下代码不持有 GIL，可以安全多核并行 ===
    result = do_heavy_numerical_computation();
    // === 不能在此区域调用任何 Python C API ===
    Py_END_ALLOW_THREADS     // 重新获取 GIL

    return PyFloat_FromDouble(result);
}
```

NumPy、OpenCV 等 C 扩展库正是利用此机制在内部实现真正的多核并行。

## 常见误解澄清

### 误解 1："Python 多线程没有用"

**纠正**：对于 I/O 密集型任务，多线程效果显著。GIL 在 I/O 等待时释放，多线程可以同时等待多个 I/O。

### 误解 2："多核机器上 Python 多线程更快"

**纠正**：对于 CPU 密集型任务，多线程在多核机器上不会更快（甚至更慢），因为 GIL 串行化了字节码执行。应使用 multiprocessing。

### 误解 3："GIL 保护了所有线程安全问题"

**纠正**：GIL 只保证单条字节码的原子性，不保证多步操作的原子性。详见 [[python/concurrency/06_线程安全/竞态条件详解]]。

### 误解 4："asyncio 不受 GIL 影响"

**纠正**：asyncio 运行在单线程事件循环中，确实没有 GIL 竞争问题。但如果协程中有 CPU 密集计算，仍然需要 multiprocessing 或 run_in_executor 来避免阻塞事件循环。

## 相关概念

- [[python/concurrency/01_GIL/GIL性能影响与量化]] — GIL 对性能的量化影响
- [[python/concurrency/01_GIL/GIL历史与PEP703]] — 自由线程（No-GIL）的未来
- [[python/concurrency/01_GIL/绕过GIL的策略]] — 在 GIL 限制下实现并行
- [[python/concurrency/06_线程安全/竞态条件详解]] — GIL 不保证的安全问题

## 来源

- Python 官方文档 — Python Global Interpreter Lock：https://docs.python.org/3/glossary.html#term-GIL
- CPython 源码（ceval.c GIL 实现）：https://github.com/python/cpython/blob/main/Python/ceval.c
- Python 并发编程 HOWTO：https://docs.python.org/3/howto/sockets.html
