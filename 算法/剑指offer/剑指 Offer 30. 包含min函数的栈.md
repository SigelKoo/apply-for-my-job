定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的 min 函数在该栈中，调用 min、push 及 pop 的时间复杂度都是 O(1)。

 ```
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.min();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.min();   --> 返回 -2.
 ```

```go
type MinStack struct {
    stack []int
    minStack []int
}


/** initialize your data structure here. */
func Constructor() MinStack {
	return MinStack{
		stack: []int{},
		minStack: []int{math.MaxInt64},
	}
}


func (this *MinStack) Push(x int)  {
	this.stack = append(this.stack, x)
	this.minStack = append(this.minStack, min(x, this.minStack[len(this.minStack) - 1]))
}


func (this *MinStack) Pop()  {
	this.stack = this.stack[: len(this.stack) - 1]
	this.minStack = this.minStack[: len(this.minStack) - 1]
}


func (this *MinStack) Top() int {
	return this.stack[len(this.stack) - 1]
}


func (this *MinStack) Min() int {
	return this.minStack[len(this.minStack) - 1]
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

