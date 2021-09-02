# 简述LRU算法及其实现方式

LRU又称为最近最久未使用置换算法，每次淘汰的页面是最近最久未使用的页面。

**实现方法**：赋予每个页面对应的页表项中，用访问字段记录该页面自上次被访问以来所经历的时间t。当需要淘汰一个页面时，选择现有页面中t值最大的，即最近最久未使用的页面。

假设系统为某进程分配了四个内存块，并考虑到有以下页面号引用串：1, 8, 1, 7, 8, 2, 7, 2, 1, 8, 3, 8, 2, 1, 3, 1, 7, 1, 3, 7

在手动做题时，若需要淘汰页面，可以逆向检查此时在内存中的几个页面号。在逆向扫描过程中最后一个出现的页号就是要淘汰的页面。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201130200350527.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzOTkyOTQ5,size_16,color_FFFFFF,t_70)

就是向左数，最后一个出现的页号

**特点**：

- 优点：由于考虑程序访问的时间局部性，一般能有较好的性能；实际应用多
- 缺点：实现需要较多的硬件支持，会增加硬件成本

**实现方式**：

通常通过哈希表和双链表进行实现，使用List保存数据，Map来做快速访问即可

```go
package main

import "container/list"

type entry struct {
	key, value int
}

type LRUCache struct {
	cap int
	cache map[int] *list.Element
	lst *list.List
}

func Constructor(capacity int) LRUCache {
	return LRUCache{capacity, map[int]*list.Element{}, list.New()}
}

func (this *LRUCache) Get(key int) int {
	e := this.cache[key]
	if e == nil {
		return -1
	}
	this.lst.MoveToFront(e)
	return e.Value.(entry).value
}

func (this *LRUCache) Put(key int, value int)  {
	if e := this.cache[key]; e != nil {
		e.Value = entry{key, value}
		this.lst.MoveToFront(e)
		return
	}
	this.cache[key] = this.lst.PushFront(entry{key, value})
	if len(this.cache) > this.cap {
		delete(this.cache, this.lst.Remove(this.lst.Back()).(entry).key)
	}
}
```



