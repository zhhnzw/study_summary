## Kubernetes

k8s是容器集群管理系统，可以让部署容器化应用更加简洁和高效。

### Pod

Pod 是一组容器的集合，是 k8s 最小部署单元，一个 Pod 内的容器是共享网络的。 

Pod 的生命周期是短暂的，重启之后就是一个新的 Pod 实例了。

### Controller

确保预期的 Pod 副本数量，无状态和有状态的应用部署，执行一次性和定时任务。

### Service

定义一组 Pod 的访问规则。

由 Controller 创建 Pod 进行部署，通过 Service 对外提供访问服务。

### 使用kubeadm搭建k8s集群

mac下直接使用docker桌面软件附带的k8s[参考](https://segmentfault.com/a/1190000038167301)，借助kubeadm工具在多台虚拟机下搭建k8s集群参考。