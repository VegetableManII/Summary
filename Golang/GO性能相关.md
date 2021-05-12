# GO高性能编程

## 减少[]byte的分配

- 字节切片的**RESET**推荐使用`s = s[:0]`的方式

## 去接口，内联

- 在继承接口时，使用**实现了接口的对象指针**代替**接口类型**，提升性能

> https://colobu.com/2019/12/31/small-changes-big-improvement/

## 利用CPU缓存

> https://pengrl.com/p/9125/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io

- CPU cache line
  - CPU cache在缓存数据时，并不是以单个字节为单位缓存的，而是以**CPU cache line**大小为单位缓存，CPU cache line在一般的x86环境下为64字节。即使从内存读取1个字节的数据，也会将邻近的64个字节一并缓存至CPU cache中。
- false sharing
  - 由于一个CPU核在读取一个变量时，以cache line的方式将后续的变量也读取进来，缓存在自己这个核的cache中，而后续的变量也可能被其他CPU核并行缓存。当前面的CPU对前面的变量进行写入时，该变量同样是以cache line为单位写回内存。此时，在其他核上，尽管缓存的是该变量之后的变量，但是由于没法区分自身变量是否被修改，所以它只能认为自己的缓存失效，重新从内存中读取。

- 在高性能系统编程场景下，一般解决false sharing的方法是，在变量间添加无意义的填充数据（**cache line padding**）。需要高频并发读写的不同变量，不出现在一个cache line中。适用于多个相邻变量频繁被并发读写的场景

- go中的应用

  - Timer定时器

  ```go
  const timersLen = 64
  // pad 用作cache line的优化作用
  // 匿名结构体对timersBucket的封装，相当于在原本一个接一个的timersBucket数组元素之间，插入了pad。
  // 从而保证不同的timersBucket对象不会出现在同一个cache line中。
  var timers [timersLen]struct {
    timersBucket
    pad [cpu.CacheLinePadSize - unsafe.Sizeof(timersBucket{})%cpu.CacheLinePadSize]byte
  }
  ```

  - mheap类型

  ```go
  // pad 用作cache line优化
  // mcentral 同理 Timer 中的做法，在数组中每两个mcentral中插入pad
  type mheap struct {
    // ...
    central [numSpanClasses]struct {
      mcentral mcentral
      pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
    }
    // ...
  }
  ```

## 禁止使用没有超时的 I/O

## 优先使用strconv而不是fmt

## 避免字符串到字节的转换

## 不要在文件操作循环中使用defer

- 在文件操作循环中使用`defer`，会导致文件描述符耗尽。因为循环中在所有文件都被处理之前不会调用`defer`释放资源

## for和range性能比较

- 当for和range (只迭代下标时) 性能上无差异，如遍历 [ ]struct{}、[ ]int等
- range 对每个迭代值都创建了一个拷贝。因此如果每次迭代的值内存占用很小的情况下，for 和 range 的性能几乎没有差异，但是如果每个迭代值内存占用很大
- 使用 range 同时迭代下标和值，则需要将切片/数组的元素改为指针；使用 range，建议只迭代下标，通过下标访问迭代值

## 反射性能

- 对于一个普通的拥有 4 个字段的结构体 `Config` 来说，使用反射给每个字段赋值，相比直接赋值，性能劣化约 100 - 1000 倍。其中，`FieldByName` 的性能相比 `Field` 劣化 10 倍

```go
func BenchmarkSet(b *testing.B) {
	config := new(Config)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		config.Name = "name"
		config.IP = "ip"
		config.URL = "url"
		config.Timeout = "timeout"
	}
}

func BenchmarkReflect_FieldSet(b *testing.B) {
	typ := reflect.TypeOf(Config{})
	ins := reflect.New(typ).Elem()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ins.Field(0).SetString("name")
		ins.Field(1).SetString("ip")
		ins.Field(2).SetString("url")
		ins.Field(3).SetString("timeout")
	}
}

func BenchmarkReflect_FieldByNameSet(b *testing.B) {
	typ := reflect.TypeOf(Config{})
	ins := reflect.New(typ).Elem()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ins.FieldByName("Name").SetString("name")
		ins.FieldByName("IP").SetString("ip")
		ins.FieldByName("URL").SetString("url")
		ins.FieldByName("Timeout").SetString("timeout")
	}
}
```

```shell
$ go test -bench="Set$" .          
goos: darwin
goarch: amd64
pkg: example/hpg-reflect
BenchmarkSet-8                          1000000000               0.302 ns/op
BenchmarkReflect_FieldSet-8             33913672                34.5 ns/op
BenchmarkReflect_FieldByNameSet-8        3775234               316 ns/op
PASS
ok      example/hpg-reflect     3.066s
```

- `FieldByName`调用链

  (v Value) FieldByName -> (t *rtype) FieldByName -> (t *structType) FieldByName

-  `(t *structType) FieldByName` 中使用 for 循环，逐个字段查找，字段名匹配时返回。也就是说，在反射的内部，字段是按顺序存储的，因此按照下标访问查询效率为 O(1)，而按照 `Name` 访问，则需要遍历所有字段，查询效率为 O(N)。结构体所包含的字段(包括方法)越多，那么两者之间的效率差距则越大。

#### 性能的提高

- 在序列化和反序列化时尽量避免使用反射，使用提供的json库
- 缓存，利用map将结构体中的Name和Index保存起来

## 读写锁和互斥锁的比较

- 读写比为 9:1 时，读写锁的性能约为互斥锁的 8 倍
- 读写比为 1:9 时，读写锁性能相当
- 读写比为 5:5 时，读写锁的性能约为互斥锁的 2 倍

## 如何退出协程

#### 超时场景

##### time.After 超时控制

```go
func doBadthing(done chan bool) {
	time.Sleep(time.Second)
	done <- true
}

func timeout(f func(chan bool)) error {
	done := make(chan bool)
	go f(done)
	select {
	case <-done:
		fmt.Println("done")
		return nil
	case <-time.After(time.Millisecond):
		return fmt.Errorf("timeout")
	}
}

func test(t *testing.T, f func(chan bool)) {
	t.Helper()
	for i := 0; i < 1000; i++ {
		timeout(f)
	}
	time.Sleep(time.Second * 2)
	t.Log(runtime.NumGoroutine())
}

func TestBadTimeout(t *testing.T)  { test(t, doBadthing) }
```

- 利用 `time.After` 启动了一个异步的定时器，返回一个 channel，当超过指定的时间后，该 channel 将会接受到信号。
- 启动了子协程执行函数 f，函数执行结束后，将向 channel `done` 发送结束信号。
- 使用 select 阻塞等待 `done` 或 `time.After` 的信息，若超时，则返回错误，若没有超时，则返回 nil。
- `done` 是一个无缓冲区的 channel，如果没有超时，`doBadthing` 中会向 done 发送信号，`select` 中会接收 done 的信号，因此 `doBadthing` 能够正常退出，子协程也能够正常退出
- 当超时发生时，select 接收到 `time.After` 的超时信号就返回了，`done` 没有了接收方(receiver)，而 `doBadthing` 在执行 1s 后向 `done` 发送信号，由于没有接收者且无缓存区，发送者(sender)会一直阻塞，导致协程不能退出

##### 如何避免

- 创建有缓冲的channel，缓冲区设置为 1 即使没有接收方，发送方也不会阻塞

- 使用select尝试发送

  ```go
  func doGoodthing(done chan bool) {
  	time.Sleep(time.Second)
  	select {
  	case done <- true:
  	default:
  		return
  	}
  }
  
  func TestGoodTimeout(t *testing.T) { test(t, doGoodthing) }
  ```

#### 复杂场景

```go
func do2phases(phase1, done chan bool) {
	time.Sleep(time.Second) // 第 1 段
	select {
	case phase1 <- true:
	default:
		return
	}
	time.Sleep(time.Second) // 第 2 段
	done <- true
}

func timeoutFirstPhase() error {
	phase1 := make(chan bool)
	done := make(chan bool)
	go do2phases(phase1, done)
	select {
	case <-phase1:
		<-done
		fmt.Println("done")
		return nil
	case <-time.After(time.Millisecond):
		return fmt.Errorf("timeout")
	}
}

func Test2phasesTimeout(t *testing.T) {
	for i := 0; i < 1000; i++ {
		timeoutFirstPhase()
	}
	time.Sleep(time.Second * 3)
	t.Log(runtime.NumGoroutine())
}
```

#### 可以强制kill goroutine吗？

- 不能，goroutine 只能自己退出，而不能被其他 goroutine 强制关闭或杀死。

#### goroutine的一些使用建议

- 尽量使用非阻塞 I/O（非阻塞 I/O 常用来实现高性能的网络库），阻塞 I/O 很可能导致 goroutine 在某个调用一直等待，而无法正确结束。
- 业务逻辑总是考虑退出机制，避免死循环。
- 任务分段执行，超时后即时退出，避免 goroutine 无用的执行过多，浪费资源。

#### 其他场景

- channel忘记关闭的陷阱

  ```go
  func do(taskCh chan int) {
  	for {
  		select {
  		case t := <-taskCh:
  			time.Sleep(time.Millisecond)
  			fmt.Printf("task %d is done\n", t)
  		}
  	}
  }
  
  func sendTasks() {
  	taskCh := make(chan int, 10)
  	go do(taskCh)
  	for i := 0; i < 1000; i++ {
  		taskCh <- i
  	}
  }
  
  func TestDo(t *testing.T) {
      t.Log(runtime.NumGoroutine())
      sendTasks()
  	time.Sleep(time.Second)
  	t.Log(runtime.NumGoroutine())
  }
  ```

  ![](./GO性能相关.assets/channel.png)

- 改进

  ```go
  func doCheckClose(taskCh chan int) {
  	for {
  		select {
  		case t, beforeClosed := <-taskCh:
  			if !beforeClosed {
  				fmt.Println("taskCh has been closed")
  				return
  			}
  			time.Sleep(time.Millisecond)
  			fmt.Printf("task %d is done\n", t)
  		}
  	}
  }
  
  func sendTasksCheckClose() {
  	taskCh := make(chan int, 10)
  	go doCheckClose(taskCh)
  	for i := 0; i < 1000; i++ {
  		taskCh <- i
  	}
  	close(taskCh)
  }
  
  func TestDoCheckClose(t *testing.T) {
  	t.Log(runtime.NumGoroutine())
  	sendTasksCheckClose()
  	time.Sleep(time.Second)
  	runtime.GC()
  	t.Log(runtime.NumGoroutine())
  }
  ```

#### 通道关闭的原则

- 暴力关闭

  ```go
  func SafeClose(ch chan T) (justClosed bool) {
  	defer func() {
  		if recover() != nil {
  			// 一个函数的返回结果可以在defer调用中修改。
  			justClosed = false
  		}
  	}()
  
  	// 假设ch != nil。
  	close(ch)   // 如果 ch 已关闭，将 panic
  	return true // <=> justClosed = true; return
  }
  ```

- 礼貌关闭

  ```go
  type MyChannel struct {
  	C    chan T
  	once sync.Once
  }
  
  func NewMyChannel() *MyChannel {
  	return &MyChannel{C: make(chan T)}
  }
  
  func (mc *MyChannel) SafeClose() {
  	mc.once.Do(func() {
  		close(mc.C)
  	})
  }
  ```

- 优雅的方式

  - 情形一：M个接收者和一个发送者，发送者通过关闭用来传输数据的通道来传递发送结束信号。
  - 情形二：一个接收者和N个发送者，此唯一接收者通过关闭一个额外的信号通道来通知发送者不要再发送数据了。
  - 情形三：M个接收者和N个发送者，它们中的任何协程都可以让一个中间调解协程帮忙发出停止数据传送的信号。

## 控制协程的并发数量

- ### 利用channel的缓存区

  ```go
  func main() {
  	var wg sync.WaitGroup
  	ch := make(chan struct{}, 3)
  	for i := 0; i < 10; i++ {
  		ch <- struct{}{}
  		wg.Add(1)
  		go func(i int) {
  			defer wg.Done()
  			log.Println(i)
  			time.Sleep(time.Second)
  			<-ch
  		}(i)
  	}
  	wg.Wait()
  }
  ```

  - `make(chan struct{}, 3)` 创建缓冲区大小为 3 的 channel，在没有被接收的情况下，至多发送 3 个消息则被阻塞。
  - 开启协程前，调用 `ch <- struct{}{}`，若缓存区满，则阻塞。
  - 协程任务结束，调用 `<-ch` 释放缓冲区。
  - `sync.WaitGroup` 并不是必须的，例如 http 服务，每个请求天然是并发的，此时使用 channel 控制并发处理的任务数量，就不需要 `sync.WaitGroup`。

- ### 利用第三方库

  - Jeffail/tunny
  - panjf2000/ants

- ### 调整系统资源的上限（例如： too many open files、out of memory）

  - ulimit -n [设置打开的文件句柄的个数]

  - 虚拟内存

    - 创建交换分区

      ```shell
      udo fallocate -l 20G /mnt/.swapfile # 创建 20G 空文件
      sudo mkswap /mnt/.swapfile    # 转换为交换分区文件
      sudo chmod 600 /mnt/.swapfile # 修改权限为 600
      sudo swapon /mnt/.swapfile    # 激活交换分区
      free -m # 查看当前内存使用情况(包括交换分区)
      ```

    - 关闭交换分区

      ```shell
      sudo swapoff /mnt/.swapfile
      rm -rf /mnt/.swapfile
      ```

## sync.Pool

- **保存和复用临时对象，减少内存分配，降低 GC 压力**

- ###### json 的反序列化在文本解析和网络通信过程中非常常见，当程序并发度非常高的情况下，短时间内需要创建大量的临时对象。而这些对象是都是分配在堆上的，会给 GC 造成很大压力，严重影响程序的性能。

- sync.Pool 是可伸缩的，同时也是并发安全的，其大小仅受限于内存的大小。sync.Pool 用于存储那些被分配了但是没有被使用，而未来可能会使用的值。这样就可以不用再次经过内存分配，可直接复用已有对象，减轻 GC 的压力，从而提升系统的性能。

- sync.Pool 的大小是可伸缩的，高负载时会动态扩容，存放在池中的对象如果不活跃了会被自动清理。

- 声明对象池，如果对象池中没有对象时，通过new创建

  ```go
  var studentPool = sync.Pool{
      New: func() interface{} { 
          return new(Student) 
      },
  }
  ```

- Get & Put

  ```go
  stu := studentPool.Get().(*Student)
  json.Unmarshal(buf, stu)
  studentPool.Put(stu)
  ```

  - `Get()` 用于从对象池中获取对象，因为返回值是 `interface{}`，因此需要类型转换。
  - `Put()` 则是在对象使用完毕后，返回对象池。

## sync.Once

- `sync.Once` 是 Go 标准库提供的使函数只执行一次的实现，常应用于单例模式，例如初始化配置、保持数据库连接等。作用与 `init` 函数类似，但有区别。
  - init 函数是当所在的 package 首次被加载时执行，若迟迟未被使用，则既浪费了内存，又延长了程序加载时间。
  - sync.Once 可以在代码的任意位置初始化和调用，因此可以延迟到使用时再执行，并发场景下是线程安全的。

- 控制变量的初始化，`sync.Once` 仅提供了一个方法 `Do`，参数 f 是对象初始化函数。

  ```go
  func (o *Once) Do(f func())
  ```

  - 当且仅当第一次访问某个变量时，进行初始化（写）；
  - 变量初始化过程中，所有读都被阻塞，直到初始化完成；
  - 变量仅初始化一次，初始化完成后驻留在内存里。

- 实现原理

  ```go
  type Once struct {
      // done indicates whether the action has been performed.
      // It is first in the struct because it is used in the hot path.
      // The hot path is inlined at every call site.
      // Placing done first allows more compact instructions on some architectures (amd64/x86),
      // and fewer instructions (to calculate offset) on other architectures.
      done uint32
      m    Mutex
  }
  ```

  - `done uint32`来实现标记是否初始化
  - 互斥锁

  1. 热路径(hot path)是程序非常频繁执行的一系列指令，sync.Once 绝大部分场景都会访问 `o.done`，在热路径上是比较好理解的，如果 hot path 编译后的机器码指令更少，更直接，必然是能够提升性能的。
  2. 为什么放在第一个字段就能够减少指令呢？因为结构体第一个字段的地址和结构体的指针是相同的，如果是第一个字段，直接对结构体的指针解引用即可。如果是其他的字段，除了结构体指针外，还需要计算与第一个值的偏移(calculate offset)。在机器码中，偏移量是随指令传递的附加值，CPU 需要做一次偏移值与指针的加法运算，才能获取要访问的值的地址。因为，访问第一个字段的机器代码更紧凑，速度更快。

## sync.Cond条件变量

- ###### `sync.Cond` 条件变量用来协调想要访问共享资源的那些 goroutine，当共享资源的状态发生变化的时候，它可以用来通知被互斥锁阻塞的 goroutine。基于互斥锁，但是互斥锁通常用来保护临界区和共享资源，条件变量用来协调想要访问共享变量的goroutine

- ##### 创建实例NewCond，`func NewCond(l Locker) *Cond`

- ##### Broadcast 广播唤醒所有

- ##### Signal 唤醒一个协程

- ##### Wait 等待

  调用 Wait 会自动释放锁 c.L，并挂起调用者所在的 goroutine，因此当前协程会阻塞在 Wait 方法调用的地方。如果其他协程调用了 Signal 或 Broadcast 唤醒了该协程，那么 Wait 方法在结束阻塞时，会重新给 c.L 加锁，并且继续执行 Wait 后面的代码。

  ```go
  c.L.Lock()
  // 对条件的检查，使用了 for !condition() 而非 if，
  // 是因为当前协程被唤醒时，条件不一定符合要求，需要再次 Wait 等待下次被唤醒。
  for !condition() {
      c.Wait()
  }
  ... make use of condition ...
  c.L.Unlock()
  ```

  

## 协程泄露

- 协程泄露是指协程创建后，长时间得不到释放(阻塞)，并且还在不断地创建新的协程，最终导致内存耗尽，程序崩溃。常见的导致协程泄露的场景有以下几种：
  - 缺少收发器，导致发，收阻塞
  - 死锁
  - 无限循环

## 编译优化

#### 逃逸分析对性能的影响

- #### 逃逸分析：编译器决定内存分配位置的方式，就称之为逃逸分析(escape analysis)。逃逸分析由编译器完成，作用于编译阶段。

  - 指针逃逸：函数返回局部成员的引用
  - interface{}动态类型逃逸
  - 栈空间不足，64位系统栈空间8M(通过 ulimit -a查看)、goroutine的栈空间初始化为2K；在函数内部当切片大小达到一定大小或无法确定切片大小时逃逸到堆上
  - 闭包

- #### 利用逃逸分析提升性能

  - 在对象频繁创建和删除的场景下，传递指针导致的 GC 开销可能会严重影响性能。
  - 对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。
  - 对于只读的占用内存较小的结构体，直接传值能够获得更好的性能。

#### 死码消除和调试模式

- 使用常量代替变量声明，可以编译后的二进制文件更小

- 可推断的局部变量，函数内部的局部变量的修改只发生在函数内部比较好推断其值，包内的变量的修改不好进行推断，该变量的修改可能发生在：

  - 包初始化函数init()中，init()可能是多个，且可能位于不同的.go源文件中
  - 包内的其他函数
  - 首字母大写的Public变量，其他包引用时可修改

- #### 调试

- 定义全局常量 debug，值设置为 `false`，在需要增加调试代码的地方，使用条件语句 `if debug` 包裹

  - 正常编译，常量 debug 始终等于 `false`，调试语句在编译过程中会被消除，不会影响最终的二进制大小，也不会对运行效率产生任何影响。

- 条件编译，build tags

  - 在debug版本文件中定义常量debug为true，release版本中定义debug为false

  - 在debug版本文件中添加注释 `// +build debug`表示在build tags中包含debug则此文件参与编译；

    同理，在release中`// +build !debug`表示不包含debug则此文件参与编译

## runtime.GOMAXPROCS(num int)

- `GOMAXPROCS` 限制的是同时执行用户态 Go 代码的操作系统线程的数量，但是对于被系统调用阻塞的线程数量是没有限制的。
- `GOMAXPROCS` 的默认值等于 CPU 的逻辑核数，同一时间，一个核只能绑定一个线程，然后运行被调度的协程。

## 常量

- Go 语言中，常量分为无类型常量和有类型常量两种，`const N = 100`，属于无类型常量，赋值给其他变量时，如果字面量能够转换为对应类型的变量，则赋值成功，例如，`var x int = N`。
- 对于有类型的常量 `const M int32 = 100`，赋值给其他变量时，需要类型匹配才能成功

```go
func main() {
	var a int8 = -1	  // int8 能表示的数字的范围是 [-2^7, 2^7-1]
  // 如果a定义为 const a int8 = -1 则发生编译错误，常量除以常量结果仍是常量，常量不允许溢出
	var b int8 = -128 / a		
  // -128 是无类型常量，转换为 int8，再除以变量 -1，结果为 128，常量除以变量，结果是一个变量
	fmt.Println(b)
  // 变量转换时允许溢出，符号位变为1，转为补码后恰好等于 -128
}
```

```basic
-1 :  11111111
00000001(原码)    11111110(取反)    11111111(加一)
-128：    
10000000(原码)    01111111(取反)    10000000(加一)

-1 + 1 = 0
11111111 + 00000001 = 00000000(最高位溢出省略)
-128 + 127 = -1
10000000 + 01111111 = 11111111
```

## defer

- defer 延迟调用时，需要保存函数指针和参数，因此**链式调用**的情况下，除了最后一个函数/方法外的函数/方法都会在调用时直接执行。

- ```go
  defer t.f(1).f(2) //会直接调用f(1)再延迟调用f2
  ```

