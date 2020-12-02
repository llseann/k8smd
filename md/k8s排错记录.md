## 使用kubeadmin部署k8s，出现其中一个节点长时间未NotReady状态

情景：

```
[root@master ~]# kubectl get nodes 
NAME      STATUS     ROLES     AGE       VERSION
master    Ready      master    23h       v1.11.1
node01    NotReady   <none>    23h       v1.11.1
node02    Ready      <none>    23h       v1.11.1
```

node01长时间处于`NotReady`状态，经过多次`kubeadm reset`重置操作后还是处于未准备状态，遂开始了排查过程。

在master节点执行

`kubectl get pod -n kube-system`

```
[root@master ~]# kubectl get pod -n kube-system
NAME                             READY     STATUS                  RESTARTS   AGE
coredns-78fcdf6894-544rg         0/1       CrashLoopBackOff        17         22h
coredns-78fcdf6894-rt9z2         0/1       CrashLoopBackOff        17         22h
etcd-master                      1/1       Running                 0          22h
kube-apiserver-master            1/1       Running                 0          22h
kube-controller-manager-master   1/1       Running                 0          22h
kube-flannel-ds-7g9hl            1/1       Running                 0          22h
kube-flannel-ds-9ljnx            1/1       Running                 0          22h
kube-flannel-ds-csfhk            0/1       Init:ImagePullBackOff   0          22h
kube-proxy-bgw95                 1/1       Running                 0          22h
kube-proxy-czd9x                 1/1       Running                 0          22h
kube-proxy-r6msl                 1/1       Running                 0          22h
kube-scheduler-master            1/1       Running                 0          22h
```

可以看到`kube-flannel-ds-csfhk            0/1       Init:ImagePullBackOff   0          22h`

状态为`Init:ImagePullBackOff`网上查了下，这种问题一般是因为缺少系统必须的镜象导致的。遂执行了如下的检查：

`[root@master ~]# kubectl describe pod kube-flannel-ds-csfhk -n kube-system`

```
Name:               kube-flannel-ds-csfhk
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               node01/192.168.10.11
...省略...
Events:
  Type     Reason   Age                 From             Message
  ----     ------   ----                ----             -------
  Warning  Failed   22h (x27 over 22h)  kubelet, node01  Error: ImagePullBackOff
  Warning  Failed   22h (x7 over 22h)   kubelet, node01  Failed to pull image "quay.io/coreos/flannel:v0.10.0-amd64": rpc error: code = Canceled desc = context canceled
  Normal   Pulling  10m (x4 over 15m)   kubelet, node01  pulling image "quay.io/coreos/flannel:v0.10.0-amd64"
  Warning  Failed   9m (x4 over 14m)    kubelet, node01  Failed to pull image "quay.io/coreos/flannel:v0.10.0-amd64": rpc error: code = Canceled desc = context canceled
  Warning  Failed   9m (x4 over 14m)    kubelet, node01  Error: ErrImagePull
  Warning  Failed   5m (x13 over 14m)   kubelet, node01  Error: ImagePullBackOff
  Normal   BackOff  42s (x29 over 14m)  kubelet, node01  Back-off pulling image "quay.io/coreos/flannel:v0.10.0-amd64"
```

在末尾可以很明确的看到在node01缺少镜像`quay.io/coreos/flannel:v0.10.0-amd64`，ssh至node01检查，确实缺少该image。

处理操作：

尝试直接pull该镜像，失败告终，后面是通过在其他节点拷贝了该镜像。操作如下：

master节点：

```
[root@master ~]# docker save quay.io/coreos/flannel:v0.10.0-amd64 > flannel.tar
[root@master ~]# scp flannel.tar node01:/root/
```

node01节点：

```
[root@node01 ~]# docker load -i flannel.tar
b883fd48bb96: Loading layer [==================================================>]  5.12 kB/5.12 kB
Loaded image: quay.io/coreos/flannel:v0.10.0-amd64
```

过了一会，在master查看了`kubectl get nodes`

```
[root@master ~]# kubectl get nodes 
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    23h       v1.11.1
node01    Ready     <none>    23h       v1.11.1
node02    Ready     <none>    23h       v1.11.1
```

至此，问题完结。

其实仔细查看 会发现其实还有一个问题，`coredns-78fcdf6894-544rg`状态也不对，使用描述查看该容器：
```
[root@master ~]# kubectl describe pod coredns-78fcdf6894-544rg -n kube-system 
Name:               coredns-78fcdf6894-544rg
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               master/192.168.10.10
Start Time:         Wed, 03 Jun 2020 13:02:53 -0400
Labels:             k8s-app=kube-dns
                    pod-template-hash=3497892450
Annotations:        <none>
Status:             Running
IP:                 10.244.0.3
Controlled By:      ReplicaSet/coredns-78fcdf6894
Containers:
....
Events:
  Type     Reason   Age                 From             Message
  ----     ------   ----                ----             -------
  Warning  BackOff  1m (x1096 over 6d)  kubelet, master  Back-off restarting failed container
```
没有发现什么有用的信息，遂查看日志：
```
[root@master ~]# kubectl logs -f coredns-78fcdf6894-544rg -n kube-system
standard_init_linux.go:178: exec user process caused "operation not permitted"
```
这个问题一般是因为selinux导致的， 需要关闭系统的该参数，我这里出现问题是因为我改了`/etc/selinux/config`但是并没重启，而是用的`setenforce 0`出现的问题，重启所有机器后恢复正常，生产环境谨慎操作。
```
[root@master ~]# kubectl get pod -n kube-system
NAME                             READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-544rg         1/1       Running   55         6d
coredns-78fcdf6894-rt9z2         1/1       Running   55         6d
etcd-master                      1/1       Running   3          6d
kube-apiserver-master            1/1       Running   3          6d
kube-controller-manager-master   1/1       Running   3          6d
kube-flannel-ds-7g9hl            1/1       Running   1          6d
kube-flannel-ds-9ljnx            1/1       Running   3          6d
kube-flannel-ds-csfhk            1/1       Running   1          6d
kube-proxy-bgw95                 1/1       Running   1          6d
kube-proxy-czd9x                 1/1       Running   1          6d
kube-proxy-r6msl                 1/1       Running   3          6d
kube-scheduler-master            1/1       Running   3          6d
```
至此，环境问题已经解决完毕！