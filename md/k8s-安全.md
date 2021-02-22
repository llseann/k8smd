## 1、 先验证authentication

### basic-auth-file：

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

### token-auth-file：输入token来进行验证

## 2、 后授权authorization

