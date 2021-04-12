# GO语言调度器

\* [GO语言调度器](#go语言调度器)

   \* [调度器数据结构](#调度器数据结构)

   \* [GO程序的运行流程](#go程序的运行流程)

​     \* [调度器的初始化](#调度器的初始化)

​     \* [main goroutine的创建](#main-goroutine的创建)

​     \* [调度main goroutine运行](#调度main-goroutine运行)

​     \* [非main goroutine的退出](#非main-goroutine的退出)

​     \* [调度策略](#调度策略)

​      \* [调度发生的条件](#调度发生的条件)

​      \* [被动调度](#被动调度)

​      \* [主动调度](#主动调度)

​      \* [抢占调度](#抢占调度)

​      \* [schedu函数](#schedu函数)

​        \* [从全局队列获取](#从全局队列获取)

​        \* [从本地运行队列获取](#从本地运行队列获取)

​        \* [从其他工作线程中窃取](#从其他工作线程中窃取)

   \* [循环调度](#循环调度)

## 调度器数据结构

1. g结构体

   goroutine的所有信息都保存在一个 g 对象中，当goroutine被调离CPU时，调度器负责把CPU寄存器的值保存在g对象的成员变量中，当goroutine被调度起来运行时，调度器负责把g对象的成员变量保存的寄存器的值复制到CPU的寄存器当中

   ```go
   //用于记录goroutine使用的栈的起始和结束位置
   type stack struct {  
       lo uintptr    // 栈顶，指向内存低地址
       hi uintptr    // 栈底，指向内存高地址
   }
   type gobuf struct {
       sp   uintptr  // 保存CPU的rsp寄存器的值
       pc   uintptr  // 保存CPU的rip寄存器的值
       g    guintptr // 记录当前这个gobuf对象属于哪个goroutine
       ctxt unsafe.Pointer
    
       // 保存系统调用的返回值，因为从系统调用返回之后如果p被其它工作线程抢占，
       // 则这个goroutine会被放入全局运行队列被其它工作线程调度，其它线程需要知道系统调用的返回值。
       ret  sys.Uintreg  
       lr   uintptr
    
       // 保存CPU的rip寄存器的值
       bp   uintptr // for GOEXPERIMENT=framepointer
   }
   // g结构体，它代表了一个goroutine
   type g struct {
       // 记录该goroutine使用的栈
       stack       stack   // offset known to runtime/cgo
       // 下面两个成员用于栈溢出检查，实现栈的自动伸缩，抢占调度也会用到stackguard0
       stackguard0 uintptr // offset known to liblink
       stackguard1 uintptr // offset known to liblink
   
       ......
    
       // 此goroutine正在被哪个工作线程执行
       m              *m      // current m; offset known to arm liblink
       // 保存调度信息，主要是几个寄存器的值
       sched          gobuf
    
       ......
       // schedlink字段指向全局运行队列中的下一个g，
       //所有位于全局运行队列中的g形成一个链表
       schedlink      guintptr
   
       ......
       // 抢占调度标志，如果需要抢占调度，设置preempt为true
       preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt
   
      ......
   }
   ```

   

2. m结构体

   保存工作线程中的相关信息，如栈的起止位置、当前正在执行的goroutine以及是否空闲等状态信息，同时通过指针维护与p结构体对象的绑定关系。每个工作线程都有唯一一个m结构体对象与之对应

   通过线程本地存储TLS实现，定义私有全局变量（在不同的工作线程中使用相同的全局变量名访问不同的m结构体对象）

   ```go
   type m struct {
       // g0主要用来记录工作线程使用的栈信息，在执行调度代码时需要使用这个栈
       // 执行用户goroutine代码时，使用用户goroutine自己的栈，调度时会发生栈的切换
       g0      *g     // goroutine with scheduling stack
   
       // 通过TLS实现m结构体对象与工作线程之间的绑定
       tls           [6]uintptr   // thread-local storage (for x86 extern register)
       mstartfn      func()
       // 指向工作线程正在运行的goroutine的g结构体对象
       curg          *g       // current running goroutine
    
       // 记录与当前工作线程绑定的p结构体对象
       p             puintptr // attached p for executing go code (nil if not executing go code)
       nextp         puintptr
       oldp          puintptr // the p that was attached before executing a syscall
      
       // spinning状态：表示当前工作线程正在试图从其它工作线程的本地运行队列偷取goroutine
       spinning      bool // m is out of work and is actively looking for work
       blocked       bool // m is blocked on a note
      
       // 没有goroutine需要运行时，工作线程睡眠在这个park成员上，
       // 其它线程通过这个park唤醒该工作线程
       park          note
       // 记录所有工作线程的一个链表
       alllink       *m // on allm
       schedlink     muintptr
   
       // Linux平台thread的值就是操作系统线程ID
       thread        uintptr // thread handle
       freelink      *m      // on sched.freem
   
       ......
   }
   ```

3. p结构体

   保存每个工作线程私有的局部goroutine运行队列，工作线程优先使用自己的局部运行队列，在必要情况下才去访问全局运行队列，尽量减少锁的冲突提高工作线程的并发性。每一个工作线程都会与一个p结构体对象的示例关联

   ```go
   type p struct {
       lock mutex
   
       status       uint32 // one of pidle/prunning/...
       link            puintptr
       schedtick   uint32     // incremented on every scheduler call
       syscalltick  uint32     // incremented on every system call
       sysmontick  sysmontick // last tick observed by sysmon
       m                muintptr   // back-link to associated m (nil if idle)
   
       ......
   
       // Queue of runnable goroutines. Accessed without lock.
       //本地goroutine运行队列
       runqhead uint32  // 队列头
       runqtail uint32     // 队列尾
       runq     [256]guintptr  //使用数组实现的循环队列
     
       runnext guintptr
   
       // Available G's (status == Gdead)
       gFree struct {
           gList
           n int32
       }
   
       ......
   }
   ```

4. schedt结构体

   保存调度器自身的状态信息和保存goroutine的运行队列（全局运行队列）。每个go程序中schedt结构体只有一个实例对象，在源码中被定义成一个共享全局变量，访问队列中的数据需要互斥锁的操作。

   ```go
   type schedt struct {
       // accessed atomically. keep at top to ensure alignment on 32-bit systems.
       goidgen  uint64
       lastpoll uint64
   
       lock mutex
   
       // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
       // sure to call checkdead().
   
       // 由空闲的工作线程组成链表
       midle        muintptr // idle m's waiting for work
       // 空闲的工作线程的数量
       nmidle       int32    // number of idle m's waiting for work
       nmidlelocked int32    // number of locked m's waiting for work
       mnext        int64    // number of m's that have been created and next M ID
       // 最多只能创建maxmcount个工作线程
       maxmcount    int32    // maximum number of m's allowed (or die)
       nmsys        int32    // number of system m's not counted for deadlock
       nmfreed      int64    // cumulative number of freed m's
   
       ngsys uint32 // number of system goroutines; updated atomically
   
       // 由空闲的p结构体对象组成的链表
       pidle      puintptr // idle p's
       // 空闲的p结构体对象的数量
       npidle     uint32
       nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.
   
       // Global runnable queue.
       // goroutine全局运行队列
       runq     gQueue
       runqsize int32
   
       ......
   
       // Global cache of dead G's.
       // gFree是所有已经退出的goroutine对应的g结构体对象组成的链表
       // 用于缓存g结构体对象，避免每次创建goroutine时都重新分配内存
       gFree struct {
           lock          mutex
           stack        gList // Gs with stacks
           noStack   gList // Gs without stacks
           n              int32
       }
    
       ......
   }
   ```

5. 与调度相关的重要全局变量

   ```go
   allgs     []*g     // 保存所有的g
   allm       *m    // 所有的m构成的一个链表，包括下面的m0
   allp       []*p    // 保存所有的p，len(allp) == gomaxprocs
   
   ncpu             int32   // 系统中cpu核的数量，程序启动时由runtime代码初始化
   gomaxprocs int32   // p的最大值，默认等于ncpu，但可以通过GOMAXPROCS修改
   
   sched      schedt     // 调度器结构体对象，记录了调度器的工作状态
   
   m0  m       // 代表进程的主线程
   g0   g        // m0的g0，也就是m0.g0 = &g0
   ```

## GO程序的运行流程

### 调度器的初始化

![](https://img-blog.csdnimg.cn/20210412104644668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**主线程和m0的关联**

通过线程本地存储 m0和g0的绑定，把g0的地址赋于主线程的线程本地存储

**settle函数**

通过arch_prctl系统调用把m0.tls[1]的地址设置成了fs段的段基址

**schedinit函数**

getg函数是编译器实现，可以从线程本地存储中获取当前正在运行的g

mcommoninit函数对m0(g0.m)进行必要的初始化（把m0放入到全局链表allm中之后返回）

### main goroutine的创建

![](https://img-blog.csdnimg.cn/20210412104729942.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**newproc函数**

参数一：fn函数的参数以字节为单位的大小 参数二：参数fn，新创建出来的goroutine将从fn这个函数开始执行

**通用的newproc1函数**

参数一：入口函数的地址 参数二：入口函数的第一个参数的地址 参数三：入口函数的参数以字节为单位的大小 在初始化加载完成的情况下不需要切换g0栈（本身就在g0栈） 用户使用 go 开启新的goroutine时需要使用systemstack函数切换到g0 该函数运行在g0栈

**初始化g的sched成员**

sched的sp成员表示newg被调度起来运行时应该使用的栈的栈顶

sched的pc成员表示当newg被调度起来运行时从这个地址开始执行指令

![](https://img-blog.csdnimg.cn/20210412104801511.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

### 调度main goroutine运行

![](https://img-blog.csdnimg.cn/2021041210482281.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**goroutine的调度过程**

保存g0的调度信息，主要是保存CPU栈顶寄存器SP到g0.sched.sp成员之中；

调用schedule函数寻找需要运行的goroutine，程序加载的场景找到的是main goroutine；

调用gogo函数首先从g0栈切换到main goroutine的栈，然后从main goroutine的g结构体对象之中取出sched.pc的值并使用JMP指令跳转到该地址去执行；

 main goroutine执行完毕直接调用exit系统调用退出进程。

**save函数**

函数功能：保存了调度相关的所有信息，包括最为重要的当前正在运行的g的下一条指令的地址和栈顶地址，不管是对g0还是其它goroutine来说这些信息在调度过程中都是必不可少

**exit函数**

对于main goroutine会直接exit，而非main goroutine执行完成后就会返回到goexit继续执行进行一些清理工作

### 非main goroutine的退出

![](https://img-blog.csdnimg.cn/2021041210484858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**mcall函数**

属于runtime逻辑代码，使用mcall是已经从用户g切换到了g0上，切换到g0.sched.sp固定位置不会造成栈溢出的情况 函数功能： 先从当前运行的g(我们这个场景是g2)切换到g0，这一步包括保存当前g的调度信息，把g0设置到tls中，修改CPU的rsp寄存器使其指向g0的栈； 以当前运行的g(我们这个场景是g2)为参数调用fn函数(此处为goexit0)。

### 调度策略

#### 调度发生的条件

![](https://img-blog.csdnimg.cn/20210412110950788.png#pic_center)

1. goroutine因为某个操作条件不满足（channel阻塞，网络连接阻塞，加锁阻塞或select操作阻塞）需要等待而发生调度
2. goroutine主动调用Gosched函数让CPU发生调度
3. Goroutine运行时间太长或长时间处于系统调用而被调度器剥夺运行权而发生调度

#### 被动调度

![](https://img-blog.csdnimg.cn/20210412104913827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**读取channel阻塞而发生被动调度**

读取channel调用runtime.chanrecv1函数 chanrecv1直接调用chanrecv函数实现读取操作，chanrecv首先会判断channel是否有数据可读，如果有数据则直接读取并返回，但如果没有数据，则需要把当前goroutine挂入channel的读取队列之中并调用goparkunlock函数阻塞该goroutine goparkunlock函数直接调用gopark函数，gopark则调用mcall从当前main goroutine切换到g0去执行park_m函数

**park_m函数**

park_m首先把当前goroutine的状态设置为_Gwaiting（因为它正在等待其它goroutine往channel里面写数据），然后调用dropg函数解除g和m之间的关系，最后通过调用schedule函数进入调度循环

**唤醒阻塞在channel上的goroutine**

channel的发送操作是对runtime.chansend1函数的调用 channel发送和读取的流程类似，如果能够立即发送则立即发送并返回，如果不能立即发送则需要阻塞，如果有正挂在channel的读取队列上等待数据，所以这里直接调用send函数发送给待读取的g，send函数则调用goready函数切换到g0栈并调用ready函数来唤醒正在等待读的goroutine

**ready函数**

ready函数首先把需要唤醒的goroutine的状态设置为_Grunnable，然后把其放入运行队列之中等待调度器的调度 如果当前有空闲的p而且没有处于spinning状态的工作线程，那么就需要通过wakep函数把空闲的p唤醒起来工作

**wakeup函数**

首先通过cas操作再次确认是否有其它工作线程正处于spinning状态 如果已经有工作线程进入了spinning状态而在四处寻找需要运行的goroutine，就没有必要再启动一个多余的工作线程出来了

**startm函数**

首先判断是否有空闲的p结构体对象，如果没有则直接返回，如果有则需要创建或唤醒一个工作线程出来与之绑定 在确保有可以绑定的p对象之后，首先尝试从m的空闲队列中查找正处于休眠状态的工作线程，如果找到则通过notewakeup函数唤醒它，否则调用newm函数创建一个新的工作线程

**notewakeup函数**

工作线程会通过notesleep函数睡眠在m.park成员上，所以notewakeup使用m.park成员作为参数把睡眠在该成员之上的工作线程唤醒 首先使用atomic.Xchg设置note.key值为1，然后notewakeup函数继续调用futexwakeup函数 futexwakeup调用包装了futex系统调用的futex函数来实现唤醒睡眠在内核中的工作线程。

**nwem函数**

首先调用allocm函数从堆上分配一个m结构体对象，然后调用newm1函数，newm1继续调用newosproc函数，newosproc的主要任务是调用clone函数创建一个系统线程，新建的这个系统线程将从mstart函数开始运行

**clone函数**

clone函数首先用了4条指令为clone系统调用准备参数，该系统调用一共需要四个参数，根据Linux系统调用约定，这四个参数需要分别放入rdi， rsi，rdx和r10寄存器中。

最重要的是第一个参数和第二个参数，分别用来指定内核创建线程时需要的选项和新线程应该使用的栈。

因为即将被创建的线程与当前线程共享同一个进程地址空间，所以这里必须为子线程指定其使用的栈，否则父子线程会共享同一个栈从而造成混乱，从上面的newosproc函数可以看出，新线程使用的栈为m.g0.stack.lo～m.g0.stack.hi这段内存，而这段内存是newm函数在创建m结构体对象时从进程的堆上分配而来的。 

准备好系统调用的参数之后，还有另外一件很重的事情需要做，那就是把clone函数的其它几个参数（mp, gp和线程入口函数）保存到寄存器中，原因在于这几个参数目前还位于父线程的栈之中，而一旦通过系统调用把子线程创建出来之后，子线程将会使用我们在clone系统调用时给它指定的栈，所以这里需要把这几个参数先保存到寄存器，等子线程从系统调用返回后直接在寄存器中获取这几个参数。

这里要注意的是虽然这个几个参数值保存在了父线程的寄存器之中，但创建子线程时，操作系统内核会把父线程的所有寄存器帮我们复制一份给子线程，所以当子线程开始运行时就能拿到父线程保存在寄存器中的值，从而拿到这几个参数。

准备工作完成之后代码调用syscall指令进入内核，由内核帮助我们创建系统线程。 

新工作线程的初始化完成之后，便开始执行mstart函数，mstart函数首先会去设置m.g0的stackguard成员，然后调用mstart1()函数把当前工作线程的g0的调度信息保存在m.g0.sched成员之中，最后通过调用schedule函数进入调度循环。 

clone函数会返回两次，在子线程返回值为0继续执行子线程的代码，父线程返回值为子线程的线程ID保存到栈最后通过RET指令最为newosproc的返回值

首先通过系统调用获取到子线程的线程id，并赋值给m.procid，然后调用settls设置线程本地存储并通过把m.g0的地址放入线程本地存储之中，从而实现了m结构体对象与工作线程之间的关联

#### 主动调度

![](https://img-blog.csdnimg.cn/20210412104935679.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**macall函数**

保存调用Gosched的goroutine的现场信息 把保存在g0的sched.sp和sched.bp字段的值恢复到CPU的rsp和rbp寄存器，由此完成g2的栈到g0栈的切换 在g0栈执行goched_m函数。（gosched_m函数是runtime.Gosched函数调用mcall时传递给mcall的参数）。

**goshedlmpl函数**

把主动调度的goroutine（即调用runtime.sched的g）的状态从 _Grunning 设置为 _Grunnable，通过dropg函数解除当前工作线程m和g的关系，然后调用globrunqput函数把主动调度的g放入全局队列中

#### 抢占调度

![image-20210412104138973](https://img-blog.csdnimg.cn/2021041210515521.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**P的运行队列里有等待运行的gortoutine**

这用来保证当前p的本地运行队列中的goroutine得到及时的调度，因为该p对应的工作线程正处于系统调用之中，无法调度队列中goroutine，所以需要寻找另外一个工作线程来接管这个p从而达到调度这些goroutine的目的

**没有空闲的p**

表示其它所有的p都已经与工作线程绑定且正忙于执行go代码，这说明系统比较繁忙，所以需要抢占当前正处于系统调用之中而实际上系统调用并不需要的这个p并把它分配给其它工作线程去调度其它goroutine。

**P对应的m处于系统调用中超过10毫秒**

这表示只要系统调用超时，就对其抢占，而不管是否真的有goroutine需要调度，这样保证sysmon线程不至于觉得无事可做（sysmon线程会判断retake函数的返回值，如果为0，表示retake并未做任何抢占，所以会觉得没啥事情做）而休眠太长时间最终会降低sysmon监控的实时性

**handoffp函数**

通过各种条件判断是否需要启动工作线程来接管_p_，如果不需要则把_p_放入P的全局空闲队列

**Syscall6函数**

主要内容： 调用runtime.entersyscall函数； 使用SYSCALL指令进入系统调用； 调用runtime.exitsyscall函数。

**reentersyscall函数**

主动解除m和p的绑定关系之后，sysmon线程就不需要通过加锁或cas操作来修改m.p成员从而解除m和p之间的关系； 记录进入系统调用之前的g可以让工作线程从系统调用返回之后快速找到一个可能可用的p，而不需要加锁从sched的pidle全局队列中去寻找空闲的p

**exitsyscallfast_pidle函数**

从p的全局空闲队列中获取一个p出来绑定，获取过程需要加锁控制

注意这里使用了systemstack(func())函数来调用exitsyscallfast_pidle，systemstack(func())函数有一个func()类型的参数，该函数首先会把栈切换到g0栈，然后调用通过参数传递进来的函数(闭包函数，包含了对exitsyscallfast_pidle函数的调用)，最后再切换回原来的栈并返回 

原则上来说，只要调用链上某个函数有nosplit这个编译器指示就需要在g0栈上去执行，因为有nosplit指示，编译器就不会插入检查溢出的代码，这样在非g0栈上执行这些nosplit函数就有可能导致栈溢出，g0栈其实就是操作系统线程所使用的栈，它的空间比较大，不需要对runtime代码中的每个函数都做栈溢出检查，否则会严重影响效率。

**preemptone函数**

preemptone函数只是简单的设置了被抢占goroutine对应的g结构体中的 preempt成员为true和stackguard0成员为stackPreempt（stackPreempt是一个常量，一个非常大的数）

**调用链：morestack_noctxt()->morestack()->newstack()**

main函数的第三条指令jbe指令跳转，jbe条件跳转指令，根据g结构体中的stackguard0的值与栈顶寄存器rsp的值比较是否比stackguard0的值小以此来判断是否需要扩展或被抢占调度，

如果发现stackguard0被设置为抢占标记则执行call指令调用morestack_noctxt() 

morestack_noctxt()使用jmp指令直接跳转至morestack继续执行，morestack会被编译器自动插入到函数序言(prologue)

#### schedu函数

![](https://img-blog.csdnimg.cn/20210412105138234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

##### 从全局队列获取

为了保证调度的公平性，每个工作线程每经过61次调度就需要优先尝试从全局运行队列中找出一个goroutine来运行 全局运行队列是所有工作线程都可以访问的，所以在访问它之前需要加锁

**globrunqget函数**

从全局获取g的过程中有实现负载均衡

```go
//根据p的数量平分全局运行队列中的goroutines   
n := sched.runqsize / gomaxprocs + 1   
if n > sched.runqsize { //上面计算n的方法可能导致n大于全局运行队列中的goroutine数量     
n = sched.runqsize   
}   
if max > 0 && n > max {     
n = max  //最多取max个goroutine   
}   
if n > int32(len(_p_.runq)) / 2 {     
n = int32(len(_p_.runq)) / 2  
//最多只能取本地队列容量的一半   }
```

##### 从本地运行队列获取

首先查看runnext成员是否为空，如果不为空则返回runnext所指的goroutine，并把runnext成员清零，如果runnext为空，则继续从循环队列中查找goroutine

**CAS操作**

必要性在获取可运行g的过程中可能有其他工作线程正在窃取

对runqhead的操作使用了atomic.LoadAcq和atomic.CasRel，它们分别提供了load-acquire和cas-release语义。 

对于atomic.LoadAcq来说，其语义主要包含如下几条： 

1. 原子读取，也就是说不管代码运行在哪种平台，保证在读取过程中不会有其它线程对该变量进行写入； 
2. 位于atomic.LoadAcq之后的代码，对内存的读取和写入必须在atomic.LoadAcq读取完成后才能执行，编译器和CPU都不能打乱这个顺序；
3. 当前线程执行atomic.LoadAcq时可以读取到其它线程最近一次通过atomic.CasRel对同一个变量写入的值，与此同时，位于atomic.LoadAcq之后的代码，不管读取哪个内存地址中的值，都可以读取到其它线程中位于atomic.CasRel（对同一个变量操作）之前的代码最近一次对内存的写入。 

对于atomic.CasRel来说，其语义主要包含如下几条：

1. 原子的执行比较并交换的操作； 
2. 位于atomic.CasRel之前的代码，对内存的读取和写入必须在atomic.CasRel对内存的写入之前完成，编译器和CPU都不能打乱这个顺序； 
3. 线程执行atomic.CasRel完成后其它线程通过atomic.LoadAcq读取同一个变量可以读到最新的值，与此同时，位于atomic.CasRel之前的代码对内存写入的值，可以被其它线程中位于atomic.LoadAcq（对同一个变量操作）之后的代码读取到。

##### 从其他工作线程中窃取

**工作线程M的自旋状态**

工作线程在从其它工作线程的本地运行队列中盗取goroutine时的状态称为自旋状态 把spinning标志设置成了true，同时增加处于自旋状态的M的数量，而盗取结束之后则把spinning标志还原为false，同时减少处于自旋状态的M的数量

**窃取算法**

遍历allp中的所有p，查看其运行队列是否有goroutine，如果有，则取其一半到当前工作线程的运行队列，然后从findrunnable返回，如果没有则继续遍历下一个p。 

为了保证公平性，遍历allp时并不是固定的从allp[0]即第一个p开始，而是从随机位置上的p开始，而且遍历的顺序也随机化了，并不是现在访问了第i个p下一次就访问第i+1个p，而是使用了一种伪随机的方式遍历allp中的每个p，防止每次遍历时使用同样的顺序访问allp中的元素

**stopm函数**

stopm的核心是调用mput把m结构体对象放入sched的midle全局空闲队列，然后通过notesleep(&m.park)函数让自己进入睡眠状态

当其他工作线程发现有更多的goroutine需要运行时可以通过全局的m空闲队列找到处于睡眠状态的m，然后调用notewakeup(&m.park)将其唤醒，M的全局空闲队列相当于m的对象池

**note机制**

Runtime实现的一次性睡眠和唤醒机制 note的底层实现机制跟操作系统相关，不同系统使用不同的机制，比如linux下使用的futex系统调用，而mac下则是使用的pthread_cond_t条件变量，note对这些底层机制做了一个抽象和封装

**线程可以通过调用notesleep进入睡眠**

notesleep函数调用futexsleep进入睡眠，需要用一个循环，是因为futexsleep有可能意外从睡眠中返回，所以从futexsleep函数返回后还需要检查note.key是否还是0，如果是0则表示并不是其它工作线程唤醒，只是futexsleep意外返回，需要再次调用futexsleep进入睡眠。 futexsleep调用futex函数进入睡眠，主要功能就是执行futex系统调用进入操作系统内核进行睡眠。 

```c++
int64 futex(int32 *uaddr, int32 op, int32 val, struct timespec *timeout, int32 *uaddr2, int32 val2) 
```

futex系统调用为我们提供的功能为如果 `*uaddr` == `val `则进入睡眠，否则直接返回。需要在内核判断`*uaddr`与`val`是否相等，而不能在用户态先判断它们是否相等，原因在于判断`*uaddr`与val是否相等和进入睡眠这两个操作必须是一个原子操作，否则会存在一个竞态条件：如果不是原子操作，则当前线程在第一步判断完`*uaddr`与`val`相等之后进入睡眠之前的这一小段时间内，有另外一个线程通过唤醒操作把`*uaddr`的值修改了，这就会导致当前工作线程永远处于睡眠状态而无人唤醒它。而在用户态无法实现判断与进入睡眠这两步为一个原子操作，所以需要内核来为其实现原子操作

## 循环调度

![](https://img-blog.csdnimg.cn/2021041210522628.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA1NjkwMA==,size_16,color_FFFFFF,t_70#pic_center)

**shcedule函数**

函数功能：通过调用globrunqget()和runqget()函数分别从全局运行队列和当前工作线程的本地运行队列中选取下一个需要运行的goroutine，如果这两个队列都没有需要运行的goroutine则通过findrunnalbe()函数从其它p的运行队列中盗取goroutine，一旦找到下一个需要运行的goroutine，则调用excute函数从g0切换到该goroutine去运行

**excute函数**

函数功能：第一个参数gp即是需要调度起来运行的goroutine

首先把gp的状态从 _Grunnable修改为 _Grunning，然后把gp和m关联起来，这样通过m就可以找到当前工作线程正在执行哪个goroutine

**gogo函数**

汇编代码函数功能： 把gp.sched的成员恢复到CPU的寄存器完成状态以及栈的切换； 跳转到gp.sched.pc所指的指令地址（runtime.main）处执行。CPU执行权的转让和切换

**macll函数**