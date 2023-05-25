---
title: 深入Go：并发迷思-消失的赋值语句
tags: Go MemoryModel
key: go-gone-go-statement
---

对全局变量的赋值，为何无缘无故消失？等候了千万个时钟周期的打印语句，为何发现变量没有一丝改变？意料之外的结果，却为何又是在情理之中？这究竟是编译器的背叛，还是随机的巧合——本篇文章将带您深入Go内存模型，一起走近并发。
<!--more-->

### 热身

先看一个[经典的问题](http://preshing.com/20120515/memory-reordering-caught-in-the-act/ )，下列代码输出的结果可能是多少？

```go
wg := sync.WaitGroup{}
x, y, r1, r2 := 0, 0, 0, 0
wg.Add(2)
go func() { // goroutine A
  y = 1 // line A1
  r1 = x // line A2
  wg.Done()
}()
go func() { // goroutine B
  x = 1 // line B1
  r2 = y // line B2
  wg.Done()
}
fmt.Println(r1 + r2)
```

输出当然可能是1，执行顺序可能是：

```
A1: y = 1                    | B1: x = 1
A2: r1 = x // 0        | B2: r2 = y // 0
B1: x = 1                    | A1: y = 1
B2: r2 = y // 1        | A2: r1 = x // 1
------------- 1     | ------------- 1
```

输出也可能是2，因为执行顺序可能是：

```
A1: y = 1
B1: x = 1
-- then --
A2: r1 = x // 1
B2: r2 = y // 1
------------ 2 --
```

但是运行10000000次，有9994463次结果为1，有23次结果为2，有5514次结果为0！

为什么结果为0，也就是执行顺序可能变成了：

```
A2: r1 = x
B2: r2 = y
A1: y = 1
B1: x = 1
```

实际上，CPU的指令执行顺序是乱序执行的，因为但就一个协程执行的代码而言，两行语句是无关的，CPU完全可能会乱序执行；指令乱序执行也是现代CPU能运行如此之快的原因之一——否则，如果一个store指令需要等待写入，后面的load指令只能白白等待。

（也许不仅仅因为CPU的指令乱序导致迷思，后面我们可以看到。）

### 意料之外的迷思

再看另一段代码，请问输出应该是什么？

```go

var isRunning = int32(1)

func fg1() {
    for {
        isRunning--
    }
}

func fg2() int {
    count := 1
    for isRunning > 0 {
        count++
    }
    return count
}

func main() {
    count := 0
    go fg1()
    go func() {
        count = fg2()
    }()
    time.Sleep(3 * time.Second)
  println(isRunning)
    println(count)
}
```

答案是：`isRunning: 1, count: 0`，也就是`fg1`中的`isRunning--`没有被执行，`fg2`根本没有返回。

这就很让人意外，足足等了3秒的时间，而`fg1`里的循环完全没有产生任何的效果。实际上，查看go汇编代码（`go tool compile -S file.go > file.s`），可以发现如下结果：

```
"".fg1 STEXT nosplit size=3 args=0x0 locals=0x0
  ...
  // swap the value of $(AX) and $(AX) atomically, or NOP -- do nothing
    0x0000 00000 (pkg/main/file.go:10)    XCHGL    AX, AX 
    // jump to last line
    0x0001 00001 (pkg/main/file.go:1)    JMP    0        
```

即，`fg1`什么都没有做，不说等3秒了，等10年也没用！那，是不是Go的编译器背叛了这段代码？

### 情理之中的解答

最后再问一个问题，在Go当中，对一个变量的write在什么情况下才能保证被对该变量的read所感知到？虽然你可能有Go的编程经验，但很可能你也说不清楚这个问题。它实际上有[官方的解答](https://golang.org/ref/mem)。

我们一边举例子，一边来解释。

#### 早于、晚于、并发于

首先我们要定义偏序关系（回想大学知识，偏序关系是非自反、反对称、传递的关系）“早于”（*Happens Before*）。为方便起见，我们记“A早于B”为$A<B$；我们也定义“晚于”关系（*Happens After*），“B晚于A”记为$B>A$——如果$A<B$，则有$B > A$。如果$A < B$ 、$A > B$都不成立，则称“A并发于B”。

首先，单个goroutine中顺序执行的语句，在先的与在后的形成“早于”关系，例如下方代码中，$A_1 < A_2$：

```go
func f1() {
  y = 1 // A_1
    r1 = x // A_2
}
```

其次，包的`init`、goroutine的创建、channel交互、锁、once也定义了偏序关系；这里，我们选相关的goroutine创建、销毁与锁的使用进行介绍。

##### Goroutine的创建

创建goroutine的代码一定早于该goroutine中代码的执行。例如下面的代码中，由上面的规则有$A < B$，由本条规则有$B < C$，由关系的传递性，有$A < C$；如果没有其他协程对`x, y`的write的话，一定有`y = 1`。

```go
x = 1 // A
go func() { // B
  y = x // C
}()
```

##### 锁

对于任意`sync.Mutex`变量`l`和$n < m \in \mathbb N$，第$n$次`l.Unlock()`的调用早于第$m$次调用`l.Lock()`返回，例如以下代码中，我们根据本规则有：$V < C$。

```go
var l sync.Mutex
var a string
func f() {
  a = "hello world" // U
  l.Unlock() // V
}

func main() {
  l.Lock() // A
  go f() // B
  l.Lock() // C
  print(a) //D
}
```

因此，我们有：$A < B < U < V$，$V < C < D$，故：$A < B < U < V < C < D$。也就是说，如果没有其他线程修改`a`，我们一定可以打印出`hello world`。

`sync.RWMutex`有类似情况，不再赘述。

#### 保证write能被read观察到的条件

回到第三个问题，对变量`x`的write $w$如何才能保证被read $r$观察到，[Go内存模型](https://golang.org/ref/mem)规定了：

1. $w < r$；且
2. 其他对于`x`的写要么早于$w$，要么晚于$r$。

##### 注意

Go内存模型也说明了：

一，Goroutine代码的执行、销毁时间没有任何保证，甚至下方的代码行$A$可以被编译器直接删除：

```go
var a string
func hello() {
  go func() { a = "hello" }() // A
  print(a)
}
```

二，如果read $r$观察到了并发于$r$的write $w$，也不能保证任何晚于$r$的read能观察到任何早于$w$的write——这是因为$r$观察到$w$不能推出$r$晚于$w$。

举个例子，下方代码中，我们只能得到$C < D < E$和$C < A < B$，并不能得出$A < E$；因此即使偶然有循环$D$退出，也不能保证能打印出`hello`。

```go
var a string
var done bool

func setup() {
    a = "hello" // A
    done = true // B
}

func main() {
    go setup() // C
    for !done { // D
    }
    print(a) // E
}
```

#### 解答

```go

var isRunning = int32(1)

func fg1() {
    for {
        isRunning-- // A
    }
}

func fg2() int {
    count := 1
    for isRunning > 0 { // B
        count++
    }
    return count
}

func main() {
    count := 0
    go fg1() // C
    go func() {
        count = fg2() // D
    }()
    time.Sleep(3 * time.Second)
  println(isRunning) // E
    println(count)
}
```

我们可以得出：

* $C < A$
* $D < B$
* $C < E$

但我们不能得到$A$和$B$、$A$和$E$之间的早于关系。因此，编译器完全可以优化掉`fg1`中的赋值语句。详细讨论还可以见[码客](http://mk.woa.com/q/269855)与[Google Groups - golang-nuts](https://groups.google.com/g/golang-nuts/c/eT9vsOORTsA)（至于为什么编译器在`short circuit`阶段优化掉该赋值，尚在讨论之中，后续会继续更新）。

### 讨论

再来看一段代码：

```go
var hasLoad = uint32(0)
var instance *T
var m sync.Mutex

func getInstance() *T {
  if hasLoad == 0 { // A
    m.Lock()
    if hasLoad == 0 {
      instance = &T{}
      hasLoad = 1
    }
    m.Unlock()
  }
}
```

这段代码究竟有没有问题？

运行`go run -race`来查看data race的情况，马上会得到在$A$处会有一个协程写一个协程读的情况，我们之前的做法都是，把`hasLoad`的读写都使用`sync/atomic`包进行操作。但真的需要吗？

#### Go Memory Model: Advice

> Programs that modify data being simultaneously accessed by multiple goroutines must serialize such access.
>
> To serialize access, protect the data with channel operations or other synchronization primitives such as those in the [`sync`](https://golang.org/pkg/sync/) and [`sync/atomic`](https://golang.org/pkg/sync/atomic/) packages.
>
> If you must read the rest of this document to understand the behavior of your program, you are being too clever.
>
> Don't be clever.

> 被多个goroutines并发读写数据的程序必须串行化这样的读写。
>
> 为此，请使用channel操作或其他例如`sync`、`sync/atomic`中的同步原语保护该类数据。
>
> 如果你一定要阅读本文（笔者注：即Go Memory Model）剩余部分以理解你程序的行为，那么你就是耍小聪明了。
>
> 别耍小聪明。

遇到并发读写变量的情况，请一定使用`mutex`或atomic操作；我们可以认为这段代码是存在问题的。

#### 可以跳过的小聪明分析

其实这段代码严格意义上是没有问题的，我们再来分析（为方便起见，我们假设只有2个协程访问`getInstance`，所以我们可以把它分别命名为`getInstance1`与`getInstance2`）：

```go
var hasLoad = uint32(0)
var instance *T
var m sync.Mutex

func getInstance1() *T { // 和getInstance完全相同
  if hasLoad == 0 { // A1
    m.Lock() // B1
    if hasLoad == 0 { // C1
      instance = &T{} // D1
      hasLoad = 1 // E1
    }
    m.Unlock() // F1
  }
}

func getInstance2() *T { // 和getInstance完全相同
  if hasLoad == 0 { // A2
    m.Lock() // B2
    if hasLoad == 0 { // C2
      instance = &T{} // D2
      hasLoad = 1 // E2
    }
    m.Unlock() // F2
  }
}

func main() {
  go getInstance1()
  go getInstance2()
}
```

我们假设$B_2 > F_1$，也就是说`getInstance1`先调用`m.Lock()`并最终`m.Unlock()`，然后才是`getInstance2`的`m.Lock()`返回（对称地可以分析另一情况，因此不再赘述）。

在`getInstance1`中，因为是串行执行，有$D_1 < E_1 < F_1$。

在`getInstance2`中，也是因为串行执行，有$B_2 < C_2$。

所以由假设$B_2 > F_1$，有$D_1 < E_1 < F_1 < B_2 < C_2$，由于对于`hasLoad`没有其他的写，根据保证，可以知道$E_1$中的`hasLoad = 1`可以被$C_2$中的`if hasLoad == 0`观察到，因此不会进入$D_2$、$E_2$，不会导致`instance`被多次赋值——代码是正确的。而$A_1$、$A_2$和$E_1$、$E_2$的data race其实无关紧要，因为有$C_1$、$C_2$处的双重校验，第一次`m.Unlock()`之前的`hasLoad = 1`被观察到了。

实际上可以运行以上代码，每次用多个协程调用`getInstance`，重复1000000次，没有一次有发生`instance`重复赋值。

### 结论

* 并发中意料之外的结果，总是有着情理之中的解释；
* 虽然仔细地分析可以得到结论，但还是请合理使用`mutex`与`atomic`操作。
