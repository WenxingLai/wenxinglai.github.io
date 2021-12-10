---
title: 深入Go：Context
tags: Go
---
在理解了 package context 的使用后，我们很自然地想问其背后的设计哲学有什么？实际上，我们发现无论是在关于 Context 的批评/讨论也不少，那么 Context 的设计合不合理？带着这些疑虑，我们深入 context 的源码，尝试对这些问题作出解答。
<!--more-->

在之前的文章中我们了解了`Context`的使用，我们很自然地会提出一些问题：

* `Context`被传递到多个goroutine中，如何保证没有data race？
* 有`CancelFunc`的`Context`，如果不被取消，会有怎样的风险？
* 父`Context`被取消，如何自动地使得所有子节点被取消？
* `Context`的`Value`如果多次被同一个`key`写入值，结果会是怎样？
* 听说`Context`以链表的形式存储Value，会不会有性能问题？

带着这些问题，我们一同进入`context`包，看看`Context`的设计有着怎样的精妙之处与潜在的坑。

我们这里略过包的注释、`Canceled, DeadlineExceeded error`以及`Context`的定义（可以参见上一篇文章），直接从“原初”的`emptyCtx`开始。

注意，从下文开始，我们会在源码中加入形如⑴、⑵或㊟的记号，表示后续会进行详细阐释；如未特殊说明，`//`开头的注释为源码注释的翻译，`/* */`包围的注释为笔者所加评注。

## 源码与解析

源码来自go 1.17.3。

### emptyCtx, Background()与TODO()

```go
// 一个emptyCtx不能被取消、没有Values或deadline。
// 它不是struct{}类型因为每个emptyCtx实例都需要不同的地址㊟。
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}

func (e *emptyCtx) String() string { /* 见下方 var background 和 todo */
    switch e {
    case background:
        return "context.Background"
    case todo:
        return "context.TODO"
    }
    return "unknown empty Context"
}

var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)
/* 即，emptyCtx的实例有且仅有这里的 backgroud 与 todo */

// Background 返回非nil的空Context。它不能被取消、没有Values或deadline。
// 它被用于main函数、初始化、测试与顶层的请求。
func Background() Context {
    return background
}

// TODO 返回非nil的空Context。当不确定应该使用哪一个Context或Context暂时不可用
// （因该处的函数还没有被扩展以接收Context参数）时，代码应使用context.TODO。
func TODO() Context {
    return todo
}
```

#### ㊟ 为什么不使用 struct{}？

请看下方代码片段：

```go
type S struct{}

func s1() {
    s1, s2 := S{}, S{}
    println(&s1 == &s2)
}

func s2() {
    s1, s2 := S{}, S{}
    fmt.Printf("%p, %p, %t\n", &s1, &s2, &s1 == &s2)
}

func main() {
    s1()
    s2()
}
```

请问输出应该是什么？答案是：

```
false
0x119e408, 0x119e408, true
```

我们用`go run -gcflags '-m' main.go`运行，可以发现：

```
./main.go:19:2: moved to heap: s1
./main.go:19:6: moved to heap: s2
./main.go:20:43: &s1 == &s2 escapes to heap
./main.go:20:12: []interface {} literal does not escape
```

如果`struct{}`的实例逃逸到heap上，那它们的地址可能相同。事实上，[The Go Programming Language Specification: Size and alignment guarantees](https://go.dev/ref/spec#Size_and_alignment_guarantees)确认了：

> A struct or array type has size zero if it contains no fields (or elements, respectively) that have a size greater than zero. Two distinct zero-size variables may have the same address in memory.
>
> struct或者array如果其不包含size大于0的字段，则其size为0。两个不同的size为0的变量可能拥有同一个地址。

因此，`emptyCtx`类型不使用`struct{}`，为了确保`todo`和`background`拥有不同的地址。

### WithCancel与ConcelFunc

```go
// 一个CancelFunc告知相关操作应被取消。
// 一个CancelFunc并不等待操作结束。
// 一个CancelFunc可以被多个goroutine并发调用。
// 第一次调用后，对CancelFunc的随后调用无实际效果。
type CancelFunc func()

// WithCancel 返回parent的一个拷贝以及一个新的Done channel。
// 在返回的cancel函数被调用时，或其父context的Done channel被关闭时，
// 返回的context的Done channel被关闭。
//
// 取消该context将释放相关的资源⑴，
// 因此代码应该在本Context相关的操作结束时立即调用cancel。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx 返回初始化后的cancelCtx实例。
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}

// goroutines ...; 测试用。
var goroutines int32

// propagateCancel 使得parent被cancel时，cancel掉child。
func propagateCancel(parent Context, child canceler) {
    done := parent.Done()
    if done == nil {
        return // parent不能被cancel
    }

    select {
    case <-done:
        // 此时parent已经被cancel
        child.cancel(false, parent.Err())
        return
    default:
    }
    /* parentCancelCtx 尝试返回parent的cancelCtx指针 */
    if p, ok := parentCancelCtx(parent); ok { 
        p.mu.Lock()
        if p.err != nil {
            // parent 已经被cancel
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else { /* 例外，此时只能新启动协程监听parent.Done */
    /* 例如parent恰好被cancel，或parent为非cancelCtx结构(没法通过children来cancel) */
        atomic.AddInt32(&goroutines, +1)
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

// &cancelCtxKey 为cancelCtx返回其自身的指针值的key。
var cancelCtxKey int

// parentCancelCtx 返回承载parent的*cancelCtx。
// 本函数通过查parent.Value(&cancelCtxKey)来找到最里层的*cancelCtx，
// 并检查parent.Done()是否匹配该*cancelCtx。
// （如果不匹配，则该*cancelCtx已被嵌入非默认的、提供不同done channel的实现中，
// 此时我们不应绕过该它⑵。）
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    pdone, _ := p.done.Load().(chan struct{})
    if pdone != done {
        return nil, false
    }
    return p, true
}

// removeChild 从parent处移除该context。
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}

// 一个canceler为可以被直接cancel的context类型。
// 实现包括：*cancelCtx和*timerCtx。
type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}

// closedchan 为用于复用的、代表已关闭的channel⑶。
/* 在init中确保closedchan被关闭 */
var closedchan = make(chan struct{})

func init() {
    close(closedchan)
}

// cancelCtx实例可以被cancel。当其被cancel，
// 该实例也将cancel其所有实现了canceler的子节点
type cancelCtx struct {
    Context /* 此即parent Context */

  mu       sync.Mutex            // mu用于保护下列字段
  done     atomic.Value          // 延迟创建的done用于存储chan struct{}
                                  // 并在第一次cancel时被关闭
    children map[canceler]struct{} // 第一次cancel将该字段设为nil
    err      error                 // 第一次cancel将该字段设为非nil
}

func (c *cancelCtx) Value(key interface{}) interface{} {
    if key == &cancelCtxKey {
        return c
    }
    return c.Context.Value(key) /* 在parent的Value中查找 */
}

func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load()
    if d != nil {
        return d.(chan struct{})
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    d = c.done.Load()
    if d == nil {
        d = make(chan struct{})
        c.done.Store(d)
    }
    return d.(chan struct{})
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}

type stringer interface {
    String() string
}

func contextName(c Context) string {
    if s, ok := c.(stringer); ok {
        return s.String()
    }
    return reflectlite.TypeOf(c).String()
}

func (c *cancelCtx) String() string {
    return contextName(c.Context) + ".WithCancel"
}

// cancel 关闭c.done、cancel所有c的子Context，且如果
// removeFromParent为true，则将其从parent的children中移除
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // 已经被cancel过
    }
    c.err = err
    d, _ := c.done.Load().(chan struct{})
  if d == nil { // done还没被使用chan struct{}创建
        c.done.Store(closedchan)
    } else {
        close(d)
    }
    for child := range c.children {
        // 注意：此时持有parent的锁，并申请child的锁
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

#### ⑴ 完成后立即cancel

源码的注释或`go vet`都要求我们在操作完成后立即调用`cancel`保证资源及时释放，例如通过`defer cancel()`的方式。那么对于`cancelCtx`，及时`cancel`释放了什么资源？

一是我们留意到，`propagateCancel`这里有如下函数：

```go
go func() {
  select {
    case <-parent.Done():
    child.cancel(false, parent.Err())
    case <-child.Done():
    }
}()
```

即某种情况下，可能需要新增协程来监听parent或自身的done channel，如果及时cancel则该协程会及时退出。（至于什么时候会新增协程来监听，见注释⑵。）

第二点，显然cancel可以使得本context从parent.children中移除；并且，这是在parent不被cancel的情况下，唯一释放该child的方法。经测试，parent为`*cancelCtx`，以此调用1000000次`WithCancel`然后直接返回，系统内存占用为205MB，如果child都立即被cancel，则系统内存占用为70MB。

三也很显然，cancel该Context后，所有子节点都会被cancel掉，从而可以使得更多地资源被及时回收。

#### ⑵ parentCancelCtx

为什么通过`p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)`找到`*cancelCtx`（即`ok == true`）之后，还需要确保`pDone == p`呢？

看如下代码：

```go
type CustomContext struct {
    context.Context
    c chan struct{}
}

func (c *CustomContext) Done() <-chan struct{} {
    return c.c
}
```

这里，如果`CustomContext.Context`为`*cancelCtx`，且被传入`context.WithCancel`，那么在`parentCancelCtx`中会找到`CustomContext.Context`，但这里如果直接返回，就返回的是祖父节点的的指针。

我们也可以用如下代码验证：

```go
func main() {
    println(runtime.NumGoroutine()) // 1
    inner, cancel := context.WithCancel(context.Background())
    defer cancel()
    c := &CustomContext{inner, make(chan struct{})}
    _, cancel2 := context.WithCancel(c)
    defer cancel2()
    println(runtime.NumGoroutine()) // 2
}
```

正因为`WithCancel`的操作，新增了一个goroutine用于监听`c.Done()`。

#### ⑶ closedchan

为什么需要全局变量（且专门在`init`中close掉的）`closedchan`？

这是因为，`done`的注释说明了该`chan struct{}`是延迟创建的，且正好是被调用`Done`时创建，如果一个Context尚未被调用`Done`就被cancel了，那么如果没有`closedchan`则需要新创建一个channel并立即close掉。

### WithDeadline与WithTimeout

```go
// WithDeadline 返回附有截止时间不晚于d的parent的拷贝。
// 如果parent的deadline早于d，
// WithDeadline(parent, d)语义上与parent相同。
// 返回的Context的Done channel将被关闭，当下列任一情况满足：
// 到达截止时间，或
// 返回的cancel函数被调用，或
// parent的Done channel被关闭。
//
// 取消该context将释放相关的资源㊟，
// 因此代码应该在本Context相关的操作结束时立即调用cancel。
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // 已有的截止时间早于参数d
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // 截止时间已超过
        return c, func() { c.cancel(false, Canceled) } /* 此时返回的c已被cancel */
    }
    c.mu.Lock() /* 因为可能此时parent被cancel，所以需要用c.mu保护 */
    defer c.mu.Unlock()
    if c.err == nil { /* 也是防止在获取锁之前因parent而被cancel */
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}

// timerCtx实例包含一个计时器与截止时间。它嵌入了一个cancelCtx来实现Done与Err。
// 它通过停止其计时器并使用cancelCtx.cancel来实现cancel。
type timerCtx struct {
    cancelCtx
    timer *time.Timer // 受cancelCtx.mu保护。

    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) String() string {
    return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
        c.deadline.String() + " [" +
        time.Until(c.deadline).String() + "])"
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // 从parent处移除c
        removeChild(c.cancelCtx.Context, c) /* 真正的parent是c.cancelCtx.Context */
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}

// WithTimeout 返回WithDeadline(parent, time.Now().Add(timeout))。
//
// 取消该context将释放相关的资源，
// 因此代码应该在本Context相关的操作结束时立即调用cancel：
//
//     func slowOperationWithTimeout(ctx context.Context) (Result, error) {
//         ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
//         defer cancel()  // 如果slowOperation在超时之前就完成了，则释放资源
//         return slowOperation(ctx)
//     }
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

#### ㊟ 完成后立即cancel

这里需要及时cancel的原因与`cancelCtx`类似，只不过有timer兜底，不会“永远地”内存泄漏。

### WithValue

```go
// WithValue 返回parent的拷贝，并附上key对应的值val。
//
// 仅对不同进程与API间转移的请求范畴内的数据使用context的Value，
// 而不是用以传递函数的可选参数。
//
// 提供的key必须是可比较的类型，且为避免在各使用context的包内冲突，
// 它不应是字符串或任意内置类型。使用WithValue的用户应自定义key的类型。
// 为避免赋值给interface{}时的内存分配，context key通常使用类型struct{}。
// 另外，导出的context key变量的静态类型应该为pointer或interface。
func WithValue(parent Context, key, val interface{}) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}

// valueCtx实例携带key-value pair。它对该key实现了Value函数，
// 并使用嵌入的Context来应对其余函数调用。
It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
    Context
    key, val interface{}
}

// stringify 尝试在不使用fmt的情况下将v转换为字符串，这是因为
// context并不希望依赖unicode表。本函数仅在*valueCtx.String()中使用。
func stringify(v interface{}) string {
    switch s := v.(type) {
    case stringer:
        return s.String()
    case string:
        return s
    }
    return "<not Stringer>"
}

func (c *valueCtx) String() string {
    return contextName(c.Context) + ".WithValue(type " +
        reflectlite.TypeOf(c.key).String() +
        ", val " + stringify(c.val) + ")"
}
/* Value 以类似链表的形式实现，我们在稍后会讨论到 */
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

## 对Context的批评（并尝试回应）

### 批评

Michal Štrba在文章*[Context should go away for Go 2](https://faiface.github.io/post/context-should-go-away-go2/)*中批评了Context的设计，甚至说：

> If you use `ctx.Value` in my (non-existent) company, you’re fired.
>
> 如果你在我（并不存在的）公司中使用`ctx.Value`，你将被开除。

TA的指责主要可以总结为：

* 从请求生命周期开始到结束的每一个函数，即使有的并不需要使用到`ctx`，也被迫需要将其作为第一个参数。

    > Context is like a virus.

* `context.Value`问题重重：

    * 并非静态类型安全，需要类型断言。
    * 存储的内容并非静态可感知。
    * 可能命名冲突造成问题。

* 使用类似于链表的结构，存在性能问题。

* `ctx context.Context`看起来就很啰嗦（然后TA顺便黑了一下Java：让人想起`Foo foo = new Foo();`）。

这里，我们尝试对Michal Štrba的观点作一个回应。

### 回应

#### cxt污染

Context本来就是用于控制请求的生命周期的，所以很自然地从始至终需要传递；退一步讲，如果换其他实现，想达到能控制整个请求生命周期的目的，也需要始终传递某个参数——不然怎么能实现“控制整个请求生命周期”？

以及，并不是所有核请求相关的函数都需要`ctx`参数，那些与API调用无关的过程自然也就不需要该参数——该参数仅存在于请求的“主干”上。Michal Štrba其实误解了Context的用法。

#### Value的问题

的确，没有固定类型的Value是代价，但换取的是灵活性。我们总能想起来“但是，古尔丹，代价是什么呢”，那么我们也应该想到，“但是，gopher，代价带来的是什么呢”。

我们考虑以下两点，其实可以或多或少地排除掉这个顾虑：

* 正如源码文档所写，应该使用存取函数来完成值的读写，而不是直接操纵`ctx.Value`本身；并且，`key`都使用非导出的包作用域的变量，自然不会存在冲突的问题；
* 在此前的文章*[The Context of the Package context](https://km.woa.com/articles/view/529748)*中，我们非常认可Jack Lindamood在Gophercon UK 2017所述：*`Context.Value` should inform, not control*；真正必不可少的“参数”，应该是通过函数参数来传递，而不是Context——这也在源码文档里有专门提到。

#### 性能问题

首先回应Michal Štrba指责的，`cancelCtx`有时需要goroutine来通知，但通常我们不会去重定义Context的`Done`返回的channel，实际上你是否能想到非得重定义`Done`的行为的必要场景？

其次是，Context之间的内嵌使得节点关系是近似于链表的结构，而不是更高效的数据结构。比如很容易想到的，`WithValue`竟然是通过新增Context节点来完成的。

那这个代价换来了什么？换来的是严格意义上的父子节点的关系。或者思考，如何实现一个仅能访问自身节点与祖先节点所存储的数据的结构？

并且我们退一步讲，代价究竟有多大。首先，`WithValue`并不是一个应该被频繁调用的函数，这点我们不再赘述，所以用于存储Value的这部分链表的长度其实是有限的；其次是，对于cancel被传递下去的代价是什么，其实回顾`cancelCtx`或`timerCtx`的代码，可以发现里面的操作都是必须的，并没有什么冗余，且基本没有需要等待channel的情况（如果不考虑重新定义`Done`返回的channel这一情况）。

实际上，我们测试了，通过连续调用1000次`WithCancel`，然后第一个Context的cancel被调用，第1000个Context平均在0.10ms后`Done()`接收到结果。要留意到，Context是用来控制耗时以毫秒为单位的请求的，似乎看起来Context本身的开销其实微乎其微。

#### 啰嗦

Emmm，除非把Context设为内置类型并缩短命名，使得可以变为`func (ctx Ctx)`外，好像没啥好说的。

#### 小结

我们对Context提出诘难的时候也应该思考，我们是否真正用对了Context，我们是否有更好的解决方法呢？

## 解答提出的问题

### 如何保证没有data race

看了代码可以知道，`todo`/`background`无需保护、`cancelCtx`使用mutex和原子操作来保护`done`、`children`和`err`、`timerCtx`嵌入了`cancelCtx`并以其mutex保护自己的`timer`，而`valueCtx`的key-value pair都是只读的，因此不用担心data race。

不过这里需要注意的是，`Context.Value`不应存储并发访问不安全的数据。

### 关于cancel

通过代码，我们知道了cancel的传递（在非自定义`Done`返回的channel的情况下）是通过`cancelCtx.children`来完成的；cancelCtx的子节点不手动cancel的话，可能会使得`parent.children`持续膨胀，导致泄露。

### 关于重复赋值value

请不要重复赋值value。但阅读代码之后可以发现，`Value`的调用是从子节点回溯到祖先节点，因此会找到最新的value（但并不会覆盖原有值）。

```go
func main() { // 注意，不要用内置类型作为key的类型
    c1 := context.WithValue(context.Background(), hello, "world")
    c2 := context.WithValue(c1, foo, "bar")
    c3 := context.WithValue(c2, hello, "today")
    c4 := context.WithValue(c3, bar, "baz")
    fmt.Println(c4.Value(hello)) // today
    fmt.Println(c2.Value(hello)) // world
}
```

### 关于性能问题

在上一节已经探讨过了，不再赘述。

## 总结

关于Context的使用，请参加之前的文章《使用context包轻松完成并发控制》。

我们以如下代码行的示意图图来作结：

```go
v1 := context.WithValue(context.Background(), foo, 1)
c, cancel := context.WithCancel(v1)
defer cancel()
done := c.Done()
t, cancel1 := context.WithTimeout(c, time.Second)
defer cancel2()
v2 := context.WithValue(t, bar, "baz")
c2, cancel3 := context.WithCancel(t)
cancel3()
// <- now we give the image representing the state here
return
```

![contexts](/static/images/2021-12-06/contexts.png)

此时，我们调用Context的各函数，会发生：

|      | Deadline     | Done          | Err        | Value(foo) | Value(bar) |
| ---- | ------------ | ------------- | ---------- | ---------- | ---------- |
| `v1` | `0, false`   | `nil`         | `nil`      | `1`        | `nil`      |
| `c`  | `0, false`   | `done`        | `nil`      | `v1.Value` | `v1.Value` |
| `t`  | `+1s, true`  | new a channel | `nil`      | `c.Value`  | `c.Value`  |
| `v2` | `t.Deadline` | `t.Done`      | `nil`      | `t.Value`  | `"baz"`    |
| `c2` | `t.Deadline` | `closedchan`  | `Canceled` | `t.Value`  | `t.Value`  |

-----

我的博客即将同步至腾讯云+社区，邀请大家一同入驻：[腾讯云+社区链接](https://cloud.tencent.com/developer/support-plan?invite_code=1jpnp7ydes4cr)
