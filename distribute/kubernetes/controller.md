## Controller

Pod 是 Kubernetes 项目里最核心的编排对象，而编排功能则是由 Controller（kube-controller-manager） 完成。Pod 和 Controller 之间通过 label 标签建立关系。

Deployment 是控制器的一种，控制器要么就是创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。

### Deployment

Deployment 所管理的 Pod，互相之间没有顺序，也无所谓运行在哪台宿主机上。需要的时候，Deployment 就可以通过 Pod 模板创建新的 Pod；不需要的时候，Deployment 就可以“杀掉”任意一个 Pod。

典型应用场景是：Web服务。

### StatefulSet

有状态应用， StatefulSet 可以认为是对 Deployment 的改良。

StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，编号与 Pod 的名字和 hostname 等标识信息绑定上，并且按照编号顺序逐一完成创建工作。当需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。

典型应用场景是：按顺序启动的主从关系，或者一个数据库应用的多个存储实例。

应用的多个实例分别绑定了不同的存储数据，一对一绑定，即使 Pod 被重新创建，也会按照之前的编号顺序和绑定关系重新创建出新的 Pod，虽然新的 Pod 的集群ip发生了变化，由于这些 Pod 名字里的编号不变，那么 Service 里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，网络标识和数据的绑定关系与之前都是相同的。