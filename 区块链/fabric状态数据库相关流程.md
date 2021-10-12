# 状态数据库

状态数据库保存了最新的有效交易执行结果读写集（实际上只保存了写集合，读集合是为了过滤出有效交易），其状态数据表示通道上指定键的最新值，及世界状态。

在背书节点模拟执行交易期间，都会为每笔交易生成一个读写集。

`读集(readset)`包含了在模拟执行期间读取到的键的列表和它们上次被提交时的版本号，

`写集(writeset)`包含了键的列表（可能于读集中的键重复）和交易即将对它们写的值

# 流程

peer节点启动startCmd()，调用serve() -> ledgermgmt.NewLedgerMgr() -> kvledger.NewProvider()创建账本提供者，创建历史数据库提供者、状态数据库提供者等，其中openIDStore调用了leveldbhelper.CreateDB()，其他函数调用leveldbhelper.NewProvider()，其实就是调用leveldb的提供者的NewProvider()函数，打开数据库并进行检查；而p.initStateDBProvider()调用privacyenabledstate.NewDBProvider()通过不同的stateDBConfig配置（docker-compose-test-net.yaml这里设置为CORE_LEDGER_STATE_STATEDATABASE=badger/////core.yaml文件中的goleveldb）去打开不同的数据库，这里添加了badger的配置，本来是可以将所有leveldb全部替换为badger的，但导师只让做了一个。

从创世区块创建provider.CreateFromGenesisBlock() -> p.open() -> p.dbProvider.GetDBHandle() 向下执行 newKVLedger()利用状态数据库初始化交易管理器。Peer节点账本对象通过交易管理器的验证与状态数据库，对当前提交到账本的交易读集合执行MVCC检查。

peer节点通过ValidateAndPrepare验证并准备区块数据，验证过的数据更新批量操作batch，Commit()添加有效交易的状态数据到状态数据库，调用ApplyPrivacyAwareUpdates -> ApplyUpdates() -> NewUpdateBatch()创建batch，循环取出所有更新，组装key，调用Put()与Delete()，最终调用WriteBatch()，实际调用badger数据库的Set()、Delete()与Flush()等操作。

调用NewQueryExecutor()创建状态数据库查询执行器，有多种Query函数，可以进行命名空间与指定键范围查询，调用newResultsltr进行判断pageSize是否为0，很奇怪的是不管是不是0，都调用了GetStateRangeScanIteratorWithPagination()组装命名空间与startkey/endkey，若endkey为空，组装为byte(0x01)，调用GetIterator()组装dbName与startkey/endkey，若endkey为空，组装为byte(0x01)，创建并返回一个迭代器，最终在上层应用使用Next等函数将数据从迭代器中一条条取出。

