## go函数式选择模式

- 函数式选项模式

```go
type Option struct {
	A string
	B string
	C int
}

type OptionFunc func(*Option)

var (
	defaultOption = &Option{
		A: "A",
		B: "B",
		C: 100,
	}
)

func newOption(opts ...OptionFunc) (opt *Option) {
	opt = defaultOption
	for _, o := range opts {
		o(opt)
	}
	return
}

func WithA(a string) OptionFunc {
	return func(o *Option) {
		o.A = a
	}
}

func WithB(b string) OptionFunc {
	return func(o *Option) {
		o.B = b
	}
}

func WithC(c int) OptionFunc {
	return func(o *Option) {
		o.C = c
	}
}
```

## go并发安全的单例模式

```go
package singleton

import (
    "sync"
)

type singleton struct {}

var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
// sync.Once 的实现简易介绍
type Once struct {
	done uint32 // 标识动作是否已经执行
	m    Mutex
}
// check-lock-check 模式
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 { // check
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()                          // lock
	defer o.m.Unlock()
	
	if o.done == 0 {                    // check
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

## go性能分析pprof

- 工具型应用`runtime/pprof`

  - 开启CPU性能分析`pprof.StartCPUProfile(w io.Writer)`，执行结束后会生成报告文件使用`go tool pprof`工具可以解析文件进行CPU性能分析
  - 内存性能优化`pprof.WriteHeapProfile(w io.Writer)`，执行结束后使用`go tool pprof`进行性能分析。默认使用 `-inuse_space`统计，还可以使用`-inuse_objects`查看对象分配数量

- 服务型应用`net/http/pprof`

  - 可以直接使用`http.ListenAndServe(“0.0.0.0:8000”, nil)）`

  - 也可以手动注册路由

    ```go
    r.HandleFunc("/debug/pprof/", pprof.Index)
    r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    r.HandleFunc("/debug/pprof/profile", pprof.Profile)
    r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    r.HandleFunc("/debug/pprof/trace", pprof.Trace)
    ```

## go中的select实现优先级

- `select`不存在任何`case`，永久阻塞当前goroutine

- `select`只存在一个`case`，阻塞的发送/接收

- `select`存在多个`case`，随机选择一个满足条件的`case`运行

- `select` 存在 `default`，其他`case`都不满足时：执行`default`语句中的代码

- `for`和`LABLE`实现`select`优先级

  ```go
  // kubernetes/pkg/controller/nodelifecycle/scheduler/taint_manager.go 
  func (tc *NoExecuteTaintManager) worker(worker int, done func(), stopCh <-chan struct{}) {
  	defer done()
  
  	// 当处理具体事件的时候，我们会希望 Node 的更新操作优先于 Pod 的更新
  	// 因为 NodeUpdates 与 NoExecuteTaintManager无关应该尽快处理
  	// -- 我们不希望用户(或系统)等到PodUpdate队列被耗尽后，才开始从受污染的Node中清除pod。
  	for {
  		select {
  		case <-stopCh:
  			return
  		case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
  			tc.handleNodeUpdate(nodeUpdate)
  			tc.nodeUpdateQueue.Done(nodeUpdate)
  		case podUpdate := <-tc.podUpdateChannels[worker]:
  			// 如果我们发现了一个 Pod 需要更新，我么你需要先清空 Node 队列.
  		priority:
  			for {
  				select {
  				case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
  					tc.handleNodeUpdate(nodeUpdate)
  					tc.nodeUpdateQueue.Done(nodeUpdate)
  				default:
  					break priority
  				}
  			}
  			// 在 Node 队列清空后我们再处理 podUpdate.
  			tc.handlePodUpdate(podUpdate)
  			tc.podUpdateQueue.Done(podUpdate)
  		}
  	}
  }
  ```

## go中的unsafe.Pointer和uintptr

- ### unsafe.Pointer

  - 保存内存地址（包括动态链接的地址），表示指向任意类型的指针可以转换为任何类型或`uintptr`的指针值
  - 实现快速拷贝，两个指针共享同一内存空间，一方改变另一方也会受到影响

  ```go
  ptrT1 := &T1{}
  ptrT2 = (*T2)(unsafe.Pointer(ptrT1))
  ```

- ### uintptr

  - 一个数字， 表示内存中的一个地址编号
  - 当把`unsafe.Pointer`转换为`uintptr`之后，原来的指针不再引用指向的变量而且在转换回来之前，会被GC，解决方案可以使用`runtime.KeepAlive`，也可以在一个表达式中进行两次转换
  - `reflect.Value.Pointer`和`reflect.Value.UnsafeAddr`这两个方法都会返回`uintptr`类型，在使用时应该立即转换为`unsafe.Pointer`类型

## go中常见的坑

1. **可变参数为空接口类型**时，传入空接口的切片时要注意参数展开问题

2. `JSON`序列化时空接口存放的数字类型都会序列化成`float64`类型

3. **数组是值传递**而且不同大小的数组是不同的参数类型

4. **返回值屏蔽**，在局部作用域中同名的局部变量会屏蔽命名返回值

5. **recover必须在defer函数中直接调用**，recover捕获的是祖父级别的异常，直接调用无效，在defer中嵌套函数使用recover也无效

6. goroutine会因为**main函数提前退出**而无法保证完成任务

7. **select**会使当前goroutine永久进入阻塞且无法被唤醒

8. 通过sleep或gosched来阻塞主函数等待并发运行的结束

   ```go
   func main() {
       go println("hello")
       time.Sleep(time.Second)
   }
   func main() {
       go println("hello")
       runtime.Gosched()
   }
   ```

9. **独占CPU导致其他goroutine饿死**，goroutine协作式抢占调度，G本身不会主动放弃CPU

   ```go
   func main() {
       runtime.GOMAXPROCS(1)
   
       go func() {
           for i := 0; i < 10; i++ {
               fmt.Println(i)
           }
       }()
   
       for {} // 占用CPU
   }
   ```

   这里for循环不会阻塞而是一直运行，解决方案在for循环中添加runtime.Gosched()，或者for循环替换为select{}阻塞

10. **不同goroutine之间不满足顺序一致性模型**

   ```go
   var msg string
   var done bool
   
   func setup() {
       msg = "hello, world"
       done = true
   }
   
   func main() {
       go setup()
       for !done {
       }
       println(msg)
   }
   // 不能保证能成功打印hello，world
   // 解决方案
   var msg string
   var done = make(chan bool)
   
   func setup() {
       msg = "hello, world"
       done <- true
   }
   
   func main() {
       go setup()
       <-done
       println(msg)
   }
   ```

11. **在循环内部执行defer**

    ```go
    func main() {
        for i := 0; i < 5; i++ {
            f, err := os.Open("/path/to/file")
            if err != nil {
                log.Fatal(err)
            }
            defer f.Close()
        }
    }
    // defer只有在for函数退出时才执行，导致资源的延迟释放
    // 解决方案
    // 在for循环中添加局部函数
    func main() {
        for i := 0; i < 5; i++ {
            func() {
                f, err := os.Open("/path/to/file")
                if err != nil {
                    log.Fatal(err)
                }
                defer f.Close()
            }()
        }
    }
    ```

12. **空指针和空接口不等价**，例如返回一个错误指针而不是一个空的error接口，总是会返回non-nil  error

13. **对象内存地址会变化**，不能通过从其他非指针类型生成指针类型

14. **goroutine泄露**，可以使用cancelCtx来通知goroutine退出

15. **go中的泛型**，append函数的实现