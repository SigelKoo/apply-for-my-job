在一个由 `'0'` 和 `'1'` 组成的二维矩阵内，找到只包含 `'1'` 的最大正方形，并返回其面积。

![img](https://assets.leetcode.com/uploads/2020/11/26/max1grid.jpg)

```
输入：matrix = [["1","0","1","0","0"],["1","0","1","1","1"],["1","1","1","1","1"],["1","0","0","1","0"]]
输出：4
```

![img](https://assets.leetcode.com/uploads/2020/11/26/max2grid.jpg)

```
输入：matrix = [["0","1"],["1","0"]]
输出：1
```

```
输入：matrix = [["0"]]
输出：0
```

```go
func maximalSquare(matrix [][]byte) int {
    maxSize := 0
    for i := 0; i < len(matrix); i++ {
        for j := 0; j < len(matrix[0]); j++ {
            if matrix[i][j] == '1' {
                maxSize = max(find(matrix, i, j), maxSize)
            }
        }
    }
    return maxSize * maxSize
}

func find(matrix [][]byte, m, n int) int {
    size := 1
    a := m + size
    b := n + size
    for a < len(matrix) && b < len(matrix[0]) {
        for i := m; i <= a; i++ {
            if matrix[i][b] == '0' {
                return size
            }
        }
        for i := n; i <= b; i++ {
            if matrix[a][i] == '0' {
                return size
            }
        }
        size++
        a++
        b++
    }
    return size
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

