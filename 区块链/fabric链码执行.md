# Peer与容器通信

1. 客户端应用程序发送实例化模拟提案
2. peer节点收到提案后，执行模拟提案
3. peer节点（使用fsouza/go-dockerclient第三方库与Docker API来构建Docker镜像）判断链码容器是否启动，如果没有启动，则启动链码容器，调用shim.Start()方法，并最终建立起链码侧与peer节点两端的handler的GRPC通信。
   1. 链码侧发送REGISTER消息给peer节点，当前链码侧状态为created
   2. peer节点当前状态为created,收到REGISTER消息以后，将其注册到peer节点的handler上，发送REGISTERED消息给链码侧，同时更新peer节点状态为Established
   3. peer节点再发送Ready消息给链码侧，同时更新peer节点状态为Ready
   4. 链码侧收到REGISTERED消息后，更新链码侧状态为Established
   5. 链码侧收到Ready消息后，更新链码侧状态为Ready
   6. 此时两侧状态都是Ready状态，互相通信
4. peer节点判断提案是lscc的deploy请求，则发送INIT消息给链码侧
5. 链码侧收到INIT消息以后，调用链码的Init()方法，并返回COMPLETED消息
6. peer节点收到COMPLETED消息以后，执行Notify()方法，往交易上下文的响应通道中发送响应数据
7. callChaincode()方法内部会最终会调用到一个Execute()方法，监听这个通道，并拿到响应数据
8. 拿到响应数据之后，进行背书，并最终将背书结果发送给客户端应用程序
9. 客户端应用程序收到背书结果后，生成一笔交易，发送给排序节点
10. 排序节点排序好之后，发送给记账节点，记账节点验证无误之后记账，整个流程结束

#  链码执行

ProcessProposal()启动链码容器与初始化链码执行环境，模拟执行合法的签名提案消息，并将模拟执行结果记录在交易模拟器中。对于chainID不为空的签名提案消息，ProcessProposal()方法为该交易创建交易模拟器GetTxSimulator()与历史查询执行器GetHistoryQueryExecutor()，用于保存模拟执行结果与查询历史数据（交易模拟器可以访问本地账本的状态数据库）。调用SimulateProposal()模拟执行交易提案。

SimulateProposal()从ChaincodeInput中解析提取出链码调用规范对象，为callChaincode()封装了链码调用执行的参数，如请求系统链码、调用命令以及参数列表，判断是用户链码还是系统链码。

callChaincode()，首先检查并设置交易模拟器到context上下文对象，接着通过Execute()方法执行链码

Execute()实际调用Invoke()用于检查与启动容器，请求执行链码的相应方法（ExecuteLegacyInit()要被移除）

Invoke()执行CheckInvocation()，Launch()，execute()。CheckInvocation检查调用的参数，并确定是否、如何以及该调用应该路由到哪里。

Launch()调用start()，转了很多次，实际调用了DockerVM.Start()负责启动Docker容器以支持用户链码服务，使用fsouza/go-dockerclient第三方库与Docker API来构建Docker镜像。GetVMNameForDocker()获取镜像名；GetVMName()创建容器名；调用stopInternal() -> StopContainer()杀死和删除同名的Docker容器对象；createContainer()创建用户链码的Docker容器。

调用shim.Start()方法，并最终建立起链码侧与peer节点两端的handler的GRPC通信。

1. 链码侧发送REGISTER消息给peer节点，当前链码侧状态为created
2. peer节点当前状态为created,收到REGISTER消息以后，将其注册到peer节点的handler上，发送REGISTERED消息给链码侧，同时更新peer节点状态为Established
3. peer节点再发送Ready消息给链码侧，同时更新peer节点状态为Ready
4. 链码侧收到REGISTERED消息后，更新链码侧状态为Established
5. 链码侧收到Ready消息后，更新链码侧状态为Ready
6. 此时两侧状态都是Ready状态，互相通信

