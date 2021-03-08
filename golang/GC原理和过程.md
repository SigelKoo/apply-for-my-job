# Golang GC

**垃圾回收**（Garbage Collection，简称GC）是编程语言中提供的自动的**内存管理**机制，**自动释放**不需要的对象，让出存储器资源，无需程序员手动执行。

Golang中的垃圾回收主要应用**三色标记法**，GC过程和其他用户goroutine可并发运行，但需要一定时间的**STW(stop the world)**，STW的过程中，CPU不执行用户代码，全部用于垃圾回收，这个过程的影响很大，Golang进行了多次的迭代优化来解决这个问题。

## 标记-清除（mark and sweep）方法

**Go 1.3 使用**

![img](https://img.kancloud.cn/01/60/0160c38ec63623f3108550ff648f0959_1494x1248.png)

1. 暂停程序业务逻辑，找出不可达的对象，然后做上标记。第二步，回收标记好的对象。

   操作非常简单，但是有一点需要额外注意：mark and sweep算法在执行的时候，需要程序暂停！即 `STW(stop the world)`。也就是说，这段时间程序会卡在哪儿。

2. 开始标记，程序找出它所有可达的对象，并做上标记。如下图所示：

   ![img](https://img.kancloud.cn/36/32/3632e8ce6e28998dd370298c5f2f2815_1548x1230.png)

3. 标记完了之后，然后开始清除未标记的对象。结果如下。

   ![img](https://img.kancloud.cn/3e/a9/3ea9ec35364a573c669f5f32c03c8b50_1344x1326.png)

4. 停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束。

### 缺点

1. STW：stop the world，让程序暂停，程序出现卡顿（重要问题）
2. 标记需要扫描整个heap，复杂度大
3. 清除数据会产生heap碎片

Go V1.3版本之前就是以上来实施的，流程是：![img](https://img.kancloud.cn/c7/da/c7da67305d321015d28af3f505ccc748_2426x578.png)

Go V1.3做了简单的优化，将STW提前，减少STW暂停的时间范围。如下所示：![img](https://img.kancloud.cn/7f/c9/7fc93a9ae9387d34e9843eb1edec31fe_2410x520.png)

**最重要的问题就是：mark-and-sweep 算法会暂停整个程序**

## 三色标记法

**Go 1.5 使用**

三色标记法。实际上就是通过三个阶段的标记来确定清楚的对象都有哪些。我们来看一下具体的过程。

### 流程：

1.  就是只要是新创建的对象，默认的颜色都是标记为“白色”。

   ![img](https://img.kancloud.cn/4a/0c/4a0c45a0aafa546feaab109dd6d97d89_2152x1364.png)

   这里面需要注意的是，所谓“程序”，则是一些对象的根节点集合。

   ![img](https://img.kancloud.cn/e3/a5/e3a5759be1646a805ca4a12b0fbadfaa_1920x1080.jpeg)

2. 每次GC回收开始，然后从根节点开始遍历所有对象，把遍历到的对象从白色集合放入“灰色”集合。只遍历第一个对象。

   ![img](https://img.kancloud.cn/47/e0/47e0df9bb3e6a8dbf2c067cf1458d6e6_1920x1080.jpeg)

3. 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合。![img](https://img.kancloud.cn/75/50/755096e23bf5b8110de33ae8899ab35f_1920x1080.jpeg)

4. 重复第三步，直到灰色中无任何对象。

   ![img](https://img.kancloud.cn/82/41/8241e5b771f6265d704220955531ecbd_1920x1080.jpeg)

   ![img](https://img.kancloud.cn/a9/e1/a9e16da6ef4eb3b5e9da9ba2e0387b16_1920x1080.jpeg)

5. 回收所有的白色标记表的对象，也就是回收垃圾。

#### 如果三色标记过程不启动STW

我们还是基于上述的三色并发标记法来说，他是一定要依赖STW的。因为如果不暂停程序，程序的逻辑改变对象引用关系，这种动作如果在标记阶段做了修改，会影响标记结果的正确性。我们举一个场景。

如果三色标记法，标记过程不使用STW将会发生什么事情？

![img](https://img.kancloud.cn/6b/18/6b18a939e13214cd648251520bdc146f_1920x1080.jpeg)

![img](https://img.kancloud.cn/fc/15/fc15a2549f89a685bd93ec96d9479468_1920x1080.jpeg)

![img](https://img.kancloud.cn/cc/be/ccbef3f78a00821cd6135b64ec0f96bd_1920x1080.jpeg)

![img](https://img.kancloud.cn/20/a0/20a03b3e350d754fd3e958a3a5634d52_1920x1080.jpeg)

![img](https://img.kancloud.cn/0d/a1/0da11e89ed4d4bfe80ac19a4afd0c680_1920x1080.jpeg)

可以看出，有两个问题，在**三色标记法不希望被发生**的

- 条件1：一个白色对象被黑色对象引用**(白色被挂在黑色下)**
- 条件2：灰色对象与它之间的可达关系的白色对象遭到破坏**(灰色同时丢了该白色)**

当以上两个条件同时满足时，就会出现对象丢失现象。

当然，如果上述中的白色对象3，如果他还有很多下游对象的话，也会一并都清理掉。

 为了防止这种现象的发生，最简单的方式就是STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是**STW的过程有明显的资源浪费，对所有的用户程序都有很大影响**，如何能在保证对象不丢失的情况下合理的尽可能的提高GC效率，减少STW时间呢？

答案就是，那么我们只要使用一个机制，来破坏上面的两个条件就可以了。

### 解决方法：

#### 强三色不变式

![img](https://img.kancloud.cn/40/dd/40dd8d5e63aa3b7ec4104d7da162178f_1920x1080.jpeg)

不存在黑色对象引用到白色对象的指针。

#### 弱三色不变式

![img](https://img.kancloud.cn/86/76/8676a065ee333c705a93e28362de9a17_1920x1080.jpeg)

所有被黑色对象引用的白色对象都处于灰色保护状态。

#### 屏障机制

回收时加上一定的判断，间接遏制三色标记问题

- 插入屏障——对象被引用时触发的机制

  **具体操作**：在A对象引用B对象的时候，B对象被标记为灰色。（将B挂在A下游，B必须被标记为灰色）

  **满足**：**强三色不变式**（不存在黑色对象引用白色对象的情况了， 因为白色会强制变成灰色）

  ```
  // 伪代码
  添加下游对象(当前下游对象slot, 新下游对象ptr) {   
    //1
    标记灰色(新下游对象ptr)   
    
    //2
    当前下游对象slot = 新下游对象ptr  				  
  }
  ```

  **场景**：

  ```
  A.添加下游对象(nil, B)   //A 之前没有下游，新添加一个下游对象B，B被标记为灰色
  A.添加下游对象(C, B)     //A 将下游对象C更换为B，B被标记为灰色
  ```

  我们知道，黑色对象的内存槽有两种位置，`栈`和`堆`。栈空间的特点是容量小，但是要求相应速度快，因为函数调用弹出频繁使用，所以“插入屏障”机制在**栈空间的对象操作中不使用**，而仅仅使用在堆空间对象的操作中。

  ![img](https://img.kancloud.cn/16/57/16572fc059aeafe81256ec0922c6189e_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/de/ad/dead5c7327aa36a9dd6491fcd8ae75be_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/29/42/294216ca5997f0df13b621781a47cd24_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/62/c3/62c363973c3baf17dee6871b8fd5fd79_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/54/57/545783724293dc5769123f2ead384eda_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/b3/53/b3536074823deff4ee9a0d50706c2caf_1920x1080.jpeg)

  但是如果栈不添加，当全部三色标记扫描之后，栈上有可能依然存在白色对象被引用的情况(如上图的对象9)。所以要对栈重新进行三色标记扫描，但这次为了对象不丢失，要对本次标记扫描启动STW暂停。直到栈空间的三色标记结束。

  ![img](https://img.kancloud.cn/4a/24/4a2463054b2f336d5f1ee08409e32f11_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/89/5e/895ea8ca38e0c80f8dc8e5f6445c207f_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/9c/c7/9cc7fd99761d60d386d2ca87d3a01fbd_1920x1080.jpeg)

  最后将栈和堆空间扫描剩余的全部白色节点清除，这次STW大约的时间在10~100ms间

  ![img](https://img.kancloud.cn/58/cb/58cb90c72f84312af826b22fc3cbbb15_1920x1080.jpeg)

- 删除屏障，对象被删除时触发的机制

  **具体操作**：被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。

  **满足**：**弱三色不变式**（保护灰色对象到白色对象的路径不会断）

  ```
  // 伪代码
  添加下游对象(当前下游对象slot， 新下游对象ptr) {
    //1
    if (当前下游对象slot是灰色 || 当前下游对象slot是白色) {
    		标记灰色(当前下游对象slot)     //slot为被删除对象， 标记为灰色
    }
    
    //2
    当前下游对象slot = 新下游对象ptr
  }
  ```

  **场景**：

  ```
  A.添加下游对象(B, nil)   //A对象，删除B对象的引用。B被A删除，被标记为灰(如果B之前为白)
  A.添加下游对象(B, C)		//A对象，更换下游B变成C。B被A删除，被标记为灰(如果B之前为白)
  ```

  ![img](https://img.kancloud.cn/65/f2/65f2b58b0b3a1b20f26dcde525315599_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/d2/f2/d2f2a76d2aaf5c16cf9b7c094073fbbc_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/dc/78/dc7866c2f884a1c245630c3ed91644e5_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/c2/f0/c2f05206cd9ae498025973c8bc763daa_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/a8/54/a8541799ee4f9e598bef49136d448ade_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/fc/17/fc176d88b2eab093ebd5aee643e0677a_1920x1080.jpeg)

  ![img](https://img.kancloud.cn/8e/d3/8ed3690aa81a7ee78a1ce739c0adab38_1920x1080.jpeg)

  这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。

## 三色标记法 + 混合写屏障机制

**Go 1.8 使用**

插入写屏障和删除写屏障的短板：

- 插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
- 删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

**具体操作**：

1. GC开始将栈上的对象全部扫描并标记为黑色（之后不再进行第二次重复扫描，无需STW），

2. GC期间，任何在栈上创建的新对象，均为黑色。

3. 被删除的对象标记为灰色。

4. 被添加的对象标记为灰色。

**满足**：变形的**弱三色不变式**

```
// 伪代码
添加下游对象(当前下游对象slot, 新下游对象ptr) {
  	//1 
	标记灰色(当前下游对象slot)    //只要当前下游对象被移走，就标记灰色
  	
  	//2 
  	标记灰色(新下游对象ptr)
  		
  	//3
  	当前下游对象slot = 新下游对象ptr
}
```

> 这里我们注意， 屏障技术是不在栈上应用的，因为要保证栈的运行效率。

### 流程

![img](https://img.kancloud.cn/45/2c/452c55637b22078abad29786241d5000_1920x1080.jpeg)

![img](https://img.kancloud.cn/42/aa/42aa1f73230061792851a43ce495acb6_1920x1080.jpeg)

#### 场景1：对象被一个堆对象删除引用，成为栈对象的下游

```
//前提：堆对象4->对象7 = 对象7；  //对象7 被 对象4引用
栈对象1->对象7 = 堆对象7；  //将堆对象7 挂在 栈对象1 下游
堆对象4->对象7 = null；    //对象4 删除引用 对象7
```

对象7由对象4下游变为对象1下游

![img](https://img.kancloud.cn/64/c7/64c76eea3706c37f160b8345b7b3742c_1920x1080.jpeg)

![img](https://img.kancloud.cn/4d/67/4d6728d276d2786017cde37b824333aa_1920x1080.jpeg)

#### 场景2：对象被一个栈对象删除引用，成为另一个栈对象的下游

```
new 栈对象9；
对象8->对象3 = 对象3；      //将栈对象3 挂在 栈对象9 下游
对象2->对象3 = null；      //对象2 删除引用 对象3
```

![img](https://img.kancloud.cn/be/ed/beedb81ec3cd5a4813aaa5bce1341949_1920x1080.jpeg)

![img](https://img.kancloud.cn/48/56/48569d6dfb8ac6f1b0d6238a9d8150b3_1920x1080.jpeg)

![img](https://img.kancloud.cn/46/e6/46e6be62e880e0f5796bc1e6f050b512_1920x1080.jpeg)

#### 场景3：对象被一个堆对象删除引用，成为另一个堆对象的下游

```
堆对象10->对象7 = 堆对象7；       //将堆对象7 挂在 堆对象10 下游
堆对象4->对象7 = null；         //对象4 删除引用 对象7
```

![img](https://img.kancloud.cn/a6/b7/a6b76e3f99029e603dbfe49fc7da30e8_1920x1080.jpeg)

![img](https://img.kancloud.cn/d0/1e/d01e30f003f4a40e439d1a68ced89f34_1920x1080.jpeg)

![img](https://img.kancloud.cn/ef/af/efaf7b7e32498db84eea797ed11201bf_1920x1080.jpeg)

#### 场景4：对象从一个栈对象删除引用，成为另一个堆对象的下游

```
栈对象1->对象2 = null；       //对象1 删除引用 堆对象2
堆对象4->对象2 = 栈对象2；     //对象4 添加下游 栈对象2
堆对象4->对象7 = null；       //对象4 删除引用 对象7
```

![img](https://img.kancloud.cn/a3/a7/a3a7d82de782d14d28fa5999b7d5b36d_1920x1080.jpeg)

![img](https://img.kancloud.cn/17/9c/179c86e25de0f0d0dbb24f371229d19d_1920x1080.jpeg)

![img](https://img.kancloud.cn/7a/cb/7acb9b30746955ae0467ca2871a69e01_1920x1080.jpeg)

## 总结

GoV1.3-普通标记清除法，整体过程需要启动STW，效率极低

GoV1.5-三色标记法，堆空间启动写屏障，栈空间不启动，全部扫描之后，需要重新扫描一次栈(需要STW)，效率普通

GoV1.8-三色标记法，混合写屏障机制，栈空间不启动，堆空间启动。整个过程几乎不需要STW，效率较高