# 二维数组中的查找

在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个高效的函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

现有矩阵 matrix 如下：

```
[
  [1,   4,  7, 11, 15],
  [2,   5,  8, 12, 19],
  [3,   6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]
```

给定 target = 5，返回 true。

给定 target = 20，返回 false。



1. 蛮力法

2. 线性查找法，从数组的右上角出发，如果当前目标元素小于右上角元素，j左移；如果当前目标大于右上角元素，i下移；等于返回true

   例如：找16

   16 > 15，变为：

   ```
   [
     [2,   5,  8, 12, 19],
     [3,   6,  9, 16, 22],
     [10, 13, 14, 17, 24],
     [18, 21, 23, 26, 30]
   ]
   ```

   16 < 19，变为：

   ```
   [
     [2,   5,  8, 12],
     [3,   6,  9, 16],
     [10, 13, 14, 17],
     [18, 21, 23, 26]
   ]
   ```

   16  > 12，变为：

   ```
   [
     [3,   6,  9, 16],
     [10, 13, 14, 17],
     [18, 21, 23, 26]
   ]
   ```

   16 == 16，返回true
```go
func findNumberIn2DArray(matrix [][]int, target int) bool {
	if len(matrix) == 0 || len(matrix[0]) == 0 {
		return false
	}
	i := 0
	j := len(matrix[0]) - 1
	h := len(matrix)
	for i < h && j >= 0{
		if target == matrix[i][j] {
			return true
		} else if target > matrix[i][j] {
			i++
		} else if target < matrix[i][j] {
			j--
		}
	}
	return false
}
```
3. 使用sort包下的SearchInts方法（慢）

因为数组已经是递增关系，利用sort包下的SeachInts方法，遍历数组切片，查找数组中是否含有target值，如果查找不到，返回值是target应该插入数组的位置（会保持数组的递增顺序），此时需要分情况：
情况1：返回值=len(a)，即插入后长度+1，说明target数组内不存在。
情况2：返回值<len(a)，此时不能认为target在数组内，因为返回的是“插入后保持递增顺序的位置“，所以必须将target与数组中该位置的值进行比较，相同才返回true。

```go
func findNumberIn2DArray(matrix [][]int, target int) bool {
	for _, nums := range matrix {
		i := sort.SearchInts(nums, target)
		if i < len(nums) && target == nums[i] {
			return true
		}
	}
	return false
}
```

