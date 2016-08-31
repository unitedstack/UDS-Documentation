# 6 OSD管理

OSD是集群中负责数据存储的重要组件，集群容量不足时需要进行扩容也就是创建并添加新的OSD、集群中出现坏盘或者节点下线时需要删除已有的OSD，这里主要介绍一下OSD创建和删除的标准流程。

## 6.1 OSD的创建

要添加一个OSD，需要依此进行如下操作：

* 申请一个OSD id或者向集群注册一个OSD id。

```
# ceph osd create [{uuid} [{id}]]
```

* 在新OSD主机上创建OSD的数据存储目录。

```
# sudo mkdir /var/lib/ceph/osd/ceph-{osd-number}
```

* 如果准备用于 OSD 的是单独的而非系统盘，则先创建文件系统，并把它挂载到刚创建的目录下

```
# sudo mkfs -t {fstype} /dev/{drive} # sudo mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}
```

这里文件系统fstype目前一般设置为xfs。

* 初始化 OSD 数据目录。

```
# ceph-osd -i {osd-num} --mkfs --mkkey 
```

* 注册 OSD 认证密钥， ceph-{osd-num} 路径里的 ceph 值应该是 $cluster-$id，如果你的集群名字不是 ceph ，那就用改过的名字。

```
# ceph auth add osd.{osd-num} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-{osd-num}/keyring 
```

* 把 OSD 加入 CRUSH 图，这样它才开始收数据。用 ceph osd crush add 命令把 OSD 加入 CRUSH 分级结构的合适位置。如果你指定了不止一个桶，此命令会把它加入你所指定的桶中最具体的一个，并且把此桶挪到你指定的其它桶之内。重要：如果你只指定了 root 桶，此命令会把 OSD 直接挂到 root 下面，但是 CRUSH 规则期望它位于主机内。

```
# ceph osd crush add {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
```

这里关于OSD的权重值的设置，一般参考这个公式 osd weight = osd capacity\(in GB\) \/ 1024\(GB\)得出来的值。

## 6.2 OSD的删除

删除一个OSD可以有很多做法，不同的做法对集群的影响也不尽相同。具体需要分两种情况：

1）删除一个正常工作的OSD（仍处于UP状态），可以按照以下步骤进行：

先将OSD的权重设置为0，这时候会触发集群内部的数据迁移，并且这个OSD会参与数据迁移并且会加快迁移过程，等待数据迁移完成之后，执行下一步操作。

* 将OSD out掉。
* 停掉OSD对应的服务进程。
* 将OSD从CRUSH中删除掉。
* 删除OSD对应的Auth信息。
* 从集群中删除OSD。
* 编辑ceph.conf文件，从中删除掉对应的OSD的信息。

```
# ceph osd crush reweight osd.x 0.0# ceph osd out osd.x# service ceph stop osd.x# ceph osd crush remove osd.x# ceph auth del osd.x# ceph osd rm osd.x
```

2）删除一个异常的OSD（已经处于Down状态的OSD），可以按照以下步骤完成：

* 将OSD out掉，不用等待数据迁移完成，接着执行如下操作。
* 将OSD从CRUSH中删除。
* 删除OSD对应的Auth信息。
* 从集群中删除OSD。
* 编辑ceph.conf文件，从中删除掉OSD的信息。

```
# ceph osd out osd.x
# ceph osd crush remove osd.x
# ceph auth del osd.x
# ceph osd rm osd.x
```

