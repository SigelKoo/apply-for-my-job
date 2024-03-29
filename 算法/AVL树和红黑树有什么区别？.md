# AVL树和红黑树有什么区别？

| 平衡二叉树 | 平衡度 | 调整频率 | 适用场景       |
| ---------- | ------ | -------- | -------------- |
| AVL树      | 高     | 高       | 查询多，增删少 |
| 红黑树     | 低     | 低       | 增删频繁       |

1. AVL树
   1. 一般用平衡因子判断是否平衡并通过旋转来实现平衡，左右子树高度差不超过1，和红黑树相比，AVL树是高度平衡的二叉树，平衡条件必须满足（所有节点的左右子树高度差）。不管我们是执行插入还是删除操作，只要不满足上面的条件，就要通过旋转来保持平衡，而由于旋转比较耗时，我们可知AVL树适合用于插入与删除次数较少，但查找多的场景
   2. 由于维护这种高度平衡所付出的代价比从中获得的收益效率要大，故而实际的应用不多，更多的地方是用追求局部而不是非常严格要求整体平衡的红黑树
   3. 早期Windows使用AVL树来管理进程地址空间，因为进程地址空间的查询操作更频繁
2. 红黑树
   1. 每个节点有一个存储位表示节点的颜色，可以是红色或黑色，通过对任何一条从根到叶子的路径上各个节点着色的方式限制。红黑树确保没有一条路径会比其他路径长出2倍，因此，在相同的节点情况下，AVL树的高度<=红黑树。红黑树旋转次数少，所以对于插入删除操作较多的情况使用红黑树
   2. * 节点是红色或黑色
      * 根是黑色
      * 所有叶子都是黑色（叶子是NIL节点）
      * 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
      * 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点（简称黑高）
   3. 应用于Java中的TreeMap，JDK 1.8中的HashMap，C++中的map；Linux中进程调度，红黑树管理进程控制块，进程的虚拟内存空间都存储在一棵红黑树上，每个虚拟内存空间都对应红黑树的一个节点，左指针指向相邻的虚拟内存空间，右指针指向相邻的高地址虚拟内存空间；IO多路复用的epoll采用红黑树组织管理sockfd，支持快速增删改查；Nginx使用红黑树管理定时器，红黑树有序，可以快速得到距离当前最小的定时器
