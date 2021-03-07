# 数值的整数次方

实现函数double Power(double base, int exponent)，求base的exponent次方。不得使用库函数，同时不需要考虑大数问题。

```
输入: 2.00000, 10
输出: 1024.00000
```

```
输入: 2.10000, 3
输出: 9.26100
```

```
输入: 2.00000, -2
输出: 0.25000
解释: 2^-2 = 1/2^2 = 1/4 = 0.25
```

我的解，超出时间限制

```go
func myPow(x float64, n int) float64 {
	if x == 0.0 {
		return 0.0
	}
	sum := 1.0
	if n > 0 {
		for i := 0; i < n; i++ {
			sum = sum * x
		}
	} else if n == 0 {
		return 1.0
	} else {
		for i := 0; i < -n; i++ {
			sum = sum * x
		}
		sum = 1 / sum
	}
	return sum
}
```

二分法，迭代求解

```go
func myPow(x float64, n int) float64 {
    if n >= 0 {
        return quickMul(x, n)
    }
    return 1.0 / quickMul(x, -n)
}

func quickMul(x float64, n int) float64 {
    if n == 0 {
        return 1
    }
    y := quickMul(x, n / 2)
    if n % 2 == 0 {
        return y * y
    }
    return y * y * x
}
```

