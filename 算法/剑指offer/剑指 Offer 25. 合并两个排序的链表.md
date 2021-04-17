输入两个递增排序的链表，合并这两个链表并使新链表中的节点仍然是递增排序的。

```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

```go
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    res := &ListNode{Val : 0, Next : nil}
    p := res
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            p.Next = l1
            l1 = l1.Next
        } else {
            p.Next = l2
            l2 = l2.Next
        }
        p = p.Next
    }
    if l1 == nil {
        p.Next = l2
    }
    if l2 == nil {
        p.Next = l1
    }
    return res.Next
}
```

