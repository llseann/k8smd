## 节点选择器

背景：实际中常常会出现需要把某些Pod部署到指定机器运行，此处记录三种节点选择器的使用



DaemonSet会忽略Node的unschedulable状态，有两种方式来指定Pod只运行在指定的Node节点上：

- nodeSelector：只调度到匹配指定label的Node上
- nodeAffinity：功能更丰富的Node选择器，比如支持集合操作
- podAffinity：调度到满足条件的Pod所在的Node上

### 1、nodeSelector

给node01/02都打上disktype的标签

```
[root@master stateful]# kubectl label nodes node01 disktype=ssd
[root@master stateful]# kubectl label nodes node02 disktype=ssd
[root@master stateful]# kubectl get nodes --show-labels
NAME      STATUS    ROLES     AGE       VERSION   LABELS
master    Ready     master    127d      v1.11.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=master,node-role.kubernetes.io/master=
node01    Ready     <none>    127d      v1.11.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/hostname=node01
node02    Ready     <none>    127d      v1.11.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/hostname=node02
```

修改的ds.yaml

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
      nodeSelector:				#加上节点选择器
        disktype: ssd			#指明标签
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 80
```

```
[root@master stateful]# kubectl get pod -o wide
NAME            READY     STATUS      RESTARTS   AGE       IP            NODE
ds-test-2z9nc   1/1       Running     0          2m        10.244.1.61   node01
ds-test-68ql5   1/1       Running     0          2m        10.244.2.65   node02
```

正如预期，只调度到了有disktype的节点

### 2、nodeAffinity

nodeAffinity目前支持两种：

`requiredDuringSchedulingIgnoredDuringExecution`： 表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试。其中IgnoreDuringExecution表示pod部署之后运行的时候，如果节点标签发生了变化，不再满足pod指定的条件，pod也会继续运行。

`requiredDuringSchedulingRequiredDuringExecution`： 表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试。其中RequiredDuringExecution表示pod部署之后运行的时候，如果节点标签发生了变化，不再满足pod指定的条件，则重新选择符合要求的节点。

`preferredDuringSchedulingIgnoredDuringExecution`： 表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。

`preferredDuringSchedulingRequiredDuringExecution`：表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。其中RequiredDuringExecution表示如果后面节点标签发生了变化，满足了条件，则重新调度到满足条件的节点。

比如下面的例子代表调度到包含标签`ram`并且值为`32GB`或`64GB`的Node上，并且优选还带有标签`ramHz`为`2666`的节点上

给node01和node02分别打上ram为32和64

给node02打上ramHz为2666

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
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: ram
                operator: In
                values:
                - "32"
                - "64"
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: ramHz
                operator: In
                values:
                - "2666"
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 80
```

```
[root@master stateful]# kubectl get pod -o wide
NAME            READY     STATUS    RESTARTS   AGE       IP            NODE
ds-test-jqp7n   1/1       Running   0          28m       10.244.1.62   node01
ds-test-s5mxv   1/1       Running   0          28m       10.244.2.66   node02
```

### 3、podAffinity

podAffinity基于Pod的标签来选择Node，仅调度到满足条件Pod所在的Node上，相当于反向选择，前面两种策略都是从node的标签满足与否进行选择pod的位置，该方案是对pod的标签去选择，常见的有将Pod分布在同一个Node上，就可以使用这种策略。支持podAffinity和podAntiAffinity。这个功能比较绕，以下面的例子为例：

- 如果一个“Node所在Zone中包含至少一个带有security=S1标签且运行中的Pod”，那么可以调度到该Node
- 不调度到“包含至少一个带有security=S2标签且运行中Pod”的Node上

podAffinity.yaml

```

```

with-podAffinity.yaml

```

```

