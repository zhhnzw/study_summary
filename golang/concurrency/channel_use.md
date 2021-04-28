## 管道的用法

### channel的使用场景

把channel用在**数据流动的地方**：

1. 消息传递、消息过滤
2. 信号广播
3. 事件订阅与广播
4. 请求、响应转发
5. 任务分发
6. 结果汇总
7. 并发控制
8. 同步与异步
9. ...

### channel的基本操作和注意事项

channel存在`3种状态`：

1. nil，未初始化的状态，只进行了声明，或者手动赋值为`nil`
2. active，正常的channel，可读或者可写
3. closed，已关闭，**千万不要误认为关闭channel后，channel的值是nil**

channel可进行`3种操作`：

1. 读
2. 写
3. 关闭

把这3种操作和3种channel状态可以组合出`9种情况`：

| 操作      | nil的channel | 正常channel | 已关闭channel |
| --------- | ------------ | ----------- | ------------- |
| <- ch     | 阻塞         | 成功或阻塞  | 读到零值      |
| ch <-     | 阻塞         | 成功或阻塞  | panic         |
| close(ch) | panic        | 成功        | panic         |

对于nil通道的情况，也并非完全遵循上表，**有1个特殊场景**：当`nil`的通道在`select`的某个`case`中时，这个case会阻塞，但不会造成死锁。

![channel用法总结](/Users/zhhnzw/workspace/mygithub/study_summary/src/golang/channel.png)

发送逻辑要点:

- 如果channel已经关闭，会抛出panic: `send on closed channel`
- 当存在等待的接收者时，会直接将数据发送给阻塞的接收者；
- 当缓冲区存在空余空间时，将发送的数据写入 Channel 的缓冲区；
- 当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据；

### 1. 使用for range读channel

- 场景：当需要不断从channel读取数据时
- 原理：使用`for-range`读取channel，这样既安全又便利，当channel关闭时，for循环会自动退出，无需主动监测channel是否关闭，可以防止读取已经关闭的channel，造成读到数据为通道所存储的数据类型的零值。
- 用法：

```go
for x := range ch{
    fmt.Println(x)
}
```

### 2. 使用`_,ok`判断channel是否关闭

- 场景：读channel，但不确定channel是否关闭时
- 原理：读已关闭的channel会得到零值，如果不确定channel，需要使用`ok`进行检测。ok的结果和含义：
  - `true`：读到数据，并且通道没有关闭。
  - `false`：通道关闭，无数据读到。
- 用法：

```go
if v, ok := <- ch; ok {
    fmt.Println(v)
}
```

### 3. 使用select处理多个channel

- 场景：需要对多个通道进行同时处理，但只处理最先发生的channel时
- 原理：`select`可以同时监控多个通道的情况，只处理未阻塞的case。**当通道为nil时，对应的case永远为阻塞，无论读写。特殊关注：普通情况下，对nil的通道写操作是要panic的**。
- 用法：

```go
// 分配job时，如果收到关闭的通知则退出，不分配job
func (h *Handler) handle(job *Job) {
    select {
    case h.jobCh<-job:
        return 
    case <-h.stopCh:
        return
    }
}
```

### 4. 使用channel的声明控制读写权限

- 场景：协程对某个通道只读或只写时
- 目的：A. 使代码更易读、更易维护，B. 防止只读协程对通道进行写数据，但通道已关闭，造成panic。
- 用法：
  - 如果协程对某个channel只有写操作，则这个channel声明为只写。
  - 如果协程对某个channel只有读操作，则这个channe声明为只读。

```go
// 只有generator进行对outCh进行写操作，返回声明
// <-chan int，可以防止其他协程乱用此通道，造成隐藏bug
func generator(int n) <-chan int {
    outCh := make(chan int)
    go func(){
        for i:=0;i<n;i++{
            outCh<-i
        }
    }()
    return outCh
}

// consumer只读inCh的数据，声明为<-chan int
// 可以防止它向inCh写数据
func consumer(inCh <-chan int) {
    for x := range inCh {
        fmt.Println(x)
    }
}
```

### 5. 使用缓冲channel增强并发

- 场景：并发
- 原理：有缓冲通道可供多个协程同时处理，在一定程度可提高并发性。
- 用法：

```go
// 无缓冲
ch1 := make(chan int)
ch2 := make(chan int, 0)
// 有缓冲
ch3 := make(chan int, 1)
func test() {
    inCh := generator(100)
    outCh := make(chan int, 10)

    // 使用5个`do`协程同时处理输入数据
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go do(inCh, outCh, &wg)
    }

    go func() {
        wg.Wait()
        close(outCh)
    }()

    for r := range outCh {
        fmt.Println(r)
    }
}

func generator(n int) <-chan int {
    outCh := make(chan int)
    go func() {
        for i := 0; i < n; i++ {
            outCh <- i
        }
        close(outCh)
    }()
    return outCh
}

func do(inCh <-chan int, outCh chan<- int, wg *sync.WaitGroup) {
    for v := range inCh {
        outCh <- v * v
    }

    wg.Done()
}
```

### 6. 为操作加上超时

- 场景：需要超时控制的操作
- 原理：使用`select`和`time.After`，看操作和定时器哪个先返回，处理先完成的，就达到了超时控制的效果
- 用法：

```go
func doWithTimeOut(timeout time.Duration) (int, error) {
    select {
    case ret := <-do():
        return ret, nil
    case <-time.After(timeout):
        return 0, errors.New("timeout")
    }
}

func do() <-chan int {
    outCh := make(chan int)
    go func() {
        // do work
    }()
    return outCh
}
```

### 7. 使用time实现channel无阻塞读写

- 场景：并不希望在channel的读写上浪费时间
- 原理：是为操作加上超时的扩展，这里的操作是channel的读或写
- 用法：

```go
func unBlockRead(ch chan int) (x int, err error) {
    select {
    case x = <-ch:
        return x, nil
    case <-time.After(time.Microsecond):
        return 0, errors.New("read time out")
    }
}

func unBlockWrite(ch chan int, x int) (err error) {
    select {
    case ch <- x:
        return nil
    case <-time.After(time.Microsecond):
        return errors.New("read time out")
    }
}
```

*注：time.After等待可以替换为default，则是channel阻塞时，立即返回的效果*

### 8. 使用`close(ch)`关闭所有下游协程

- 场景：退出时，显示通知所有协程退出
- 原理：所有读`ch`的协程都会收到`close(ch)`的信号
- 用法：

```go
func (h *Handler) Stop() {
    close(h.stopCh)

    // 可以使用WaitGroup等待所有协程退出
}

// 收到停止后，不再处理请求
func (h *Handler) loop() error {
    for {
        select {
        case req := <-h.reqCh:
            go handle(req)
        case <-h.stopCh:
            return
        }
    }
}
```

### 9. 使用`chan struct{}`作为信号channel

- 场景：使用channel传递信号，而不是传递数据时
- 原理：没数据需要传递时，传递空struct
- 用法：

```go
// 上例中的Handler.stopCh就是一个例子，stopCh并不需要传递任何数据
// 只是要给所有协程发送退出的信号
type Handler struct {
    stopCh chan struct{}
    reqCh chan *Request
}
```

### 10. 使用channel传递结构体的指针而非结构体

- 场景：使用channel传递结构体数据时
- 原理：**channel本质上传递的是数据的拷贝，拷贝的数据越小传输效率越高，传递结构体指针，比传递结构体更高效**
- 用法：

```go
reqCh chan *Request

// 好过
reqCh chan Request
```

### 11. 使用channel传递channel

- 场景：使用场景有点多，通常是用来获取结果。
- 原理：channel可以用来传递变量，channel自身也是变量，可以传递自己。
- 用法：下面示例展示了有序展示请求的结果，另一个示例可以见[另外文章的版本3](http://lessisbetter.site/2019/06/09/golang-first-class-function/#版本3)。

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

func main() {
    reqs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}

    // 存放结果的channel的channel
    outs := make(chan chan int, len(reqs))
    var wg sync.WaitGroup
    wg.Add(len(reqs))
    for _, x := range reqs {
        o := handle(&wg, x)
        outs <- o
    }

    go func() {
        wg.Wait()
        close(outs)
    }()

    // 读取结果，结果有序
    for o := range outs {
        fmt.Println(<-o)
    }
}

// handle 处理请求，耗时随机模拟
func handle(wg *sync.WaitGroup, a int) chan int {
    out := make(chan int)
    go func() {
        time.Sleep(time.Duration(rand.Intn(3)) * time.Second)
        out <- a
        wg.Done()
    }()
    return out
}
```

### CSP（communicating sequential processes）并发模型

> 使用通信来共享内存，而不是通过共享内存来通信

通过[goroutine](/golang/goroutine.md)和[channel](/golang/channel.md)来实现

#### 流水线FAN模型

![FAN-OUT和FAN-IN模式](/Users/zhhnzw/workspace/mygithub/study_summary/src/fan.png)

#### 协程池

![协程池](/Users/zhhnzw/workspace/mygithub/study_summary/src/goroutine_pool.png)

### 合理退出并发协程

* 使用`context`
* 使用`for-range`,`range`能够感知到`channel`的关闭，当`channel`被close，range就会结束
* 使用`for-select`和`,ok`，继续读closed的`channel`，`ok`的值会是`false`, 此时置`channel`为`nil`，`select`不会在`nil`的通道上进行等待
* 使用退出通道退出。定义一个`stopCh`,发送退出信号，监听方式: `case <-stopCh`，这样只需要发送1条消息，每个worker都会收到信号，进而关闭，[参考资料](https://segmentfault.com/a/1190000017251049)

### FAQ

Q: 如何避免向closed channel发送消息？

A: 说明所编写的程序有设计bug。channel关闭原则:

* 不要在消费端关闭channel

* 不要在有多个并行的生产者时对channel执行关闭操作

* 应该只在[唯一的或者最后唯一剩下]的生产者协程中关闭channel，来通知消费者已经没有值可以继续读

* 只要坚持这个原则，就可以确保向一个已经关闭的channel发送数据的情况不可能发生

Q: 退出程序时, 如何防止channel没有消费完？

1. 多个消费者，一个生产者。直接让生产者关闭channel即可。

2. 多个生产者，一个消费者。由消费者通过channel发送停止生产数据的信号给生产者，生产者在生产数据的同时监听此信号channel，收到信号return即可。值得注意的是，这个例子中生产端和接受端都没有关闭消息数据的channel，channel在没有任何goroutine引用的时候会自行关闭，而不需要显示进行关闭。

3. 多个生产者，多个消费者。多个消费者成了信号channel的多个生产者（上一种情况是单一的），因此不可以在信号channel消费端（也即数据channel生产端）关闭此信号channel，这违背了关闭原则，多个消费者发送信号有先后，信号channel被关闭时，其他消费者继续发送信号将引发panic。解决方案是引入一个额外的协调者来关闭附加的退出信号channel。

   设计一个停止生产的信号和一个生产者已经停止的信号，生产者监听停止生产的信号，在return之前发送已经停止的信号，额外的协调者goroutine用于搜集生产者已经停止生产的信号，搜集完整则调用close。

Q: 生产者把消息发送完立刻调用close, 不等消费者消费完，会丢数据吗？

A: 不会丢数据。原理[参考](http://xiaorui.cc/archives/5007)

### goroutine的使用注意事项

子goroutine的panic会引起整个进程crash, 所以需要在goroutine函数写上defer recover, 注意recover只在defer语句生效。