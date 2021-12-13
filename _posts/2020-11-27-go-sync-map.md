---
title: 深入Go：sync.Map
tags: Go
key: go-sync-map
---

我们在使用Go的项目中需要有并发读写的map时，我们了解到Go提供`sync.Map`这一数据结构；通过对其简单了解，发现它正好适合我们需要的场景。随着了解的深入，我们又有了疑惑：为什么不像Java SE 8之前的ConcurrentHashMap一样，使用分段锁？为什么在内部需要一个哨兵指针`expunged`？这两个问题我们简单Google后都没有找到解析和讨论，因此我们决定深入`sync.Map`的源代码，尝试回答这两个问题。
<!--more-->

#### 太长不看版

##### 预备知识

* map的读写删除都不是原子操作，因此需要控制并发访问，而Go的原生map不支持并发读写；
* Go在1.9的版本中新增了`sync.Map`的数据结构，通过空间换时间的方式降低了加锁的频率：
  * 使用`read`和`dirty`两个map来保存键值，map的值为指向`entry`的指针，`entry`存储指向真实值的指针
  * 当需要向`dirty`中插入新值时，如果`dirty`为空则将除`entry.p == nil`外的`read`的键值浅拷贝到`dirty`（`nil`的处理我们后面会详细讨论，这也是`sync.Map`的highlight）
  * `read`中包含的键可不加锁地读和删（不是删除对应值，而是将`entry.p = nil`以表示对应值被删除了）
  * 除了一种特殊情况外，更新`read`中已经存在的键的值也无需加锁，该特殊情况在后文会详细讲到
  * 对`dirty`的读写删除都需要加锁，当`dirty`中包含`read`中没有的键值时（`read.amended = true`），会
    * 读写`read`中不存在的键：加锁并在`dirty`中完成对应操作
    * 删除`read`中不存在的键：直接`delete(dirty, key)`
  * 当对`dirty`中的访问数量大于其长度时，直接将`read`中map的指针替换为`dirty`，并将`dirty`置为`nil`、`read.amended`置为`false`。

##### 两个问题的回答

* 为什么不像Java SE 8之前的ConcurrentHashMap一样，使用分段锁？
  * 我们猜测是因为为了避免重写一套map的逻辑，`sync.Map`可以直接使用`map`作为内部的实现而无需重写分配到哪一段的hashing逻辑；
  * 也因为`sync.Map`的适用场景和ConcurrentHashMap不同。`sync.Map`适合读写删除的键值范围比较固定的情况（即基本都能在`read`中命中而无需加锁查询`read`），不适合需要频繁增加、访问新值的情况（即频繁读写`dirty`）。后者建议使用分段锁的map。
* 哨兵指针`expunged`有什么作用？
  * 为了保证`dirty`和`read`键值的同步，以保证在将`read`替换为`dirty`时能一步完成。

#### 为什么需要可并发访问的Map

Map是Go语言中广泛使用的数据结构，但它并不是可并发读写的。尝试以下代码，会得到`fatal error: concurrent map read and map write`。

```go
func main() {
    m := make(map[int]int)

    go func() {
        for i := 0; i < 100000; i += 1 {
            m[i] = i
        }
    }()
    for i := 0; i < 100000; i += 1 {
        _ = m[i]
    }
}
```

这是因为在Map的源码中，有

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
  // 如果在读map的时候检测到hashWriting flag为1（即，有协程在写该map）则panic
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
}
```

本质上是因为，map的读写都不是“原子操作”，试想：当你尝试读（copy）一个struct的时候，突然另一个协程将该struct删除并触发垃圾回收、或者另一个协程更新了该struct——我们就变成了读一个非法的struct，从而导致问题。

在实践中，往往我们又需要一个可并发读写的Map。那么，如何解决？

#### 并发读写的Map

我们不假思索就可以想到，可以通过加锁解决——每次读写，我们都给map加上mutex——但加锁意味着性能损失，因此我们要尝试减少加锁的次数。

首先我们可以考虑使用`RWMutex`——该mutex可被任意多的读协程获取，或被一个写协程获取——使用`RWMutex`直接降低了多读少写的Map的性能损失。

当然，我们也可以借鉴Java SE 8之前经典的`ConcurrentHashMap`的实现方式——将Map进行分段，每个分段进行加`RWMutex`的操作。这一思路在orcaman实现的[concurrent-map](https://github.com/orcaman/concurrent-map)中被使用。

Go的思路和Java的`ConcurrentHashMap`不同，因为`ConcurrentHashMap`总的来说是对map的重写，因为分段加锁需要新的hashing逻辑（因此orcaman实现的concurrent-map，就是为了避免实现新的hashing逻辑，所以只支持键的类型为string）；Go的map本来已经实现得非常优秀，那么如何利用已经存在的map，构建出并发读写安全的数据结构呢？

#### Introducing sync.Map

Go在1.9的版本中引入了`sync.Map`这一支持并发读写的数据结构，其利用空间换时间的思路，使用成员`read`、`dirty`来优化固定场景下读写的性能——只对`dirty`的访问加锁，即当用户读写`read`中的entry时，sync.Map并不加锁，当用户读写`dirty`中存在的entry时，sync.Map才对该操作加锁。我们接下来通过读`sync.Map`的[源代码](https://github.com/golang/go/blob/master/src/sync/map.go)来了解：

* `sync.Map`有着怎样的结构
* 如何读
* 如何写
* 如何删
* 为何使用`expunged`值
* `sync.Map`优化了哪些场景下的性能

##### sync.Map的结构

`sync.Map`使用`map`作为底层的实现，因此其内部继承了map良好的读写性能；不过其依赖的底层`map`存储的“值”并不是存的用户提供的值，而是指向`entry`的指针类型，用以判断对应键值的状态，关于`entry`的代码如下：

```go
// expunged是初始化时随机生成的哨兵值，
// 标志该值已经从map的read中删除且不存在于dirty中。
var expunged = unsafe.Pointer(new(interface{}))

type entry struct {
    // p为指针，可指向：
  // -> nil，表示对应键值已从map的read中删除，但存在于dirty中
  // -> expunged，表示对应键值已从map中删除，但不存在于dirty中
  // -> 其他地址，指向真实数据
    p unsafe.Pointer
}
```

然后看`sync.Map`中`read`成员`readOnly`结构体：

```go
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 如果dirty包含read中不包含的entry指针时为true。
}
```

下面我们就可以来看`sync.Map`本身的代码了。

```go
type Map struct {
   mu Mutex
  
   // read包含并发安全的map部分，访问它（的entries）永远不用加锁
   // 当entry需要从read删除时，并不直接删除该entry的指针e，
   // 而是将e.p置为nil。
   // 除了进行写操作且对应键的entry的e.p == expunged时，
   // （此时也不是对read加锁，而是对dirty加锁）
   // 对read中的键值进行读、写和删除都不用加锁。
   read atomic.Value // readOnly

   // dirty包括map中需要加锁读写的部分。
   // 为保证从dirty至read的更新耗时足够小，
   // 因此需要把read中从expunged状态恢复的entry加入dirty
   // （此时read仍未被加锁，是写到dirty从而对dirty加锁）。
   // 当向dirty写的时候，如果dirty为nil，
   // 则需要浅拷贝read（unplunged entries除外，详情见Store代码）
   dirty map[interface{}]*entry

   // misses是计算read更新后访问map时获取mu的计数器
   // 当misses数量超过dirty长度时，dirty会跃升为新的read并将dirty置为nil
   misses int
}
```
##### Load操作

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended { // 如果找不到，且dirty含有新键值
        m.mu.Lock()
        // 双重检查，避免另一协程已经将read替换为dirty
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
      // 从dirty中获取
            e, ok = m.dirty[key]
      // 增加misses计数，如果超过len(dirty)则将read替换为dirty并将dirty置为nil
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok { // 未找到对应键
        return nil, false
  }
    return e.load() // 返回真实值
}
```

因此我们可以知道，以下情况`Load`不需要加锁：

* 当`read.amended == false`的时候，此时`dirty == nil`
* `key`存在于`read.m`中，无论对应entry的指针为`nil`、`expunged`还是真实地址

而当`key`不存在于`read.m`中、`read.amended == true`时，`Load`需要去`dirty`中检查一次。

##### Store操作

```go
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
  // tryStore：当e.p为expunged时返回false，否则存储
    if e, ok := read.m[key]; ok && e.tryStore(&value) { 
    // 这里，未加锁尝试store
    // 因为read的entry非expunged时，要么dirty为空则无需操作dirty
    // 否则该entry指针一定和dirty中对应的entry指针指向同一entry
    // 因此只需改这里就可以使dirty中值保持一致（从而也无需加锁）
        return 
    } 

    m.mu.Lock() // 否则需要加锁访问
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() { // 此前e为expunged状态，将对应`e.p`置为nil
            // expunged状态说明此时dirty不为nil且该键不在dirty中，直接存到dirty
            m.dirty[key] = e
        }
        e.storeLocked(&value) // 然后store pointer到entry e
    } else if e, ok := m.dirty[key]; ok { // 此时为dirty中有值的状态
        e.storeLocked(&value)
    } else { // 此时为dirty中无该值的状态
        if !read.amended { // 此时为dirty中第一个值
            m.dirtyLocked() // 创建m.dirty并根据read初始化，如下
/*
func (m *Map) dirtyLocked() {
    if m.dirty != nil { // 此时m.dirty一定是nil，否则不会调用dirtyLocked
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m { // 处理read中每个entry
        if !e.tryExpungeLocked() { // 将read中含空指针的entry变为expunged并返回true
            m.dirty[k] = e // 因此read中为expunged的entry在dirty中一定不存在对应entry
        }
    }
}*/
      // 现在有dirty了，改动amended
            m.read.Store(readOnly{m: read.m, amended: true}) 
        }
        m.dirty[key] = newEntry(value) // 在dirty中存储新值，此时该值仅在dirty中存在
    }
    m.mu.Unlock()
}
```

因此我们可以知道，`Store`在`key`对应的entry `e.p != expunged`时不用加锁，除此之外都需要加锁，包括：

* `e.p == expunged`状态，需要`m.dirty[key] = e`
* 当`key`不在`read.m`中时，需要直接写到`dirty`（可能触发`dirty`从`read`的浅拷贝）

##### Delete操作
```go
// LoadAndDelete被Delete调用，是实施删除键值的函数
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
    read, _ := m.read.Load().(readOnly)
  e, ok := read.m[key] // 如果在read中找到，直接跳到下方e.delete()
    if !ok && read.amended { // 说明仅仅存在于dirty中
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly) // 双重检查
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            delete(m.dirty, key) // 在dirty中删除，此时read、dirty中都不再有该键
            m.missLocked() // 增加misses，如果需要则用dirty替换read
        }
        m.mu.Unlock()
    }
    if ok {
        return e.delete() // 如果有真实值，则将其置为nil
/*
func (e *entry) delete() (value interface{}, ok bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return nil, false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return *(*interface{})(p), true
        }
    }
} */
    }
    return nil, false
}
```

和`Load`类似，删除`read`中存在的键值无需加锁，否则需要加锁然后直接`delete(dirty, key)`。

##### 为何需要expunged的值

entry的指针一共有三个可选值，`nil`、`expunged`和指向真实的元素。为了弄清楚为什么使用`expunged`，我们需要知道：

* 指针在什么时候会变为`expunged`的值
* 为什么不仅仅使用`nil`

第一点，通过阅读代码我们知道，一个entry的`p`变为`expunged`当且仅当在加锁后、`dirty`为空，从`read`浅拷贝所有entry指针到`dirty`的时候——此时的read中所有`p`为`nil`的entry指针，`p`都变为`expunged`，此时`dirty`中将不会有对应entry的指针。

第二点，我们来看看如果仅仅使用`nil`会发生什么：

* 如果`nil`在`read`浅拷贝至`dirty`的时候仍然保留entry的指针，即拷贝完成后，对应键值下`read`和`dirty`中都有对应键下entry e的指针，且`e.p = nil`，那么之后在`dirty`跃升为`read`的时候对应entry的指针仍然会保留——那么对应键的entry指针**永远都会存在**，也就是说，`sync.Map`的规模只会越来越大；
* 如果`nil`在`read`浅拷贝时不进入`dirty`，那么之后store对应键的时候，就会出现`read`和`dirty`不同步的情况，即此时`read`中包含`dirty`不包含的键，那么之后用`dirty`替换`read`的时候就会出现数据丢失的问题；
* 如果`nil`在`read`浅拷贝时直接把`read`中对应键删除（从而避免了不同步的问题），但这又必须对`read`加锁，违背了`read`读写不加锁的初衷。

##### sync.Map优化了哪些情况下的性能

从代码中我们可以知道，对于`read`的访问是不需要加锁的，因此对于读多更新多而插入新值少的情况，也就是读写删的键值范围基本固定的情况下，`sync.Map`有着更佳的性能，因为读写删存在的键基本无需加锁；反复插入与读取新值，即操作dirty的情况更多时，`sync.Map`需要频繁加锁、更新`read`，从而性能会变差，因此不适合作为in-memory数据库使用（这样的情形更适合于使用利用分段锁的map）。
