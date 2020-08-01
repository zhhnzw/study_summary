## 管道

> 使用通信来共享内存，而不是通过共享内存来通信

常见用法
*  `for select`
* `for range channel`会一直迭代到`channel`被 close

![channel用法总结](../src/channel.png)

发送逻辑要点:

- 如果channel已经关闭，会抛出panic: `send on closed channel`
- 当存在等待的接收者时，通过 [`runtime.send`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L270) 直接将数据发送给阻塞的接收者；
- 当缓冲区存在空余空间时，将发送的数据写入 Channel 的缓冲区；
- 当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据；

Q: 如何避免向closed channel发送消息？

A: 说明所编写的程序有设计bug。channel关闭原则:

* 不要在消费端关闭channel

* 不要在有多个并行的生产者时对channel执行关闭操作

* 应该只在[唯一的或者最后唯一剩下]的生产者协程中关闭channel，来通知消费者已经没有值可以继续读

* 只要坚持这个原则，就可以确保向一个已经关闭的channel发送数据的情况不可能发生

Q: 退出程序时, 如何防止channel没有消费完？
1. 多个消费者，一个生产者。直接让生产者关闭channel即可
2. 多个生产者，一个消费者。结合具体情况，由生产者同时监听一个停止生产数据的channel作为return的信号
3. 多个生产者，多个消费者。设计一个停止生产的信号和一个生产者已经停止的信号，生产者监听停止生产的信号，在return之前发送已经停止的信号，额外的专门用于关闭数据channel的协程监听和搜集所有生产者已经停止生产的信号，搜集完整则调用close

Q: 生产者把消息发送完立刻调用close, 不等消费者消费完，会丢数据吗？

A: 不会丢数据。原理[参考](http://xiaorui.cc/archives/5007)

channel数据结构:

```go
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint  
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
```

* 从某种程度上说，channel 是一个用于同步和通信的有锁队列(互斥锁解决线程竞争问题)
* channel的缓冲区其实是一个环形队列
* `qcount`表示队列中元素的数量
* `dataqsiz`表示环形队列的总大小
* `buf`表示一个指向循环数组的指针
* `sendx`和`recvx`分别用来标识当前发送和接收的元素在循环队列中的位置
* `recvq`和`sendq`都是一个列表，分别用于存储当前处于等待接收和等待发送的`Goroutine`

