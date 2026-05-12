+++
title = "Go Base"
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
series_order = 1

+++

# 1. 语言特性

## 与其他语言相比，使用 Go 的好处

Go 语言有 5 个特性，分别是**简单、显式、组合、并发和面向工程**。

1. **简单**

   * 关键字在主流编程语言中最少，仅 25 个关键字；
   * 内置垃圾收集（GC），明显比 Java 和 Python 更有效；
   * 内置并发支持，简化并发程序设计；
   * 单⼀的标准代码格式，Golang 通常被认为⽐其他语⾔更具可读性；
   * …

2. **显式**：所有行为、依赖和错误处理都必须通过清晰、明确的语法和接口来表达，而不依赖隐式或神秘的自动机制，从而让程序的意图一目了然、易于理解与维护

3. **组合**：Go 提供**垂直组合**与**水平组合**

   * **垂直组合**，即通过类型嵌入，快速让一个新类型“复用”其他类型已经实现的能力，实现功能的垂直扩展；
   * **水平组合**，是一种能力委托（Delegate），通常使用接口类型来实现水平组合。

4. **并发**：将面向多核、**原生支持并发**作为了新语言的设计原则之一，Go 放弃了传统的基于操作系统线程的并发模型，而采用了**用户层轻量级线程**，Go 将之称为**goroutine**。

5. **面向工程**：指的是语言和工具的设计始终以解决现实工程问题为出发点，这些问题包括**程序构建慢**、**依赖管理失控**、**代码难于理解**、**跨语言构建难**等。

   > Go 语言的交叉编译十分方便，只需要设置一下`goenv`
   >
   > * **`GOOS`**：目标操作系统（如 `windows`、`linux`、`darwin`）。
   > * **`GOARCH`**：目标架构（如 `amd64`、`arm64`、`386`）。
   > * **`CGO_ENABLED`**：通常设为 `0` 禁用 CGO，避免依赖本地库。
   >
   > ```go
   > GOOS=<目标系统> GOARCH=<目标架构> CGO_ENABLED=0 go build -o <输出文件名>
   > ```

# 2. defer 语句

> 先进后出；参数传递、引用传递；匿名返回值、命名返回值；return 流程；panic、recover；

## defer 是什么

defer 是 Go 语言提供的一种用于**注册延迟调用**的机制：让函数或语句可以在当前函数执行完毕后（包括通过 return 正常结束或者 panic 导致的异常结束）执行。

### defer 的使用场景

defer 在需要释放资源的场景非常有用，可以很方便地在函数结束前做一些清理操作。在打开资源语句的下一行，直接使用 defer 就可以在函数返回前释放资源。

**常用场景：**

* 打开 / 关闭连接；
* 加 / 释放锁；
* 打开 / 关闭文件；
* …

当然，defer 会有短暂延迟，对时间要求特别高的程序，可以避免使用它，其他情况一般可以忽略它带来的延迟。特别是 Go 1.14 又对 defer 做了很大幅度的优化，效率提升了不少。

## defer 执行顺序

每次 defer 语句执行的时候，会把函数“压栈”，函数参数会被复制下来； 当外层函数（注意不是代码块，如一个 for 循环块并不是外层函数）退出时，defer 函数按照定义的顺序逆序执行；如果 defer 执行的函数为 nil，那么会在最终调用函数的时候产生 panic。

在 defer 函数定义时，对外部变量的引用有两种方式：**函数参数、闭包引用**。前者在 defer 定义时就把值传递给 defer，并被 cache 起来；后者则会在 defer 函数**真正调用**时根据整个上下文确定参数当前的值。

- ```go
  // 函数参数
  func passByValue() {
      x := 10
      // 此时 x=10 已经被作为参数传入，Go 会立刻记录下 10 这个值
      defer fmt.Println("值传递:", x) 
      
      x = 20 // 这里的修改不会影响 defer 已经缓存好的快照
  }
  // 最终输出: 值传递: 10
  ```

- ```go
  // 闭包引用
  func closureReference() {
      x := 10
      // 这是一个匿名函数，内部直接引用了外部的 x，没有传参
      defer func() {
          fmt.Println("闭包引用:", x) // 执行到这里时，去读 x 的最新值
      }()
      
      x = 20 // 这里的修改会被闭包感知到
  }
  // 最终输出: 闭包引用: 20
  ```

defer 后面的函数在执行的时候，函数调用的参数会被保存起来，也就是复制了一份。真正执行的时候，实际上用到的是这个复制的变量，因此如果此变量是一个“值”，那么就和定义的时候是一致的。如果此变量是一个“引用”，那就可能和定义的时候不一致。

```go
func referenceType() {
    arr := []int{1, 2, 3} // 切片本质上包含指向底层数组的指针
    
    // 拷贝了切片的描述符（包含指针），但没有深拷贝底层数据
    defer fmt.Println("引用类型传递:", arr) 
    
    arr[0] = 99 // 修改了底层数组的内容
}
// 最终输出: 引用类型传递: [99 2 3]
```

### Go 语言中 defer 的变量快照在什么情况下会失效

在 Go 语言中，`defer` 的变量快照是指在 `defer` 语句**定义时所捕获的变量的状态**。

1. **匿名函数闭包**：当 `defer` 语句中使用的**匿名函数捕获了外部变量**时。如果变量的值在 `defer` 语句定义后发生变化，`defer` 执行时会使用变化后的值。
2. **引用类型**：当 `defer` 引用持有**引用类型的变量**（如指针、切片、映射、通道和函数）时，虽然引用本身的地址不会变，但指向的内容可能变化，这也会导致最终的结果和预期的快照结果不同。

### return 之后定义的 defer 语句会被执行吗？

```go
func main() {
	defer func(){
		fmt.Println("before return")
	}()
	if true {
		fmt.Println("during return")
		return
	}
	defer func(){
		fmt.Println("after return")
	}()
} 
```

运行结果：

```bash
during return
before return
```

return 之后的定义的 defer 函数不能被注册，因此不能打印出 after return。

### defer 的妙用

场景：在一个函数里，需要打开两个文件进行合并操作，合并完成之后，在函数结束前关闭打开的文件句柄。

```go
func mergeFile() error {
	f, _ := os.Open("file1.txt")
	if f != nil {
		defer func(f io.Closer) {
			if err := f.Close(); err != nil {
				fmt.Printf("defer close file1.txt err %v\n", err)
			}
		}(f)
	}
	// write file1.txt ...
	f, _ = os.Open("file2.txt")
	if f != nil {
		defer func(f io.Closer) {
			if err := f.Close(); err != nil {
				fmt.Printf("defer close file2.txt err %v\n", err)
			}
		}(f)
	}
	// write file2.txt ...
	return nil
}
```

上面的代码中就用到了 defer 的原理，defer 函数定义的时候，参数就已经复制进去了，之后，真正执行 `close()` 函数的时候就刚好关闭的是正确的“文件”了，很巧妙。如果不这样，将 f 当成函数参数传递进去的话，最后两个语句关闭的就是同一个文件了：都是最后一个打开的文件。

在调用 `close()` 函数的时候，要注意一点：先判断调用主体是否为空，否则可能会解引用了一个空指针，进而 panic。

## 如何拆解 defer 语句

防止误用 defer 陷入泥潭，就必须深刻理解这句话：

```go
return xxx
```

这条语句，经过编译后，实际产生了三条语句：

1. 返回值 = xxx；
2. 调用 defer 函数；
3. 空的 return。

这样看来，第 1 句和第 3 句生成的指令并不是一条原子指令，第二句是 defer 定义的语句，这里可能会操作返回值，从而影响最终结果。

第一个例子：

```go
func f() (r int) {
	t := 5
	
	defer func() {
		t = t + 5
	}()
	return t
}
```

拆解之后就会变成：

```go
func f() (r int) {
	t := 5

	// 1. 赋值指令.
	r = t

	// 2. defer 被插入到赋值与返回之间执行，这个例子中返回值 r 没被修改过.
	func() {
		t = t + 5
	}
	
	// 空的 return
	return
}
```

这里第二步实际上并没有操作返回值 r，因此，main 函数中调用 f() 得到 5。

再来看第二个例子：

```go
func f() (r int) {
	defer func(r int) {
		r = r + 5
	}(r)
	return 1
}
```

1. 命名返回值初始化，r = 0
2. 注册 defer 函数：以当前`r`的值（此时为`0`）作为参数**立即求值并注册**。
3. 执行 return 1，将命名返回值`r`的值设置为`1`。此时外部的`r`变为`1`。
4. 执行 defer 函数，执行已注册的`defer`函数。匿名函数内部的局部变量`r`（值为`0`）被修改为`0 + 5 = 5`。由于这是对局部参数的修改，**外部的命名返回值`r`不受影响**，仍然为`1`。
5. **最终返回值**：函数返回外部的`r`，即`1`。

拆解后：

```go
func f() (r int) {
	// 1. 赋值.
	r = 1
	
	// 2. 这里改的 r 是之前传进去的 r，不会改变要返回的那个 r 值.
	func(r int) {
		r = r + 5
	}(r)
	
	// 3. 空的 return.
	return
}
```

再看最后一个例子：

```go
func f() (r int) {
    defer func() {
        r += 5 // 直接操作外部r
    }()
    return 1 // 最终返回6
}
```

拆解后：

```go
func f() (r int) {
	// 1. 赋值.
	r = 1

	// 2. 执行注册的 defer
	func() {
		r = r + 5
	}()

	// 3. 空的 return.
	return
}
```

## 如何确定 defer 的参数

```go
func f1() {
	var err error
	defer fmt.Println(err)
	err = errors.New("defer1 error")
	return
}
func f2() {
	var err error
	defer func() {
		fmt.Println(err)
	}()
	err = errors.New("defer2 error")
	return
}
func f3() {
	var err error
	defer func(err error) {
		fmt.Println(err)
	}(err)
	err = errors.New("defer3 error")
	return
}
func main() {
	f1()
	f2()
	f3()
}
```

运行结果：

```bash
<nil>
defer2 error
<nil> 
```

第 1 和第 3 个函数中，因为作为参数，err 在函数定义的时候就会求值，并且定义的时候 err 的值都是 nil，所以最后打印的结果都是 nil；第 2 个函数的参数其实也会在定义的时候求值，但第 2 个例子中是一个闭包，它引用的变量 err 在执行的时候值最终变成 defer2 error 了。

现实中第 3 个函数比较容易犯错误，在生产环境中，很容易写出这样的错误代码，导致最后 defer 语句没有起到作用，造成一些线上事故，要特别注意。

## 闭包是什么

闭包是由函数及其相关引用环境组合而成的实体，即：**闭包=函数+引用环境**。

一般的函数都有函数名，而匿名函数没有。匿名函数不能独立存在，但可以直接调用或者赋值于某个变量。匿名函数也被称为闭包，一个闭包继承了函数声明时的作用域。在 Go 语言中，所有的匿名函数都是闭包。

不太恰当的比喻：可以**把闭包看成是一个类，一个闭包函数调用就是实例化一个类**。闭包在运行时可以有多个实例，它会**将同一个作用域里的变量和常量捕获下来**，无论闭包在什么地方被调用（实例化）时，都可以使用这些变量和常量。而且，**闭包捕获的变量和常量是引用传递，不是值传递**。

例子：

```go
func main() {
	var a = Accumulator()
	fmt.Printf("%d\n", a(1))
	fmt.Printf("%d\n", a(10))
	fmt.Printf("%d\n", a(100))
	fmt.Println("------------------------")

	var b = Accumulator()
	fmt.Printf("%d\n", b(1))
	fmt.Printf("%d\n", b(10))
	fmt.Printf("%d\n", b(100))
}
func Accumulator() func(int) int {
	var x int
	return func(delta int) int {
		fmt.Printf("(%+v, %+v) - ", &x, x)
		x += delta
		return x
	}
}
```

执行结果是：

```bash
$ go run 04-closure/main.go 
(0xc000010100, 0) - 1
(0xc000010100, 1) - 11
(0xc000010100, 11) - 111
------------------------
(0xc000010120, 0) - 1
(0xc000010120, 1) - 11
(0xc000010120, 11) - 111
```

闭包引用了 x 变量，a，b 可看作 2 个不同的实例，实例之间互不影响。实例内部，x 变量 是同一个地址，因此具有“累加效应”。

## 闭包经典题

```go
package main

import "fmt"

// 题目 1：基础的捕获方式对比
func f1() {
	i := 0
	defer fmt.Println("f1-A:", i)
	defer func() {
		fmt.Println("f1-B:", i)
	}()
	i++
}

// 题目 2：defer 与匿名返回值
func f2() int {
	i := 1
	defer func() {
		i++
	}()
	return i
}

// 题目 3：defer 与命名返回值
func f3() (i int) {
	i = 1
	defer func() {
		i++
	}()
	return i
}

// 题目 4：进阶组合（参数传递 + 命名返回值）
func f4() (i int) {
	defer func(j int) {
		i += j
	}(i)
	return 2
}

func main() {
	f1()
	fmt.Println("f2 return:", f2())
	fmt.Println("f3 return:", f3())
	fmt.Println("f4 return:", f4())
}
```

结果如下：

```go
f1():
    f1-B: 1
    f1-A: 0
f2():
	1
f3():
	2
f4():
	2
```

分析：

- `f1()`：defer 先进后出，所以先输出后面的闭包，由于闭包是引用传递，引用的是指针，由于后面 `i++` ，所以先输出1，而第一个defer是值传递，值传递在当时就会计算出结果，所以是输出 0；
- `f2()`：这里使用的是匿名返回值，由于 `return xxx` 的步骤，会先给返回值赋值，即 `返回值 = 1`，然后执行 defer 中的 `i++`，然后再 return。由于是匿名返回值，返回的就是返回值；
- `f3()`：这里使用的是命名返回值，和 `f2` 类似，不过返回值返回是的 `i`，而 `i` 受到了闭包的影响，所以返回值为 `i = 2`；
- `f4()`：这里有闭包的参数传递和命名返回值，参数传递，在当时就会计算出结果，所以在闭包中，传入的参数 `i` 为 0，而闭包里的 `i` 是引用传递，接受到的就是 `return 2` 里的 2，在闭包里计算后，命名返回值为 2，所以 `f4()` 的结果为 2。

## defer 语句如何配合 recover 语句

panic 会停掉当前正在执行的程序，而不只是当前线程。在这之前，它会有序地执行完当前线程 defer 列表里的语句，其他协程里定义的 defer 语句不作保证。所以在 defer 里定义一个 recover 语句，防止程序直接挂掉，就可以起到类似 Java 里 try...catch 的效果。

### panic 和 recover 的底层运行原理：

首先整个 panic -> recover 的流程中，有四个关键对象在协同（或对抗）工作：

1. **业务函数（注册函数）**：写了 `defer` 语句，并且由于某种原因触发了 `panic` 的那个倒霉函数。当崩溃发生时，它会被立刻挂起，失去执行控制权。
2. **`_defer` 结构体**：Go 运行时的“时间胶囊”。它是一份延迟执行档案，以链表形式存在。里面存储了：
   - `fn`：延迟函数的内存地址。
   - `sp` / `pc`：注册 `defer` 时的栈指针和程序计数器（用于还原现场）。
   - `_panic`：关联当前正在触发它的那个 `panic` 对象。
3. **延迟函数 D**：你在 `defer` 后面挂载的那个函数（通常是匿名函数 `func() {...}`）。它是被底层系统**亲手唤醒**的执行单元。
4. **Go Runtime（运行时系统）**：真正的幕后控制者，分为两个关键组件：
   - **`gopanic`（抢救室医生）**：接管崩溃现场，提取 `_defer` 结构体，分配栈空间，并亲自调用“延迟函数 D”。
   - **`gorecover`（安检员）**：处在调用栈的最顶端，负责极其严苛的“栈帧指针比对”，决定是否允许吞掉当前的 `panic`。

在 Go 语言的底层运行机制中，`panic` 和 `defer` 都是和当前的 Goroutine 强绑定的。

- **触发 `panic`：** 当程序发生 `panic` 时，当前 Goroutine 立即进入底层的 `runtime.gopanic(v)`。它会做以下几件事：
  - 组装 `_panic` 结构体并挂载；
  - 接下来，`gopanic` 进入一个死循环：不断从当前 Goroutine 的 `_defer` 链表中弹出（pop）最顶部的延迟函数。
  - 在真正调用取出的延迟函数 `D` 之前，`gopanic` 会把 `D` 的参数**栈帧地址**赋值给 `gp._panic.argp`（颁发合法凭证）。然后，执行 `D`；
  - 当延迟函数 `D` 执行完毕并返回到 `gopanic` 时，`gopanic` 会检查当前 `_panic` 的状态；
    - 如果 `recovered == false`：说明刚才执行的 `D` 没有救活它，继续弹下一个 `_defer`。如果全弹光了还是 false，直接调用 `fatalpanic` 杀掉整个进程。
    - 如果 `recovered == true`：说明刚才执行的 `D` 成功调用了合法的 `recover()`。此时进入**“起死回生”**流程（见第三步）。

- **执行 `recover()`：**`recover()` 的本质是一个**编译器宏**配合**运行时状态**的校验器。其核心逻辑就是验证下面这个等式：**`传入的 argp` == `记录的 p.argp`**
  - **右侧：记录的 `p.argp` (合法凭证 / 指针 X)**
    - **谁记录的？** `gopanic` 在调用 `延迟函数 D` 之前，记录在当前协程状态机（`gp._panic`）中的。
    - **指向哪里？** 指向 `gopanic` 为 `延迟函数 D` 准备参数的那块物理内存地址（即 `gopanic` 和 `D` 栈帧的交界处）。
  - **左侧：传入的 `argp` (实际持卡人 / 指针 Y)**
    - **谁传入的？** Go 编译器。
    - **指向哪里？** 编译器会提取**代码中真正直接调用 `recover()` 的那个函数**的参数栈帧基址，作为参数硬塞给 `gorecover`。

### 正常调用

```go
func 业务函数() {
    defer func() { // 延迟函数 D
        recover()  // D 直接调用
    }()
    panic("boom")
}
```

```Plaintext
|-------------------|
|    gorecover      |  <-- 执行等式比对：传入的 argp == 记录的 p.argp 吗？
|-------------------|
|   延迟函数 D      |  <-- 编译器提取：D 的栈帧基址 (传入的 argp)
|-------------------|  <-- gopanic 记录：D 的栈帧基址 (记录的 p.argp)
|     gopanic       |  
|-------------------|
|    业务函数       |  <-- 挂起状态
|-------------------|
```

**校验结果：成功。** 编译器提取的 `argp`（延迟函数 D 的基址）完美匹配 `gopanic` 记录的 `p.argp`（也是延迟函数 D 的基址）。等式成立，`gorecover` 判定调用合法，清除 panic 状态，程序恢复执行。

### 嵌套调用

```go
func myRecover() {
    recover()      // 被嵌套函数调用
}

func 业务函数() {
    defer func() { // 延迟函数 D
        myRecover()
    }()
    panic("boom")
}
```

底层调用栈帧分布：

```go
|-------------------|
|    gorecover      |  <-- 执行等式比对：传入的 argp == 记录的 p.argp 吗？
|-------------------|
|    myRecover      |  <-- 编译器提取：myRecover 的栈帧基址 (传入的 argp) ❌
|-------------------|  
|   延迟函数 D      |  
|-------------------|  <-- gopanic 记录：D 的栈帧基址 (记录的 p.argp) ✅
|     gopanic       |  
|-------------------|
|    业务函数       |  <-- 挂起状态
|-------------------|
```

## 为什么无法从父 goroutine 恢复子 goroutine 的 panic

> 为什么无法 recover 其他 goroutine 里产生的 panic

因为 goroutine 被设计为一个独立的代码执行单元，拥有自己的执行栈，不与其他 goroutine 共享任何数据。这意味着，无法让 goroutine 拥有返回值、也无法让 goroutine 拥有自身的 ID 编号等。

若需要与其他 goroutine 产生交互，要么可以使用 channel 的方式与其他 goroutine 进行通信，要么通过共享内存同步方式对共享的内存添加读写锁。

有个解决方案【但不是完美方法】：

如果希望有一个全局的恐慌捕获中心，那么可以通过创建一个恐慌通知 channel，并在产生恐慌时，通过 recover 字段将其恢复，并将发生的错误通过 channel 通知给这个全局的恐慌通知器

```go
package main

import (
	"fmt"
	"time"
)

var notifier chan interface{}

func startGlobalPanicCapturing() {
	notifier = make(chan interface{})
	go func() {
		for {
			select {
			case r := <-notifier:
				fmt.Println(r)
			}
		}
	}()
}
func main() {
	startGlobalPanicCapturing()
	// 产生恐慌，但该恐慌会被捕获
	Go(func() {
		a := make([]int, 1)
		println(a[1])
	})
	time.Sleep(time.Second)
}

// Go 是一个恐慌安全的 goroutine
func Go(f func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				notifier <- r
			}
		}()
		f()
	}()
}
```

这里的不完美指的是只能恢复可恢复的 panic，无法恢复不可恢复的 panic【例如并发读写 map】

# 3. 数据容器

## array 和 slice 的区别

**数组：**

* 数组类型包含**数组长度**和**元素类型**，即`[3]int` 和`[4]int` 是两种不同的数组类型；
* 声明时必须指定长度，例如`var arr [5]int`
* 数组是值类型，赋值和传递会复制整个数组。
* 修改副本不会影响原数组。
* 数组是连续内存块，所有元素都在相邻的内存位置上。

**Slice：**

* Slice 是只有3个字段的数据结构：1. 指向底层数组的指针；2. Slice的长度；3. Slice 的容量；
* 切片的长度是动态的，可以在运行时改变。
* 切片是引用类型，赋值和传递只是引用的传递，不会复制底层数组。
* 修改切片会影响到所有引用该切片的变量。

## slice 是如何被截取的

基于已有 slice 创建新 slice 对象，被称为 reslice。

新 slice 和老 slice 共用底层数组，新老 slice 对底层数组的更改都会影响到彼此【前提是必须共用底层数组】。基于数组创建的新 slice 也是同样的效果：对数组或 slice 元素做的更改都会影响到彼此。

> tips: 如果因为执行 append 操作使得新 slice 或老 slice 底层数组扩容，移动到了新的位置，两者就不会相互影响了。

示例：

```go
slice := data[low:high:max]
```

要求：low <= high <= max，当 high == low 时，新 slice 为空。

### 截取越界问题

这段代码为什么没有越界：

```go
x := []int{1}
xa := x[1:]
```

毕竟 `x[1]`会越界。

主要是因为，`x[1:]`是切片截取操作，这个操作仅仅**是对底层数据的一个视图构造，并不直接访问某个具体元素的直接访问**。在生成切片时**只需验证下标范围是否满足 `0 <= i <= j <= len`**。只要这个条件满足，函数仅更新 slice 结构体中的指针、长度和容量，而不会主动去解引用索引为 `j` 的位置。

## slice 的扩容机制

关键代码：

```go
const threshold = 256
if oldCap < threshold {
    return doublecap
}
for {
    // Transition from growing 2x for small slices
    // to growing 1.25x for large slices. This formula
    // gives a smooth-ish transition between the two.
    newcap += (newcap + 3*threshold) >> 2

    // We need to check `newcap >= newLen` and whether `newcap` overflowed.
    // newLen is guaranteed to be larger than zero, hence
    // when newcap overflows then `uint(newcap) > uint(newLen)`.
    // This allows to check for both with the same comparison.
    if uint(newcap) >= uint(newLen) {
        break
    }
}
```

当容量小于`256`【个数】时，需要扩容时，直接二倍扩容。

当容量大于等于`256`时，需要`1.25`倍扩容+固定值`192`，直到满足要求，最后还需要进行内存对齐。

### 扩容前后的 slice 是否相同

1. 原 slice 还有容量可扩容（实际容量未填充完毕）时，扩容后的 slice 还是原来的，对切片的扩容可能影响多个指针指向相同地址的 slice。
2. 原 slice 容量已达到最大值时，Go 默认会先开一片内存将原 slice 拷贝过来，之后进行 append 扩容，不影响原slice。**扩容后的切片**与原切片**不共享底层数组**，因此它们是**不同的存储实体**

## slice 作为函数参数会被改变吗

slice 其实是一个结构体，包含了三个成员：len, cap, array；

当 slice 作为函数参数时，就是一个普通的结构体。从这个角度其实很好理解：若直接传 slice，在调用者看来，实参 slice 并不会被函数中对形参的操作改变，实参是形参的一个复制；若传的是 slice 的指针，则会影响实参。这里的影响指的是 slice 结构体。

## 内建函数 make 和 new 的区别

`new`和`make`是两个用于内存分配的**内键函数**【Build-in】。但它们用于不同的场景，且有不同的行为。

1.  `new` 函数，**分配内存 + 零值化 + 返回指针+使用值类型**
    - 用途：**`new` 函数用于分配内存，但不初始化实例**。

    - 返回值：返回一个指向分配类型的**指针**。

    - 适用类型：适用于**值类型**，如基本数据类型（int、float32 等）、结构体（struct）等。

    - 初始化：分配的内存会被初始化为零值。
2.  `make`函数：**分配内存 + 结构初始化 + 返回实例值+仅限 S、M、C**
    - 用途：**`make` 函数用于创建并初始化引用类型**。
    - 返回值：返回引用类型的**初始值**（非指针）。
    - 适用类型：适用于切片（slice）、映射（map）和通道（channel）。
    - 初始化：不仅分配内存，还执行必要的初始化操作。

## 空切片 和 nil 切片是什么，有什么区别

nil 切片和空切片并不相同，但它们的表现行为几乎是相同的。

* nil 切片，通过 `var s []T`**声明但未初始化的切片**。他的值为 `nil`，**没有底层数组，长度，容量**；例如：

  ```go
  var s []int  // Nil 切片，值为 nil
  fmt.Println(s == nil) // 输出: true
  fmt.Println(len(s))   // 输出: 0
  fmt.Println(cap(s))   // 输出: 0
  ```

* 空 切片，通过字面量`[]T{}`或`make([]T,0)`常见的切片，**分配了空数组**。例如：

  ```go
  s := []int{} // 空切片，长度为 0，容量为 0
  fmt.Println(len(s)) // 输出: 0
  fmt.Println(cap(s)) // 输出: 0
  ```

## 如何判断两个字符串切片（slice）是否相等

在 Go 中要判断两个切片是否相等，考虑以下两点（不考虑容量）：

1. **长度相同**：首先，两个切片的长度必须相同。
2. **元素相同**：如果长度相同，那么我们逐个比较切片中的元素是否相等。

主要有以下两个方式实现对比：

1. 自定义循环。

2. 通过 `reflect.DeepEqual` ：

   ```go
   package main
   
   import (
       "fmt"
       "reflect"
   )
   
   func main() {
       slice1 := []string{"apple", "banana", "cherry"}
       slice2 := []string{"apple", "banana", "cherry"}
       slice3 := []string{"apple", "banana"}
   
       fmt.Println(reflect.DeepEqual(slice1, slice2)) // true
       fmt.Println(reflect.DeepEqual(slice1, slice3)) // false
   }
   ```

## slice 的 for 循环中 append 元素会发生什么

在 **Go 语言** 中，`for range` 循环遍历切片时，切片长度在开始遍历时就已经被固定下来，即使循环中使用 `append` 动态修改切片的长度，也不会影响 `range` 的遍历次数。

## map 的底层实现原理是什么

> Go 语言采用哈希查找表，并且使用寻址法+链表法解决哈希冲突
>
> 构建 hmap；读 map；删除 map；扩容 map；

### map 如何扩容

装载因子：`count/2^B`

触发条件：

1. 装填因子是否大于 6.5【key-value 总数 / 桶数组长度】key-value 总数包括溢出桶，而桶数组长度不包括溢出桶
2. overflow bucket 是否太多，即溢出桶是否大于等于`2^B`【B=min{15,B}】

解决方法：

1. 双倍扩容：扩容采取了一种称为“渐进式”地方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。
2. 等量扩容：重新排列，极端情况下，重新排列也解决不了，map 成了链表， 性能大大降低，此时哈希种子 hash0 的设置，可以降低此类极端场景的发 生。

## map 中的 key 为什么是无序的

在 Go 的实现中，当遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，而是每次都从一个随机序号的 bucket 开始，并且从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对。

### 如何实现 map 的顺序遍历

可以先将 map 中的所有 key 提取到一个切片中，然后对切片进行排序，再按照排序后的 key 顺序遍历 map。

## map 是线程安全的吗

在查找、赋值、遍历、删除的过程中都会检测写标志，一旦发现写标志置位（等于 1），则直接 panic。赋值和删除函数在检测完写标志是复位状态（等于 0）之后，先将写标志位置位（置为 1），才会进行之后的操作。

## map 的 key 要求

在 Go 语言中，**可以作为 `map` 键（key）的类型必须满足可比较（comparable）的条件**。具体来说，Go 要求键的类型必须支持 `==` 和 `!=` 操作符的相等性比较，且这种比较在逻辑上是确定的。

不允许作为 Key 的类型：

1. Slice；
2. Map；
3. Function；
4. 包含不可比较字段的结构体或数组。

注意事项：

1. `interface{}`类型的键，要确保其动态的值可比较；

2. 结构体和数组的深度比较，结构体和数组作为键时，会递归比较所有字段或元素的值

3. float 的精度问题；当用 float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中。具体通过 Float64frombits 函数完成：

4. 使用 `math.NAN()`时，由于 `NAN`与任何值【包括自身】的比较结果均为`false`。`NAN()` 作为键时，直接调用 Float64frombits，传入写死的 const 型变量 0x7FF8000000000001，得到 NAN 型值。对于 64 位浮点数，他的哈希函数如下：

   ```go
   // src/runtime/alg.go
   func f64hash(p unsafe.Pointer, h uintptr) uintptr {
   	f := *(*float64)(p)
   	switch {
   	case f == 0:
   		return c1 * (c0 ^ h) // +0, -0
   	case f != f: 
   		return c1 * (c0 ^ h ^ uintptr(fastrand()))
   	default:
   		return memhash(p, h, 8)
   	}
   }
   ```

   第 2 个 case，`f != f` 就是针对 NAN，这里会再加一个随机数。

## map 如何实现两种 get 操作

Go 语言中读取 map 有两种语法：带 comma 和不带 comma。

当要查询的 key 不在 map 里，带 comma 的用法会返回一个 bool 型变量提示 key 是否在 map 中；而不带 comma 的语句则只会返回一个 key 类型的零值。如果 key 是 int 型就会返回 0，如果 key 是 string 类型，则会返回空字符串。

但为什么 Go 语言不支持重载，为什么同样的赋值语句会有两种不同类型的返回值呢？这其实是编译器在背后做的工作：编译器在分析完用户代码后，会将这两种语法分别对应到 runtime 两个不同的函数。

```go
// src/runtime/map.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) 
```

## 渐进式扩容

由于 Map 采用渐进式扩容操作，渐进式扩容时，如果发生读取、写入、删除会怎样？

### 读取操作

即使当前 Map 正在扩容，有了新桶数组，读取操作第一步依然是计算这个 Key 如果在**旧桶数组**里，应该落在哪一个桶。

找到对应的旧桶后，Go 会检查这个旧桶的标志位（底层的 `tophash` 数组中记录了状态）。它只关心一件事：**这个旧桶里的数据，有没有被搬走？**

**情况 A：旧桶还没被搬迁** 说明该 Key 如果存在，肯定还在旧桶里。Go 会直接在这个**旧桶及其溢出桶**中寻找。

- 找到了，直接返回。
- 找不到，就说明整个 Map 里都没有这个 Key，**直接返回零值，绝不会再去新桶找**（因为数据根本还没过去）。

**情况 B：旧桶已经被搬迁（Evacuated）** 说明这个旧桶里的数据之前已经被某次“写/删”操作顺手搬到新桶去了。此时，旧桶已经是空的废弃状态。Go 这时才会重新计算该 Key 在**新桶数组**中的新索引，然后去对应的**新桶**里寻找。

## 写入、删除

当发生写入或删除时，Go 的“渐进式搬迁”策略是：**每次操作最多只搬迁 2 个旧桶**。

1. **目标桶：** 首先把当前写入/删除操作所命中的那个旧桶，立刻搬迁到新桶中。
2. **顺序桶（顺手牵羊）：** Go 底层维护了一个搬迁进度游标（`evacuate` 指针），它会从旧数组索引 0 开始。每次有写/删操作，除了搬迁目标桶，还会顺手把游标指向的那个旧桶也搬过去，然后游标前移。

## 如何比较两个map是否相等

两个 map 深度相等的条件如下：【或的关系，满足其一即可】

1. 都为 nil；
2. 非空、长度相等，指向同一个 map 实体对象；
3. 相应的 key 指向的 value “深度”相等；

使用`==`比较 map 是错误的，这种写法只能比较 map 是否为 nil。

## 使用值为 nil 的 slice 和 map 会发生什么

在 Go 语言中，slice 和 map 都可以被初始化为 `nil`，这是合法的。对于 `nil` slice 和 `nil` map，可以执行一些特定的操作，但有些操作会导致运行时错误。具体来说：

对于 `nil` slice：

* `len` 和 `cap` 都返回 0
* 可以进行迭代操作
* 不能直接通过索引赋值或取值，这会导致运行时错误
* 可以使用 `append` 函数来为其**追加元素**

对于`nil` map：

* `len` 返回 0；
* 可以进行**读取**操作，返回的是该类型的零值；
* 不能进行插入或更新，这会导致运行时错误

## 如何实现 set

空结构体来实现set，例如 `map[T]struct{}`

### struct{} 的妙用

1. 在 `map` 或 `slice` 中作为占位符，可用来实现 set ；
2. **无数据信号量**的传递；

   > **即 struct{} 不占空间。**
3. 方法接收器，例如`type Logger struct`；

### Go 语言中 cap 函数作用于哪些内容

可作用于：

* `array`：获取 `array` 的容量；
* `slice`：获取 `slice` 的容量；
* `channel`：获取 `channel` 缓冲区大小。

不可用于：`map`，其动态哈希表实现不暴露底层存储的“容量”信息

# 4. channel

## channel 注意事项

### 1. `nil Channel`的使用：

- 发送数据到一个`nil`的`Channel` 会导致永远阻塞

  ```go
  var ch chan int
  ch <- 42 // 永久阻塞
  ```

- 从一个`nil`的`Channel`接受数据也会导致永久阻塞：

  ```go
  var ch chan int
  <- ch // 永久阻塞
  ```

### 2. 向已关闭的 `Channel` 发送数据：

- 向已关闭的 `Channel` 发送数据会引发 `panic`：

  ```go
  ch := make(chan int)
  close(ch)
  ch <- 42 // panic
  ```

### 3. 从已关闭的 Channel 接收数据：

- 从已关闭的 Channel 接收数据，**如果缓冲区中为空，则返回该类型的零值**：

  ```go
  ch := make(chan int, 1)
  ch <- 42
  close(ch)
  fmt.Println(<-ch) // 输出 42
  fmt.Println(<-ch) // Channel 关闭，输出零值 0
  ```

### 4. `Channel`缓冲：

无缓冲的Channel是同步的，有缓冲的Channel是非同步的。

### 5. 避免死锁：

- 不正确使用 `Channel` 容易导致死锁。例如，在无缓冲 `Channel` 上发送数据而没有相应的接收者会导致死锁。同样，如果所有发送者和接收者都阻塞，也会导致死锁。

### 6. 关闭 Channel 的时机：

- 应该只有发送方关闭 `Channel`，而接收方不应该关闭 `Channel`。关闭已经关闭的 `Channel` 会导致 `panic`：

  ```go
  ch := make(chan int)
  close(ch)
  close(ch) // panic
  ```

## channel 引起资源泄漏

**channel 可能会引发 goroutine 泄漏。**

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中。

## 无缓冲chan和1缓存chan

**无缓冲 Channel (`make(chan T)`) —— “当面交接” (一手交钱，一手交货)** 它没有任何存储空间。发送者（Sender）和接收者（Receiver）必须**在同一时刻**准备好，才能完成数据的传递。如果发送者先到，他必须原地死等（阻塞），直到接收者出现；反之亦然。

**容量为 1 的缓冲 Channel (`make(chan T, 1)`) —— “单格快递柜”** 它拥有一个容量为 1 的存储空间。发送者不需要等接收者出现，只要快递柜是空的，他就可以把东西塞进去，然后**立刻转身离开（不阻塞）**。只有当快递柜已经满（里面已经有 1 个元素）时，发送者才需要等待。

# 5. Interface

## 值接收者与指针接收者的区别

方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，它就变成了方法。接收者可以是值接收者，也可以是指针接收者。

在**调用方法的时候**，值类型既可以调用值接收者的方法，也可以调用指针接收者的方法；指针类型既可以调用指针接收者的方法，也可以调用值接收者的方法。

在**实现方法的时候**，实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。

### 两者分别在何时使用？

主要看类型是否具备 “原始的本质”，如果是 Go 语言里内置的原始类型，如字符串、整型值等构成，那就定义**值接收者**类型的方法。像内置的引用类型，如 slice、map、interface、channel，这些类型比较特殊，声明它们的时候，实际上是创建了一个 header，对于它们也是直接定义**值接收者类型**的方法。

如果类型具备非原始的本质，不能被安全地复制，这种类型总是应该被共享，那就定义指针接收者的方法。比如 Go 源码里的文件结构体（struct File）就不应该被复制，应该只有一份实体，就要定义指针接收者类型的方法。

## go 是面向对象的吗？

Go官网的回答中提到，Yes or No。也就是说Go不是面向对象语言，但也可以进行面向对象风格的编程，

* 封装：Go语言里面字段首字母大小写来决定字段是否可以被外包访问；

  > 封装：在一个对象内部，某些方法和数据可以是私有的，不能被外界访问，封装为对象内部数据提供了不同级别的保护。

* 继承：Go语言里面用组合结构体的方式 或 接口继承来实现

  > 继承：让某个类型的对象获得另一个类型的对象的属性
  >
  > 继承概念的实现方式有二类：实现继承与接口继承。
  >
  > * 实现继承就是直接使用父类的属性和方法而无需额外编码的能力
  > * 接口继承是指仅使用属性和方法的名称、但是子类必须提供实现的能力。

* Go语言中通过接口来实现多态，不同的类型实现对应接口，然后调用接口变量的方法，结果取决于接口存储的对应类型的方法。

  > 多态：指一个类实例的相同方法能有不同表现形式。多态机制可以让不同内部结构的对象共享相同的外部接口。这意味着，虽然针对不同对象的具体操作不同，但通过一个公共的类，它们（那些操作）可以通过相同的方式予以调用。

### 如何使用 interface 实现多态

多态是一种运行期的行为，它有以下几个特点：

1. 一种类型具有多种类型的能力；
2. 允许不同的对象对同一消息做出灵活的反应；
3. 以一种通用的方式对待个使用的对象；
4. 非动态语言必须通过继承和接口的方式来实现。

例子：

```go
package main

import "fmt"

type Person interface {
	job()
	growUp()
}

func whatJob(p Person) {
	p.job()
}

func growUp(p Person) {
	p.growUp()
}

type Student struct {
	age int
}

func (s Student) job() {
	fmt.Println("I am a student.")
	return
}

func (s *Student) growUp() {
	s.age += 1
	return
}

type Programmer struct {
	age int
}

func (s Programmer) job() {
	fmt.Println("I am a programmer.")
	return
}

func (s *Programmer) growUp() {
	s.age += 10
	return
}

func main() {
	qcrao := Student{age: 18}
	// 注意只有指针接收器实现了 Person interface.
	whatJob(&qcrao)
	growUp(&qcrao)
	fmt.Println(qcrao)

	stefno := Programmer{age: 35}
	whatJob(&stefno)
	growUp(&stefno)
	fmt.Println(stefno)
}
```

代码里先定义了 1 个 Person 接口，包含两个函数：

```go
job()
growUp()
```

然后，又定义了 2 个结构体，Student 和 Programmer，同时，类型 `*Studen`t、Programmer 实 现了 Person 接口定义的两个函数。注意，`*Student` 类型实现了接口，Student 类型却没有。

之后，又定义了函数参数是 Person 接口的两个函数：

```go
func whatJob(p Person)
func growUp(p Person) 
```

在 main 函数里先生成 Student 和 Programmer 的对象，再将它们分别传入到函数 whatJob 和 growUp 函数中，直接调用接口拥有的函数，实际执行的时候是看最终传入的实体类型是什么，调用的是实体类型实现的函数。于是，不同对象针对同一消息就有多种表现形态，多态就实现了。

## Go 语言使用断言时会发生拷贝吗

在 Go 语言中，**类型断言是否发生拷贝**取决于接口内部持有的数据类型：

* **值类型**：当接口持有的是值类型（例如 `int`、`float`、`struct` 等），进行类型断言时会**发生拷贝**，因为接口存储的是这个值的副本，断言后得到的是该值的拷贝。
* **引用类型**：当接口持有的是引用类型（例如指针、切片、映射、通道等），进行类型断言时不会发生拷贝，因为接口存储的是一个引用，断言得到的也是相同的引用。

因此，如果接口中存储的是一个结构体实例，通过断言得到的是结构体的值拷贝，修改断言后的变量不会影响接口中的值；而如果接口中存储的是指针，通过断言得到的依然是指针引用，修改断言后的指针值会影响接口内的原数据。

## 让编译器自动检测类型是否实现了接口

```go
var _ io.Writer = (*myWriter)(nil) 
```

实际上，上述赋值语句会发生隐式地类型转换，在转换的过程中，编译器会检测等号右边的类 型是否实现了等号左边接口所定义的函数。

## 两个接口之间可以存在的关系

如果两个接口有相同的方法列表，那么它们是等价的，可以相互赋值。并且可以相互替换使用。这意味着一个实现了这些方法的类型可以满足这两个接口中的任何一个，甚至是两个都满足。

## 接口比较

接口(interface) 是对非接口值(例如指针，struct等)的封装，内部实现包含 2 个字段，类型 T 和 值 V。一个接口等于 nil，当且仅当 T 和 V 处于 unset 状态（T=nil，V is unset）。

两个接口值比较时，会先比较 T，再比较 V。接口值与非接口值比较时，会先将非接口值尝试转换为接口值，再比较。

```go
func main() {
     var p *int = nil
     var i interface{} = p
     fmt.Println(i == p) // true
     fmt.Println(p == nil) // true
     fmt.Println(i == nil) // false
 }
```

* 例子中，将一个nil非接口值p赋值给接口i，此时,i的内部字段为(`T=*int, V=nil`)，i 与 p 作比较时，将 p 转换为接口后再比较，因此 i == p，p 与 nil 比较，直接比较值，所以 p == nil。

* 但是当 i 与 nil 比较时，因为 i 为接口指，会将 nil 转换为接口(`T=nil, V=nil`)，与 i (`T=*int, V=nil`)不相等，因此 `i != nil`。因此 V 为 nil ，但 T 不为 nil 的接口不等于 nil。

# 6. 其他

## Go 语言的 switch 中如何强制执行下一个 case 代码块

可以使用`fallthrough`关键字。需要注意的是，`fallthrough` 只能在 switch 语句的 case 中使用。【`fallthrough`只能跨越一个 case 代码块，不能连续跨多个 case。】

并且其作用是强制程序流继续进入下一个紧接着的 case 代码块，**而不考虑下一个 case 条件是否符合**。

## Go 语言中打印字符串时，%v 和 %+v 有什么区别

在 Go 语言中，格式化打印字符串时，%v 和 %+ v 的主要区别在于打印结构体（struct）的效果不同。具体如下：

1. `%v`：适用于打印变量的值。当打印结构体时，它**只会显示字段的值，而不会显示字段名**；
2. `%+v`：适用于详细打印结构体的内容。与 %v 不同的是，当打印结构体时，它会**显示字段的名字和值**；
3. `%#v`：输出**结构体名称和结构体各成员的名称和值**。

**但是要注意以下几点：**

1. 对于指针类型的数据，使用 `%+v`  不会自动解指针，因此会直接打印地址类型，要注意。

2. 对于性能要求不高的场景，推荐使用`json.Marshal`函数序列化之后在输出，可以针对所有情况的打印，包括但不限于结构体，指针等。

   ```go
   package main
   
   import (
   	"encoding/json"
   	"fmt"
   )
   
   type Person struct {
   	Name string `json:"name"`
   	Age  int    `json:"age"`
   }
   
   func main() {
   	p := Person{
   		Name: "凉柠",
   		Age:  22,
   	}
   
   	// 使用 json.Marshal 将结构体编码为 JSON 格式的字节数组.
   	jsonBytes, err := json.Marshal(p)
   	if err != nil {
   		fmt.Println("编码错误: ", err)
   		return
   	}
   	fmt.Println("结构化输出：")
   	// 将字节数组转换为字符串并打印.
   	fmt.Println(string(jsonBytes))
   	fmt.Println("---------------------------------")
   	// 如果希望输出格式化后的【进行缩进处理的】JSON
   	// 也可以使用 json.MarshalIndent.
   	jsonBytesIndent, err := json.MarshalIndent(p, "", "\t")
   	if err != nil {
   		fmt.Println("编码错误: ", err)
   		return
   	}
   	fmt.Println("缩进的结构化输出：")
   	fmt.Println(string(jsonBytesIndent))
   }
   ```

## Go语言中什么情况使用指针

**如下场景一般会使用指针：**

1. **结构体很大或者变量很大**，为了提高性能，会选择使用指针，传变量的地址；

2. **修改结构体中的字段值**，需要使用指针；

3. **通过判断某个字段是否设置了值**，做一些业务逻辑处理，选择使用指针。通过 if xx == nil 来进行判断；

   理解：当需要区分**字段是否被显式赋值**（而非默认零值）时，使用指针类型。通过检查指针是否为`nil`，可以明确判断字段是**未设置**还是**被设置为零值**，从而处理不同的业务逻辑。

   * `JSON`反序列化中的可选字段：

     ```go
     type UserRequest struct {
         Name string  `json:"name"`
         Age  *int    `json:"age"` // 使用指针区分"未传字段"和"传了0"
     }
     
     // 场景1：{"name": "Alice"} → Age为nil（未设置）
     // 场景2：{"name": "Bob", "age": 0} → Age指向0（显式设置）
     ```

     * 若`Age`是普通`int`类型，无法区分用户是否传递了该字段。
     * 指针允许通过`if req.Age != nil`判断字段是否存在。

   * 数据库可空字段映射：

     ```go
     type User struct {
         ID   int
         Bio  *string // 对应数据库允许为NULL的列
     }
     ```

     * 数据库中的`NULL`映射为`Bio`指针的`nil`。
     * 普通`string`类型无法区分`NULL`和空字符串`""`。

   * 部分更新逻辑

     ```go
     func UpdateUser(id int, data *UserUpdateData) {
         if data.Age != nil {
             // 仅当Age字段被显式设置时，更新数据库
             db.Exec("UPDATE users SET age = ? WHERE id = ?", *data.Age, id)
         }
     }
     ```

4. **API接口**的请求参数，想**保持兼容性**；

   当 API 升级时新增可选字段，旧版客户端可能不会传递这些字段。使用指针可以避免旧请求因新字段的默认零值引发逻辑错误。

   ```go
   // v1 版本结构体
   type RequestV1 struct {
       ID int `json:"id"`
   }
   
   // v2 新增可选字段（指针）
   type RequestV2 struct {
       ID     int  `json:"id"`
       Enable *bool `json:"enable"` // 旧客户端不会传此字段
   }
   
   // 旧客户端请求 {"id": 1} 反序列化到 RequestV2 时：
   // Enable 为 nil，不会干扰服务端逻辑
   ```

   这样，服务端可安全处理新旧客户端请求，无需强制升级客户端。

5. **实现某些数据结构时**，例如链表。因为需要在节点之间存储关系；

6. 变量类型即**规范**：通过使用指针，告诉 Go 开发者，这个变量可能会被改变；

**下面情况通常不使用指针：**

1. 不想某个结构体，或者变量的内容被改变；
2. 没有需要使用指针的场景；
3. 想直接使用零值，而非指针。使用零值更加安全。如果使用指针，就需要先判断指针是否为nil，否则直接使用会导致程序panic；

## GO 语言 init() 函数的执行流程

![img3](http://images.liangning7.cn/typora/202504062249239.jpeg)

## 未分配内存的指针

在 Go 语言中，不分配内存的指针类型可以使用，但是只能用该指针本身，不可以用`*`去解引用出具体的值，会导致 panic 。这是因为 Go 允许声明指针变量，但如果不分配内存（没有指向有效的地址），该指针会是 nil。访问 nil 指针会导致运行时错误。

## struct 的比较

1. 对于不同类型的 `struct` 无法进行比较；

   ```go
   type User struct {
   	ID int
   }
   
   type Student struct {
   	ID int
   }
   ```

2. 而同一个 `struct` 的两个实例可比较也不可比较。

   在 Go 中，`Slice`、`map`、`func` 无法比较，当一个 struct 的成员是这三种类型中的任意一个，就无法进行比较。反之，struct是可以进行比较的，**要注意的是**，不能直接使用 `<` 或 `>` 之类的操作符来比较 struct，只能用等于和不等于。

## 不可比较类型的比较

1. 像 string、int、float、interface 等可以通过 `reflect.DeepEqual` 和 `==` 进行比较

2. 像 slice，struct，map 则一般使用 `reflect.DeepEqual` 来检测是否相等。
