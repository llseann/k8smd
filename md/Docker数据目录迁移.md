背景：build镜像失败,查看本机目录发现根目录已经使用了96%,查看发现空间全被`/var/lib/docker`占用,所以需要进行该目录的迁移

**前置步骤**

使用下面两种办法都需要进行该步骤
```
[root@ta-docker ~]# systemctl stop docker #停止docker服务

[root@ta-docker ~]# mkdir -p /data1/docker/   #创建新的docker目录，执行命令df -h,找一个大的磁盘
```

## 方法一：修改默认配置文件

```
[root@ta-docker ~]# rsync -avz /var/lib/docker/ /data1/docker/  #迁移到新路径下
[root@ta-docker ~]# vim /usr/lib/systemd/system/docker.service

[Service]
 10 Type=notify
 11 # the default is not to use systemd for cgroups because the delegate issues still
 12 # exists and systemd currently does not support the cgroup feature set required
 13 # for containers run by docker
 14 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
 15 ExecReload=/bin/kill -s HUP $MAINPID
 16 TimeoutSec=0
 17 RestartSec=2
 18 Restart=always
 
 #修改14行,在后面添加上--graph=/data/docker/
 #修改后为如下：
 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --graph=/data/docker/
 
[root@ta-docker ~]# systemctl daemon-reload
[root@ta-docker ~]# systemctl start docker
[root@ta-docker ~]# docker info | grep Root
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled
 Docker Root Dir: /data/docker

#校验现有镜像和容器
[root@ta-docker ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
tkdemo              v1                  09e4daab0640        20 hours ago        16.8GB
centos_163          v7                  0f793ba5281b        2 years ago         356MB

#确认无误删除原来的/var/lib/docker即可
```

## 方法二：使用软连接
**同样需要执行前置步骤**
```
移动原始目录
[root@ta-docker ~]# rsync -avz /var/lib/docker/ /data1/docker/  #迁移到新路径下
[root@ta-docker ~]# mv /var/lib/docker /data/docker_bak
[root@ta-docker ~]# ln -s /data/docker /var/lib/docker
[root@ta-docker ~]# systemctl daemon-reload
[root@ta-docker ~]# systemctl start docker

执行完参考方法一检查步骤进行检查即可
没问题可以删除/data下的docker_bak目录
```