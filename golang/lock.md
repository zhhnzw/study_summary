## 锁

锁能保证多个 Goroutine 在访问同一片内存时不会出现竞争条件（Race condition）等问题

### 互斥锁 (sync.Mutex)

确保锁定的操作在同一时刻只有一个goroutine能执行

* 不要重复锁定互斥锁（重复锁定，第2次锁定会一直阻塞，自己锁死了自己）
* 不要忘记解锁互斥锁，必要时可使用`defer`语句解锁（锁定和解锁应当成对出现，对于每一个锁定操作，都要有且仅有一个对应的解锁操作）
* 不要对尚未锁定或者已解锁的互斥锁解锁（会立即引发panic，而且无法被recover，程序会立刻崩溃）
* 不要在多个函数之间直接传递互斥锁（sync.Mutex是结构体类型，属于值类型，把它传递给一个函数会导致它的副本产生，而且原值和它的副本是完全独立的，它们都是不同的互斥锁。如果把互斥锁作为参数传给一个函数，那么在这个函数中对传入的锁的所有操作，都不会对存在于该函数之外的那个原锁产生任何影响）

### 读写互斥锁 (sync.RWMutex)

读写互斥锁在互斥锁之上提供了额外的更细粒度的控制，能够在读操作远远多于写操作时提升性能。

- 写锁: `sync.RWMutex.Lock`对应`sync.RWMutex.Unlock`
- 读锁: `sync.RWMutex.RLock`对应`sync.RWMutex.RUnlock`
- 在读锁已被锁定的情况下再试图锁定读锁，不会阻塞当前goroutine（主要就是利用读写锁消除了读操作互斥这一特性了）
- 重复锁定写锁、写锁被锁定再施加读锁、读锁被锁定再施加写锁，都会阻塞当前goroutine

### sync.WaitGroup

`sync.WaitGroup` 可以等待一组 Goroutine 的返回，一个比较常见的使用场景是批量发出 RPC 或者 HTTP 请求：

```go
requests := []*Request{...}
wg := &sync.WaitGroup{}
wg.Add(len(requests))

for _, request := range requests {
    go func(r *Request) {
        defer wg.Done()
        // res, err := service.call(r)
    }(request)
}
wg.Wait()
```

- `sync.WaitGroup`  必须在 `sync.WaitGroup.Wait`方法返回之后才能被重新使用；
- `sync.WaitGroup.Done`只是对 `sync.WaitGroup.Add`方法的简单封装，我们可以向 `sync.WaitGroup.Add`方法传入任意负数（需要保证计数器非负）快速将计数器归零以唤醒其他等待的 Goroutine；
- 可以同时有多个 Goroutine 等待当前 `sync.WaitGroup`计数器的归零，这些 Goroutine 会被同时唤醒；

### Once

Go 语言标准库中 `sync.Once` 可以保证在 Go 程序运行期间的某段代码只会执行一次。在运行如下所示的代码时，我们会看到如下所示的运行结果：

```go
func main() {
    o := &sync.Once{}
    for i := 0; i < 10; i++ {
        o.Do(func() {
            fmt.Println("only once")
        })
    }
}
$ go run main.go
only once
```

- `sync.Once.Do`方法中传入的函数只会被执行一次，哪怕函数中发生了 `panic`；
- 两次调用 `sync.Once.Do` 方法传入不同的函数也只会执行第一次调用的函数；

### Cond

条件变量`sync.Cond`是基于互斥锁的工具，并不是被用来保护临界区和共享资源的，是用于协调想要访问共享资源的那些线程的。当共享资源的状态发生变化时，它可以被用来通知被互斥锁阻塞的线程。

在遇到长时间条件无法满足时，与使用 `for {}` 进行忙碌等待相比，`sync.Cond`能够让出处理器的使用权。在使用的过程中我们需要注意以下问题：

- [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 方法在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；
- [`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 方法唤醒的 Goroutine 都是队列最前面、等待最久的 Goroutine；
- [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 会按照一定顺序广播通知等待的全部 Goroutine；

锁的具体实现比较复杂，[参考](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/)

