# 数组中重复的数字

在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

```
输入：
[2, 3, 1, 0, 2, 5, 3]
输出：2 或 3
```

使用哈希表

```go
func findRepeatNumber(nums []int) int {
    hashtable := make(map[int]bool)
    for _, num := range nums {
        if hashtable[num] {
            return num
        } else {
            hashtable[num] = true
        }
    }
    return -1
}
```

数组长度为n，数字在0~n-1范围内，只需判断下标i与nums[i]相等即可（都相等，说明没有重复元素）

i与nums[i]相等，继续向下扫描

i与nums[i]不相等，先比较nums[i]与nums[nums[i]]的值：

​	nums[i]与nums[nums[i]]相等，返回nums[i]

​	nums[i]与nums[nums[i]]不相等，交换nums[i]与nums[nums[i]]，一直重复

```go
func findRepeatNumber(nums []int) int {
    for i := 0; i < len(nums); i++ {
        if nums[i] == i {
            continue;
        }
        if nums[i] == nums[nums[i]] {
            return nums[i];
        } else {
            nums[i], nums[nums[i]] = nums[nums[i]], nums[i];
        }
    }
    return -1;
}
```

