# 剪绳子

给你一根长度为 n 的绳子，请把绳子剪成整数长度的 m 段（m、n都是整数，n>1并且m>1），每段绳子的长度记为 k[0],k[1]...k[m-1] 。请问 k[0]*k[1]*...*k[m-1] 可能的最大乘积是多少？例如，当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到的最大乘积是18。

```
输入: 2
输出: 1
解释: 2 = 1 + 1, 1 × 1 = 1
```

```
输入: 10
输出: 36
解释: 10 = 3 + 3 + 4, 3 × 3 × 4 = 36
```

```go
func cuttingRope(n int) int {
    dp := make([]int, n + 1)
    dp[0], dp[1], dp[2] = 0, 1, 1
    for i := 3; i <= n; i++ {
        for j := 2; j < i; j++ {
            dp[i] = max(dp[i], dp[i - j] * j, (i - j) * j)
        }
    }
    return dp[n]
}

func max(a int, b int, c int) int {
    if a >= b && a >= c {
        return a
    } else if b >= a && b >= c {
        return b
    } else {
        return c
    }
}
```

