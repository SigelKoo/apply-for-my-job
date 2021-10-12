# 总流程

1. SDK/cli提交一笔交易提案給1个/多个Peer
2. 每个Peer模拟执行的结果（每个Peer都有ledger，每次更新都会更新ledger的version）并响应背书签名
3. SDK/cli判断Peer的结果是否满足背书策略
4. SDK/cli向orderer发起更新申请
5. orderer排序、校验数据合法性，避免双花，检查签名，检查每个peer背书的读写集是否一致，没有问题的话，通知每个Peer去应用新的读写集。Orderer内部有消息队列，可控制队列数量、容量、提交周期、优化hyperledger执行效率
6. orderer让Peer调用、更新

# 概念

- CA：为不同的用户生成不同的证书，每个用户有不同的属性

- App/SDK：客户端使用SDK与Fabric网络进行通信

- orderer：负责对合法交易进行排序，并且打包排序后的交易为区块。以太坊中的矿工
- peer
  - endorser：完成对交易提案的背书处理
  - committer：存储与同步账本数据

- 通道：每个通道是一个fabric的实例（区块链子网），可理解为微信群，peer为群中用户，peer加入群有加入策略（管理员同意/两个以上管理员同意）

- 链码：智能合约，用来更新账本数据。SDK发起交易，peer执行链码，链码属于某个通道，执行时要求出示权限，明确通道、链码、函数、参数。链码要在通道中每个peer安装。要先安装，再实例化启动docker，在容器中运行。install -> init -> invoke