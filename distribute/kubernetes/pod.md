## Pod
Pod是 Kubernetes 项目里最核心的编排对象，是 Kubernetes 项目中的最小编排单位。

如果有一组容器，它们之间有亲密的耦合关系，要做统一的调度管理，这样的基础设施就是Pod。

如果把 Pod 看成传统环境里的“机器”、把容器看作是运行在这个“机器”里的“用户程序”，那么很多关于 Pod 对象的设计就非常容易理解了。

比如，**凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。**

这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。比如，配置这个“机器”的网卡（即：Pod 的网络定义），配置这个“机器”的磁盘（即：Pod 的存储定义），配置这个“机器”的防火墙（即：Pod 的安全定义）。更不用说，这台“机器”运行在哪个服务器之上（即：Pod 的调度）。

**Pod只是一个逻辑概念**，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和 Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。

![Pod](../../src/distribute/k8s/pod.png)

Pod 的实现需要使用一个中间容器，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。对于 Pod 里的容器 A 和容器 B 来说：

- 它们可以直接使用 localhost 进行通信；

- 它们看到的网络设备跟 Infra 容器看到的完全一样；

- 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；

- 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；

- Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。

### 例子

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

debian-container 和 nginx-container 都声明挂载了 shared-data 这个 Volume。而 shared-data 是 hostPath 类型。所以，它对应在宿主机上的目录就是：/data。而这个目录，其实就被同时绑定挂载进了上述两个容器当中。

这就是为什么，nginx-container 可以从它的 /usr/share/nginx/html 目录中，读取到 debian-container 生成的 index.html 文件的原因。

#### sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```

所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

geektime/sample:v2 容器执行了一句"cp /sample.war /app"，把应用的 WAR 包拷贝到 /app 目录下，然后退出。而后这个 /app 目录，就挂载了一个名叫 app-volume 的 Volume。

Tomcat 容器，同样声明了挂载 app-volume 到自己的 webapps 目录下。

所以，等 Tomcat 容器启动时，它的 webapps 目录下就一定会存在 sample.war 文件：这个文件正是 WAR 包容器启动时拷贝到这个 Volume 里面的，而这个 Volume 是被这两个容器共享的。

像这样，我们就用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题，WAR 包与 Tomcat 容器就可以分开做版本管理了。**这个“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。**

另一个场景：容器的日志收集。Pod 内的容器挂载到一个 Volume，同时运行一个做日志收集的 sidecar 容器也挂载上该 Volume，接下来 sidecar 容器就只需要做一件事儿，那就是不断地从自己挂载的目录里读取日志文件，转发到 MongoDB 或者 Elasticsearch 中存储起来。这个 sidecar 容器可以与主容器一起写到 spec.container 下，这也属于 sidecar 容器设计模式。


### 一个单 Pod 的 Deployment 与一个 Pod 最主要的区别?

Pod 恢复机制（restartPolicy），默认值是 Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

但一定要强调的是，Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。

而如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理 Pod，哪怕你只需要一个 Pod 副本。

### Pod的状态是 Running，但是应用其实已经停止服务？

对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。

解决方法：给 Pod 配置健康检查 livenessProbe，livenessProbe 可以定义为发起 HTTP 或者 TCP 。