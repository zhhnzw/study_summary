## 锁

锁能保证多个 Goroutine 在访问同一片内存时不会出现竞争条件（Race condition）等问题

### 互斥锁 (sync.Mutext)

互斥锁的加锁过程比较复杂，它涉及自旋、信号量以及调度等概念, [参考](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/)：

- 如果互斥锁处于初始化状态，就会直接通过置位 `mutexLocked` 加锁；
- 如果互斥锁处于 `mutexLocked` 并且在普通模式下工作，就会进入自旋，执行 30 次 `PAUSE` 指令消耗 CPU 时间等待锁的释放；
- 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
- 互斥锁在正常情况下会通过 [`sync.runtime_SemacquireMutex`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L70-L72) 函数将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒当前 Goroutine；
- 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，当前 Goroutine 会将互斥锁切换回正常模式；

互斥锁的解锁过程与之相比就比较简单，其代码行数不多、逻辑清晰，也比较容易理解：

- 当互斥锁已经被解锁时，那么调用 [`sync.Mutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L179-L192) 会直接抛出异常；
- 当互斥锁处于饥饿模式时，会直接将锁的所有权交给队列中的下一个等待者，等待者会负责设置 `mutexLocked` 标志位；
- 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，就会直接返回；在其他情况下会通过 [`sync.runtime_Semrelease`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L65-L67) 唤醒对应的 Goroutine；

### 读写互斥锁 (sync.RWMutext)

读写互斥锁在互斥锁之上提供了额外的更细粒度的控制，能够在读操作远远多于写操作时提升性能。

- 调用`sync.RWMutex.Lock`尝试获取写锁时；
  - 每次 [`sync.RWMutex.RUnlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L62-L75) 都会将 `readerWait` 其减一，当它归零时该 Goroutine 就会获得写锁；
  - 将 `readerCount` 减少 `rwmutexMaxReaders` 个数以阻塞后续的读操作；
- 调用 [`sync.RWMutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L118-L140) 释放写锁时，会先通知所有的读操作，然后才会释放持有的互斥锁；

### sync.WaitGroup

[`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 可以等待一组 Goroutine 的返回，一个比较常见的使用场景是批量发出 RPC 或者 HTTP 请求：

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

- [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 必须在 [`sync.WaitGroup.Wait`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L103-L141) 方法返回之后才能被重新使用；
- [`sync.WaitGroup.Done`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L98-L100) 只是对 [`sync.WaitGroup.Add`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L53-L95) 方法的简单封装，我们可以向 [`sync.WaitGroup.Add`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L53-L95) 方法传入任意负数（需要保证计数器非负）快速将计数器归零以唤醒其他等待的 Goroutine；
- 可以同时有多个 Goroutine 等待当前 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 计数器的归零，这些 Goroutine 会被同时唤醒；

### Once

Go 语言标准库中 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 可以保证在 Go 程序运行期间的某段代码只会执行一次。在运行如下所示的代码时，我们会看到如下所示的运行结果：

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

- [`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 方法中传入的函数只会被执行一次，哪怕函数中发生了 `panic`；
- 两次调用 [`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 方法传入不同的函数也只会执行第一次调用的函数；

### Cond

[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 不是一个常用的同步机制，在遇到长时间条件无法满足时，与使用 `for {}` 进行忙碌等待相比，[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 能够让出处理器的使用权。在使用的过程中我们需要注意以下问题：

- [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 方法在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；
- [`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 方法唤醒的 Goroutine 都是队列最前面、等待最久的 Goroutine；
- [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 会按照一定顺序广播通知等待的全部 Goroutine；

### SingleFlight

Groupcache

