## helm

作用：helm的作用就是把许多的定义 比如svc，比如deployment，比如securt一次性全部定义好，放在源里统一管理

### helm的架构

C/S架构：Helm 客户端 和 Tiller 服务器

Helm 客户端是终端用户使用的命令行工具，用户可以：

1. 在本地开发 chart。
2. 管理 chart 仓库。
3. 与 Tiller 服务器交互。
4. 在远程 Kubernetes 集群上安装 chart。
5. 查看 release 信息。
6. 升级或卸载已有的 release。

Tiller 服务器运行在 Kubernetes 集群中，它会处理 Helm 客户端的请求，与 Kubernetes API Server 交互。Tiller 服务器负责：

1. 监听来自 Helm 客户端的请求。
2. 通过 chart 构建 release。
3. 在 Kubernetes 中安装 chart，并跟踪 release 的状态。
4. 通过 API Server 升级或卸载已有的 release。

### 安装

#### 安装helm client

提前下载下列文件

```
https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get
https://kubernetes-helm.storage.googleapis.com/helm-v2.11.0-linux-amd64.tar.gz
https://kubernetes-helm.storage.googleapis.com/helm-v2.11.0-linux-amd64.tar.gz.sha256
```

修改get脚本

```
#在123行添加
cp helm* $HELM_TMP_ROOT/

#注释124-133内容
124 #  if type "curl" > /dev/null; then
125 #    curl -SsL "$CHECKSUM_URL" -o "$HELM_SUM_FILE"
126 #  elif type "wget" > /dev/null; then
127 #    wget -q -O "$HELM_SUM_FILE" "$CHECKSUM_URL"
128 #  fi
129 #  if type "curl" > /dev/null; then
130 #    curl -SsL "$DOWNLOAD_URL" -o "$HELM_TMP_FILE"
131 #  elif type "wget" > /dev/null; then
132 #    wget -q -O "$HELM_TMP_FILE" "$DOWNLOAD_URL"
133 #  fi

#修改checkDesiredVersion方法中TAG的取值逻辑
全部改成固定值,类似如下
checkDesiredVersion() {
 80   if [ "x$DESIRED_VERSION" == "x" ]; then
 81     # Get tag from release URL
 82     local release_url="https://github.com/helm/helm/releases"
 83     if type "curl" > /dev/null; then
 84       TAG=v2.11.0
 85     elif type "wget" > /dev/null; then 86       TAG=v2.11.0 87     fi 88   else 89     TAG=v2.11.0 90   fi 91 } 
```

修改完成执行get脚本，看到`Run 'helm init' to configure helm.`即为成功client

#### 安装tiller

```
#创建命名空间
kubectl create serviceaccount --namespace kube-system tiller

#创建授权
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

#安装tailler,并指定一个Chart
helm init --service-account tiller --tiller-image registry.us-east-1.aliyuncs.com/acs/tiller:v2.11.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

看到`Happy Helming!`表示tailler安装成功

#### 添加命令补全

`helm completion bash > ~/.hemlrc; echo "source ~/.hemlrc" >> ~/.bashrc`

### Helm安装Mysql

#### 直接安装

```
[root@master helm]# helm search mysql
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                                 
stable/mysql                 	0.3.5        	           	Fast, reliable, scalable, and easy to use open-source rel...
stable/percona               	0.3.0        	           	free, fully compatible, enhanced, open source drop-in rep...
stable/percona-xtradb-cluster	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source drop-in rep...
stable/gcloud-sqlproxy       	0.2.3        	           	Google Cloud SQL Proxy                                      
stable/mariadb               	2.1.6        	10.1.31    	Fast, reliable, scalable, and easy to use open-source rel...
[root@master helm]# helm install stable/mysql
#直接运行会因为没有pvc导致pod长时间处于Pending状态
[root@master helm]# kubectl delete pod toned-sasquatch-mysql-58975bd66b-b6drv
pod "toned-sasquatch-mysql-58975bd66b-b6drv" deleted
#切换到/root/.helm/cache/archive
#解压mysql-0.3.5.tgz
[root@master archive]# tar -zxvf mysql-0.3.5.tgz
#创建好符合条件的pv后
#进入到/root/.helm/cache/archive/mysql,修改自定义参数
[root@master mysql]# helm install ./	#执行本地安装
[root@master mysql]# kubectl get pod
toned-sasquatch-mysql-58975bd66b-dqbsw   0/1       Init:0/1           0          13m
[root@master mysql]# kubectl get pod
NAME                                READY     STATUS             RESTARTS   AGE
jazzy-bear-mysql-68d7746899-8n4cq   1/1       Running            0          4m
```

#### 修改文件后自定义安装

```
helm inspect values stable/mysql > mysql_helm.yaml
#修改mysql_helm.yaml内容
#指定yaml,使用helm进行安装
helm install --values=mysql_helm.yaml stable/mysql
[root@master mysql]# kubectl get pod -o wide
NAME                                      READY     STATUS             RESTARTS   AGE       IP            NODE
solitary-penguin-mysql-588754fd44-c2kz6   1/1       Running            0          1m        10.244.1.93   node01
[root@master mysql]# mysql -uroot -pthinkingdata2019 -h 10.244.1.93
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 13
Server version: 5.7.18 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> exit
Bye
```

#### 添加私有源

```
#nfs机器操作
[root@nfs charts]# mkdir /data/charts
[root@nfs charts]# docker run -dit --name=c2 -p 8081:80 -v /data:/usr/share/nginx/html docker.io/nginx


#helm机器操作
[root@master seanng]# helm repo add myotherrepo http://192.168.10.250:8081/charts
[root@master seanng]# helm repo list
NAME       	URL                                                   
stable     	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local      	http://127.0.0.1:8879/charts                                             
myotherrepo	http://192.168.10.250:8081/charts

#即可看到刚刚添加的myotherrepo源
```

### 添加chart包

一个正常的chart包应该包含以下文件结构

```
[root@master ~]# tree mysql
mysql
├── Chart.yaml
├── README.md
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── pvc.yaml
│   ├── secrets.yaml
│   └── svc.yaml
└── values.yaml
```

`Chart.yaml` 描述了chart的概要
`values.yaml` 定义了所需的各种参数，包括镜像、pv等等用户可以自定义的参数

#### 从现有的tar包修改并添加

此处以mysql为例

```
#拷贝~/.helm/cache/archive/mysql-0.3.5.tgz到该文件夹下并解压
#修改value.yaml
#打包
helm package mysql
#创建test文件夹,移动打包后的tgz包到该文件夹内
#生成index文件
helm repo index test/ --url http://192.168.10.250:8080/charts
#拷贝test下所有文件(index.yaml和mysql-0.3.5.tgz)到私有源地址
scp -r test/* 192.168.10.250:/data/charts/
```

#### 从0开始创建一个chart包

```
#创建名为mychart的chart模板
helm create mychart

[root@master ~]# tree mychart/
mychart/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
[root@master ~]# helm package mychart/
[root@master ~]# mkdir mychart
[root@master ~]# mv mychart-0.1.0.tgz mychart
[root@master ~]# helm repo index mychart/ --url http://192.168.10.250:8080/charts
[root@master ~]# scp -r mychart/* 192.168.10.250:/data/charts
[root@master ~]# helm repo add myrepo http://192.168.10.250:8080/charts
[root@master ~]# helm search mychart
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                
local/mychart 	0.1.0        	1.0        	A Helm chart for Kubernetes
myrepo/mychart	0.1.0        	1.0        	A Helm chart for Kubernetes

```

