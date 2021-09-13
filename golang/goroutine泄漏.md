如果你启动了一个 goroutine，但并没有符合预期的退出，直到程序结束，此goroutine才退出，这种情况就是 goroutine 泄露。当 goroutine 泄露发生时，该 goroutine 的栈(一般 2k 内存空间起)一直被占用不能释放，goroutine 里的函数在堆上申请的空间也不能被 垃圾回收器 回收。这样，在程序运行期间，内存占用持续升高，可用内存越来也少，最终将导致系统崩溃。

回顾一下 goroutine 终止的场景：

- 当一个goroutine完成它的工作
- 由于发生了没有处理的错误
- 有其他的协程告诉它终止

那么当这三者同时没发生的时候，就会导致 goroutine 始终不会终止退出。

### goroutine泄露的场景

goroutine泄露一般是因为channel操作阻塞而导致整个routine一直阻塞等待或者 goroutine 里有死循环的时候。可以细分为下面五种情况：

#### 1. 从 channel 里读，但是没有写

```go
// leak 是一个有 bug 程序。它启动了一个 goroutine 阻塞接收 channel。当 Goroutine 正在等待时，leak 函数会结束返回。此时，程序的其他任何部分都不能通过 channel 发送数据，那个 channel 永远不会关闭，fmt.Println 调用永远不会发生， 那个 goroutine 会被永远锁死

func leak() {
     ch := make(chan int)

     go func() {
        val := <-ch
        fmt.Println("We received a value:", val)
    }()
}
```

#### 2. 向 unbuffered channel 写，但是没有读

```go
// 一个复杂一点的例子
func sendMsg(msg, addr string) error {
    conn, err := net.Dial("tcp", addr)
    if err != nil {
        return err
    }
    defer conn.Close()
    _, err = fmt.Fprint(conn, msg)
    return err
} 

func broadcastMsg(msg string, addrs []string) error {
    errc := make(chan error)
    for _, addr := range addrs {
        go func(addr string) {
            errc <- sendMsg(msg, addr)
            fmt.Println("done")
        }(addr)
    }

    for _ = range addrs {
        if err := <-errc; err != nil {
            return err
        }
    }
    return nil
}

func main() {
    addr := []string{"localhost:8080", "http://google.com"}
    err := broadcastMsg("hi", addr)

    time.Sleep(time.Second)

    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("everything went fine")
}
```

对于 broadcastMsg 里的这一段

```go
for _ = range addrs {
        if err := <-errc; err != nil {
            return err
        }
    }
```

当遇到 第一条不为 nil 的 err，broadcastMsg就返回了，那么从第二个调用 sendMsg 后返回值 err 不为 nil 的 goroutine 在 `errc <- sendMsg(msg, addr)` 这里都将阻塞而造成这些 goroutine 不能退出。

#### 3. 向已满的 buffered channel 写，但是没有读

和第二种情况比较类似。

在 channel 的接收值数量有限，且可以用 buffered channel 的情况下，那 buffer size 就分配的和 接收值数量 一样，这样可以解决掉第2、3种原因造成的泄露。比如在第二种中，改成

```go
errc := make(chan error, len(addrs))
```

问题就解决了。

注意：time package里的定时器使用不当也会造成 goroutine 泄露。

```go
  tick := time.Tick(1 * time.Second)
    for countdown := 10; countdown > 0; countdown-- {
        fmt.Println(countdown)
        select {
        case <-tick:
            // Do nothing.
        case <-abort:
            fmt.Println("aborted!")
            return
        }
    }
```

以上的代码中，当 for 循环结束后，tick 将不再有接收者，time.Tick 启动的 goroutine 将产生泄露。

建议在程序的整个生命周期需要 ticks 时才使用 time.Tick，否则建议按如下模式使用：

```go
    ticker := time.NewTicker(1 * time.Second)
    <- ticker.C 
    ticker.Stop() // 当不再使用后，结束 ticker 的 goroutine
```

#### 4. select操作在所有case上阻塞

实现一个 fibonacci 数列生成器，并在独立的 goroutine 中运行，在读取完需要长度的数列后，如果 用于 退出生成器的 quit 忘了被 close (或写入数据)，select 将一直被阻塞造成 该 goroutine 泄露。

```go
func fibonacci(c, quit chan int)  {
    x, y := 0, 1
    for{
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)

    go fibonacci(c, quit)

    for i := 0; i < 10; i++{
        fmt.Println(<- c)
    }
    
  // close(quit)
}
```

在这种需要一个独立的 goroutine 作为生成器的场景下，为了能在外部结束这个 goroutine，我们通常有两种方法：

- 使用上述实现里的模式，传入一个 quit channel，配合 select，当不需要的时候，close 这个 quit channel，该 goroutine 就可以退出。
- 使用 context 包：

```go
func fibonacci(c chan int, ctx context.Context)  {
    x, y := 0, 1
    for{
        select {
        case c <- x:
            x, y = y, x+y
        case <-ctx.Done():
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    c := make(chan int)

    go fibonacci(c, ctx)

    for i := 0; i < 10; i++{
        fmt.Println(<- c)
    }

    cancel()

    time.Sleep(5 * time.Second)
}
```

#### 5. goroutine进入死循环中，导致资源一直无法释放

通常由于代码里循环的退出条件实现的不对，导致死循环。

```go
// 粗暴的示例
func foo() {
  for{
    fmt.Println("fooo")
  }
}
```

### goroutine 泄露检测和定位

1. 监控工具：固定周期对进程的内存占用情况进行采样，数据可视化后，根据内存占用走势（持续上升），很容易发现是否发生内存泄露。可以使用云服务提供的内存使用监控服务或者自己实现一个 daemon 脚本周期采集内存占用数据。
2. 使用Go提供的pprof工具分析是否发生内存泄露。使用 pprof 的 heap 能够获取程序运行时的内存信息，通过对运行的程序多次采样对比，分析出内存的使用情况。

### goroutine 泄露的防范

- 创建goroutine时就要想好该goroutine该如何结束
- 使用channel时，要考虑到 channel 阻塞时协程可能的行为
- 实现循环语句时注意循环的退出条件，避免死循环