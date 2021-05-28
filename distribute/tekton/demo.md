

先安装 tekton

```bash
kubectl apply -f git-resource.yaml
```



```bash
kubectl create secret docker-registry dockerhub --docker-server=https://index.docker.io/v1/ --docker-username=<USERNAME> --docker-password=<PASSWORD>
```

```bash
kubectl apply -f serviceaccount.yaml
```

```bash
kubectl apply -f source-to-image.yaml
```

```bash
kubectl apply -f deploy-to-k8s.yaml
```

```bash
kubectl apply -f build-pipeline.yaml
```

```bash
kubectl apply -f run.yaml
```

查看日志：

```bash
kubectl describe pod generic-pipeline-run-source-to-image
tkn pipelinerun logs generic-pipeline-run -f
```

