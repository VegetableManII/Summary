# GO语言高级编程

## go中常见的坑

1. **可变参数为空接口类型**时，传入空接口的切片时要注意参数展开问题

2. **数组是值传递**而且不同大小的数组是不同的参数类型

3. **返回值屏蔽**，在局部作用域中同名的局部变量会屏蔽命名返回值

4. **recover必须在defer函数中直接调用**，recover捕获的是祖父级别的异常，直接调用无效，在defer中嵌套函数使用recover也无效

5. goroutine会因为**main函数提前退出**而无法保证完成任务

6. 通过sleep或gosched来阻塞主函数等待并发运行的结束

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

7. **独占CPU导致其他goroutine饿死**，goroutine协作式抢占调度，G本身不会主动放弃CPU

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

8. **不同goroutine之间不满足顺序一致性模型**

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

9. **在循环内部执行defer**

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

10. **空指针和空接口不等价**，例如返回一个错误指针而不是一个空的error接口，总是会返回non-nil  error

11. **对象内存地址会变化**，不能通过从其他非指针类型生成指针类型

12. **goroutine泄露**，可以使用cancelCtx来通知goroutine退出

13. **go中的泛型**，append函数的实现