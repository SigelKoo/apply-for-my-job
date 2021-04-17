# 重建二叉树

输入某二叉树的前序遍历和中序遍历的结果，请重建该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。

例如，给出

```
前序遍历 preorder = [3,9,20,15,7]
中序遍历 inorder = [9,3,15,20,7]
```

返回如下的二叉树：

```
    3
   / \
  9  20
    /  \
   15   7
```

递归：

```go
func buildTree(preorder []int, inorder []int) *TreeNode {
	if len(preorder) == 0 || len(inorder) == 0 {
		return nil
	}
	root := TreeNode{Val: preorder[0], Left: nil, Right: nil}
	index := 0
	for i := 0; i < len(inorder); i++ {
		if preorder[0] == inorder[i] {
			index = i
			break
		}
	}
	root.Left = buildTree(preorder[1:index + 1], inorder[0:index])
	root.Right = buildTree(preorder[index + 1:], inorder[index + 1:])
	return &root
}
```

