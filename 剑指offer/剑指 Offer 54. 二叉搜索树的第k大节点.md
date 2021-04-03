给定一棵二叉搜索树，请找出其中第k大的节点。

```
输入: root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
输出: 4
```

```go
func kthLargest(root *TreeNode, k int) int {
	res := []int{}
	dfs54(root, &res)
	return res[k - 1]
}

func dfs54(root *TreeNode, res *[]int) {
	if root.Right != nil {
		dfs54(root.Right, res)
	}
	if root != nil {
		*res = append(*res, root.Val)
	}
	if root.Left != nil {
		dfs54(root.Left, res)
	}
}
```

