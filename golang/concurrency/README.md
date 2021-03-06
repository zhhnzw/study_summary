## 并发模型

### 并发和并行的区别
并发是**同时处理**多件事情，同一时间可以只做一件，而把其他任务做到一半先暂停，后续被暂停的任务再恢执行，这样来回切换达到并发处理多件事情的目的。并行是**同时做**多件事情，需要多核CPU支持。

### 什么是进程？

进程没被执行时，只是一个二进制文件。一旦被执行起来，它就从磁盘上的二进制文件，变成了计算机内存中的数据、寄存器里的值、堆栈中的指令、被打开的文件，以及各种设备的状态信息的一个集合。像这样一个程序运起来后的计算机执行环境的总和，就是进程。

### 进程、线程、协程的区别

**进程**：是系统资源分配的最小单位

**线程**：是CPU调度最小单位

**协程**：用户态轻量级线程，协程调度不需要内核参与而是由用户态程序来决定，协程对于操作系统而言是无感知的

* 进程之间是相互隔离的，各自拥有自己的内存空间，因此CPU对进程切换执行成本高
* 进程是线程的载体容器，进程内的多线程共享其内存空间，因此线程之间的通信比进程通信容易得多
* CPU在多线程之间快速切换调度执行，以此达到多线程并发执行的效果
* 虽然线程比较轻量，但是也有不小的调度开销，当线程开的太多，也会导致高CPU调度消耗
* 多个coroutine可以绑定在同一个线程之上，由协程调度器管理，因此coroutine的调度减少了线程频繁切换的调度开销(go程序默认只使用与CPU数量相等的线程数量)

Q: 用户态的协程如何适时的主动出让CPU控制权？

A: goroutine控制器可能切换的点：

1. io，select关键字
2. channel阻塞
3. 等待锁
4. 函数调用(有时)
5. runtime.Gosched()

Q: 协作式调度和抢占式调度的优缺点？

A: 协作式调度可以降低线程切换开销，提高CPU的使用率。但协作式调度不够智能的时候，会白白浪费CPU的执行能力，而抢占式调度可以规避这个问题。

Q: Goroutine是如何调度的？

A: 老版的Go语言只能依靠goroutine主动出让CPU控制权才能触发调度，而在后续的版本迭代中引入了抢占式调度，GPM模型

