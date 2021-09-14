# 一

匿名函数没有函数名，只有函数体，它只有在被调用的时候才会被初始化。匿名函数一般被当作一种类型被赋值给类型为函数类型的变量，经常用于实现回调函数和闭包等功能。

Golang 的匿名函数的声明样式如下所示：

```go
func(params)(return params){
	function body
}
```

在声明匿名函数之后，在其后加上调用的参数列表，即可立即对匿名函数进行调用。除此之外，我们还可以将匿名函数赋值给函数类型的变量，用于多次调用或者求值，如下例子所示：

```go
currentTime := func() {
	fmt.Println(time.Now())
}
// 调用匿名函数
currentTime()
```

上述例子中，我们通过匿名函数实现了一个简单的报时器，并赋值给 `currentTime`，每次调用 `currentTime`，我们都能知道当前系统的最新时间。

匿名函数一个比较常用的场景是用作回调函数。我们在接下来的例子中定义这样一个函数：它接受 `string` 和匿名函数的参数输入，然后使用匿名函数对 `string` 进行处理，代码如下所示：

```go
package main

import "fmt"

func proc(input string, processor func(str string))  {
	// 调用匿名函数
	processor(input)
}

func main()  {
	proc("王小二", func(str string) {
		for _, v := range str{
			fmt.Printf("%c\n", v)
		}
	})
}
```

上面代码中的匿名函数被作为回调函数用于对传递的字符串进行处理，用户可以根据自己的需要传递不同的匿名函数实现对字符串进行不同的处理操作。

# 二

闭包是携带状态的函数，它是将函数内部和函数外部连接起来的桥梁。通过闭包，我们可以读取函数内部的变量。我们也可以使用闭包封装私有状态，让它们常驻于内存当中。

闭包能够引用其作用域上部的变量并进行修改，被捕获到闭包中的变量将随着闭包的生命周期一直存在，函数本身是不存储信息的，但是闭包中的变量使闭包本身具备了存储信息的能力。我们可以用闭包的特性实现一个简单的计数器，代码如下所示：

```go
package main

import "fmt"

func createCounter(initial int) func() int {

	if initial < 0{
		initial = 0
	}

	// 引用 initial，创建一个闭包
	return func() int{
		initial++
		// 返回当前计数
		return initial;
	}

}

func main()  {

	// 计数器 1
	c1 := createCounter(1)

	fmt.Println(c1()) // 2
	fmt.Println(c1()) // 3

	// 计数器 2
	c2 := createCounter(10)

	fmt.Println(c2()) // 11
	fmt.Println(c1()) // 4

}
```

`createCounter` 函数返回了一个闭包，该闭包中封装了计数值 `initial`，从外部代码根本无法直接访问该变量。不同的闭包之间变量不会互相干扰，`c1` 和 `c2` 两个计数器都是独立进行计数。



```go
func f(i int) func() int {
    return func() int {
        i++
        return i
    }
}
```

函数f返回了一个函数，返回的这个函数，返回的这个函数就是一个闭包。这个函数中本身是没有定义变量i的，而是引用了它所在的环境（函数f）中的变量i。

```go
c1 := f(0)
c2 := f(0)
c1()    // reference to i, i = 0, return 1
c2()    // reference to another i, i = 0, return 1
```

c1跟c2引用的是不同的环境，在调用i++时修改的不是同一个i，因此两次的输出都是1。函数f每进入一次，就形成了一个新的环境，对应的闭包中，函数都是同一个函数，环境却是引用不同的环境。

变量i是函数f中的局部变量，假设这个变量是在函数f的栈中分配的，是不可以的。因为函数f返回以后，对应的栈就失效了，f返回的那个函数中变量i就引用一个失效的位置了。所以闭包的环境中引用的变量不能够在栈上分配。

## 逃逸分析

在继续研究闭包的实现之前，先看一看Go的一个语言特性：

```
func f() *Cursor {
    var c Cursor
    c.X = 500
    noinline()
    return &c
}
```

Cursor是一个结构体，这种写法在C语言中是不允许的，因为变量c是在栈上分配的，当函数f返回后c的空间就失效了。但是，在Go语言规范中有说明，这种写法在Go语言中合法的。语言会自动地识别出这种情况并在堆上分配c的内存，而不是函数f的栈上。

为了验证这一点，可以观察函数f生成的汇编代码：

```
MOVQ    $type."".Cursor+0(SB),(SP)    // 取变量c的类型，也就是Cursor
PCDATA    $0,$16
PCDATA    $1,$0
CALL    ,runtime.new(SB)    // 调用new函数，相当于new(Cursor)
PCDATA    $0,$-1
MOVQ    8(SP),AX    // 取c.X的地址放到AX寄存器
MOVQ    $500,(AX)    // 将AX存放的内存地址的值赋为500
MOVQ    AX,"".~r0+24(FP)
ADDQ    $16,SP
```

识别出变量需要在堆上分配，是由编译器的一种叫escape analyze的技术实现的。如果输入命令：

```
go build --gcflags=-m main.go
```

可以看到输出：

```
./main.go:20: moved to heap: c
./main.go:23: &c escapes to heap
```

表示c逃逸了，被移到堆中。escape analyze可以分析出变量的作用范围，这是对垃圾回收很重要的一项技术。

## 闭包底层结构体

回到闭包的实现来，前面说过，闭包是函数和它所引用的环境。那么是不是可以表示为一个结构体呢：

```go
type Closure struct {
    F func()() 
    i *int
}
```

事实上，Go在底层确实就是这样表示一个闭包的。让我们看一下汇编代码：

```go
func f(i int) func() int {
    return func() int {
        i++
        return i
    }
}


MOVQ    $type.int+0(SB),(SP)
PCDATA    $0,$16
PCDATA    $1,$0
CALL    ,runtime.new(SB)    // 是不是很熟悉，这一段就是i = new(int)    
...    
MOVQ    $type.struct { F uintptr; A0 *int }+0(SB),(SP)    // 这个结构体就是闭包的类型
...
CALL    ,runtime.new(SB)    // 接下来相当于 new(Closure)
PCDATA    $0,$-1
MOVQ    8(SP),AX
NOP    ,
MOVQ    $"".func·001+0(SB),BP
MOVQ    BP,(AX)                // 函数地址赋值给Closure的F部分
NOP    ,
MOVQ    "".&i+16(SP),BP        // 将堆中new的变量i的地址赋值给Closure的值部分
MOVQ    BP,8(AX)
MOVQ    AX,"".~r1+40(FP)
ADDQ    $24,SP
RET    ,
```

其中func·001是另一个函数的函数地址，也就是f返回的那个函数。

## 小结

1. Go语言支持闭包
2. Go语言能通过escape analyze识别出变量的作用域，自动将变量在堆上分配。将闭包环境变量在堆上分配是Go实现闭包的基础。
3. 返回闭包时并不是单纯返回一个函数，而是返回了一个结构体，记录下函数返回地址和引用的环境中的变量地址。

# 三

闭包作用：缩小变量作用域，减少对全局变量的污染

```go
package main

import "fmt"

func adder() func(int) int {
    sum := 0
    return func(x int) int {
        sum += x
        return sum
    }
}

func main() {
    pos, neg := adder(), adder()
    for i := 0; i < 10; i++ {
        fmt.Println(
            pos(i),
            neg(-2*i),
        )
    }
}
```

可以理解为：执行adder()，然后将其return的内容赋值给pos，而return的内容就是一个标准的函数，即：

```
func(x int) int {
    sum += x
    return sum
}
```

## 闭包函数本身可用函数值书写

```go
package main

import "fmt"

func main() {
    adder := func () func(int) int {
        sum := 0
        return func(x int) int {
            sum += x
            return sum
        }
    }

    pos, neg := adder(), adder()
    for i := 0; i < 10; i++ {
        fmt.Println(
            pos(i),
            neg(-2*i),
        )
    }
}
```

## 什么是闭包

Go 函数可以是一个闭包。闭包是一个函数值，它引用了函数体之外的变量。 这个函数可以对这个引用的变量进行访问和赋值；换句话说这个函数被“绑定”在这个变量上。

例如，函数 adder 返回一个闭包。每个返回的闭包都被绑定到其各自的 sum 变量上。

```go
package main

import "fmt"

func adder() func(int) int {
    sum := 0
    return func(x int) int {
        sum += x
        return sum
    }
}

func main() {
    pos, neg := adder(), adder()
    for i := 0; i < 10; i++ {
        fmt.Println(
            pos(i),
            neg(-2*i),
        )
    }
}
```

上面  return func(x int) int {} 就是一个闭包，如pos := adder()的adder()表示返回了一个闭包，并赋值给了pos，同时，这个被赋值给了pos的闭包函数被绑定在sum变量上，因此pos闭包函数里的变量sum和neg变量里的sum毫无关系。

## 我对闭包的理解

没有闭包的时候，函数就是一次性买卖，函数执行完毕后就无法再更改函数中变量的值（应该是内存释放了）；有了闭包后函数就成为了一个变量的值，只要变量没被释放，函数就会一直处于存活并独享的状态，因此可以后期更改函数中变量的值（因为这样就不会被go给回收内存了，会一直缓存在那里）。

比如，实现一个计算功能：一个数从0开始，每次加上自己的值和当前循环次数（当前第几次，循环从0开始，到9，共10次），然后*2，这样迭代10次：

```go
func abc(x int) int {
    return x * 2
}

func main() {
    var a int
    for i := 0; i < 10; i ++ {
        a = abc(a+i)
        fmt.Println(a)
    }
}
```

可以使用闭包

```go
func abc() func(int) int {
    res := 0
    return func(x int) int {
        res = (res + x) * 2
        return res
    }
}

func main() {
    a := abc()
    for i := 0; i < 10; i++ {
        fmt.Println(a(i))
    }
}
```

从上面例子可以看出闭包的3个好处：

1. 不是一次性消费，被引用声明后可以重复调用，同时变量又只限定在函数里，同时每次调用不是从初始值开始（函数里长期存储变量）

   这有点像使用面向对象的感觉，实例化一个类，这样这个类里的所有方法、属性都是为某个人私有独享的。但比面向对象更加的轻量化

2. 用了闭包后，主函数就变得简单了，把算法封装在一个函数里，使得主函数省略了a=abc(a+i)这种麻烦事了

3. 变量污染少，因为如果没用闭包，就会为了传递值到函数里，而在函数外部声明变量，但这样声明的变量又会被下面的其他函数或代码误改。