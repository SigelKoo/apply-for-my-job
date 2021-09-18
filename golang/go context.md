# go context

使用一个goroutine来处理一个任务，一个goroutine创建多个子任负责不同子任务的场景非常常见，一个goroutine需要向剩余goroutine传递截止时间、取消信号（通知任务取消）或其他与请求相关的数据。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/KVl0giak5ib4iaVJ9V2fpeV70bGa4WFWL1jnPcicUhGjTbwLy2kfgleRN64htIUDy4pVibb18RndDtOnkTjxODC7ticw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这种场景可以使用context，主要概括为一个context接口，四种实现emptyCtx，cancelCtx，timerCtx，valueCtx，以及六种函数Background、TODO、WithCancel、WithDeadline、WithTimeout、WithValue

Context接口提供了四种方法：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

empytCtx本质上是整型，*empytCtx对Context接口的实现只是简单的返回nil,false等

```go
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
```

Background与TODO两个函数内部都会创建emptyCtx

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

其中，Background函数主要用于在初始化时获取一个context，data本质上是一个int，tab *itab（inter为Context类型元数据，_type为\*emptyCtx类型元数据）。

TODO函数，官方文档建议在本来应该使用外层传递的ctx，外层却没有传递的地方使用，留下一个TODO

cancelCtx是可以取消的context，done用于获取取消的通知，children用于存储以当前节点为根节点的所有可取消的Context，以便在根节点取消时，可以把它们一并取消，err用于存储取消时指定的错误信息，mu是用来保护这几个字段的锁，保证cancelCtx是线程安全的。

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex
	done     chan struct{}
	children map[canceler]struct{}
	err      error
}
```

WithCancel函数可以将一个context包装成一个cancelCtx，并提供一个取消函数，可以cancel对应的context

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

timerCtx在cancelCtx的基础上又封装一个计时器和一个截止时间，这样既可以根据需要主动取消，也可以在到达deadline时，通过timer来触发取消动作，timer也由cancelCtx.mu来保护，确保取消操作也是线程安全的

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

通过WithDeadline()和WithTimeout()函数可以创建timerCtx，区别是WithDeadline函数要指定一个时间点，WithTimeout函数接收一个时间段

```
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

例：ctx2（timerCtx）是基于ctx1（cancelCtx）创建的，ctx1是基于ctx（empytCtx）创建的，基于每个context可以创建多个context，形成一棵树。可取消的context都会被注册到离它最近、可取消的祖先节点中，对ctx2来说离它最近的、可取消的祖先节点是ctx1，ctx1的children（map[canceler]struct{}）中会增加ctx2这组键值对。ctx2先取消，只影响以他为根节点的context；ctx1先取消，会把ctx1子节点中所有可取消的context全部cancel掉。

valueCtx用来支持键值对打包

```go
type valueCtx struct {
	Context
	key, val interface{}
}
```

WithValue可以给context附加一个键值对信息，可以通过context传递数据

```
func WithValue(parent Context, key, val interface{}) Context
```

子节点会覆盖父节点数据，使用自定义类型包装key，不要去修改context中的值。

在http、sql相关库中，对context支持方便我们实现超时自动取消或传递请求相关的控制数据等





# 原

context主要的作用是在goroutine中进行上下文的传递，而在传递信息中又包含了goroutine的运行控制、上下文信息传递等功能。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/KVl0giak5ib4iaVJ9V2fpeV70bGa4WFWL1j3cvfToriaeeHbyuERibn2hS8tlGaOYmqO0HWbPNOrypg5ebHJiaM8SpiaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Go语言的独有的功能之一Context，**最常听说开发者说的一句话就是 “函数的第一个形参真的要传ctx 吗？”，第二句话可能是 “有没有什么办法不传，就能达到传入的效果？”**，听起来非常魔幻。

在 Go 语言中 context 作为一个 “一等公民” 的标准库，许多的开源库都一定会对他进行支持，因为标准库 context 的定位是上下文控制。会在跨 goroutine 中进行传播：



本质上 Go 语言是基于 context 来实现和搭建了各类 goroutine 控制的，并且与 `select-case` 联合，就可以实现进行上下文的截止时间、信号控制、信息传递等跨 goroutine 的操作，是 Go 语言协程的重中之重。

```go
func main() {
	parentCtx := context.Background()
	ctx, cancel := context.WithTimeout(parentCtx, 1*time.Millisecond)
	defer cancel()

	select {
	case <- time.After(1 * time.Second):
		fmt.Println("overslept")
	case <- ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

结果为

```go
context deadline exceeded
```

我们通过调用标准库 `context.WithTimeout` 方法针对 `parentCtx` 变量设置了超时时间，并在随后调用 `select-case` 进行 `context.Done` 方法的监听，最后由于达到截止时间。因此逻辑上 `select` 走到了 `context.Err` 的 `case` 分支，最终输出 `context deadline exceeded`。

除了上述所描述的方法外，标准库 `context` 还支持下述方法：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
type Context
    func Background() Context
    func TODO() Context
    func WithValue(parent Context, key, val interface{}) Context
```

- WithCancel：基于父级 context，创建一个可以取消的新 context。
- WithDeadline：基于父级 context，创建一个具有截止时间（Deadline）的新 context。
- WithTimeout：基于父级 context，创建一个具有超时时间（Timeout）的新 context。
- Background：创建一个空的 context，一般常用于作为根的父级 context。
- TODO：创建一个空的 context，一般用于未确定时的声明使用。
- WithValue：基于某个 context 创建并存储对应的上下文信息。

```go
func WithXXXX(parent Context, xxx xxx) (Context, CancelFunc)
```

其返回值分别是 `Context` 和 `CancelFunc`，接下来我们将进行分析这两者的作用。

1. Context 接口：

   ```go
   type Context interface {
       Deadline() (deadline time.Time, ok bool)
       Done() <-chan struct{}
       Err() error
       Value(key interface{}) interface{}
   }
   ```

   - Deadline：获取当前 context 的截止时间。
   - Done：获取一个只读的 channel，类型为结构体。可用于识别当前 channel 是否已经被关闭，其原因可能是到期，也可能是被取消了。
   - Err：获取当前 context 被关闭的原因。
   - Value：获取当前 context 对应所存储的上下文信息。

2.  Canceler 接口：

   ```go
   type canceler interface {
       cancel(removeFromParent bool, err error)
       Done() <-chan struct{}
   }
   ```

   - cancel：调用当前 context 的取消方法。
   - Done：与前面一致，可用于识别当前 channel 是否已经被关闭。



在标准库 context 的设计上，一共提供了四类 context 类型来实现上述接口。分别是 `emptyCtx`、`cancelCtx`、`timerCtx` 以及 `valueCtx`。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/KVl0giak5ib4iaVJ9V2fpeV70bGa4WFWL1j2cYbh8G2tkxa4GtjibygbINrWaQUQGibhN1SWXVICnvCDZeWJJ2zwwNA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### emptyCtx

在日常使用中，常常使用到的 `context.Background` 方法，又或是 `context.TODO` 方法。

源码如下：

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

其本质上都是基于 `emptyCtx` 类型的基本封装。而 `emptyCtx` 类型本质上是实现了 Context 接口：

```go
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
```

实际上 `emptyCtx` 类型的 context 的实现非常简单，因为他是空 context 的定义，因此没有 deadline，更没有 timeout，可以认为就是一个基础空白 context 模板。