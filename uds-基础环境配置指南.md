# 1 UDS 基础环境配置指南

UDS Ceph是一个分布式的系统，其具有弹性扩展，统一接口的特性，能够提供PB级别的存储的能力。目前集群的角色有OSD节点，MON节点，RGW节点，MDS节点，各司其职，对外提供一个整体的集群视角。

一个可用的UDS Ceph集群至少需要有MON与OSD角色，通常情况下，我们推荐为每块磁盘部署一个OSD进程，进行管理，MON进程可与OSD进程部署在相同的服务器上，也可以分开在不同服务器上。而是否部署MDS与RGW取决于是否需要使用CephFS与RGW Gateway服务，MDS与RGW服务的部署推荐使用单独的物理节点（高性能，大内存）。

部署UDS Ceph之前，需要准备好具备UDS Ceph各个服务部署的最小环境。我们推荐用户在CentOS 7 的操作系统上部署 UDS Ceph，UDS Ceph 发布版本 Azeroth 为最后一个支持CentOS 6，CentOS 6.5 操作系统的版本，在这些操作系统上部署UDS Ceph将会增加后续UDS Ceph跨版本升级的复杂度。

本部署指南将基于CentOS 7版本，使用CentOS 6 与 CentOS 6.5进行UDS Ceph的安装部署类似，请读者自行尝试。在部署UDS之前，你需要将节点准备好，如果首次安装，请按照如下步骤来完成环境的准备。

## 1.1 操作系统

UDS Ceph Azeroth发布版本支持操作系统：CentOS 6.x， CentOS 7.x —— 我们推荐使用 CentOS 7.x 的操作系统系统

为系统盘进行分区

| **Boot分区** | **1024M** |
| :--- | :--- |
| Swap分区 | 4096M |

剩余空间划分为LVM，PV名称： ustack\_pv

| **分区** | **名称** | **大小** |
| :--- | :--- | :--- |
| \/ | rootvol | 20G |
| \/home | homevol | 5G |
| \/var | varvol | 剩余所有空间 |

## 1.2 Repo准备

UDS Ceph Azeroth基于Ceph 0.94.7的官方版本进行构建，UDS Ceph Azeroth 提供在线安装与离线安装两种方式

在线安装

如果是在线安装请添加UDS的源

```
[uds-ceph]
name=UDS Packages for CentOS 7
baseurl=http://uds.ustack.com/repo/Azeroth/el7/
enabled=1
gpgcheck=0
priority=1
```

离线安装

需要带着UnitedStack自己的Repo。具体可以从http:\/\/uds.ustack.com\/repo\/下载。

## 1.3 DNS

Ceph节点除了有fqdn之外，还必须能够通过短域名来进行解析，通过如下方式来验证短域名

```
hostname -s
```

## 1.4 网卡

UDS Ceph节点至少需要一张网卡来组成public network 和 cluster network，实现Client对UDS Ceph集群的访问以及集群内部Ceph的数据复制，数据恢复，数据平衡和数据校验。我们建议使用两张网卡，一张用于public network，一张用于cluster network。

## 1.5 网络

请确保对网卡的配置都是持久化的，请检查 \/etc\/sysconfig\/network-scripts，并确保 ifcfg-&lt;iface&gt; 对 public-network 和 cluster-network 的配置都正确的配置好了。

例如，public-network规划的网段是10.10.1.0\/24，cluster-network规划的网段是10.10.16.0\/24，可以这么写

eth0：

```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOTPLUG=yes
TYPE=Ethernet
IPADDR=10.10.1.70
NETMASK=255.255.255.0
GATEWAY=10.10.1.1
PEERDNS=no
NM_CONTROLLED=no
```

eth2：

```
DEVICE=eth2
BOOTPROTO=none
ONBOOT=yes
HOTPLUG=yes
TYPE=Ethernet
IPADDR=10.10.16.70
NETMASK=255.255.255.0
GATEWAY=10.10.16.1
PEERDNS=no
NM_CONTROLLED=no
```

配置好网络后，确认网络配置正确，可以通过如下步骤来检测检查连接是否有效

```
~]# ethtool eth2
Settings for eth2:
Supported ports: [ FIBRE ]
Supported link modes: 1000baseT/Full 
10000baseT/Full 
Supported pause frame use: No
Supports auto-negotiation: Yes
Advertised link modes: 1000baseT/Full 
10000baseT/Full 
Advertised pause frame use: No
Advertised auto-negotiation: Yes
Speed: 10000Mb/s                       ＃网络速度正确匹配
Duplex: Full
Port: FIBRE
PHYAD: 0
Transceiver: external
Auto-negotiation: on
Supports Wake-on: umbg
Wake-on: g
Current message level: 0x00000007 (7)
drv probe link
Link detected: yes                     ＃网络物理连接正常
```

## 1.6 防火墙

关闭防火墙

```
# systemctl stop firewalld.service
# systemctl disable firewalld.service
```

## 1.7 NTP

NTP可以将系统时间与公网NTP服务器或内网NTP服务器进行同步，我们推荐使用内网NTP服务器进行时间同步，以减小服务器间同步时间的误差 。我们建议使用一个内网节点作为内部ntp服务器，将该节点时间与更高一级ntp服务器进行时间同步，如：公网服务器，并向内网节点提供ntp时间同步服务。

搭建私有NTP服务器，具体步骤如下ntp

```
# yum install -y ntp
```

NTP安装完成后，编辑\/etc\/ntp.conf配置文件

```
＃ vim /etc/ntp.conf
```

修改配置文件ntp.conf，其中server部分定义了向更高一级NTP服务器同步时间，这边使用的为 2.cn.pool.ntp.org ，1.asia.pool.ntp.org ， 2.asia.pool.ntp.org ，可自行修改为最合适的NTP服务器。并且通过restrict关键字，允许某个子网网段进行时间同步（这里使用的ntp服务器ip地址为192.168.0.9，并且允许192.168.0.0\/24 网段与该服务器同步时间，请根据实际情况自行修改），一个典型的配置文件配置如下：

```bash
# For more information about this file, see the man pages 
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).  

driftfile /var/lib/ntp/drift

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery

# Permit all access over the loopback interface. This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1 restrict ::1

# Hosts on local network are less restricted.
restrict 192.168.0.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 2.cn.pool.ntp.org iburst
server 1.asia.pool.ntp.org iburst
server 2.asia.pool.ntp.org iburst
server 127.127.1.0 # local clock
fudge 127.127.1.0 stratum 10

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor
```

配置文件修改完成后，重启NTP服务，并且将其设置为开机启动

```
# systemctl enable ntpd 
# systemctl start ntpd
```

配置内网节点与NTP服务器（192.168.0.9）同步时间

```
# yum install -y ntp
```

安装完成后，有两种方式实现与NTP服务器（192.168.0.9）的时间同步

方式一：

修改配置文件ntp.conf，将时间同步服务器指向192.168.0.9，一个典型的配置文件如下：



