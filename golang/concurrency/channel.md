## 管道

如果说`goroutine`是Go程序并发的执行体，`channel`就是它们之间的连接。`channel`是可以让一个`goroutine`发送特定值到另一个`goroutine`的通信机制。

Go 语言中的通道（channel）是一种特殊的类型。通道像一个传送带或者队列，总是遵循先入先出（First In First Out）的规则，保证收发数据的顺序。

使用全局变量，不同的线程使用锁来处理全局变量的资源竞争，这种方式就是通过共享内存来通信，而Go语言主张：

> 使用通信来共享内存，而不是通过共享内存来通信

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

