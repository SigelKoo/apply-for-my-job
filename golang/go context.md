# go context

context主要的作用是在goroutine中进行上下文的传递，而在传递信息中又包含了goroutine的运行控制、上下文信息传递等功能。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/KVl0giak5ib4iaVJ9V2fpeV70bGa4WFWL1j3cvfToriaeeHbyuERibn2hS8tlGaOYmqO0HWbPNOrypg5ebHJiaM8SpiaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Go语言的独有的功能之一Context，**最常听说开发者说的一句话就是 “函数的第一个形参真的要传ctx 吗？”，第二句话可能是 “有没有什么办法不传，就能达到传入的效果？”**，听起来非常魔幻。

在 Go 语言中 context 作为一个 “一等公民” 的标准库，许多的开源库都一定会对他进行支持，因为标准库 context 的定位是上下文控制。会在跨 goroutine 中进行传播：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/KVl0giak5ib4iaVJ9V2fpeV70bGa4WFWL1jnPcicUhGjTbwLy2kfgleRN64htIUDy4pVibb18RndDtOnkTjxODC7ticw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

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