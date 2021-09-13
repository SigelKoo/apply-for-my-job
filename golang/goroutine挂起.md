挂起27种原因

## 一

| 标识                      | 含义                             |
| ------------------------- | -------------------------------- |
| waitReasonZero            | 主要在sleep和lock的2个场景中使用 |
| waitReasonGCAssistMarking | GC辅助标记阶段会使得阻塞等待     |
| waitReasonIOWait          | IO阻塞等待时，网络请求等         |

## 二

| 标识                         | 含义                          |
| :--------------------------- | :---------------------------- |
| waitReasonChanReceiveNilChan | 对未初始化的channel进行读操作 |
| waitReasonChanSendNilChan    | 对未初始化的channel进行写操作 |

## 三

| 标识                            | 含义                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| waitReasonDumpingHeap           | 对Go Heap堆dump时，这个的使用场景仅在runtime.debug时，也就是常见的 pprof 这一类采集时阻塞 |
| waitReasonGarbageCollection     | 在垃圾回收时，主要场景是 GC 标记终止(GC Mark Termination)阶段时触发 |
| waitReasonGarbageCollectionScan | 在垃圾回收扫描时，主要场景是GC标记(GC Mark)扫描Root阶段时触发 |

## 四

| 标识                   | 含义                                                   |
| :--------------------- | :----------------------------------------------------- |
| waitReasonPanicWait    | 在main goroutine发生panic时，会触发                    |
| waitReasonSelect       | 在调用关键字select时会触发                             |
| waitReasonSelectNoCase | 在调用关键字 select 时，若一个 case 都没有，会直接触发 |

## 五

| 标识                     | 含义                                                         |
| :----------------------- | :----------------------------------------------------------- |
| waitReasonGCAssistWait   | GC辅助标记阶段中的结束行为，会触发                           |
| waitReasonGCSweepWait    | GC清扫阶段中的结束行为，会触发                               |
| waitReasonGCScavengeWait | GC scavenge阶段的结束行为，会触发。GC Scavenge主要是新空间的垃圾回收，是一种经常运行、快速的 GC，负责从新空间中清理较小的对象 |

## 六

| 标识                    | 含义                                                         |
| :---------------------- | :----------------------------------------------------------- |
| waitReasonChanReceive   | 在channel进行读操作，会触发                                  |
| waitReasonChanSend      | 在channel进行写操作，会触发                                  |
| waitReasonFinalizerWait | 在finalizer结束的阶段，会触发。在Go程序中，可以通过调用 runtime.SetFinalizer函数来为一个对象设置一个终结者函数。这个行为对应着结束阶段造成的回收。 |

## 七

| 标识                  | 含义                           |
| :-------------------- | :----------------------------- |
| waitReasonForceGCIdle | 强制GC(空闲时间)结束时，会触发 |
| waitReasonSemacquire  | 信号量处理结束时，会触发       |
| waitReasonSleep       | 经典的sleep行为，会触发        |

## 八

| 标识                         | 含义                                                         |
| :--------------------------- | :----------------------------------------------------------- |
| waitReasonSyncCondWait       | 结合sync.Cond用法能知道，是在调用sync.Wait方法时所触发       |
| waitReasonTimerGoroutineIdle | 与Timer相关，在没有定时器需要执行任务时，会触发              |
| waitReasonTraceReaderBlocked | 与Trace相关，ReadTrace会返回二进制跟踪数据，将会阻塞直到数据可用 |

## 九

| 标识                     | 含义                               |
| :----------------------- | :--------------------------------- |
| waitReasonWaitForGCCycle | 等待GC周期，会休眠造成阻塞         |
| waitReasonGCWorkerIdle   | GC Worker空闲时，会休眠造成阻塞    |
| waitReasonPreempted      | 发生循环调用抢占时，会休眠等待调度 |
| waitReasonDebugCall      | 调用GODEBUG时，会触发              |

今天这篇文章是对开头 runtime.gopark 函数的详解文章的一个补充，我们能够对此了解到其诱发的因素。

主要场景为：

- 通道(Channel)。
- 垃圾回收(GC)。
- 休眠(Sleep)。
- 锁等待(Lock)。
- 抢占(Preempted)。
- IO 阻塞(IO Wait)
- 其他，例如：panic、finalizer、select 等。

我们可以根据这些特性，去拆解可能会造成阻塞的原因。其实也就没必要记了，他们会导致阻塞肯定是由于存在影响控制流的因素，才会导致 gopark 的调用。