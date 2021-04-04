# 从尾到头打印链表

输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

```
输入：head = [1,3,2]
输出：[2,3,1]
```

```go
func reversePrint(head *ListNode) []int {
    arr := []int{}
	res := []int{}
	for head != nil {
		arr = append(arr, head.Val)
		head = head.Next
	}
	for i := len(arr) - 1; i >= 0; i-- {
		res = append(res, arr[i])
	}
	return res
}
```

