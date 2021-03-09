### sync.WaitGroup

#### goroutine 监测

在使用 golang 中的 goroutine 的时候，有一个细节很容易被新手忽略，那就是很容易就让 主goroutine 返回了，从而导致其他的 goroutine 都被关闭了，也就没有所谓的并发了。

```go
func wgTest(wg *sync.WaitGroup)  {
	defer wg.Done()
	fmt.Println("Test")
}

func main() {
	wg := sync.WaitGroup{}
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go wgTest(&wg)
	}
	wg.Wait()
	fmt.Println("done")
}
```

简单的WaitGroup使用，在每个goroutine启动前增加一个计数，同时在每一个 goroutine 结束时递减计数

- **Add(delta int)**：增加/减少若干计数
- **Done**：减少 1 个计数，等价于 **Add(-1)**
- **Wait**：卡住，直到计数等于 0

wgTest中使用指针类型的`*sync.WaitGroup`作为参数，不能使用值类型`sync.WaitGroup`作为参数，每个goroutine都拷贝一份wg，每个goroutine都使用自己的wg是不合理的，多个goroutine应该共享一个wg

#### 源码

##### 结构体：

```go
type WaitGroup struct {
    // 保证sync.WaitGroup不会被开发者通过再赋值的方式拷贝
	noCopy noCopy
    // 存储着状态和信号量
	state1 [3]uint32
}
```

noCopy：它是 sync 包下的一个特殊标记，可以嵌入到结构中，在第一次使用后不可复制，使用`go vet`作为检测使用，并因此只能进行指针传递，从而保证全局唯一；

state1：是用来存放任务计数器和等待者计数器，应该会涉及到一些位操作相关的内容

##### Add()

```go
func (wg *WaitGroup) Add (delta int) {
    // 首先获取状态值
    statep, semap := wg.state ()
    // 对于 statep 中 counter + delta
    state := atomic.AddUint64 (statep, uint64 (delta)<<32)
    // 获取任务计数器的值
    v := int32 (state >> 32)
    // 获取等待者计数器的值
    w := uint32 (state)
    
    // 任务计数器不能为负数
    if v < 0 {
        panic ("sync: negative WaitGroup counter")
    }
    // 已经有人在等待，但是还在添加任务
    if w != 0 && delta > 0 && v == int32 (delta) {
        panic ("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 没有等待者或者任务还有没做完的
    if v > 0 || w == 0 {
        return
    }
    // 有等待者，但是在这个过程中数据还在变动
    if *statep != state {
        panic ("sync: WaitGroup misuse: Add called concurrently with Wait")
    }

    // Reset waiters count to 0.
    // 重置状态，并用发出等同于等待者数量的信号量，告诉所有等待者任务已经完成
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease (semap, false, 0)
    }
}
```

信号量机制

##### Wait()

```go
func (wg *WaitGroup) Wait () {
    // 先获取状态
    statep, semap := wg.state ()
  
    for {
        // 这里注意要用 atomic 的 Load 来保证一下写操作已经完成
        state := atomic.LoadUint64 (statep)
        // 同样的，这里是任务计数
        v := int32 (state >> 32)
        // 这里是等待者计数
        w := uint32 (state)
        // 如果没有任务，那么直接结束，不用等待了
        if v == 0 {
            return
        }
        // 使用 cas 操作，如果不相等，证明中间已经被其他人修改了状态，重新走 for 循环
        // 注意这里 if 进去之后等待者的数量就 +1 了
        if atomic.CompareAndSwapUint64 (statep, state, state+1) {
            // 等待信号量
            runtime_Semacquire (semap)
            // 如果信号量来了，但是状态还不是 0，则证明 wait 之后还是在人在 add，证明有人想充分利用 wg 但是时机不对
            if *statep != 0 {
                panic ("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

wait使用load和cas + 循环避免锁

##### 注意事项

1. Wait 可以被调用多次，并且每个都会收到完成的通知
2. Wait 之后，如果再 Wait 的过程中不能在 Add，否则会 panic，但是 Wait 结束之后可以继续使用 Add 进行重用
3. 可以使用 Add 传递负数的方式一次性结束多个任务，但是需要保证任务计数器非负，否则会 panic
4. wg作为参数传递的时候需要注意传递指针，或者尽量避免传递
5. 官方利用位操作节约了空间，存在在同一个地方；利用信号量来实现任务结束的通知....

### sync.Map

##### 栗子

```go
type mySyncMap struct {
	sync.Map
}

func (m mySyncMap) Print(k interface{})  {
	value, ok := m.Load(k)
	fmt.Println(value, ok)
}

func main() {
	var syncMap mySyncMap
	syncMap.Print("Key1")
	syncMap.Store("Key1", "Value1")
	syncMap.Print("Key1")

	syncMap.Store("Key2", "Value2")

	syncMap.Store("Key3", 2)
	syncMap.Print("Key3")

	syncMap.Store(4, 4)
	syncMap.Print(4)

	syncMap.Delete("Key1")
	syncMap.Print("Key1")
}
```

##### 方法

存储数据，存入key以及value可以为任意类型

```go
func (m *Map) Store(key, value interface{})
```

删除对应key

```go
func (m *Map) Delete(key interface{})
```

读取对应key的值，ok表示是否在map中查询到key

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool)
```

针对某个key的存在读取，不存在就存储，loaded为true表示存在值，false表示不存在值

```go
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)
```

表示对所有key进行遍历，并将遍历出的key，value传入回调函数进行函数调用，回调函数返回false时遍历结束，否则遍历完所有key

```go
func (m *Map) Range(f func(key, value interface{}) bool)
```

#### 源码

**通过引入两个map，将读写分离到不同的map，其中read map只提供读，而dirty map则负责写。**

read map可以在不加锁的情况下进行并发读取，当read map中没有读取到值时，再加锁进行后继续读取，并累加未命中数，当未命中数到达一定数量后，将dirty map上升为read map。

另外，虽然引入了两个map，但是底层数据存储的是指针，指向的是同一份值。

##### 流程

如插入key 1，2，3时均插入了dirty map中，此时read map没有key值，读取时从dirty map中读取，并记录miss数

![图片描述](https://segmentfault.com/img/bV7l2O?w=626&h=246)

当miss数大于等于dirty map的长度时，将dirty map直接升级为read map，这里直接对dirty map进行地址拷贝

![图片描述](https://segmentfault.com/img/bV7l2T?w=625&h=246)

当有新的key为4插入时，将read map中的key值拷贝到dirty map中，这样dirty map就含有所有的值，下次升级为read map时直接进行地址拷贝

![图片描述](https://segmentfault.com/img/bV7l27?w=625&h=246)

##### entry结构

用于保存value的interface指针，通过atomic进行原子操作

```go
type entry struct {
    p unsafe.Pointer // *interface{}
}
```

##### Map结构

提供对外的方法，以及数据存储

```go
type Map struct {
    mu Mutex    

    //存储readOnly，不加锁的情况下，对其进行并发读取
    read atomic.Value // readOnly

    //dirty map用于存储写入的数据，能直接升级成read map
    dirty map[interface{}]*entry

    //misses 主要记录read读取不到数据加锁读取read map以及dirty map的次数.
    misses int
}
```

##### readOnly 结构

```go
// readOnly 通过原子操作存储在Map.read中, 
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}
```

##### Load方法

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        //加锁，然后再读取一遍read map中内容，主要防止在加锁的过程中，dirty map转换成read map，从而导致读取不到数据
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            //记录miss数, 在dirty map提升为read map之前,
            //这个key值都必须在加锁的情况下在dirty map中读取到
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}
```

##### Store方法

```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    //如果在read map读取到值，则尝试使用原子操作直接对值进行更新，更新成功则返回
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    //如果未在read map中读取到值或读取到值进行更新时更新失败，则加锁进行后续处理
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        //在检查一遍read，如果读取到的值处于删除状态，将值写入dirty map中
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        //使用原子操作更新key对应的值
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        //如果在dirty map中读取到值，则直接使用原子操作更新值
        e.storeLocked(&value)
    } else {
        //如果dirty map中不含有值，则说明dirty map已经升级为read map，或者第一次进入
        //需要初始化dirty map，并将read map的key添加到新创建的dirty map中
        if !read.amended {
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}
```

##### LoadOrStore方法

代码逻辑和Store类似

```go
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    // 不加锁的情况下读取read map
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        //如果读取到值则尝试对值进行更新或读取
        actual, loaded, ok := e.tryLoadOrStore(value)
        if ok {
            return actual, loaded
        }
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    // 在加锁的请求下在确定一次read map
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        actual, loaded, _ = e.tryLoadOrStore(value)
    } else if e, ok := m.dirty[key]; ok {
        actual, loaded, _ = e.tryLoadOrStore(value)
        m.missLocked()
    } else {
        if !read.amended {
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
        actual, loaded = value, false
    }
    m.mu.Unlock()

    return actual, loaded
}
```

##### Range方法

```go
func (m *Map) Range(f func(key, value interface{}) bool) {

    //先获取read map中值
    read, _ := m.read.Load().(readOnly)
    //如果dirty map中还有值，则进行加锁检测
    if read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        
        if read.amended {
            //将dirty map中赋给read,因为dirty　map包含了所有的值
            read = readOnly{m: m.dirty}
            m.read.Store(read)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }

    //进行遍历
    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue
        }
        if !f(k, v) {
            break
        }
    }
}
```

##### Delete方法

```go
func (m *Map) Delete(key interface{}) {
    //首先获取read map
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        //加锁二次检测
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        //没有在read map中获取到值，到dirty map中删除
        if !ok && read.amended {
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    if ok {
        e.delete()
    }
}
```

#### 局限性

sync.map并不适合同时存在大量读写的场景，大量的写会导致read map读取不到数据从而加锁进行进一步读取，同时dirty map不断升级为read map

从而导致整体性能较低，特别是针对cache场景。针对append-only以及大量读，少量写场景使用sync.map则相对比较合适

对于map，还有一种基于hash的实现思路，具体就是对map加读写锁，但是分配n个map，根据对key做hash运算确定是分配到哪个map中。这样锁的消耗就降到了1/n(理论值)

### sync.Cond

我需要完成一项任务，但是这项任务需要满足一定条件才可以执行，否则我就等着。那我可以怎么获取这个条件呢？一种是循环去获取，一种是条件满足的时候通知我就可以了。显然第二种效率高很多。通知的方式的话，golang里面通知可以用channel的方式。

但是channel的方式的话还是比较适用于一收一发的场景，多收一发不太适合。

sync.Cond就是用于实现条件变量的，是基于sync.Mutex的基础上，增加了一个通知队列，通知的线程会从通知队列中唤醒一个或多个被通知的线程。

```
sync.NewCond(&mutex)：生成一个cond，需要传入一个mutex，因为阻塞等待通知的操作以及通知解除阻塞的操作就是基于sync.Mutex来实现的。
sync.Wait()：用于等待通知
sync.Signal()：用于发送单个通知
sync.Broadcat()：用于广播
```

#### 结构体

```go
type Cond struct {
	noCopy noCopy  // noCopy可以嵌入到结构中，在第一次使用后不可复制,使用go vet作为检测使用
	L Locker // 根据需求初始化不同的锁，如*Mutex 和 *RWMutex
	notify  notifyList  // 通知列表,调用Wait()方法的goroutine会被放入list中,每次唤醒,从这里取出
	checker copyChecker // 复制检查,检查cond实例是否被复制
}
```

#### Wait函数

```go
func (c *Cond) Wait() {
    // 检查c是否是被复制的，如果是就panic
	c.checker.check()
	// 将当前goroutine加入等待队列
	t := runtime_notifyListAdd(&c.notify)
	// 解锁
	c.L.Unlock()
	// 等待队列中的所有的goroutine执行等待唤醒操作
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```

功能： 必须获取该锁之后才能调用Wait()方法，Wait方法在调用时会释放底层锁Locker，并且将当前goroutine挂起，直到另一个goroutine执行Signal或者Broadcase，该goroutine才有机会重新唤醒，并尝试获取Locker，完成后续逻辑。

也就是在等待被唤醒的过程中是不占用锁Locker的，这样就可以有多个goroutine可以同时处于Wait（等待被唤醒的状态）

#### Signal函数

```go
func (c *Cond) Signal() {
    // 检查c是否是被复制的，如果是就panic
	c.checker.check()
	// 通知等待列表中的一个 
	runtime_notifyListNotifyOne(&c.notify)
}
```

功能：唤醒等待队列中的一个goroutine，一般都是任意唤醒队列中的一个goroutine

#### Broadcast函数

```go
func (c *Cond) Broadcast() {
    // 检查c是否是被复制的，如果是就panic
	c.checker.check()
	// 唤醒等待队列中所有的goroutine
	runtime_notifyListNotifyAll(&c.notify)
}
```

功能：唤醒等待队列中的所有goroutine

#### 栗子

```go
func syncCondExplain() {
	m := sync.Mutex{}
	c := sync.NewCond(&m)

	// Tip: 主协程先获得锁
	c.L.Lock()
	go func() {
		// Tip: 协程一开始无法获得锁
		c.L.Lock()
		defer c.L.Unlock()
		fmt.Println("3. 该协程获得了锁")
		time.Sleep(2 * time.Second)
		// Tip: 通过notify进行广播通知
		c.Broadcast()
		fmt.Println("4. 该协程执行完毕，即将执行defer中的解锁操作")
	}()
	fmt.Println("1. 主协程获得锁")
	time.Sleep(1 * time.Second)
	fmt.Println("2. 主协程依旧抢占着锁获得锁")
	// Tip: 看一下Wait的大致实现，可以了解到，它是先释放锁，直到收到了notify，又进行加锁
	c.Wait()
	// Tip: 记得释放锁
	c.L.Unlock()
	fmt.Println("Done")
}

func syncCond() {
	lock := sync.Mutex{}
	cond := sync.NewCond(&lock)

	for i := 0; i < 5; i++ {
		go func(i int) {
			cond.L.Lock()
			defer cond.L.Unlock()
			cond.Wait()
			fmt.Printf("No.%d Goroutine Receive\n", i)
		}(i)
	}
	time.Sleep(time.Second)
	cond.Broadcast()
	//cond.Signal()
	time.Sleep(time.Second)
}
```

### sync.Pool

使用 Go 开发一个高性能的应用程序的话，就必须考虑垃圾回收给性能带来的影响。因为Go 在垃圾回收的时候会有一个STW（stop-the-world，程序暂停）的时间，并且如果对象太多，做标记也需要时间。

所以如果采用对象池来创建对象，增加对象的重复利用率，使用的时候就不必在堆上重新创建对象可以节省开销。

在Go中，sync.Pool提供了对象池的功能。它对外提供了三个方法：New、Get 和 Put。

##### 栗子

```go
var pool *sync.Pool

type Person1 struct {
	Name string
}

func init()  {
	pool = &sync.Pool{
		New: func() interface{} {
			fmt.Println("creating a new person")
			return new(Person1)
		},
	}
}

func main() {
	person := pool.Get().(*Person1)
	fmt.Println(person)
	person.Name = "first"
	pool.Put(person)

	fmt.Println(pool.Get().(*Person1))
	fmt.Println(pool.Get().(*Person1))
}
```

这里我用了init方法初始化了一个pool，然后get了三次，put了一次到pool中，如果pool中没有对象，那么会调用New函数创建一个新的对象，否则会从put进去的对象中获取。

#### 源码分析

- sync.Pool这玩意儿存的是一堆 临时对象
- 根据上一条的 临时对象 的含义就是sync.Pool里面存储的相关对象
- 这些 临时对象 随时可能被抛弃掉，这个抛弃不是指的后面说的GC清除，而是直接将这个 临时对象 抛弃不要将其设置为nil
- 另外，sync.Pool里面的 临时对象 也可随时会被GC清除，但是GC清除的前提是这个 临时对象 没有被任何除sync.Pool之外的东西引用，才会被GC清除。
- 一个sync.Pool可以安全地同时供多个Goroutine使用。每个sync.Pool都是绑定其对应的GMP模型中的P的（默认读者已经知道了GMP是个啥）。

##### 结构体

```go
type Pool struct {
	// noCopy 结构体，从而能看出 sync.Pool与Mutex等一样是不能复制的
	noCopy noCopy
	// local 字段 存储的是一个 存储  [P]poolLocal 类型的数组指针，其中P代表runtime.GOMAXPROCS()设置的值
	// 这是一个本地的池，几乎所有临时对象的存取都在这个字段完成
	local     unsafe.Pointer 
	// local存存储  [P]poolLocal的P大小 代表runtime.GOMAXPROCS()设置的值 
	localSize uintptr        

	// 这两个字段与local两个字段的存储东西相同，但是区别是
	// victim存储的是local抛弃下的数组，随时会被gc清除
	// 但是也有可能被捡回去重新使用
	// 可以把这个字段理解为一个随时抛弃随时捡起的垃圾堆
	victim     unsafe.Pointer  	
	victimSize uintptr         
	
	// 这个是唯一开放的字段，用于在初始化的时候传入构建新临时对象的函数
	// 如果这个字段为nil 在调用Get的且没有可用临时对象的时候就不会创建新对象而是返回nil
	New func() interface{}
}

// 这个结构体是一个存储着临时对像和全局对象链的结构体
 type poolLocalInternal struct {
	 // 临时对象存储在这里 这个对象只会被一个P 使用因此无需加锁
	private interface{}  
	// 这是一个无锁队列，有点类似GMP模型里面的全局任务队列（概念类似），当private 就回去里面取
	//hared，可以由任意的 P 访问，但是只有本地的 P 才能 pushHead/popHead，其它 P 可以 popTail，
	shared  poolChain   
}
// 一个P对应一个该结构体
type poolLocal struct {
	poolLocalInternal

	// 做了内存对齐
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

结构体中，只有`New`这个字段是可以包外访问的，这个字段需要在初始化`sync.Pool`时候传入一个用于生成临时对象的函数，如果要不传入的话当`Get`在`private`与`shared`都没获取到临时对象时，会返回`nil`而不是新建。

##### poolCleanup

sync.Pool会在不定时间的时候对已创建对象进行清除和GC，这就需要用到sync.Pool.poolCleanup函数，其会在GC开始时STW阶段被调用，他将sync.Pool中victim中的对象移除，然后把 local的数据给victim，这样的话，local就会被清空，而 victim就像一垃圾堆，里面的东西可能会被当做垃圾丢弃了，但是里面有用的东西也可能被捡回来重新使用。

```go
func poolCleanup() {
	// oldPools 是一个全局的[]*Pool变量
	// 存的是
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// 移动local 到 victim
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// 此时 所有的 Pool 里面的victim 都是no nil
	// 而local 是nil
	oldPools, allPools = allPools, nil
}
```

那么什么时候垃圾堆`victim`中的对象会重新使用呢？就是当`Get`时在`private`与`shared`都没获取到临时对象时会去垃圾堆`victim`找，如果找到了，该对象下次`Put`回的时候就不会放到垃圾堆`victim`里了。

##### Get

接下来看一下最重要的一个接口`Get`的实现，其可以返回一个可用的临时对象，如果有`New`这个字段如果初始化时未被传入，则当`Get`在`private`与`shared`都没获取到临时对象时，会返回`nil`而不是新建。

```go
func (p *Pool) Get() interface{} {
	// pin 函数是用于将当前goroutine固定在当前的P上 
	// 为的是防止突然的上下文切换被其他的P执行了
	l, pid := p.pin()
	// 获取当前本地的 临时对象
	x := l.private
	// 临时对象 设为nil
	l.private = nil
	// 如果本地没有
	if x == nil {
		// 就去自己的shared里面找，因为是自己的所以从Head处获取
		// 如果是别人的就从Tail处获取
		x, _ = l.shared.popHead()
		// 还是没有
		if x == nil {
		// 去其他的P的poolLocalInternal里去 “偷”
			x = p.getSlow(pid)
		}
	}
	// 和pin 是相反的
	runtime_procUnpin()
 	 // 如果还是没找到，并且New被设置了就新建一个 否则返回nil
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

// 去别的P的poolLocalInternal“偷”
func (p *Pool) getSlow(pid int) interface{} {
 	// 获取有多少个p
 	size := atomic.LoadUintptr(&p.localSize) 
 	 // 获取最开始的指针
	locals := p.local                         
	// 每个P的poolLocalInternal的share都看看 看看有没有可“偷”的临时对象，有就返回
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
			// 因为是别人的shared所以就从Tail处获取
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}
	
	// 没有就去victim 垃圾堆里面去找，找的方式和 “偷”一样
	 size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	// 先从垃圾堆的 private 找 没有就去垃圾堆的shared去“偷”
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}
	// 如果垃圾堆中都没有，则把这个victim标记为空，以后的查找就可以忽略
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

主要有以下几个步骤：

1. 先从当前P的poolLocal里面找看看private是否有临时对象可以返回，有的话返回，没有的话查一下自己的shared有没有。
2. 如果没有就去其他P的poolLocal的shared里找看看有没有
3. 如果没有就去垃圾堆victim里去找
4. 如果垃圾堆也没有，就看看是否初始化了New，初始化了就重新创建一个，否则返回nil。

##### Put

```go
func (p *Pool) Put(x interface{}) {
// 如果返回的x是nil 直接忽略
	if x == nil {
		return
	}
	 // 同样将当前goroutine固定在当前的P上 
	l, _ := p.pin()
	// 如果当前P的private是nil 就放在上面
	if l.private == nil {
		l.private = x
		x = nil
	}
	// 如果不是那就放到当前P的shared队列头上
	if x != nil {
		l.shared.pushHead(x)
	}
		// 和pin 是相反的
	runtime_procUnpin()
}
```

1. 如果传入对象是 `nil`那就不要他，要他也没用。
2. 如果当前`private`是`nil`就直接把传入对象赋值给他
3. 否则，就放入`shared`的头部，供日后使用

