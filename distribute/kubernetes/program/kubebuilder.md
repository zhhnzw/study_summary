## Kubebuilder

### GV & GVK & GVR

GV: Api Group & Version

- API Group 是相关 API 功能的集合
- 每个 Group 拥有一或多个 Versions

GVK: Group Version Kind

- 每个 GV 都包含 N 个 api 类型，称之为 Kinds，不同 Version 同一个 Kinds 可能不同

GVR: Group Version Resource

- Resource 是 Kind 的对象标识，一般来 Kind 和 Resource 是 1:1 的，但是有时候存在 1:n 的关系，不过对于 Operator 来说都是 1:1 的关系

举个🌰，我们在 k8s 中的 yaml 文件都有下面这么几行

```yaml
apiVersion: apps/v1 # 这个是 GV，G 是 apps，V 是 v1 
kind: Deployment    # 这个就是 Kind 
sepc:               # 加上下放的 spec 就是 Resource了   
... 
```

根据 GVK K8s 就能找到我们到底要创建什么类型的资源，根据定义的 Spec 创建好资源之后就成为了 Resource，也就是 GVR。GVK/GVR 就是 K8s 资源的坐标，是我们创建/删除/修改/读取资源的基础。

### 使用kubebuilder

#### 安装

我用的Mac：

```bash
$ brew install kubebuilder
$ kubebuilder version
Version: main.version{KubeBuilderVersion:"3.1.0", KubernetesVendor:"unknown", GitCommit:"92e0349ca7334a0a8e5e499da4fb077eb524e94a", BuildDate:"2021-05-29T06:00:59+01:00", GoOs:"darwin", GoArch:"amd64"}
```

#### 初始化项目

```bash
$ mkdir kubebuilder-demo
$ cd kubebuilder-demo/
# domain 是我们项目的域名，repo 是仓库地址，同时也是 go mod 中的 module
$ kubebuilder init --domain mock.com --repo github.com/zhhnzw/k8s-demo/kubebuilder-demo
# 创建 api
$ kubebuilder create api --group zhhnzw --version v1 --kind=CustomPod --resource=true --controller=true
```

```bash
$ tree
.
├── Dockerfile
├── Makefile  # 这里定义了很多脚本命令，例如运行测试，开始执行等 
├── PROJECT   # 这里是 kubebuilder 的一些元数据信息 
├── README.md
├── api
│   └── v1
│       ├── custompod_types.go    # CRD 定义的 go 文件，需要修改这个文件
│       ├── groupversion_info.go  # GV 的定义，无需修改 
│       └── zz_generated.deepcopy.go  # 自动生成的 DeepCopy 方法
├── bin
├── config
│   ├── crd  # 自动生成的 crd 文件，不用修改这里，修改 *_types.go 文件之后执行 make generate 即可
│   ├── default  # 一些默认配置 
│   ├── manager  # 在集群中以 pod 的形式启动控制器（部署在集群中时）
│   ├── prometheus  # 监控指标数据采集配置 
│   ├── rbac  # 在自己的账户下运行控制器所需的权限
│   └── samples  # CRD 的实例 CR
├── controllers
│   ├── custompod_controller.go  # 在这里实现 controller 的逻辑 
│   └── suite_test.go  # 这里写测试 
├── go.mod
├── go.sum
├── hack
└── main.go
```

#### 功能编写

修改`customtype_types.go`文件：

```go
// 期望的状态
// CustomTypeSpec defines the desired state of CustomType
type CustomTypeSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Replicas int `json:"replicas"`
}

// 实际的状态
// CustomTypeStatus defines the observed state of CustomType
type CustomTypeStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Replicas int      `json:"replicas"`
	PodNames []string `json:"podNames"` // 记录已经运行的 Pod 名字
}
```

```bash
# 更新 项目依赖
$ go mod tidy
# 更新 crd yaml
$ make manifests
# 更新 zz_generated.deepcopy.go
$ make generate
```

kubebuilder 已经帮我们实现了 Operator 所需的大部分逻辑，我们只需要在`customtype_controller.go`文件的`Reconcile`函数中实现业务逻辑即可，[完整代码](https://github.com/zhhnzw/k8s-demo/tree/main/kubebuilder-demo)：

```go
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.8.3/pkg/reconcile
func (r *CustomPodReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// your logic here

	return ctrl.Result{}, nil
}
```

#### 本地功能验证

修改crd实例cr的config/samples/zhhnzw_v1_custompod.yaml：

```yaml
apiVersion: zhhnzw.mock.com/v1
kind: CustomPod
metadata:
  name: custompod-sample
spec:
  replicas: 1
```

```bash
# 更新 项目依赖
$ go mod tidy
# 更新 crd yaml
$ make manifests
# 安装 CRD
$ make install
# 运行 controller
$ make run
# 发布一个crd实例
$ kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml
# 查看pod，看是否如期望创建了1个pod
$ kubectl get pods
# 修改 sample yaml 文件的 replicas 为 3
$ kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml
# 查看pod，看是否如期望的把pod副本数量增加到了3个
$ kubectl get pods
# 修改 sample yaml 文件的 replicas 为 1
$ kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml
# 查看pod，看是否如期望的把pod副本数量减少到了1个
$ kubectl get pods
```

#### 部署

上一步只是把 controller 在集群外跑起来的，可以把它部署到集群内

```bash
# 同步本地的项目依赖
$ go mod vendor
# 登录 docker hub，输入用户名和密码
$ docker login
# 构建及推送镜像到 docker hub
$ make docker-build docker-push IMG="2804696160/operator-demo:v1"
```

```bash
# 启动 operator
$ make deploy
# 指定 namespace 查看执行状态
$ kubectl get deployment -n operator-demo-system
# 开启另一个终端，发布上面修改的crd实例
$ kubectl apply -f config/samples/zhhnzw_v1_customtype.yaml
# 查看日志，下面的 pod name 需要换一下
$ kubectl logs operator-demo-controller-manager-6445b5c58c-f4ccs -n operator-demo-system -c manager
```

#### 可选操作

##### 搭建 Docker Registry

相当于本地的 dockerhub

```bash
$ docker run -d -p 5000:5000 --restart always --name registry -v ~/docker-data/docker_registry:/var/lib/registry registry:2
```

修改Docker服务的配置，配置一个本地域名："insecure-registries": ["mock.com:5000"]

修改 /etc/host，添加一行记录：`127.0.0.1    mock.com`

查看 registry 服务当中存储的镜像：http://mock.com:5000/v2/_catalog

在部署推送镜像的时候就可以推送到本地镜像仓库了。

##### 环境准备

```bash
$ git clone git@github.com:kubernetes/kubernetes.git
# 把 kubernetes/staging/src/k8s.io 拷贝到 $GOPATH/src 目录下
$ cp -r kubernetes/staging/src/k8s.io $GOPATH/src
$ mkdir $GOPATH/src/sigs.k8s.io
$ cd $GOPATH/src/sigs.k8s.io
$ git clone git@github.com:kubernetes-sigs/controller-runtime.git
```

### 使用 Operator SDK

Operator 是一个感知应用状态的控制器

CoreOS 推出此 SDK 旨在简化复杂的，有状态应用的管理控制

#### 安装

我用的Mac

```bash
$ brew install operator-sdk
$ operator-sdk version
operator-sdk version: "v1.8.0", commit: "d3bd87c6900f70b7df618340e1d63329c7cd651e", kubernetes version: "v1.20.2", go version: "go1.16.4", GOOS: "darwin", GOARCH: "amd64"
```

#### 初始化项目

```bash
$ mkdir operator-demo
$ cd operator-demo
$ operator-sdk init --domain mock.com --repo github.com/zhhnzw/operator-demo
$ operator-sdk create api --group zhhnzw --version v1 --kind=CustomPod --resource=true --controller=true
```

### 功能编写

生成的代码和 kubebuilder 都一样，省略。

### 部署

大部分操作与 kubebuilder 都一样，只有细节不同，如下：

```bash
# 启动 operator
$ make deploy
# 指定 namespace 查看执行状态
$ kubectl get deployment -n operator-demo-system
```

