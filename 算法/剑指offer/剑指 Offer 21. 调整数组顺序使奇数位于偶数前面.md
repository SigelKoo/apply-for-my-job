输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位于数组的后半部分。

```
输入：nums = [1,2,3,4]
输出：[1,3,2,4] 
注：[3,1,2,4] 也是正确的答案之一。
```

```go
func exchange(nums []int) []int {
    l, r := 0, len(nums) - 1
    for l < r {
        for l < r && nums[l] % 2 != 0 {
            l++
        }
        for l < r && nums[r] % 2 == 0 {
            r--
        }
        nums[l], nums[r] = nums[r], nums[l]
    }
    return nums
}
```

