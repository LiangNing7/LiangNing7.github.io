+++
title = "Go Runtime"
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
series_order = 2

+++

# 1. memMgmt

## Go 内存模型

![image-20260415173705619](https://images.liangning7.cn/typora/image-20260415173705619.png)

堆【heap】是 Go 运行时中最大的**临界共享资源**，这意味着每次存取都要加锁，在性能层面是一件很可怕的事情。在解决这个问题，Golang 在堆 mheap 之上，依次细化粒度，建立了 mcentral、mcache 的模型，下面对三者作个梳理：

- `mheap`：全局的内存起源，访问要加全局锁；

- `mcache`：每个 P（正是 GMP 中的 P）持有一份的内存缓存，访问时无锁。

- `mcentral`：每种对象大小规格（全局共划分为 68【67 mspan + 1】 种）对应的缓存，锁的粒度也仅限于同一种规格以内；

  - 多级规格，提高利用率：

    ![image-20260415174103365](https://images.liangning7.cn/typora/image-20260415174103365.png)

  - 首先理下 page 和 mspan 两个概念：

    - page：最小的存储单元。Golang 借鉴操作系统分页管理的思想，每个最小的存储单元也称之为页 page，但大小为 8 KB
    - mspan：最小的管理单元。mspan 的物理空间大小为 page 的整数倍，且 mspane 的规格从 8B 到 80 KB 被划分为 67 种不同的规格，分配对象时，会根据大小映射到不同规格的 mspan，从中获取空间.

mspan 大小为 page 的整数倍，且从 8B 到 80 KB 被划分为 67 种不同的规格，分配对象时，会根据大小映射到不同规格的 mspan，从中获取空间。多规格 mspan 下产生的特点：

1. 根据规格大小，产生了等级的制度
2. 消除了外部碎片，但不可避免会有内部碎片
3. 宏观上能提高整体空间利用率
4. 正是因为有了规格等级的概念，才支持 mcentral 实现细锁化

![image-20260415174351441](https://images.liangning7.cn/typora/image-20260415174351441.png)

## 存储单元核心概念

### 内存单元 mspan

![image-20260415174935543](https://images.liangning7.cn/typora/image-20260415174935543.png)

mspan 的特性：

- mspan 是 Golang 内存管理的最小单元；
- mspan 的物理空间大小是 page 的整数倍（Go 中的 page 大小为 8KB），且内部的页是连续的（至少在虚拟内存的视角中是这样）；
- 每个 mspan 根据空间大小以及面向分配对象的大小，会被划分为不同的等级；
- 同等级的 mspan 会从属同一个 mcentral，最终会被组织成链表，因此带有前后指针（prev、next）；
- 由于同等级的 mspan 内聚于同一个 mcentral，所以会基于同一把互斥锁管理；
- mspan 会基于 bitMap 辅助快速找到空闲内存块（块大小为对应等级下的 object 大小），此时需要使用到 Ctz64 算法。

基础字段讲解：

```go
type mspan struct {
    // 标识前后节点的指针 
    next *mspan     
    prev *mspan    
    // ...
    // 起始地址
    startAddr uintptr 
    // 包含几页，页是连续的
    npages    uintptr 


    // 标识此前的位置都已被占用 
    freeindex uintptr
    // 最多可以存放多少个 object
    nelems uintptr // number of object in the span.


    // bitmap 每个 bit 对应一个 object 块，标识该块是否已被占用
    allocCache uint64
    // ...
    // 标识 mspan 等级，包含 class 和 noscan 两部分信息
    spanclass             spanClass    
    // ...
}
```

### 内存单元等级 spanClass

mspan 根据空间大小和面向分配对象的大小，被划分为 67 种等级（1-67，实际上还有一种隐藏的 0 级，用于处理更大的对象，上不封顶）

下表展示了部分的 mspan 等级列表，数据取自 `runtime/sizeclasses.go` 文件中：

| **class** | **bytes/obj** | **bytes/span** | **objects** | **tail waste** | **max waste** |
| --------- | ------------- | -------------- | ----------- | -------------- | ------------- |
| 1         | 8             | 8192           | 1024        | 0              | 87.50%        |
| 2         | 16            | 8192           | 512         | 0              | 43.75%        |
| 3         | 24            | 8192           | 341         | 8              | 29.24%        |
| 4         | 32            | 8192           | 256         | 0              | 21.88%        |
| ...       |               |                |             |                |               |
| 66        | 28672         | 57344          | 2           | 0              | 4.91%         |
| 67        | 32768         | 32768          | 1           | 0              | 12.50%        |

对上表各列进行解释：

1. class：mspan 等级标识，1-67

2. bytes/obj：该大小规格的对象会从这一 mspan 中获取空间. 创建对象过程中，大小会向上取整为 8B 的整数倍，因此该表可以直接实现 object 到 mspan 等级的映射

3. bytes/span：该等级的 mspan 的总空间大小【这里的大小是 page 的整数倍】

4. object：该等级的 mspan 最多可以 new 多少个对象，结果等于 （3）/（2）

5. tail waste：（3）/（2）可能除不尽，于是该项值为（3）%（2）

6. max waste：通过下面示例解释：

   ![image-20260415180332646](https://images.liangning7.cn/typora/image-20260415180332646.png)

   以 class 3 的 mspan 为例，class 分配的 object 大小统一为 24B，由于 object 大小 <= 16B 的会被分配到 class 2 及之前的 class 中，因此只有 17B-24B 大小的 object 会被分配到 class 3.

   最不利的情况是，当 object 大小为 17B，会产生浪费空间比例如下：

   $((24-17)*341 + 8)/8192 = 0.292358 ≈ 29.24%$

   除了上面谈及的根据大小确定的 mspan 等级外，每个 object 还有一个重要的属性叫做 nocan，标识了 object 是否包含指针，在 gc 时是否需要展开标记.

   在 Golang 中，会将 span class + nocan 两部分信息组装成一个 uint8，形成完整的 spanClass 标识. 8 个 bit 中，高 7 位表示了上表的 span 等级（总共 67 + 1 个等级，8 个 bit 足够用了），最低位表示 nocan 信息.

   ```go
   // runtime/mheap.go
   type spanClass uint8
   
   // uint8 左 7 位为 mspan 等级，最右一位标识是否为 noscan
   func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
       return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
   }
   
   func (sc spanClass) sizeclass() int8 {
       return int8(sc >> 1)
   }
   
   func (sc spanClass) noscan() bool {
       return sc&1 != 0
   }
   ```

### 线程缓存 mcache

![image-20260415180553637](https://images.liangning7.cn/typora/image-20260415180553637.png)

- mcache 是每个 P 独有的缓存，因此交互无锁
- mcache 将每种 spanClass 等级的 mspan 各缓存了一个，总数为 2（nocan 维度） * 68（大小维度）= 136
- mcache 中还有一个为对象分配器 tiny allocator，用于处理小于 16B 对象的内存分配。

### 中心缓存 mcentral

![image-20260415180734318](https://images.liangning7.cn/typora/image-20260415180734318.png)

- 每个 mcentral 对应一种 spanClass
- 每个 mcentral 下聚合了该 spanClass 下的 mspan
- mcentral 下的 mspan 分为两个链表，分别为有空间 mspan 链表 partial 和满空间 mspan 链表 full
- 每个 mcentral 一把锁

### 全局堆缓存 mheap

- 对于 Golang 上层应用而言，堆是操作系统虚拟内存的抽象
- 以页（8KB）为单位，作为最小内存存储单元
- 负责将连续页组装成 mspan
- 全局内存基于 bitMap 标识其使用情况，每个 bit 对应一页，为 0 则自由，为 1 则已被 mspan 组装
- 通过 heapArena 聚合页，记录了页到 mspan 的映射信息（2.7小节展开）
- 建立空闲页基数树索引 radix tree index，辅助快速寻找空闲页（2.6小节展开）
- 是 mcentral 的持有者，持有所有 spanClass 下的 mcentral，作为自身的缓存
- 内存不够时，向操作系统申请，申请单位为 heapArena（64M）

```go
// runtime/mheap.go

type mheap struct {
    // 堆的全局锁
    lock mutex

    // 空闲页分配器，底层是多棵基数树组成的索引，每棵树对应 16 GB 内存空间
    pages pageAlloc 

    // 记录了所有的 mspan. 需要知道，所有 mspan 都是经由 mheap，使用连续空闲页组装生成的
    allspans []*mspan

    // heapAreana 数组，64 位系统下，二维数组容量为 [1][2^22]
    // 每个 heapArena 大小 64M，因此理论上，Golang 堆上限为 2^22*64M = 256T
    arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

    // ...
    // 多个 mcentral，总个数为 spanClass 的个数
    central [numSpanClasses]struct {
        mcentral mcentral
        // 用于内存地址对齐
        pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
    }

    // ...
}
```

这里给出总览定义：

```go
+-------------------------------------------------------------------+
|                               mheap                               |
|  (全局内存池，负责向 OS 申请连续大块内存，按 page 管理。包含所有 mcentral) |
+-------------------------------------------------------------------+
           |                                             |
           | (按 spanClass 分类管理)                       |
           v                                             v
+------------------------+                    +------------------------+
| mcentral (spanClass=1) |      ......        | mcentral (spanClass=x) |
| (中心缓存：全局共享，需加锁)  |                    | (中心缓存：全局共享，需加锁)  |
+------------------------+                    +------------------------+
           |                                             |
           | (当 mcache 对应规格为空时，向 mcentral 申请一个 mspan)
           v                                             v
+-------------------------------------------------------------------+
|                     mcache (绑定在特定的 P 上)                        |
| (本地缓存：P 独享，分配无锁。内部维护了一个数组，保存了各种规格的 mspan)     |
|   alloc[1] -> 指向当前正在使用的规格为 1 的 mspan                         |
|   alloc[x] -> 指向当前正在使用的规格为 x 的 mspan                         |
+-------------------------------------------------------------------+
           |
           | (Goroutine 需要分配对象时，从对应的 mspan 中拿一个格子)
           v
+-------------------------------------------------------------------+
|                              mspan                                |
| (物理上是 N 个 page，逻辑上被划分为同一规格的 object 格子)                 |
|  [ object ] [ object ] [ 空闲格子 ] [ 空闲格子 ]                        |
+-------------------------------------------------------------------+
```

### heapArena

- 每个 heapArena 包含 8192 个页，大小为 8192 * 8KB = 64 MB
- heapArena 记录了页到 mspan 的映射. 因为 GC 时，通过地址偏移找到页很方便，但找到其所属的 mspan 不容易. 因此需要通过这个映射信息进行辅助.
- heapArena 是 mheap 向操作系统申请内存的单位（64MB）

```go
// runtime/mheap.go
const pagesPerArena = 8192

type heapArena struct {
    // ...
    // 实现 page 到 mspan 的映射
    spans [pagesPerArena]*mspan


    // ...
}
```

## 空闲页索引 pageAlloc

要理清这棵技术树，首先需要明白以下几点：

> 运行在 mheap 中，用于管理 page.

### 数据结构背后的含义

- mheap 会基于 bitMap 标识内存中各页的使用情况，bit 位为 0 代表该页是空闲的，为 1 代表该页已被 mspan 占用.
- 每棵基数树聚合了 16 GB 内存空间中各页使用情况的索引信息，用于帮助 mheap 快速找到指定长度的连续空闲页的所在位置
- mheap 持有 2^14 棵基数树，因此索引全面覆盖到 2^14 * 16 GB = 256 T 的内存空间.

### 基数树节点设定

![image-20260415181403397](https://images.liangning7.cn/typora/image-20260415181403397.png)

基数树中，每个节点称之为 PallocSum，是一个 uint64 类型，体现了索引的聚合信息，包含以下四部分：

- start：最右侧 21 个 bit，标识了当前节点映射的 bitMap 范围中首端有多少个连续的 0 bit（空闲页），称之为 start；
- max：中间 21 个 bit，标识了当前节点映射的 bitMap 范围中最多有多少个连续的 0 bit（空闲页），称之为 max；
- end：左侧 21 个 bit，标识了当前节点映射的 bitMap 范围中最末端有多少个连续的 0 bit（空闲页），称之为 end.
- 最左侧一个 bit，弃置不用

### 父子关系

![image-20260415181445332](https://images.liangning7.cn/typora/image-20260415181445332.png)

- 每个父 pallocSum 有 8 个子 pallocSum
- 根 pallocSum 总览全局，映射的 bitMap 范围为全局的 16 GB 空间（其 max 最大值为 2^21，因此总空间大小为 2^21*8KB=16GB）；

- 从首层向下是一个依次八等分的过程，每一个 pallocSum 映射其父节点 bitMap 范围的八分之一，因此第二层 pallocSum 的 bitMap 范围为 16GB/8 = 2GB，以此类推，第五层节点的范围为 16GB / (8^4) = 4 MB，已经很小
- 聚合信息时，自底向上. 每个父 pallocSum 聚合 8 个子 pallocSum 的 start、max、end 信息，形成自己的信息，直到根 pallocSum，坐拥全局 16 GB 的 start、max、end 信息
- mheap 寻页时，自顶向下. 对于遍历到的每个 pallocSum，先看起 start 是否符合，是则寻页成功；再看 max 是否符合，是则进入其下层孩子 pallocSum 中进一步寻访；最后看 end 和下一个同辈 pallocSum 的 start 聚合后是否满足，是则寻页成功.

![image-20260415181559460](https://images.liangning7.cn/typora/image-20260415181559460.png)

![image-20260415181627182](https://images.liangning7.cn/typora/image-20260415181627182.png)

## 对象分配流程

### 分配流程总览

Golang 中，依据 object 的大小，会将其分为下述三类：

- tiny 微对象【0~16B】
- small 小对象【16B~32KB】
- large 大对象【32KB~∞】

不同类型的对象，会有着不同的分配策略，这些内容在 mallocgc 方法中都有体现.

核心流程类似于读多级缓存的过程，由上而下，每一步只要成功则直接返回. 若失败，则由下层方法兜底.

**对于微对象的分配流程：**

1. 从 P 专属 mcache 的 tiny 分配器取内存（无锁）
2. 根据所属的 spanClass，从 P 专属 mcache 缓存的 mspan 中取内存（无锁）
3. 根据所属的 spanClass 从对应的 mcentral 中取 mspan 填充到 mcache，然后从 mspan 中取内存（spanClass 粒度锁）
4. 根据所属的 spanClass，从 mheap 的页分配器 pageAlloc 取得足够数量空闲页组装成 mspan 填充到 mcache，然后从 mspan 中取内存（全局锁）
5. mheap 向操作系统申请内存，更新页分配器的索引信息，然后重复（4）.

**对于小对象的分配流程**：跳过（1）步，执行上述流程的（2）-（5）步；

**对于大对象的分配流程**：跳过（1）-（3）步，执行上述流程的（4）-（5）步.

![image-20260415183821037](https://images.liangning7.cn/typora/image-20260415183821037.png)

## 若干线程发生OOM，会发生什么？Goroutine呢？如何解决？

如果线程发生OOM，也就是内存溢出，发生OOM的线程会被kill掉，其它线程不受影响。

如果Goroutine 发生OOM时，由于Goroutine的堆栈是动态扩展的，当一个Goroutine的堆栈无法扩展时，Go runtime 会尝试回收其他Goroutine的内存，以便为当前Goroutine分配更多的内存。如果回收失败，Go运行时会抛出一个运行时错误（如`runtime: out of memory`），但不会导致整个进程崩溃。

在实际项目中，可以采取以下措施：

1. 使用内存池`sync.Pool`，为频繁使用的对象创建内存池，以减少内存分配和垃圾回收的开销
2. 限制Goroutine的数量：使用协程池来限制并发的Goroutine数量，避免过多的Goroutine导致内存耗尽
3. 监控内存的使用，使用`pprof`定期监控程序的内存使用情况

# 2. GC

## 标记清除法

标记清除法：在Go V1.3之前使用的垃圾回收机制。

1. 暂停业务逻辑，找出不可达对象和可达对象
2. 开始标记，程序找出他所有可达的对象，并做上标记
3. 标记完了之后，然后开始清除未标记的对象
4. 暂停恢复，让程序继续跑。然后循环重复这个过程，直到 process 程序声明周期结束

缺点：

- STW，`stop the world`，让程序出现暂停，程序出现卡顿
- 标记需要扫描整个 heap
- 清除数据会产生heap碎片

优化：将第四步与第三步换位置，减少 STW 的范围

## 三色标记法

三色标记法：在 GoLang V1.5 使用。

1. 只要是新创建的对象，默认的颜色都是标记为白色
2. 每次GC回收开始，然后从根节点开始遍历所有对象，把遍历到的对象从白色集合放入灰色集合
3. 遍历灰色集合，将灰色对象引用的对象从白色对象放入灰色集合，之后将灰色对象放入黑色集合
4. 重复第三步，直到灰色集合没有任何对象
5. 回收所有的表示标记的对象，也就是回收垃圾

如果三色标记法不被STW保护：

- 条件1：一个白色对象被黑色对象引用【白色被挂在黑色下】
- 条件2：灰色对象与白色对象的可达性关系遭到破坏【灰色同时丢了该白色】

当两个条件同时满足，那么就会出现对象丢失的现象。

## 强弱三色不变式

- 强三色不变式：破坏条件1，强制性的不允许黑色对象引用白色对象
- 弱三色不变式：破坏条件2，黑色可以引用白色对象，但是需要白色对象存在其他灰色对象对它的引用；或者可达它的链路上游存在灰色对象。

如果三色标记满足强弱不等式之一，即可保存不丢失对象

## 插入屏障

对象被引用时，触发的机制：

具体操作：在A对象引用B对象的时候，B对象被标记为灰色。【将B挂在A下游，B必须被标记为灰色】

满足：强三色不变式【不存在黑色对象引用白色对象的情况了，因为白色会强制变为灰色】

缺点：结束时需要STW来重新扫描栈。因为栈空间不启动屏障机制，而如果栈空间内添加对象，也不会启动插入屏障，也就是插入了一个白色对象。不重新扫描一遍就会直接将其删除。在重新扫描时，会启动STW。

## 删除屏障

对象被删除时，触发的机制：

具体操作：要被删除的对象，如果为白色，那么被标记为灰色

满足：弱三色不变式【保护灰色对象到白色对象的路径不会断】

缺点：回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉

## 混合写屏障

在 GoLang 1.8 之后使用的GC回收机制。即三色标记法+混合写屏障

**栈上不启用屏障，堆启用屏障。**

具体操作：

1. GC开始时，将**栈上**的对象全部扫描标记为黑色【之后不再进行重复扫描，不需要STW】
2. GC期间，任何在**栈上**创建的新对象，都被标记为黑色
3. 被删除的对象被标记为灰色
4. 被添加的对象被标记为灰色

满足：变形的弱三色不等式【结合了插入、删除写屏障两者的优点】

优点：堆空间启动，栈空间不启动，整个过程几乎不需要STW，效率较高

## GC 触发机制

主动触发：`runtime.GC`

被动触发：

- 使用系统监控，当超过两分钟没有产生任何GC时，强制触发GC。
- 使用步调算法，其核心思想是控制内存增长的比例。是一种比例GC，下一次GC结束时的堆大小和上一次GC存活堆大小的比例。即`GOGC`

## GC 的流程

当前版本的 Go 以 STW 为界限，可以将 GC 划分为五个阶段：

1. GCMark 标记准备阶段，为并发标记做准备工作，启动写屏障 
2. STWGCMark 扫描标记阶段，与赋值器并发执行，写屏障开启并发
3. GCMarkTermination 标记终止阶段，保证一个周期内标记任务完成，停止写屏障 
4. STWGCoff 内存清扫阶段，将需要回收的内存归还到堆中，写屏障关闭并发 
5. GCoff 内存归还阶段，将过多的内存归还给操作系统。

## GC 如何调优

通过 `go tool trace`等工具进行查看

- 控制内存分配的速度，限制goroutine的数量，从而提高赋值器对CPU的利用率
- 减少并复用内存，可以使用 sync.Pool 来复用，需要频繁创建临时对象，例如提前分配足够的内存来降低多余的拷贝。
- 需要时，增大 GOGC 的值，降低GC的运行频率

## goroutine 泄漏

**泄漏原因：**

- goroutine 内进行channel/mutex等读写操作被一直阻塞
- goroutine 内的业务逻辑进入死循环，资源一直无法释放
- goroutine 内的业务逻辑进入长时间等待，有不断新增的 goroutine 进入等待

**泄漏场景：**

如果输出的 goroutines 数量是在不断的增加，就说明存在泄漏。

- nil channle：

  channel 如果忘记初始化，那么无论读还是写，都会造成阻塞

- 发送不接收：

  channel 发送数量超过 channel 的接收数量，就会造成阻塞

- 接收不发送：

  channel 接收数量超过 channel 的发送数量，就会造成阻塞

- http request body 未关闭

  `resp.Body.Close()`未调用时，goroutine 不会退出，一般发起 http 请求时，需要确保关闭 body

- 互斥锁忘记解锁：

  第一个协程获取 `sync.Mutex`加锁了，但是在他可能在处理业务逻辑，又或是忘记 `Unlock()`了，因此导致后面的协程想加锁，却因为锁未释放被阻塞。

- `sync.WaitGroup`使用不当：

  由于`wg.Add`的数量和`wg.Done`数量不匹配，因此在调用`wg.Wait`方法后，一直进行等待。

**如何排查：**

- 单个函数：调用`runtime.NumGoroutine`方法来打印执行代码前后 goroutine 的运行数量，进行前后比较，就能直到有没有泄漏
- 生成/测试环境：使用 `PProf`实时监测 goroutine 的数量。

# 3. Go sheduler

## GPM 指的是什么

即 用户态下的 goroutine 调度器

G（Goroutine）：我们所说的协程，为用户级的轻量级线程，每个 Goroutine 对象的 sched 保存着其上下文信息。

M（Machine）：对内核级线程的封装，M 的数量是动态调整的，并不是固定的，而是根据系统负载和资源动态调整的（真正干活的对象）

P（Processor）：即为 G 和 M 的调度对象，用来调度G和M之间的关联关系，其数量可通过 `GOMAXPROCS()`来设置，默认为 CPU 核心数。

**每个 M 必须绑定一个 P 才能运行 G**。

## GMP 调度流程及其时机

**调度流程图：**

![image-20240528183510560](https://gitee.com/liangningi/typora_picture/raw/master/Go/202405281835781.png)

1. 通过 `go func()` 创建一个 goroutine；
2. 有两个存储 G 的队列，一个是局部调度器 P 的本地队列，一个是全局 G 队列。新创建的G会先保存在 P 的本地队列中，如果P的本地队列已经满了就会保存在全局的队列中；
3. G只能运行在M中，一个M必须持有一个P，M和P是1：1的关系。M会从P的本地队列弹出一个可执行状态的G来执行，如果P的本地队列为空，就会想其他的MP组合偷取一个可执行的G来执行
4. 一个M调度G执行的过程是一个循环机制；
5. 当M执行某一个G的时候如果发生了`syscall`或者其他阻塞操作，M会阻塞【此时，发生阻塞的 G 和该 M 绑定】，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程（如果有空闲的线程可用就复用）来服务这个P；
6. 当M系统调用结束时候，这个M会尝试获取之前的P执行，如果在 M 阻塞的这段时间里，原 P 并没有被系统后台监控线程（sysmon）强行剥夺交给其他 M（比如阻塞时间极其短暂），那么 M 就会重新和这个原 P 绑定， M 和 P 重新绑定，G 恢复运行。。【Fast Path】
   - 如果原 P 已经被系统剥夺并分配给其他 M 工作了，此时当前 M 就会去全局的**空闲 P 队列（Idle P List）**中寻找是否有闲置的 P。如果有，M 就会和这个空闲 P 绑定，M 和新的 P 绑定，G 恢复运行。。
   - 如果原 P 被抢走了，且全局也没有空闲的 P，这意味着系统当前的计算资源已经跑满了，没有多余的逻辑处理器可以分配给这个 G。此时，M 必须接受现实，G 和 M 只能“分手”，线程M变成休眠状态，加入到空闲线程，然后这个G会被放入全局队列中。。

调度时机：

- 新建一个协程和协程执行完毕
- 阻塞的系统调用，如文件io、网络io
- channel、mutex等阻塞操作
- time.Sleep
- 垃圾回收之后
- 主动调用 `runtime.Gosched()`
- 运行过久或系统调用过久

## goroutine的状态和线程的状态

goroutine状态：

- `Gidle`：空闲状态，表示Goroutine未被调度执行。
- `Grunnable`：可运行状态，表示Goroutine已经准备好运行，等待被调度到一个线程（M）上执行。
- `Grunning`：运行状态，表示Goroutine正在一个线程（M）上执行。
- `Gsyscall`：系统调用状态，表示Goroutine正在执行阻塞的系统调用。
- `Gwaiting`：等待状态，表示Goroutine正在等待某个事件（如通道操作、同步原语等）。
- `Gdead`：死亡状态，表示Goroutine已经执行完成或被终止。

线程的状态：

**自旋**和**非自旋**

自旋线程：每当创建G时，会尝试唤醒休眠的线程，如果唤醒了休眠的线程，则该线程会尝试和P绑定，该P会调用G0，并且该P的本地队列没有G，则该状态为自旋线程

## 如果 goroutine 一直占用资源怎么办，GMP模型怎么解决这个问题

如果有一个goroutine一直占用资源的话，GMP模型会从正常模式转为饥饿模式，通过信号协作强制处理在最前的 goroutine 去分配使用。

## GMP 中的 work stealing 机制

当本地队列P没有G之后，如果全局队列有，则直接取`min(len(GRQ)/GOMAXPROCS + 1,len(GRQ)/2)`个到本地队列

work stealing 机制是一种用于调度协程的策略，当P本地队列或者是全局队列没有G可以运行时，尝试从其他P偷取G，以实现负载均衡，而不是直接销毁线程。【一般是偷取其它P的前半部分】

## GMP 中的 hand off 机制

当本线程因为G进行系统调用进行阻塞时，将该G和M进行绑定，把P交给其他空闲的线程【无法交给自旋线程，因为自旋线程等待的是G，而不是P】，可以从休眠M队列中唤醒M。

当被阻塞的G被唤醒时，优先去申请原来的P，如果原来的P已经被M处理，不在空闲P队列中，就去空闲P队列中，找一个P来处理，如果空闲P队列中也没有，则G加入全局队列，M进入休眠队列。

## 当开辟过多的G，导致本地队列无法装下，如何处理？

把该本地队列的G前半部分和新创建的G打散加入到全局队列中。

## Goroutine中的panic问题

当一个Goroutine发生panic时，它会导致当前Goroutine的执行终止，并在调用栈上执行延迟函数（defer）。如果没有捕获到panic（使用`recover`函数），则会导致该goroutine崩溃。需要注意的是，**一个Goroutine的panic会影响其他Goroutine的执行**。

defer函数只能捕获到当前Goroutine的panic，而不能捕获子Goroutine的panic。因此，在使用Goroutine时，需要确保在每个Goroutine内部处理好panic，避免程序崩溃。

## sysmon 有什么作用

sysmon 是一个管理线程或者说守护线程，其是对GMP调度架构的补充和兜底。GMP在某些情况下会出现单个 goroutine 长期占据时间片甚至一直占据时间片的情况。例如：

- 某个goroutine不执行主动调度、不调用系统调用、不做函数调用，就会一直运行直到goroutine退出；
- 某个goroutine处于syscall状态时也无法触发主动调度，可能会造成该goroutine长时间占据时间片；

sysmon 的作用就是处理类似上面的情况，其主要工作内容有：

- 定期查看netpoll有无就绪的任务，防止netpoll阻塞队列中的goroutine饥饿；
- 定期查看是否有p长时间(10ms)处于syscall状态，如有则将p的持有权释放以执行其他g；
- 定期查看是否有p长时间(10ms)没有调度，如有则对当前m发送信号，触发基于信号的异步抢占调度

# 4. 逃逸分析

## 逃逸分析是什么

逃逸分析的基本思想是**检查变量的声明周期是否是可知**的，即当一个对象的指针被多个方法或线程引用时，则称这个指针发生了逃逸。

逃逸分析决定一个变量是分配在**堆上**还是分配在**栈上**。

## 逃逸分析有什么作用

通过逃逸分析，可以尽量把那些不需要分配到堆上的变量直接分配到栈上，堆上的变量少了，会减轻堆内存分配的开销，同时也会减少垃圾回收（Garbage Collection，GC）的压力，提高程序的运行速度。

### 为什么分配在栈上的变量比分配在堆上快

**栈内存上分配的优势：**

* 简易的内存管理：只需要**移动栈顶指针**就可以添加和删除数据，并且栈上的变量通常具有很好的**数据局部性**，因此，栈内存的分配和回收都非常快速；

**堆内存分配的相对成本：**

* 堆内存**分配**通常需要使用更复杂的**内存管理算法**，堆内存**回收依赖 GC**，因此，堆分配和回收比栈内存慢得多

## 逃逸分析是怎么完成的

逃逸分析也就是由编译器决定哪些变量放在栈上，哪些放在堆中，编译器会根据变量是否被外部引用来决定是否逃逸：

1. 如果在**函数外部没有引用**，则**优先**放到**栈**中
   * 如果栈上放不下，则必定放到堆中
2. 如果在**函数外部存在引用**，则**必定放到堆**中

## 如何确定是否发生了逃逸

参考如下代码：

```go
package main

import "fmt"

func foo() *int {
	t := 3
	return &t
}
func main() {
	x := foo()
	fmt.Println(*x)
} 
```

通过编译参数 `-gcflags '-m -l'`可以查看编译过程中的逃逸分析，例如：

```bash
$ go build -gcflags '-m -l' 03-memoryEA/main.go
```

其中 `-gcflags` 参数用于启用编译器支持的额外标志。例如，

- `-m`，即 `Memory`， 用于输出编译器的优化细节（包括使用逃逸分析这种优化），相反可以使用 `-N` 来关闭编译器优化；
- `-l` ，即 `Inlining` ，用于禁用内联优化，防止逃逸被编译器通过内联彻底的抹除。

得到如下输出：

```bash
# command-line-arguments
03-memoryEA/main.go:6:2: moved to heap: t
03-memoryEA/main.go:12:13: ... argument does not escape
03-memoryEA/main.go:12:14: *x escapes to heap
```

`foo` 函数里的变量 `t` 逃逸了，和预想的一致，不解的是为什么 `main` 函数里的 `x` 也逃逸了？这是因为有些函数的参数为 `interface` 类型，比如 `fmt.Println(a ...interface{})`，编译期间很难确定其参数的具体类型，也会发生逃逸。

> 也可以通过反汇编语言看出变量是否发生逃逸。

## 返回局部变量的指针是否安全？

函数返回局部变量的指针是安全的。这是由于Go的编译器和运行时系统通过逃逸分析可以意识到该局部变量将在函数外被引用，它们会在堆（而不是栈）上为它分配内存。这样，即使函数结束，局部变量的内存位置仍会保留。

虽然在 Go 中返回局部变量的指针是安全的，**但对返回局部变量的指针进行操作不一定安全**。例如，对指向 slice 类型对象的指针进行了扩容操作，这个指针可能由于底层数组的改变就会变得无效。

## 常见发生逃逸的场景

1. 指针逃逸；

2. 动态类型；

3. 栈空间不足【栈空间大小 2 KB】；

4. 变量大小不确定【例如，编译期间无法确定 Slice 的长度】；

5. 闭包引用对象，闭包函数中局部变量 i 在后续函数是继续使用的，编译器将其分配到堆上；

   ```go
   package main
   
   func escape() func() int {
   	var i int = 1
   	return func() int {
   		i++
   		return i
   	}
   }
   
   func main() {
   	escape() // moved to heap: i
   }
   ```

> tips：无论变量的大小，只要是指针变量都会在堆上分配，所以对于小变量还是使用传值效率【而不是传指针】更高一些

