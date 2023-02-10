
# 操作系统

## 1. 操作系统的基本特征

并发，共享，虚拟，异步。

* **并发**是指在一段时间内同时运行多个程序。
* **共享**是分为同时共享和互斥共享，互斥共享也被称为临界资源，也就是同一时刻内只允许一个进程访问。
* **虚拟**分为时分复用技术和空分复用技术。

时分复用是指每个进程可以占用 CPU 一段时间，交替执行。实现了一个处理器上并发执行。

对于空分复用，例如虚拟内存，每个进程都有独立的地址空间，地址空间被映射到物理地址上。如果内存不够用就执行页面置换算法(LRU)。

* **异步**是指交错执行，走走停停向前推进，双方没有同步的时钟。

## 2. 进程和线程的区别

进程是资源分配的最小单位，线程是独立调度的最小单位。

线程是独立调度的基本单位，线程的开销远小于进程的开销，线程内通信共享，线程外通信需要借助 OS 。

虚拟内存和物理内存的差别，为什么要用虚拟内存？
-   虚拟内存有哪些部分？
    -   八股文
    -   内核区、用户区
    -   用户区有代码段、数据段、堆栈段（中间还有个文件映射区）
-   虚拟内存的查询流程
    -   说了下二级页表查询


## 3. 死锁

死锁的四个条件

1. 互斥：每个资源要么已经分配给了一个进程，要么就是可用的。
2. 占有和等待：已经得到了某个资源的进程可以再请求新的资源。
3. 不可抢占：已经分配给一个进程的资源不能强制性地被抢占，它只能被占有它的进程显式地释放。
4. 环路等待：有两个或者两个以上的进程组成一条环路，该环路中的每个进程都在等待下一个进程所占有的资源。

> 首先是资源是互斥的，也就是资源只能被一个进程占用，其次占用了后想要别的进程的资源于是等待，资源不能被抢占，最后是环路等待，资源形成了一个环。

死锁处理主要有以下四种方法：

1. 鸵鸟策略：也就是忽略。
2. 死锁检测与死锁恢复：通过检测有向图是否存在环来实现，深搜，访问过后就标记一下，一旦访问到标记过的那么一定有环。可以通过抢占，回滚，杀死进程来实现恢复。
3. 死锁预防：
   1. 破坏互斥：例如假脱机技术。
   2. 破坏占有和等待条件，在进程开始之前一口气把资源都要完。
4. 死锁避免：银行家算法。


## 4. 静态链接、动态链接的区别
- 静态链接：在编译时将所有需要的库和程序代码一起打包，生成可执行文件。这意味着，如果需要更新或修改库，必须重新编译整个程序。
- 动态链接：程序在运行时加载所需的库，并与程序代码进行链接。这意味着，如果需要更新或修改库，可以直接替换动态链接库，而不必重新编译整个程序。
- 总的来说，动态链接更灵活，但可执行文件体积较大，而静态链接则相反。

## 5. 操作系统如何处理中断？
-   一个进程切换到另一个进程，如何找到另一个进程要执行的地址？
	- （通过pc寄存器，保存下一条指令的地址）
-   线程间共享什么资源？
-   全局数据区的全局变量、堆区变量
-   进程间通信方式？
	-  管道
	- 消息队列
	- 共享内存
	- 信号量
	- socket通信
	- 信号
	- 匿名管道
-   进程中的一个线程崩了，会引发进程崩吗？
	- 是的，如果一个线程在进程中崩溃，它所在的进程也可能会崩溃。
	- 当线程崩溃时，它可能导致整个进程的内存环境破坏，从而影响到其他线程的正常执行。如果整个进程的内存环境受到严重影响，操作系统可能无法修复它，因此进程可能会崩溃。
	- 但是，如果进程实现了适当的错误处理机制，可以在线程崩溃时保护其他线程，从而防止整个进程崩溃。
-   互斥锁，信号量使用的场景区别
    -   一个线程互斥，另一个线程同步
-   互斥锁，读写锁，自旋锁的区别
    -   八股文，只说了自旋锁，就没让继续说了
-   怎么实现自旋锁
    -   说了原子操作，test and set指令（tsl）