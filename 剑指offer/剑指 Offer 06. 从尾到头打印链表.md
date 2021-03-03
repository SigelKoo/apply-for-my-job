# 从尾到头打印链表

输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

```
输入：head = [1,3,2]
输出：[2,3,1]
```

1. 从链表中顺序取出放入数组，数组倒序返回

```go
func reversePrint(head *ListNode) []int {
	var a, b []int
	for head != nil {
		a = append(a, head.Val)
		head = head.Next
	}
	for i := len(a) - 1; i >= 0; i-- {
		b = append(b, a[i])
	}
	return b
}
```

2. 先将链表倒序，再顺序输出到数组中

```go
func reversePrint(head *ListNode) []int {
	a := reverseList(head)
	var b []int
	for a != nil {
		b = append(b, a.Val)
        a = a.Next
	}
	return b
}

func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	for head != nil {
		head.Next, head, prev = prev, head.Next, head
	}
	return prev
}
```

3. 递归法

```go
func reversePrint(head *ListNode) []int {
	if head == nil {
		return []int{}
	}
	res := reversePrint(head.Next)
	return append(res, head.Val)
}
```

4. 栈，递归在运行的时候就是存储在栈中