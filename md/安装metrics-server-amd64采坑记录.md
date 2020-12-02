问题描述：在使用kubectl top指令时，提示

```Error from server (NotFound): the server could not find the requested resource (get services h```

原因是因为没有安装`metrics-server-amd64`

```
git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git
kubectl create -f kubernetes-metrics-server/
```

clone`kubernetes-metrics-server`项目到本地，使用kucectl启动，

通过`kubectl get pods -n kube-system`可以看到，metrics长时间处于`ImagePullBackOff`状态

```
metrics-server-7c7f858d99-v8s6j   0/1       ImagePullBackOff   0          15m
```

然后使用`kubectl describe pod metrics-server-7c7f858d99-v8s6j -n kube-system`

```
Events:
  Type     Reason          Age                From               Message
  ----     ------          ----               ----               -------
  Normal   Scheduled       2m                 default-scheduler  Successfully assigned kube-system/metrics-server-7c7f858d99-v8s6j to node02
  Normal   SandboxChanged  1m                 kubelet, node02    Pod sandbox changed, it will be killed and re-created.
  Normal   Pulling         1m (x2 over 1m)    kubelet, node02    pulling image "k8s.gcr.io/metrics-server-amd64:v0.3.1"
  Warning  Failed          29s (x2 over 1m)   kubelet, node02    Failed to pull image "k8s.gcr.io/metrics-server-amd64:v0.3.1": rpc error: code = Unknown desc = Get https://k8s.gcr.io/v1/_ping: dial tcp 64.233.189.82:443: connect: connection refused
  Warning  Failed          29s (x2 over 1m)   kubelet, node02    Error: ErrImagePull
  Normal   BackOff         14s (x2 over 28s)  kubelet, node02    Back-off pulling image "k8s.gcr.io/metrics-server-amd64:v0.3.1"
  Warning  Failed          14s (x2 over 28s)  kubelet, node02    Error: ImagePullBackOff
```

从上可以很明显的看到错误，原因是因为缺少`k8s.gcr.io/metrics-server-amd64:v0.3.1`镜像，且由于某些原因，无法pull到他我使用了如下的解决办法

```
docker pull xxlaila/metrics-server-amd64:v0.3.1
docker tag docker.io/xxlaila/metrics-server-amd64:v0.3.1 k8s.gcr.io/metrics-server-amd64:v0.3.1
```

从docker hub拉取到该镜像，重新打tag，将该镜像分发到所有节点

执行`kubectl replace --force -f kubernetes-metrics-server/`使得pod强制从配置文件进行更新

经过一段时间，发现报错依旧。

在一帖子中看到解决办法

```
vim kubernetes-metrics-server/metrics-server-deployment.yaml
修改为imagePullPolicy: IfNotPresent
保存退出
```

重新更新pod，问题解决

```
[root@master ~]# kubectl get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-pf4tf          1/1       Running   5          104d
coredns-78fcdf6894-wpbnh          1/1       Running   5          104d
etcd-master                       1/1       Running   5          104d
kube-apiserver-master             1/1       Running   5          104d
kube-controller-manager-master    1/1       Running   5          104d
kube-flannel-ds-kvwbg             1/1       Running   5          104d
kube-flannel-ds-ls4vk             1/1       Running   5          104d
kube-flannel-ds-xf4vv             1/1       Running   5          104d
kube-proxy-98dj7                  1/1       Running   5          104d
kube-proxy-g4mw8                  1/1       Running   5          104d
kube-proxy-sm4tl                  1/1       Running   5          104d
kube-scheduler-master             1/1       Running   5          104d
metrics-server-76f7d5dbf7-gd9fg   1/1       Running   0          13m
```

可以愉快的使用`kubectl top pod nginx-7dbc7585b7-ccvrv`