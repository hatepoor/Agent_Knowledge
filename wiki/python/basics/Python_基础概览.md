---
tags: [concept, Python, 基础, 概览, 导航]
type: overview
created: 2026-07-08
updated: 2026-07-08
---

# Python 基础

> Python 是一门解释型、动态类型、面向对象的高级编程语言。以简洁易读著称，广泛用于 Web 开发、数据科学、AI/ML、自动化脚本等领域。

## 知识导航

### 基础语法
- [[python/basics/Python_基础概览]] — 本页，总览与导航
- [[python/basics/数值类型]] — int、float、bool、complex 及运算符
- [[python/basics/字符串]] — str 类型、格式化、全部内置方法
- [[python/basics/控制流]] — if/elif/else、for/while 循环、推导式
- [[python/basics/异常处理]] — try/except/else/finally、自定义异常
- [[python/basics/模块与包]] — import 机制、标准库、包管理
- [[python/basics/数学函数]] — math 模块与 numpy 数学函数、精度对比

### 数据结构
- [[python/data_structures/列表]] — list 类型、切片、全部内置方法
- [[python/data_structures/字典]] — dict 类型、遍历、全部内置方法
- [[python/data_structures/元组]] — tuple 类型、不可变序列、命名元组
- [[python/data_structures/集合]] — set/frozenset、集合运算、去重

### 输入输出与文件
- [[python/io/输入输出]] — print/input、f-string 格式化（数字/字符串/日期/表格对齐）
- [[python/io/文件操作]] — 文件读写（open/with）、pathlib 路径操作、JSON/CSV/pickle 序列化

### 函数
- [[python/functions/函数]] — def、参数类型、lambda、闭包、装饰器

### 面向对象
- [[python/oop/类与面向对象]] — class 定义、继承、魔术方法、property

### 进阶主题（Agent 开发相关）
- [[python/advanced/异步编程]] — asyncio、并发执行、异步迭代器、Agent 异步调用
- [[python/advanced/Pydantic数据验证]] — 数据验证、结构化输出、LangChain Tool 定义
- [[python/advanced/装饰器进阶]] — 回调系统、中间件、工具注册装饰器
- [[python/advanced/生成器与异步迭代器]] — yield、流式输出、LLM streaming
- [[python/advanced/类型提示进阶]] — TypeVar、泛型、Annotated、Agent 状态类型
- [[python/advanced/序列化与状态管理]] — JSON/pickle、Checkpoint、Agent 状态持久化

## Python 核心特性速览

| 特性 | 说明 |
|------|------|
| 动态类型 | 变量无需声明类型，运行时确定 |
| 缩进语法 | 用缩进（通常 4 空格）代替花括号定义代码块 |
| 多范式 | 支持面向对象、函数式、过程式编程 |
| 丰富标准库 | "自带电池"——文件、网络、正则、JSON 等开箱即用 |
| 跨平台 | Windows/macOS/Linux 通用 |
| 垃圾回收 | 自动内存管理，基于引用计数 + 分代回收 |

## 内置类型总览

| 类型 | 关键字 | 可变性 | 有序 | 示例 |
|------|--------|--------|------|------|
| 整数 | `int` | 不可变 | — | `42` |
| 浮点数 | `float` | 不可变 | — | `3.14` |
| 布尔 | `bool` | 不可变 | — | `True` / `False` |
| 复数 | `complex` | 不可变 | — | `1+2j` |
| 字符串 | `str` | 不可变 | 有序 | `"hello"` |
| 列表 | `list` | **可变** | 有序 | `[1, 2, 3]` |
| 元组 | `tuple` | 不可变 | 有序 | `(1, 2, 3)` |
| 字典 | `dict` | **可变** | 有序(3.7+) | `{"a": 1}` |
| 集合 | `set` | **可变** | 无序 | `{1, 2, 3}` |
| 冻结集合 | `frozenset` | 不可变 | 无序 | `frozenset({1,2})` |

## 来源

- Python 官方文档：https://docs.python.org/3/
- Python Tutorial: https://docs.python.org/3/tutorial/
