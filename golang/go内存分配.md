# go内存分配

## 在堆上分配

采用类似TCMalloc的分配策略将对象根据大小分类，并设计多层级的组件提高内存分配器的性能。

### 多层级组件

![go-memory-layout](https://img.draveness.me/2020-02-29-15829868066479-go-memory-layout.png)

Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件。

采用多层级cache来减少分配的冲突，per-P无锁的mcache、全局67*2个对应不同size的span的mcentral，全局1个的mheap。即从无锁到粒度低的锁，再到全局一个锁，再到系统调用分配堆。

#### 内存管理单元mspan

mspan结构体包含next和prev两个字段，指向前一个和后一个mspan

![mspan-and-linked-list](https://img.draveness.me/2020-02-29-15829868066485-mspan-and-linked-list.png)

因为相邻的管理单元会互相引用，所以我们可以从任意一个结构体访问双向链表中的其他节点

##### mspan字段详情

![mspan-and-objects](https://img.draveness.me/2020-02-29-15829868066499-mspan-and-objects.png)

- startAddr起始地址

- npages页数，startAddr和npages可以确定该结构体管理的多个页所在的内存，每个页的大小都是8KB

- freeindex空闲对象的初始索引的可能位置

- allocBits上一次GC后哪些slot被占用

- gcmarkBits上一次GC后哪些slot被回收

- allocCache是allocBits的补码，快速查找内存中未被使用的内存，从freeindex开始64个slot的分配情况

- mSpanState存储了内存管理单元的四种状态mSpanDead、mSpanInUse、mSpanManual和mSpanFree；在垃圾回收的任意阶段，可能从mSpanFree转换到mSpanInUse和mSpanManual；在垃圾回收的清除阶段，可能从mSpanInUse和mSpanManual转换到mSpanFree；在垃圾回收的标记阶段，不能从mSpanInUse和mSpanManual转换到mSpanFree

- spanClass跨度类，枚举67*2中span（每个开度类都存储class，slot的大小，页span的大小，对象数量，余数，最大浪费率）

  | class | bytes/obj | bytes/span | objects | tail waste | max waste |
  | :---: | --------: | ---------: | ------: | :--------: | :-------: |
  |   1   |         8 |       8192 |    1024 |     0      |  87.50%   |
  |   2   |        16 |       8192 |     512 |     0      |  43.75%   |
  |   3   |        24 |       8192 |     341 |     0      |  29.24%   |
  |   4   |        32 |       8192 |     256 |     0      |  46.88%   |
  |   5   |        48 |       8192 |     170 |     32     |  31.52%   |
  |   6   |        64 |       8192 |     128 |     0      |  23.44%   |
  |  ...  |       ... |        ... |     ... |    ...     |    ...    |

  例：class = 1， 每个slot大小为8byte，每个span有8192大小，可存1024个slot对象，余数为0，最大浪费率为87.5%

  之所以是67*2，区分指针和非指针两种，其中spanClass中noscan函数可以判断是否包含指针

#### 线程缓存mcache

与线程上处理器（GMP中的P）一一绑定，用来缓存用户程序申请的微小对象，每一个线程缓存都持有68*2个mspan；线程缓存在刚刚被初始化时是不包含mspan的，只有当用户程序申请内存时才会从上一级组件获取新的 runtime.mspan满足内存分配的需求。

##### 微分配器

线程缓存中还包含几个用于分配微对象的字段tiny、tinyoffset、local_tinyallocs，专门管理 16 字节以下的对象

#### 中心缓存mcentral

与线程缓存不同，访问中心缓存中的内存管理单元需要使用互斥锁。每个中心缓存都会管理某个跨度类的内存管理单元，它会同时持有两个runtime.spanSet，分别存储包含空闲对象和不包含空闲对象的内存管理单元。

#### 页堆mheap

是内存分配的核心结构体，Go 语言程序会将其作为全局变量存储，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表central，另一个是管理堆区内存区域的arenas以及相关字段。

页堆中包含一个长度为68*2的runtime.mcentral数组，其中68个为跨度类需要scan的中心缓存，另外的68个是noscan的中心缓存

### 大小分类

![allocator-and-memory-size](https://img.draveness.me/2020-02-29-15829868066537-allocator-and-memory-size.png)

- 微对象 `(0, 16B)` — 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；
- 小对象 `[16B, 32KB]` — 依次尝试使用线程缓存、中心缓存和堆分配内存；
- 大对象 `(32KB, +∞)` — 直接在堆上分配内存；

微对象使用线程缓存上的微分配器提高微对象分配的性能。微分配器可以将多个较小的内存分配请求合入同一个内存块中，只有当内存块中的所有对象都需要被回收时，整片内存才可能被回收。

小对象，首先确定分配对象的大小以及跨度类runtime.spanClass；从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；调用runtime.memclrNoHeapPointers清空空闲内存中的所有数据；

大对象，运行时对于大于32KB的大对象会单独处理，我们不会从线程缓存或者中心缓存中获取内存管理单元，而是直接调用runtime.mcache.allocLarge分配大片内存

## 在栈上分配

在栈上分配也是多层次和多类别的

