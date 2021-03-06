# 机器人的运动范围

地上有一个m行n列的方格，从坐标 [0,0] 到坐标 [m-1,n-1] 。一个机器人从坐标 [0, 0] 的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），也不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格 [35, 37] ，因为3+5+3+7=18。但它不能进入方格 [35, 38]，因为3+5+3+8=19。请问该机器人能够到达多少个格子？

```
输入：m = 2, n = 3, k = 1
输出：3
```

```
输入：m = 3, n = 1, k = 0
输出：1
```

解法：

```
func movingCount(m int, n int, k int) int {
    arr := make([][]int, m + 1)
    for i := range arr {
        arr[i] = make([]int, n + 1)
    }
    return dfs(m, n, 0, 0, k, arr)
}

func dfs(m, n, i, j, k int, arr [][]int) int {
    if i < 0 || j < 0 || i >= m || j >= n || arr[i][j] == 1 || (sum(i) + sum(j)) > k {
        return 0
    }
    arr[i][j] = 1
    sum := 1
    sum += dfs(m, n, i + 1, j, k, arr)
    sum += dfs(m, n, i - 1, j, k, arr)
    sum += dfs(m, n, i, j + 1, k, arr)
    sum += dfs(m, n, i, j - 1, k, arr)
    return sum
}

func sum(num int) int {
    sum := 0
    for num > 0 {
        sum += num % 10
        num = num / 10
    }
    return sum
}
```

https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/solution/qing-xi-yi-dong-de-go-shuang-100-jie-fa-lei-si-dao/