# 链表倒数第K个数

实现一种算法，找出单向链表中倒数第 k 个节点。返回该节点的值。

```
输入： 1->2->3->4->5 和 k = 2
输出： 4
```

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func kthToLast(head *ListNode, k int) int {

}
```


我的解法（反转链表，然后顺序拿数据）
```go
func kthToLast(head *ListNode, k int) int {
    tail := reserveList(head);
    var res int
    for i := 0; i < k; i++ {
        if i == k - 1 {
            return tail.Val
        }
        tail = tail.Next
    }
    return res
}

func reserveList(head *ListNode) *ListNode {
    var prev *ListNode
    for head != nil {
        head.Next, head, prev = prev, head.Next, head
    }
    return prev
}
```

双指针解法

```go
func kthToLast(head *ListNode, k int) int {
	fast := head
	slow := head
	for k > 0 {
		fast = fast.Next
		k--
	}
	for fast != nil {
		slow = slow.Next
		fast = fast.Next
	}
	return slow.Val
}
```

