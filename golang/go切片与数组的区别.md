### 数组

数组是一个包含相同类型的固定长度序列，数组是值类型，你将数组赋值给另一个数组，将整个数组拷贝一份

```go
var arrayInt64 [3]int64
arrayInt64[0], arrayInt64[1], arrayInt64[2] = 0, 1, 2

arrayString := []string{"hello", "go"}

arrayFloat := [...]{1.0, 2.0, 3.0}

matrix := [2][2]int64{{0, 1}, {2, 3}}
```

### 切片

比数组更加灵活，长度是可变化的，可自动扩容，可以理解为一个结构体，包括3个元素：一个指针，指向数组中slice指定的开始位置长度、最大长度、

数组长度是被赋值的最大下标 + 1，容量是指切片可容纳的最多元素个数，在追加元素时，如果容量cap不足时将按len的2倍扩容

```go
names := []string{"zhang", "wang", "li", "zhao"}
names2 := names[0:3]
// 修改子切片中内容，切片中内容也会相应的改变
names2[0] = "laozhang"

ints := make([]int, 3)
ints[0], ints[1] = 1, 2
```

