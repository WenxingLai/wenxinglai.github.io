---
title: 深入Go：错误的包装与解包
tags: Go
---

仔细想想，我们的Go代码中可能有四分之一的代码都是和错误处理相关的，而我们已经接受了，error无处不在。但似乎Golang的error处理并不够强大，也缺乏统一的错误处理流程的逻辑；在经历了大量的讨论后，Go 1.13引入了错误的包装和解包，也许某种程度上可以优化我们的错误处理流程。
<!--more-->

###### 太长不看版

* `error`的作用有两点：一是让代码进入特定的错误处理流程，二是告诉程序员发生了什么状况
* `error` interface只包含了`Error() string`这一方法，error难以仅仅通过字符串的匹配来完成上述两种角色
* Go在1.13版本中引入了错误的包装与解包
  * 仅需`fmt.Errorf("...%w...", ..., err, ...)`就可完成`error`的包装
  * 可通过`errors.Is(err error, target error) bool`和`errors.As(err error, target interface{}) bool`实现解包，作用分别是：`error`是否包含`target`、是否包含可转换为`target`的错误
* 在实践中，我们总是可以
  * 包装error以便添加函数调用的上下文参数以便问题排查
  * 在最终的栈底进行打印与解包，打印直接使用`Error() string`方法，解包解析出需要的固定错误以作为API接口的响应返回

（太长不看版结束）

---------------------

假设我们需要实现一个服务，对于管理员用户返回请求中ID所对应的数据，否则返回错误；该服务需要符合云API3.0的错误码规范，代码很简单：

```go
func HasPermission(ctx context.Context, uin string) error {
  role, err := getRole(ctx, uin)
  if err != nil {
    // logging uin
    return err
  }
  if role != admin {
    return apierr.NewUnauthorizedOperationNoPermission()
  }
  return nil
}

func (s *Service) GetData(ctx context.Context, req *Request) (*Response, error) {
  if err := HasPermission(ctx, req.Uin); err != nil {
    if err == apierr.UnauthorizedOperationNoPermission {
      // new a Response with error message and return it
    }
    // logging req and err, pack and return a Response
  }
}
```

这里我们省略了查数据库并返回结果的逻辑。这只是一个简单的接口，只包含了两个步骤——鉴权和数据库查询——每一个步骤都可能有不同的错误：有的可能需要直接返回符合规范的云API 3.0错误码便于返回给请求方，有的可能需要打日志记录中间状态与参数以便我们调试。

错误处理变得非常复杂，我们常常需要进行`err == SomeError`或者`err.Error() == SomeErrorString`来进行比较，但这样做我们又很难把错误发生的上下文关联起来、问题排查变得困难。

仅仅包含两个步骤的接口的错误处理就变得那么复杂，那么我们应该怎样重构我们Go代码的错误处理逻辑？

## error的角色

在解答上个问题前，我们需要回想，Golang的Error究竟要承担怎样的职责、在代码运行中应该扮演怎样的角色？

实际上，error的角色分为：针对代码的和针对程序员的。

* 针对代码的：让代码进入特定的错误处理流程
* 针对程序员的：告诉程序员发生了什么状况

所以，error的处理应该面向这两点：

* 针对代码的：类型判断（错误是哪一种错误）
* 针对程序员的：打印字符串（把错误如何出现呈现出来）

但是`error`就只是一个拥有`Error() string`的接口，如何实现error的双重角色？

## error的包装与解包

Golang在1.13的release中引入了error的包装与解包，详见[[Working with Errors in Go 1.13](https://blog.golang.org/go1.13-errors)](https://blog.golang.org/go1.13-errors)。这里我们进行一个简单的语法介绍，然后在后文中详细说明如何实践。

### error的包装

举个例子，假设函数接收到了一个error，希望加入更多的上下文信息：

```go
func NewOSError(msg string) error {
  return &OSError{msg}
}

var usingWindows = NewOSError("Upgrading Windows. Sit back and relax.")

func send(message string) error {
  return usingWindows
}

func Send(message string) error {
  err := send(message)
    if err != nil {
    e := fmt.Errorf("Send(%q): %w", message, err)
    return e
    }
  return nil
}

func main() {
  err := Send("I'm using a Mac.")
  if err != nil {
    println(err.Error())
  }
}
// Send("I'm using a Mac."): Upgrading Windows. Sit back and relax.
```

我们单纯只是在调用fmt.Errorf的时候，把`%v`换成了`%w`，然后打印错误信息的时候，error自动调用了其`Error() string`方法。但之所以叫“error的包装”，是因为这样的方法得到的新error可以被解包。

### error的解包

#### `errors.Is(err error, target error) bool`

`errors.Is(err error, target error) bool`方法会解包所有`err`里包装的error，如果里面有任何一个解包后`== target`，则返回`true`。例如：

```go
func main() {
  err := Send("I'm using a Mac.")
  if err != nil {
    if errors.Is(err, usingWindows) {
      println(" Less than a minute remaining...")
    } else {
      println(err.Error())
    }
  }
}
//  Less than a minute remaining...
```

#### `errors.As(err error, target interface{}) bool`

`func As(err error, target interface{}) bool`方法会解包所有`err`里包装的error，并且看是否能类型转换为`target`的类型，如果可以，则将转换后的结果赋值到`target`。例如：

```go
func main() {
  err := Send("I'm using a Mac.")
  if err != nil {
    var osError *OSError
    if errors.As(err, &osError) {
      println("Got an OSError!")
    } else {
      println(err.Error())
    }
  }
}
// Got an OSError!
```

## error包装解包的实践

回到我们刚才的代码，我们的希望也就是对应于error的两个角色：

* 针对代码的：接口能根据error最终能正确返回符合云API 3.0的Response
* 针对程序员的：能记录下调用链中的上下文并最终打印出来

因此，原有代码可以这样设计：

```go
func HasPermission(ctx context.Context, uin string) error {
  var err error
  defer func() { // 添加上下文信息
    if err != nil {
      err = fmt.Errorf("HasPermission(%q): %w", uin, err)
    }
  }()
  role, err := getRole(ctx, uin)
  if err != nil {
    return err
  }
  if role != admin {
    return apierr.NewUnauthorizedOperationNoPermission()
  }
  return nil
}

func (s *Service) GetData(ctx context.Context, req *Request) (*Response, error) {
  var err error
  handler := func(e error) (*Response, error) {
    log.Errorf("GetData(%q): %q", req, e.Error()) // 打印错误信息
    r := &Response{}
    var apiError *APIError
    if errors.As(e, &apiError) { // 解包错误并得到“可返回”的错误
      r.Error = apiError.ToError()
    } else { // 无法解包，使用默认的“可返回”的错误
      r.Error = apierr.NewFailedOperationError(e)
    }
  }
  if err := HasPermission(ctx, req.Uin); err != nil {
    return handler(err)
  }
  data, err := retriveData(ctx, req.Key)
  if err != nil {
    return handler(err)
  }
  // return normally
}
```
