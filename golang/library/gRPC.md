### RPC（Remote Procedure Call）

调用远程计算机上提供的函数。服务端实现具体的函数功能，客户端本地留有远程服务接口定义存根，表现形式是函数调用，但实际上与api调用一样是向远程计算器发起的网络请求。

与HTTP的不同点是：传输协议、序列化方式。

### gRPC

通信协议基于标准的 HTTP/2 设计，支持双向流、消息头压缩、单 TCP 的多路复用、服务端推送等特性。

序列化支持 PB（Protocol Buffer）和 JSON，PB 是一种语言无关的高性能序列化框架，基于 HTTP/2 + PB, 保障了 RPC 调用的高性能。

gRPC还可以很简单的插入身份认证、负载均衡、日志和监控等功能。

### Protocol Buffers

可以理解为与 json、xml 作用相类似。

为什么使用Protocol Buffer？

* 它和开发语言无关
* 可以生成所有主流开发语言的代码
* 数据是二进制格式的，串行化的效率高，载荷（Payload）比较小
* 也很适合传递大量的数据

gRPC默认使用 Protocol Buffer 作为接口设计语言（IDL，interface design language），这个 .proto 文件包括两部分：

* gRPC服务的定义
* 服务端和客户端之间传递的消息

```protobuf
syntax = "proto3";

package xxx.project; // 一个项目用到多个proto时，可能会命名重复，用这个可以隔离开命名空间

message Date {
	int32 year = 1;
	int32 month = 2;
	int32 day = 3;
}

message Person {
	int32 id = 1; 
	string name = 2;
	float height = 3;
	float weight = 4; // 这个数字是tag，tag的数值是非常重要的，Protocol Buffer会根据tag的数值来序列化，而不是根据字段名。举个例子，服务端对字段做了增删，把tag也调整了一下，而此时客户端的proto文件还在用老的，则此时客户端仍然会按老的tag绑定关系来解析。
	bytes avatar = 5; 
	string email = 6;
	bool email_verified = 7;
	repeated string phone_numbers = 8; // repeated 表示这是个数组字段
	Gender gender = 11;
	Date birthday = 12; // 引用了自定义的消息类型，在golang中会被解析成结构体
	
	enum Gender {
		NOT_SPECIFIED = 0;
		FEMALE = 1;
		MALE = 2;
	}
	
	// 针对以上proto文件更新tag绑定引起的问题，引入了保留字段的功能，标记为不再使用的tag和字段名，客户端如果在使用这些tag或字段名，就会抛出异常
	reserved 9, 10, 20 to 100, 200 to max; // 保留的tag
	reserved "foo", "bar"; // 保留的字段名
}

service Employee {
  rpc GetByName (Request) returns (Reply) {}
}
```

### gRPC四种通信方式

　　1. 简单RPC（Simple RPC）：就是一般的rpc调用，一个请求对象对应一个返回对象。

　　2. 服务端流式RPC（Server-side streaming RPC）：一个请求对象，服务端返回数据流（数组，每次传一个元素）。

　　3. 客户端流式RPC（Client-side streaming RPC）：客户端传入连续的请求对象（数组），服务端返回一个响应结果。

　　4. 双向流式RPC（Bidirectional streaming RPC）：结合客户端流式RPC和服务端流式RPC，可以传入多个对象，返回多个响应对象。

流式接口的使用场景：一个接口要发送大量数据时，一次只传输一部分数据，分批传输数据，比如文件的传输，用流式接口可以降低服务器的瞬时压力，对客户端的响应也更快。

### gRPC拦截器

与HTTP服务的拦截器功能类似，可以在RPC方法前、后执行一些操作。

#### 拦截器分类

* 一元拦截器 UnaryInterceptor
* 流式拦截器 StreamInterceptor

典型应用场景：统一接口身份认证。

gRPC支持拦截器链。