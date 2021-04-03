请完成一个函数，输入一个二叉树，该函数输出它的镜像。

```
     4
   /   \
  2     7
 / \   / \
1   3 6   9
```

```
     4
   /   \
  7     2
 / \   / \
9   6 3   1
```

```go
func mirrorTree(root *TreeNode) *TreeNode {
	dfs(root)
	return root
}

func dfs(root *TreeNode) {
	if root == nil {
		return
	}
	root.Left, root.Right = root.Right, root.Left
	dfs(root.Left)
	dfs(root.Right)
}
```

