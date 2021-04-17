请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

例如，二叉树 [1,2,2,3,4,4,3] 是对称的。

```
    1
   / \
  2   2
 / \ / \
3  4 4  3
```

但是下面这个 [1,2,2,null,3,null,3] 则不是镜像对称的:

```
    1
   / \
  2   2
   \   \
   3    3
```

如果两个子树 p 和 q 是对称的，那么

p.Val == q.Val
p.Left 和 q.Right 是对称的
p.Right 和 q.Left 是对称的

(p.Val == q.Val) && _isSymmetric(p.Left, q.Right) && _isSymmetric(p.Right, q.Left)

中止条件：

p == nil && q == nil，返回 true
(p == nil && q != nil) || (p != nil && q == nil)，返回 false

```go
func isSymmetric(root *TreeNode) bool {
	if root == nil {
		return true
	}
	return _isSymmetric(root.Left, root.Right)
}

func _isSymmetric(p *TreeNode, q *TreeNode) bool {
	if p == nil && q == nil {
		return true
	}
	if (p == nil && q != nil) || (p != nil && q == nil) {
		return false
	}
	return p.Val == q.Val && _isSymmetric(p.Left, q.Right) && _isSymmetric(p.Right, q.Left)
}

```

