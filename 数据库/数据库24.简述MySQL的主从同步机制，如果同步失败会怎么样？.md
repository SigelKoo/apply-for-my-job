# 简述MySQL的主从同步机制，如果同步失败会怎么样？

当master(主)库的数据发生变化的时候，变化会实时的同步到slave(从)库。

不管是delete、update、insert，还是创建函数、存储过程，所有的操作都在master上。
当master有操作的时候，slave会快速的接收到这些操作，从而做同步。

在master机器上，主从同步事件会被写到特殊的log文件中(binary-log);
在slave机器上，slave读取主从同步事件，并根据读取的事件变化，在slave库上做相应的更改。

### 好处

- 水平扩展数据库的负载能力。
- 容错，高可用。Failover(失败切换)/High Availability
- 数据备份。

### 主从同步事件有哪些

在master机器上，主从同步事件会被写到特殊的log文件中(binary-log)

主从同步事件有3种形式：statement、row、mixed。

- statement：会将对数据库操作的sql语句写入到binlog中。
- row：会将每一条数据的变化写入到binlog中。
- mixed：statement与row的混合。Mysql决定什么时候写statement格式的，什么时候写row格式的binlog。

#### 在master机器上的操作

##### binlog dump线程

当master上的数据发生改变的时候，该事件(insert、update、delete)变化会按照顺序写入到binlog中。

当slave连接到master的时候，master机器会为slave开启binlog dump线程。
当master的binlog发生变化的时候，binlog dump线程会通知slave，并将相应的binlog内容发送给slave。

#### 在slave机器上的操作

当主从同步开启的时候，slave上会创建2个线程。

- I/O线程。该线程连接到master机器，master机器上的**binlog dump线程**会将binlog的内容发送给该**I/O线程**。该**I/O线程**接收到binlog内容后，再将内容写入到本地的relay log。
- SQL线程。该线程读取I/O线程写入的relay log。并且根据relay log的内容对slave数据库做相应的操作。

## 如果同步失败会怎么样？

1. 对于主从库数据相差不大，或者要求数据可以不完全统一的情况，数据要求不严格的情况下。**忽略错误后，继续同步**
2. 主从库数据相差较大，或者要求数据完全统一的情况。**重新做主从，完全同步**

