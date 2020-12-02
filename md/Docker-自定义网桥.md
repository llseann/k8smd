## 自定义网桥的使用

背景：Docker容器默认会创建默认网桥docker0，以此来和容器内进行网络通讯，默认网桥地址默认为172.0.0.1，诸多时候不是很方便，所以自定义网桥，以此来实现控制容器网段的需求，同时作为熟悉docker系统配置文件的以此尝试。

使用自定义网桥一般得使用`brctl`工具，系统默认没有该项工具，此时需要我们进行安装`yum -y install bridge-utils`。

- 添加虚拟网桥

```
brctl addbr br0                                    #添加名为br0的网桥
ifconfig br0 192.168.100.1 netmask 255.255.255.0   #修改br0的地址和掩码
brctl show                                         #查看本机网桥信息
```

- 修改docker环境文件

两种方法，其一为修改`/usr/lib/systemd/system/docker.service`服务文件，添加参数，其二为修改`/etc/dcoker/daemon.json`文件，

**方法一：**

```
#修改配置文件
vi /usr/lib/systemd/system/docker.service
```

```
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/etc/default/docker       #此处为新加项
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock $DOCKER_OPTS    #末尾$DOCKER_OPTS参数为新加项
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock 
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
.
省略N行
.
[Install]
WantedBy=multi-user.target
EnvironmentFile=-/etc/default/docker        #此处为新加项
```

```
vim /etc/default/docker                     #此文件默认不存在，手动创建即可
#DOCKER_OPTS="-bip=192.168.100.1/24 --mtu=1400"
DOCKER_OPTS="-b=br0"                        #指定网桥为br0
```

操作完成后需要完成重启操作，使其生效

```
#重载
systemctl daemon-reload
#重启docker服务
service docker restart 
```

**方法二：**

```
# vim /etc/docker/daemon.json
{
 "bridge":"br0"
}
```

同样做完需要进行重载和重启操作。

***两种方法不能同时使用，使用会造成docker主程序由于配置文件冲突无法正常启动。***