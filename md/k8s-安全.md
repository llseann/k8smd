## 1、 验证-authentication

### basic-auth-file

开启验证：

```
#修改kube-apiserver.yaml
vim /etc/kubernetes/manifests/kube-apiserver.yaml

#添加下列内容,selfkey.csv为密码对应文件
#最好写在kubernetes的子目录,别的目录可能访问不到
- --basic-auth-file=/etc/kubernetes/pki/selfkey.csv

#重启kubectl
systemctl restart kubelet
```

```
#selfkey.csv文件格式
#格式：   密码,用户名,UID
#如下
0000,gakki,1
0000,sean,2
```

此处验证需要使用独立的kubectl的client,需要其具备以下功能:

```
  -s, --server='': The address and port of the Kubernetes API server
      --stderrthreshold=2: logs at or above this threshold go to stderr
      --token='': Bearer token for authentication to the API server
      --user='': The name of the kubeconfig user to use
      --username='': Username for basic authentication to the API server
```

能够使用`--user`以及`--username`参数

#### 输入用户名和密码-明文形式

**以下操作为非master节点操作**

完整的访问命令为：

`./kubectl -s="https://192.168.10.10:6443" --username=sean --password=0000 get pods`

会提示如下

`Unable to connect to the server: x509: certificate signed by unknown authority`

出现该提示的原因是因为需要使用https和kube-api进行通讯



测试阶段可使用`--insecure-skip-tls-verify=true`跳过验证,如：

`./kubectl -s="https://192.168.10.10:6443" --insecure-skip-tls-verify=true --username=sean --password=0000 get pods`

提示：

`Error from server (Forbidden): pods is forbidden: User "sean" cannot list pods in the namespace "default"`

这种情况是因为没有权限对默认的命名空间进行查看,需要进行额外的授权操作



还有一种情况是：

`./kubectl -s="https://192.168.10.10:6443" --insecure-skip-tls-verify=true --username=seanxxx --password=0000 get pods`

故意使用错误的username进行访问，回收到以下的回显

`error: You must be logged in to the server (Unauthorized)`

此时则需要检查秘钥文件中的秘钥对



**在master节点操作会和非master有不同的现象**

具体表现为：

1. 当用户名和密码都与秘钥文件吻合时，在没有授权的情况下会出现

`Error from server (Forbidden): pods is forbidden: User "sean" cannot list pods in the namespace "default"`

2. 当用户名和秘钥不吻合时，反而能看到命令效果，此时是因为不吻合时使用的是访客用户，故能看到结果

#### 使用证书登录

```
#生成签名证书
#以下命令在node01执行
生成私钥
openssl genrsa -out client.key 2048
生成证书请求文件
openssl req -new -key client.key -subj"/CN=node01" -out client.csr
拷贝到CA
scp client.csr master:/etc/kubernetes/pki/
签名证书
cd /etc/kubernetes/pki/
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 3650
拷贝回client
scp ca.crt client.crt node01:/ca
```

```
#指定证书文件链接
~/kubectl -s="https://192.168.10.10:6443" --certificate-authority=ca.crt --client-certificate=client.crt --client-key=client.key --username="sean" --password="0000" get pods
#在没有授权的情况下会返回
#用户和密码都正确情况下
Error from server (Forbidden): pods is forbidden: User "sean" cannot list pods in the namespace "default"
#不正确的情况下会返回
Error from server (Forbidden): pods is forbidden: User "node01" cannot list pods in the namespace "default"
```

### token-auth-file

```
#新建csv文件
[root@master pki]# touch selftoken.csv
#生成10位的随机字符串
[root@master pki]# openssl rand -hex 10
e3c213235e1340f5f109
#在文件中写入以下内容
[root@master pki]# vim selftoken.csv
e3c213235e1340f5f109,admin-sean,3
#修改kube-api文件，添加下列选项,等待api重启完成
[root@master kubernetes]# vim manifests/kube-apiserver.yaml
- --token-auth-file=/etc/kubernetes/pki/selftoken.csv
```

```
#正确token
[root@node01 ~]# ~/kubectl -s="https://192.168.10.10:6443" --insecure-skip-tls-verify=true --token="e3c213235e1340f5f109" get pods
Error from server (Forbidden): pods is forbidden: User "admin-sean" cannot list pods in the namespace "default"
#错误token
[root@node01 ~]# ~/kubectl -s="https://192.168.10.10:6443" --insecure-skip-tls-verify=true --token="e3c213235e1340f5f109xxx" get pods
error: You must be logged in to the server (Unauthorized)
```

## 2、 授权-authorization

查看kube-api文件

`- --authorization-mode=Node,RBAC`

该配置有以下类型的配置：

`AlwaysAllow`：允许所有请求

`AlwaysDeny`：拒绝所有请求

`ABAC`：Attribute-Based Access Control 基于权限,使用较少

`RBAC`：Role Based Access Control基于角色的权限控制

`NODE`：用于各个node上的kubelet访问apiserver时使用

常见的权限有`get`、`create`、`delete`、`patch`...`watch`

实际中不会吧权限授权给用户，而是把权限授权给角色(Role)，然后把角色赋予给实际用户。

把角色授权给用户的过程叫做`rolebinding`---角色绑定

常见的角色是限定于命名空间之内的。也会存在全局角色(clusterrole),全局角色的绑定称为`cluserrolebinding`

#### Role和RoleBinding

案例一：

针对`sean`用户，提供pod和svc的查看权利

`seanRole.yaml`

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: sean-reader
rules:
- apiGroups: [""]						#apiVersion类型,得和对应资源对应起来
  resources: ["pods","services"]		#资源类型
  verbs: ["get","watch","list"]			#权限列表

#也可以写成
rules:
- apiGroups: [""]
  resources:
    - pod
    - services
  verbs:
    - get
    - watch
    - list
```

`seanRolebind.yaml`

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sean-read-pods
  namespace: default
subjects:
- kind: User
  name: sean
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: sean-reader
  apiGroup: rbac.authorization.k8s.io
```

全部应用后

```
[root@master auth]# kubectl get role
NAME          AGE
sean-reader   9m
[root@master auth]# kubectl get rolebindings.rbac.authorization.k8s.io 
NAME             AGE
sean-read-pods   8m
```

此时在外部机器使用正确的用户和密码可以进行相关的查询操作

```
[root@node01 ca]# ~/kubectl -s="https://192.168.10.10:6443" --certificate-authority=ca.crt --client-certificate=client.crt --client-key=client.key --username="sean" --password="0000" get pods
NAME                                      READY     STATUS             RESTARTS   AGE
nginx-78f5d695bd-6245g                    1/1       Running            5          46d
```

案例二：

角色不仅仅是查看pod和svc,还需要查看deploment，并且能够删除和新建deploments

`seanRoleDeploment.yaml`

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: sean-reader-deploments
rules: 
- apiGroups: ["","extensions"]
  resources: ["pods","services","deployments","replicasets"]
  verbs: ["get","watch","list","create","delete","update"]
```

`seanRolebindDeploment.yaml`

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sean-read-deploments
  namespace: default
subjects: 
- kind: User
  name: sean
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role 
  name: sean-reader-deploments
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRole和ClusterRoleBinding



`seanClusterRole.yaml`

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: sean-ClusterRole
rules: 
- apiGroups: ["extensions","apps"]
  resources: ["deployments","replicasets"]
  verbs: ["get","watch", "list","create","update","delete"]
```

**通过yaml文件创建绑定**

`seanClusterRoleBind.yaml`

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sean-ClusterRoleBinding
  namespace: default
subjects: 
- kind: User
  name: gakki
  apiGroup: rbac.authorization.k8s.io
  namespace: default
roleRef:
  kind: ClusterRole
  name: sean-ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

分别创建后可以使用gakki用户在其他机器进行相关资源的访问

**使用命令行创建绑定**

`kubectl create clusterrolebinding seanClusterRoleBind --clusterrole=sean-ClusterRole --user=gakki`

同样实现可以将全局角色`sean-ClusterRole`和用户`gakki`进行绑定



总结常见的apiVersion的类型

```
pod--------- v1
deployment------extensions/v1beta1
daemonset--------extensions/v1beta1
job ----batch/v1
cronjob ---- batch/v2alpha1
service ---- v1
role ---rbac.authorization.k8s.io/v1
RoleBinding ---rbac.authorization.k8s.io/v1
```

