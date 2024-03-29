- [背景以及动机](#背景以及动机)
  - [关于 RocksDB 的一些简单介绍](#关于-rocksdb-的一些简单介绍)
  - [Cgo:权宜之计](#cgo权宜之计)
  - [艰巨的任务](#艰巨的任务)
- [LSM 树](#lsm-树)
- [Badger](#badger)
  - [设计指导思想](#设计指导思想)
  - [Key 和 Value 分离](#key-和-value-分离)
  - [写放大](#写放大)
  - [读放大](#读放大)
  - [故障恢复](#故障恢复)
  - [占用空间](#占用空间)
  - [硬件成本估算](#硬件成本估算)
- [性能测试](#性能测试)
  - [测试配置](#测试配置)
  - [测试结果](#测试结果)
- [未来的一些工作](#未来的一些工作)
  - [区间迭代性能优化](#区间迭代性能优化)
  - [压缩的性能（compaction 和 compression 的区别是什么？）](#压缩的性能compaction-和-compression-的区别是什么)
  - [LSM 树的压缩](#lsm-树的压缩)
  - [B+ 树](#b-树)
- [结论](#结论)

最近我们使用纯 Go 语言编写了一套基于 LSM Tree 的 KV 存储数据库，能够实现高效的数据存取以及支持持久化。这个数据库主要基于这篇论文：WiscKey。这种设计基于 SSD 进行高度优化，同时将 key 和 value 进行拆分存储来最大程度的降低 IO 的写放大，一并提升 SSD 的顺序和随机读写性能，我们将它命名为 badger。在随机读性能方面，Badger 至少是 RocksDB 3.5 倍的性能。在 128B 到 16KB 块大小的数据加载方面，Badger 比 RocksDB 快了 0.86-14 倍，随着块大小的增加，Badger 的性能增加尤为明显。在另一方面，虽然现在 Badger 在 key-value 的区间迭代性能方面还有不足，但是还有很大的优化空间。

## 背景以及动机

### 关于 RocksDB 的一些简单介绍

RocksDB 基本可以算是市面上最为流行并且性能最好的 key-value 数据库了。它起源于 google 内部的 SSTable，最早是 Bigtable 的基础，后来作为 LevelDB 的产品发布。然后 Facebook 改进了 LevelDB，增加了并发支持并对 SSD 进行了优化，发布为一个叫做 RocksDB 的产品。对 RocksDB 的改进持续了很多年，现在已经使用在 Facebook 和其他很多公司的线上产品中。 所以如果你需要一款 key-value 存储的话，RocksDB 确实是不二的选择，它已经是一款成熟的产品并且很稳定。但是 RocksDB 最大的问题是它使用 C++ 编写，如果要在 GO 的技术栈中使用，必须要引入 CGO.

### Cgo:权宜之计

在 Dgraph，我们从一开始就已经通过 Cgo 使用 RocksDB 了，在这个过程中，我们遇到了很多依赖方面的问题。Cgo 并不是 go，但是如果在 C++ 上有比 Go 更好的第三方库的话，使用 Cgo 就成为了一个权宜之计。

但是问题在于，Go 的 CPU Profiler 并不能分析 Cgo 的调用。Go 的 Memory Profiler 更甚，不要妄想 profiler 能够详细深入分析 Cgo 的内存空间栈，它甚至不能检测到 Cgo 代码的存在。任何 Cgo 使用的内存空间 Go 的 memory profiler 都无法监测。其他工具类似于 Go 的 race dector 也不能在 Cgo 上正常工作。

在 Go1.4 版本上，pthread_create 的问题困扰了我们很久，在 Go1.5 上又重现了，归咎于 bug 回归缺陷。如果引入了 Cgo，本来轻量级的 goroutine 就回退化为代价昂贵的 pthread 线程，因此我们不得不修改数据写入 RocksDB 的方式来避免生成过多的 goroutine。

Cgo 同时也有内存泄漏的问题。我们不清楚在 Cgo 调用的时候到底是谁在拥有并管理着这块内存。Go 和 C 是相互对立的，一个不允许你释放内存，另一个又要申请内存。因此你执行一个 Go 调用，但是忘记 Free 内存了，当下不会有什么问题，但是长时间运行以后，你懂的。

Cgo 是的我们的代码不可维护，并且引入了不和谐的代码风格。在公司里，没有人愿意去碰和 RocksDB 连接的 Cgo 层的代码。

当然，后来在我们的 API 使用层面我们修复了内存泄漏的问题。实际上，到现在为止，我认为我们已经修复了所有的问题，但是我不能保证。Go 的 memory profiler 是不会告诉你有没有内存泄漏问题的。每次客户抱怨 Dgraph 的产品占用了太多内存或者因为 OOM 的原因被杀掉了，我都很紧张这是因为内存泄漏的问题。

### 艰巨的任务

每当我向别人抱怨 Cgo 不爽的时候，他们总是建议我只是去改改 bug 就好了，重写一套和 RocksDB 性能相仿的 KV 存储的活实在是太艰巨了，不是很值得。我的组员也是不置可否，我对此也有所怀疑。

这么些年来，我对那些由世界上最聪明的工程师创造出来的技术都心怀敬意，RocksDB 就是其中一个。如果我们的 Dgraph 是用 C++ 编写的话，我会很开心地使用 RocksDB。 但是，我真的忍不了丑陋的代码

我也讨厌回归复现的问题。无论付出多大努力，我们都无法保证通过 Cgo 使用 RocksDB 不会出现更多问题。我想要干干净净的代码，同时我也期望那些 profiler 工具能够正常使用。用 Go 语言从头构建一个 KV 存储看起来是我们唯一的选择。

我做了一些业界扫描，现存的 Go 编写的 KV 存储都无法和 RocksDB 性能相比。这是成败的决定性问题，不能拿性能换代码整洁，我们两个都需要。

最后，我决定替换我们对 RocksDB 的依赖。但是这不是 Dgraph 的首要任务，我们的组员不应该在这个任务上花太多时间。这应该只是一个由我来负责的支线任务。我开始研读 B+ 了 LSM 树的相关文档以及他们最近的研究成果，同时详细研究了 WiscKey 的论文。这篇论文给了我很大的启发，我决定从 Dgraph 的核心开发任务中抽出一个月的时间来构建 Badger。

实际上并不是像我想象的这样发展的。我不可能完全脱离 Dgraph 一个月的时间。由于我创始人的职责， 我也不可能全职投入编码。Badger 是在我们的编码运动会开发出来的，同时也包含我一个组员的兼职投入。开发从 1 月底开始，现在我认为是一个比较好的时间点来贡献给 Go 社区让大家来试用了。

## LSM 树

当我们深入探讨 Badger 之前，我们先来理解一下 KV 存储的设计思路。KV 存储在数据密集型的应用中扮演着非常重要的角色，包括数据库。KV 存储尤其擅长高效的更新操作，定点查询以及区域查询操作。

KV 存储有 2 种流行的实现方式：：LSM 树和 B+ 树。LSM 树的优势在于它前台的写操作都是在内存中完成的，后台的写操作可以保持为顺序的写操作，因此它能实现高吞吐的写入操作。另外，在 B+-树上进行的小块的更新操作会引入重复的磁盘随机写，因此在写操作上不可能维持太高的吞吐性能。

为了实现写入操作的高性能，LSM 树批量打包 KV 对并将它们顺序写入磁盘。同时为了高效的查询操作，LSM 树在后台会持续的读取，排序并且回写 KV 对，这个操作就是我们常说的 compaction 压缩。LSM 树会在多个 level 上进行压缩，每一层都比上一层存储更多的数据，典型的遵循这个公式：size of Li+1 = 10 * size of Li

在某一 level 中，kv 对会被排序后写入到固定大小的文件中。除了 0 level，其他每一层中每个文件存储的 key 值都不会重叠。

每一个 level 都有最大存储容量。如果 Li 层的空间填满以后，Li 层的数据会合并到下一层的 Li+1 层中，同时 Li 层的文件会被删除以腾出空间接收新的数据，数据就按照这样的流程从 level 0 流入 level 2,3…以此类推。同一个数据在它的生命周期中会写入多次，每一次 key 值的更新都会出发多次的写操作直至数据最终落地，这个过程就形成了所谓的 “写放大”。对于一个 7 层并具有 10x 的放大因子的 LSM 树来说，它会将写操作放大 60 倍，排除 L0 有特殊操作以外，L1-L6 都会有传导影响。

相反的，从 LSM 树中读取一个数据，需要检查所有的 level 层。如果数据在多个层中都存在，那么会提取最靠近 Level0 的数据。因此，单一一个 key 的查询就回导致在多个文件中的频繁的读操作，这就是所谓的 “读放大”。WiscKey 的论文估计：对于 1KB 的 KV 对来说，这个放大值将近 336

LSM 树本身是围绕 HDD 传统硬盘来设计的。HDD 的随机 I/O 速率比其顺序 IO 要慢超过 100 倍，因此后台进行数据压缩来持续的排序 key 值同时提供高效的查询是一个不错的权衡。

但是，SSD 和 HDD 从本质上就是不同的。SSD 上顺序读和随机读的性能差异并不像 HDD 上的差异那么大。实际上，SSD 顶级产品例如三星 960Pro 可以提供 440K，4KB 的随机读性能。因此，LSM 树中转化为顺序写来降低了随机读操作的设计对带宽来说是个巨大的浪费，没有什么实际用途。

## Badger

Badger 是一个简单，高效，持久化的 KV 存储。受到 LevelDB 简洁化的影响，我们只设计了 Get,Set,Delete 和 Iterate 几个操作，在其上，我们还增加了 CompareAndSet 和 CompareAndDelete 这样的原子操作。Badger 的目标不是一个数据库，因此没有提供事务，版本控制以及快照功能。这些功能基于 Badger 都很好自行实现。

Badger 将 Key 和 Value 的存储进行分离。Key 存储在 LSM 树中，value 存储在 wal（write-ahead log）中。Keys 的总体容量一般来说会比 value 小，因此这个设计会生成一棵体积小很多的 LSM 树。当进行数据查询的时候，vlaue 的值会直接从 SSD 盘上的 log 文件中读取，以便最大利用 SSD 的随机读性能。

### 设计指导思想

下面是 Badger 设计上的一些考量，Badger 是什么以及 Badger 不是什么 纯 Go 语言开发 使用了最新的研究成果来构建快速的 KV 存储  简单，傻瓜  SSD 为中心的设计

### Key 和 Value 分离

LSM 树最大的性能损耗来自于其压缩的处理过程。在压缩过程中，多个文件读取到内存中进行排序并写回磁盘。排序对于检索是很关键的，不论对于 key 的查找还是区间迭代来说都是如此。经过排序后，key 的查找只需要在每个 level 中访问一个文件即可（level 0 除外，这一层中需要检查所有的文件）.迭代操作会转化为多对个文件的顺序访问。

每个文件都是固定大小来增强缓存的能力。Value 通常来说都大于 Key。当把 value 和 key 一起存储的时候，要压缩的数据的容量会明显地增加。

在 Badger 的设计中，和 key 一起存储的只有一个指向 vlaue 日志中的指针值。Badger 后期计划引入 delta encoding 对 key 进行编码来继续减小 key 的体积。假设有 16 字节的 key 和 16 字节的指向 value 的指针，那么一个单独的 64MB 的文件可以存储 200 万个 KV 对。

### 写放大

由上所述，Badger 生成的 LSM 树会比 RocksDB 小很多，由此生成的 level 的层次数也减少了，相应的用于稳定 LSM 树的压缩操作也减少了。同时，value 的值存储在 value 日志中，并不会和 key 的值一起移动。假设有 1KB 的 value 和 16 字节的 key，现在每一层的写放大的倍率是：(10*16+1024)/(16+1024)~1.14，减小了很多。

当 value 的大小增长的时候你就会看到这样设计在性能方面的收益，同时当 Badger 导入数据的时候所需的时间也会按照倍率减小。

### 读放大

上面说过，Badger 产生的 LSM 树体积小很多，相比于传统的 LSM 树来说，每层的文件可以存储更多的 key 值。因此对于相同数量的数据，Badger 需要的 level 层次数也会降低。一个典型的 key 的查找会读取 level 0 的所有文件，然后会在每层读取一个文件直到找到为止，因此在 Badger 中这就意味着查找一个 key 需要读取的文件总数减少。一旦 key 的值被找到，value 的值就可以从 SSD 中的 value 日志中进行随机读取获得。

此外，在性能测试中，我们发现 Badger 生成的 LSM 树非常小，特别适合 RAM。对于 1KB 的 value 值和 75,000,000 个 22 字节的 key 值来说，整个数据集的大小为 72GB。Badger 生成的 LSM 树仅仅只有 1.7G，非常容易放置在内存中。这就是为什么 Badger 的随机 key 查找的性能是 RocksDB 的 3.5 倍，同时 Badger 的 key 单独迭代的性能更是吊打 RocksDB。

### 故障恢复

LSM 树的更新操作首先都是更新内存中的 memtables，一旦填满，memtables 会被转换为不可变的 memtables，这个不可变的 memtables 最终会写到磁盘中的 level 0 层的文件中。

如果 Badger 崩溃，那么所有最近存储在内从中的更新都会丢失。为了处理这个问题，KV 存储会先将更新写入到 WAL（write-ahead-log）中，Badger 也有 WAL，叫做 value 日志。

和传统的 WAL 一样，更新的数据写入 LSM 树之间会先写到 value 日志中。一旦程序崩溃，Badger 会遍历 value 日志中所有最近的更新，然后将相应的数据写回到 LSM 树中。

Badger 并不会遍历整个 value log，相反 Badger 会存储一个指针到 memtable 中最近的一个 value 中。这样最近往磁盘中写如数据的 memtable 都会有一个 value 指针，在这个指针之前的数据都已经落盘。因此，我们只需要从这个指针处向前处理数据，将所有的更新数据更新到 LSM 树中即可完全找回我们的更新数据。

### 占用空间

RocksDB 采用了压缩来减小总体占用的空间。Badger 的 LSM 树比 RocksDB 小了很多，可以整体载入内存中，因此它不需要做任何的压缩操作。但是 value 日志的大小会增长的很快，每一个更新操作都会在 value 日志中记录为一个条目，因此对于同一个 key 的多次更新操作也会记录多次，占用多个条目空间。

Badger 采用了 2 个措施来处理这种情况。它允许在 value 日志中对 vlaues 的值进行压缩，我们并不是将多个 KV 对绑定在一起进行压缩，而是仅仅单独压缩每一个 KV 对，这样的方式提供了最好的随机读的性能。客户端可以进行相应配置，当 KV 的大小超过了某个阈值的时候开启压缩特性，这个默认值为 1KB

第二点，Badger 对 value 进行了垃圾回收。垃圾回收会周期性运行，从 value 日志文件中随机选择 100MB 的空间进行回收。它会检查如果在后期的 log 中有新的更新，那么前面明显的旧更新就可以丢弃了。因此，如果一个新的 KV 对添加到了日志中，那么久的文件就可以丢弃，同时更新 LSM 书中的最近的 value 指针。这样做的缺点是，LSM 树需要进行额外的操作，因此在导入海量数据的过程中最好不要进行垃圾回收。后期计划增加新的特性，使其在空闲的阶段触发垃圾回收。

### 硬件成本估算

SSD 的成本是越来越便宜，相比于使用内存来存储数据提供 LSM 树服务来说，SSD 的成本基本可以忽略不计。考虑如下的因素： 对于 16 字节的 key+1KB 的 value 值，存储 75,000,000 个这样的 KV 对，RocksDB 中的 LSM 树大概需要 50GB 的空间。Badger 的 value 日志在不压缩的情况下需要 74GB 的空间，而 LSM 树需要 1.7GB。由此推断如果增长 3 倍，225,000,000 个 KV 对，RocksDB 会占用 150GB 空间，而 Badger 的 value 日志占用 222GB，同时包括 5.1GB 的 LSM 树。

使用亚马逊 AWS US East（Ohio）数据中心的成本如下： 1. 如果想要达到和 Badger 同样的随机读性能（3.5 倍），RocksDB 必须要使用 r3.4xlarger 的虚拟机实例，这个实例提供 122GB 的内存，每个小时的收费是 1.33 美元，这样所有的 LSM 树的数据都可以存储在内存中。 2. Badger 可以运行在 AWS 最便宜的为存储提供的实例中 i3.large，这个实例提供了 475GB 的 NVMe SSD（fio 的测试其能达到 4KB 100K 的 IOPS 性能），同时提供了 15.25GB 的内存，每个小时的费用是 0.156 美元。 3. 这样折算下来，在 EC2 集群上，Badger 比 RocksDB 要便宜 8.5 倍 4. 以一年为期，预支付情况下，RocksDB 需要支付 6182 美元而 Badger 只需要 870 美元，仍然要便宜 7.1 倍，相比来说 Badger 节省了 86% 的费用

## 性能测试

### 测试配置

我们租借了一台亚马逊的 AWS 服务器，i3.large 的实例配置，提供了 450GB 的 NVMe SSD，2 个虚拟 CPU 和 15.25GB 的内存。这个实例提供了本地的 SSD，经过我们使用 fio 测试，4KB 块大小的随机读性能可以稳定在 100K 的 IOPS。 无论对于 RocksDB 还是 Badger 来说，我们生成的测试数据集足够庞大，不可能全部放入内存中。如下表：

首先我们测试数据串行一条一条进行载入，不使用并发操作，然后我们记录载入数据所使用的时间和最终数据的大小。对于随机的 Get 和 Iterate 操作，我们使用 Go 的测试集测试 3 分钟，然后后对于 16KB 的 vlaue 值测试 1 分钟。

所有的代码都托管在[这里](https://github.com/dgraph-io/badger-bench)。所有执行的命令行以及测量方法都记录在这个[日志文件](https://github.com/dgraph-io/badger-bench/blob/master/BENCH-rocks.txt)。生成的图表和数据在[这里](https://docs.google.com/a/dgraph.io/spreadsheets/d/1x8LUw_85g8Jo9jFtbAwuXrLm_DB1SOG8QCTSjXj8Hk0/edit?usp=sharing)

### 测试结果

我们主要测试 4 种场景： 1. 数据载入性能 2. 输出数据的大小 3. 随机 Key 查找的性能（Get） 4. 排序的区间迭代的性能（Iterate）

所有 4 个测试结果图如下所示：



[![测试结果](https://blog.dgraph.io/images/badger-benchmarks.png)](https://blog.dgraph.io/images/badger-benchmarks.png)



\* 数据载入性能 *：当 value 的大小增长的时候，Badger KV 分离存储的设计对性能的提升尤为明显。对于 1KB 和 16KB 的 value 值，Badger 的性能分别是 RocksDB 的 4.5 倍和 11.7 倍。对于小一些的 value 值，例如 16 字节的 value，Badger 可能会慢 2-3 倍，主要原因还是压缩的性能较低（后期会改进）

\* 存储空间大小 *：Badger 生成的 LSM 树会比较小，但是 value 日志文件会大一些。LSM 树的大小和 key 的数量等比增长，和 value 的大小没有关系。因此当 value 的值从 128 字节增长到 16KB 的时候，LSM 树的大小呈现减小的趋势。在所有 3 个场景中，Badger 生成的 LSM 树都非常适合全部载入内存中。

\* 随机读延迟 *：Badger 的 Get 操作的延迟只有 RocksDB 的 18%-27%。这主要归功于我们的设计：Badger 的整个 LSM 树都可以装载进内存，不再需要寻找相应的 table，检查布隆顾虑器，获取相应的块获得 key 的值，然后 value 的值通过 SSD 上一个单独的 file.pread 获取。

相反，RocksDB 无法将整棵 LSM 树载入内存，就算假设其可以将 table 的索引和布隆过滤器放在内存中，它仍需要从磁盘中获取整个块，并解压进行查询获取 KV 对（Badger 的 LSM 树很小，以致于并不需要进行压缩）。这个流程明显更加耗时，同时数据获取无法做到局部化，因此 cache 的命中率也很低。

\* 区间迭代延迟 *：同样从 SSD 获取 value 值的情况下，Badger 的区间迭代明显比 RocksDB 慢。这个现象超出了我们的预期，同时我们也很是不理解。可能的原因是 Badger 需要在 SSD 上做随机读 IOPS 的操作，而 RocksDB 是纯粹的顺序读。但是，就算是使用了能达到 100K IOPS 的 i3.large 的虚拟机实例，开启了预取模式后，Badger 仍然只是用了部分的磁盘带宽，SSD 的带宽并没有用满。因此这个问题需要后期再继续定位解决。 但是在另一方面，Badger 的单纯 key 的区间迭代性能比 RocksDB 要快很多。这个特性在我们 Dgraph 中的某些特定场景下很有用，例如我们迭代一组 key，进行过滤并获取一小部分 key 所对应的 value 值。

## 未来的一些工作

### 区间迭代性能优化

Badger 里单独 Key 迭代的性能非常的快，但是当需要查询 value 的值的时候，性能下降明显。理论上不应该出现这种情况，亚马逊 i3.large 的虚拟机实例优化后可以达到 4k 100,000 的随机读性能，这样算下来的话，我们应该能够每秒迭代 100K 的 KV 对，也就是每分钟 6,000,000 的 KV 对。 但是，当前的 Badger 对 SSD 产生的随机读压力完全无法到达理论线速，导致 KV 的迭代性能大打折扣，这个情况需要深度的分析和优化。

### 压缩的性能（compaction 和 compression 的区别是什么？）

相比于 RocksDB 来说，Badger 现在执行压缩操作时速度会慢很多。因此对于 value 值偏小的数据集来说，Badger 在载入数据方面的性能会差一些，这方面也需要更深入的优化工作。

### LSM 树的压缩

对于 value 值偏小的数据集来说，Badger 的 LSM 树会比 RocksDB 生成的 LSM 树更大一些，因为 Badger 对 LSM 树不进行压缩。不过如果需要的话，这个特性很容易就可以添加进来。

### B+ 树

现在的 SSD 的发展使得 B+ 树成为一个可行的选择，就如 WiscKey 的论文提到的，现在 SSD 的随机写的性能提高很快。一个比较有意思的新方向是：结合 value 日志的方式，把 key 和 value 的指针存储在 B+-树中。这样设计以后，就不需要使用 LSM 树的读 - 排序 - 合并，顺序写到磁盘的操作了，而是在每个 key 更新的时候对 SSD 磁盘进行随机写操作，这样仍然能够获得和 LSM 树同样的写吞吐性能，同时设计上更为简单了。

## 结论

我们构建了一个高效的 KV 存储，可以和市面一线的 KV 存储性能上相媲美。当前阶段它确实还是有一些缺点的，但是无论是作为数据存储还是构建其他数据库来说，它已经可以作为商业应用软件的坚实平台了。

我们已经在 Dgraph 内部使用 Badger 替代对 RocksDB 的依赖了，它使得我们平台的构建更加容易和快速，同时实现了 Dgraph 的跨平台特性，使得可嵌入式的 Dgraph 成为可能。Badger 最大的优势在于其高性能的基于纯 GO 的 KV 存储设计。好的方面是由于减少了对 RAM 的依赖，更多依赖于更加快速和便宜的 SSD，Badger 在 Get 性能方面 4 倍于 RocksDB，同时潜在节省了 86% 的亚马逊虚拟主机开销。