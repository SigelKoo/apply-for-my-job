# 打印从1到最大的n位数

输入数字 `n`，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数 999。

```
输入: n = 1
输出: [1,2,3,4,5,6,7,8,9]
```

```
func printNumbers(n int) []int {
	res := make([]int, 0)
	if n == 0 {
		return res
	}
	end := math.Pow(10, float64(n))
	for i := 1; i < int(end); i++ {
		res = append(res, i)
	}
	return res
}
```

