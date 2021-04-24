

Go语言的内存模型规定了一个goroutine可以看到另外一个goroutine修改同一个变量的值的条件

当多个goroutine并发同时存取同一个数据时候必须把并发的存取的操作顺序化，在go中可以实现操作顺序化的工具有高级的通道（channel）通信和同步原语比如sync包中的Mutex(互斥锁)、RWMutex(读写锁)或者和sync/atomic中的原子操作。

在单goroutine中，读取和写入的行为一定是和程序指定的执行顺序表现一致。换言之，编译器和处理器在不改变语言规范所定义的行为前提下才可以对单个goroutine中的指令进行重排序。

```text
a := 1
b := 2
```

由于指令重排序，`b := 2`可能先于`a := 1`执行。单goroutine中，该执行顺序的调整并不会影响最终结果。但多个goroutine场景下可能就会出现问题。

```text
var a, b int
// goroutine A
go func() {
    a := 5
    b := 1
}()
// goroutine B
go func() {
    for b == 1 {}
    fmt.Println(a)
}()
```

执行上述代码时，预期goroutine B能够正常输出5，但因为指令重排序，`b := 1`可能先于`a := 5`执行，最终goroutine B可能输出0。

### Happen Before原则

**1) 单线程**

**2) Init 函数**

如果包P1中导入了包P2，则P2中的init函数Happens Before 所有P1中的操作
main函数Happens After 所有的init函数
**3) Goroutine**

Goroutine的创建Happens Before所有此Goroutine中的操作
Goroutine的销毁Happens After所有此Goroutine中的操作
**4) Channel**

对一个元素的send操作Happens Before对应的receive 完成操作 , [先发后接]
对channel的close操作Happens Before receive 端的收到关闭通知操作 [先关后接,接到零值]
对于无缓冲channel(unbuffered Channel)，对一个元素的receive 操作Happens Before对应的send完成操作 [先接后发]
对于Buffered Channel，假设Channel 的buffer 大小为C，那么对第k个元素的receive操作，Happens Before第k+C个send完成操作。可以看出上一条Unbuffered Channel规则就是这条规则C=0时的特例 [先接后发]
**5) Lock**

Go里面有Mutex和RWMutex两种锁，RWMutex除了支持互斥的Lock/Unlock，还支持共享的RLock/RUnlock。

对于一个Mutex/RWMutex，设n < m，则第n个Unlock操作Happens Before第m个Lock操作。
对于一个RWMutex，存在数值n，RLock操作Happens After 第n个UnLock，其对应的RUnLock Happens Before 第n+1个Lock操作。
简单理解就是这一次的Lock总是Happens After上一次的Unlock，读写锁的RLock HappensAfter上一次的UnLock，其对应的RUnlock Happens Before 下一次的Lock。