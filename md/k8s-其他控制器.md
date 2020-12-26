## Controller控制器-有状态

### 1、概念：有状态和无状态

无状态：(Deploment)：

- 认为Pod都是相同的
- 没有顺序要求
- 不考虑应用在哪个node运行
- 可以随意进行伸缩和扩展

有状态：

- 需要考虑无状态下的所有条件
- 让每个Pod都是独立的，保持Pod的启动顺序和唯一性
- 唯一的网络标识符，持久存储
- 有固定的启动顺序，例如MySQL主从

### 2、部署有状态应用-StatefulSet

准备：无头service(ClusterIP的IP一栏为none)

```
#生成无头service
kubectl create service clusterip nginx --clusterip="None" --dry-run -o yaml > tst.yaml
```

将其修改为：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: default
spec:
  serviceName: nginx
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

创建pod，可以查看到

```
[root@master stateful]# kubectl get pod
NAME                  READY     STATUS    RESTARTS   AGE
nginx-statefulset-0   1/1       Running   0          7m
nginx-statefulset-1   1/1       Running   0          7m
nginx-statefulset-2   1/1       Running   0          6m
```

每个pod的名字都是唯一的

Deployment和Statefulset区别：有身份的(可被唯一标识)

固定的域名规则：

```
#格式：
主机名.services名字.名称空间.svc.cluster.local
#此处应为：
nginx-statefulset-0.nginx.defaule.svc.cluster.local
```

### 3、部署守护进程-DeamonSet

含义：在每个node运行一个相同的pod，新加入集群的node也会默认加上该Pod，常用于监控的agent或者日志采集等场景

ds.yaml

假装下面的yaml在进行日志采集

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-test 
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 8
```

根据yaml配置文件启动pod后，可以看到如下：

```
[root@master stateful]# kubectl get pod
NAME            READY     STATUS              RESTARTS   AGE
ds-test-d7g4r   0/1       ContainerCreating   0          11s
ds-test-hjb2m   0/1       ContainerCreating   0          11s
[root@master stateful]# kubectl get pod -o wide
NAME            READY     STATUS    RESTARTS   AGE       IP            NODE
ds-test-d7g4r   1/1       Running   0          22s       10.244.1.60   node01
ds-test-hjb2m   1/1       Running   0          22s       10.244.2.58   node02
```

因为此处使用的三节点集群，主节点的污点未去除，按照原理来说，在去除主节点污点后，主节点也会被调度上一个名为`ds-test-xxxx`的Pod

此时查看master节点

```
[root@master stateful]# kubectl describe node master
Name:               master
Roles:              master
...
Taints:             node-role.kubernetes.io/master:NoSchedule
...
[root@master stateful]# kubectl taint node master node-role.kubernetes.io/master:NoSchedule-
node/master untainted
```

去除污点后查看pod

```
[root@master stateful]# kubectl get pod -o wide
NAME            READY     STATUS    RESTARTS   AGE       IP            NODE
ds-test-d7g4r   1/1       Running   0          6m        10.244.1.60   node01
ds-test-hjb2m   1/1       Running   0          6m        10.244.2.58   node02
ds-test-lxb27   1/1       Running   0          41s       10.244.0.31   master
```

正如预期

### 4、部署一次性任务和定时任务

一次性任务：

适用场景：任务只需要被调度一次，如某些初始化场景

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

```
[root@master stateful]# kubectl get pod -o wide
NAME            READY     STATUS      RESTARTS   AGE       IP            NODE
ds-test-d7g4r   1/1       Running     0          1h        10.244.1.60   node01
ds-test-hjb2m   1/1       Running     0          1h        10.244.2.58   node02
ds-test-lxb27   1/1       Running     0          1h        10.244.0.31   master
pi-wkbgk        0/1       Completed   0          1m        10.244.2.59   node02
```

状态为`Completed`表示已经完成，且只会完成一次

```
[root@master stateful]# kubectl get jobs
NAME      DESIRED   SUCCESSFUL   AGE
pi        1         1            1m
```

定时任务：

适用场景，需要定期导数据或者定期截取日志

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```
[root@master stateful]# kubectl get pod
\NAME                     READY     STATUS      RESTARTS   AGE
hello-1608913740-pb2zb   0/1       Completed   0          27s
[root@master stateful]# kubectl get pod
NAME                     READY     STATUS              RESTARTS   AGE
hello-1608913740-pb2zb   0/1       Completed           0          2m
hello-1608913800-jd2g4   0/1       Completed           0          1m
hello-1608913860-mdccj   0/1       ContainerCreating   0          4s
```

效果为会在每分钟进行一次运行，具体控制为cron表达式

删除cronjob后。对应的pod也会被清除