+++
title = "Go Green-Tea-GC"
date = 2026-05-12
draft = false

categories = [
    "Go"
]

tags = [
    "Go",
    "interview"
]

series = [
    "Go Interview"
]
series_order = 4

+++

# Green Tea GC 详解

> 来源：[The Green Tea Garbage Collector](https://go.dev/blog/greenteagc)，Michael Knyszek & Austin Clements，2025-10-29  
> 参考：[Go 1.26 Release Notes](https://go.dev/doc/go1.26#new-garbage-collector)，2026-02-10

---

## 1. 背景：mark-sweep 的基本原理

Go 的垃圾回收器采用的是 **mark-sweep（标记-清除）** 算法，核心是一个图遍历（graph flood）：

1. **Mark 阶段**：从根对象（全局变量、局部变量）出发，沿着指针遍历对象图，标记所有访问到的对象为"存活"。
2. **Sweep 阶段**：遍历堆上所有未标记的对象，将它们的内存标记为空闲，供分配器复用。

这个算法本身很简单，但问题在于**实际执行时面临的性能挑战**。

---

## 2. 问题：传统 mark-sweep 为什么变慢了

Go 团队在多年 CPU profiling 中发现两个关键数据：

- **90%** 的 GC 开销在 Mark 阶段，仅约 10% 在 Sweep 阶段
- 在 Mark 阶段中，**至少 35%** 的时间消耗在等待内存访问上（stalled on accessing heap memory）

博客用了一个生动的比喻：

> 把 CPU 想象成一辆车，程序想象成一条公路。CPU 想要飙到高速，需要视野开阔、道路畅通。但图遍历算法就像在市区街道里开车——频繁转弯、遇红绿灯、绕行人，发动机再快也没用。

具体来说，图遍历的问题在于：

1. **内存跳跃严重**：GC 在堆里跳来跳去，频繁在不同页面（page）之间切换，每次只做很小一块工作。
2. **缓存失效**：现代 CPU 缓存基于最近访问的内存和相邻内存来预取，但互相指向的对象在物理内存上未必相邻，导致缓存命中极低。
3. **NUMA（Non-Uniform Memory Access）**：内存访问成本取决于从哪个 CPU 核心发起，跨节点访问更慢。
4. **内存带宽下降**：CPU 核心数越来越多，但每核心能提交的内存请求数在相对减少。
5. **向量指令无用武之地**：新硬件有 AVX-512 这样的向量指令，但图遍历的随机性使得这类"一次处理大量数据"的指令完全无法利用。

---

## 3. 核心思路：Green Tea 的关键 idea

> **"Work with pages, not objects"**

Green Tea 的核心想法简单到惊人：**以页（page）为单位工作，而不是以对象为单位**。

具体变化：

| 传统 mark-sweep | Green Tea |
|---|---|
| 扫描对象 | 扫描整页 |
| 工作列表中放对象 | 工作列表中放整页 |
| 对象级元数据 | 每页内部维护对象级元数据（seen bit + scanned bit） |

这样做的目的：**让一次遍历能做更多连续内存访问**，更好地利用 CPU 缓存和预取机制。

---

## 4. 机制详解

### 4.1 元数据设计

每页（8 KiB）新增两套比特位（bit），对应页面内的每个对象槽位：

- **seen bit**：对象是否已被发现（从根或指针可达）
- **scanned bit**：对象的指针是否已被扫描过

```
对象槽 1    对象槽 2    对象槽 3  ...
[seen|scanned] [seen|scanned] [seen|scanned] ...
```

`seen=1, scanned=0` 的对象是需要扫描指针的对象。

### 4.2 Mark 流程对比

**传统图遍历（Graph Flood）**：

```
工作列表（栈，LIFO）：
1. 从根出发，找到 A 对象 → 把 A 加入工作列表
2. 弹出 A，扫描 A 的指针 → 找到 B → 把 B 加入工作列表
3. 弹出 B，扫描 B 的指针 → 找到 C → 把 C 加入工作列表
...
（不断跳跃，频繁跨页）
```

**Green Tea**：

```
工作列表（队列，FIFO）：
1. 从根出发，找到 A 对象 → 把 A 所在页面（P）加入工作列表
   在页 P 的元数据中设置 A 的 seen bit
2. 弹出页面 P：
   查看 P 的 seen/scanned bits，找出需要扫描的对象
   批量扫描 P 中所有待扫描对象的指针
   将新发现的页面加入工作列表
3. 每个页面被处理时，优先在同一页内按内存顺序连续扫描多个对象
```

关键差异：
- **FIFO 而非 LIFO**：让同一页上的待扫描对象有时间积累，从而能批量处理
- **页级去重**：同一页可能被多次加入工作列表（因为不同对象指向了同一页），但只在页级去重而非对象级
- **内存局部性**：对同一页内的多个对象的扫描在物理上连续，大量减少缓存失效

### 4.3 图解对比

博客中的图示清晰展示了两种算法的路径差异：

- **Graph Flood**：7 次扫描，路径在 A/B/C/D 各页之间跳跃不断
- **Green Tea**：仅 4 次扫描，对 A、B 页做更长的连续扫描路径

> 箭头越长（left-to-right pass），意味着缓存利用率越高，CPU 预取越有效。

---

## 5. 向量化加速（AVX-512）

Green Tea 带来的另一个重大收益是**向量指令终于可以派上用场**了。

### 5.1 为什么以前用不上

传统图遍历每个对象大小不定（有的 16 字节，有的几十 KB），元数据分散，工作列表项不可预测——完全没有规律性，无法用向量指令批量处理。

### 5.2 Green Tea 的规律性

Green Tea 每个页面的所有对象是**同一大小类（size class）**的，这意味着：
- 一页的元数据大小固定（e.g., 7 个对象的页面 = 14 bits seen + 14 bits scanned）
- AVX-512 有 512-bit 宽寄存器，一页的完整元数据只需要**两个寄存器**就能装下

### 5.3 扫描内核（Scanning Kernel）

AVX-512 扫描的逻辑步骤（每个 size class 有不同的 expansion 矩阵）：

```
1. 取出"seen" bits 和 "scanned" bits（各一 bit/对象）
2. 求差集：seen XOR scanned → "active objects" bitmap
3. 将 active objects bitmap 从"每对象一 bit"扩展为"每 word(8字节)一 bit"
   → "active words" bitmap（这一步用 VGF2P8AFFINEQB 指令高效完成）
4. 取 pointer/scalar bitmap（内存分配器维护，每 word 一 bit 标记是否为指针）
   与 active words bitmap 求交 → "active pointers" bitmap
5. 按 active pointers bitmap 迭代，每找到一组 set bits 用向量指令一次加载 64 字节
```

核心指令是 `VGF2P8AFFINEQB`（Galois Field New Instructions 的一部分），这是一种"位向量瑞士军刀"指令，能在**几个 CPU 周期内**完成步骤 3 的 expansion 操作。

### 5.4 效果

- 整个页面的扫描可以用少量直线型指令完成，不再是循环+分支+随机内存访问
- 尚在 rollout 阶段，预计带来额外 **~10% GC CPU 降低**

---

## 6. 性能数据

| 指标 | 改善幅度 |
|---|---|
| GC CPU 时间降低（不含向量） | **10%~40%**（中位数 ~10%） |
| GC CPU 时间降低（加上向量增强后） | **额外 ~10%** |
| 最低收益保障 | 即使每页只积累 2% 的对象就扫描，也比旧算法好 |

典型场景：应用 10% 时间在 GC → 降低 10%~40% GC 时间 → 整体 CPU 降低 1%~4%。

---

## 7. 局限性

Green Tea 的假设是：每页能积累足够多的对象一起扫描，从而弥补积累过程的开销。

**不适用的场景**：
- 堆结构不规则，每页通常只有 1 个对象需要扫描
- 这种情况甚至可能比旧算法更差（单对象扫描还要承受页面积累的额外开销）

实现中已有**单对象页面的特殊处理**来减少 regression，但不能完全消除。

---

## 8. 可用性

| 版本 | 状态 | 启用方式 |
|---|---|---|
| Go 1.25 | 实验性 | `GOEXPERIMENT=greenteagc` 编译 |
| Go 1.26 | **默认启用** | 可用 `GOEXPERIMENT=nogreenteagc` 禁用 |
| Go 1.26+ | 逐步推送向量增强 | 仅在支持的 x86 硬件上（Zen 4+, Ice Lake+） |

---

## 9. 总结

Green Tea 不是一种新算法（底层仍是 mark-sweep），而是对 mark-sweep **遍历顺序和元数据组织方式的深度优化**：

```
传统 mark-sweep  →  随机跳跃对象图遍历  →  缓存不友好，NUMA不友好，向量无用
Green Tea        →  按页积累+批量扫描  →  缓存友好，NUMA可扩展，向量加速
```

核心思想：**让每次内存访问做更多事情，减少跳转**，一句话概括——"Work with pages, not objects"。

---

*附：Go 1.26 起 Green Tea 已替代旧的 GC 成为默认，这是一个从 2018 年开始探索、经历大量设计迭代才落地的工程成果。参与贡献者包括 Michael Knyszek、Austin Clements、Michael Pratt、Cherry Mui、David Chase、Keith Randall 等。*
