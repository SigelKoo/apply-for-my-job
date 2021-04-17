输入一棵二叉树的根节点，求该树的深度。从根节点到叶节点依次经过的节点（含根、叶节点）形成树的一条路径，最长路径的长度为树的深度。

给定二叉树 `[3,9,20,null,null,15,7]`，

```
    3
   / \
  9  20
    /  \
   15   7
```

```go
var max int

func maxDepth(root *TreeNode) int {
	max = 0
	dfs(root, 1)
	return max
}

func dfs(root *TreeNode, i int) {
	if root == nil {
		return
	}
	
	if max < i {
		max = i
	}

	dfs(root.Left, i + 1)
	dfs(root.Right, i + 1)
}
```

