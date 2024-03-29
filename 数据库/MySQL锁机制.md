![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/23/164c6d7ae44d8ac6~tplv-t2oaga2asx-watermark.awebp)

锁被数据库隐式加了：

- UPDATE、DELETE、INSERT语句，**InnoDB**会自动给涉及数据集加排他锁(X)
- **MyISAM**在执行查询语句SELECT前，会自动给涉及的所有表加**读锁**，在执行更新操作（UPDATE、DELETE、INSERT等）前，会自动给涉及的表加**写锁**，这个过程并不需要用户干预

粒度分为两大类：

- 表锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突概率高，并发度最低
- 行锁：开销大，加锁慢；会出现死锁；锁定粒度小，发生锁冲突的概率低，并发度高

属性

- InnoDB行锁和表锁都支持
- MyISAM只支持表锁
- InnoDB通过索引条件检索数据才使用行锁，否则使用表锁

表锁分为表读锁与表写锁，读读不阻塞，读写阻塞，写写阻塞

- 读读不阻塞：当前用户在读数据，其他的用户也在读数据，不会加锁
- 读写阻塞：当前用户在读数据，其他的用户**不能修改当前用户读的数据**，会加锁
- 写写阻塞：当前用户在修改数据，其他的用户**不能修改当前用户正在修改的数据**，会加锁
- 如果某个进程想要获取读锁，**同时**另外一个进程想要获取写锁。在mysql里边，**写锁是优先于读锁的**
- MyISAM可以支持查询和插入操作的**并发**进行，默认：如果MyISAM表中没有空洞（即表的中间没有被删除的行），MyISAM允许在一个进程读表的同时，另一个进程从表尾插入记录。

行锁：

- 共享锁：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。也叫做读锁：读锁是共享的，多个客户可以同时读取同一个资源，但不允许其他客户修改。
- 排他锁：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。也叫做写锁：写锁是排他的，写锁会阻塞其他的写锁和读锁。

为了允许行锁与表锁共存，实现多粒度锁机制，InnoDB还有两种内部使用的意向锁，两种意向锁都是表锁：

- 意向共享锁：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的意向共享锁。
- 意向排他锁：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的意向排他锁。

MVCC快照有两种：

- 语句级，针对于读已提交隔离级别
- 事务级，针对于可重复读隔离级别

读未提交：操作完该数据就立即释放掉锁。导致脏读---读的数据就变成了无用的或者是错误的数据。

读已提交：事务提交后才释放掉锁，事务提交前其他进程无法对该行数据进行读取。导致不可重复读---一个事务读取到另外一个事务已经提交的数据，也就是说一个事务可以看到其他事务所做的修改。

可重复读：每次读取的都是当前事务的版本，即使被修改了，也只会读取当前事务版本的数据。幻读---一个事务内读取到了别的事务插入的数据，导致前后读取不一致。

串行化：可重复读+GAP间隙锁。

当我们**用范围条件检索数据**而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给**符合范围条件的已有数据记录的索引项加锁**；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”。InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁。

死锁

1）以**固定的顺序**访问表和行。比如对两个job批量更新的情形，简单方法是对id列表先排序，后执行，这样就避免了交叉等待锁的情形；将两个事务的sql顺序调整为一致，也能避免死锁。

2）**大事务拆小**。大事务更倾向于死锁，如果业务允许，将大事务拆小。

3）在同一个事务中，尽可能做到**一次锁定**所需要的所有资源，减少死锁概率。

4）**降低隔离级别**。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。

5）**为表添加合理的索引**。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大。



