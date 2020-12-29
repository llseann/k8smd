## 健康性检查

使用场景：控制器判断Pod是否健康，实际情况中往往会出现Pod状态为Running，但是其实实际上里面运行的程序已经出现问题了，此时就需要对容器进行探测(probe )，而不是仅仅只判断Pod状态是否正常

探测分为两种：

- liveness ：如果检测有问题，直接进行重启Pod
- readness：如果检测有问题，不加入service

如何知道程序本身有问题：

- Command：命令行
- http GET：发送http请求
- TCP

### 1、liveness command

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 10
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5		#容器启动的5s内不监测
      periodSeconds: 5				#每5s钟检测一次
```

运行起来后执行：

```
[root@master probe]# kubectl exec -it liveness-exec ls /tmp/
healthy
[root@master probe]# kubectl exec -it liveness-exec ls /tmp/
[root@master probe]#
[root@master probe]# kubectl get pods
NAME                          READY     STATUS      RESTARTS   AGE
liveness-exec                 0/1       Completed   0          1m
[root@master probe]# kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
liveness-exec                 1/1       Running   1          1m
[root@master probe]# kubectl exec -it liveness-exec ls /tmp/
healthy
```

很明显，Pod被重启了一次。 该文件又被创建出来，判断命令的返回值，非0code则认为服务异常，会进行重启

### 2、liveness httpGet

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: nginx
    livenessProbe:
      httpGet:
        path: /index.html
        port: 80
#        schema: HTTP
      failureThreshold: 3
      initialDelaySeconds: 3
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 10
```

运行后执行：

```
[root@master probe]# kubectl get pods -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP            NODE
liveness-http                 1/1       Running   0          25s       10.244.2.77   node02
[root@master probe]# curl -I 10.244.2.77
HTTP/1.1 200 OK
Server: nginx/1.19.6
...
[root@master probe]# kubectl exec -it liveness-http bash
root@liveness-http:/# rm -rf /usr/share/nginx/html/index.html 		#删掉index.html文件
root@liveness-http:/# exit											#立马退出
exit
[root@master probe]# curl -I 10.244.2.77							#重新访问
HTTP/1.1 403 Forbidden
Server: nginx/1.19.6
Date: Tue, 29 Dec 2020 16:52:02 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive
[root@master probe]# kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
liveness-http                 1/1       Running   3          10m
```

判断能否访问到指定网页，能访问到则认为正常，反之异常，进行重启

### 3、liveness tcp

```
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
  labels:
    app: liveness-tcp
spec:
  containers:
  - name: liveness-tcp
    image: nginx
    livenessProbe:
      tcpSocket:
        port: 80		#特意改成81进行试验
      failureThreshold: 3
      initialDelaySeconds: 3
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 10
```

```
[root@master probe]# curl -I 10.244.2.78
curl: (7) Failed connect to 10.244.2.78:80; Connection refused
[root@master probe]# curl -I 10.244.2.78
HTTP/1.1 200 OK
Server: nginx/1.19.6
Date: Tue, 29 Dec 2020 17:05:03 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
Connection: keep-alive
ETag: "5fd8c14a-264"
Accept-Ranges: bytes
#因为策略问题，会有短暂的时间状态正常
#Pod会无限次的进行重启
```

会去与固定的端口建立tcp连接，连接失败会认为Pod异常，进行重启

**参数小结**

| 参数类型            | 参数含义                                                     |
| ------------------- | ------------------------------------------------------------ |
| initialDelaySeconds | 容器启动后第一次执行探测是的等待时间                         |
| periodSeconds       | 执行探测的频率，默认是10秒，最小1秒                          |
| timeoutSeconds      | 探测超时时间，默认1秒，最小1秒                               |
| successThreshold    | 探测失败后，最少连续探测成功多少次才被认定为成功，默认是1，对于liveness必须是1，最小值是1 |
| failureThreshold    | 探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。 |

### 4、readiness command

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness		#改标签
  name: readiness-exec	#改名
spec:
  containers:
  - name: readiness
    image: docker.io/busybox:latest
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 10
    readinessProbe:			#改动类型
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

也会重启，但是不加入service，相当于将其冷藏起来。直到恢复