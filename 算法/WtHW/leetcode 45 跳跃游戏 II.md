
给定一个非负整数数组，你最初位于数组的第一个位置。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

你的目标是使用最少的跳跃次数到达数组的最后一个位置。

```
输入: [2,3,1,1,4]
输出: 2
解释: 跳到最后一个位置的最小跳跃数是 2。
     从下标为 0 跳到下标为 1 的位置，跳 1 步，然后跳 3 步到达数组的最后一个位置。
```

```go
func jump(nums []int) int {
    dp := make([]int, len(nums))
    for i := 1; i < len(nums); i++ {
        dp[i] = len(nums)
        for j := 0; j < i; j++ {
            if j + nums[j] >= i {
                dp[i] = min(dp[i], dp[j] + 1)
            }
        }
    }
    return dp[len(nums) - 1]
}

func min(a, b int) int {
    if a > b {
        return b
    }
    return a
}
```

