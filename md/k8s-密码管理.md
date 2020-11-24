## 密码管理

pass：为了安全性，并不在pod中写密码。而是把密码写到其他地方。在pod需要的时候进行引用即可

常见有两种形式一位`secret`另一种为`configmaps`

## 一、secret

### 1、密码创建

#### 方式一：命令行直接创建

```
kubectl create secret generic mysecret1 --from-literal=user=sean --from-literal=password1=seansecret1 --from-literal=password2=seansecret2
```

创建名为mysecret1，包含三个密码，分别名为user，password1，password2，使用`--from-literal`添加密码内容

#### 方式二：从文件导入(每个文件只存一个密码串)

```
#创建密码文件
echo -n sean > user
echo -d seansecret1 > password1
echo -d seansecret2 > password2
#生成secret
kubectl create secret generic mysecret2 --from-file=./user --from-file=./password1 --from-file=./password2
```

使用`--from-file`添加包含密码的文件，文件名就是密码对应的名字

#### 方式三：从文件导入(env文件，可以包含多对密码，以换行进行分隔)

`serect.env`文件

```
user=sean
password1=seanserect1
password2=seanserect2
```

生成serect,使用`--from-env-file`指明文件路径

```
kubectl create secret generic mysecret3 --from-env-file=./serect.env
```

#### 方式四：使用yml生成serect

serect.yml

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret4
type: Opaque
data:
  user: c2Vhbg==
  password1: c2VhbnNlY3JldDE=
  password2: c2VhbnNlY3JldDI=
```

```
kubectl apply -f serect.yaml 
```

写在yaml里面的密码需要经过base64的编码后才能写入。

**查看现有的serect**

```
kubectl get secrets mysecret1 -o yaml
kubectl edit secrets mysecret2
```

两种方法效果一样，第二种方法能够进行编辑

如：

```
apiVersion: v1
data:
  password1: c2VhbnNlY3JldDE=
  password2: c2VhbnNlY3JldDI=
  user: c2Vhbg==
kind: Secret
metadata:
  creationTimestamp: 2020-11-23T14:43:38Z
  name: mysecret1
  namespace: default
  resourceVersion: "13386"
  selfLink: /api/v1/namespaces/default/secrets/mysecret1
  uid: 45ab4e13-2d9a-11eb-9840-000c29f669e3
type: Opaque
```

可以看到`password1`a字段为密文，需要使用base64进行解码查看

`echo -n c2VhbnNlY3JldDE= | base64 -d`

### 2、如何使用

#### 方式一：以卷的方式使用

pod1.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
  - name: xx
    secret:
      secretName: mysecret1
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - name: xx
      mountPath: "/etc/xx"
      readOnly: true
```

```
[root@master secret]# kubectl exec -it nginx bash
root@nginx:/# ls /etc/xx/
password1  password2  user
root@nginx:/# cat /etc/xx/user 
seanroot@nginx:/#
```

可以看到在挂载点会有所有的密码文件，且经过了base64解码。但实际情况中，并不一定会需要使用到所有的密码串，此时可以使用`items`进行选择

pod2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
  - name: xx
    secret:
      secretName: mysecret1
      items:
      - key: user
        path: ll/user
      - key: password1
        path: uu/passwd
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - name: xx
      mountPath: "/etc/xx"
      readOnly: true
```

```
root@nginx:~# ls /etc/xx/
ll  uu
root@nginx:~# cat /etc/xx/ll/user 
seanroot@nginx:~# 
```

可以看到，密码文件的路径由挂载点路径拼接了items下的路径而来。这种方法和上面一种不一样的是，可以选择需要的密码进行使用，并可指定其挂载路径。但是要注意把密码的名字写对，否则会出现pod启动失败的情况。通过挂载卷来进行密码文件的存储还有一个好处是可以进行动态修改，在宿主机使用`kubectl edit secrets mysecret2`可以对密码进行动态修改(需要重新进入容器内部才能看到更改后的效果)

#### 方式二：变量的方式来使用sercert

mysql.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mysql
  name: mysql
spec:
  containers:
  - image: mysql
    name: mysql
    ports:
    - containerPort: 3306
      name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret1
          key: password1
```

这种方法下，mysql的必须环境变量可以被赋值的密码所取代，从而正常的启动服务，但是***不支持动态更新***

## 二、ConfigMap

**configmap和secret最大的区别是前者不会保存编码后的密码，两者使用上完全一致，下面略作解释**

 密码创建

```
kubectl create configmap my1 --from-literal=user=tom --from-literal=password=redhat
kubectl create configmap my2 --from-file=./user --from-file=./password1
kubectl create configmap my3 --from-env-file=./env.txt
```

以上三种方式完全类似于secret的使用，分别实现从命令行传密码，文件存储单独密码以及env传密码

同样的，configmap也可以从yaml中挂载密码卷以及使用变量进行密码使用

my1.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
  - name: xx
    configMap:			#修改此处类型
      name: my1			#修改此处为configmap名
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - name: xx
      mountPath: "/my"
      readOnly: true
```

以上yaml实现了讲configmap挂载到指定目录。以卷的方式来使用

my2.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mysql
  name: mysql
spec:
  containers:
  - image: mysql
    name: mysql
    ports:
    - containerPort: 3306
      name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        configMapKeyRef:			#修改此处
          name: my1
          key: password1
```

