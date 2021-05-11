# LevelDB概述

LevelDB是Google开源的持久化KV单机数据库，具有很高的随机写，顺序读/写性能，但是随机读的性能很一般，也就是说，**LevelDB很适合应用在查询较少，而写很多的场景**。LevelDB应用了LSM(Log Structured Merge) 策略，lsm_tree对索引变更进行延迟及批量处理，并通过一种类似于归并排序的方式高效地将更新迁移到磁盘，降低索引插入开销。

## 特点

1、key和value都是任意长度的字节数组；
2、entry（即一条K-V记录）默认是按照key的字典顺序存储的，当然开发者也可以重载这个排序函数；
3、提供的基本操作接口：Put()、Delete()、Get()、Batch()；
4、支持批量操作以原子操作进行；
5、可以创建数据全景的snapshot(快照)，并允许在快照中查找数据；
6、可以通过前向（或后向）迭代器遍历数据（迭代器会隐含的创建一个snapshot）；
7、自动使用Snappy压缩数据；
8、可移植性；

## 限制

1、非关系型数据模型（NoSQL），不支持sql语句，也不支持索引；
2、一次只允许一个进程访问一个特定的数据库；
3、没有内置的C/S架构，但开发者可以使用LevelDB库自己封装一个server；

# LevelDB框架

![img](https://upload-images.jianshu.io/upload_images/1190574-ae111bd8ee0844f9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1029/format/webp)

上图简单展示了 LevelDB 的整体架构。LevelDB 的静态结构主要由六个部分组成：
**MemTable(wTable)**：内存数据结构，具体实现是 SkipList。 接受用户的读写请求，新的数据修改会首先在这里写入。
**Immutable MemTable(rTable)**：当 MemTable 的大小达到设定的阈值时，会变成 Immutable MemTable，只接受读操作，不再接受写操作，后续由后台线程 Flush 到磁盘上。
**SST Files**：Sorted String Table Files，磁盘数据存储文件。分为 Level0 到 LevelN 多层，每一层包含多个 SST 文件，文件内数据有序。Level0 直接由 Immutable Memtable Flush 得到，其它每一层的数据由上一层进行 Compaction 得到。
**Manifest Files**：Manifest 文件中记录 SST 文件在不同 Level 的分布，单个 SST 文件的最大、最小 key，以及其他一些 LevelDB 需要的元信息。由于 LevelDB 支持 snapshot，需要维护多版本，因此可能同时存在多个 Manifest 文件。
**Current File**：由于 Manifest 文件可能存在多个，Current 记录的是当前的 Manifest 文件名。
**Log Files (WAL)**：用于防止 MemTable 丢数据的日志文件。

**粗箭头表示写入数据的流动方向：**
 (1) 先写入 MemTable。
 (2) MemTable 的大小达到设定阈值的时候，转换成 Immutable MemTable。
 (3) Immutable Table 由后台线程异步 Flush 到磁盘上，成为 Level0 上的一个 sst 文件。
 (4) 在某些条件下，会触发后台线程对 Level0 ~ LevelN 的文件进行 Compaction。

**读操作的流动方向和写操作类似：**
 (1) 读 MemTable，如果存在，返回。
 (2) 读 Immutable MemTable，如果存在，返回。
 (3) 按顺序读 Level0 ~ Leveln，如果存在，返回。
 (4) 返回不存在。

![img](https://upload-images.jianshu.io/upload_images/1190574-4721b60b19fcf827.png?imageMogr2/auto-orient/strip|imageView2/2/w/716/format/webp)

**全程的操作流程如下描述：**
 MemTable(wTable)的大小达到一个阈值，LevelDB 将它凝固成只读的Immutable MemTable(rTable)，同时生成一个新的 wtable 继续接受写操作。rtable 将会被异步线程刷到磁盘中。Get 操作会优先查询 wtable，如果找不到就去 rtable 中去找，rtable 如果还找不到，再去磁盘文件里去找。

因为 wtable 要支持多线程读写，所以访问它是需要加锁控制。而 rtable 是只读的，它就不需要，但是它的存在时间很短，rtable 一旦生成，很快就会被异步线程序列化到磁盘上，然后就会被置空。但是异步线程序列化也需要耗费一定的时间，如果 wtable 增长过快，很快就被写满了，这时候 rtable 还没有完成序列化，而wtable 急需变身怎么办？这时写线程就会阻塞等待异步线程序列化完成，这是 LevelDB 的卡顿点之一，也是未来 RocksDB 的优化点。

图中还有个日志文件，记录了近期的写操作日志。如果 LevelDB 遇到突发停机事故，没有持久化的 wtable 和 rtable 数据就会丢失。这时就必须通过重放日志文件中的指令数据来恢复丢失的数据。注意到日志文件也是有两份的，它和内存的跳跃列表正好对应起来。当 wtable 要变身时，日志文件也会跟着变身。待 rtable 落盘成功之后，只读日志文件就可以被删除了。

# LevelDB基础部件

## 跳跃表SkipList

![img](https://upload-images.jianshu.io/upload_images/1190574-c91a38a0df9ed8ad.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp)

LevelDB 的内存中维护了 2 个跳跃列表，一个是只读的Immutable MemTable(rTable) ，一个是可修改的MemTable(wTable)。简单理解，跳跃列表就是一个 Key 有序的 Set 集合，排序规则由全局的「比较器」决定，默认是字典序。跳跃列表的查找和更新操作时间复杂度都是 Log(n)。

## SSTable

LevelDB 的键值对内容都存储在扩展名为 sst 的 SSTable 文件中。

![img](https://upload-images.jianshu.io/upload_images/1190574-100db1cbc187260f.png?imageMogr2/auto-orient/strip|imageView2/2/w/864/format/webp)

SSTable 文件的内容分为 5 个部分，Footer、IndexBlock、MetaIndexBlock、FilterBlock 和 DataBlock。其中存储了键值对内容的就是 DataBlock，存储了布隆过滤器二进制数据的是 FilterBlock，DataBlock 有多个，FilterBlock 也可以有多个，但是通常最多只有 1 个，之所以设计成多个是考虑到扩展性，也许未来会支持其它类型的过滤器。另外 3 个部分为管理块，其中 IndexBlock 记录了 DataBlock 相关的元信息，MetaIndexBlock 记录了过滤器相关的元信息，而 Footer 则指出 IndexBlock 和 MetaIndexBlock 在文件中的偏移量信息，它是元信息的元信息，它位于 sstable 文件的尾部。

### Footer 结构

它的占用空间很小只有 48 字节，内部只存了几个字段。下面我们用伪代码来描述一下它的结构

```c++
// 定义了数据块的位置和大小
struct BlockHandler {  
	varint offset; 
	varint size;
}
struct Footer {  
	BlockHandler metaIndexHandler;  // MetaIndexBlock的文件偏移量和长度  
	BlockHandler indexHandler; // IndexBlock的文件偏移量和长度  
	byte[n] padding;  // 内存垫片  
	int32 magicHighBits;  // 魔数后32位  
	int32 magicLowBits; // 魔数前32位
}
```

Footer 结构的中间部分增加了内存垫片，其作用就是将 Footer 的空间撑到 48 字节。结构的尾部还有一个 64位的魔术数字 0xdb4775248b80fb57，如果文件尾部的 8 字节不是这个数字说明文件已经损坏。这个魔术数字的来源很有意思，它是下面返回的字符串的前64bit。

```
$ echo http://code.google.com/p/leveldb/ | sha1sumdb4775248b80fb57d0ce0768d85bcee39c230b61
```

IndexBlock 和 MetaIndexBlock 都只有唯一的一个，所以分别使用一个 BlockHandler 结构来存储偏移量和长度。

### Block 结构

除了 Footer 之外，其它部分都是 Block 结构，在名称上也都是以 Block 结尾。所谓的 Block 结构是指除了内部的有效数据外，还会有额外的压缩类型字段和校验码字段。

```c++
struct Block { byte[] data;  int8 compressType;  int32 crcValue;}
```

每一个 Block 尾部都会有压缩类型和循环冗余校验码（crcValue），这会要占去 5 字节。如果是压缩类型，块内的数据 data 会被压缩。校验码会针对压缩和的数据和压缩类型字段一起计算循环冗余校验和。压缩算法默认是 snappy ，校验算法是 crc32。

```c++
crcValue = crc32(data, compressType)
```

在下面介绍的所有 Block 结构中，我们不再提及压缩和校验码。

### DataBlock 结构

DataBlock 的大小默认是 4K 字节（压缩前），里面存储了一系列键值对。前面提到 sst 文件里面的 Key 是有序的，这意味着相邻的 Key 会有很大的概率有共同的前缀部分。正是考虑到这一点，DataBlock 在结构上做了优化，这个优化可以显著减少存储空间。

Key 会划分为两个部分，一个是 sharedKey，一个是 unsharedKey。前者表示相对基准 Key 的共同前缀内容，后者表示相对基准 Key 的不同后缀部分。

![img](https://upload-images.jianshu.io/upload_images/1190574-d61b9c454e4ea4f0?imageMogr2/auto-orient/strip|imageView2/2/w/503/format/webp)

比如基准 Key 是 helloworld，那么 hellouniverse 这个 Key 相对于基准 Key 来说，它的 sharedKey 就是 hello，unsharedKey 就是 universe。

![img](https://upload-images.jianshu.io/upload_images/1190574-ca59d38a65e506d8?imageMogr2/auto-orient/strip|imageView2/2/w/804/format/webp)

DataBlock 中存储的是连续的一系列键值对，它会每隔若干个 Key 设置一个基准 Key。基准 Key 的特点就是它的 sharedKey 部分是空串。基准 Key 的位置，也就是它在块中的偏移量我们称之为「重启点」RestartPoint，在 DataBlock 中会记录所有「重启点」位置。第一个「重启点」的位置是零，也就是 DataBlock 中的第一个 Key。

```c++
struct Entry { 
    varint sharedKeyLength;  
	varint unsharedKeyLength;  
	varint valueLength;  
	byte[] unsharedKeyContent;  
	byte[] valueContent;
}
struct DataBlock {  
	Entry[] entries; 
	int32 [] restartPointOffsets;  
	int32 restartPointCount;
}
```

DataBlock 中基准 Key 是默认每隔 16 个 Key 设置一个。从节省空间的角度来说，这并不是一个智能的策略。比如连续 26 个 Key 仅仅是最后一个字母不同，DataBlock 却每隔 16 个 Key 强制「重启」，这明显不是最优的。这同时也意味着 sharedKey 是空串的 Key 未必就是基准 Key。

一个 DataBlock 的默认大小只有 4K 字节，所以里面包含的键值对数量通常只有几十个。如果单个键值对的内容太大一个 DataBlock 装不下咋整？

这里就必须纠正一下，DataBlock 的大小是 4K 字节，并不是说它的严格大小，而是在追加完最后一条记录之后发现超出了 4K 字节，这时就会再开启一个 DataBlock。这意味着一个 DataBlock 可以大于 4K 字节，如果 value 值非常大，那么相应的 DataBlock 也会非常大。DataBlock 并不会将同一个 Value 值分块存储。

### FilterBlock 结构

如果没有开启布隆过滤器，FilterBlock 这个块就是不存在的。FilterBlock 在一个 SSTable 文件中可以存在多个，每个块存放一个过滤器数据。不过就目前 LevelDB 的实现来说它最多只能有一个过滤器，那就是布隆过滤器。

布隆过滤器用于加快 SSTable 磁盘文件的 Key 定位效率。如果没有布隆过滤器，它需要对 SSTable 进行二分查找，Key 如果不在里面，就需要进行多次 IO 读才能确定，查完了才发现原来是一场空。布隆过滤器的作用就是避免在 Key 不存在的时候浪费 IO 操作。通过查询布隆过滤器可以一次性知道 Key 有没有可能在里面。

![img](https://upload-images.jianshu.io/upload_images/1190574-4dee96ad21dcf377?imageMogr2/auto-orient/strip|imageView2/2/w/431/format/webp)

单个布隆过滤器中存放的是一个定长的位图数组，该位图数组中存放了若干个 Key 的指纹信息。这若干个 Key 来源于 DataBlock 中连续的一个范围。FilterBlock 块中存在多个连续的布隆过滤器位图数组，每个数组负责指纹化 SSTable 中的一部分数据。

```c++
struct FilterEntry {  
	byte[] rawbits;
}
struct FilterBlock {  
	FilterEntry[n] filterEntries;  
	int32[n] filterEntryOffsets;  
	int32 offsetArrayOffset;  
	int8 baseLg;  // 分割系数
}
```

其中 baseLg 默认 11，表示每隔 2K 字节（2<<11）的 DataBlock 数据（压缩后），就开启一个布隆过滤器来容纳这一段数据中 Key 值的指纹。如果某个 Value 值过大，以至于超出了 2K 字节，那么相应的布隆过滤器里面就只有 1 个 Key 值的指纹。每个 Key 对应的指纹空间在打开数据库时指定。

```
options.SetFilterPolicy(levigo.NewBloomFilter(10))
```

这里的 2K 字节的间隔是严格的间隔，这样才可以通过 DataBlock 的偏移量和大小来快速定位到相应的布隆过滤器的位置 FilterOffset，再进一步获得相应的布隆过滤器位图数据。

### MetaIndexBlock 结构

MetaIndexBlock 存储了前面一系列 FilterBlock 的元信息，它在结构上和 DataBlock 是一样的，只不过里面 Entry 存储的 Key 是带固定前缀的过滤器名称，Value 是对应的 FilterBlock 在文件中的偏移量和长度。

```c++
key = "filter." + filterName// value 定义了数据块的位置和大小
struct BlockHandler {  varint offset;  varint size;}
```

就目前的 LevelDB，这里面最多只有一个 Entry，那么它的结构非常简单，如下图所示

![img](https://upload-images.jianshu.io/upload_images/1190574-28850710a44ee837?imageMogr2/auto-orient/strip|imageView2/2/w/684/format/webp)

### IndexBlock 结构

它和 MetaIndexBlock 结构一样，也存储了一系列键值对，每一个键值对存储的是 DataBlock 的元信息，SSTable 中有几个 DataBlock，IndexBlock 中就有几个键值对。键值对的 Key 是对应 DataBlock 内部最大的 Key，Value 是 DataBlock 的偏移量和长度。不考虑 Key 之间的前缀共享，不考虑「重启点」，它的结构如下图所示

![img](https://upload-images.jianshu.io/upload_images/1190574-79b8d9aa09f80e40?imageMogr2/auto-orient/strip|imageView2/2/w/655/format/webp)

## 日志文件(LOG)

"LOG文件在LevelDb中的主要作用是系统故障恢复时，能够保证不会丢失数据。因为在将记录写入内存的Memtable之前，会先写入Log文件，这样即使系统发生故障，Memtable中的数据没有来得及Dump到磁盘的SSTable文件，LevelDB也可以根据log文件恢复内存的Memtable数据结构内容，不会造成系统丢失数据，在这点上LevelDb和Bigtable是一致的。"

### 日志数据结构

日志中的每条记录由Record Header + Record Content组成，其中Header大小为kHeaderSize(7字节)，由CRC(4字节) + Size(2字节) + Type(1字节)三部分组成。除此之外才是content的真正内容：

![img](https://upload-images.jianshu.io/upload_images/5135780-4e9e8274f275c786.png?imageMogr2/auto-orient/strip|imageView2/2/w/766/format/webp)

日志文件的基础部件很简单，只需要能够创建文件、追加操作、实时刷新数据即可。为了做到跨平台、解耦，LevelDB还是对此做了封装。Leveldb命名空间下，有一个名为log的子命名空间，其下有Writer、Reader两个实现类。按前几节的命名规则，Writer其实是一个Builder，它对外提供了唯一的AddRecord方法用于追加操作记录。

### 创建日志的时机

- 数据库启动
- 插入数据
- 数据库恢复

## 版本控制

LevelDB如何能够知道每一层有哪些SST文件；如何快速的定位某条数据所在的SST文件；重启后又是如何恢复到之前的状态的，等等这些关键的问题都需要依赖元信息管理模块。对其维护的信息及所起的作用简要概括如下：

- 记录Compaction相关信息，使得Compaction过程能在需要的时候被触发；
- 维护SST文件索引信息及层次信息，为整个LevelDB的读、写、Compaction提供数据结构支持；
- 负责元信息数据的持久化，使得整个库可以从进程重启或机器宕机中恢复到正确的状态；
- 记录LogNumber，Sequence，下一个SST文件编号等状态信息；

LeveDB用**Version**表示一个版本的元信息，Version中主要包括一个FileMetaData指针的二维数组，分层记录了所有的SST文件信息。**FileMetaData**数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。除此之外，Version中还记录了触发Compaction相关的状态信息，这些信息会在读写请求或Compaction过程中被更新。可以知道在CompactMemTable和BackgroundCompaction过程中会导致新文件的产生和旧文件的删除。每当这个时候都会有一个新的对应的Version生成，并插入VersionSet链表头部。

**VersionSet**是一个Version构成的双向链表，这些Version按时间顺序先后产生，记录了当时的元信息，链表头指向当前最新的Version，同时维护了每个Version的引用计数，被引用中的Version不会被删除，其对应的SST文件也因此得以保留，通过这种方式，使得LevelDB可以在一个稳定的快照视图上访问文件。VersionSet中除了Version的双向链表外还会记录一些如LogNumber，Sequence，下一个SST文件编号的状态信息。

![img](https://upload-images.jianshu.io/upload_images/530927-41785ae1255ee69a.png?imageMogr2/auto-orient/strip|imageView2/2/w/914/format/webp)

