前面已经介绍了从Go程序执行后的第一条指令，到启动runtime.main的主要流程，比如其中要设置好本地线程存储，设置好main函数参数，根据环境变量GOMAXPROCS设置好使用的procs，初始化调度器和内存管理等等。

接下来将是从runtime.main到main.main之间的一些过程。注意，main.main是在runtime.main函数里面调用的。不过在调用main.main之前，还有一些工作要做。

newm新建一个结构体M，第一个参数是这个结构体M的入口函数，也就说会在一个新的物理线程中运行sysmon函数。由此可见sysmon是一个地位非常高的后台任务，整个函数体一个死循环的形式，目前主要处理两个事件：对于网络的epoll以及抢占式调度的检测。大致过程如下：

sysmon会根据系统当前的繁忙程度睡一小段时间，然后每隔10ms至少进行一次epoll并唤醒相应的goroutine。同时，它还会检测是否有P长时间处于Psyscall状态或Prunning状态，并进行抢占式调度。

newproc创建一个goroutine，第一个参数是goroutine运行的函数。scavenger的地位是没有sysmon那么高的——sysmon是由物理线程运行的，而scavenger只是由goroutine运行的。接下来的章节会说明goroutine与物理线程的区别。

那么，scavenger执行什么工作？它又为什么不像sysmon那样呢？其实scavenger执行的是runtime·MHeap_Scavenger函数。它将一些不再使用的内存归还给操作系统。Go是一门的语言，垃圾回收会在系统运行过程中被触发，内存会被归还到Go的内存管理系统中，Go的内存管理是基于内存池进行重用的，而这个函数会真正地将内存归还给操作系统。

main.main在这些后台任务运行起来之后执行，不过在它执行之前，还有最后一个：main.init，每个包的init函数会在包使用之前先执行。