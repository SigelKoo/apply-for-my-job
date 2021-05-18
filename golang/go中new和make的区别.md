# go中new和make区别

new创建非map，slice，chan对象；返回值时指针；new只是分配内存，创建的对象是0值；struct类型使用new时底层调用runtime.newobject

```go
type st struct {
	i int
}

func newfuncs()  {
	i := new(st)
	fmt.Println(i)
}
```

输出值为&{0}

new的作用

```go
// 会报panic
var v *int
*v = 8
fmt.Println(*v)

var v *int
v = new(int)
*v = 8
fmt.Println(*v)
```

我们可以看到初始化一个指针变量，其值为nil，nil的值是不能直接赋值的。通过new其返回一个指向新分配的类型为int的指针，指针值为0xc00004c088，这个指针指向的内容的值为零



slice、map、chan初始化为nil，nil不能直接赋值，并且不能使用new分配内存；make不仅可以分配内存，还可以给内存的类型初始化其零值。



make创建map，slice，chan对象；返回值时make函数的第一个参数；底层调用runtime.makechan，runtime,makemap，runtime.makemap_small，runtime.makeslice；make创建的对象值不是零值

```go
func makefuncs()  {
	s := make([]int, 0, 0)
	if s == nil {
		fmt.Println("s")
		log.Println(s)
	}
	var sc []int
	if sc == nil {
		fmt.Println("sc")
		log.Println(sc)
	}
}
```

输出值为sc  []，说明make后的不是nil值，而只是声明的时nil值



想要看底层调用，可以进行反汇编进行查看

在src/cmd/compile/internal/gc/walk.go中OMAKECHAN看到实际编译调用的时makechan，在src/runtime/chan.go下有makechan函数，底层大概new了一个hchan结构，为new的hchan进行了mallocgc操作，最后返回new的hchan结构

```
c = new(hchan)
c.buf = mallocgc(mem, elem, true)
```



