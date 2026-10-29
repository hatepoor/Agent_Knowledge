---
tags: [java, 中段, 并发编程, 内存模型, 线程安全]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# 线程安全与 Java 内存模型

> **什么时候该读**：写代码时发现「多线程下数据对不上」，或者面试被问「什么是 JMM」「`volatile` 能保证原子性吗」「`happens-before` 有哪些规则」。
> **阅读门槛**：读完 [[java/02-中段阶段-专题精讲/并发编程/01-线程模型与生命周期]]，知道线程共享堆、私有栈即可。

## 学习目标

读完本篇，你应当能够：

1. 用自己的话定义「线程安全」，并说清原子性 / 可见性 / 有序性三个层次
2. 画出 JMM 的主内存—工作内存模型，解释「为什么一个线程改的值另一个线程读不到」
3. 判断一段代码有没有可见性问题、原子性问题，并选择正确的修复手段（`volatile` / `AtomicInteger` / `synchronized`）
4. 列出 `happens-before` 的主要规则，用它证明一段代码是「安全发布」的
5. 解释双重检查锁定为什么必须加 `volatile`

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[java/02-中段阶段-专题精讲/并发编程/01-线程模型与生命周期]] | 线程状态、共享与私有区域、`ThreadLocal` |
| [[java/02-中段阶段-专题精讲/JVM/JVM内存区域与对象布局]] | 堆、栈、PC 的划分是理解 JMM 的物理基础 |

---

## 一、什么是线程安全

### 1.1 定义

**线程安全（thread safety）** 的严格定义是：

> 当多个线程并发访问某个类的一个实例时，这个类**始终都能表现出正确的行为**，且调用方不需要额外的同步措施或协调。

换成人话：**代码在单线程下跑对了，不等于多线程下也跑对。** 线程安全要求多线程并发执行的结果，和「把这些操作排成一条队顺序执行」的结果一致。

### 1.2 线程不安全长什么样

```java
public class UnsafeCounter {
    private int count = 0;

    public void increment() {
        count++;      // ⚠️ 这不是一条原子操作
    }

    public int get() {
        return count;
    }
}
```

两个线程各调用 10000 次 `increment()`，你期望 `count == 20000`，实际很可能小于 20000。

### 1.3 三个层次 ⭐

所有并发问题都能归到这三类。**分清是哪一类，才知道该用什么工具修**。

| 层次 | 问题 | 表现 | 典型修复 |
|------|------|------|---------|
| **原子性（atomicity）** | 一个操作被拆成了多步，中途被打断 | 计数丢失、更新丢失 | `synchronized`、`AtomicInteger`、CAS |
| **可见性（visibility）** | 一个线程改了，另一个线程看不到 | 读到旧值、死循环不退出 | ⭐ `volatile`、`synchronized` |
| **有序性（ordering）** | 指令执行顺序被重排 | 单例拿到半初始化对象 | ⭐ `volatile`、`final`、锁 |

⭐ **一句话记法**：

- **原子性**管「操作会不会被插队」；
- **可见性**管「改动能不能传出去」；
- **有序性**管「传出去的顺序对不对」。

## 二、Java 内存模型（JMM）⭐

### 2.1 为什么需要 JMM

上一篇说过：堆和元空间被所有线程共享。但「共享」不等于「立刻可见」。硬件层面有两个原因：

1. **CPU 缓存**：多核 CPU 各自有一级/二级缓存。线程 A 写变量后，值可能还留在 A 的缓存里，没写回内存；
2. **编译器与 CPU 重排序**：为了流水线效率，编译器和 CPU 都可能**打乱指令顺序**。

**JMM（Java Memory Model，Java 内存模型）** 是 Java 语言规范里的一套抽象规则，它规定：

- 一个线程的写操作，何时对其他线程可见；
- 多线程间的操作，允许被重排到什么程度。

⚠️ **JMM 是一套「规则」，不是「真实硬件」**。它不说「CPU 缓存」这种实现细节，而是抽象成「主内存」和「工作内存」。

### 2.2 主内存与工作内存

```
┌──────────────────── 线程 A ────────────────────┐   ┌────────── 线程 B ──────────┐
│                                                │   │                            │
│  【工作内存】                                   │   │  【工作内存】                │
│   ┌────────────────┐                          │   │   ┌────────────────┐        │
│   │  a = 1         │  ← 线程 A 独占的副本      │   │   │  a = ?（旧值）   │        │
│   │  （寄存器/缓存）  │                          │   │   │  （寄存器/缓存）  │        │
│   └────────────────┘                          │   │   └────────────────┘        │
└────────────────┬───────────────────────────────┘   └───────────┬────────────────┘
                 │ ① write：把工作内存的值写回主内存                  │ ③ read：从主内存读
                 ▼                                                   ▼
        ┌──────────────────────────────────────────────────────────────────┐
        │                    【主内存】                                    │
        │        堆 / 元空间中的共享变量  ——  所有线程共享的那一份              │
        │                     a = 1                                       │
        └──────────────────────────────────────────────────────────────────┘

  ② 线程 A 写完后，如果没有 volatile / 锁，它的"工作内存"里的副本与主内存
     之间的一致性无法保证 —— 线程 B 的 read 可能拿到过期的值。
```

### 2.3 ⭐ 关键机制：读写缓存 + 不保证立刻同步

JMM 规定，线程对变量的访问分两步：

| 步骤 | 动作 | 说明 |
|------|------|------|
| **read（读）** | 从**主内存**取出值，放进线程的**工作内存** | 之后线程读这个变量，都从工作内存读 |
| **write（写）** | 把工作内存的值写回**主内存** | ⚠️ **不保证立刻同步、也不保证不被重排** |

⭐ **于是「可见性问题」的成因就是**：

```
线程 A: write(a=2)      ← 值写回了主内存，但没通知线程 B
线程 B: read(a)         ← 可能命中自己工作内存里的旧副本 a=1
```

⚠️ **注意**：单核 CPU 上这个现象很少出现（因为只有一个缓存），多核机器上高频出现。所以「我本地跑没事」不能作为并发代码正确的证据。

### 2.4 三个保证可见性的手段

| 手段 | 原理 | 保证范围 |
|------|------|---------|
| **`volatile` 关键字** | 每次写都直接刷主内存，每次读都绕过工作内存直读主内存 | 可见性 + 有序性（**不保证原子性**） |
| **锁（`synchronized` / `Lock`）** | 释放锁会刷新缓存，重新获取锁会丢弃缓存副本 | 可见性 + 原子性 + 互斥 |
| **`final` 字段** | 构造完成后的引用写入可以安全发布 | 仅限「正确初始化后不再修改」的字段 |

## 三、并发三大问题逐个击破

### 3.1 原子性：`i++` 不是原子操作 ⭐

`count++` 在字节码层面是**三步**：

```
① 读：  读 count 的当前值到操作数栈       ← iload
② 算：  加 1                            ← iadd
③ 写：  把结果写回 count 变量            ← iput
```

线程 A 执行到①之后被抢占，线程 B 完整执行了①②③，然后 A 继续执行③——**A 用的是它自己读到的旧值**，B 的加一被覆盖了。

```
时间轴：
        线程 A                    线程 B
t1      ① 读到 count=5
t2      (被抢占)                ① 读到 count=5
t3                              ② +1 = 6
t4                              ③ 写回 count=6
t5      ② +1 = 6                (结束)
t6      ③ 写回 count=6     ← ⚠️ B 的 6 被覆盖，最终结果是 6 而不是 7
```

⭐ **这就是「加锁」的用武之地**——把①②③变成一个不可分割的整体。

**修法一：加锁**

⭐ **临界区（critical section）**：被锁保护起来的那段代码。同一时刻只允许一个线程进入临界区——加锁的作用就是把「读-改-写」这几步圈成一个不可分割的整体。

```java
private int count = 0;

public synchronized void increment() {   // ⭐ 整个方法体成了一个临界区
    count++;
}
```

**修法二：原子类**（更轻量，见 [[java/02-中段阶段-专题精讲/并发编程/03-锁机制与原子操作]]）

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();   // ⭐ 内部用 CAS 保证原子性，不加锁
}
```

### 3.2 可见性：一个线程改，另一个线程读不到 ⭐

这是最反直觉的一种 bug：**代码逻辑完全正确，单线程跑 100 万次都对，多线程下就是「有时候不生效」**。

```java
public class VisibilityBug {
    // ⚠️ 没有 volatile
    private static boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            long count = 0;
            while (running && !Thread.currentThread().isInterrupted()) {   // ⚠️ 可能永远读到 true
                count++;
            }
            System.out.println("worker 结束了，count = " + count);
        }, "worker");
        worker.start();

        Thread.sleep(1000);
        running = false;               // 主线程改了
        System.out.println("主线程已把 running 设为 false");

        worker.join(3000);             // 最多等 3 秒
        System.out.println("worker 还活着吗？—— " + worker.isAlive());
        if (worker.isAlive()) {
            System.out.println("⚠️ 复现了可见性问题：主线程改了，worker 看不到");
            worker.interrupt();        // 收尾，否则程序不退出
            worker.join(1000);
        }
    }
}
```

⚠️ **为什么循环里要同时检查 `isInterrupted()`**——`interrupt()` 只是设置中断标记，**只有在 `sleep`/`wait`/`join` 处才会被检查**。worker 跑的是纯计算死循环，不自己检查中断标记就永远停不下来，程序会挂死。

⭐ **修法**：

```java
private static volatile boolean running = true;   // ⭐ 加 volatile
```

加上 `volatile` 后：主线程的 `write` 会**直接刷主内存**，worker 的 `read` 会**绕过工作内存直读主内存**，一定能看到最新值。

### 3.3 有序性：编译器 / CPU 重排序

即使代码一行没改，JIT 编译器和 CPU 也会为了性能**重排指令**。

```java
public class Reordering {
    private static boolean flag = false;   // ⚠️ 没有 volatile
    private static int data = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            long spins = 0;
            // ⚠️ 加上循环上限，避免线程永远看不到 flag 而把程序挂死
            while (!flag && spins < 100_000_000L) {
                spins++;
            }
            System.out.println("data = " + data + "（期望 1，若为 0 说明读到了半成品状态）");
        });
        t.start();

        data = 1;       // 步骤①
        flag = true;    // 步骤②
        t.join();
    }
}
```

⚠️ **编译器可能把步骤②重排到步骤①之前**（因为它看到「先写 flag 再写 data」对单线程语义没有影响）。这样另一线程可能看到 `flag == true` 却看到 `data == 0`——**读到「半成品」状态**。

⭐ **`volatile` 的第二个作用**：给变量加上 volatile 后，编译器会插入**内存屏障（memory barrier）**，禁止特定的指令重排。

**内存屏障**是编译器/CPU 插入的「一道栅栏」，要求特定类型的指令不能跨越它移动。

| 屏障 | 作用 |
|------|------|
| **写屏障（StoreStore / StoreLoad）** | 保证这个 volatile 写之前的写操作都先落地（不被重排到它后面） |
| **读屏障（LoadLoad / LoadStore）** | 保证这个 volatile 读之后的读操作不被重排到它前面 |

⚠️ **只加 `volatile` 不够的场景**：`data = 1; flag = true;` 如果只给 `flag` 加 volatile，写屏障能保证 `data=1` 不跑到 `flag=true` 后面，**这就够了**。但如果两个变量都写、还想控制它们之间的顺序，就要用锁。

**`final` 字段的安全发布语义**：`final` 修饰的字段，在**构造器执行完之前不会被其他线程看到**——JMM 保证 `final` 字段的初始化对正确读到的线程可见（详见 4.4 节的「安全发布」）。

## 四、`volatile` 的语义与局限 ⭐

### 4.1 `volatile` 到底保证什么

| 保证项 | `volatile` | 说明 |
|--------|----------|------|
| **可见性** | ✅ 保证 | 写立即刷主内存，读绕过工作内存 |
| **有序性** | ✅ 保证 | 插入内存屏障，禁止特定重排 |
| **原子性** | ❌ **不保证** | 单次读写是原子的，但复合操作不是 |

⭐ **一句话**：`volatile` 是一个**「可见性 + 顺序性」**的承诺，**绝不是**「原子性」的承诺。

### 4.2 ⚠️ `volatile` 不能保证原子性

```java
public class VolatileCounter {
    private static volatile int count = 0;   // ⭐ 加了 volatile，但依然是错的

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> { for (int i = 0; i < 100000; i++) count++; });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 100000; i++) count++; });
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("期望 200000，实际 " + count);   // ⚠️ 结果小于 200000
    }
}
```

**为什么**：`count++` 是「读-改-写」三步（见 3.1）。`volatile` 只保证「每一次单独的读写」是原子的，**但这三步合起来不是原子的**——两个线程可能都读到 5，都算出 6，都写回 6。

⚠️ **面试标准问法**：「`volatile` 能保证原子性吗？」→ **不能。它保证可见性和有序性，复合操作仍需 `AtomicInteger` 或锁。**

### 4.3 三种手段对比表 ⭐

| 特性 | `volatile` | `AtomicInteger` | `synchronized` |
|------|-----------|-----------------|----------------|
| **可见性** | ✅ | ✅ | ✅ |
| **有序性**（禁止重排） | ✅ | ✅ | ✅ |
| **原子性**（单次复合操作） | ❌ | ✅ | ✅ |
| **互斥**（同一时刻只有一个线程进临界区） | ❌ | ❌（同一变量上 CAS 会串行化，但粒度不同） | ✅ |
| **线程阻塞** | ❌ 不阻塞 | ❌ 不阻塞（CAS 失败自旋） | ✅ 会阻塞挂起 |
| **典型场景** | 状态标记位开关、配置热更新、`DCL` 的单例引用 | 计数器、ID 生成、累加统计 | 保护复杂临界区、复合不变量 |

⭐ **选择口诀**：

- 只是「一个标志位 / 一个开关 / 一个引用」→ `volatile`
- 「一个数字要准确加」→ `AtomicInteger` / `LongAdder`
- 「多个变量要一起改对」→ `synchronized` / `ReentrantLock`

### 4.4 `happens-before` 规则 ⭐

**happens-before（先行发生）** 是 JMM 里最核心的概念。它的定义：

> 如果操作 A **happens-before** 操作 B，那么 **A 的结果对 B 可见**，且 **A 在 B 之前执行**（JMM 保证不会因为重排导致 B 看到 A 之前的状态）。

**用法**：如果一段代码能通过 happens-before 规则串起来，那么**访问共享变量就是安全的**，不需要额外加锁。

#### 主要规则表

| # | 规则 | 说明 | 例子 |
|---|------|------|------|
| 1 | **程序次序规则** | 同一线程内，前面的操作 happens-before 后面的操作 | `int a = 1; int b = 2;` |
| 2 | **监视器锁规则** | 对一个锁的 **unlock** happens-before 后续对它的 **lock** | `synchronized(lock){...}` 的解锁 → 下一个线程的加锁 |
| 3 | **volatile 变量规则** | 对一个 volatile 变量的**写** happens-before 后续对它的**读** | `volatile boolean running;` 的写 → 读 |
| 4 | **线程启动规则** | `Thread.start()` happens-before 该线程的任何动作 | 父线程里设好的值 → 子线程能看到 |
| 5 | **线程中断规则** | 对一个线程的 `interrupt()` happens-before 该线程检测到中断事件 | `interrupt()` → 目标线程抛 `InterruptedException` |
| 6 | **线程终结规则** | 线程内的所有操作 happens-before 其他线程 `join()` 返回 | 子线程写完数据 → 父线程 `join()` 后读到 |
| 7 | **对象终结规则** | 构造器结束 happens-before 该对象引用的**安全发布** | 见下方「安全发布」 |

#### 用规则 4 和 6 举例

```java
// ⭐ 规则 4：start() 之前写的值，子线程一定能看到
int[] data = new int[]{1, 2, 3};
Thread t = new Thread(() -> {
    System.out.println(data[0]);   // 一定是 1，不会是 0
});
t.start();

// ⭐ 规则 6：join() 返回后，子线程写的数据一定能看到
int[] result = new int[1];
Thread t2 = new Thread(() -> { result[0] = 42; });
t2.start();
t2.join();
System.out.println(result[0]);     // 一定是 42
```

⚠️ **注意**：这两条规则**只保证"可见性"**。如果子线程执行的是 `count++` 这种复合操作，规则 4/6 **救不了你**——可见性有了，原子性还是没有。

#### 安全发布 ⭐

**安全发布（safe publication）** 指「一个对象的引用被其他线程看到时，这个对象已经**正确初始化**完毕」。

| 安全发布手段 | 做法 | 适用 |
|-------------|------|------|
| **不可变对象** | 所有字段 `final` 且初始化后不再修改；类本身 `final` | ⭐ 最推荐，天然线程安全 |
| **volatile 引用** | `private static volatile Singleton instance;` | 引用指向的对象已完全构造好 |
| **`final` 字段语义** | 对象构造完成后再发布引用 | JMM 保证 `final` 字段的初始化对读到的线程可见 |
| **静态初始化器** | `static { ... }` 块 | 由 JVM 保证只执行一次且发布安全 |
| **锁保护** | 在 `synchronized` 里构造并赋值 | 通用但笨重 |

⚠️ **危险发布（this escape）**：**对象还没构造完，就把 `this` 泄漏出去了**。例如在构造器里把 `this` 注册给监听器：

```java
import java.util.HashMap;
import java.util.Map;

public class BadRegistry {
    private final Map<String, Object> map = new HashMap<>();

    public BadRegistry() {
        // ⚠️ 危险：this 在构造完成前就泄漏出去了
        Registry.register(this);   // 其他线程可能拿到一个字段全为 null/默认值的半成品
        map.put("ready", true);    // 构造还没做完
    }

    /** 模拟一个全局注册中心（真实项目里可能是监听器列表、事件总线） */
    static class Registry {
        private static final Map<String, Object> INSTANCES = new HashMap<>();

        static void register(Object o) {
            INSTANCES.put("current", o);
        }
    }
}
```

其他线程通过 `Registry` 拿到这个对象时，`map` 可能还是空的。⭐ **规则**：**构造器里不要启动线程、不要把 `this` 注册到外部、不要调用可被重写的方法**。

## 五、可运行示例：亲手复现可见性问题

```java
public class VisibilityDemo {
    // ⚠️ 故意不加 volatile
    private static boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            long count = 0;
            // ⭐ 同时检查中断标记，保证程序一定能收尾退出
            while (running && !Thread.currentThread().isInterrupted()) {
                count++;
            }
            System.out.println("[worker] 退出，最终 count = " + count);
        }, "worker");
        worker.start();

        Thread.sleep(1000);
        System.out.println("[main] 把 running 改成 false");
        running = false;                    // ⚠️ worker 可能看不到

        worker.join(3000);
        if (worker.isAlive()) {
            System.out.println("[main] ⚠️ worker 还在跑 → 可见性问题复现（本次没退出是被 interrupt 踢下线的）");
            worker.interrupt();
            worker.join(1000);
        } else {
            System.out.println("[main] 本次跑出了预期结果（不同 CPU/JIT 行为不同，复现率不保证 100%）");
        }
    }
}
```

⭐ **诚实的提醒**：可见性问题的**复现率不是 100%**，取决于 CPU 架构、JIT 编译时机、内存布局。**复现不出来不代表代码是对的**——测试环境没暴露，只是因为你运气好。

⭐ **加 `volatile` 后一定对**：

```java
private static volatile boolean running = true;   // ⭐ 只改这一行
```

## 六、双重检查锁定（DCL）为什么需要 `volatile`

**双重检查锁定（Double-Checked Locking，DCL）** 是「懒加载单例」的标准写法：

```java
public class Singleton {
    // ⭐⭐ 必须 volatile
    private static volatile Singleton instance;

    private Singleton() { }

    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查：无锁快速返回
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查：加锁后确认
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 6.1 没有 `volatile` 会出什么问题

`new Singleton()` 编译后是**三步**：

```
① 分配内存       → 给对象分配一块空间
② 初始化对象     → 调用构造器，把字段赋上初值
③ 把引用赋给变量  → instance = 这块内存的地址
```

⚠️ **JIT 可能把 ② 和 ③ 重排**（因为单线程下 ②③ 换顺序没有区别）：

```
正常顺序：① 分配 → ② 初始化 → ③ 赋引用      ✅ 安全
重排之后：① 分配 → ③ 赋引用 → ② 初始化      ⚠️ 危险！
```

**危险时间线**：

```
时间轴：
        线程 A                        线程 B
t1      ① 分配内存
t2      ③ instance = addr  ← 引用已可见！
t3                                  读 instance != null → 直接 return！
t4                                  ⚠️ 拿到的是「还没初始化」的对象
t5      ② 初始化（太晚了）
```

线程 B 在 t3 拿到的对象，`final` 字段都还是默认值（`null` / `0`）——**这就是「半初始化对象」**。加上 `volatile` 后：

- volatile 写之前的写操作不会被重排到它后面 → 保证 ② 一定在 ③ 之前；
- volatile 读保证看到最新值 → 线程 B 不会读到「还没初始化」的引用。

⭐ **面试标准问法**：「DCL 为什么要加 `volatile`？」→ **防止构造完成前的引用被其他线程看到（禁止指令重排），保证安全发布。**

### 6.2 现代替代方案

既然 `volatile` + DCL 这么绕，Java 提供了更简单的写法：

```java
// ⭐ 枚举单例（JDK 天然线程安全，推荐）
public enum EnumSingleton {
    INSTANCE;
    public void doWork() { }
}
```

```java
// ⭐ 静态内部类（懒加载 + JVM 保证线程安全，无 volatile 需求）
public class HolderSingleton {
    private HolderSingleton() { }

    private static class Holder {
        static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;      // 第一次访问 Holder 时才加载，且 JVM 保证只加载一次
    }
}
```

## 七、排查手段：`jstack` 看线程状态

并发 bug 常常**不报错、只是结果不对**，或者**卡住不动**。这时要看线程到底在哪。

```bash
jstack <pid> > threads.txt
grep -A 20 "java.lang.Thread.State" threads.txt
```

| 现象 | 状态特征 | 排查方向 |
|------|---------|---------|
| CPU 100% | 大量 `RUNNABLE` 且栈顶是业务方法 | 死循环 / 正则回溯 |
| 接口卡住 | 大量 `BLOCKED` | ⭐ **锁竞争**——看 `locked <0x...>` 找持锁线程 |
| 线程池不干活 | 大量 `WAITING` 在 `ThreadPoolExecutor.getTask()` | 队列满了 / 线程数不够 |
| 可见性 bug | 线程停在某个 `while` 条件上 | 该字段加 `volatile` |
| 死锁（两个线程互相等待、永久卡住） | 输出末尾有 `Found one Java-level deadlock` | ⭐ 直接看 JVM 给的持锁链 |

完整的排查流程、四件套用法、Arthas 命令，见 [[java/02-中段阶段-专题精讲/JVM/排查工具与故障案例]]。

## 八、陷阱

1. **⚠️ 以为「加了 `synchronized` 就万事大吉」**——`synchronized` 保证原子性，但**保护范围要自己圈对**；锁范围过大等于没锁，过小等于漏保护；
2. **⚠️ 以为 `volatile` 能保证 `count++` 的原子性**——它是「读-改-写」三步，`volatile` 管不了；计数要用 `AtomicInteger` / `LongAdder`；
3. **⚠️ 用 `volatile` 保护多个变量之间的不变式**——`volatile` 只保证单个变量的可见性，「两个变量必须同时更新」必须用锁；
4. **⚠️ 以为「本地变量也会出问题」**——方法局部变量在栈上，每个线程各一份，天然安全；
5. **⚠️ 用「加了 `volatile` 就不需要 `synchronized`」一刀切**——涉及复合不变式（先检查后使用、check-then-act）时，锁不可省；
6. **⚠️ DCL 单例漏掉 `volatile`**——会拿到半初始化对象；更推荐枚举单例或静态内部类；
7. **⚠️ 在构造器里把 `this` 泄漏出去**（注册监听器、启动线程）——危险发布，读到半成品；
8. **⚠️ 相信「测试没复现就是没问题」**——可见性问题复现率与 CPU、JIT 有关，没复现可能只是运气；
9. **⚠️ 以为 `jstack` 里 `RUNNABLE` 就是「在算东西」**——等网络 IO 的线程也显示 `RUNNABLE`，要看 `%CPU`；
10. **⚠️ 忽略 `final` 的安全发布语义**——不可变对象是并发编程里最省心的武器，别动不动就上锁。

## 小结

- **线程安全**：并发执行的结果与顺序执行一致；分**原子性 / 可见性 / 有序性**三层
- **JMM**：主内存 + 每线程工作内存；`read` 从主内存取、`write` 写回主内存，**不保证立刻同步** → 可见性问题
- ⭐ **三大问题与对策**：
  - 原子性 → `i++` 是读-改-写三步 → 用 `AtomicInteger` 或锁
  - 可见性 → 一线程改、另一线程读不到 → 用 `volatile`
  - 有序性 → 编译器/CPU 重排 → 用 `volatile` 的内存屏障、`final`
- ⭐ **`volatile` 保证可见性 + 有序性，不保证原子性**；`AtomicInteger` 保证原子性；`synchronized` 三样都保证但会阻塞
- **`happens-before`** 7 条规则是判断「什么时候不需要加锁」的依据；**安全发布**要靠不可变对象 / volatile 引用 / `final`
- **DCL 必须加 `volatile`**：防止 `new` 的「分配→初始化→赋引用」被重排成「分配→赋引用→初始化」，导致其他线程拿到半成品
- 排查靠 `jstack` 看线程状态（见 [[java/02-中段阶段-专题精讲/JVM/排查工具与故障案例]]）

## 练习

1. 用自己的话定义线程安全，并解释「原子性」「可见性」「有序性」三者的区别，各举一个 bug 例子。
2. 画出 JMM 的主内存—工作内存图，说明线程 A 写变量后线程 B 为什么可能读不到。
3. 把下面代码改成三种正确写法（`volatile` 不行的情况请说明原因）：
   ```java
   private int count = 0;
   void add() { count++; }
   ```
4. 为什么「`volatile int count` + `count++`」还是错的？用字节码的读-改-写三步解释。
5. 列出 `happens-before` 的至少 5 条规则，各举一个实际应用。
6. 解释「安全发布」的三种手段，并说明「构造完成前把 `this` 泄漏出去」为什么危险。
7. 双重检查锁定为什么必须加 `volatile`？画出没有 `volatile` 时线程 B 拿到半初始化对象的时间线。
8. 写一个能复现可见性问题的程序，说明为什么「复现率不是 100%」。
9. `synchronized` 保护的方法里，能不能调用一个可能很慢的第三方接口？会带来什么问题？
10. 用 `jstack` 的思路分析：线上接口偶尔卡住、CPU 却不高，你的排查步骤是什么？

## 导航

[[java/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[java/02-中段阶段-专题精讲/并发编程/01-线程模型与生命周期]] · 下一篇：[[java/02-中段阶段-专题精讲/并发编程/03-锁机制与原子操作]]
