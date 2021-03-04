```go
package main

import (
	"fmt"
	"time"
)

func producer(out chan<- int)  {
	for i := 0; i < 10; i++ {
		out <- i // 被消费者sleep阻塞
		fmt.Println("生产 ", i)
	}
	close(out)
}

func consumer(in <- chan int)  {
	for num := range in {
		fmt.Println("消费者拿到 ", num)
		time.Sleep(time.Second) // 消费者sleep不从管道中拿数据，生产者不能存数据
	}
}

func main()  {
	// 公共缓冲区channel，channel可为有缓冲区，有缓冲区不会阻塞，为异步，可为无缓冲区，无缓冲区因为阻塞变为同步
	ch := make(chan int, 6) // 在输出时生产者大于6，因为生产者生产6个后，消费者已消费，但打印时造成延时，因为打印时有I/O操作
	// ch := make(chan int)

	go producer(ch) // 子go程，生产者

	consumer(ch) // 主go程，消费者
}
```

