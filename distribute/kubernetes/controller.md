## Controller

Pod 是 Kubernetes 项目里最核心的编排对象，而编排功能则是由 Controller（kube-controller-manager） 完成。Pod 和 Controller 之间通过 label 标签建立关系。

Deployment 是控制器的一种，控制器要么就是创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。

