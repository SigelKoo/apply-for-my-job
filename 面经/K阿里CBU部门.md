## 一面

自我介绍，聊项目

1. String str = "aa"
   String str = new String("aa")区别
2. Integer a = new Integer(100)
   Integer b = new Integer(100)
3. exception和error的区别，error可以捕获嘛，之间的联系
4. Hashmap可以存null嘛，hashTable和hashmap有什么区别，hashmap加入元素的过程，为什么使用红黑树
5. Object会有哪些方法
6. sychronized和ReentrantLock区别
7. 乐观锁 和 悲观锁的区别，乐观锁如何实现，乐观锁会有什么问题，公平锁和非公平锁区别，非公平锁如何实现，sychronized锁的升级过程
8. 类是什么时候加载到内存的，类加载机制，能打破嘛，如何实现自定义的加载器
9. 字节码的结构
10. spring是什么，spring的bean周期，spring循环依赖如何解决，哪些接口可以修改bean的加载过程
11. 垃圾回收算法，垃圾回收器，新生代用什么，老年代用什么，cms垃圾回收器会stop the world嘛，cms会出现什么问题，知道G1嘛
12. Redis哪些数据结构，哪些编码方式，hash的结构，缓存穿透，分布式锁可以用redis实现，会出现什么问题
13. 消息中间件作用，事务形的消息中间件如何实现
14. Mysql 隔离级别，如何解决幻读，mvcc，如果表的两列a和b，a是一个比较大的数据，b是整型，如何建立索引

## 二面
全程问项目

1. 用到GO语言？
2. 为什么使用Go语言？
3. 为什么使用Gin框架？
4. Java和Go有什么区别
5. Go有垃圾回收机制
6. Go最大特点（协程）
7. 微服务知道嘛
8. spring cloud知道嘛
9. spring cloud使用什么序列化机制
10. K8S的架构
11. K8S使用什么存储（etcd）
12. 为什么使用etcd
13. 为什么不使用redis
14. etcd的特点
15. etcd的协议
16. K8S和docker的区别
17. 一定要用docker
18. docker如何构建镜像
19. docker的多阶段构建
20. docker原理
21. 如何隔离
22. 用到了哪些系统调用
23. CGroup了解吗
24. 守护线程是什么
25. 你自己先的话如何写