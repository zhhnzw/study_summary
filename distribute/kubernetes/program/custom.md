## 二次开发

从“使用 Kubernetes 部署代码”，到“使用 Kubernetes 编写代码”的蜕变过程，正是从一个 Kubernetes 用户，到 Kubernetes 玩家的晋级之路。

如何理解“Kubernetes 编程范式”，如何为 Kubernetes 添加自定义 API 对象，编写自定义控制器，正是这个晋级过程中的关键点。

在互联网级别的大规模集群里，Kubernetes 内置的编排对象，很难做到完全满足所有需求。所以，很多实际的容器化工作，都会要求设计一个自定义的编排对象，实现自定义的控制器模式。

### 创建一个Network的CRD（Custom Resource Definition）

#### 创建CR实例配置文件

创建 example-network.yaml ：

```yaml
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
```

可以看到，描述“网络”的 API 资源类型是 Network；API 组是`samplecrd.k8s.io`；API 版本是 v1。

这个 YAML 文件，就是一个具体的“自定义 API 资源实例，也叫 CR（Custom Resource）。

#### 创建CRD配置文件

创建 network.yaml ：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```

可以看到，在这个 CRD 中，我指定了“`group: samplecrd.k8s.io`”“`version: v1`”这样的 API 信息，也指定了这个 CR 的资源类型叫作 Network，复数（plural）是 networks。

然后，我还声明了它的 scope 是 Namespaced，即：我们定义的这个 Network 是一个属于 Namespace 的对象，类似于 Pod。

#### 编码

上述两个 YAML 文件 是给出了 Network API 资源类型的 API 部分的宏观定义，这时候，Kubernetes 已经能够认识和处理所有声明了 API 类型是“`samplecrd.k8s.io/v1/network`”的 YAML 文件了。

接下来，我还需要让 Kubernetes“认识”这种 YAML 文件里描述的“网络”部分，比如“cidr”（网段），“gateway”（网关）这些字段的含义。

创建如下结构的代码项目：

```bash
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```

pkg/apis/samplecrd 目录下的 register.go 文件，用来放置后面要用到的全局变量。

pkg/apis/samplecrd/v1 目录下的 doc.go 文件（Golang 的文档源文件）：

```go
// +k8s:deepcopy-gen=package
 
// +groupName=samplecrd.k8s.io
package v1
```

在这个文件中，有 +<tag_name>[=value] 格式的注释，这就是 Kubernetes 进行代码生成要用的 Annotation 风格的注释。

其中，+k8s:deepcopy-gen=package 意思是，请为整个 v1 包里的所有类型定义自动生成 DeepCopy 方法；而`+groupName=samplecrd.k8s.io`，则定义了这个包对应的 API 组的名字。

源码如下的 types.go 文件作用就是定义一个 Network 类型到底有哪些字段（比如，spec 字段里的内容），这个文件也有Annotation 风格的注释。

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Network describes a Network resource
type Network struct {
	// TypeMeta is the metadata for the resource, like kind and apiversion
	metav1.TypeMeta `json:",inline"`
	// ObjectMeta contains the metadata for the particular object, including
	// things like...
	//  - name
	//  - namespace
	//  - self link
	//  - labels
	//  - ... etc ...
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec is the custom resource spec
	Spec NetworkSpec `json:"spec"`
}

// NetworkSpec is the spec for a Network resource
type NetworkSpec struct {
	// Cidr and Gateway are example custom spec fields
	//
	// this is where you would put your custom resource data
	Cidr    string `json:"cidr"`
	Gateway string `json:"gateway"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkList is a list of Network resources
type NetworkList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Network `json:"items"`
}
```

可以看到 Network 类型定义方法跟标准的 Kubernetes 对象一样，都包括了 TypeMeta（API 元数据）和 ObjectMeta（对象元数据）字段。

而其中的 Spec 字段，就是需要我们自己定义的部分。所以，在 networkspec 里，我定义了 Cidr 和 Gateway 两个字段。其中，每个字段最后面的部分比如`json:"cidr"`，指的就是这个字段被转换成 JSON 格式之后的名字，也就是 YAML 文件里的字段名字。

此外，除了定义 Network 类型，你还需要定义一个 NetworkList 类型，用来描述**一组 Network 对象**应该包括哪些字段。之所以需要这样一个类型，是因为在 Kubernetes 中，获取所有 X 对象的 List() 方法，返回值都是List 类型，而不是 X 类型的数组。这是不一样的。

+genclient 注释的意思是：请为下面这个 API 资源类型生成对应的 Client 代码。而 +genclient:noStatus 的意思是：这个 API 资源类型定义里，没有 Status 字段。否则，生成的 Client 就会自动带上 UpdateStatus 方法。

如果是需要 Status 字段的场景，删掉这句 +genclient:noStatus 注释，保留 +genclient 这句即可。

+genclient 只需要写在 Network 类型上，而不用写在 NetworkList 上。因为 NetworkList 只是一个返回值类型，Network 才是“主类型”。

有一句`+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`的注释。它的意思是，请在生成 DeepCopy 的时候，实现 Kubernetes 提供的 runtime.Object 接口。否则，在某些版本的 Kubernetes 里，你的这个类型定义会出现编译错误。这是一个固定的操作，记住即可。

最后再看 pkg/apis/samplecrd/v1/register.go 文件，Network 资源类型在服务器端的注册的工作，APIServer 会自动帮我们完成。但与之对应的，我们还需要让客户端也能“知道” Network 资源类型的定义。这就是需要添加的 register.go 文件的作用，它最主要的功能，就是定义了如下所示的 addKnownTypes() 方法：

```go
package v1
...
// addKnownTypes adds our types to the API scheme by registering
// Network and NetworkList
func addKnownTypes(scheme *runtime.Scheme) error {
 scheme.AddKnownTypes(
  SchemeGroupVersion,
  &Network{},
  &NetworkList{},
 )
 
 // register the type in the scheme
 metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
 return nil
}
```

有了这个方法，Kubernetes 就能够在后面生成客户端的时候，“知道” Network 以及 NetworkList 类型的定义了。

像上面这种 register.go 文件里的内容其实是非常固定的，以后可以直接使用这部分代码做模板，然后把其中的资源类型、GroupName 和 Version 替换成其他定义即可。

#### 代码生成

![](../../src/distribute/k8s/crd-demo00.png)

可以看到经过上面的代码编写，编译还不能通过，提示了 Network 结构体实例不满足`AddDnowTypes`函数要接收的`Object`接口，缺少了`DeepCopyObject`方法的实现。

注：从 https://github.com/kubernetes/code-generator 拷贝 hack 目录到我们的工程根目录下

在 hack 目录下添加 tools.go 文件，写入如下内容：

```go
// +build tools
package tools

import _ "k8s.io/code-generator"
```

然后修改 hack 目录下的 update-codegen.sh 文件：

```sh
SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..

# generate the code with:
# - --output-base because this script should also be able to run inside the vendor dir of
#   k8s.io/kubernetes. The output-base is needed for the generators to output into the vendor dir
#   instead of the $GOPATH directly. For normal projects this can be dropped.

./vendor/k8s.io/code-generator/generate-groups.sh all \
  github.com/zhhnzw/k8s-demo/crd_demo/pkg/client \
  github.com/zhhnzw/k8s-demo/crd_demo/pkg/apis \
  samplecrd:v1 \
  --go-header-file ${SCRIPT_ROOT}/hack/boilerplate.go.txt

```

接下来使用 Kubernetes 提供的代码生成工具，为定义的 Network 资源类型自动生成 clientset、informer 和 lister，和 DeepCopyObject 方法：

```bash
# 安装项目依赖
$ go mod tidy 

# 生成 vendor 目录
$ go mod vendor

# 给予脚本执行权限

$ chmod 777 ./vendor/k8s.io/code-generator/generate-groups.sh

# 在项目根目录下执行代码自动生成脚本
$ ./hack/update-codegen.sh
```

代码生成工作完成之后，我们再查看一下这个项目的目录结构：

```
$ tree
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    │   └── samplecrd
    │       ├── constants.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        ├── informers
        └── listers
```

其中，pkg/apis/samplecrd/v1 下面的 zz_generated.deepcopy.go 文件，就是自动生成的 DeepCopy 代码文件。

而整个 client 目录，以及下面的三个包（clientset、informers、 listers），都是 Kubernetes 为 Network 类型生成的客户端库，这些库会在后面编写自定义控制器的时候用到。

#### 注册CRD并创建CR

```bash
$ kubectl apply -f crd/network.yaml
$ kubectl apply -f example/example-network.yaml
```

查看创建的自定义 Network 对象：

```bash
$ kubectl get network
$ kubectl describe network example-network
```

### 编写自定义控制器（Custom Controller）

“声明式 API”并不像“命令式 API”那样有着明显的执行逻辑。这就使得基于声明式 API 的业务功能实现，往往需要通过控制器模式来“监视”API 对象的变化（比如，创建或者删除 Network），然后以此来决定实际要执行的具体工作。

#### 涉及到的概念和执行机制

所谓的 Informer 通知器，是自定义控制器跟 APIServer 进行数据同步的重要组件，是一个自带缓存和索引机制，可以触发 Handler 的客户端库。这个本地缓存在 Kubernetes 中一般被称为 Store，索引一般被称为 Index。

Informer 使用了 Reflector 包，它是一个可以通过 ListAndWatch 机制获取并监视 API 对象变化的客户端封装。

Reflector 和 Informer 之间，用到了一个“增量先进先出队列”进行协同。而 Informer 与要编写的控制循环之间，则使用了一个工作队列来进行协同。

在实际应用中，除了控制循环之外的所有代码，实际上都是 Kubernetes 为你自动生成的，即：pkg/client/{informers, listers, clientset}里的内容。

而这些自动生成的代码，就为我们提供了一个可靠而高效地获取 API 对象“期望状态”的编程库。

所以，接下来，作为开发者，就只需要关注如何拿到“实际状态”，然后如何拿它去跟“期望状态”做对比，从而决定接下来要做的业务逻辑即可。

### Operator SDK

Operator 是一个感知应用状态的控制器

CoreOS 推出此 SDK 旨在简化复杂的，有状态应用的管理控制

#### 安装

我用的Mac

```bash
$ brew install operator-sdk
$ operator-sdk version
operator-sdk version: "v1.8.0", commit: "d3bd87c6900f70b7df618340e1d63329c7cd651e", kubernetes version: "v1.20.2", go version: "go1.16.4", GOOS: "darwin", GOARCH: "amd64"
```

#### 搭建 Docker Registry

相当于本地的 dockerhub

```bash
$ docker run -d -p 5000:5000 --restart always --name registry -v ~/docker-data/docker_registry:/var/lib/registry registry:2
```

修改Docker服务的配置，配置一个本地域名："insecure-registries": ["mock.com:5000"]

修改 /etc/host，添加一行记录：`127.0.0.1    mock.com`

查看 registry 服务当中存储的镜像：http://mock.com:5000/v2/_catalog

#### 环境准备

```bash
$ git clone git@github.com:kubernetes/kubernetes.git
# 把 kubernetes/staging/src/k8s.io 拷贝到 $GOPATH/src 目录下
$ cp -r kubernetes/staging/src/k8s.io $GOPATH/src
$ mkdir $GOPATH/src/sigs.k8s.io
$ cd $GOPATH/src/sigs.k8s.io
$ git clone git@github.com:kubernetes-sigs/controller-runtime.git
```

#### 初始化项目

```bash
$ mkdir operator-demo
$ cd operator-demo
$ operator-sdk init --domain mock.com --repo github.com/zhhnzw/operator-demo/v1
$ operator-sdk create api --group zhhnzw --version v1 --kind=CustomPod --resource=true --controller=true
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
# 更新 crd yaml
$ make manifests
# 更新 zz_generated.deepcopy.go
$ make generate
```

把业务功能写在`customtype_controller.go`文件的`Reconcile`函数即可：
```go
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.8.3/pkg/reconcile
func (r *CustomPodReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// your logic here

	return ctrl.Result{}, nil
}
```

#### 功能验证

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
# 运行
$ make install run
# 发布一个crd实例
$ kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml
# 查看pod
$ kubectl get pods
```

修改 yaml 文件的 replicas 为 3，执行：`kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml`，然后再查看 pod，检查是否如期望的增加了副本数量。

再修改 yaml 文件的 replicas 为 1，执行：`kubectl apply -f config/samples/zhhnzw_v1_custompod.yaml`，然后再查看 pod，检查是否如期望的减少了副本数量。

#### 部署

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
# 查看执行状态
$ kubectl get deployment -n operator-demo-system
# 开启另一个终端，发布上面修改的crd实例
$ kubectl apply -f config/samples/zhhnzw_v1_customtype.yaml
$ kubectl get CustomType
# 查看日志，下面的 pod name 需要换一下
$ kubectl logs operator-demo-controller-manager-6445b5c58c-f4ccs -n operator-demo-system -c manager
```



构建项目，并推送新的镜像：`sudo make docker-build docker-push IMG="2804696160/operator-demo:v1"`，这个`mock.com:5000`就是前三步搭建的本地 Docker Registry 服务。

### 使用kubebuilder

#### 安装

我的Mac：

```bash
$ brew install kubebuilder
$ kubebuilder version
Version: main.version{KubeBuilderVersion:"3.1.0", KubernetesVendor:"unknown", GitCommit:"92e0349ca7334a0a8e5e499da4fb077eb524e94a", BuildDate:"2021-05-29T06:00:59+01:00", GoOs:"darwin", GoArch:"amd64"}
```

#### 初始化项目

```bash
$ mkdir kubebuilder-demo
$ cd kubebuilder-demo/
$ kubebuilder init --domain mock.com --repo github.com/zhhnzw/k8s-demo/kubebuilder-demo/v1
$ kubebuilder create api --group zhhnzw --version v1 --kind=CustomPod --resource=true --controller=true
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

把业务功能写在`customtype_controller.go`文件的`Reconcile`函数即可：
```go
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.8.3/pkg/reconcile
func (r *CustomPodReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// your logic here

	return ctrl.Result{}, nil
}
```
