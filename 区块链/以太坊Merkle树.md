# Merkle树

二叉树，一个根节点，一组中间节点与一组叶子节点组成。

特点：

- 最下面的叶节点包含存储数据或其哈希值
- 非叶子节点（包括中间节点和根节点）都是它的两个孩子节点内容的哈希值

默克尔树逐层记录哈希值的特点，让它具有了一些独特的性质。例如，底层数据的任何变动，都会传递到其父节点，一层层沿着路径一直到树根。这意味树根的值实际上代表了对底层所有数据的“数字摘要”。

### 作用

1. **证明某个集合中存在或不存在某个元素**：通过构建集合的默克尔树，并提供该元素各级兄弟节点中的Hash 值，可以不暴露集合完整内容而证明某元素存在。

   另外，对于可以进行排序的集合，可以将不存在元素的位置用空值代替，以此构建稀疏默克尔树（Sparse Merkle Tree）。该结构可以证明某个集合中不包括指定元素。

2. **快速比较大量数据**：对每组数据排序后构建默克尔树结构。当两个默克尔树根相同时，则意味着所代表的两组数据必然相同。否则，必然不同。

   由于 Hash 计算的过程可以十分快速，预处理可以在短时间内完成。利用默克尔树结构能带来巨大的比较性能优势。

3. **快速定位修改**：以下图为例，基于数据 D0……D3 构造默克尔树，如果 D1 中数据被修改，会影响到 N1，N4 和 Root。

   ![img](https://files.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-M5xTVjj6plOWgHcmTHq%2Fsync%2F01dd48b2fe29d3d5ba879d8eddfe6db037b13596.png?generation=1588030734755898&alt=media)

   因此，一旦发现某个节点如 Root 的数值发生变化，沿着 Root --> N4 --> N1，最多通过 O(lgN) 时间即可快速定位到实际发生改变的数据块 D1。

### 优势

- 快速重哈希

Merkle tree的特点之一就是当树节点内容发生变化时，能够在前一次哈希计算的基础上，仅仅将被修改的树节点进行哈希重计算，便能得到一个新的根哈希用来代表整棵树的状态。

- 轻节点扩展

采用Merkle tree，可以在公链环境下扩展一种“轻节点”。轻节点的特点是对于每个区块，仅仅需要存储约80个字节大小的区块头数据，而不存储交易列表，回执列表等数据。然而通过轻节点，可以实现在非信任的公链环境中验证某一笔交易是否被收录在区块链账本的功能。这使得像比特币，以太坊这样的区块链能够运行在个人PC，智能手机等拥有小存储容量的终端上。

对于轻节点来说，验证一条交易只需要验证包含该交易的路径即可，并不需要把所有交易的Hash全部重新算一遍。

![img](https://pic1.zhimg.com/80/v2-1acfb4f6da08977b33ee75e349f83d88_720w.jpg)

# Merkle Patricia Trie

融合了Merkle树和前缀树两种树结构优点的数据结构，是以太坊中用来组织管理账户数据、生成交易集合哈希的重要数据结构。

## Trie

前缀树或字典树，是一种有序树，用于保存关联数组。其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定 。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。

一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。实际上trie每个节点是一个确定长度的数组，数组中每个节点的值是一个指向子节点的指针，最后有个标志域，标识这个位置为止是否是一个完整的字符串。

常见的用来存英文单词的trie每个节点是一个长度为27的指针数组，index0-25代表a-z字符，26为标志域。如图：

![img](https://pic4.zhimg.com/80/v2-07a263568193a61ed457ea441a69513f_720w.jpg)

### 优势

相比于哈希表，使用前缀树来进行查询拥有共同前缀key的数据时十分高效，例如在字典中查找前缀为pre的单词，对于哈希表来说，需要遍历整个表，时间效率为O(n)，然而对于前缀树来说，只需要在树中找到前缀为pre的节点，且遍历以这个节点为根节点的子树即可。

但是对于最差的情况（前缀为空串)，时间效率为O(n)，仍然需要遍历整棵树，此时效率与哈希表相同。

相比于哈希表，在前缀树不会存在哈希冲突的问题。

### 劣势

- 直接查找效率低下。前缀树的查找效率是O(m)，m为所查找节点的key长度，而哈希表的查找效率为O(1)。且一次查找会有m次IO开销，相比于直接查找，无论是速率、还是对磁盘的压力都比较大。
- 可能会造成空间浪费。当存在一个节点，其key值内容很长（如一串很长的字符串），当树中没有与他相同前缀的分支时，为了存储该节点，需要创建许多非叶子节点来构建根节点到该节点间的路径，造成了存储空间的浪费。

## Patricia Trie

一种更节省空间的Trie。对于基数树的每个节点，如果该节点是唯一的儿子的话，就和父节点合并。

![img](https://pic1.zhimg.com/80/v2-932c5c442939901fa4259789029eb190_720w.jpg)

## Merkle Patricia Trie

- 空节点：简单的表示空，在代码中是一个空串。
- 叶子节点：表示为 [key,value]的一个键值对，其中key是key的一种特殊十六进制编码（MP编码）， value是value的RLP编码。
- 分支节点：因为MPT树中的key被编码成一种特殊的16进制的表示，再加上最后的value，所以分支节点是一个长度为17的list， 前16个元素对应着key中的16个可能的十六进制字符 ， 如果有一个[key,value]对在这个分支节点终止，最后一个元素代表一个值 ，即分支节点既可以搜索路径的终止也可以是路径的中间节点。
- 扩展节点：也是[key，value]的一个键值对，但是这里的value是其他节点的hash值，这个hash可以被用来查询数据库中的节点。也就是说通过hash链接到其他节点。

### 三种编码格式

1. Raw编码（原生的字符）；
2. Hex编码（扩展的16进制编码）；
3. Hex-Prefix编码（16进制前缀编码）；

#### Raw编码

Raw编码就是原生的key值，不做任何改变。这种编码方式的key，是MPT对外提供接口的默认编码方式。

例如：一条key为"cat"，value为"dog"的数据项，其key的Raw编码就是['c', 'a', 't']，换成ASCII表示方式就是[63, 61, 74]（Hex）

#### Hex编码

Hex编码就是把一个8位的字节数据用两个十六进制数展示出来，编码时，将8位二进制码重新分组成两个4位的字节，其中一个字节的低4位是原字节的高四位，另一个字节的低4位是原数据的低4位，高4位都补0，然后输出这两个字节对应十六进制数字作为编码。Hex编码后的长度是源数据的2倍。

ASCII码：A (65)；二进制码：0100_0001；重新分组：0000_0100 0000_0001；十六进制： 4 1；Hex编码：41

若该Key对应的节点存储的是真实的数据项内容（即该节点是叶子节点），则在末位添加一个ASCII值为16的字符作为terminator；

若该key对应的节点存储的是另外一个节点的哈希索引（即该节点是扩展节点），则不加任何字符；

['c','a','t'] -> [6,3,6,1,7,4,**16**]

#### HP编码

目的：

- 区分叶子节点和扩展节点
- 把奇数路径变成偶数路径

步骤：

- 如果有terminator（16）那么就去掉terminator。
- 根据表格给key加上prefix

```text
node type    path length    |    prefix    hexchar
--------------------------------------------------
extension    even           |    0000      0x0
extension    odd            |    0001      0x1
leaf         even           |    0010      0x2
leaf         odd            |    0011      0x3
```

如果prefix是0x0或者0x2，加一个padding nibble 0 在prefix后面，所以最终应该是 0x00 和 0x20。原因是为了保证key（path）的长度为偶数。

例子：末尾的字符“16”说明该节点为叶子结点，并且加上了0x20

```text
[ 0, f, 1, c, b, 8, 16] -> '20 0f 1c b8'
```

#### 编码转换关系

- Raw编码：原生的key编码，是MPT对外提供接口中使用的编码方式，当数据项被插入到树中时，Raw编码被转换成Hex编码；
- Hex编码：16进制扩展编码，用于对内存中树节点key进行编码，当树节点被持久化到数据库时，Hex编码被转换成HP编码；
- HP编码：16进制前缀编码，用于对数据库中树节点key进行编码，当树节点被加载到内存时，HP编码被转换成Hex编码；

![img](https://pic3.zhimg.com/80/v2-232537cfac5103618cb9b0b45e9a47e2_720w.jpg)

### MPT流程

MPT树的特点如下:

- 叶子节点和分支节点可以保存value，扩展节点保存key；
- 没有公共的key就成为2个叶子节点；key1=[1,2,3] key2=[2,2,3]
- 有公共的key需要提取为一个扩展节点；key1=[1,2,3] key2=[1,3,3] => ex-node=[1],下一级分支node的key
- 如果公共的key也是一个完整的key，数据保存到下一级的分支节点中；key1=[1,2] key2=[1,2,3] =>ex-node=[1,2],下一级分支node的key; 下一级分支=[3],上一级key对应的value

![img](https://pic4.zhimg.com/80/v2-e2698781750973ae4a083f3dbe8bb517_720w.jpg)

过程：

```text
key  	 |  values
----------------------
a711355  |  45.0 ETH
a77d337  |  1.00 WEI
a7f9365  |  1.1  ETH
a77d397  |  0.12 ETH
```

**插入第一个<a711355, 45>，由于只有一个key，直接用leaf node既可表示**

![img](https://pic2.zhimg.com/80/v2-ed1f460bb6353d5817ce4cb24a6d5d3d_720w.jpg)

**接着插入a77d337，由于和a711355共享前缀'a7'，因而可以创建'a7'扩展节点。**

![img](https://pic3.zhimg.com/80/v2-b8117db4cafb85f52e840899ff22241e_720w.jpg)

**接着插入a7f9365，也是共享'a7'，只需新增一个leaf node**

![img](https://pic2.zhimg.com/80/v2-fb9cace75809b83abb90c79957a6d675_720w.jpg)

**最后插入a77d397，这个key和a77d337共享'a7'+'d3'，因而再需要创建一个'd3'扩展节点**

![img](https://pic2.zhimg.com/80/v2-4fcf5ab87c1b497a34f9e113a3da5be9_720w.jpg)

**将叶子节点和最后的short node合并到一个节点了，事实上源码实现需要再深一层，最后一层的叶子节点只有数据**

![img](https://pic1.zhimg.com/80/v2-345a4e2896e9605aea982b206dcbd940_720w.jpg)

```go
// nodeFlag contains caching-related metadata about a node.
type nodeFlag struct {
    hash  hashNode // cached hash of the node (may be nil)
    gen   uint16   // cache generation counter
    dirty bool     // whether the node has changes that must be written to the database
}
```

**MPT节点有个flag字，nodeFlag，记录了一些辅助数据：**

- 节点哈希：若该字段不为空，则当需要进行哈希计算时，可以跳过计算过程而直接使用上次计算的结果（当节点变脏时，该字段被置空）；
- 诞生标志：当该节点第一次被载入内存中（或被修改时），会被赋予一个计数值作为诞生标志，该标志会被作为节点驱除的依据，清除内存中“太老”的未被修改的节点，防止占用的内存空间过多；
- 脏标志：当一个节点被修改时，该标志位被置为1；

**flag.hash会保存该节点采用merkle tree类似算法生成的hash。同时会将hash和源数据以<hash, node.rlp.rawdata>方式保存在leveldb数据库中。这样后面通过hash就可以反推出节点数据。具体结构如下(蓝色的hash部分就是flag.hash字段)**

![img](https://pic4.zhimg.com/80/v2-7d1caad737c77663a8ae43611eff29af_720w.jpg)

#### 核心思想

hash可以还原出节点上的数据，这样只需要保存一个root(hash)，即可还原出完整的树结构，同时还可以按需展开节点数据，比如如果只需要访问<a771355, 45>这个数据，只需展开h00, h10, h20, h30这四个hash对应的节点

# 时间复杂度分析

Trie树的时间复杂度是  O(m) (m 为字符串长度) 

我们假设插入一个节点需要 1 个单位时间

增加一个分支节点的过程会将一个叶子节点拆分成一个分支节点和两个叶子节点，并将分支节点的索引当作value值存储起来

又因为所有的key都被转换成hex进制编码并作为树的路径

分支节点的概率会很高 （1 / 16）

插入一个分支节点的开销 ->  两个叶子节点的开销  2 +  分支节点1个字符的对应数组

时间消耗变成了跟树的高度有关系

 

​        ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/6/1714e6322fa1e15c~tplv-t2oaga2asx-watermark.awebp)

插入节点的时间复杂度为

T(n) = 1 + 2 + 2^2 +  2^4 +  2^(n - 1) = ((1 + 2^n - 1) * n) / 2 = 2 ^ n - 1

公式推算: 

n = 1

S1 = 1 + 2 + 2^2 + 2^3 .... 2^k-1

S2 =       2 + 2^2 + 2^3 ..... 2^k-1 + 2^k

S2 - S1 = 2^k - 1

n = 2^k - 1

k = log(n)

MPT 时间复杂度 O(log(n))

# 源码

```
type (
	// 分支节点，它的结构体现了原生trie的设计特点
	fullNode struct {
		Children [17]node // 17个子节点，其中16个为0x0-0xf;第17个子节点存放数据
		flags    nodeFlag // 缓存节点的Hash值，同时标记dirty值来决定节点是否必须写入数据库
	}
	// 扩展节点和叶子节点，它的结构体现了PatriciaTrie的设计特点
	// 区别在于扩展节点的value指向下一个节点的hash值(hashNode)；叶子节点的value是数据的RLP编码(valueNode)
	shortNode struct {
		Key   []byte
		Val   node
		flags nodeFlag
	}
	// 节点哈希，用于实现节点的折叠
	hashNode  []byte
	// 存储数据
	valueNode []byte
)
```

首先看MPT树新建：

```go
func New(root common.Hash, db *Database) (*Trie, error) {
	if db == nil {
		panic("trie.New called without a database")
	}
	trie := &Trie{
		db: db,
	}
	// 如果根哈希不为空，说明是从数据库加载一个已经存在的MPT树
	if root != (common.Hash{}) && root != emptyRoot {
		rootnode, err := trie.resolveHash(root[:], nil)
		if err != nil {
			return nil, err
		}
		trie.root = rootnode
	}
    // 否则，直接返回的是新建的MPT树
	return trie, nil
}
```

MPT树的插入：

```go
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
	if len(key) == 0 {
		if v, ok := n.(valueNode); ok {
			return !bytes.Equal(v, value.(valueNode)), value, nil
		}
		return true, value, nil
	}
	switch n := n.(type) {
	case *shortNode:
        // 如果是叶子节点，首先计算共有前缀
		matchlen := prefixLen(key, n.Key)
        // 1.1如果共有前缀和当前的key一样，说明节点已经存在  只更新节点的value即可
		if matchlen == len(n.Key) {
			dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
			if !dirty || err != nil {
				return false, n, err
			}
			return true, &shortNode{n.Key, nn, t.newFlag()}, nil
		}
		// 1.2构造形成一个分支节点(fullNode)
		branch := &fullNode{flags: t.newFlag()}
		var err error
        // 1.3将原来的节点拆作新的后缀shortNode插入
		_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
		if err != nil {
			return false, nil, err
		}
        // 1.4将新节点作为shortNode插入
		_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
		if err != nil {
			return false, nil, err
		}
		// 1.5如果没有共有的前缀，则新建的分支节点为根节点
		if matchlen == 0 {
			return true, branch, nil
		}
		// 1.6如果有共有的前缀，则拆分原节点产生前缀叶子节点为根节点
		return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil

	case *fullNode:
        // 2如果是分支节点，则直接将新数据插入作为子节点
		dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
		if !dirty || err != nil {
			return false, n, err
		}
		n = n.copy()
		n.flags = t.newFlag()
		n.Children[key[0]] = nn
		return true, n, nil

	case nil:
        // 3空节点，直接返回该值得叶子节点作为根节点
		return true, &shortNode{key, value, t.newFlag()}, nil

	case hashNode:
        // 4.1哈希节点 表示当前节点还未加载到内存中，首先需要调用resolveHash从数据库中加载节点
		rn, err := t.resolveHash(n, prefix)
		if err != nil {
			return false, nil, err
		}
        // 4.2然后在该节点后插入新节点
		dirty, nn, err := t.insert(rn, prefix, key, value)
		if !dirty || err != nil {
			return false, rn, err
		}
		return true, nn, nil

	default:
		panic(fmt.Sprintf("%T: invalid node: %v", n, n))
	}
}
```

