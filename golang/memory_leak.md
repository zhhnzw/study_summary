## 内存泄露

### goroutine可能泄露的缓冲区

案例: `for select`包含`default`子句时, 可能造成内存泄露, [参考]([https://go-zh.org/doc/effective_go.html#%E6%B3%84%E9%9C%B2%E7%BC%93%E5%86%B2](https://go-zh.org/doc/effective_go.html#泄露缓冲)

当`select`的`case`为写操作时，刚好该channel缓冲区满，发生阻塞时，没写进去，会执行`default`，此时没写进去的那个变量将被丢弃，被垃圾回收器回收

