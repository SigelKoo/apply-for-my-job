socket是一种”打开—读/写—关闭”模式的实现，服务器和客户端各自维护一个”文件”，在建立连接打开后，可以向自己文件写入内容供对方读取或者读取对方内容，通讯结束时关闭文件。

Socket 是实现“打开–读/写–关闭”这样的模式，以使用 TCP 协议通讯的 socket 为例。如下图所示：

![img](https://segmentfault.com/img/remote/1460000022734663)

## 加上epoll的流程

```c
int s = socket(AF_INET, SOCK_STREAM, 0);   
bind(s, ...)
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1){
    int n = epoll_wait(...)
    for(接收到数据的socket){
        //处理
    }
}
```

## socket错误码

ETIMEOUT：110
1、操作超时。一般设置了发送接收超时，遇到网络繁忙的情况，就会遇到这种错误。
2、服务器做了读数据做了超时限制，读时发生了超时。
3、错误被描述为“connect time out”，即“连接超时”，这种情况一般发生在服务器主机崩溃。此时客户 TCP 将在一定时间内（依具体实现）持续重发数据分节，试图从服务 TCP 获得一个 ACK 分节。当最终放弃尝试后（此时服务器未重新启动），内核将会向客户进程返回 ETIMEDOUT 错误。如果某个中间路由器判定该服务器主机已经不可达，则一般会响应“destination unreachable”－“目的地不可达”的ICMP消息，相应的客户进程返回的错误是 EHOSTUNREACH 或ENETUNREACH。当服务器重新启动后，由于 TCP 状态丢失，之前所有的连接信息也不存在了，此时对于客户端发来请求将回应 RST。如果客户进程对检测服务器主机是否崩溃很有必要，要求即使客户进程不主动发送数据也能检测出来，那么需要使用其它技术，如配置 SO_KEEPALIVE Socket 选项，或实现某些心跳函数。
————————————————
原文链接：https://blog.csdn.net/bytxl/article/details/40855917

## 创建子进程实现父子进程监听相同的端口

方法：在绑定端口号（bind函数）之后，监听端口号之前（listen函数），用fork（）函数生成子进程，这样子进程就可以克隆父进程，达到监听同一个端口的目的。

## server

```go
package main

import (
    "bufio"
    "fmt"
    "net"
)

func process(conn net.Conn) {
    // 处理完关闭连接
    defer conn.Close()

    // 针对当前连接做发送和接受操作
    for {
        reader := bufio.NewReader(conn)
        var buf [128]byte
        n, err := reader.Read(buf[:])
        if err != nil {
            fmt.Printf("read from conn failed, err:%v\n", err)
            break
        }

        recv := string(buf[:n])
        fmt.Printf("收到的数据：%v\n", recv)

        // 将接受到的数据返回给客户端
        _, err = conn.Write([]byte("ok"))
        if err != nil {
            fmt.Printf("write from conn failed, err:%v\n", err)
            break
        }
    }
}

func main() {
    // 建立 tcp 服务
    listen, err := net.Listen("tcp", "127.0.0.1:9090")
    if err != nil {
        fmt.Printf("listen failed, err:%v\n", err)
        return
    }

    for {
        // 等待客户端建立连接
        conn, err := listen.Accept()
        if err != nil {
            fmt.Printf("accept failed, err:%v\n", err)
            continue
        }
        // 启动一个单独的 goroutine 去处理连接
        go process(conn)
    }
}
```

## client

```go
package main

import (
    "bufio"
    "fmt"
    "net"
    "os"
    "strings"
)

func main() {
    // 1、与服务端建立连接
    conn, err := net.Dial("tcp", "127.0.0.1:9090")
    if err != nil {
        fmt.Printf("conn server failed, err:%v\n", err)
        return
    }
    // 2、使用 conn 连接进行数据的发送和接收
    input := bufio.NewReader(os.Stdin)
    for {
        s, _ := input.ReadString('\n')
        s = strings.TrimSpace(s)
        if strings.ToUpper(s) == "Q" {
            return
        }

        _, err = conn.Write([]byte(s))
        if err != nil {
            fmt.Printf("send failed, err:%v\n", err)
            return
        }
        // 从服务端接收回复消息
        var buf [1024]byte
        n, err := conn.Read(buf[:])
        if err != nil {
            fmt.Printf("read failed:%v\n", err)
            return
        }
        fmt.Printf("收到服务端回复:%v\n", string(buf[:n]))
    }
}
```

# 与三次握手对应

客户端调用connect的时候，就是发一个syn

服务端accept的时候，实际上是从内核的accept队列里面取一个连接，如果这个队列为空，则进程阻塞（阻塞模式下）。如果accept返回则说明成功取到一个连接，返回到应用层。

大致的过程是客户端发一个syn之后，服务端将这个连接放入到backlog队列，在收到客户端的ack之后将这个请求移到accept队列。

所以accept一定是发生在三次握手之后，connect只是发一个syn而已

![img](https://img2018.cnblogs.com/blog/1169746/201810/1169746-20181001231741928-751354816.png)

