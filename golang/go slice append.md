我们可能会遇到给其他函数传递一个slice，让其他函数给这个slice做一些修改的情况。想到slice是引用传递，可以直接传递slice用作修改，于是可能出现下面这种情况：

```go
package main
 
import (
	"fmt"
	"testing"
)
 
func TestModifySlice(t *testing.T) {
	s := []string{"123"}
	modifySlice(s)
	fmt.Println(s)
}

func modifySlice(s []string) {
	s[0] = "modify:123"
	s = append(s, "456")
}
```

我们利用modifySlice函数对传入的s进行修改和添加的操作，想到slice是引用传递，在modifySlice中的修改必然会生效。

但是实际上呢，我们看一下运行情况：

![img](https://img-blog.csdnimg.cn/20190206125458373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpeXVubG9uZzQx,size_16,color_FFFFFF,t_70)

运行结果表明我们对s[0]做的修改生效了，但是添加新元素的操作却未生效，这是为什么呢？

原因就在于添加新元素是用的append，并将原先的引用重新赋值了！

我们把引用当成指针来看，指针指向的内容就是slice的数据内容。

指针本身是一种变量类型，指针类型在传递时，是值传递！

也就是说我们在外面调用modifySlice时，传入s是一个指针变量，当进入到modifySlice函数时，形参s‘跟外面传入的s并不是一个值，是s的值拷贝！但是他们的值是一样的，即指向的数据内容都是slice中的数据。

我们用s[0] = ”modify:123“做了修改，实际上是修改指针指向的数据的内容！这里肯定会生效！

然而我们用这种方式：s = append(s, "456")，这是在修改指针，并不是在修改指针指向的数据！

append可能会因为s的cap不足，重新分配空间，所以append有一个返回参数。

我们对modifySlice中的s进行了修改，但是s是指针，仅仅是外层传入的slice指针的拷贝，说明我们并没有对外层的slice重新赋值！

这就是添加新元素失败的原因。

 

如何改呢？

我们要对指针进行修改，指针是值传递，若想修改，自然要传递指针的指针啦！

于是我们可以修改成这样：

```go
package main
 
import (
	"fmt"
	"testing"
)
 
func TestModifySlice(t *testing.T) {
	s := []string{"123"}
	modifySlice(&s)
	fmt.Println(s)
}

func modifySlice(s *[]string) {
	(*s)[0] = "modify:123"
	*s = append(*s, "456")
}
```

再来看运行结果：

![img](https://img-blog.csdnimg.cn/20190206132151919.png)

完美解决了我们在上面遇到的问题~~

等等，是不是感觉写法很难看？golang的特点是简单易懂，这种写法是c、c++式写法，各种取地址、解引用，看起来很高端，但是理解起来要稍微转个弯。

golang一般不会对传入的参数进行修改，如果修改，以返回值的形式返回修改的内容。golang式写法是简单易懂的代表，写法也很优美，直接将slice作为返回值返回：

```go
package main
 
import (
	"fmt"
	"testing"
)
 
func TestModifySlice(t *testing.T) {
	s := []string{"123"}
	s = modifySlice(s)
	fmt.Println(s)
}

func modifySlice(s []string) []string {
	s[0] = "modify:123"
	s = append(s, "456")
	return s
}
```

结果毫无疑问也是正确的。
