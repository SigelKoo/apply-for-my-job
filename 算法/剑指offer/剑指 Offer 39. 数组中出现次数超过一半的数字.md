数组中有一个数字出现的次数超过数组长度的一半，请找出这个数字。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

```
输入: [1, 2, 3, 2, 2, 2, 5, 4, 2]
输出: 2
```

```go
func majorityElement(nums []int) int {
    hashtable := make(map[int]int)
    for i := 0; i < len(nums); i++ {
        hashtable[nums[i]]++
    }
    max := 0
    maxIndex := 0
    for k, v := range hashtable {
        if max < v {
            max = v
            maxIndex = k
        }
    }
    return maxIndex
}
```

