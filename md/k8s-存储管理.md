[TOC]



## k8s的存储

k8s的支持的存储有很多：

### 本地存储

1. emptyDir 等价于如下

```shell
docker run -name a1 -v /xx nginx
```

```shell
spec:
  volumes:
  - name: volume1
    emptyDir: {}
  containers:
  - name: demo1
    image: busybox
    command: ['sh','-c','sleep 5000']
    volumeMounts:
    - mountPath: /xx
      name: volume1
  
  - name: demo2
    image: busybox
    command: ['sh','-c','sleep 5000']
    volumeMounts:
    - mountPath: /xx
      name: volume1
#如上：两个容器demo1和demo2共享存储卷volume1,切都挂在到同名路径/xx下
#此时可以实现的效果为在demo1的/xx下写入文件,在demo2的/xx中也能看到该文件
```

2. hostPath 等价于如下

```shell
docker run -name a1 -v /xx:/data nginx
```

```shell
spec:
  volumes:
  - name: volume1
    hostPath:
      path: /data/gakki				#注意空格
  containers:
  - name: demo1
    image: busybox
    command: ['sh','-c','sleep 5000']
    volumeMounts:
    - mountPath: /xx
      name: volume1
  
  - name: demo2
    image: busybox
    command: ['sh','-c','sleep 5000']
    volumeMounts:
    - mountPath: /xx
      name: volume1
#此时实现的效果是将容器中的路径映射到宿主机的某个路径下
```

两者区别在于是否指定了宿主机的绝对路径,存在的问题在于,本地存储和宿主机的耦合性很强,倘若跨主机工作,该模式存在问题,无法读取到宿主机的数据

### 网络存储

1. NFS(客户端多的时候会产生瓶颈)

```shell
spec:
  volumes:
  - name: nfs
    nfs:
      server: 192.168.10.250
      path: "/root/share"			#注意不要写错
  containers:
  - image: docker.io/nginx:latest
    name: nginx
    volumeMounts:
    - name: nfs
      mountPath: "/usr/share/nginx/html"
[root@k8syum01 pods]# kubectl apply -f nfs.yaml 
pod "nginx" created
[root@k8syum01 pods]# kubectl exec nginx -it bash
root@nginx:/# df -Th
Filesystem                 Type     Size  Used Avail Use% Mounted on
overlay                    overlay   17G  5.6G   12G  33% /
tmpfs                      tmpfs    912M     0  912M   0% /dev
tmpfs                      tmpfs    912M     0  912M   0% /sys/fs/cgroup
/dev/mapper/centos-root    xfs       17G  5.6G   12G  33% /etc/hosts
shm                        tmpfs     64M     0   64M   0% /dev/shm
192.168.10.250:/root/share nfs4      17G  4.2G   13G  25% /usr/share/nginx/html
tmpfs                      tmpfs    912M  8.0K  912M   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                      tmpfs    912M     0  912M   0% /proc/acpi
tmpfs                      tmpfs    912M     0  912M   0% /proc/scsi
tmpfs                      tmpfs    912M     0  912M   0% /sys/firmware
#NFS目录已经被挂载
```

   ```shell
#此处还出现一个问题,pod创建后长时间处于ContainerCreating状态,如下进行排查
[root@k8syum01 pods]# kubectl describe pod nginx		#善用describle关键字
...
Output: mount.nfs: access denied by server while mounting 192.168.10.250:/share
#此时反应过来,我在NFS中的目录为/root/share,修改后删除旧pod,重新启动即可
   ```

2. iscis

```
spec:
  containers:
  - name: iscsipd-rw
    image: nginx
    volumeMounts:
    - mountPath: "/mnt/iscsipd"
      name: iscsipd-rw
  volumes:
  - name: iscsipd-rw
    iscsi:
      targetPortal: 192.168.10.250:3260
      portals: ['192.168.10.21:3260', '192.168.10.22:3260']		#别落下此处
      iqn: iqn.2020-07.cn.llsean:disk
      lun: 0
      fsType: ext4
      readOnly: false
[root@k8syum01 pods]# kubectl apply -f iscsi.yaml
[root@k8syum01 pods]# kubectl exec iscsipd -it bash
root@iscsipd:/# df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                   17G  5.6G   12G  33% /
tmpfs                    912M     0  912M   0% /dev
tmpfs                    912M     0  912M   0% /sys/fs/cgroup
/dev/mapper/centos-root   17G  5.6G   12G  33% /etc/hosts
/dev/sdb                  20G   45M   19G   1% /mnt/iscsipd
shm                       64M     0   64M   0% /dev/shm
tmpfs                    912M  8.0K  912M   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    912M     0  912M   0% /proc/acpi
tmpfs                    912M     0  912M   0% /proc/scsi
tmpfs                    912M     0  912M   0% /sys/firmware
```

**要记得修改客户端`initiatorname.iscsi`文件,并且yaml中不要落下`portals`字段**

2. ceph：pass
3. gluster：pass

### 持久性存储

```shell
#pv模板
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvdemo01
spec:
  capacity:
    storage: 5Gi			#大小
  volumeMode: Filesystem	#模式
  accessModes:
    - ReadWriteOnce			#权限模式
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /pv01
    server: 192.168.10.250
#pv格式
#ReadWriteOnce
#ReadOnlyMany
#ReadWriteMany 
```

`pv PersistenVolume`:基于全局，是外部存储系统中的一块存储空间，由管理员创建和维护。与 Volume 一样，PV 具有持久性，生命周期独立于 Pod

`pv PersistenVolume claim`:基于命名空间，是对 PV 的申请 (Claim)。PVC 通常由普通用户创建和维护。需要为 Pod 分配存储资源时，用户可以创建一个 PVC，指明存储资源的容量大小和访问模式（比如只读）等信息，Kubernetes 会查找并提供满足条件的 PV。

常用指令:

```
[root@master pods]# kubectl get pv
[root@master pods]# kubectl get pvc
[root@master pods]# kubectl get pv -n sean			#使用-n执行命令在指定的命名空间
[root@master pods]# kubectl describe pv pvdemo01	#查看pv的具体描述
[root@master pods]# kubectl get pv -o yaml
```

```yaml
#pvc模板
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvcdemo01
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
```

**pv和pvc关联的依据是`accessModes`和`storage`两个字段,只要两字段完全符合,就会进行关联,不需要手动进行操作,如果有多个yaml文件对应同一个pv创建pvc,则后创建的pvc会一直处于`Pending`状态。**

NG实例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: pvcdemo01
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    volumeMounts:
      - mountPath: "/mnt"
      name: mypvc
  restartPolicy: Always
```

小插曲：

创建NG的POD的时候,pod长时间处于`ContainerCreating`状态,`describe`查看,发现如下报错：

```shell
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/pods/824f6817-e561-11ea-b9a8-000c29f669e3/volumes/kubernetes.io~nfs/pvdemo01 --scope -- mount -t nfs 192.168.10.250:/pv01 /var/lib/kubelet/pods/824f6817-e561-11ea-b9a8-000c29f669e3/volumes/kubernetes.io~nfs/pvdemo01
Output: Running scope as unit run-10966.scope.
mount: wrong fs type, bad option, bad superblock on 192.168.10.250:/pv01,
       missing codepage or helper program, or other error
       (for several filesystems (e.g. nfs, cifs) you might
       need a /sbin/mount.<type> helper program)

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
```

看起来是挂载nfs分区的问题,上网查阅后确定了此想法,需要安装nfs的相关工具,如下:

`yum install nfs-common  nfs-utils -y`

**NFS服务器共享目录的权限很关键，`no_root_squash`选项经常会带来很多奇怪问题.务必设置正确**