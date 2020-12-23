## deployment概念

## 1、创建

### 1.1、命令行操作

```
kubectl get deployments									#查看所有的deployments
kubectl run nginx --image=nginx --dry-run -o yaml		#生成配置文件模板
```

`kubectl run nginx --image=nginx --replicas=5`创建nginx的容器，设置为5个副本

```
[root@master ~]# kubectl get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     5         5         5            5           10m
[root@master ~]# kubectl get pods
NAME                     READY     STATUS                       RESTARTS   AGE
nginx-64f497f8fd-5fk2b   1/1       Running                      0          11m
nginx-64f497f8fd-fl7v8   1/1       Running                      0          11m
nginx-64f497f8fd-gbd9f   1/1       Running                      0          11m
nginx-64f497f8fd-hrwtj   1/1       Running                      0          11m
nginx-64f497f8fd-z9qf4   1/1       Running                      0          11m
```

可以看到副本数为5,下面进行修改副本数

`kubectl edit deployments. nginx`

```
replicas: 5			#将其修改成6,保存后退出
```

通过`kubectl get pods`可以看到副本数已经变成6了

```
[root@master ~]# kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-64f497f8fd-5fk2b   1/1       Running   0          16m
nginx-64f497f8fd-8q7p4   1/1       Running   0          1m
nginx-64f497f8fd-fl7v8   1/1       Running   0          16m
nginx-64f497f8fd-gbd9f   1/1       Running   0          16m
nginx-64f497f8fd-hrwtj   1/1       Running   0          16m
nginx-64f497f8fd-z9qf4   1/1       Running   0          16m
```

`kubectl delete deployments. nginx`删除deployments

### 1.2、用yaml文件创建

`kubectl run nginx --image=nginx --dry-run -o yaml >> deployments1.yaml`

修改deployments1.yaml文件中不需要的部分

`kubectl apply -f deployments1.yaml`根据yaml启动deployment

```
[root@master deployments]# kubectl get pods
NAME                     READY     STATUS              RESTARTS   AGE
nginx-64f497f8fd-2wcdn   0/1       ContainerCreating   0          8s
nginx-64f497f8fd-br2g5   0/1       ContainerCreating   0          8s
```

继续修改`deployments1.yaml`副本数为5，再次执行`kubectl apply -f deployments1.yaml`

```
[root@master deployments]# kubectl get pods
NAME                     READY     STATUS              RESTARTS   AGE
nginx-64f497f8fd-2wcdn   1/1       Running             0          1m
nginx-64f497f8fd-br2g5   1/1       Running             0          1m
nginx-64f497f8fd-kjpwv   0/1       ContainerCreating   0          6s
nginx-64f497f8fd-rgt4l   0/1       ContainerCreating   0          6s
nginx-64f497f8fd-xffx4   0/1       ContainerCreating   0          6s
```

可以看到副本数是可以进行动态修改的。

小结：修改副本数有3种方法

1. `kubectl scale deployment nginx --replicas=10 `
2. `kubectl edit deployments. nginx`
3. 修改yaml文件后,`kubectl apply -f deployments1.yaml`即可

## 2、升级

```
[root@master deployments]# docker images | grep nginx
docker.io/nginx                                                  latest              bc9a0695f571        14 hours ago        133 MB
docker.io/nginx                                                  1.9                 c8c29d842c09        4 years ago         183 MB
docker.io/nginx                                                  1.7.9               84581e99d807        5 years ago         91.7 MB
[root@master deployments]# kubectl get deployments. -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES    SELECTOR
nginx     3         3         3            3           21h       nginx        nginx     run=nginx
```

可以看到本机存在三个nginx镜像，版本分别为`1.7.9`、`1.9`、`latset`，当前正在运行的deployment为latest版本，倘若现在因为业务需要，需要进行镜像的迭代，变更ng的版本。可以使用以下的方式进行版本变更

### 2.1、升级方式

**方式一：直接修改deployment的配置**

`kubectl edit deployment. nginx`

修改

```
spec:
      containers:
      - image: nginx
```

为

```
spec:
      containers:
      - image: nginx:1.7.9
```

保存退出，此时查看deployment可以看到版本已经被修改

```
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES        SELECTOR
nginx     5         7         3            4           5m        nginx        nginx:1.7.9   run=nginx
```

**方式二：通过修改yaml配置文件**

```
spec:
      containers:
      - image: nginx
        name: nginx
```

修改为

```
spec:
      containers:
      - image: nginx:1.9
        name: nginx
```

执行`kubectl apply -f deployments1.yaml`

```
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES      SELECTOR
nginx     5         7         3            4           8m        nginx        nginx:1.9   run=nginx
```

可以看到版本已经得到了修改

**方式三：通过命令行进行set**

`kubectl set image deployment/nginx nginx=nginx:1.9 <--record> `

```
设置deployment中的nginx中的镜像名nginx。变更为nginx:1.9
<--record>为可选项，作用在于保存操作历史在history里面，后文会进行演示
```

### 2.2、 参数解析

在滚动升级的过程中，有两个比较重要的参数

## 3、回滚

`kubectl rollout undo deployment nginx `：使得名为nginx的deployment回滚到上一个版本

`kubectl rollout history deployment/nginx `：查看变更历史,默认效果如下

```
deployments "nginx"
REVISION  CHANGE-CAUSE
1         <none>
4         <none>
5         <none>
```

配合前文中的`--record`使用，可以记录下变更历史

```
[root@master deployments]# kubectl set image deployment/nginx nginx=nginx:1.7.9 --record
deployment.extensions/nginx image updated
[root@master deployments]# kubectl set image deployment/nginx nginx=nginx --record
deployment.extensions/nginx image updated
[root@master deployments]# kubectl rollout history deployment/nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
5         <none>
6         kubectl set image deployment/nginx nginx=nginx:1.7.9 --record=true
7         kubectl set image deployment/nginx nginx=nginx --record=true
```

`kubectl rollout undo deployment/nginx --to-revision=6`：使得deployment回退到指定版本6上

## 4、HPA水平自动伸缩 

概念：通过检测pod CPU的负载， 解决deployment里某pod负载太重， 动态伸缩pod的数量来负载均衡 

![1606925880872](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606925880872.png)

使用：

```
[root@master ~]# kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
horizontalpodautoscaler.autoscaling/nginx autoscaled
参数解析：
--min=2 ：最少的副本数为2
--max=10 ：最多的副本数为10
--cpu-percent=80 ：cpu使用占比超过80%会发生扩容行为
```

但是实际使用中会出现一个问题，如下:

```
[root@master ~]# kubectl get hpa
NAME      REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx     Deployment/nginx   <unknown>/80%   2         10        0          15s
```

cpu占用会显示为`<unknown>`暂未解决，后续再进行讨论

进去某个pod ，执行`cat /dev/zero > /dev/null & `

观察pod的数目变化，及hpa的cpu使用量 

在物理机里killall -9 cat ，查看pod数量