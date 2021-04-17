请实现 copyRandomList 函数，复制一个复杂链表。在复杂链表中，每个节点除了有一个 next 指针指向下一个节点，还有一个 random 指针指向链表中的任意节点或者 null。

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2020/01/09/e1.png)

```
输入：head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
输出：[[7,null],[13,0],[11,4],[10,2],[1,0]]
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2020/01/09/e2.png)

```
输入：head = [[1,1],[2,1]]
输出：[[1,1],[2,1]]
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2020/01/09/e3.png)

```
输入：head = [[3,null],[3,0],[3,null]]
输出：[[3,null],[3,0],[3,null]]
```

```
输入：head = []
输出：[]
解释：给定的链表为空（空指针），因此返回 null。
```

```go
func copyRandomList(head *Node) *Node {
	maps := make(map[*Node]*Node)
	curr := head
	for curr != nil {
		temp := &Node{
			Val: curr.Val,
		}
		maps[curr] = temp
		curr = curr.Next
	}
	curr = head
	for curr != nil {
		maps[curr].Next = maps[curr.Next]
		maps[curr].Random = maps[curr.Random]
		curr = curr.Next
	}
	return maps[head]
}
```

