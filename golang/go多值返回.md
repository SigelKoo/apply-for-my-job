## go多值返回

```go
func f(arg1, arg2 int)(ret1, ret2 int)
```

Go的做法是在传入的参数之上留了两个空位，**被调者直接将返回值放在这两空位**，函数f调用前其内存布局是这样的：

```
为ret2保留空位
为ret1保留空位
参数3
参数2
参数1<-SP 
```

调用后

```
为ret2保留空位
为ret1保留空位
参数2
参数1<-FP
保存PC <-SP
f的栈
...
```

![多值返回 - 图1](https://static.bookstack.cn/projects/go-internals/zh/images/3.2.funcall.jpg?raw=true)

Go的C编译器按是plan9的C编译器实现的，在被调函数中对参数值的修改是会返回到调用函数中的。在函数体中设置ret1和ret2的值，实际上会被编译成这样：

```
MOVQ    BX,ret1+16(FP)
...
MOVQ    BX,ret2+24(FP)
```

**对ret1+16(FP)的赋值其实是修改的调用函数的栈中的内容，这样就会将结果返回给调用函数了**。这就是Go和C函数调用协议中很重要的一个区别：为了实现多值返回，Go是使用栈空间来返回值的。而常见的C语言是通过寄存器来返回值的。