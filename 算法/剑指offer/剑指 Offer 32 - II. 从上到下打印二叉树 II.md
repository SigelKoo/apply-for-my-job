从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

给定二叉树: `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

```
[
  [3],
  [9,20],
  [15,7]
]
```

```go
var res [][]int

func levelOrder(root *TreeNode) [][]int {
    res = [][]int{}
    dfs(root, 0)
    return res
}

func dfs(root *TreeNode, level int) {
    if root == nil {
        return
    }

    if level == len(res) {
        res = append(res, []int{})
    }

    res[level] = append(res[level], root.Val)

    dfs(root.Left, level + 1)
    dfs(root.Right, level + 1)
}
```

