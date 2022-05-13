## Nginx运行在K8S集群中

### 登录Nginx

1. pod

```shell
kubectl get po -A
```

2. exec

```shell
kubectl exec -it nginx-proxy-deployment-595123458b-f8wea -n default sh
```

### Nginx资源

1. CPU

#### 如果不指定 CPU 限制

如果你没有为容器指定 CPU 限制，则会发生以下情况之一：

* 容器在可以使用的 CPU 资源上没有上限。因而可以使用所在节点上所有的可用 CPU 资源。
* 容器在具有默认 CPU 限制的名字空间中运行，系统会自动为容器设置默认限制。 集群管理员可以使用 LimitRange 指定 CPU 限制的默认值。
