# 2 UDS 快速安装指南

## 2.1 简介

UDS快速安装主要是基于ceph-deploy工具完成。

UDS Ceph Azeroth基于Ceph 0.94.7的官方版本进行构建，UDS Ceph Azeroth 提供在线安装与离线安装两种方式

1）在线安装

如果是在线安装请添加UDS的源

```
[uds-ceph]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
```

2）离线安装

需要带着UnitedStack自己的Repo。具体可以从[http:\/\/uds.ustack.com\/repo\/](http://uds.ustack.com/repo/)下载。

在配置了UDS的源之后可以通过yum来安装最新的ceph-deploy

```
## 执行命令，并确认安装包版本 >= 1.5.33
yum info ceph-deploy
## 安装 ceph-deploy
yum install ceph-deploy
```

Ceph-deploy通过一个admin节点，ssh到目标机上执行命令，来完成UDS Ceph的安装与部署。没有使用Puppet，所以相对来说

比较简单。前提是你已经将admin节点配置成ssh无密码登陆了。配置ssh无密码登录如下

```
# server-69:~ ssh-keygen -t rsa
# server-69:~ ssh-copy-id root@server-68
```

这样就会把节点server-69的public-key拷贝到节点server-68上的authorized\_keys里，使得节点server-69可以通过密钥无密码登录节点server-68。

## 2.2 创建UDS Ceph集群

如果之前节点上有Ceph残留的数据，请先将先前数据和配置清理干净，执行：

```
ceph-deploy purge host [host....]

比如：
ceph-deploy purge server-{68,68,70}
```

然后执行：

```
ceph-deploy new host [host....]

比如：
ceph-deploy new server-{68,69,90}
```

注：这里填写的是初始化MON需要的节点，OSD所在的节点不需要填写

执行该命令将自动生成两个配置文件：ceph.conf, ceph.mon.keyring ，后续的操作，将依赖这两个配置文件来进行。ceph.conf 里面存放了一些默认的配置，可以在此基础上添加或者修改配置。ceph.mon.keyring 是MON初始的keyring，在建立MON的时候使用，通过这个keyring，才有权限对MON进行操作。

### **2.2.1 改变配置文件**

ceph-deploy生成的配置文件里面的配置项很少，需要根据需求，自行添加配置，比如让某些debug信息更详细，调整messenger，journal，filestore等等，都可以在这里设置。

### **2.2.2 安装CEPH**

ceph-deploy中增加了对repo的管理，可以通过如下方式，很便捷的管理所有节点上的repo，启用本地源或者互联网上的源进行UDS Ceph的安装

```
ceph-deploy repo --repo-url http://uds.ustack.com/repo/Azeroth/el7/ ceph server-{68,69,70}
ceph-deploy pkg --install ceph server-{68,69,70}
```

### **2.2.3 添加初始化MONITORS**

创建MONITOR

```
ceph-deploy mon create-initial
```

### **2.2.4 调整CRUSH TUNABLES**

UDS Ceph CRUSH tunables默认是bobtail，这是为了兼容老的版本，在Centos7上可以将其设置为optimal，有助于改善数据在节点间的分布。

在部署好Monitor之后，就可以通过命令来设置

```
ceph osd crush tunables optimal
```

### **2.2.5 添加OSD**

如果OSD节点上的磁盘曾经有被使用过，磁盘上面是有数据的，请先进行清理，如果是全新环境，则可跳过本步骤，建议都执行该步骤以确保后续的部署能够成功，执行：

```
ceph-deploy disk zap <ceph-node>:<disk-device> 

比如：需要清理server-68,server-69,server-70 上的磁盘 /dev/sdb, /dev/sdc, /dev/sdd，可执行：
ceph-deploy disk zap server-{68,69,70}:/dev/sd{b,c,d}
```

UDS Ceph OSD节点推荐使用全SSD（会有较好的性能和速度），也可以节约成本，使用SSD盘作为UDS Ceph Journal，SAS或者SATA作为data盘。ceph-deploy可以帮你实现不同架构的部署，命令如下：

```
ceph-deploy osd prepare <ceph-node>:<disk-device>[:disk-journal] [<ceph-node>:<disk-device>[:disk-journal]] 

比如：需要在server-68,server-69,server-70 上的用磁盘 /dev/sdb, /dev/sdc, /dev/sdd 创建OSD，可执行：
ceph-deploy osd prepare server-{68,69,70}:/dev/sd{b,c,d}
```

如果在prepare的时候没有指定journal，则ceph-deploy会自动将disk-device的最开始5G作为UDS Ceph Journal的分区来使用。如果想自己指定Journal路径，就需要手动进行分区，建议使用sgdisk来做分区。

在prepare完毕之后，需要使用activate命令来将osd激活，需要注意的是prepare的disk-device参数和activate的时候是不一样的，如果你在prepare的时候没有指定Journal路径，则在activate必须找到数据盘，即：activate只能输入数据盘

```
ceph-deploy osd activate ceph-node:disk-device[:disk-journal] 

比如：需要在server-68,server-69,server-70 激活 /dev/sdb, /dev/sdc, /dev/sdd 初始化的OSD，执行：
ceph-deploy osd activate server-{68,69,70}:/dev/sd{b,c,d}1
```

而如果指定了Journal，则activate的时候就可以必须将Journal分区也带上

```
ceph-deploy osd activate server-{68,69,70}:/dev/sd{b,c,d}:/dev/sda
```

注：\/dev\/sda 为Journal盘，不需要对\/dev\/sda进行分区，Journal分区默认大小为5GB，若需要改变Journal分区大小，请在ceph.conf文件中加入osd\_journal\_size字段进行修改，如：Journal分区需要15G，可设置为osd\_journal\_size ＝ 15360
执行命令时，需对远端配置文件进行覆盖，使其生效。
ceph-deploy --overwrite-conf osd activate server-{68,69,70}:\/dev\/sd{b,c,d}:\/dev\/sda

### **2.2.6 创建CRUSH 层级**

在OSD创建好之后，仅仅建立了host和root的对应关系，如果有更复杂的物理层级，需要进行额外配置（如：rack，room）。配置是否合理关系到生产环境集群的可靠性及可用性，应认真规划

执行如下命令，添加新的层级

```
添加新的Map typs导出decodemap：
ceph osd getcrushmap -o mapcrushtool -d map -o decodemap 

编辑decodemap：
vim decodemap

在typs下定义新的类型
# types
type 11 osd-domain
type 12 host-domain
type 13 replica-domain
type 14 failure-domain

导入encodemap：
crushtool -c decodemap -o encodemapceph osd setcrushmap -i encodemap 

将OSD加入物理故障域：
ceph osd crush create-or-move <osdname (id|osd.id)> <float[0.0-]> <args> [<args>...] 

如：将server-68下的OSD进入物理故障域，并且将server-68加入逻辑故障域
ceph osd crush create-or-move osd.1 0.42 host=server-68 rack=rack-01 root=default
ceph osd crush create-or-move osd.5 0.42 host=server-68 rack=rack-01 root=default 

将OSD加入逻辑故障域：ceph osd crush link <osdname> <args> [<args>...]
注：将OSD加入逻辑故障域前，必须先将OSD加入物理故障域 加入逻辑故障域（host-domain方式）
ceph osd crush link server-68 host-domain=host-group-0-rack-01 replica-domain=replica-0 failure-domain=apple 

加入逻辑故障域（osd-domain方式）
ceph osd crush add osd.1 0.42 osd-domain=osd-group-0-rack-01 replica-domain=replica-0 failure-domain=apple
ceph osd crush add osd.5 0.42 osd-domain=osd-group-0-rack-01 replica-domain=replica-0 failure-domain=apple 

注：逻辑故障域只需要从host-domain或osd-domain中选择一种加入即可。以上操作不需要手动创建bucket（server-68,rack-01等），如果bucket不存在，会自动被创建
```

当然这里的前提是CRUSH里已经有rack这个类型，如果没有，就需要手动添加这个类型（getcrushmap后，使用vim编辑CRUSHMap，最后setcrushmap）。通常情况下，用户不需要额外设置，UDS Ceph 提供默认的类型已经足够覆盖绝大多数的使用场景，从上至下分别为：root, region, datacenter, room, pod, pdu, row, rack, chassis, host, osd。

UDS Ceph没有使用官方默认的层级，而是在其物理层级之上，建立了逻辑层级，来增强Ceph的可用性与可靠性。

### **2.2.7 CHECK CRUSH层级**

在CRUSH层级建立好之后，通过ceph osd tree 验证CRUSH是否创建成功，并且是符合预期的，如下

![](/assets/DDF264E8-D34C-400F-A054-36AE23323E57.png)

### **2.2.8 建立一个POOL**

由于POOL的建立牵扯到crush\_ruleset，而crush\_ruleset的编写又牵扯到CRUSH，CRUSH本身比较复杂，但是其本质是简单的。CRUSH本质上就是一个n叉树，每一层都可以由rule来控制，最终的叶子节点是OSD，这样根据输入，选择出OSD，就把pg与OSD对应起来了，具体原理，请参见UDS Ceph技术手册（需要有链接）。

如何创建一个资源池POOL

#### **RBD服务相关**

RBD对应的是块接口，UDS Ceph会根据不同的硬件类型来选择建立不同的资源池，而不同的资源池通过crush\_ruleset来进行选择，一般openstack-00的资源池将数据映射至纯SSD的OSD，sata-00资源池将数据映射至SATA的OSD，或者SSD Journal+SATA的OSD。可以通过以下命令来创建资源池

```
rbd_pg_num=64*osd个数
rbd_pgp_num =$rbd_pg_num
rbd_min_size=2
rbd_size=3
ceph osd pool create openstack-00 $rbd_pg_num $rbd_pgp_num
ceph osd pool set openstack-00 min_size $rbd_min_size
ceph osd pool set openstack-00 size $rbd_size
```

#### **RGW服务相关**

RadosGW在启动的时候，需要建立13个资源池（POOLS），如果一个一个手动建立效率低下，一般可以这么做

#### **CephFS文件服务相关 —— 仅供测试使用，UDS Ceph Azeroth 版本不提供对文件服务相关支持**

CephFS 的建立需要两个POOL，一个POOL作为metadata，一个POOL作为data。我们推荐将metadata POOL放在全SSD的OSD上，而将data POOL放在SSD+SAS\/SATA 的OSD上面。具体可以通过如下命令来建立POOL

```
#!/bin/bashosd_num=2
metadata_pg_num=$((16*osd_num))
metadata_pgp_num=$((16*osd_num))
metadata_size=2
metadata_minsize=1
metadata_ruleset=0
data_pg_num=$((64*osd_num))
data_pgp_num=$((64*osd_num))
data_size=2
data_minsize=1
data_ruleset=0
ceph osd pool create metadata $metadata_pg_num $metadata_pgp_num
ceph osd pool set metadata size $metadata_size
ceph osd pool set metadata min_size $metadata_minsize
ceph osd pool set metadata crush_ruleset $metadata_ruleset
ceph osd pool create data $data_pg_num $data_pgp_num
ceph osd pool set data size $data_size
ceph osd pool set data min_size $data_minsize
ceph osd pool set data crush_ruleset $data_ruleset
```

### **2.2.9 MDS的部署 —— 仅供测试使用，UDS Ceph Azeroth 版本不提供对文件服务相关支持**

MDS用于CephFS分布式文件系统，实现元数据的快速查询。通常传统的文件系统在处理海量数据的时候，元数据的管理十分低效，而解决方法有两种，一种是直接把文件语义丢弃，换成对象存储，简单的kv对应，就能处理海量数据且扩展性好，另一种解决办法是，通过分布式元数据服务器解决元数据的管理问题，而CephFS中的MDS正是用来做分布式文件系统元数据管理的，并且引入了动态子树分区，使得元数据可以动态漂移，具有较高的复杂度，且目前实现并不稳定，仍不适合生产环境使用。

ceph-deploy部署MDS十分简单，执行

```
ceph-deploy mds create server-68
ceph fs new cephfs metadata data
```

只有在建立了cephfs之后，在可以在ceph -s中看到MDS，具体如下所示

![](/assets/83806D58-2B38-4FA5-BB3D-DB53D2523E99.png)

### **2.2.10 RGW服务的部署**

部署RGW服务，必须先安装依赖包ceph-radosgw

```
ceph-deploy pkg --install ceph-radosgw server-68
```

然后使用ceph-deploy进行部署，执行

```
ceph-deploy rgw create server-68
```

