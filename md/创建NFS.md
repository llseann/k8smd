### 创建NFS

服务端:

```shell
[root@nfs ~]# yum install nfs-u*				#安装NFS相关
[root@nfs ~]# systemctl start nfs				#启动
[root@nfs ~]# systemctl enable nfs				#开机自启
Created symlink from /etc/systemd/system/multi-user.target.wantsice.
[root@nfs ~]# mkdir share						#创建被共享的目录
[root@nfs ~]# cd share/
[root@nfs share]# vi /etc/exports				#编辑共享目录配置文件
/root/share *(rw,async,no_root_squash)			#异步async提高效率,no_root_squash避免权限问题
[root@nfs share]# exportfs -avr					#开始共享
exporting *:/root/share
```

客户端:

```shell
[root@k8syum01 pods]# yum install */showmount -y					#安装客户端工具
[root@k8syum01 pods]# showmount -e 192.168.10.250 					#显示NFS信息
Export list for 192.168.10.250:
/root/share *
[root@k8syum01 pods]# mkdir /share									#创建本地挂载点
[root@k8syum01 pods]# mount 192.168.10.250:/root/share /share		#挂载
```

### iscsi

服务端

```shell
[root@nfs ~]# yum install target* -y
[root@nfs ~]# systemctl start target
[root@nfs ~]# systemctl enable target
Created symlink from /etc/systemd/system/multi-user.target.wants/target.service to /usr/lib/systemd/system/target.service.
[root@nfs ~]# targetcli
targetcli shell version 2.1.fb49
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> backstores/block create blocksean /dev/sdb
Created block storage object blocksean using /dev/sdb.

/> iscsi/ create iqn.2020-07.cn.llsean:disk
Created target iqn.2020-07.cn.llsean:disk.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.

/> cd iscsi/iqn.2020-07.cn.llsean:disk/tpg1/
/iscsi/iqn.20...ean:disk/tpg1> ls
o- tpg1 ........................................................
  o- acls ......................................................
  o- luns ......................................................
  o- portals ...................................................
    o- 0.0.0.0:3260 ............................................
    
/iscsi/iqn.20...ean:disk/tpg1> acls/ create iqn.2020-07.cn.llsea
Created Node ACL for iqn.2020-07.cn.llsean:gakki
/iscsi/iqn.20...ean:disk/tpg1> luns/ create /backstores/block/blocksean 
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2020-07.cn.llsean:gakki
```

客户端

```shell
[root@k8syum01 ~]# yum install iscsi*
[root@k8syum01 ~]# vim /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2020-07.cn.llsean:gakki
#此处要记得修改
```

