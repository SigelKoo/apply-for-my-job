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
