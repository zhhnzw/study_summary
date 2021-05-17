## Golang

### Golang的背景优势

Golang 由 Google 公司出品，由C语言之父Ken Thompson、Rob Pike等牵头发明的，影响力和生命力都有保障，是名副其实的“牛二代”。

<img src="../src/golang/golang_developer.jpeg" alt="Golang 开发团队" />

<img src="../src/golang/golang_project.jpeg" alt="Golang 项目" />

### Golang的特性优势

Golang 为并发而生，通过轻量级协程 Goroutine 来实现程序并发运行，可以充分利用现代CPU多核的特性，有很好的运行性能，而且使用 Goroutine 不需要到操作系统内核切换，占用内存和调度开销小。

Golang 提供了线程安全的`channel`作为协程之间的消息通信方式。

提供关键字defer，延迟执行，适合善后逻辑处理，比如可以尽早避免可能出现的资源泄漏问题。在很大程度上简化了编程，并且在语言描述上显得更为自然，增强了代码的可读性。

Go 语言不是严格的面向对象编程（OOP），它采用的是面向接口编程（IOP），是相对于 OOP 更先进的编程模式。

Golang 可直接编译成机器码，不依赖其他库，部署非常简单，而且还支持交叉编译。

Golang 工具链完善，包括编码风格、静态分析、效率检查、依赖管理这些工具都内置在语言中。

### 更换默认的包安装地址

如果用的 Goland，在创建项目时把 Environment 填好：https://goproxy.io

命令行设置方式：export GOPROXY=https://goproxy.io

[官方文档](https://go-zh.org/doc/effective_go.html)

[大彬博客](https://segmentfault.com/u/lessisbetter)

[draven博客](https://draveness.me/)