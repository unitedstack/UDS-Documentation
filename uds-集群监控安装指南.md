# 4 UDS 集群监控安装指南

UDS Ceph集群监控基于社区版本Calamari，能够完成一些基本的运维操作以及简单的集群监控，目前UDS Ceph监控只支持CentOS 7版本操作系统的部署安装，其它操作系统的部署安装，需自行编译源码

安装部署calamari之前，需要准备一个calamari master节点（可以是物理机，也可以是虚拟机），但是必须保证该节点网络能够与所有Ceph节点（包括MON，MDS，OSD）正常通信。一下的安装配置操作均在calamari master节点上执行。

## 4.1 calamari master节点安装与配置

### **4.1.1 配置calamari 安装源**

配置并启用UDS源作为calamari的默认安装源

```
[uds-ceph]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
```

安装calamari-server与calamari-client

```
yum install -y calamari-server calamari-clients
```

### **4.1.2 初始化calamari**

初始化calamari数据库，以及系统管理员账号

```
calamari-ctl initialize
```

按照提示输入管理员账号，邮箱，密码并完成初始化

注：如果无法完成初始化，则可能由于postgresql数据库未安装，或者postgresql数据库为启动，执行一下命令安装并启动postgresql数据库

```
yum install -y postgresql
systemctl enable postgresql
systemctl start postgresql
```

## 4.2 Calamari Ceph 端配置

Calamari通过saltstack 与 diamond收集MON，MDS，OSD节点的监控数据，通过ceph-deploy配置各个节点

```
ceph-deploy calamari connect --master <master_node_ip> <ceph-node>
 
比如：
ceph-deploy calamari connect --master 192.168.0.2 server-{68,69,70}
```

接受saltstack key

```
salt-key -A
 
并通过一下命令，确认所有Ceph节点的key都已经被Accepted了
salt-key -L
```

触发所有Ceph节点的信息，初始化进calamari master节点的数据库中，执行

```
salt '*' state.highstate
```

