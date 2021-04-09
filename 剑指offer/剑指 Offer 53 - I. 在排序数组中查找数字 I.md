统计一个数字在排序数组中出现的次数。

```
输入: nums = [5,7,7,8,8,10], target = 8
输出: 2
```

```
输入: nums = [5,7,7,8,8,10], target = 6
输出: 0
```

```GO
func search(nums []int, target int) int {
	hashtable := make(map[int]int)
	for i := 0; i < len(nums); i++ {
		hashtable[nums[i]]++
	}
	return hashtable[target]
}
```

