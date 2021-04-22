输入整数数组 `arr` ，找出其中最小的 `k` 个数。例如，输入4、5、1、6、2、7、3、8这8个数字，则最小的4个数字是1、2、3、4。

```
输入：arr = [3,2,1], k = 2
输出：[1,2] 或者 [2,1]
```

```
输入：arr = [0,1,2,1], k = 1
输出：[0]
```

```go
func getLeastNumbers(arr []int, k int) []int {
	if k == 0 || k > len(arr){
		return []int{}
	}
	h := &IHeap40{}
	heap.Init(h)

	for _, num := range arr {
		if h.Len() < k {
			heap.Push(h, num)
		} else {
			if (*h)[0] > num {
				heap.Pop(h)
				heap.Push(h, num)
			}
		}
	}
	return *h
}

type IHeap40 []int

func (h IHeap40) Len() int {
	return len(h)
}

func (h IHeap40) Less(i, j int) bool {
	return h[i] > h[j]
}

func (h IHeap40) Swap(i, j int) {
	h[i], h[j] = h[j], h[i]
}

func (h *IHeap40) Push(x interface{}) {
	*h = append(*h, x.(int))
}

func (h *IHeap40) Pop() interface{} {
	x := (*h)[len(*h) - 1]
	*h = (*h)[: len(*h) - 1]
	return x
}
```

