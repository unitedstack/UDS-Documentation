# 通过ceph-fuse访问文件系统

## 安装ceph-fuse

ceph-fuse不是Ceph默认安装的软件，如果想要试用ceph-fuse来挂载文件系统，必须先安装它。UDS本身提供ceph-fuse的源，所以安装起来也比较简单。

```
# yum install ceph-fuse
```

## 挂载CephFS

因为在UDS中默认是开启认证的，所以通过ceph-fuse挂载文件系统之前，首先需要保证客户端上有ceph的配置文件以及能够访问CephFS的密钥。关于密钥，如果您使用admin挂载CephFS，则需要将ceph.client.admin.keyring文件从其它节点上拷贝到客户端上。然后通过如下命令挂载即可，

```
ceph-fuse -m {monitor-ip:port} [-m {monitor-ip:port}] {mount-point}
```

例如，

```
# ceph-fuse -m 192.168.0.7:6789:/ /mnt/
ceph-fuse[6658]: starting ceph client
2016-06-25 13:46:45.222713 7f446740e780 -1 init, newargv = 0x48e1b50 newargc=11
ceph-fuse[6658]: starting fuse
```

挂载之后，就可以在系统中看到了

```
# mount|grep ceph-fuse
ceph-fuse on /mnt type fuse.ceph-fuse (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions,allow_other)
```

进入\/mnt\/就可以像使用正常文件系统一样使用cephfs了。

如果是通过其它用户访问，则可以按照以下步骤为其创建密钥，假设使用名为test的用户挂载。

```
生成key
# ceph auth get-or-create client.test mon 'allow r' mds 'allow r' osd 'allow rwx pool=cephfs_data, allow rwx pool=cephfs_metadata'
[client.test]
	key = AQAkHG5XM47fBhAAsRouAV/5xOis0aZMACGNoQ==
 
导出keyring到文件中
#  ceph auth export client.test > /etc/ceph/client.test.keyring
 
挂载文件系统
# ceph-fuse --id test -k /etc/ceph/client.test.keyring -m 192.168.0.7:/ /mnt/
ceph-fuse[2016-06-25 13:53:57.810079 7f4a56c93780 -1 init, newargv = 0x3eb0b50 newargc=11
6854]: starting ceph client
ceph-fuse[6854]: starting fuse
```

## 卸载CephFS

卸载时，只需要将已经mount的文件系统umount掉即可。

```
# umount /mnt/
[root@ceph-client ~]# ps -ef|grep ceph-fuse
root      6962  6505  0 14:00 pts/0    00:00:00 grep --color=auto ceph-fuse
```

