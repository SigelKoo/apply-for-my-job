## channel读写特性

### 給一个nil channel发送数据，造成永远阻塞

```go
var channel chan string
channel <- "test"
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
```

**原因**

src/runtime/chan.go

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
      // 不能阻塞，直接返回 false，表示未发送成功
      if !block {
        return false
      }
      gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
      throw("unreachable")
    }
  // 省略其他逻辑
}
```

未初始化的`chan`此时是等于`nil`，当它不能阻塞的情况下，直接返回 `false`，表示写 `chan` 失败（使用select就是非阻塞的）

当`chan`能阻塞的情况下，则直接阻塞 `gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)`，然后调用`throw(s string)`抛出错误,其中`waitReasonChanSendNilChan`就是刚刚提到的报错`"chan send (nil chan)"`

### 从一个nil channel接收数据，造成永远阻塞

```go
var channel chan string
a, ok := <- channel
fmt.Println(a, ok)
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
```

**原因**

src/runtime/chan.go

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    //省略逻辑...
    if c == nil {
        if !block {
          return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    //省略逻辑...
} 
```

未初始化的`chan`此时是等于`nil`，当它不能阻塞的情况下，直接返回 `false`，表示读 `chan` 失败

当`chan`能阻塞的情况下，则直接阻塞 `gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)`, 然后调用`throw(s string)`抛出错误,其中`waitReasonChanReceiveNilChan`就是刚刚提到的报错`"chan receive (nil chan)"`

### 关闭一个nil channel，引起panic

```go
var channel chan string
close(channel)
```

```
panic: close of nil channel

goroutine 1 [running]:
main.main()
```

### 給一个已经关闭的channel发送数据，引起panic

```go
var channel chan string
channel = make(chan string, 6)
close(channel)
channel <- "test"
```

```
panic: send on closed channel

goroutine 1 [running]:
main.main()
```

### 从一个已经关闭的channel接收数据，如果缓冲区中为空，则返回一个零值

```go
var channel chan int
channel = make(chan int, 6)
channel <- 1
channel <- 2
channel <- 3
channel <- 4
channel <- 5
channel <- 6
close(channel)
a := <- channel
b := <- channel
c := <- channel
d := <- channel
e := <- channel
f := <- channel
g := <- channel
h := <- channel
fmt.Println(a, b, c, d, e, f, g, h)
```

```
1 2 3 4 5 6 0 0
```

### 无缓冲的channel是同步的，而有缓冲的channel是非同步的

## 区别

### 读操作

| channel状态 | 结果            |
| ----------- | --------------- |
| nil         | 阻塞            |
| 打开且非空  | 输出值          |
| 打开但空    | 阻塞            |
| 关闭的      | <默认值>, false |
| 只写        | 编译错误        |

### 写操作

| channel状态 | 结果     |
| ----------- | -------- |
| nil         | 阻塞     |
| 打开但填满  | 阻塞     |
| 打开且不满  | 写入值   |
| 关闭的      | panic    |
| 只写        | 编译错误 |

### close操作

| channel状态 | 结果                                                        |
| ----------- | ----------------------------------------------------------- |
| nil         | panic                                                       |
| 打开且非空  | 关闭channel，读取成功，直到通道耗尽，然后读取产生值的默认值 |
| 打开但空    | 关闭channel，读到生产者的默认值                             |
| 关闭的      | panic                                                       |
| 只写        | 编译错误                                                    |