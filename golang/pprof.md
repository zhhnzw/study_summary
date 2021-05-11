## pprof

### 工具型应用

#### 使用方法

```go
import "runtime/pprof"  // 工具型应用用这个包
```

##### CPU执行信息采集

开启CPU信息采集：

```go
pprof.StartCPUProfile(w io.Writer)
```

停止CPU信息采集：

```go
pprof.StopCPUProfile()
```

应用执行结束后，就会生成一个文件，保存了 CPU profiling 数据。得到采样数据之后，使用`go tool pprof`工具进行程序的CPU性能问题检查。

##### 内存信息采集

记录程序的堆栈信息

```go
pprof.WriteHeapProfile(w io.Writer)
```

应用执行结束后，就会生成另一个文件，下一步使用`go tool pprof`工具进行程序的内存问题排查。

`go tool pprof`默认是使用`-inuse_space`进行统计，还可以使用`-inuse-objects`查看分配对象的数量。

#### 使用示例

```go
package main

import (
	"flag"
	"fmt"
	"os"
	"runtime/pprof"
	"time"
)

// 一段有问题的代码
func logicCode() {
	var c chan int
	for {
		select {
		case v := <-c:
			fmt.Printf("recv from chan, value:%v\n", v)
		default:

		}
	}
}

func main() {
	var isCPUPprof bool
	var isMemPprof bool

	flag.BoolVar(&isCPUPprof, "cpu", false, "turn cpu pprof on")
	flag.BoolVar(&isMemPprof, "mem", false, "turn mem pprof on")
	flag.Parse()

	if isCPUPprof {
		file, err := os.Create("./cpu.pprof")
		if err != nil {
			fmt.Printf("create cpu pprof failed, err:%v\n", err)
			return
		}
		pprof.StartCPUProfile(file)
		defer pprof.StopCPUProfile()
	}
	for i := 0; i < 8; i++ {
		go logicCode()
	}
	time.Sleep(30 * time.Second)
	if isMemPprof {
		file, err := os.Create("./mem.pprof")
		if err != nil {
			fmt.Printf("create mem pprof failed, err:%v\n", err)
			return
		}
		pprof.WriteHeapProfile(file)
		file.Close()
	}
}
```

通过flag可以通过命令行控制是否开启CPU和Mem的性能分析，执行时加上`-cpu`命令行参数如下：

```bash
go run main.go -cpu
```

等待30秒后会在当前目录下生成一个`cpu.pprof`文件。

### 服务型应用

#### 使用方法

如果应用程序是一直运行的，比如 web 应用，那么可以使用`net/http/pprof`库，它能够在提供 HTTP 服务的同事进行数据采集。

如果使用了默认的`http.DefaultServeMux`（通常是代码直接使用`http.ListenAndServe(“0.0.0.0:8000”, nil)）`，只需要在你的web server端代码中按如下方式导入`net/http/pprof`

```go
import _ "net/http/pprof"  // 服务型应用用这个包
```

如果你使用自定义的 Mux，则需要手动注册一些路由规则：

```go
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

如果使用的是gin框架，那么推荐使用[github.com/gin-contrib/pprof](https://github.com/gin-contrib/pprof)，在代码中通过以下命令注册pprof相关路由。

```go
pprof.Register(router)
```

不管哪种方式，都可以通过访问上述路由查看相关内容：

- /debug/pprof/profile：访问这个链接会自动进行 CPU profiling，持续 30s，并生成一个文件供下载
- /debug/pprof/heap： Memory Profiling 的路径，访问这个链接会得到一个内存 Profiling 结果的文件
- /debug/pprof/block：block Profiling 的路径
- /debug/pprof/goroutines：运行的 goroutines 列表，以及调用关系

### go tool pprof 命令工具

不管是工具型应用还是服务型应用，使用相应的pprof库获取到数据文件之后，下一步都要对这些数据进行分析检查。

```bash
go tool pprof [binary] [source]
```

其中：

- binary 是应用的二进制文件，用来解析各种符号；
- source 表示 profile 数据的来源，可以是本地的文件，也可以是 http 地址。

**注意事项：** 获取的 Profiling 数据是动态的，要想获得有效的数据，请保证应用处于较大的负载（比如正在生成中运行的服务，或者通过其他工具模拟访问压力）。否则如果应用处于空闲状态，得到的结果可能没有任何意义。

#### 使用示例

以上述代码为案例，生成了`cpu.pprof`文件，然后在命令行执行：

```bash
go tool pprof cpu.pprof
```

执行上面的代码会进入交互界面如下：

```bash
runtime_pprof $ go tool pprof cpu.pprof
Type: cpu
Time: Jun 28, 2019 at 11:28am (CST)
Duration: 20.13s, Total samples = 1.91mins (568.60%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)  
```

接下来可以在交互界面输入`top3`来查看程序中占用CPU前3位的函数：

```bash
(pprof) top3
Showing nodes accounting for 100.37s, 87.68% of 114.47s total
Dropped 17 nodes (cum <= 0.57s)
Showing top 3 nodes out of 4
      flat  flat%   sum%        cum   cum%
    42.52s 37.15% 37.15%     91.73s 80.13%  runtime.selectnbrecv
    35.21s 30.76% 67.90%     39.49s 34.50%  runtime.chanrecv
    22.64s 19.78% 87.68%    114.37s 99.91%  main.logicCode
```

其中：

- flat：当前函数占用CPU的耗时
- flat：:当前函数占用CPU的耗时百分比
- sun%：函数占用CPU的耗时累计百分比
- cum：当前函数加上调用当前函数的函数占用CPU的总耗时
- cum%：当前函数加上调用当前函数的函数占用CPU的总耗时百分比
- 最后一列：函数名称

在列表中找出自己编写的函数，可以看出，`main.logicCode`对CPU的消耗过高，接下来使用`list`命令对该函数的实现代码CPU耗时进行下一步检查：

```bash
(pprof) list logicCode
Total: 1.82mins
ROUTINE ======================== main.logicCode in /Users/zhhnzw/workspace/mygithub/chat/t.go
    23.19s   1.82mins (flat, cum)   100% of Total
         .          .     11:// 一段有问题的代码
         .          .     12:func logicCode() {
         .          .     13:   var c chan int
         .          .     14:   for {
         .          .     15:           select {
    23.19s   1.82mins     16:           case v := <-c:
         .          .     17:                   fmt.Printf("recv from chan, value:%v\n", v)
         .          .     18:           default:
         .          .     19:
         .          .     20:           }
         .          .     21:   }
```

通过检查发现大部分CPU资源被16行占用，`select`语句中的default没有内容会导致`select`代码块不会发生阻塞，从而`case v:=<-c:`被`for`无限制的循环执行，浪费了CPU资源。解决办法：可以在default分支里添加一行`time.Sleep(time.Second)`。