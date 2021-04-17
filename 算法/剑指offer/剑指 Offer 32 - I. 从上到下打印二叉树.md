从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

给定二叉树: `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

```
[3,9,20,15,7]
```

```go
var arr32 [][]int

func levelOrder(root *TreeNode) []int {
	arr32 = [][]int{}
	dfs(root, 0)
	res := []int{}
	for i := 0; i < len(arr32); i++ {
		for j := 0; j < len(arr32[i]); j++ {
			res = append(res, arr32[i][j])
		}
	}
	return res
}

func dfs(root *TreeNode, level int)  {
	if root == nil {
		return
	}
	if level == len(arr32) {
		arr32 = append(arr32, []int{})
	}
	arr32[level] = append(arr32[level], root.Val)
	dfs(root.Left, level + 1)
	dfs(root.Right, level + 1)
}

```

