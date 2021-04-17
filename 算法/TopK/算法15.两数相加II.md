# 两数相加 II

给你两个非空链表来代表两个非负整数。数字最高位位于链表开始位置。它们的每个节点只存储一位数字。将这两数相加会返回一个新的链表。

你可以假设除了数字 0 之外，这两个数字都不会以零开头。

如果输入链表不能修改该如何处理？换句话说，你不能对列表中的节点进行翻转。

```
输入：(7 -> 2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 8 -> 0 -> 7
```

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {

}
```

对于逆序处理应该首先想到栈，用栈反转链表

```go
	var a, b []int
	for p := l1; p != nil; p = p.Next {
		a = append(a, p.Val)
	}
	for p := l2; p != nil; p = p.Next {
		b = append(b, p.Val)
	}
	var res *ListNode
	carry := 0
	aLen, bLen := len(a), len(b)
	for i, j := 0, 0; i < aLen || j < bLen; i, j = i + 1, j + 1 {
		sum := carry
		if i < aLen {
			sum += a[aLen - i - 1]
		}
		if j < bLen {
			sum += b[bLen - j - 1]
		}
		node := ListNode{Val: sum % 10, Next: nil}
		if res == nil {
			res = &node
		} else {
			node.Next = res
			res = &node
		}
		carry = sum / 10
	}
	if carry != 0 {
		res = &ListNode{Val: carry, Next: res}
	}
	return res
```

















