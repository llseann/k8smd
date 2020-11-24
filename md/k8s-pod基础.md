## POD基础

k8s不直接操控容器，而是通过管理pod进行间接操控。由pod去管理容器，pod里面至少运行一个容器，理想状态也是运行一个容器，方便进行管理和维护。实际情况中，客户端和pod通信，pod通过内部网络和容器进行通信。

### 创建方式

主要通过YAML文件方式来创建如下示例：

```yaml
apiVersion: v1 		    #kubectl api-versions查看所有api-version
kind: Pod	 			#定义种类,services还是job
metadata: 
  name: static-web 		#pod的名字,可以自定义
  labels: 
    role: myrole 		#标签
spec: 					#定义容器
  containers: 
    - name: myapp-container		#第一个容器的名字
      image: busybox			#容器使用的镜像
      command: ['sh', '-c', 'echo OK!&& sleep 6000000000']		#相当于dockerfile里面的CMD

    - name: web 				#第二个容器的名字
      image: nginx 				
      ports: 
        - name: web 
          containerPort: 80 	#容器的端口
          protocol: TCP			#协议种类
```

常用命令：

```shell
kubectl get pods					#获取所有pods
kubectl get nods					#获取所有节点
kubectl get pods --show-labels		#获取所有pods以及展示标签
kubectl get pods -o wide --show-labels		#获取所有pods、标签、以及容器运行的node位置
kubectl run nginx --image=nginx		#创建一个获取deployment
kubectl get deployment.				#获取deployment数量信息,后面有个点
kubectl delete deployment. nginx	#删除名为nginx的deployment
kubectl run nginx --image=nginx --dry-run	#模拟运行,实际不会运行
kubectl describe pods static-web	#查看某个pods的具体信息
kubectl run nginx --image=nginx --dry-run -o yaml		#查看yaml模板
```

创建pods：

```shell
[root@k8syum01 pods]# kubectl create
...
Examples:
  # Create a pod using the data in pod.json.
  kubectl create -f ./pod.json
  
  # Create a pod based on the JSON passed into stdin.
  cat pod.json | kubectl create -f -
  
  # Edit the data in docker-registry.yaml in JSON using the v1 API format then create the resource using the edited
data.
  kubectl create -f docker-registry.yaml --edit --output-version=v1 -o json
  ...省略
#####################################分割线############################################
[root@k8syum01 pods]# kubectl apply
error: You must provide one or more resources by argument or filename.
Example resource specifications include:
   '-f rsrc.yaml'
   '--filename=rsrc.json'
   'pods my-pod'
   'services'
#####################################分割线############################################
kubectl create -f pod1.yaml		#指定yaml文件创建pods
kubectl apply -f pod1.yaml		#创建pod、区别create在于apply可以动态更新,更新完后重复执行即可
```

删除pods：

```shell
[root@k8syum01 pods]# kubectl delete
error: You must provide one or more resources by argument or filename.Example resource specifications include:
   '-f rsrc.yaml'
   '--filename=rsrc.json'
   'pods my-pod'
   'services'
#####################################分割线############################################
kubectl delete -f pod1.yaml			#指定yaml文件删除pods
kubectl delete pods static-web		#指定pods名删除pod
```

从官方镜像获取yaml模板：

```shell
kubectl run nginx --image=nginx --dry-run -o yaml	#模拟运行并输出其yaml文件,使用重定向写入文件
```

### POD的操作

```shell
kubectl exec static-web ls /		#在static-web中执行某个cmd
kubectl exec -it pod bash 			#如果pod里有多个容器，则命令是在第一个容器里执行,yaml定义的第一个容器里面执行
kubectl describe pod/s static-web	#获取名为static-web的相关信息,有无s不影响
kubectl logs static-web				#获取名为static-web的日志信息
```

<font color=red>**pod是k8s的最基本调度单位，pod里面可以有多个容器，并且pod是可以分布在不同机器上面，但是同一个pod的多个容器只能在同一个节点运行，一般情况下，一个pod只创建一个容器！**</font>

### POD相关

镜像下载策略：

```
containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
#####################################分割线############################################
Always 每次都下载最新镜像 
Never 只使用本地镜像，从不下载 
IfNotPresent 本地没有才下载
```

### POD的生命周期

CMD的三种形式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
#####################################分割线############################################
#以下三种写法等价
spec:
  containers: 
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo OK! && sleep 60']
#####################################分割线############################################
spec:
  containers: 
  - name: myapp-container
    image: busybox
    command: 
    - sh
    - -c
    - echo OK! && sleep 60
#####################################分割线############################################
spec:
  containers: 
  - name: myapp-container
    image: busybox
    args: 
    - sh
    - -c
    - echo OK! && sleep 60
```

POD中使用变量：即传入环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: env-demo
    image: nginx
    env:					#参数等同于docker启动时的-e参数，将env中设置的变量传入容器
    - name: DEMOx
      value: "Hello x sir"
    - name: DEMOy
      value: "Hello y sir"
#####################################分割线############################################
spec:
  containers:
  - name: env-demo
    image: nginx
    env:
    - name: DEMOx
      value: "Hello x sir"
    command: ["/bin/echo"]
    args: ["$(DEMOx)"]
```

POD的重启策略

```yaml
spec:
  restartPolicy: Never			#三种模式
  containers: 
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo OK! && sleep 5']
#####################################分割线############################################
重启模式三种策略：
Always 			总是重启,不论运行状态
OnFailure 		失败了才重启
Never 			从不重启
#####################################分割线############################################
tips：pod中有多个容器，只要有一个容器有报错，则整体状态会显示报错，那么整体都会根据重启策略进行操作，倘若为always，，并且会一直进行重试。如果一个一直运行，一个正常退出，则整体显示为running
```

容器状态：

```
Pending 		#pod已经建立，但是pod里还有容器没有创建完成
Running 		#pod已经被调度到节点上，且容器工作正常
Completed 		#pod里所有容器正常退出
```

### 初始化容器

```
1、它们总是运行到完成。
2、每个都必须在下一个启动之前成功完成。
3、如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的
restartPolicy 为 Never，它不会重新启动。
4、Init 容器支持应用容器的全部字段和特性，但不支持 Readiness Probe，因为它们必须在 Pod 就绪之前运行完
成。
5、如果为一个 Pod 指定了多个 Init 容器，那些容器会按顺序一次运行一个。 每个 Init 容器必须运行成功，下一个
才能够运行。
6、因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的代码应该是幂等的。 特别地，被写到
EmptyDirs 中文件的代码，应该对输出文件可能已经存在做好准备。
7、在 Pod 上使用 activeDeadlineSeconds，在容器上使用 livenessProbe，这样能够避免 Init 容器一直失败。 这就
为 Init 容器活跃设置了一个期限。
8、在 Pod 中的每个 app 和 Init 容器的名称必须唯一；与任何其它容器共享同一个名称，会在验证时抛出错误。
9、对 Init 容器 spec 的修改，被限制在容器 image 字段中。 更改 Init 容器的 image 字段，等价于重启该 Pod。
```

### 静态pod

所谓静态pod就是，不在master上创建的，而是需要到Node的/etc/kubelet.d/里创建一个yaml文件，然后根据这个yaml文件，创建一个pod，这样创建出来的node，是不会接受master的管理的。当然，要是想创建静态pod的话，需要对node的kubelet配置文件进行一些设置。

步骤：

```
[root@k8syum01 pods]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet Server
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2020-08-13 12:11:56 EDT; 1h 45min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1626 (kubelet)
   Memory: 65.7M
   CGroup: /system.slice/kubelet.service
           ├─1626 /usr/bin/kubelet --logtostderr=true --v=0 --api-servers=http://192.168.10.21:8080 --address=0.0.0.0 --por...
           └─1706 journalctl -k -f
#####################################分割线############################################
修改/etc/kubernetes/kubelet
在KUBELET_ARGS中添加：--pod-manifest-path=/etc/kubelet.d/		#路径可自定义
```

### pod的调度策略

```
[root@k8syum01 ~]# kubectl run nginx --image=nginx --replicas=5			#设置5个副本
[root@k8syum01 ~]# kubectl get pods -o wide								#查看副本位置
[root@k8syum01 ~]# kubectl get pods -o wide
NAME                       READY     STATUS              RESTARTS   AGE       IP           NODE
nginx-701339712-dn6lg      0/1       ContainerCreating   0          1m        <none>       192.168.10.22
nginx-701339712-f4dpz      1/1       Running             0          51s       172.17.0.3   192.168.10.21
nginx-701339712-fn6fr      0/1       ContainerCreating   0          1m        <none>       192.168.10.22
nginx-701339712-nb414      0/1       ContainerCreating   0          1m        <none>       192.168.10.22
nginx-701339712-txcsw      1/1       Running             0          1m        172.17.0.4   192.168.10.21
static-web-192.168.10.21   1/1       Running             1          3d        172.17.0.2   192.168.10.21
```

**调度的三个对象：**

- 调度的对象
- 调度算法
  - 主机过滤
  - 主机打分
- 调度策略

> 主机过滤：

```
NoDiskConflict
PodFitsResources
PodFitsPorts
MatchNodeSelector
HostName
NoVolumeZoneConflict
PodToleratesNodeTaints
CheckNodeMemoryPressure
CheckNodeDiskPressure
MaxEBSVolumeCount
MaxGCEPDVolumeCount
MaxAzureDiskVolumeCount
MatchInterPodAffinity
GeneralPredicates
NodeVolumeNodeConflict
```

> 主机打分

```
LeastRequestedPriority
公式
score=cpu((capacity-sum (requested))*10/capacity)+memory((capacity-sum(requested))*10/capacity)/2

BalanceResourceAllocation
公式
score = 10 -abs ( cpuFraction - memoryFraction ) * 10

CalculateSpreadPriority
公式
Score = 10 * （（maxCount -counts）/ （maxCount））
```

**为pod指定运行位置:**

场景：倘若机器中有一批硬盘为SSD，通过标签(label)进行区分，选择器(selector)选择哪个label，不管是pod还是node都会有label，并且可定义

```shell
[root@k8syum01 ~]# kubectl get nodes --show-labels			#显示label
NAME            STATUS    AGE       LABELS
192.168.10.21   Ready     20d       beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.21
192.168.10.22   Ready     20d       beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.22
[root@k8syum01 ~]# kubectl get nodes --selector=kubernetes.io/hostname=192.168.10.22
#使用选择器选择kubernetes.io/hostname=192.168.10.22的node
NAME            STATUS    AGE
192.168.10.22   Ready     20d
#同理，pod也只支持以上用法
#####################################分割线############################################
[root@k8syum01 ~]# kubectl label node 192.168.10.21 disktype=ssd	#为某节点设置label
node "192.168.10.21" labeled
[root@k8syum01 ~]# kubectl get nodes --show-labels
NAME            STATUS    AGE       LABELS
192.168.10.21   Ready     20d       beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/hostname=192.168.10.21
192.168.10.22   Ready     20d       beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.22
[root@k8syum01 ~]# kubectl get nodes --selector=disktype=ssd		#通过选择器找对应node
NAME            STATUS    AGE
192.168.10.21   Ready     20d
[root@k8syum01 ~]# kubectl label node 192.168.10.21 disktype-		#删除某个label,减号即可
node "192.168.10.21" labeled
```

**如何指定pod运行位置：**

```shell
[root@k8syum01 pods]# vim nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  nodeSelector:			#设置选择器,与容器标识对其
    disktype: ssd		#指定选择器的值,如果没有pod与之匹配,则pod启动后将一直处于Pending状态
  containers:
  - image: nginx
    name: nginx
[root@k8syum01 pods]# kubectl get pods -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP           NODE
nginx     1/1       Running   0          3s        172.17.0.2   192.168.10.21
```

**设置警戒线cordon**

场景:某节点进行维护,新产生的pod不被调度到维护节点上

```shell
[root@k8syum01 pods]# kubectl cordon 192.168.10.22			#设置警戒线
[root@k8syum01 pods]# kubectl get nodes
NAME            STATUS                     AGE
192.168.10.21   Ready                      20d
192.168.10.22   Ready,SchedulingDisabled   20d
#如果一个node被标记为cordon，新创建的pod不会被调度到此node上
#已经调度上去的还在运行，需要被删除让其重新生成
#如果yaml指定了节点选择器选择了设置了cordon的节点,创建的pod会一直处于Pending状态
[root@k8syum01 pods]# kubectl uncordon 192.168.10.22		#撤销警戒线
```

**节点的drain**

```shell
[root@k8syum01 pods]# kubectl drain 192.168.10.22		#设置22节点为drain状态
node "192.168.10.22" cordoned
pod "nginx-701339712-hpcq6" evicted			#被驱逐到可用节点上
pod "nginx-701339712-jlmzg" evicted			#被驱逐到可用节点上
pod "nginx-701339712-72fl3" evicted		
pod "nginx-701339712-vshg0" evicted
pod "nginx-701339712-44mvt" evicted
node "192.168.10.22" drained
#倘若使用的kubeadmin部署的集群,使用该命令会报错,因为该方式为容器部署的方式
#有很多系统服务以pod方式运行,这些pod不能被驱逐,需要将其忽略才能将节点设置为drain状态,使用下面方法即可
[root@k8syum01 pods]# kubectl drain 192.168.10.22  --ignore-daemonsets
[root@k8syum01 pods]# kubectl get nodes
NAME            STATUS                     AGE
192.168.10.21   Ready                      20d
192.168.10.22   Ready,SchedulingDisabled   20d
```

> drain本质还是cordon,只不过加上了驱逐步骤

**节点的taint**

```shell
[root@k8syum01 pods]# kubectl taint nodes 192.168.10.22 like=gakki:NoSchedule
#为节点设置taint,like=gakki都可以自定义,NoSchedule为系统字段,不能改变
[root@k8syum01 pods]# kubectl describe nodes 192.168.10.22 | grep -A1 Taint
Taints:			like=gakki:NoSchedule
CreationTimestamp:	Tue, 28 Jul 2020 12:10:05 -0400
#只有标签为like=gakki的pod才能调度到此node，否则不能
```

```
[root@k8syum01 pods]# vim nginxTaint.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  tolerations:
  - key: "like"
    operator: "Equal"
    value: "gakki"
    effect: "NoSchedule"
#这样才能指定到固定的节点运行
```

