# 用两个栈实现队列

用两个栈实现一个队列。队列的声明如下，请实现它的两个函数 appendTail 和 deleteHead ，分别完成在队列尾部插入整数和在队列头部删除整数的功能。(若队列中没有元素，deleteHead 操作返回 -1 )

```
输入：
["CQueue","appendTail","deleteHead","deleteHead"]
[[],[3],[],[]]
输出：[null,null,3,-1]
```

```
输入：
["CQueue","deleteHead","appendTail","appendTail","deleteHead","deleteHead"]
[[],[],[5],[2],[],[]]
输出：[null,-1,null,null,5,2]
```

```go
type stack []int

func (s stack) Len() int {
	return len(s)
}

func (s *stack) Push(value int) {
	*s = append(*s, value)
}

func (s *stack) Pop() int {
	n := s.Len() - 1
	x := (*s)[n]
	*s = (*s)[: n]
	return x
}

type CQueue struct {
	in stack
	out stack
}

func Constructor() CQueue {
	return CQueue{}
}

func (this *CQueue) AppendTail(value int)  {
	this.in.Push(value)
}

func (this *CQueue) DeleteHead() int {
	if this.out.Len() != 0 {
		return this.out.Pop()
	} else if this.in.Len() != 0 {
		for this.in.Len() != 0 {
			this.out.Push(this.in.Pop())
		}
		return this.out.Pop()
	}
	return -1
}
```

