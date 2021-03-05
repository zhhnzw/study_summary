## 操作案例

yaml 文件需要配置的字段比较多，用以下命令可以取得导出的yaml文件

```bash
kubectl create deployment web --image=nginx:1.14 --dry-run -o yaml > web.yaml
```

然后以yaml配置文件部署应用

```bash
kubectl apply -f web.yaml
```

然后以下命令可以看到发布的应用状态，`STATUS`为`Running`时就是正常发布了

```shell
kubectl get pods
```

此时还只能在集群内部访问发布的服务，所以还需要对外暴露这个服务

```bash
kubectl expose deployment web --port=80 --type=NodePort --target-port=80 --name=web-service -o yaml > web-service.yaml
```

再发布这个`Service`，修改 web-service.yaml 文件的`nodePort`字段，可以修改对外暴露的端口号

```bash
kubectl apply -f web-service.yaml
```

然后查看暴露出来的端口，用集群任意一个节点的ip加此端口都可以访问到发布的服务

```
kubectl get pods,svc
```

### 升级

上一步使用的 nginx 版本是 1.14，现在对它进行升级，k8s会对所有服务实例逐个滚动升级，服务不会中断

```
kubectl set image deployment web nginx=nginx:1.15
```

用以下命令查看升级的状态

```bash
kubectl rollout status deployment web
```

### 回滚

查看历史版本

```
kubectl rollout history deployment web
```

回退到上一个版本

```bash
kubectl rollout undo deployment web
```

查看回滚状态

```bash
kubectl rollout status deployment web
```

查看历史版本时，可以看到版本号，可以回滚到指定版本

```bash
kubectl rollout undo deployment web --to-revision=2
```

### 弹性伸缩

修改服务数量

```bash
kubectl scale deployment web --replicas=10
```

查看运行状态

```bash
kubectl get pods
```

