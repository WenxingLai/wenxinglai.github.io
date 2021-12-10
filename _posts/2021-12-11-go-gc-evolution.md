---
title: 深入Go：垃圾回收的演进
tags: Go
---

Stop the world 是讨论垃圾回收（Garbage Collection，GC）时绕不开的话题，曾经Go语言的GC机制也威胁着服务的响应时间——Discord技术团队的文章*Why Discord is switching from Go to Rust*讨论了Go语言GC带来的问题。Go通过版本迭代已经极大地改善了GC的问题，平均每次STW时间从100+ms降低到了0.5ms——是什么神奇的魔法使得世界几乎无需暂停？在本文中，我们通过提问、解答的方式尝试对该演进的主要过程进行梳理。（本文首发于[腾讯云+社区](https://cloud.tencent.com/developer/article/1916989)。）
<!--more-->

#### 垃圾回收有哪些机制，各有什么利弊？

垃圾回收主要有「引用计数」与「追踪」的机制：

* 引用计数（Reference Counting）：给每一个对象增加一个计数器，每当该对象被引用/解除引用时，则计数器自增/自减1；当该计数器归零时，则该对象应该被回收。
* 追踪（Tracing）：定期遍历内存空间，从若干根储存对象开始查找与之相关的存储对象，然后标记其余的没有关联的存储对象，最后回收这些没有关联的存储对象占用的内存空间。

这两种方法各有好处和坏处：

|                  | 引用计数                               | 追踪 |
| ---------------- | -------------------------------------- | ---- |
| 需要暂停整个程序 | 否                                     | 是   |
| 运行时的额外开销 | 计数器的并发控制<br />计数器的内存空间 | 无   |
| 无法回收的情况   | 对象循环引用*                          | 无   |

\*注：类似以下代码，`person`与`apartment`的对象间循环引用，造成最终无法被基于循环引用的垃圾回收器回收。

```python
person = Person()
apartment = Apartment()
person.apartment = apartment
apartment.owner = person
person = None
apartment = None
```

其中，追踪的方法被`Java`的多个垃圾回收算法、`Go`语言等使用；引用计数的方法被`swift`、`Objective-C`、`Python`等使用，`C++`的智能指针也可以被认为是引用计数的实现——其中`Python`提供循环引用检测，而`swift`、`Objective-C`则使用ARC（[Automatic Reference Counting](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)），需要代码编写者对引用类型进行判断，区分强引用与弱引用，例如

```swift
class Apartment {
    weak var owner: Person?
}
```

#### Go使用追踪式的垃圾回收机制具体是怎样的？

Go 1.5之后，使用三色标记法进行垃圾回收。

传统的标记-清扫算法思想在70年代就提出了。某一时刻挂起用户程序（STW，stop the world），执行垃圾回收机制——分两个阶段：

* 标记：从一些根节点开始标记对象，逐步遍历存活单元，直至无剩余可遍历单元；这一步需要STW，否则可能错误 回收用户新创建的对象。
* 清扫：回收剩余未标记单元。

三色标记法是对传统标记阶段的改进，分为白色（未扫描对象）、灰色（待遍历对象）与黑色（已遍历对象）：

![mark with colors](/static/images/2021-02-19/mark-with-colors.png)

```go
// 1. 标记所有栈上对象为灰色
// 2. 取出所有灰色对象并标记为黑色，将该对象引用的对象标记为灰色，直至没有灰色对象
func mark() {
  for object in range getStackObjects() {
    markGray(object)
  }
  for {
    object, ok := getNextGrayObject()
    if !ok {
      break
    }
    markBlack(object)
    markGray(object.GetReferences())
  }
}
```

三色标记算法可以使得STW的时间大大减少，使得用户程序和垃圾回收算法能并发进行。实际上，Go 1.10版本之后，平均每次STW的时间降低到了**0.5ms**：

![](https://blog.golang.org/ismmkeynote/image6.png)

#### 是什么原因使得用户程序和标记-清扫算法不能并发执行？

是因为用户程序对于标记过程的干扰。为避免错误清扫对象，从而需要STW。

那么，“干扰”有哪些呢？

* 用户新创建了对象——该对象一直保持白色，最后可能被错误地回收；
* 用户将一个白色对象从灰色对象解除引用，并使一个黑色对象引用它——该白色对象不会被扫描到，因为黑色对象意味着相关引用对象已经扫描完毕，从而该白色对象被错误地回收。

#### 与用户程序并行运行时的垃圾回收算法，如何保证正确性？

依赖于写屏障（Write Barrier）。写屏障本意是操作系统内的一种机制，它保证写入存储系统的过程按特定顺序进行；在垃圾回收算法中，写屏障是**在每次写入时所执行的特定的代码**。

我们在标记过程中开启写屏障，从而试图避免用户程序对标记过程的干扰。Go 1.8之前的方法是使用Dijkstra在78年提出的方法——插入写屏障（Insertion Write Barrier）：

每次引用发生的时候，如果被引用对象是白色，则将之设置为灰色，类似于：

```go
func writePointer(slot, ptr) {
  if isWhite(ptr) {
    markGray(ptr)
  }
    *slot = ptr // 设置引用
}
```

这一写屏障恰好解决了上文提到的用户程序对于标记过程的干扰：

* 用户新创建的对象，直接被标记为灰色，避免了错误回收；
* 当白色对象的父节点从灰色对象改为黑色对象时，该对象被标记为灰色，也避免了错误回收。

![insertion write barrier](/static/images/2021-02-19/insertion-write-barrier.png)

这是因为插入写屏障满足了**强三色不变性**：

>  黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象。

#### 插入写屏障有怎样的cost？

栈上对象被置黑后，为了避免错误回收，要么需要对栈上对象加入写屏障（频繁操作开销增加），要么需要重新扫描栈（STW）。

因此，Go在1.5版本至1.7版本，开启插入写屏障后，只对堆上的指针变动进行置灰，而对于栈上的指针不作更改；**标记完成后的STW，会对栈上的白色对象重新进行一次标记**。

Go从1.5以前每次STW耗时从\~**100ms**降低到了该阶段所需要花费的\~**10ms**。

#### 如何进一步优化插入写屏障，降低STW耗时？

这里引入1990年由Yuasa提出的删除写屏障（Deletion Write Barrier）：

对象的引用被删除时，如果该对象是白色，则该对象被置为灰色。代码示例如下：

删除写屏障的可靠性来源于其满足**弱三色不变性**：

> 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径

从而保证了白色对象在删除引用时，其自身和子节点总能在标记阶段被标记为黑色，从而避免错误回收造成悬垂指针。

最终，Go 1.8版本引入了结合插入写屏障和删除写屏障的混合写屏障（Hybrid Write Barrier）：

```go
func writePointer(slot, ptr) {
  if isWhite(*slot) {
    markGray(*slot) // [删除写屏障]删除引用时，被解除引用的对象标记灰色
  }
  if isCurrentStackGray() {
    markGray(ptr) // 当前goroutine栈如果尚未被扫描完，则指针指向对象标记灰色
    // 这是因为Go 1.8之前已经设置了，写屏障开启时，所有新对象都被标记为黑色
    // 因此该指针所在的goroutine的栈还没被扫描时，该指针置为灰色以便进一步扫描
    // 若当前指针所在goroutine已经为黑色时，
    // * 该指针要么已经被扫描（灰色/黑色）
    // *要么是新分配对象（黑色）
  }
  *slot = ptr
}
```

##### 混合写屏障是否解决了插入写屏障的问题？

实际上问题在于：混合写屏障是否避免了1. 在标记结束后STW然后重新扫描栈； 2. 对栈上对象开启写屏障？

插入写屏障之所以需要重新扫描栈，是白色对象被栈上黑色对象的指针引用；现在因为删除写屏障，这类白色对象会被置灰。因此无需重新扫描栈。且注意到，写屏障是对该类白色对象置灰而不会改变栈上黑色对象的颜色，因此避免了对栈上对象开启写屏障的性能损失。

因此，Go 1.8引入的混合写屏障即保证了性能，又降低了重新扫描栈带来的STW开销。每次GC的STW时间从插入写屏障1.5版本的~10ms降低到了1.8的~0.5ms。

#### Go 1.8之后为什么还需要STW？

还有两个阶段需要STW：

* GC开始前的准备工作，例如设置写屏障；
* 标记结束时，重新扫描全局变量、扫描系统栈、结束标记过程等。