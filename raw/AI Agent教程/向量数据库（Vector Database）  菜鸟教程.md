---
title: "向量数据库（Vector Database） | 菜鸟教程"
source: "https://www.runoob.com/ai-agent/vector-database.html"
author:
published:
created: 2026-06-23
description:
tags:
  - "clippings"
---
## 向量数据库（Vector Database）

向量数据库（Vector Database）是一种专门用于存储、索引和检索高维向量数据的数据库系统。

你可以把它理解为：把意思相近的东西存在一起，并能快速找到和这个最像的那些东西。

与传统数据库通过精确匹配来查询（WHERE name = 'Alice'）不同，向量数据库通过相似度来查询（找到和这张图最相似的 10 张图）。

### 一个直观的类比

想象一个图书馆的场景：

| 数据库类型 | 检索方式 | 类比 |
| --- | --- | --- |
| 传统数据库 | 按书号、书名精确检索 | 找一本指定编号的书 |
| 向量数据库 | 按内容相关性检索 | 找"所有和《三体》风格类似的科幻小说" |

这种语义上的相似，正是向量数据库解决的核心问题。

---

## 为什么需要向量数据库

在深入技术细节之前，我们先理解向量数据库解决了什么问题。

### 传统数据库的局限

传统关系型数据库（MySQL、PostgreSQL）非常擅长处理结构化数据，但在面对以下需求时力不从心：

- 图片搜索（找出视觉相似的图片）
- 语义搜索（用户搜"苹果手机"，能找到"iPhone"的相关内容）
- 推荐系统（找到"和你喜欢的歌曲风格类似的歌"）
- 异常检测（找到"和正常行为差异最大的日志"）

这些问题的共同特征是：需要理解内容的"含义"，而不是做字面匹配。

### 传统方案的问题

```
用 LIKE '%苹果%' 搜索 → 找不到 "iPhone"、"Apple"
用全文索引搜索     → 找不到语义相关但用词不同的内容
```

### 对比示意图

下面的图表直观展示了传统数据库和向量数据库在查询方式上的根本差异。

<svg viewBox="0 0 760 340" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="760" height="340" fill="#f8f9fa" rx="12"></rect><text x="380" y="32" text-anchor="middle" font-size="15" font-weight="bold" fill="#1a1a2e">传统数据库 vs 向量数据库：查询方式对比</text> <rect x="20" y="50" width="340" height="270" fill="#fff" rx="10" stroke="#e0e0e0" stroke-width="1.5"></rect><rect x="20" y="50" width="340" height="40" fill="#e74c3c" rx="10"></rect><rect x="20" y="78" width="340" height="12" fill="#e74c3c"></rect><text x="190" y="76" text-anchor="middle" font-size="13" font-weight="bold" fill="#fff">传统数据库（精确匹配）</text> <rect x="40" y="105" width="300" height="36" fill="#fdecea" rx="6" stroke="#e74c3c" stroke-width="1"></rect><text x="190" y="125" text-anchor="middle" font-size="12" fill="#c0392b" font-style="italic">SELECT * WHERE name = '苹果手机'</text> <text x="55" y="165" font-size="11" fill="#555">匹配结果：</text> <rect x="40" y="172" width="300" height="28" fill="#fdecea" rx="5"></rect><text x="55" y="191" font-size="12" fill="#e74c3c">√ 苹果手机 Pro 128GB</text> <rect x="40" y="205" width="300" height="28" fill="#f5f5f5" rx="5"></rect><text x="55" y="224" font-size="12" fill="#aaa">x iPhone 15（未匹配）</text> <rect x="40" y="238" width="300" height="28" fill="#f5f5f5" rx="5"></rect><text x="55" y="257" font-size="12" fill="#aaa">x Apple 手机（未匹配）</text> <rect x="40" y="271" width="300" height="28" fill="#f5f5f5" rx="5"></rect><text x="55" y="290" font-size="12" fill="#aaa">x 智能手机 iOS（未匹配）</text> <rect x="400" y="50" width="340" height="270" fill="#fff" rx="10" stroke="#e0e0e0" stroke-width="1.5"></rect><rect x="400" y="50" width="340" height="40" fill="#2ecc71" rx="10"></rect><rect x="400" y="78" width="340" height="12" fill="#2ecc71"></rect><text x="570" y="76" text-anchor="middle" font-size="13" font-weight="bold" fill="#fff">向量数据库（语义相似）</text> <rect x="420" y="105" width="300" height="36" fill="#eafaf1" rx="6" stroke="#2ecc71" stroke-width="1"></rect><text x="570" y="125" text-anchor="middle" font-size="12" fill="#27ae60" font-style="italic">search(embed("苹果手机"), top_k=4)</text> <text x="435" y="165" font-size="11" fill="#555">相似结果（含相似度）：</text> <rect x="420" y="172" width="300" height="28" fill="#eafaf1" rx="5"></rect><text x="435" y="191" font-size="12" fill="#27ae60">√ 苹果手机 Pro 128GB 0.98</text> <rect x="420" y="205" width="300" height="28" fill="#eafaf1" rx="5"></rect><text x="435" y="224" font-size="12" fill="#27ae60">√ iPhone 15 0.95</text> <rect x="420" y="238" width="300" height="28" fill="#eafaf1" rx="5"></rect><text x="435" y="257" font-size="12" fill="#27ae60">√ Apple 手机 0.93</text> <rect x="420" y="271" width="300" height="28" fill="#eafaf1" rx="5"></rect><text x="435" y="290" font-size="12" fill="#27ae60">√ 智能手机 iOS 0.87</text></svg>

---

## 核心概念：向量与嵌入

理解向量和嵌入是掌握向量数据库的第一步。

### 什么是向量（Vector）

在数学上，向量就是一组有序的数字。

```
[0.12, -0.54, 0.87, 0.03, ..., 0.61]   ← 这就是一个向量
```

在机器学习中，这组数字代表某个对象的语义特征，维度通常在 128 到 4096 之间。

### 什么是嵌入（Embedding）

嵌入（Embedding）是将现实世界的对象（文字、图片、音频等）转换成向量的过程和结果。

这个转换由嵌入模型完成，其核心思想是：语义相近的对象，其向量在空间中的距离也更近。

<svg viewBox="0 0 720 320" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="720" height="320" fill="#f8f9fa" rx="12"></rect><text x="360" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a1a2e">嵌入（Embedding）过程示意</text> <rect x="20" y="55" width="130" height="50" rx="8" fill="#3498db" opacity="0.9"></rect><text x="85" y="76" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">文本</text> <text x="85" y="94" text-anchor="middle" font-size="11" fill="#dce9f7">"今天天气真好"</text> <rect x="20" y="130" width="130" height="50" rx="8" fill="#9b59b6" opacity="0.9"></rect><text x="85" y="151" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">图像</text> <text x="85" y="169" text-anchor="middle" font-size="11" fill="#e8d5f5">一张猫咪的照片</text> <rect x="20" y="205" width="130" height="50" rx="8" fill="#e67e22" opacity="0.9"></rect><text x="85" y="226" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">音频</text> <text x="85" y="244" text-anchor="middle" font-size="11" fill="#fde8cc">一段音乐片段</text> <line x1="152" y1="80" x2="255" y2="155" stroke="#bbb" stroke-width="1.5" marker-end="url(#arrow1)"></line><line x1="152" y1="155" x2="255" y2="165" stroke="#bbb" stroke-width="1.5" marker-end="url(#arrow1)"></line><line x1="152" y1="230" x2="255" y2="175" stroke="#bbb" stroke-width="1.5" marker-end="url(#arrow1)"></line><defs><marker id="arrow1" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#bbb"></path></marker><marker id="arrow2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#2ecc71"></path></marker></defs><rect x="255" y="115" width="150" height="90" rx="10" fill="#1a1a2e"></rect><text x="330" y="150" text-anchor="middle" font-size="12" fill="#fff" font-weight="bold">嵌入模型</text> <text x="330" y="168" text-anchor="middle" font-size="10" fill="#aaa">Embedding Model</text> <text x="330" y="186" text-anchor="middle" font-size="10" fill="#f39c12">text-embedding-3</text> <text x="330" y="200" text-anchor="middle" font-size="10" fill="#f39c12">CLIP / ResNet...</text> <line x1="407" y1="150" x2="450" y2="100" stroke="#2ecc71" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="407" y1="160" x2="450" y2="160" stroke="#2ecc71" stroke-width="1.5" marker-end="url(#arrow2)"></line><line x1="407" y1="170" x2="450" y2="220" stroke="#2ecc71" stroke-width="1.5" marker-end="url(#arrow2)"></line><rect x="452" y="60" width="245" height="55" rx="8" fill="#eafaf1" stroke="#2ecc71" stroke-width="1.2"></rect><text x="465" y="79" font-size="10" fill="#888">文本向量 (1536维):</text> <text x="465" y="98" font-size="10.5" fill="#27ae60" font-family="monospace">[0.12, -0.54, 0.87, 0.03,...]</text> <rect x="452" y="132" width="245" height="55" rx="8" fill="#f5eafb" stroke="#9b59b6" stroke-width="1.2"></rect><text x="465" y="151" font-size="10" fill="#888">图像向量 (512维):</text> <text x="465" y="170" font-size="10.5" fill="#8e44ad" font-family="monospace">[-0.33, 0.71, 0.22, 0.95,...]</text> <rect x="452" y="203" width="245" height="55" rx="8" fill="#fef5e7" stroke="#e67e22" stroke-width="1.2"></rect><text x="465" y="222" font-size="10" fill="#888">音频向量 (256维):</text> <text x="465" y="241" font-size="10.5" fill="#ca6f1e" font-family="monospace">[0.66, -0.11, 0.48, -0.72,...]</text> <text x="360" y="295" text-anchor="middle" font-size="11" fill="#888">Tip: 语义相近的对象，转换后的向量在空间中距离也更近</text></svg>

### 语义近则向量近

用一个 2D 简化示例来理解（实际是几百至几千维）：

<svg viewBox="0 0 500 420" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="500" height="420" fill="#f8f9fa" rx="12"></rect><text x="250" y="28" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a1a2e">向量空间中的语义聚类（二维示意）</text> <line x1="60" y1="370" x2="460" y2="370" stroke="#ccc" stroke-width="1.5"></line><line x1="60" y1="370" x2="60" y2="50" stroke="#ccc" stroke-width="1.5"></line><text x="465" y="374" font-size="10" fill="#aaa">x</text> <text x="58" y="45" font-size="10" fill="#aaa">y</text> <line x1="60" y1="50" x2="460" y2="50" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><line x1="60" y1="150" x2="460" y2="150" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><line x1="60" y1="250" x2="460" y2="250" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><line x1="160" y1="50" x2="160" y2="370" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><line x1="260" y1="50" x2="260" y2="370" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><line x1="360" y1="50" x2="360" y2="370" stroke="#eee" stroke-width="1" stroke-dasharray="4,4"></line><ellipse cx="150" cy="280" rx="65" ry="55" fill="#fdecea" stroke="#e74c3c" stroke-width="1.5" stroke-dasharray="5,3"></ellipse><text x="150" y="348" text-anchor="middle" font-size="10" fill="#c0392b">动物类</text> <circle cx="120" cy="270" r="5" fill="#e74c3c"></circle><text x="125" y="260" font-size="11" fill="#e74c3c">猫</text> <circle cx="170" cy="260" r="5" fill="#e74c3c"></circle><text x="175" y="252" font-size="11" fill="#e74c3c">狗</text> <circle cx="140" cy="305" r="5" fill="#e74c3c"></circle><text x="145" y="298" font-size="11" fill="#e74c3c">兔子</text> <circle cx="180" cy="290" r="5" fill="#e74c3c"></circle><text x="185" y="285" font-size="11" fill="#e74c3c">熊</text> <ellipse cx="370" cy="140" rx="65" ry="55" fill="#eaf4fb" stroke="#3498db" stroke-width="1.5" stroke-dasharray="5,3"></ellipse><text x="370" y="205" text-anchor="middle" font-size="10" fill="#2980b9">科技类</text> <circle cx="340" cy="130" r="5" fill="#3498db"></circle><text x="347" y="124" font-size="11" fill="#3498db">电脑</text> <circle cx="390" cy="120" r="5" fill="#3498db"></circle><text x="397" y="114" font-size="11" fill="#3498db">手机</text> <circle cx="355" cy="165" r="5" fill="#3498db"></circle><text x="362" y="158" font-size="11" fill="#3498db">键盘</text> <circle cx="395" cy="155" r="5" fill="#3498db"></circle><text x="402" y="148" font-size="11" fill="#3498db">显示器</text> <ellipse cx="280" cy="310" rx="60" ry="45" fill="#eafaf1" stroke="#2ecc71" stroke-width="1.5" stroke-dasharray="5,3"></ellipse><text x="280" y="366" text-anchor="middle" font-size="10" fill="#27ae60">食物类</text> <circle cx="255" cy="300" r="5" fill="#2ecc71"></circle><text x="262" y="294" font-size="11" fill="#27ae60">披萨</text> <circle cx="300" cy="295" r="5" fill="#2ecc71"></circle><text x="307" y="288" font-size="11" fill="#27ae60">汉堡</text> <circle cx="265" cy="325" r="5" fill="#2ecc71"></circle><text x="272" y="319" font-size="11" fill="#27ae60">面条</text> <circle cx="200" cy="255" r="8" fill="#f39c12" stroke="#fff" stroke-width="2"></circle><text x="213" y="250" font-size="11" fill="#e67e22" font-weight="bold">查询：宠物</text> <line x1="200" y1="255" x2="120" y2="270" stroke="#f39c12" stroke-width="1.5" stroke-dasharray="4,3"></line><line x1="200" y1="255" x2="170" y2="260" stroke="#f39c12" stroke-width="1.5" stroke-dasharray="4,3"></line><text x="250" y="395" text-anchor="middle" font-size="10" fill="#888">查询"宠物"的向量，距离"猫""狗"更近，属于动物聚类</text></svg>

> 关键理解：向量空间中距离近的两个向量，其原始内容在语义上也更相近。这是向量数据库所有能力的基础。

---

## 相似度计算方法

找到"最相似的向量"的核心是计算两个向量的距离或相似度。以下是三种最常用的方法。

### 余弦相似度（Cosine Similarity）

余弦相似度衡量两个向量的方向角，忽略长度。这是最常用的方法，尤其适合文本场景。

公式：

$$
\text{CosineSimilarity}(A, B)
=
\frac{A \cdot B}
{\|A\|\|B\|}
=
\frac{\sum_{i=1}^{n} A_i B_i}
{\sqrt{\sum_{i=1}^{n} A_i^2}\sqrt{\sum_{i=1}^{n} B_i^2}}
$$

- 结果范围：-1 到 1，值越大越相似
- 适用场景：文本语义搜索、文档相似度

### 欧氏距离（Euclidean Distance）

欧氏距离衡量两点之间的直线距离，距离越小越相似。

公式：

$$
d(A,B)
=
\sqrt{\sum_{i=1}^{n}(A_i-B_i)^2}
$$

- 结果范围：0 到 ∞，值越小越相似
- 适用场景：图像检索、地理位置相关应用

### 点积（Dot Product）

点积是向量相乘求和，结合了方向和长度信息。

公式：

$$
A \cdot B
=
\sum_{i=1}^{n} A_i B_i
$$

- 适用场景：推荐系统（向量已归一化时等价于余弦相似度）

### 三种方法对比

<svg viewBox="0 0 700 220" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="700" height="220" fill="#f8f9fa" rx="12"></rect><text x="350" y="26" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a1a2e">三种相似度计算方法对比</text> <rect x="15" y="38" width="670" height="30" fill="#1a1a2e" rx="6"></rect><text x="100" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">方法</text> <text x="270" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">原理</text> <text x="430" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">结果含义</text> <text x="590" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">推荐场景</text> <rect x="15" y="70" width="670" height="42" fill="#eaf4fb" rx="4"></rect><text x="100" y="97" text-anchor="middle" font-size="11" fill="#2980b9" font-weight="bold">余弦相似度</text> <text x="270" y="90" text-anchor="middle" font-size="10" fill="#444">计算两向量夹角的余弦值</text> <text x="270" y="104" text-anchor="middle" font-size="10" fill="#444">关注方向，忽略大小</text> <text x="430" y="97" text-anchor="middle" font-size="10" fill="#444">[-1, 1]，越接近 1 越相似</text> <text x="590" y="97" text-anchor="middle" font-size="10" fill="#444">文本搜索、NLP 首选</text> <rect x="15" y="114" width="670" height="42" fill="#fef9ec" rx="4"></rect><text x="100" y="141" text-anchor="middle" font-size="11" fill="#e67e22" font-weight="bold">欧氏距离</text> <text x="270" y="134" text-anchor="middle" font-size="10" fill="#444">两点之间的直线距离</text> <text x="270" y="148" text-anchor="middle" font-size="10" fill="#444">关注绝对位置差异</text> <text x="430" y="141" text-anchor="middle" font-size="10" fill="#444">[0, ∞)，越接近 0 越相似</text> <text x="590" y="141" text-anchor="middle" font-size="10" fill="#444">图像检索、坐标系数据</text> <rect x="15" y="158" width="670" height="42" fill="#eafaf1" rx="4"></rect><text x="100" y="185" text-anchor="middle" font-size="11" fill="#27ae60" font-weight="bold">点积</text> <text x="270" y="178" text-anchor="middle" font-size="10" fill="#444">向量各分量乘积之和</text> <text x="270" y="192" text-anchor="middle" font-size="10" fill="#444">方向+长度综合考量</text> <text x="430" y="185" text-anchor="middle" font-size="10" fill="#444">值越大越相似（无固定范围）</text> <text x="590" y="185" text-anchor="middle" font-size="10" fill="#444">推荐系统、归一化场景</text></svg>

### Python 代码示例

以下示例演示了三种相似度计算方法的 Python 实现：

## 实例

import numpy as np  
  
\# 余弦相似度：衡量方向相似性  
def cosine\_similarity(a, b):  
return np.dot(a, b) / (np.linalg.norm(a) \* np.linalg.norm(b))  
  
\# 欧氏距离：衡量绝对位置差异  
def euclidean\_distance(a, b):  
return np.linalg.norm(a - b)  
  
\# 点积：结合方向与长度  
def dot\_product(a, b):  
return np.dot(a, b)  
  
\# 示例向量  
v1 = np.array(\[0.12, -0.54, 0.87, 0.03\])  
v2 = np.array(\[0.10, -0.50, 0.90, 0.05\])  
v3 = np.array(\[-0.80, 0.20, -0.30, 0.70\])  
  
print(f"v1 vs v2 余弦相似度: {cosine\_similarity(v1, v2):.4f}") # 约 0.9997（非常相似）  
print(f"v1 vs v3 余弦相似度: {cosine\_similarity(v1, v3):.4f}") # 约 -0.55（不相似）

```
v1 vs v2 余弦相似度: 0.9997
v1 vs v3 余弦相似度: -0.5512
```

---

## 向量索引算法

数据量大时（百万、亿级），对每一条数据做相似度计算（暴力检索）太慢。向量数据库使用专门的索引算法来加速查询。

### 暴力检索（Flat / Brute-force）

暴力检索遍历所有向量，逐一计算相似度。

| 维度 | 说明 |
| --- | --- |
| 原理 | 遍历所有向量，逐一计算相似度 |
| 优点 | 结果 100% 精确 |
| 缺点 | 数据量大时极慢，O(n) 复杂度 |
| 适用 | 数据量小于 10 万，对精度要求极高 |

### IVF（倒排文件索引）

<svg viewBox="0 0 640 300" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="640" height="300" fill="#f8f9fa" rx="12"></rect><text x="320" y="26" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a1a2e">IVF 索引原理：先聚类，再在桶内搜索</text> <ellipse cx="110" cy="160" rx="80" ry="70" fill="#fdecea" stroke="#e74c3c" stroke-width="1.5"></ellipse><circle cx="90" cy="140" r="4" fill="#e74c3c"></circle><circle cx="120" cy="155" r="4" fill="#e74c3c"></circle><circle cx="100" cy="175" r="4" fill="#e74c3c"></circle><circle cx="130" cy="170" r="4" fill="#e74c3c"></circle><circle cx="85" cy="185" r="4" fill="#e74c3c"></circle><circle cx="107" cy="165" r="8" fill="#c0392b" stroke="#fff" stroke-width="2"></circle><text x="107" y="200" text-anchor="middle" font-size="10" fill="#c0392b">簇 1 中心</text> <ellipse cx="320" cy="120" rx="85" ry="65" fill="#eaf4fb" stroke="#3498db" stroke-width="1.5"></ellipse><circle cx="290" cy="100" r="4" fill="#3498db"></circle><circle cx="340" cy="105" r="4" fill="#3498db"></circle><circle cx="300" cy="130" r="4" fill="#3498db"></circle><circle cx="350" cy="125" r="4" fill="#3498db"></circle><circle cx="310" cy="145" r="4" fill="#3498db"></circle><circle cx="360" cy="140" r="4" fill="#3498db"></circle><circle cx="325" cy="118" r="8" fill="#2980b9" stroke="#fff" stroke-width="2"></circle><text x="325" y="200" text-anchor="middle" font-size="10" fill="#2980b9">簇 2 中心</text> <ellipse cx="530" cy="200" rx="80" ry="65" fill="#eafaf1" stroke="#2ecc71" stroke-width="1.5"></ellipse><circle cx="505" cy="185" r="4" fill="#2ecc71"></circle><circle cx="550" cy="190" r="4" fill="#2ecc71"></circle><circle cx="515" cy="215" r="4" fill="#2ecc71"></circle><circle cx="555" cy="220" r="4" fill="#2ecc71"></circle><circle cx="530" cy="230" r="4" fill="#2ecc71"></circle><circle cx="531" cy="207" r="8" fill="#27ae60" stroke="#fff" stroke-width="2"></circle><text x="531" y="280" text-anchor="middle" font-size="10" fill="#27ae60">簇 3 中心</text> <circle cx="290" cy="155" r="9" fill="#f39c12" stroke="#fff" stroke-width="2"></circle><text x="270" y="175" font-size="11" fill="#e67e22" font-weight="bold">查询</text> <line x1="290" y1="155" x2="107" y2="165" stroke="#e74c3c" stroke-width="1.2" stroke-dasharray="5,3"></line><text x="180" y="150" font-size="9" fill="#e74c3c">远</text> <line x1="290" y1="155" x2="325" y2="118" stroke="#2980b9" stroke-width="2" stroke-dasharray="5,3"></line><text x="315" y="148" font-size="9" fill="#2980b9">近!</text> <line x1="290" y1="155" x2="531" y2="207" stroke="#2ecc71" stroke-width="1.2" stroke-dasharray="5,3"></line><text x="420" y="168" font-size="9" fill="#2ecc71">远</text> <text x="320" y="295" text-anchor="middle" font-size="10" fill="#555">只在距离最近的簇 2 内做精确搜索，大幅减少计算量</text></svg>

IVF 执行步骤：

1. 训练阶段：用 K-Means 将所有向量聚成 N 个簇，记录每个簇的中心
2. 查询阶段：先找出距离最近的几个簇的中心，再只在这些簇内做精确搜索

### HNSW（分层导航小世界图）

HNSW 是目前最主流的向量索引算法，兼顾速度和精度。

<svg viewBox="0 0 680 360" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="680" height="360" fill="#f8f9fa" rx="12"></rect><text x="340" y="26" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a1a2e">HNSW 分层结构示意</text> <rect x="20" y="45" width="640" height="75" fill="#fef5e7" rx="8" stroke="#f39c12" stroke-width="1" stroke-dasharray="4,3"></rect><text x="30" y="62" font-size="10" fill="#e67e22" font-weight="bold">第 2 层（最稀疏，长程跳转）</text> <circle cx="160" cy="92" r="12" fill="#f39c12" stroke="#fff" stroke-width="2"></circle><text x="160" y="97" text-anchor="middle" font-size="10" fill="#fff">A</text> <circle cx="400" cy="92" r="12" fill="#f39c12" stroke="#fff" stroke-width="2"></circle><text x="400" y="97" text-anchor="middle" font-size="10" fill="#fff">B</text> <line x1="172" y1="92" x2="388" y2="92" stroke="#f39c12" stroke-width="2"></line><rect x="20" y="135" width="640" height="95" fill="#eaf4fb" rx="8" stroke="#3498db" stroke-width="1" stroke-dasharray="4,3"></rect><text x="30" y="152" font-size="10" fill="#2980b9" font-weight="bold">第 1 层（中等密度，中程跳转）</text> <circle cx="100" cy="188" r="12" fill="#3498db" stroke="#fff" stroke-width="2"></circle><text x="100" y="193" text-anchor="middle" font-size="10" fill="#fff">C</text> <circle cx="210" cy="180" r="12" fill="#3498db" stroke="#fff" stroke-width="2"></circle><text x="210" y="185" text-anchor="middle" font-size="10" fill="#fff">A</text> <circle cx="330" cy="195" r="12" fill="#3498db" stroke="#fff" stroke-width="2"></circle><text x="330" y="200" text-anchor="middle" font-size="10" fill="#fff">D</text> <circle cx="450" cy="182" r="12" fill="#3498db" stroke="#fff" stroke-width="2"></circle><text x="450" y="187" text-anchor="middle" font-size="10" fill="#fff">B</text> <circle cx="560" cy="190" r="12" fill="#3498db" stroke="#fff" stroke-width="2"></circle><text x="560" y="195" text-anchor="middle" font-size="10" fill="#fff">E</text> <line x1="112" y1="188" x2="198" y2="182" stroke="#3498db" stroke-width="1.5"></line><line x1="222" y1="181" x2="318" y2="193" stroke="#3498db" stroke-width="1.5"></line><line x1="342" y1="194" x2="438" y2="184" stroke="#3498db" stroke-width="1.5"></line><line x1="462" y1="183" x2="548" y2="189" stroke="#3498db" stroke-width="1.5"></line><rect x="20" y="245" width="640" height="95" fill="#eafaf1" rx="8" stroke="#2ecc71" stroke-width="1" stroke-dasharray="4,3"></rect><text x="30" y="262" font-size="10" fill="#27ae60" font-weight="bold">第 0 层（最密集，精确搜索）</text> <circle cx="80" cy="295" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="80" y="299" text-anchor="middle" font-size="9" fill="#fff">F</text> <circle cx="140" cy="310" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="140" y="314" text-anchor="middle" font-size="9" fill="#fff">C</text> <circle cx="200" cy="285" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="200" y="289" text-anchor="middle" font-size="9" fill="#fff">A</text> <circle cx="270" cy="305" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="270" y="309" text-anchor="middle" font-size="9" fill="#fff">G</text> <circle cx="340" cy="288" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="340" y="292" text-anchor="middle" font-size="9" fill="#fff">D</text> <circle cx="410" cy="308" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="410" y="312" text-anchor="middle" font-size="9" fill="#fff">H</text> <circle cx="470" cy="285" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="470" y="289" text-anchor="middle" font-size="9" fill="#fff">B</text> <circle cx="540" cy="300" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="540" y="304" text-anchor="middle" font-size="9" fill="#fff">E</text> <circle cx="600" cy="285" r="10" fill="#2ecc71" stroke="#fff" stroke-width="2"></circle><text x="600" y="289" text-anchor="middle" font-size="9" fill="#fff">I</text> <line x1="90" y1="295" x2="130" y2="308" stroke="#2ecc71" stroke-width="1.2"></line><line x1="150" y1="308" x2="190" y2="287" stroke="#2ecc71" stroke-width="1.2"></line><line x1="210" y1="287" x2="260" y2="303" stroke="#2ecc71" stroke-width="1.2"></line><line x1="280" y1="303" x2="330" y2="289" stroke="#2ecc71" stroke-width="1.2"></line><line x1="350" y1="289" x2="400" y2="306" stroke="#2ecc71" stroke-width="1.2"></line><line x1="420" y1="306" x2="460" y2="287" stroke="#2ecc71" stroke-width="1.2"></line><line x1="480" y1="285" x2="530" y2="298" stroke="#2ecc71" stroke-width="1.2"></line><line x1="550" y1="298" x2="590" y2="286" stroke="#2ecc71" stroke-width="1.2"></line><line x1="160" y1="104" x2="210" y2="168" stroke="#aaa" stroke-width="1" stroke-dasharray="3,2"></line><line x1="400" y1="104" x2="450" y2="170" stroke="#aaa" stroke-width="1" stroke-dasharray="3,2"></line><line x1="210" y1="192" x2="200" y2="273" stroke="#aaa" stroke-width="1" stroke-dasharray="3,2"></line><line x1="450" y1="194" x2="470" y2="273" stroke="#aaa" stroke-width="1" stroke-dasharray="3,2"></line><text x="340" y="352" text-anchor="middle" font-size="10" fill="#888">查询时从顶层大步跳转定位区域，再逐层细化精确找到最近邻</text></svg>

HNSW 核心思路：

- 构建多层图结构，顶层稀疏，底层密集
- 查询时从顶层入口开始，做"跳格游戏"：每层贪心地往更近的节点跳，再下探到下一层
- 大幅减少需要比较的节点数，时间复杂度近似 O(log n)

### 其他常用索引

| 索引类型 | 特点 | 适用场景 |
| --- | --- | --- |
| Flat（暴力） | 精确但慢 | 小数据集、精度优先 |
| IVF\_Flat | 聚类后精确搜索，速度快 | 中大规模，内存充足 |
| IVF\_PQ | 量化压缩，节省内存 | 超大规模，内存受限 |
| HNSW | 速度快、精度高，内存占用高 | 最常用，推荐首选 |
| ScaNN | Google 出品，优化吞吐量 | 高并发生产环境 |

---

## 主流向量数据库对比

以下是当前最主流的向量数据库横向对比，帮助你在不同场景下做出选择。

<svg viewBox="0 0 720 380" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="720" height="380" fill="#f8f9fa" rx="12"></rect><text x="360" y="26" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a1a2e">主流向量数据库横向对比</text> <rect x="15" y="38" width="690" height="30" fill="#1a1a2e" rx="6"></rect><text x="90" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">数据库</text> <text x="195" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">类型</text> <text x="305" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">部署方式</text> <text x="420" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">特点</text> <text x="570" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">适用场景</text> <text x="672" y="58" text-anchor="middle" font-size="11" fill="#fff" font-weight="bold">难度</text> <rect x="15" y="70" width="690" height="46" fill="#fff" rx="3"></rect><rect x="15" y="70" width="6" height="46" fill="#e74c3c" rx="3"></rect><text x="90" y="89" text-anchor="middle" font-size="12" fill="#e74c3c" font-weight="bold">Chroma</text> <text x="90" y="106" text-anchor="middle" font-size="10" fill="#888">开源免费</text> <text x="195" y="97" text-anchor="middle" font-size="11" fill="#444">纯向量DB</text> <text x="305" y="89" text-anchor="middle" font-size="10" fill="#444">本地 / 云端</text> <text x="305" y="104" text-anchor="middle" font-size="10" fill="#444">嵌入式优先</text> <text x="420" y="89" text-anchor="middle" font-size="10" fill="#444">极简 API，Python 原生</text> <text x="420" y="104" text-anchor="middle" font-size="10" fill="#444">集成 LangChain 最方便</text> <text x="570" y="97" text-anchor="middle" font-size="10" fill="#444">RAG 原型、AI 应用开发</text> <text x="672" y="97" text-anchor="middle" font-size="11" fill="#2ecc71" font-weight="bold">入门</text> <rect x="15" y="118" width="690" height="46" fill="#fafafa" rx="3"></rect><rect x="15" y="118" width="6" height="46" fill="#3498db" rx="3"></rect><text x="90" y="137" text-anchor="middle" font-size="12" fill="#3498db" font-weight="bold">Qdrant</text> <text x="90" y="154" text-anchor="middle" font-size="10" fill="#888">开源免费</text> <text x="195" y="141" text-anchor="middle" font-size="11" fill="#444">纯向量DB</text> <text x="305" y="137" text-anchor="middle" font-size="10" fill="#444">本地 / Docker</text> <text x="305" y="152" text-anchor="middle" font-size="10" fill="#444">云服务</text> <text x="420" y="137" text-anchor="middle" font-size="10" fill="#444">Rust 实现，性能强劲</text> <text x="420" y="152" text-anchor="middle" font-size="10" fill="#444">支持过滤+向量混合检索</text> <text x="570" y="141" text-anchor="middle" font-size="10" fill="#444">生产级推荐，性能优先</text> <text x="672" y="141" text-anchor="middle" font-size="11" fill="#f39c12" font-weight="bold">中级</text> <rect x="15" y="166" width="690" height="46" fill="#fff" rx="3"></rect><rect x="15" y="166" width="6" height="46" fill="#9b59b6" rx="3"></rect><text x="90" y="185" text-anchor="middle" font-size="12" fill="#9b59b6" font-weight="bold">Weaviate</text> <text x="90" y="202" text-anchor="middle" font-size="10" fill="#888">开源免费</text> <text x="195" y="189" text-anchor="middle" font-size="11" fill="#444">多模态DB</text> <text x="305" y="185" text-anchor="middle" font-size="10" fill="#444">本地 / 云端</text> <text x="305" y="200" text-anchor="middle" font-size="10" fill="#444">SaaS</text> <text x="420" y="185" text-anchor="middle" font-size="10" fill="#444">GraphQL API，内置向量化</text> <text x="420" y="200" text-anchor="middle" font-size="10" fill="#444">多模态（文本+图像）</text> <text x="570" y="189" text-anchor="middle" font-size="10" fill="#444">多模态检索，知识图谱</text> <text x="672" y="189" text-anchor="middle" font-size="11" fill="#f39c12" font-weight="bold">中级</text> <rect x="15" y="214" width="690" height="46" fill="#fafafa" rx="3"></rect><rect x="15" y="214" width="6" height="46" fill="#e67e22" rx="3"></rect><text x="90" y="233" text-anchor="middle" font-size="12" fill="#e67e22" font-weight="bold">Milvus</text> <text x="90" y="250" text-anchor="middle" font-size="10" fill="#888">开源免费</text> <text x="195" y="237" text-anchor="middle" font-size="11" fill="#444">纯向量DB</text> <text x="305" y="233" text-anchor="middle" font-size="10" fill="#444">分布式部署</text> <text x="305" y="248" text-anchor="middle" font-size="10" fill="#444">Kubernetes</text> <text x="420" y="233" text-anchor="middle" font-size="10" fill="#444">LF AI 基金会项目</text> <text x="420" y="248" text-anchor="middle" font-size="10" fill="#444">大规模分布式，功能全面</text> <text x="570" y="237" text-anchor="middle" font-size="10" fill="#444">亿级数据，企业大规模</text> <text x="672" y="237" text-anchor="middle" font-size="11" fill="#e74c3c" font-weight="bold">进阶</text> <rect x="15" y="262" width="690" height="46" fill="#fff" rx="3"></rect><rect x="15" y="262" width="6" height="46" fill="#27ae60" rx="3"></rect><text x="90" y="281" text-anchor="middle" font-size="12" fill="#27ae60" font-weight="bold">Pinecone</text> <text x="90" y="298" text-anchor="middle" font-size="10" fill="#888">商业 SaaS</text> <text x="195" y="285" text-anchor="middle" font-size="11" fill="#444">托管向量DB</text> <text x="305" y="285" text-anchor="middle" font-size="10" fill="#444">纯云端</text> <text x="305" y="300" text-anchor="middle" font-size="10" fill="#444">全托管服务</text> <text x="420" y="281" text-anchor="middle" font-size="10" fill="#444">零运维，开箱即用</text> <text x="420" y="296" text-anchor="middle" font-size="10" fill="#444">免费套餐可用</text> <text x="570" y="285" text-anchor="middle" font-size="10" fill="#444">快速上线，无运维能力团队</text> <text x="672" y="285" text-anchor="middle" font-size="11" fill="#2ecc71" font-weight="bold">入门</text> <rect x="15" y="310" width="690" height="46" fill="#fafafa" rx="3"></rect><rect x="15" y="310" width="6" height="46" fill="#2c3e50" rx="3"></rect><text x="90" y="329" text-anchor="middle" font-size="12" fill="#2c3e50" font-weight="bold">pgvector</text> <text x="90" y="346" text-anchor="middle" font-size="10" fill="#888">开源插件</text> <text x="195" y="333" text-anchor="middle" font-size="11" fill="#444">PG 扩展</text> <text x="305" y="333" text-anchor="middle" font-size="10" fill="#444">已有 PostgreSQL</text> <text x="305" y="348" text-anchor="middle" font-size="10" fill="#444">环境直接用</text> <text x="420" y="333" text-anchor="middle" font-size="10" fill="#444">复用已有 PG 基础设施</text> <text x="420" y="348" text-anchor="middle" font-size="10" fill="#444">SQL 接口，上手最快</text> <text x="570" y="333" text-anchor="middle" font-size="10" fill="#444">已用 PG 的项目，轻量接入</text> <text x="672" y="333" text-anchor="middle" font-size="11" fill="#2ecc71" font-weight="bold">入门</text></svg>

> 新手建议：从 Chroma 或 pgvector 起步，前者适合 AI 应用原型，后者适合已有 PostgreSQL 的项目。

---

## 快速上手：Python 示例

下面用 Chroma（最易入门）演示完整的增删改查流程。

### 安装

## 实例

pip install chromadb openai

### 完整示例：构建一个文档语义搜索系统

以下代码从头到尾演示了如何使用 Chroma 构建一个基于语义的文档搜索系统。

## 实例

import chromadb  
from chromadb.utils import embedding\_functions  
  
\# ─── 1. 初始化客户端 ───────────────────────────────────────────  
\# 持久化到本地（推荐）  
client = chromadb.PersistentClient(path="./my\_vector\_db")  
  
\# 使用 OpenAI 嵌入模型（也可换成本地模型）  
openai\_ef = embedding\_functions.OpenAIEmbeddingFunction(  
api\_key="your-openai-api-key", # 必填：替换为你的 API Key  
model\_name="text-embedding-3-small" # 1536 维，性价比高  
)  
  
\# ─── 2. 创建集合（类似关系库里的"表"）────────────────────────  
collection = client.get\_or\_create\_collection(  
name="my\_documents", # 集合名称  
embedding\_function=openai\_ef, # 绑定嵌入函数  
metadata={"hnsw:space": "cosine"} # 使用余弦相似度  
)  
  
\# ─── 3. 插入文档 ──────────────────────────────────────────────  
documents = \[  
"Python 是一种面向对象的解释型编程语言，广泛用于数据科学和 AI 开发",  
"机器学习是人工智能的子领域，让计算机从数据中学习规律",  
"深度学习使用多层神经网络，在图像识别和 NLP 任务中表现优异",  
"向量数据库专门存储高维向量，支持语义相似度搜索",  
"PostgreSQL 是功能强大的开源关系型数据库",  
"Redis 是基于内存的高性能键值数据库，常用于缓存",  
"Docker 容器化技术让应用可以在任何环境中一致运行",  
"Git 是分布式版本控制系统，是现代软件开发的基础工具",  
\]  
  
ids = \[f"doc\_{i}" for i in range(len(documents))\]  
  
\# 批量插入（Chroma 自动调用嵌入模型转为向量后存储）  
collection.add(  
documents=documents,  
ids=ids,  
metadatas=\[{"source": "tutorial", "index": i} for i in range(len(documents))\]  
)  
  
print(f"已插入 {len(documents)} 条文档")

```
已插入 8 条文档
```

## 实例

\# ─── 4. 语义搜索 ──────────────────────────────────────────────  
query = "如何用 Python 做人工智能"  
  
results = collection.query(  
query\_texts=\[query\],  
n\_results=3, # 返回最相似的 3 条  
include=\["documents", "distances", "metadatas"\]  
)  
  
print(f"\\n查询：{query}")  
print("-" \* 50)  
for i, (doc, dist) in enumerate(zip(  
results\["documents"\]\[0\],  
results\["distances"\]\[0\]  
)):  
similarity = 1 - dist # 余弦距离转相似度  
print(f"第 {i+1} 名（相似度 {similarity:.4f}）：")  
print(f" {doc}")  
print()

```
查询：如何用 Python 做人工智能
--------------------------------------------------
第 1 名（相似度 0.9231）：Python 是一种面向对象的解释型编程语言...
第 2 名（相似度 0.8874）：机器学习是人工智能的子领域...
第 3 名（相似度 0.8612）：深度学习使用多层神经网络...
```

## 实例

\# ─── 5. 带过滤条件的搜索（元数据过滤）───────────────────────  
results\_filtered = collection.query(  
query\_texts=\["数据库技术"\],  
n\_results=2,  
where={"source": "tutorial"}, # 只在 source=tutorial 的文档里搜索  
include=\["documents", "distances"\]  
)  
  
\# ─── 6. 更新文档 ──────────────────────────────────────────────  
collection.update(  
ids=\["doc\_0"\],  
documents=\["Python 是目前最流行的编程语言，在 AI、数据分析、Web 开发中均有广泛应用"\],  
metadatas=\[{"source": "tutorial", "index": 0, "updated": True}\]  
)  
  
\# ─── 7. 删除文档 ──────────────────────────────────────────────  
collection.delete(ids=\["doc\_7"\]) # 删除 Git 相关文档  
  
\# ─── 8. 查看集合统计 ──────────────────────────────────────────  
print(f"当前集合文档数：{collection.count()}")

### 不使用第三方嵌入 API（纯本地）

如果你不想使用 OpenAI API，可以用本地嵌入模型完全离线运行。

## 实例

import chromadb  
from sentence\_transformers import SentenceTransformer  
  
\# 使用本地嵌入模型（无需 API Key，完全离线）  
model = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2") # 支持中文  
  
client = chromadb.Client()  
collection = client.create\_collection("local\_demo")  
  
texts = \["今天天气很好", "阳光明媚，适合出门", "股市大涨了", "明天可能会下雨"\]  
  
\# 手动生成向量后插入  
embeddings = model.encode(texts).tolist()  
collection.add(  
embeddings=embeddings,  
documents=texts,  
ids=\[f"id\_{i}" for i in range(len(texts))\]  
)  
  
\# 查询  
query\_embedding = model.encode(\["今天天气怎么样"\]).tolist()  
results = collection.query(query\_embeddings=query\_embedding, n\_results=2)  
print(results\["documents"\])  
\# 输出: \[\['今天天气很好', '阳光明媚，适合出门'\]\]

### pgvector 示例（PostgreSQL 用户）

如果你的项目已经使用 PostgreSQL，pgvector 是最轻量的接入方式。

## 实例

\-- 安装扩展  
CREATE EXTENSION IF NOT EXISTS vector;  
  
\-- 建表：存储文章标题及其向量（1536维）  
CREATE TABLE articles (  
id SERIAL PRIMARY KEY,  
title TEXT NOT NULL,  
content TEXT,  
embedding vector(1536) -- 向量列，1536 维  
);  
  
\-- 创建 HNSW 索引加速查询  
CREATE INDEX ON articles  
USING hnsw (embedding vector\_cosine\_ops)  
WITH (m = 16, ef\_construction = 64);  
  
\-- 插入数据（向量由应用层生成后传入）  
INSERT INTO articles (title, embedding)  
VALUES ('Python 入门指南', '\[0.12, -0.54, 0.87,...\]'::vector);  
  
\-- 语义搜索：找最相似的 5 篇文章  
SELECT id, title,  
1 - (embedding <=> '\[0.10, -0.50, 0.90,...\]'::vector) AS similarity  
FROM articles  
ORDER BY embedding <=> '\[0.10, -0.50, 0.90,...\]'::vector  
LIMIT 5;

> 注意：<=> 是 pgvector 提供的向量距离运算符，用于计算余弦距离。用 1 减去余弦距离即得到余弦相似度。

---

## 典型应用场景

向量数据库在 AI 时代的应用场景非常广泛，以下是六个最典型的场景概览。

<svg viewBox="0 0 720 400" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="720" height="400" fill="#f8f9fa" rx="12"></rect><text x="360" y="28" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a1a2e">向量数据库的典型应用场景</text> <rect x="20" y="45" width="210" height="160" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="20" y="45" width="210" height="5" rx="10" fill="#e74c3c"></rect><text x="125" y="75" text-anchor="middle" font-size="18" fill="#e74c3c" font-weight="bold">RAG</text> <text x="125" y="96" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">RAG 知识库问答</text> <text x="35" y="116" font-size="10" fill="#555">将企业文档转换为向量，</text> <text x="35" y="130" font-size="10" fill="#555">用户提问时检索相关片段</text> <text x="35" y="144" font-size="10" fill="#555">交给 LLM 生成答案。</text> <text x="35" y="165" font-size="10" fill="#e74c3c">代表：ChatPDF、NotionAI</text> <text x="35" y="181" font-size="10" fill="#e74c3c">企业知识库助手</text> <rect x="250" y="45" width="210" height="160" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="250" y="45" width="210" height="5" rx="10" fill="#3498db"></rect><text x="355" y="75" text-anchor="middle" font-size="18" fill="#3498db" font-weight="bold">推荐</text> <text x="355" y="96" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">个性化推荐系统</text> <text x="265" y="116" font-size="10" fill="#555">将用户行为、商品信息转为</text> <text x="265" y="130" font-size="10" fill="#555">向量，寻找"品味相似的用户</text> <text x="265" y="144" font-size="10" fill="#555">喜欢什么"来做推荐。</text> <text x="265" y="165" font-size="10" fill="#3498db">代表：Spotify 歌曲推荐</text> <text x="265" y="181" font-size="10" fill="#3498db">电商相关商品推荐</text> <rect x="490" y="45" width="210" height="160" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="490" y="45" width="210" height="5" rx="10" fill="#9b59b6"></rect><text x="595" y="75" text-anchor="middle" font-size="18" fill="#9b59b6" font-weight="bold">图片</text> <text x="595" y="96" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">以图搜图</text> <text x="505" y="116" font-size="10" fill="#555">将图像编码为向量，</text> <text x="505" y="130" font-size="10" fill="#555">通过相似度搜索找出视觉</text> <text x="505" y="144" font-size="10" fill="#555">相近的图片，无需标签。</text> <text x="505" y="165" font-size="10" fill="#9b59b6">代表：Google 图片搜索</text> <text x="505" y="181" font-size="10" fill="#9b59b6">淘宝拍照搜商品</text> <rect x="20" y="225" width="210" height="155" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="20" y="225" width="210" height="5" rx="10" fill="#e67e22"></rect><text x="125" y="255" text-anchor="middle" font-size="18" fill="#e67e22" font-weight="bold">检测</text> <text x="125" y="276" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">异常检测</text> <text x="35" y="296" font-size="10" fill="#555">正常行为被映射到聚集的</text> <text x="35" y="310" font-size="10" fill="#555">向量区域，异常行为的向量</text> <text x="35" y="324" font-size="10" fill="#555">远离正常区域即可检测。</text> <text x="35" y="345" font-size="10" fill="#e67e22">代表：网络入侵检测</text> <text x="35" y="361" font-size="10" fill="#e67e22">金融欺诈识别</text> <rect x="250" y="225" width="210" height="155" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="250" y="225" width="210" height="5" rx="10" fill="#27ae60"></rect><text x="355" y="255" text-anchor="middle" font-size="18" fill="#27ae60" font-weight="bold">去重</text> <text x="355" y="276" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">内容去重 / 版权检测</text> <text x="265" y="296" font-size="10" fill="#555">通过向量相似度判断两段</text> <text x="265" y="310" font-size="10" fill="#555">内容是否高度相似，识别</text> <text x="265" y="324" font-size="10" fill="#555">抄袭或重复内容。</text> <text x="265" y="345" font-size="10" fill="#27ae60">代表：论文查重系统</text> <text x="265" y="361" font-size="10" fill="#27ae60">音乐版权检测</text> <rect x="490" y="225" width="210" height="155" rx="10" fill="#fff" stroke="#e0e0e0" stroke-width="1.2"></rect><rect x="490" y="225" width="210" height="5" rx="10" fill="#2c3e50"></rect><text x="595" y="255" text-anchor="middle" font-size="18" fill="#2c3e50" font-weight="bold">人脸</text> <text x="595" y="276" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">人脸 / 生物特征识别</text> <text x="505" y="296" font-size="10" fill="#555">将人脸编码为特征向量，</text> <text x="505" y="310" font-size="10" fill="#555">在向量库中检索最相似的</text> <text x="505" y="324" font-size="10" fill="#555">已知人脸完成识别。</text> <text x="505" y="345" font-size="10" fill="#2c3e50">代表：人脸门禁系统</text> <text x="505" y="361" font-size="10" fill="#2c3e50">手机人脸解锁</text></svg>

### RAG（检索增强生成）架构

RAG 是目前向量数据库最主要的应用场景之一。下面是其核心工作流程：

<svg viewBox="0 0 700 180" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="700" height="180" fill="#f8f9fa" rx="12"></rect><text x="350" y="24" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a1a2e">RAG 系统工作流程</text> <rect x="15" y="45" width="110" height="80" rx="8" fill="#e8f4f8" stroke="#3498db" stroke-width="1.5"></rect><text x="70" y="72" text-anchor="middle" font-size="14" fill="#2980b9" font-weight="bold">Step 1</text> <text x="70" y="92" text-anchor="middle" font-size="10" font-weight="bold" fill="#2980b9">用户提问</text> <text x="70" y="107" text-anchor="middle" font-size="9" fill="#555">"Python 怎么</text> <text x="70" y="120" text-anchor="middle" font-size="9" fill="#555">处理异常？"</text> <polygon points="128,85 145,78 145,92" fill="#aaa"></polygon><rect x="148" y="45" width="110" height="80" rx="8" fill="#fef9ec" stroke="#f39c12" stroke-width="1.5"></rect><text x="203" y="72" text-anchor="middle" font-size="14" fill="#e67e22" font-weight="bold">Step 2</text> <text x="203" y="92" text-anchor="middle" font-size="10" font-weight="bold" fill="#e67e22">问题向量化</text> <text x="203" y="107" text-anchor="middle" font-size="9" fill="#555">Embedding 模型</text> <text x="203" y="120" text-anchor="middle" font-size="9" fill="#555">-&gt; [0.12, -0.54,...]</text> <polygon points="261,85 278,78 278,92" fill="#aaa"></polygon><rect x="281" y="45" width="110" height="80" rx="8" fill="#eafaf1" stroke="#2ecc71" stroke-width="1.5"></rect><text x="336" y="72" text-anchor="middle" font-size="14" fill="#27ae60" font-weight="bold">Step 3</text> <text x="336" y="92" text-anchor="middle" font-size="10" font-weight="bold" fill="#27ae60">向量检索</text> <text x="336" y="107" text-anchor="middle" font-size="9" fill="#555">在向量DB中</text> <text x="336" y="120" text-anchor="middle" font-size="9" fill="#555">找最相似文档</text> <polygon points="394,85 411,78 411,92" fill="#aaa"></polygon><rect x="414" y="45" width="110" height="80" rx="8" fill="#fdecea" stroke="#e74c3c" stroke-width="1.5"></rect><text x="469" y="72" text-anchor="middle" font-size="14" fill="#c0392b" font-weight="bold">Step 4</text> <text x="469" y="92" text-anchor="middle" font-size="10" font-weight="bold" fill="#c0392b">LLM 生成</text> <text x="469" y="107" text-anchor="middle" font-size="9" fill="#555">检索内容+问题</text> <text x="469" y="120" text-anchor="middle" font-size="9" fill="#555">-&gt; Prompt -&gt; 答案</text> <polygon points="527,85 544,78 544,92" fill="#aaa"></polygon><rect x="547" y="45" width="138" height="80" rx="8" fill="#f5eafb" stroke="#9b59b6" stroke-width="1.5"></rect><text x="616" y="72" text-anchor="middle" font-size="14" fill="#8e44ad" font-weight="bold">Step 5</text> <text x="616" y="92" text-anchor="middle" font-size="10" font-weight="bold" fill="#8e44ad">返回答案</text> <text x="616" y="107" text-anchor="middle" font-size="9" fill="#555">基于文档的</text> <text x="616" y="120" text-anchor="middle" font-size="9" fill="#555">准确、可溯源回答</text> <text x="350" y="162" text-anchor="middle" font-size="10" fill="#888">向量数据库在步骤 3 中承担核心角色：毫秒级找到最相关的知识片段</text></svg>

---

## 选型建议与最佳实践

### 选型决策树

根据你的具体情况，按照以下决策树选择合适的向量数据库：

```
你的情况是什么？
│
├─── 已有 PostgreSQL，且数据量 < 500 万
│    └──> 用 pgvector，无缝集成，零额外运维
│
├─── 做 AI/LLM 应用原型，快速验证
│    └──> 用 Chroma，几行代码跑起来
│
├─── 需要生产级部署，性能优先，数据量 500 万 ~ 1 亿
│    └──> 用 Qdrant，Rust 实现，性能强
│
├─── 超大规模（> 1 亿），有 K8s 运维能力
│    └──> 用 Milvus，分布式，功能最全
│
└─── 团队没有运维能力，愿意付费
     └──> 用 Pinecone 云服务，开箱即用
```

### 嵌入模型的选择

选择合适的嵌入模型是向量数据库应用的关键第一步。

| 需求 | 推荐模型 |
| --- | --- |
| 中英文文本（高质量） | OpenAI text-embedding-3-small |
| 中文文本（本地离线） | BAAI/bge-large-zh-v1.5 |
| 多语言通用 | paraphrase-multilingual-MiniLM-L12-v2 |
| 图文多模态 | OpenAI CLIP 系列 |

### 性能优化技巧

1\. 批量插入：一次插入多条，避免频繁单条写入。

## 实例

\# 推荐：批量插入  
collection.add(documents=docs\_list, ids=ids\_list)  
  
\# 不推荐：循环单条（每次都重新索引，效率极低）  
\# for doc, id in zip(docs\_list, ids\_list):  
\# collection.add(documents=\[doc\], ids=\[id\])

2\. 向量归一化：使用余弦相似度前，提前归一化向量可加速计算。

## 实例

import numpy as np  
  
def normalize(v):  
"""对向量做 L2 归一化，使模长为 1"""  
return v / np.linalg.norm(v)

3\. 合理设置 n\_results：不要无脑设置很大的 top\_k，一般 RAG 场景 3~10 条足够。

4\. 善用元数据过滤：在搜索时配合 where 条件，缩小搜索范围。

## 实例

\# 只在"技术文档"分类里搜索，缩小范围提升精度  
collection.query(  
query\_texts=\["Python 异常处理"\],  
where={"category": "tech\_doc"},  
n\_results=5  
)

5\. 定期重建索引：数据量增长后，适时重建 HNSW 索引以维持查询性能。

### 常见踩坑

以下是初学者使用向量数据库时最容易遇到的问题及解决方案。

| 问题 | 说明 | 解决方案 |
| --- | --- | --- |
| 嵌入模型要统一 | 插入和查询必须用同一个模型 | 在配置文件中固定模型版本 |
| 维度不匹配 | 换了模型但没重建集合 | 换模型时删除重建集合 |
| 文本过长 | 大多数模型有 token 限制（512~8192） | 超长文本先分块（chunking） |
| 相似度不准确 | 文本没有分块，语义被稀释 | 按段落或固定长度切分文档 |
| 冷启动慢 | 数据量大时首次加载索引耗时 | 提前预热，或使用持久索引 |

---

## 总结

让我们回顾本教程的核心知识点：

<svg viewBox="0 0 700 200" xmlns="http://www.w3.org/2000/svg" font-family="'Segoe UI', Arial, sans-serif" style="max-width:100%;height:auto;"><rect width="700" height="200" fill="#1a1a2e" rx="12"></rect><text x="350" y="32" text-anchor="middle" font-size="14" font-weight="bold" fill="#fff">Vector Database 核心知识点回顾</text> <rect x="20" y="55" width="122" height="120" rx="10" fill="#e74c3c" opacity="0.85"></rect><text x="81" y="82" text-anchor="middle" font-size="16" fill="#fff" font-weight="bold">向量/嵌入</text> <text x="81" y="100" text-anchor="middle" font-size="9" fill="#fcc">把对象转为</text> <text x="81" y="114" text-anchor="middle" font-size="9" fill="#fcc">高维数字向量</text> <text x="81" y="128" text-anchor="middle" font-size="9" fill="#fcc">语义近则距离近</text> <text x="81" y="142" text-anchor="middle" font-size="9" fill="#fcc">是一切的基础</text> <rect x="152" y="55" width="122" height="120" rx="10" fill="#f39c12" opacity="0.85"></rect><text x="213" y="82" text-anchor="middle" font-size="16" fill="#fff" font-weight="bold">相似度计算</text> <text x="213" y="100" text-anchor="middle" font-size="9" fill="#fde">余弦相似度</text> <text x="213" y="114" text-anchor="middle" font-size="9" fill="#fde">欧氏距离</text> <text x="213" y="128" text-anchor="middle" font-size="9" fill="#fde">点积</text> <text x="213" y="142" text-anchor="middle" font-size="9" fill="#fde">文本首选余弦</text> <rect x="284" y="55" width="122" height="120" rx="10" fill="#2ecc71" opacity="0.85"></rect><text x="345" y="82" text-anchor="middle" font-size="16" fill="#fff" font-weight="bold">索引算法</text> <text x="345" y="100" text-anchor="middle" font-size="9" fill="#d5f5e3">HNSW 最主流</text> <text x="345" y="114" text-anchor="middle" font-size="9" fill="#d5f5e3">IVF 适合大规模</text> <text x="345" y="128" text-anchor="middle" font-size="9" fill="#d5f5e3">Flat 适合小数据</text> <text x="345" y="142" text-anchor="middle" font-size="9" fill="#d5f5e3">近似换速度</text> <rect x="416" y="55" width="122" height="120" rx="10" fill="#3498db" opacity="0.85"></rect><text x="477" y="82" text-anchor="middle" font-size="16" fill="#fff" font-weight="bold">选型参考</text> <text x="477" y="100" text-anchor="middle" font-size="9" fill="#d6eaf8">入门：Chroma</text> <text x="477" y="114" text-anchor="middle" font-size="9" fill="#d6eaf8">生产：Qdrant</text> <text x="477" y="128" text-anchor="middle" font-size="9" fill="#d6eaf8">已有PG：pgvector</text> <text x="477" y="142" text-anchor="middle" font-size="9" fill="#d6eaf8">亿级：Milvus</text> <rect x="548" y="55" width="132" height="120" rx="10" fill="#9b59b6" opacity="0.85"></rect><text x="614" y="82" text-anchor="middle" font-size="16" fill="#fff" font-weight="bold">核心应用</text> <text x="614" y="100" text-anchor="middle" font-size="9" fill="#e8daef">RAG 知识问答</text> <text x="614" y="114" text-anchor="middle" font-size="9" fill="#e8daef">个性化推荐</text> <text x="614" y="128" text-anchor="middle" font-size="9" fill="#e8daef">以图搜图</text> <text x="614" y="142" text-anchor="middle" font-size="9" fill="#e8daef">异常检测</text></svg>

向量数据库是 AI 时代基础设施的重要一环。

它解决了传统数据库无法处理的语义相似性搜索问题，是构建 RAG 系统、推荐系统、多模态搜索的核心组件。

### 学习路径建议

1. 第一步：理解向量和嵌入的概念，运行本文的 Chroma 示例
2. 第二步：尝试用 LangChain + Chroma 构建一个简单的文档问答系统
3. 第三步：学习 HNSW 等索引算法，理解精度与速度的权衡
4. 第四步：根据实际项目需求，选择合适的向量数据库并在生产环境部署

---