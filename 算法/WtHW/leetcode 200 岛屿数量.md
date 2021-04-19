给你一个由 '1'（陆地）和 '0'（水）组成的的二维网格，请你计算网格中岛屿的数量。

岛屿总是被水包围，并且每座岛屿只能由水平方向和/或竖直方向上相邻的陆地连接形成。

此外，你可以假设该网格的四条边均被水包围。

```
输入：grid = [
  ["1","1","1","1","0"],
  ["1","1","0","1","0"],
  ["1","1","0","0","0"],
  ["0","0","0","0","0"]
]
输出：1
```

```
输入：grid = [
  ["1","1","0","0","0"],
  ["1","1","0","0","0"],
  ["0","0","1","0","0"],
  ["0","0","0","1","1"]
]
输出：3
```

```go
func numIslands(grid [][]byte) int {
    m, n := len(grid), len(grid[0])
    visted := make([][]bool, m)
    for i := range visted {
        visted[i] = make([]bool, n)
    }
    res := 0
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' && !visted[i][j] {
                searchGrid(grid, visted, i, j)
                res++
            }
        }
    }
    return res
}

func searchGrid(grid [][]byte, visted [][]bool, m, n int) {
    if m >= len(grid) || n >= len(grid[0]) || m < 0 || n < 0 {
        return
    }
    if visted[m][n] {
        return
    }
    if grid[m][n] != '1' {
        return
    } else {
        visted[m][n] = true
        searchGrid(grid, visted, m, n + 1)
        searchGrid(grid, visted, m, n - 1)
        searchGrid(grid, visted, m + 1, n)
        searchGrid(grid, visted, m - 1, n)
    }
}
```

