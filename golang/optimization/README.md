## 测试与调优

### 测试

> Test Driven Development

在编写某个功能的代码之前先编写测试代码，通过测试来推动整个开发的进行。

> Debugging Sucks ! Testing Rocks !

这个口号就是鼓励我们多做测试，而不是多做调试。

### 性能调优

Go语言项目中的性能优化主要有以下几个方面：

- CPU profile：报告程序的 CPU 使用情况，按照一定频率去采集应用程序在 CPU 和寄存器上面的数据
- Memory Profile（Heap Profile）：报告程序的内存使用情况
- Block Profiling：报告 goroutines 不在运行状态的情况，可以用来分析和查找死锁等性能瓶颈
- Goroutine Profiling：报告 goroutines 的使用情况，有哪些 goroutine，它们的调用关系是怎样的