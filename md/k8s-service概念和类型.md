## service

### 1. 存在意义：

1、为了防止Pod失联，pod注册到service层，service提供服务发现，当实际中pod会多次变动，导致的ip不一致的情况下，倘若没有service  ，会找不到所需要的的pod提供服务。

2、定义一组访问策略，充当负载均衡的关系



实际是pod和service进行联系和绑定是通过selector和labtes，如下yaml

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:1.9
        name: nginx
        resources:
          requests:
            cpu: "50m"
            memory: "128Mi"
```



### 2. 常用的service类型

常用的有三种，分别为`ClusterIP`,` NodePort`,` LoadBalancer`

- ClusterIP：集群内部使用 (默认规则)

- NodePort：对外访问应用使用

- LoadBalancer：对外访问应用使用，公有云操作