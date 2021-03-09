# go语言channel的实现原理

channel是Golang在语言层面提供的goroutine间的通信方式，比Unix管道更易用也更轻便。channel主要用于进程内各goroutine间通信。

### chan数据结构

`src/runtime/chan.go:hchan`定义了channel的数据结构：

```go
type hchan struct {
	qcount   uint           // 当前队列里还剩余元素个数
	dataqsiz uint           // 环形队列长度，即缓冲区的大小，即make(chan T,N)中的N，即可以存放的元素个数
	buf      unsafe.Pointer // 环形队列指针
	elemsize uint16         // 每个元素的大小
	closed   uint32	        // 标识当前通道是否处于关闭状态，创建通道后，该字段设置0，即打开通道；通道调用close将其设置为1，通道关闭
	elemtype *_type         // 元素类型，用于数据传递过程中的赋值
	sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
	recvx    uint           // 队列下标，指示元素从队列的该位置读出
	recvq    waitq          // 等待读消息的goroutine队列
	sendq    waitq          // 等待写消息的goroutine队列
	lock mutex              // 互斥锁，为每个读写操作锁定通道，因为发送和接受必须是互斥操作
}
```

channel由buf环形队列、类型信息、recvx goroutine等待队列和recvq等待队列组成。

#### 环形队列

chan内部实现了一个环形队列作为其缓冲区，队列的长度是创建chan时指定的。

可缓存6个元素的channel：

![img](https://oscimg.oschina.net/oscnet/f1ae952fd1c62186d4bd0eb3fa1610db67a.jpg)

- dataqsiz指示了队列长度为6，即可缓存6个元素
- buf指向队列的内存，队列中还剩余两个元素
- qcount表示队列中还可以存两个元素
- sendx指示后续写入的数据存储的位置，取值[0, 6)
- recvx指示从该位置读取数据, 取值[0, 6)

#### 等待队列

从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞。

向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞。

被阻塞的goroutine将会挂在channel的等待队列waitq中：

- 因读阻塞的goroutine会被向channel写入数据的goroutine唤醒；
- 因写阻塞的goroutine会被从channel读数据的goroutine唤醒；

一个没有缓冲区的channel，有几个goroutine阻塞等待读数据：

![img](https://oscimg.oschina.net/oscnet/51d91ed6fb42117d5035cab82b283bf0b07.jpg)

**注意，一般情况下recvq和sendq至少有一个为空。只有一个例外，那就是同一个goroutine使用select语句向channel一边写数据，一边读数据。**

#### 类型信息

一个channel只能传递一种类型的值，类型信息存储在hchan数据结构中。

- elemtype代表类型，用于数据传递过程中的赋值
- elemsize代表类型大小，用于在buf中定位元素位置

#### 锁

一个channel同时仅允许被一个goroutine读写

#### recvq和sendq

recvq和sendq基本上是链表，基本如下：

![img](https://upload-images.jianshu.io/upload_images/17528797-bfc757dcc729f5ca.png?imageMogr2/auto-orient/strip|imageView2/2/w/741/format/webp)

### channel读写

#### 创建channel

创建channel的过程实际上是初始化hchan结构。其中类型信息和缓冲区长度由make语句传入，buf的大小则与元素大小和缓冲区长度共同决定。

```go
func makechan(t *chantype, size int) *hchan {
	var c *hchan
	c = new(hchan)
	c.buf = malloc(元素类型大小*size)
	c.elemsize = 元素类型大小
	c.elemtype = 元素类型
	c.dataqsiz = size

	return c
}
```

#### 向channel写数据

1. 锁定channel
2. 确定写入。recvq从等待队列中等待goroutine，然后将元素直接写入goroutine
3. 如果recvq为Empty，则确定缓冲区是否可用，如果可用那么从当前goroutine复制数据到缓冲区中
4. 如果缓冲区已经满了，则要写入的元素将保存在当前执行的goroutine结构中，并且当前goroutine在sendq中并且队列从运行时挂起
5. 写入完成释放锁

![img](https://upload-images.jianshu.io/upload_images/17528797-f02b0dbfd629ad6b.png?imageMogr2/auto-orient/strip|imageView2/2/w/609/format/webp)

#### 从channel读数据

1. 先获取channel全局锁
2. 尝试sendq等待队列中获取等待的goroutine
3. 如果有等待的goroutine，没有缓冲区，取出goroutine并读取数据，然后唤醒这个goroutine，结束读取释放锁
4. 如果有等待goroutine，且有缓冲区(缓冲区满了)，从缓冲区队列首取数据，再从sendq取出一个goroutine，将goroutine中的数据存放到buf队列尾，结束读取释放锁
5. 如果没有等待的goroutine，且缓冲区有数据，直接读取缓冲区数据，结束释放锁
6. 如果没有等待的goroutine，且没有缓冲区或者缓冲区为空，将当前goroutine加入到sendq队列，进入睡眠，等待被写入goroutine唤醒，结束读取释放锁

![img](https://upload-images.jianshu.io/upload_images/17528797-b3e7d9f3bcc9a526.png?imageMogr2/auto-orient/strip|imageView2/2/w/730/format/webp)

#### 关闭channel

关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。

除此之外，panic出现的常见场景还有：

1. 关闭值为nil的channel
2. 关闭已经被关闭的channel
3. 向已经关闭的channel写数据

### 用法

#### 单向channel

- func readChan(chanName <-chan int)： 通过形参限定函数内部只能从channel中读取数据
- func writeChan(chanName chan<- int)： 通过形参限定函数内部只能向channel中写入数据

```go
func readChan(chanName <-chan int) {
    <- chanName
}

func writeChan(chanName chan<- int) {
    chanName <- 1
}

func main() {
    var mychan = make(chan int, 10)

    writeChan(mychan)
    readChan(mychan)
}
```

mychan是个正常的channel，而readChan()参数限制了传入的channel只能用来读，writeChan()参数限制了传入的channel只能用来写。

#### select

使用select可以监控多channel。比如监控多个channel，当其中某一个channel有数据时，就从其读出数据。

```go
package main

import (
    "fmt"
    "time"
)

func addNumberToChan(chanName chan int) {
    for {
        chanName <- 1
        time.Sleep(1 * time.Second)
    }
}

func main() {
    var chan1 = make(chan int, 10)
    var chan2 = make(chan int, 10)

    go addNumberToChan(chan1)
    go addNumberToChan(chan2)

    for {
        select {
        case e := <- chan1 :
            fmt.Printf("Get element from chan1: %d\n", e)
        case e := <- chan2 :
            fmt.Printf("Get element from chan2: %d\n", e)
        default:
            fmt.Printf("No element in chan1 and chan2.\n")
            time.Sleep(1 * time.Second)
        }
    }
}
```

程序中创建两个channel： chan1和chan2。函数addNumberToChan()函数会向两个channel中周期性写入数据。通过select可以监控两个channel，任意一个可读时就从其中读出数据。

程序输出

```
D:\SourceCode\GoExpert\src>go run main.go
Get element from chan1: 1
Get element from chan2: 1
No element in chan1 and chan2.
Get element from chan2: 1
Get element from chan1: 1
No element in chan1 and chan2.
Get element from chan2: 1
Get element from chan1: 1
No element in chan1 and chan2.
```

从输出可见，从channel中读出数据的顺序是随机的，事实上select语句的多个case执行顺序是随机的。

**select的case语句读channel不会阻塞，尽管channel中没有数据。这是由于case语句编译后调用读channel时会明确传入不阻塞的参数，此时读不到数据时不会将当前goroutine加入到等待队列，而是直接返回。**

#### range

通过range可以持续从channel中读出数据，好像在遍历一个数组一样，当channel中没有数据时会阻塞当前goroutine，与读channel时阻塞处理机制一样。

```go
func chanRange(chanName chan int) {
    for e := range chanName {
        fmt.Printf("Get element from chan: %d\n", e)
    }
}
```

