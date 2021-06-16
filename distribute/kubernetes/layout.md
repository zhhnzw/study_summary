## 编排

### 声明式 API

比如：控制器中的 Deployment，服务中的 NodePort，等等，都是 Kubernetes 提供的 API 对象，我们常用的使用方式就是通过 YAML 文件来声明要使用的这些  API 对象。

#### 基于YAML文件的操作方式，就是“声明式 API”吗？

不是。

比如：在本地编写一个 Deployment 的 YAML 文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

然后，使用 kubectl create 命令在 Kubernetes 里创建 nginx 这个 Deployment 对象：

```bash
$ kubectl create -f nginx.yaml
```

这样，两个 Nginx 的 Pod 就会运行起来了。

接下来修改这个 YAML 文件里的 Pod 模板部分，把 Nginx 容器的镜像改成 1.7.9，如下所示：

```yaml
...
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

再接下来，就可以执行一句 kubectl replace 操作，来完成这个 Deployment 的更新：

```bash
$ kubectl replace -f nginx.yaml
```

但是，对于上面这种先 kubectl create，再 replace 的操作，我们称为**命令式配置文件操作。**

#### 什么才是“声明式 API”呢？

使用 kubectl apply 命令来创建这个 Deployment：

```bash
$ kubectl apply -f nginx.yaml
```

这样，Nginx 的 Deployment 就被创建了出来，这看起来跟 kubectl create 的效果一样。

然后，我再修改一下 nginx.yaml 里定义的镜像：

```yaml
...
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

这时候，关键来了。

在修改完这个 YAML 文件之后，我不再使用 kubectl replace 命令进行更新，而是继续执行一条 kubectl apply 命令，即：

```bash
$ kubectl apply -f nginx.yaml
```

这时，Kubernetes 就会立即触发这个 Deployment 的“滚动更新”。

Q：它跟 kubectl replace 命令有什么本质区别吗？

A：kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，**替换原有的 API 对象**；而 kubectl apply，则是执行了一个**对原有 API 对象做局部更新的 PATCH 操作**。 类似地，kubectl set image 和 kubectl edit 也是对已有 API 对象的修改。对于声明式请求（比如，kubectl apply），**一次能处理多个写操作，并且具备 Merge 能力**。

#### 总结

- 首先，所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。
- 其次，“声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。
- 最后，也是最重要的，有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。

Reconcile：Kubernetes 的控制器，实际上就是一个“死循环”，它不断地获取“实际状态”，然后与“期望状态”作对比，并以此为依据决定下一步的操作。

** 声明式 API，是 Kubernetes 项目编排能力“赖以生存”的核心所在。**

### 权限控制之RBAC

Kubernetes 中所有的 API 对象，都保存在 Etcd 里。而对这些 API 对象的操作，都是通过访问 kube-apiserver 实现的。其中一个非常重要的原因，就是需要 APIServer 来做授权工作。而在 Kubernetes 项目中，负责完成授权（Authorization）工作的机制，就是 RBAC：基于角色的访问控制（Role-Based Access Control）。Kubernetes 的 RBAC 与基于数据库来实现 RBAC 的方式不同，但核心思想是相同的。

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以使你在 Kubernetes 里定义的“用户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。

#### Role

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace  # 指定了它能产生作用的 Namepace 是：mynamespace
  name: example-role
rules:  # 定义的权限规则
- apiGroups: [""]
  resources: ["pods"]  # 限制的资源是 Pod
  verbs: ["get", "watch", "list"]  # 定义了允许的操作
```

以上的含义是：允许“被作用者”，对 mynamespace 下面的 Pod 对象，进行 GET、WATCH 和 LIST 操作。

#### RoleBinding

以上的 Role 还没有指定"被作用者"，需要通过 RoleBinding 来指定，它的作用相当于通过数据库来实现 RBAC 方式中的关联表。

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace  # 仅在 Namepace 是：mynamespace 下生效
subjects:  # "被作用者"
- kind: ServiceAccount  # 类型是 ServiceAccount，名字是 example-user
  name: example-sa
  namespace: mynamespace
roleRef:  # 与上一步定义的 Role 绑定
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

还缺一个 ServiceAccount 的定义：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

#### Subject

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa  # 指定要使用的 ServiceAccount 的名字是：example-sa
```

#### 操作验证

创建如上3个对象：

```bash
$ kubectl create -f svc-account.yaml
$ kubectl create -f role-binding.yaml
$ kubectl create -f role.yaml
$ kubectl get sa -n mynamespace -o yaml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: "409327"
    ...
  secrets:
  - name: example-sa-token-vmfg6
```

可以看到，Kubernetes 会为一个 ServiceAccount 自动创建并分配一个 Secret 对象，即：上述 ServiceAcount 定义里最下面的 secrets 字段。

这个 Secret，就是这个 ServiceAccount 对应的、用来跟 APIServer 进行交互的授权文件，一般称它为：Token。Token 文件的内容一般是证书或者密码，它以一个 Secret 对象的方式保存在 Etcd 当中。

等这个 Pod 运行起来之后，我们就可以看到，该 ServiceAccount 的 token，也就是一个 Secret 对象，被 Kubernetes 自动挂载到了容器的 /var/run/secrets/kubernetes.io/serviceaccount 目录下，如下所示：

```bash
$ kubectl describe pod sa-token-test -n mynamespace
Name:               sa-token-test
Namespace:          mynamespace
...
Containers:
  nginx:
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from example-sa-token-vmfg6 (ro)
$ kubectl exec -it sa-token-test -n mynamespace -- /bin/bash
root@sa-token-test:/# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
```

这样，容器里的应用，就可以使用这个 ca.crt 来访问 APIServer 了。更重要的是，此时它只能够做 GET、WATCH 和 LIST 操作。因为 example-sa 这个 ServiceAccount 的权限，已经被绑定了 Role 做了权限限制。

#### 用户组

一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：

```tex
system:serviceaccount:<ServiceAccount 名字 >
```

而它对应的内置“用户组”的名字，就是：

```tex
system:serviceaccounts:<Namespace 名字 >
```

比如，可以在 RoleBinding 里定义如下的 subjects：

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```

这就意味着这个 Role 的权限规则，作用于 mynamespace 里的所有 ServiceAccount。这就用到了“用户组”的概念。也就相当于指定 Role 与指定的 mynamespace 下所有的 ServiceAccount 都在关联表中建立了关联记录。

而下面这个例子：

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

就意味着这个 Role 的权限规则，作用于整个系统里的所有 ServiceAccount。

#### ClusterRole 与 ClusterRoleBinding

Role 和 RoleBinding 对象都是 Namespaced 对象（Namespaced Object），它们对权限的限制规则仅在它们自己的 Namespace 内有效，roleRef 也只能引用当前 Namespace 里的 Role 对象。那么，对于非 Namespaced（Non-namespaced）对象（比如：Node），或者，某一个 Role 想要作用于所有的 Namespace 的时候，该如何去做授权呢？

这时候，就必须要使用 ClusterRole 和 ClusterRoleBinding 这两个组合了。这两个 API 对象的用法跟 Role 和 RoleBinding 完全一样。只不过，它们的定义里，没有了 Namespace 字段：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

上面的例子里的 ClusterRole 和 ClusterRoleBinding 的组合，意味着名叫 example-user 的用户，拥有对所有 Namespace 里的 Pod 进行所有操作的权限。

Kubernetes 提供了四个预先定义好的 ClusterRole 来供用户直接使用：

1. cluster-admin：最高权限
2. admin：
3. edit：
4. view：规定了被作用者只有 Kubernetes API 的只读权限。