# go中new和make区别

new只分配内存

make用于slice、map和channel的初始化

内置函数new按指定类型长度分配零值内存，返回指针，并不关心类型内部构造和初始化方式。

内置函数make对引用类型进行创建，编译器会将make转换为目标类型专用的创建函数，以确保完成全部内存分配和相关属性初始化。

1. new(T) 返回的是 T 的指针

   new(T) 为一个 T 类型新值分配空间并将此空间初始化为 T 的零值，返回的是新值的地址，也就是 T 类型的指针 *T，该指针指向 T 的新分配的零值。

2. make 只能用于 slice，map，channel

   make 只能用于 slice，map，channel 三种类型，make(T, args) 返回的是初始化之后的 T 类型的值，这个新值并不是 T 类型的零值，也不是指针 * T，是经过初始化之后的 T 的引用。

```go
var s1 []int
if s1 == nil {
    fmt.Printf("s1 is nil --> %#v \n ", s1)
}
s2 := make([]int, 3)
if s2 == nil {
    fmt.Printf("s2 is nil --> %#v \n ", s2)
} else {
    fmt.Printf("s2 is not nill --> %#v \n ", s2)
}
```

**slice 的零值是 nil**，使用 **make 之后 slice 是一个初始化的 slice，即 slice 的长度、容量、底层指向的 array 都被 make 完成初始化**，此时 slice 内容被类型 int 的零值填充，形式是 [0 0 0]，map 和 channel 也是类似的。

3. make(T, args) 返回的是 T 的 引用

   如果不特殊声明，go 的函数默认都是按值穿参，即通过函数传递的参数是值的副本，在函数内部对值修改不影响值的本身，但是 **make(T, args) 返回的值通过函数传递参数之后可以直接修改**，即 map，slice，channel 通过函数穿参之后在函数内部修改将影响函数外部的值。

   这说明 make(T, args) 返回的是引用类型，在函数内部可以直接更改原始值，对 map 和 channel 也是如此。

4. 很少需要使用 new

   struct 初始化的过程，可以说明不使用 new 一样可以完成 struct 的初始化工作。

   ```go
   //声明初始化
   var foo1 Foo
   fmt.Printf("foo1 --> %#v\n ", foo1) //main.Foo{age:0, name:""}
   foo1.age = 1
   fmt.Println(foo1.age)
   //struct literal 初始化
   foo2 := Foo{}
   fmt.Printf("foo2 --> %#v\n ", foo2) //main.Foo{age:0, name:""}
   foo2.age = 2
   fmt.Println(foo2.age)
   //指针初始化
   foo3 := &Foo{}
   fmt.Printf("foo3 --> %#v\n ", foo3) //&main.Foo{age:0, name:""}
   foo3.age = 3
   fmt.Println(foo3.age)
   //new 初始化
   foo4 := new(Foo)
   fmt.Printf("foo4 --> %#v\n ", foo4) //&main.Foo{age:0, name:""}
   foo4.age = 4
   fmt.Println(foo4.age)
   //声明指针并用 new 初始化
   var foo5 *Foo = new(Foo)
   fmt.Printf("foo5 --> %#v\n ", foo5) //&main.Foo{age:0, name:""}
   foo5.age = 5
   fmt.Println(foo5.age)
   ```

   foo1 和 foo2 是同样的类型，都是 Foo 类型的值，foo1 是通过 var 声明，Foo 的 filed 自动初始化为每个类型的零值，foo2 是通过字面量的完成初始化。
   foo3，foo4 和 foo5 是一样的类型，都是 Foo 的指针 *Foo。

   但是所有 foo 都可以直接使用 Foo 的 filed，读取或修改，为什么？
   如果 x 是可寻址的，&x 的 filed 集合包含 m，x.m 和 (&x).m 是等同的，go 自动做转换，也就是 foo1.age 和 foo3.age 调用是等价的，go 在下面自动做了转换。

   

   因而可以直接使用 struct literal 的方式创建对象，能达到和 new 创建是一样的情况而不需要使用 new。

   new与map的纠纷，new为引用类型分配内存，但这是不完整创建。

   ```go
   var map1 = make(map[keytype]valuetype)。
   map1 := make(map[keytype]valuetype)。         相当于：mapCreated := map[string]float32{}。
   ```

   不要使用 new，永远用 make 来构造 map。