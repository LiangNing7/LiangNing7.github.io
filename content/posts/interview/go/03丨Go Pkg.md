+++
title = "Go Pkg"
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
series_order = 3

+++

# 1. unsafe

## Go语言如何获取私有字段

1. 在同一个包内，可以直接访问小写字母开头的私有成员。
2. 在其他包中，无法直接访问私有成员，但可以通过**公开的接口**来**间接访问**私有成员。
3. 使用**反射+un**来绕过 Go 语言的封装机制访问和修改私有字段。（不建议使用）

来看反射获取的例子：

```bash
.
├── main.go
└── private
    └── person.go
```

```go
// private/person.go
// private/person.go
package private

// Person 定义一个包含私有字段的结构体
type Person struct {
	name string
	age  int
}

// NewPerson 创建一个新的 Person 对象
func NewPerson(name string, age int) Person {
	return Person{name: name, age: age}
}

```

```go
// main.go
package main

import (
	"fmt"
	"reflect"
	"unsafe"

	"github.com/LiangNing7/study/01-privateField/private"
)

func main() {
	// 创建一个 Person 对象.
	p := private.NewPerson("LiangNing7", 22)

	// 这里直接访问会编译错误，无法找到 name 和 age 字段.
	// fmt.Println(p.name, p.age)

	// 为了能够修改或者获取未导出字段，需要获取 p 的可寻址值.
	vp := reflect.ValueOf(&p).Elem()

	// 获取私有字段 "name".
	nameField := vp.FieldByName("name")
	name := reflect.NewAt(nameField.Type(), unsafe.Pointer(nameField.UnsafeAddr())).Elem().Interface().(string)

	// 获取私有字段 "age"
	ageField := vp.FieldByName("age")
	age := reflect.NewAt(ageField.Type(), unsafe.Pointer(ageField.UnsafeAddr())).Elem().Interface().(int)

	fmt.Printf("获取到的私有字段：name = %q, age = %d\n", name, age)
}
```

## 如何利用 unsafe 获取 slice 和 map 的长度

Slice 在 Go 里是一个结构体，通常表示为 `reflect.SliceHeader`，slice header 的定义：

```go
// runtime/slice.go
type slice struct {
	array unsafe.Pointer // 元素指针
	len int // 长度
	cap int // 容量
}  
```

使用 make 新建一个 slice，底层调用的是 makeslice 函数，返回的是 slice 结构体指针：

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer
```

因此可以通过 unsafe.Pointer 和 uintptr 进行转换，进而得到 slice 的长度和容量：

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	// 返回的是 slice header.
	/*
	 array unsafe.Pointer // 元素指针 8 字节
	 len int // 长度 64 位系统 8 字节
	 cap int // 容量 64 位系统 8 字节
	*/
	s := make([]int, 9, 20)
	var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(8)))
	fmt.Println(Len, len(s)) // 9 9

	var Cap = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(16)))
	fmt.Println(Cap, cap(s)) // 20 20
}
```

转换过程如下：

```text
Len: &s => pointer => uintptr => pointer => *int => int
Cap: &s => pointer => uintptr => pointer => *int => int 
```



与 slice 不同，Map 是 Go 的一种引用类型。当你使用 `make` 构造一个 map 时，返回的值看起来是一个 map，但它其实内部只是一个指向运行时数据结构的指针。【make 创建的 slice 返回对应的结构体，map返回创建的数据结构的指针】

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap 
```

依然能通过 unsafe.Pointer 和 uintptr 进行转换，得到 hamp 字段的值，只不过，现在 count 变成二级指针了：

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	mp := make(map[string]int)
	mp["qcrao"] = 100
	mp["stefno"] = 18

	count := **(**int)(unsafe.Pointer(&mp))
	fmt.Println(count, len(mp)) // 2 2
}
```

所以 count 的转换过程如下：

```text
&mp => pointer => **int => int
```

## 如何实现字符串和 byte 切片的零拷贝转换

实现字符串和 bytes 切片之间的转换，要求是 zero-copy。为了完成这个转换，需要了解 slice 和 string 的底层数据结构：

```go
// src/reflect/value.go
type StringHeader struct {
	Data uintptr
	Len  int
}
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

上面的代码是反射包下的结构体，只需要共享底层 Data 和 Len 就可以实现 zero-copy：

```go
func string2bytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}
func bytes2string(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

原理上是利用指针的强转，代码比较简单，例如：

```go
func StringToBytes(s string) []byte {
    if s == "" {
        return nil
    }
    // 使用 unsafe.SliceData 获取字符串底层指针，再构造切片。
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

func BytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

但要注意这种这种方式：

```go
// StringToBytes 将字符串转换为 []byte，不发生数据拷贝
func StringToBytes(s string) []byte {
	// 利用 reflect.StringHeader 获取字符串的底层数据结构
	strHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))
	// 构造一个 slice header，其 Data、Len、Cap 与字符串保持一致
	sliceHeader := reflect.SliceHeader{
		Data: strHeader.Data,
		Len:  strHeader.Len,
		Cap:  strHeader.Len,
	}
	// 将 slice header 转换为 []byte
	return *(*[]byte)(unsafe.Pointer(&sliceHeader))
}
```

原因在于 `stringHeader.Data` 本身是 uintptr 类型，由于 goroutine 的栈空间可能发生移动，因 此不能将其作为中间态的值复制到 bh，再转换为 []byte。

# 2. context

## context 是什么

context 译作 “上下文”，准确来说是 goroutine 上下文。主要作用了两个：**控制链路**和**安全传值**，并且 context 是并发安全的。

## 底层原理

[【TODO】](https://jianghushinian.cn/2024/12/09/context/)

# 3. error

# 4. Timer

# 5. reflect

# 6. sync

## mutex底层原理



### Mutex 几种状态

- `mutexLocked`：表示互斥锁的锁定状态
- `mutexWoken`：表示从正常模式被唤醒
- `mutexStarving`：当前的互斥锁进入饥饿状态
- `waitersCount`：当前互斥锁上等待的 Goroutine 个数

### goroutine 唤醒标识

在实现上，sync.Mutex 通过一个 mutexWoken 标识位，标志出当前是否已有 goroutine 在自旋抢锁或存在 goroutine 从阻塞队列中被唤醒；倘若 mutexWoken 为 true，且此时有解锁动作发生时，就没必要再额外唤醒阻塞的 goroutine 从而引起竞争内耗

### Mutex 从自旋到阻塞的升级过程

一个优秀的工具需要具备探测并适应环境，从而采取不同对策因地制宜的能力【面向性能】

针对 goroutine 加锁时，发生锁已经被抢占的情形，有如下两种策略：

- 阻塞/唤醒：将当前 goroutine 阻塞挂起，直到锁被释放后，以回调的方式将阻塞 goroutine 重新唤醒，进行锁争夺
- 自旋 + CAS：基于自旋结合CAS的方式，重复校验锁的状态并尝试获取锁，始终把主动权握在手中

上述方案各有优劣，且有其使用场景：

| **锁竞争方案** | **优势**                       | **劣势**                               | **适用场景**         |
| -------------- | ------------------------------ | -------------------------------------- | -------------------- |
| 阻塞/唤醒      | 精准打击，不浪费 CPU 时间片    | 需要挂起协程，进行上下文切换，操作较重 | 并发竞争激烈的场景   |
| 自旋+CAS       | 无需阻塞协程，短期来看操作较轻 | 长时间争而不得，会浪费 CPU 时间片      | 并发竞争强度低的场景 |

sync.Mutex 结合两种方案的使用场景，制定了一个锁升级的过程，反映了面对不同并发环境通过持续试探逐渐由乐观转为悲观的态度，具体方案如下：

- 首先保持乐观，goroutine采用自旋+CAS的策略争夺锁
- 尝试持续受挫达到一定条件之后，判定当前过于激烈，则由自旋转为 阻塞/唤醒 模型

具体条件为：

- 自旋累计达到4次仍未取得
- CPU 单核或仅有单个 P 调度器；此时自旋，其他goroutine根本没有机会释放锁，自旋纯属空转
- 当前P的执行队列仍有待执行的G

### Mutex 正常模式转为饥饿模式

饥饿模式主要解决【公平性】的问题

首先理解两个概念：

- 饥饿：因为非公平机制的原因，导致 Mutex 阻塞队列中存在 goroutine 长时间取不到锁，从而陷入饥荒状态
- 饥饿模式：当 Mutex 阻塞队列中存在处于饥饿态的 goroutine 时，会进入饥饿模式，将抢锁流程由非公平机制转为公平机制.

在 sync.Mutex 允许过程中，存在两种模式：

- 正常模式/非饥饿模式：这是 sync.Mutex 默认采用的模式，当有 goroutine 从阻塞队列被唤醒时，会和此时先进入抢锁流程的 goroutine 进行锁资源的争夺，假如抢锁失败，会重新回到阻塞队列头部

  （值得一提的是，此时被唤醒的老 goroutine 相比新 goroutine 是处于劣势地位，因为新 goroutine 已经在占用 CPU 时间片，且新 goroutine 可能存在多个，从而形成多对一的人数优势，因此形势对老 goroutine 不利.）

- 饥饿模式：这是 sync.Mutex 为拯救饥荒的老 goroutine 而启用的特殊机制，饥饿模式下，锁的所有权按照阻塞队列的顺序进行依次传递，新 goroutine 进入流程时不得抢锁，而是进入队列尾部排队。

两种模式的转换条件：

- 默认是正常模式；
- 正常模式 -> 饥饿模式：当阻塞队列存在 goroutine 等锁超过 1ms 而不得，则进入饥饿模式；
- 饥饿模式 -> 正常模式：当阻塞队列已清空，或取得锁的 goroutine 等锁时间已低于 1ms 时，则回到正常模式；

总结：正常模式灵活机动，性能较好；饥饿模式严格死板，但能捍卫公平的底线。因此，两种模式的切换体现了 sync.Mutex 为适应环境变化，在公平与性能之间做出的调整与平衡。

### Mutex 允许自旋的条件

1. 锁已被占用，并且锁不处于饥饿模式
2. 积累的自旋次数小于最大自旋次数（active_spin=4）
3. cpu 核数大于 1
4. 有空闲的 P
5. 当前 goroutine 所挂载的 P 下，本地待运行队列为空

### sync.Mutex 的数据结构

```go
type Mutex struct {
	state int32
    sema  uint32
}
```

- state：锁中最核心的状态字段，不同bit位分别存储了 mutexLocked【是否上锁】、mutexWoken【是否有 goroutine 从阻塞队列中被唤醒】、mutexStarving【是否处于饥饿模式】的信息
- sema：用于阻塞和唤醒 goroutine 的信号量

### sync.Mutex 的几个全局变量

```go
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota

    starvationThresholdNs = 1e6
)
```

- mutexLocked = 1：state 最右侧的一个bit位标志是否上锁，0-未上锁，1-已上锁
- mutexWoken = 2：state 右数第二个 bit 位标志是否有 goroutine 从阻塞中被唤醒，0-没有，1-有
- mutexStarving = 4：state 右数第三个 bit 位标志 Mutex 是否处于饥饿模式，0-非饥饿，1-饥饿
- mutexWaiterShift = 3：右侧存在3个bit位标识特殊信息，分别为上述的 mutexLocked、mutexWoken、mutexStarving
- starvationThresholdNs = 1ms：sync.Mutex 进入饥饿模式的等待时间阈值，1e6纳秒

### sync.Mutex 中 state 字段

Mutex.state 字段为 int32 类型，不同 bit 位具有不同的标识含义：

低 3 位分别标识 mutexLocked（是否上锁）、mutexWoken（是否有协程在抢锁）、mutexStarving（是否处于饥饿模式），高 29 位的值聚合为一个范围为 0~2^29-1 的整数，表示在阻塞队列中等待的协程个数。

- state & mutexLocked：判断是否上锁；
- state | mutexLocked：加锁；
- state & mutexWoken：判断是否存在抢锁的协程；
- state | mutexWoken：更新状态，标识存在抢锁的协程；
- state &^ mutexWoken：更新状态，标识不存在抢锁的协程；
- state & mutexStarving：判断是否处于饥饿模式；
- state | mutexStarving：置为饥饿模式；
- state >> mutexWaiterShif：获取阻塞等待的协程数；
- state += 1 << mutexWaiterShif：阻塞等待的协程数 + 1.

## RWMutex 底层原理

【TODO】

### RWMutex 实现

通过记录 readerCount 读锁的数量来进行控制，当有一个写锁的时候，会将读锁的数量设置为负数。目的是让新进入的读锁等待写锁之后释放通知读锁。同样的写锁也会等等待之前的读锁都释放完毕，才会开始进行后续的操作。 而等写锁释放完之后，会将值重新加上 1<<30, 并通知刚才新进入的读锁（rw.readerSem)，两者互相限制

### RWMutex 注意事项

- RWMutex 是单写多读锁，该锁可以加多个读锁或者一个写锁
- 读锁占用的情况下会阻止写，不会阻止读，多个 goroutine 可以同时获取读锁
- 写锁会阻止其他 goroutine（无论读和写）进来，整个锁由该 goroutine 独占
- 适用于读多写少的场景
- RWMutex 类型变量的**零值**是一个**未锁定状态的互斥锁**。
- RWMutex 在首次被使用之后就不能再被拷贝。
- RWMutex 的读锁或写锁**在未锁定状态**，**解锁操作都会引发 panic**。
- RWMutex 的一个写锁 Lock 去锁定临界区的共享资源，如果临界区的共享资源已被（读锁或写锁）锁定，这个写锁操作的 goroutine 将被阻塞直到解锁。 
- RWMutex 的**读锁不要用于递归调用**，比较容易产生死锁。
- RWMutex 的锁定状态与特定的 goroutine 没有关联。一个 goroutine 可以 RLock（Lock），另一个 goroutine 可以 RUnlock（Unlock）。 
- 写锁被解锁后，所有因操作锁定读锁而被阻塞的 goroutine 会被唤醒，并都可以成功锁定读锁。
- 读锁被解锁后，在没有被其他读锁锁定的前提下，所有因操作锁定写锁而被阻塞的 goroutine，其中等待时间最长的一个 goroutine 会被唤醒。

## WaitGroup

### WaitGroup 用法

一个 WaitGroup 对象可以等待一组协程结束。使用方法是：

1. main 协程通过调用`wg.Add(delta int)`设置 worker 协程的个数，然后创建 worker 协程
2. worker 协程执行结束以后，都要调用`wg.Done()`
3. main 协程调用`wg.Wait()`且被 block，直到所有 worker 协程全部执行结束后返回。

### WaitGroup 实现原理

- WaitGroup 主要维护了 2 个计数器，一个是请求计数器 v，一个是等待计数器 w，二者组成一个 64bit 的值，请求计数器占高 32bit，等待计数器占低 32bit。
- 每次 Add 执行，请求计数器 v 加 1，Done 方法执行，请求计数器减 1，v 为 0 时通过信号量唤醒 Wait()。

## Cond 是什么

Cond 实现了一种条件变量，可以使用在多个 Reader 等待共享资源 ready 的场景（如果只有一读一写，一个锁或者 channe就可以了）

每个 Cond 都会关联一个 Lock(`sync.Mutex or sync.RWMutex`)，当修改条件或者调用 `Wait`方法时，必须加锁，保护condition。

## Cond 中 Wait 使用

```go
func (c *Cond) Wait()
```

`Wait()`会自动释放`c.L`，并挂起调用者的 goroutine。之后恢复执行，`Wait()`会在返回时对`c.L`加锁。

除非被 Signal 或者 Broadcast 唤醒，否则`Wait()`不会返回

由于 `Wait()` 第一次恢复时，`C.L` 并没有加锁，所以当 `Wait` 返回时，调用者通常并不能假设条件为真。

取而代之的是，调用者应该在循环中调用 Wait。（简单来说，只要想使用condition，就必须加锁。）

```go
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of conditon ...
c.L.Unlock()
```

使用示例如下：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var mu sync.Mutex
	// 创建一个与 mutex 关联的条件变量
	cond := sync.NewCond(&mu)
	
	// 共享的条件状态
	ready := false 

	// 1. 启动 3 个运动员 (Goroutine)
	for i := 1; i <= 3; i++ {
		go func(id int) {
			cond.L.Lock() // 检查条件前必须加锁
			
			// 必须使用 for 循环检查条件
			for !ready {
				fmt.Printf("运动员 %d: 在起跑线等待...\n", id)
				cond.Wait() // 挂起并解锁，被唤醒后会重新加锁
			}
			
			fmt.Printf("运动员 %d: 冲出起跑线！\n", id)
			cond.L.Unlock() // 执行完毕后解锁
		}(i)
	}

	// 2. 模拟裁判准备工作
	fmt.Println("裁判: 准备发令枪...")
	time.Sleep(2 * time.Second)

	// 3. 改变条件状态 (必须加锁保护共享变量)
	cond.L.Lock()
	ready = true
	cond.L.Unlock()

	// 4. 发出通知
	fmt.Println("裁判: 砰！(Broadcast 唤醒所有人)")
	cond.Broadcast() // 唤醒所有处于 Wait 状态的 Goroutine

	// 等待一小段时间，确保子 Goroutine 能打印完毕 (实际开发中应使用 WaitGroup)
	time.Sleep(1 * time.Second)
}
```

## Broadcast 和 Signal 区别

```go
func (c *Cond) Broadcast()
```

Broadcast 会唤醒所有等待 c 的 goroutine。

调用 Broadcast 的时候，可以加锁，也可以不加锁

```go
func (c *Cond)Signal()
```

Signal 只唤醒1个等待 c 的 goroutine

调用 Signal 的时候，可以加锁，也可以不加锁

## 什么是 sync.Once

- `Once` 可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景。
- `Once` 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。 
- `sync.Once` 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。

## 什么操作叫做原子操作

一个或者多个操作在 CPU 执行过程中不被中断的特性，称为原子性(atomicity)。这些操作对外表现成一个不可分割的整体，他们要么都执行，要么都不执行，外界不会看到他们只执行到一半的状态。

而在现实世界中，CPU 不可能不中断的执行一系列操作，但如果我们在执行多个操作时，能让他们的中间状态对外不可见，那我们就可以宣城他们拥有了“不可分割”的原子性。

## 原子操作和锁的区别

原子操作由底层硬件支持，而锁则由操作系统的调度器实现。锁应当用来保护一段逻辑，对于一个变量更新的保护，原子操作通常会更有效率，并且更能利用计算机多核的优势，如果要更新的是一个复合对象，则应当使用 `atomic.Value` 封装好的实现。

## 什么是 CAS

CAS 的全称为 Compare And Swap，直译就是比较交换。是一条 CPU 的原子指令，其作用是让 CPU 先进行比较两个值是否相等，然后原子地更新某个位置的值，其实现方式是给予硬件平台的汇编指令，在 intel 的 CPU 中，使用的 cmpxchg 指令，就是说 CAS 是靠硬件实现的，从而在硬件层面提升效率。

**简述过程是这样的：**

假设包含 3 个参数内存位置(V)、预期原值(A)和新值(B)。V 表示要更新变量的值，E 表示预期值，N 表示新值。仅当 V 值等于 E 值时，才会将 V 的值设为 N， 如果 V 值和 E 值不同，则说明已经有其他线程在做更新，则当前线程什么都不 做，最后 CAS 返回当前 V 的真实值。CAS 操作时抱着乐观的态度进行的，它总 是认为自己可以成功完成操作。基于这样的原理，CAS 操作即使没有锁，也可 以发现其他线程对于当前线程的干扰。

## sync.Pool 有什么用

对于很多需要重复分配、回收内存的地方，sync.Pool 是一个很好的选择。频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺，而 sync.Pool 可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。

例如：管理临时的切片对象
