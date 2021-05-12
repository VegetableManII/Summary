# signalflight

### 高并发的运行过程

- 大量goroutine调用`(*Group).Do`或者`(*Group).DoChan`
- Group要先加锁，抢锁成功的goroutine创建**key-call**组合并让WaitGroup加1，解锁，然后去执行用户函数；
- 其他goroutine抢锁
  - **Do( )**
    - 抢到锁之后，等待计数加一然后解锁，调用`c.wg.Wait()`等待第一个goroutine的执行结果
  - **DoChan( )**
    - 抢到锁之后，等待计数加一，然后将自己的管道添加`append`到**call**的接收管道中，然后解锁。
    - `append`操作比较快，所以在第一个goroutine执行完用户函数之后需要再次抢锁之前，会积累一些无需执行用户函数的goroutine
- 第一个goroutine执行完用户函数，调用`c.wg.Done()`此时使用`Do()`等待的goroutine可以获得数据返回
- `c.wg.Done()`完成之后，第一个goroutine再次抢锁抢到之后，首先解除**key-call**组合，然后将结果依次发送给之前积累的每一个goroutine，发送完毕之后解锁。
  - 此时其他goroutine想要执行`(*Group).Do`或者`(*Group).DoChan`，会阻塞等待，上一轮的结果都发送完之后才开始新一轮的用户函数结果的并发读取

### 问题

- **Do**中`call`对象的创建只申请内存空间而不初始化，**DoChan**中`call`对象的创建会初始化切片

  `doCall`函数中遍历 nil 切片也不会导致 **panic**

- 第一个获取锁的goroutine执行完成用户操作之后，下一次获得锁的时间

## 源码

```go
// Group 代表一组操作，实现单件模式
type Group struct {
mu sync.Mutex       // protects m
m  map[string]*call // lazily initialized
}

// 代表一次用户操作，只有首次才会执行后面的直接获取结果
type call struct {
	wg sync.WaitGroup
  
	val interface{}
	err error
  // dups 记录等待个数或共享结果的消费者数量
	dups  int
	chans []chan<- Result
}
// 代表一次操作的返回结果
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}
```

```go
// Do 方法保证对于每一个 key 对应的用户提供的方法函数，
// 在一个时间内只执行一次，后面的执行直接获取结果值即可
// 第一次执行：创建 call 并保存到 group 中，call 的wg+1，执行其对应的操作
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err, true
	}
  // 创建的 call 对象中 error、chan切片、空接口都为nil
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

// 返回接收结果的管道和一个 bool 值，当bool值为true是函数是被立刻调用的
// 当bool值为false，函数不会被调用而是使用之前的结果
func (g *Group) DoChan(key string, fn func() (interface{}, error)) (<-chan Result, bool) {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		return ch, false
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	go g.doCall(c, key, fn)

	return ch, true
}

// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	c.val, c.err = fn()
	c.wg.Done()
// 如果是 Do 当c.wg.Done()之后其他goroutine就可以进行读取
// DoChan 需要再次抢锁进行数据的发送
	g.mu.Lock()
	delete(g.m, key)
	for _, ch := range c.chans {
		ch <- Result{c.val, c.err, c.dups > 0}
	}
	g.mu.Unlock()
}

func (g *Group) ForgetUnshared(key string) bool {
	g.mu.Lock()
	defer g.mu.Unlock()
	c, ok := g.m[key]
	if !ok {
		return true
	}
	if c.dups == 0 {
		delete(g.m, key)
		return true
	}
	return false
}
```