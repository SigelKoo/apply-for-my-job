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
	for k := range inorder {
		if preorder[0] == inorder[k] {
			return &TreeNode{
				Val: preorder[0],
				Left: buildTree(preorder[1 : k + 1], inorder[0 : k]),
				Right: buildTree(preorder[k + 1 :], inorder[k + 1 :]),
			}
		}
	}
	return nil
}
```

