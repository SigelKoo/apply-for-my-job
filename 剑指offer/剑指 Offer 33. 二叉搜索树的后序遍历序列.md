输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 `true`，否则返回 `false`。假设输入的数组的任意两个数字都互不相同。

```
     5
    / \
   2   6
  / \
 1   3
```

```
输入: [1,6,3,2,5]
输出: false
```

```
输入: [1,3,2,6,5]
输出: true
```

```go
func verifyPostorder(postorder []int) bool {
	if len(postorder) <= 1 {
		return true
	}
	length := len(postorder) - 1

	root := postorder[length]
	index := 0
	for i, v := range postorder {
		if v >= root {
			index = i
			break
		}
	}

	leftArr := postorder[:index]
	rightArr := postorder[index:length]
	for _, v := range leftArr {
		if v > root {
			return false
		}
	}
	for _, v := range rightArr {
		if v < root {
			return false
		}
	}
	return verifyPostorder(leftArr) && verifyPostorder(rightArr)
}
```

