面对之前调度器的问题，Go设计了新的调度器。

在新调度器中，除了M(thread)和G(goroutine)，又引进了P(Processor)。

![[Pasted image 20221226170751.png]]

Processor，它包含了运行goroutine的资源，如果线程想运行goroutine，必须先获取P，P中还包含了可运行的G队列。


### (1) GMP模型

在Go中，**线程是运行goroutine的实体，调度器的功能是把可运行的goroutine分配到工作线程上**。

![[Pasted image 20221226170940.png]]


1.  **全局队列**（Global Queue）：存放等待运行的G。
2.  **P的本地队列**：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列。
3.  **P列表**：所有的P都在程序启动时创建，并保存在数组中，最多有`GOMAXPROCS`(可配置)个。
4.  **M**：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列**拿**一批G放到P的本地队列，或从其他P的本地队列**偷**一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。


Goroutine调度器和OS调度器是通过M结合在一起的，每个M都代表了1个内核线程，os调度器负责把内核线程分配到CPU的核上执行。

**关于P和M的个数问题**

1、P的数量

由启动的环境变量 $GOMAXPROCS 或者是由runtime的方法GOMAXPROCS()
(设置可同时使用的cpu个数)决定。这意味着在程序执行的任意时刻只有 $GOMAXPROCS个Goroutine正在同时执行。

2、M的数量

- go语言本身的限制：go程序启动时候，会设置M的最大数量，默认10000，但是内核很难支持这么多的线程数，所以这个限制可以忽略。

- runtime/debug 中的setMaxThreads函数，设置M的最大数量

- 一个M阻塞了，就会创建新的M

M和P的数量没有绝对关系，一个M阻塞，P就会创建或者切换另外一个M，所以就算P的个数默认是1，也有可能很多M创建出来。

	|P和M何时被创建

1，P何时创建：在确定了P的最大数量N之后，运行时系统会根据这个数量创建n个P
2，M何时创建：*没有足够的M来关联P*并运行其中的可运行的G，比如所有的M此时都阻塞了，而P中还有很多就绪任务，就会去寻找空闲的M，而没有空闲的，就会去创建新的M。

### (2) 调度器的设计策略

复用线程：避免频繁的创建，销毁线程，而是对线程的复用。

1）work steading机制---线程复用机制

当本线程无可运行的G时候，尝试从其他线程绑定的P偷取G，而不是销毁线程

2）hand off机制

当本线程因为G进行系统调用阻塞时候，线程释放绑定的P，把P转移给其他空闲的线程执行

**利用并行**：GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在CPU上同时运行。GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS=核数/2 ，则最多利用了一半的CPU核数进行并发。

**抢占**：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在GO中，一个Goroutine最多占用CPU10ms，防止其他Goroutine被饿死，这个也是goroutine不同于coroutine的一个地方

### runtime.schedule

调度开始了，m要找到Goroutine放置在CPU上了。

1、检查定时器
2、尝试调度 trace reader
3、尝试调度 GC worker
4、偶尔尝试下取全局队列（防止两个 G 一直不断唤醒对方占据 P）
	1.  每调度61次(具体为啥是61有待思考)，就从全局的goroutine列表中选goroutine
5、取本地队列
	1.  如果上一步没找到，就从m对应的p的本地队列中得到
6、取全局队列
	1.如果上一步还没有找到，就调findrunnable从其他线程窃取goroutine，如果发现有就窃取一半放到自己的p缓存中，如果都没有就说明真的没有待运行的goroutine了，就陷入睡眠一直阻塞在findrunnable函数，等待被唤醒
7、检查网络就绪 G
8、从其他 P 那里偷
9、没事干的话，检查下是不是要做 GC 扫描
10、继续循环

### （3）go func() 调度流程

1）开始 go func() ，创建Goroutine
2）尝试加入到局部队列中，如果局部队列已满，那么将加入到全局队列中
3）M尝试获取G，如果M1对应的P本地队列为空将会从全局队列中获取，如果全局队列为空的时候将从其他的MP组合偷取G
4）m将找到的G调度到cpu上进行执行func()
5）如果m发生阻塞，将从休眠队列中获取一个M或者创建一个M，接管当前正在阻塞G的P
6）完成调度之后，销毁G

![[Pasted image 20221227144031.png]]

通过上诉图的分析：
1）我们可以通过go func()来创建一个Goroutine

2）有两个存储G的队列，一个是局部调度器P的本地队列，一个是全局G队列，新创建的G会先保存在本地队列中，在本地队列满的情况下，将存放到全局队列中；

3）G只能运行在M中，一个M必须要持有一个P，M会从P的本地队列中弹出一个可执行状态的G来执行，如果P的本地队列为空，就会想其他的MP组合偷取一个可执行的G来执行；

4）一个M调度G执行的过程是一个循环机制

5）当M执行某一个G时候如果发生了syscall或则其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个P；

6）当M系统调用结束时候，这个M会尝试获取一个空闲的P执行，并放入到这个P的本地队列中，如果获取不到P，那么M将进入休眠状态，加入到空闲线程中，然后这个G会被放入全局队列中。


### （4）调度器的生命周期

1）开始
2）创建第一个线程M0
3）创建第一个Go协程G0
4）关联M0和G0
5）调度初始化
6）创建main()中的Goroutine
7）启动M0
8）M绑定P
9）M通过P获取到G
10）获取不到进入休眠状态，一直等待被唤醒
11）获取到M将设置G环境，执行G，G退出

**特殊的M0和G0**

*M0*

`M0`是启动程序后的编号为0的主线程，这个M对应的实例会在全局变量runtime.m0中，不需要在heap上分配，M0负责执行初始化操作和启动第一个G， 在之后M0就和其他的M一样了。

*G0*

`G0`是每次启动一个M都会第一个创建的gourtine，G0仅用于负责调度的G，G0不指向任何可执行的函数, 每个M都会有一个自己的G0。在调度或系统调用时会使用G0的栈空间, 全局变量的G0是M0的G0。







