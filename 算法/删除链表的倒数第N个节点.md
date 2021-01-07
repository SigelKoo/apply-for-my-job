# 删除链表的倒数第N个节点

给定一个链表，删除链表的倒数第n个节点，并且返回链表的头结点。

```
给定一个链表: 1->2->3->4->5, 和 n = 2.

当删除了倒数第二个节点后，链表变为 1->2->3->5.
```

自己的方法（时间空间都100）
```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func removeNthFromEnd(head *ListNode, n int) *ListNode {
	a := reverseList(head)
    a = deleteNode(a, n)
	return reverseList(a)
}

func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	for head != nil {
		head.Next, head, prev = prev, head.Next, head
	}
	return prev
}

func deleteNode(head *ListNode, n int) *ListNode {
	if head == nil {
		return head
	}
	i := 0
	newHead := &ListNode{0,head}
	pre, cur := newHead, head
	for cur != nil {
		if i == n - 1 {
			pre.Next = cur.Next
			break
		}
		pre = cur
		cur = cur.Next
		i++
	}

	return newHead.Next
}
```

标准题解

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{0, head}
    first, second := head, dummy
    for i := 0; i < n; i++ {
        first = first.Next
    }
    for ; first != nil; first = first.Next {
        second = second.Next
    }
    second.Next = second.Next.Next
    return dummy.Next
}
```

