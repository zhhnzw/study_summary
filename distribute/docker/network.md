## docker网络

| Docker网络模式 | 配置                      | 说明                                                         |
| -------------- | ------------------------- | ------------------------------------------------------------ |
| host模式       | –net=host                 | 容器和宿主机共享Network namespace。                          |
| container模式  | –net=container:NAME_or_ID | 容器和另外一个容器共享Network namespace。 kubernetes中的pod就是多个容器共享一个Network namespace。 |
| none模式       | –net=none                 | 容器有独立的Network namespace，但并没有对其进行任何网络设置，如分配veth pair 和网桥连接，配置IP等。 |
| bridge模式     | –net=bridge               | （默认为该模式）                                             |

### docker compose 网络

默认情况下，在docker中启动的各个容器是各自有各自独立的网络的，因此各个容器之间彼此隔离。而放在同一个docker compose中启动的容器，是在同一个局域网下的

因为默认情况下，compose会为我们的应用创建一个网络，服务的每个容器都会加入该网络中

容器还能以服务名称作为hostname被其他容器访问(docker embedded dns, 容器之间能够通过 vaild name or net-alias or link互相发现)

要使不同compose文件的容器位于同一局域网, 需要在compose文件中增加`networks`设置, `networks`设置可用连接到外部网络(即使不是compose创建的也可以)

### 环境变量