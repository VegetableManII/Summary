## 从硬盘读取内存，创建主线程和主进程，主线程分配栈空间，命令行参数，主线程挂入系统运行队列等待被调度

## 初始化m0和g0

> m保存工作线程相关信息（栈空间地址，fs寄存器，当前正在执行的g，是否空闲状态）
>
> 成员：
>
> 1. 指向g0的指针在调度时涉及到栈的切换，
> 2. 指向当前正在运行的goroutine的g对象的指针，
> 3. 指向绑定的p的指针，oldp保存的是进入系统调用之前的p，
> 4. spinning状态变量表示当前工作线程正在窃取goroutine，
> 5. park成员用于从系统调用中返回后发现无事可做而睡眠被其他线程唤醒（note机制），指向allm的指针
>
> g保存goroutine的相关信息（CPU寄存器的值、栈基地址栈顶地址等）
>
> 成员：
>
> 1. stack保存栈顶栈底的地址所以栈是可扩展的，
> 2. gobuf保存调度信息rsp寄存器(即sp成员保存栈顶地址)，rip寄存器(即pc成员保存下一条指令)的值上下文等，
> 3. 指向m的指针正在被哪一个m执行；
> 4. stackguard0和preempt成员变量用于调度
>
> p保存某个工作线程私有的局部g队列
>
> 1. 互斥锁——用于获取
> 2. 指向m的指针
> 3. 循环队列保存goroutine 数组大小256
> 4. gFree缓存g结构体对象的对象池
>
> schedt保存调度器状态以及全局g队列
>
> 1. 互斥锁——用于从全局队列获取g
> 2. 指向空闲的工作线程组成的链表的指针和空闲的工作线程的数量
> 3. 指向空闲的p结构体对象组成的链表的指针和空闲的p的数量
> 4. 指向所有g结构体对象组成的链表的指针，由runq成员来保存全局运行队列
> 5. gFree缓存g结构体对象的对象池
>
> allm 保存所有的m
>
> allp 保存所有的p
>
> allgs保存所有的g

*g0供runtime运行的栈，工作线程m0的本地存储存储g0的地址实现关联；初始化allp（可能再后面通过用户调用GOMAXPROCS更改大小），将allp[0]和m0绑定在一起以实现关联*

## 创建main goroutine

newproc()——>newproc1()

newproc1()的流程：

*在创建main goroutine时是在g0上创建不需要切换，使用go语句创建goroutine时newproc1()会调用systemstack实现切换到g0*

1. 从堆上分配一端2K空间作为栈使用，并为这个g的成员赋值，从newproc函数的栈拷贝参数到新创建的g的栈，将gobuf成员的pc字段设置为goexit函数而不是newproc传递进来的函数。
2. 然后调用gostartcall调整g栈空间，把goexit的第二条指令压入栈，重新设置pc字段为runtime.main，修改状态为 _Grunnable （此时 g 还未与 m 绑定）

## 调度main goroutine

1. mstart()/mstart1() [调用save] 保存g0的调度信息，调用schedule函数寻找需要运行的goroutine，程序加载时找到的是main goroutine。

   schedule()——>execute()——>gogo()——>用户goroutine——>runtime其他函数——>schedule()

2. 调用gogo函数从g0切换到main goroutine然后取出main goroutine的g结构体中保存的寄存器的值使用JUMP跳转到改地址执行。

3. 启动sysmon系统监控线程，负责gc、抢占调度以及netpoll等功能

4. main goroutine执行完毕直接调用exit系统调用退出进程其他goroutine会执行goexit进行清理

## 调度循环

#### schedule

- #### globrunqget()：从全局队列获取

- #### runqget()：从本地队列获取

#### excute

- 参数为需要调度运行的g
- 然后将g的状态从 _Grunnable修改为 _Grunning，关联g和m然后就可以通过m找到当前工作线程正在执行的goroutine

#### gogo

汇编代码。恢复 g 的gobuf的寄存器内容以及栈的切换跳转执行pc指向的指令地址

#### mcall

mcall属于runtime逻辑代码，在调用时会切换到g0栈：保存当前g的调度信息，把g0设置为当前线程的tls修改CPU寄存器指向g0的的栈。

## goroutine的退出

RET指令返回调用的goexit，goexit调用goexit1，使用mcall函数切换到g0栈，在g0栈调用goexit0。

goexit0将用户g状态从 _Grunning 变更为 _Gdead 用户g的一些字段清空，调用drog解除用户g和m之间的关系，保存g到freeg队列便于下次使用

## 调度策略

### 调度场景

- ##### 运行时间太长或处于系统调用

- ##### 主动让出CPU进行调度

- ##### 需要等待被调度（加锁、select、channel、网络阻塞）

### 被动调度

因读阻塞：goparkunlock（阻塞当前g）——gopark——mcall切换到g0——park_m（将g的状态改为 _Gwait）——dropg（解除m和g之间的关系）——schedule

唤醒：

- 以channel发送为例
- channel发送调用runtime.chansend1函数，如果能立即发送则立即返回如果不能则阻塞
- 立即发送调用send，send调用goready，goready调用ready函数，唤醒正在等待读的goroutine
  - ready先把需要唤醒的g状态设置为 _Grunnable 放入队列
  - 如果有空闲的p也没有处于spinning状态的线程调用wakeup唤醒空闲的p来工作
    - wakeup通过cas操作确认是否有其他工作线程处于spanning
    - 如果有则直接使用，没有调用startm创建一个工作线程
      - startm判断有无空闲的p，没有直接返回
      - 有空闲的p则从m的队列查找处于休眠状态的工作线程如果有则调用notewakeup唤醒
        - notewakeup
        - 工作线程会通过notesleep函数使自己睡眠作用在park成员上
        - 通过park成员将其唤醒（涉及到内核的进入）
      - 如果没有则调用newm创建工作线程并与之绑定
        - newm，从堆上分配一个m结构体，然后调用newm1，newm1调用newosproc，newosproc调用**clone函数**创建工作线程，从mstart开始执行进入调度循环

##### clone函数

- 准备好系统调用的参数，指定被创建的线程使用的栈（否则父子线程会共同使用父线程的栈导致混乱）
- 其他参数保存到CPU寄存器，此时保存在父线程的寄存器在创建完子线程之后内核会把寄存器的值复制一份给子线程
- 调用syscall指令进入内核，由内核完成线程的创建
- 线程创建完成后执行mstat函数，保存g0的信息，进入调度循环
- clone函数在子线程返回 0 ，在父线程返回子线程id保存到栈通过RET指令作为newosproc的返回值

### 主动调度

- 调用runtime.Gsched()——调用mcall
  - 保存线程信息
  - 恢复g0的sp、pc字段到CPU完成当前g到g0的切换
  - 在g0执行gosched_m函数，gosched_m调用**goshedlmpl**
    - **goshedlmpl**函数把调用runtime.Gsched的状态修改为 _Grunning 改为 _Grunnable
    - dropg解除关联
    - 调用globrunqput函数放入到全局队列

### 抢占调度

- sysmon监控线程每10s调用retake函数

- **retake函数**

- 只有 P 处于 _Psyscall(系统调用) 和 _Prunning(运行超过10ms) 状态才会被抢占

  - ##### _Prunning状态

    - ##### preemptone函数发出抢占请求

      设置g的抢占标志preempt以及stackguard0成员为stackPreempt(常量，非常大的整数)

    - 处理抢占请求 `morestack_noctxt()->morestack()->newstack()`

      - 保存调度信息，切换到g0执行newstack函数

      - ##### newstack()

        - 检查stackguard0是否可以被抢占
          - 如果可以则调用**gopreempt_m**函数
            - 调用**goschedlmpl**函数完成调度切换

  - ##### _Psyscall状态

    - goroutine正在内核进行系统调用（多条件判断是否需要抢占）
    - 条件：p对应的m处于系统调用超过10毫秒、没有空闲的p、p的运行队列有等待运行的goroutine
    - 通过CAS来修改p的状态来获取使用权（可能工作线程从系统调用中返回正在获取p的使用权）
      - 如果获得P的使用权
        - 判断是否需要开启新的工作线程接管
          - 判断条件
          - p 的本地运行队列或全局运行队列有待运行的goroutine
          - 需要帮助 gc
          - 所有其他 p 都在运行goroutine
          - 所有其他 p 都处于空闲状态但是需要监控网络读写事件
        - 不需要则直接挂入P的全局空闲队列

    **系统调用**

    - 