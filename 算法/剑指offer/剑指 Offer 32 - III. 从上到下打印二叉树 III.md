请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。

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
  [20,9],
  [15,7]
]
```

```go
var res [][]int

func levelOrder(root *TreeNode) [][]int {
    res = [][]int{}
    dfs(root, 0)
    for i := 0; i < len(res); i++ {
        if i % 2 != 0 {
            reverse(res[i])
        }
    }
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

func reverse(arr []int) []int {
    for i, j := 0, len(arr) - 1; i < j; i, j = i + 1, j - 1 {
        arr[i], arr[j] = arr[j], arr[i]
    }
    return arr
}
```

