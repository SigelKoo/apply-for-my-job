# 用Rand7()实现Rand10()

已有方法 rand7 可生成 1 到 7 范围内的均匀随机整数，试写一个方法 rand10 生成 1 到 10 范围内的均匀随机整数。

不要使用系统的 Math.random() 方法。

```
输入: 1
输出: [7]
```

```
输入: 2
输出: [8,4]
```

```
输入: 3
输出: [8,1,10]
```

```
func rand10() int {
    
}
```

我的解法：

```go
func rand10() int {
	a := rand7()
	b := rand7() - 1
	c := a + b
	res := (c % 10) * 1 + (c / 10) * 7
	return res
}
```

题解：

```go
func rand10() int {
	for {
		r1 := rand7()
		r2 := rand7()
		num := r1 + (r2 - 1) * 7
		if num <= 40 {
			return num % 10 + 1
		}
	}
}
```

