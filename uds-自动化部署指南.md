# 3 UDS 自动化部署指南

## 3.1 部署规划

在实际的部署过程中，需要选择一下部署方式，网络规划，和角色。不同的部署方式将导致不同的部署行为。网络规划对于后续的性能和稳定性也是有很大影响的。如果条件允许请使用两个万兆网卡。

### **3.1.1 部署方式**

目前部署方式可以有Puppet和Ceph-deploy, 目前生产环境中主要使用Puppet来进行部署。Ceph-deploy的部署方式主要是为了满足快速验证的需求。一般推荐在生产环境中使用Puppet来部署，这样方便后续环境的维护。而且Ceph-deploy如果操作不当具有很强的破坏性，如purge，所以除非你非常清除自己在做什么。另外随着Ceph-deploy的越来越完善，osd等系统逐渐将配置信息转移到osd自身，使用Ceph-deploy部署一个Ceph集群变的越来越方便，后面很有可能在受限生产集群（比如离线）上尝试使用Anisble组合Ceph-deploy来部署。

### **3.1.2 网络规划**

Ceph的网络分为public-nework，cluster-network，所以需要规划一下，如果成本允许，可以使用两块万兆。一块用来作为public-network，一块用来作为cluster-network。如果成本紧张，则可以使用一块万兆，这样public-network和cluster-network都使用同一块网卡。

![](/assets/117ACDFC-26C1-4DB9-95F7-C2AAAC9A71ED.png)

### **3.1.3 角色规划**

UDS Ceph中，服务器有以下角色，MON，OSD，MDS，RGW。其中MON主要存储集群中的状态，维护状态的一致性。OSD主要是用来存储数据。MDS作为CEPHFS的客户端，用来维护文件系统的元数据，实际的数据也是存储在OSD中，只是以不同的POOL来存储。RGW是将librados以对象存储的接口导出来。

MON节点的个数在生产中一般为3个。OSD节点一般为3的倍数。MDS目前只支持单活多从。RGW支持多个做负载均衡。在角色定义好之后，最后输出文档，用来指导后续的操作。

```
server-61 mon
server-62 mon
server-63 mon
server-68 osd mds
server-69 osd mds
server-70 osd mds
server-71 osd rgw
server-72 osd rgw
server-73 osd rgw
```

## 3.2 通过puppet部署UDS

Puppet的部署需要有Puppet Master和Puppet Client。其中Puppet Master用于来存储\*.pp文件与hieradata，这样只要我们将角色定义好，直接运行Puppet就可以完成部署，可以减少大量的重复工作。

### **3.2.1 安装puppetmaster和puppet**

#### **3.2.1.1 配置puppet master节点**

通常我们建议将Puppet master部署在控制节点，但也可以部署在其他的节点上，如OSD或者MON节点，选择一个节点作为Puppet master，执行如下操作

安装软件包

```
yum install -y puppet-server puppet
```

安装uds-suite，包含了puppet 部署UDS Ceph所需要的puppet模块

```
yum install http://uds.ustack.com/repo/tools/uds-suite-1.0.0-1.el7.centos.uds.noarch.rpm
```

编辑\/etc\/puppet\/puppet.conf，vim \/etc\/puppet\/puppet.conf

    ### File managed with UnitedStack puppet ###
    ## Module: 'puppet'
    ## Template source: 'MODULES/puppet/templates/puppet.conf.erb'
    [main]
      # The Puppet log directory.
      # The default value is '$vardir/log'.
      logdir = /var/log/puppet
      # Where Puppet PID files are kept.
      # The default value is '$vardir/run'.
      rundir = /var/run/puppet
      # Where SSL certificates are kept.
      # The default value is '$confdir/ssl'.
      ssldir = $vardir/ssl
      # Allow services in the 'puppet' group to access key (Foreman + proxy)
      privatekeydir = $ssldir/private_keys { group = service }
      hostprivkey = $privatekeydir/$certname.pem { mode = 640 }
      # Puppet 3.0.x requires this in both [main] and [master] - harmless on agents
      autosign = $confdir/autosign.conf { mode = 664 }
      stringify_facts = true

    [agent]
      # The file in which puppetd stores a list of the classes
      # associated with the retrieved configuratiion. Can be loaded in
      # the separate ``puppet`` executable using the ``--loadclasses``
      # option.
      # The default value is '$confdir/classes.txt'.
      classfile = $vardir/classes.txt
      # Where puppetd caches the local configuration. An
      # extension indicating the cache format is added automatically.
      # The default value is '$confdir/localconfig'.
      localconfig = $vardir/localconfig
      report = true
      pluginsync = true
      masterport = 8140
      # 这里请填写server的fqdn， 比如server = server-68.3.dev3.ustack.in
      server = {{ ansible_fqdn }}
      listen = false
      splay = true
      runinterval = 9600
      noop = false
      configtimeout = 1200
    ### Next part of the file is managed by a different template ###
    ## Module: 'puppet'
    ## Template source: 'MODULES/puppet/templates/server/puppet.conf.erb'
    [master]
      parser = future
      autosign = $confdir/autosign.conf { mode = 664 }
      ca = true
      ssldir = /var/lib/puppet/ssl
      basemodulepath = $confdir/modules/production:/etc/puppet/modules/common:/usr/share/puppet/modules

配置autosign.conf

```
echo '*' > /etc/puppet/autosign.conf
chmod 0664 /etc/puppet/autosign.conf
```

编辑hiera.yaml，如果没有需要在\/etc\/puppet\/目录下先创建，vim \/etc\/puppet\/hiera.yaml

```
---
:backends:
  - yaml
:hierarchy:
  - "%{::domain}/%{::hostname}"
  - "%{::domain}/common/ceph"

:yaml:
# datadir is empty here, so hiera uses its defaults:
# - /var/lib/hiera on *nix
# - %CommonAppData%\PuppetLabs\hiera\var on Windows
# When specifying a datadir, make sure the directory exists.
  :datadir: /etc/puppet/hieradata
```

根据前面的规划，需要在相应域下的host的yaml中配置相关角色，如：

* mon角色的配置

\/etc\/puppet\/hieradata\/[3.dev3.ustack.in](http://3.dev3.ustack.in/)\/server-61.yaml \# mon 节点的hieradata

```
uds::ceph::cluster::enable_mon: true
```

* osd角色的配置

\/etc\/puppet\/hieradata\/[3.dev3.ustack.in](http://3.dev3.ustack.in/)\/server-68.yaml \# osd 节点的hieradata其中disk\_type为ssd, sata, 或者mix

```
uds::ceph::cluster::enable_osd: true
uds::ceph::cluster::disk_type: ssd
uds::ceph::cluster::osd_device_dict:
 "wwn-0x55cd2e404bcdf711": ""
 "wwn-0x55cd2e404bcdee9a": ""
 "wwn-0x55cd2e404bcded26": ""
```

这里分为ssd，sata，mix主要是因为不同的磁盘类型，其优化参数也是不同的。mix代表的是ssd做journal盘，sata做数据盘。osd\_device\_dict接收的是一个map，其中key是数据盘，value是journal盘。如果像上面的例子，journal为空，则puppet会在数据盘上分割一块数据来做为journal盘，默认大小5G。当然如果是想自己定制化，这里可以自己做分区。只要这里osd\_device\_dict输入正确的数据盘和journal盘就可以了。

#### **3.2.1.2 配置agent 节点**

除了puppet master之外的节点作为puppet agent节点（如果想提高节点的利用率，puppetmaster也可以作为agent节点，即自己管理自己）

```
yum install -y puppet
```

其配置如下 

    ### File managed with UnitedStack puppet ###
    ## Served by: 'server-205.0.uc.ustack.in'
    ## Module: 'puppet'
    ## Template source: 'MODULES/puppet/templates/puppet.conf.erb'
    [main]
      # The Puppet log directory.
      # The default value is '$vardir/log'.
      logdir = /var/log/puppet
      # Where Puppet PID files are kept.
      # The default value is '$vardir/run'.
      rundir = /var/run/puppet
      # Where SSL certificates are kept.
      # The default value is '$confdir/ssl'.
      ssldir = $vardir/ssl
      # Allow services in the 'puppet' group to access key (Foreman + proxy)
      privatekeydir = $ssldir/private_keys { group = service }
      hostprivkey = $privatekeydir/$certname.pem { mode = 640 }
      # Puppet 3.0.x requires this in both [main] and [master] - harmless on agents
      autosign = $confdir/autosign.conf { mode = 664 }

    [agent]
      # The file in which puppetd stores a list of the classes
      # associated with the retrieved configuratiion. Can be loaded in
      # the separate ``puppet`` executable using the ``--loadclasses``
      # option.
      # The default value is '$confdir/classes.txt'.
      classfile = $vardir/classes.txt
      # Where puppetd caches the local configuration. An
      # extension indicating the cache format is added automatically.
      # The default value is '$confdir/localconfig'.
      localconfig = $vardir/localconfig
      report = true
      pluginsync = true
      masterport = 8140
      # 这里请写puppetmaster的 fqdn, 比如：server ＝ server-master.3.dev3.ustack.in
      server = host_fqdn 
      listen = false
      splay = true
      runinterval = 1800
      noop = false
      configtimeout = 1200

### **3.2.2 修改site.pp 文件**

在Puppet master，Puppet agent都配置好之后，我们需要将角色信息持久化一下，这样在Agent端执行puppet agent -vt的时候，Master才知道Agent是什么角色。

在Puppet master节点上编辑文件 \/etc\/puppet\/manifest\/site.pp

```
# 每个节点都需要，但是其功能的开启是在hieradata中
node /host_fqdn/{
      class {"uds::ceph::cluster": }
}
 
比如：
node /server-[68-70].ustack.com/{
      class {"uds::ceph::cluster": }
}
```

### **3.2.3 部署MON**

在site.pp和hieradata中都有MON节点的信息之后，通过clush在MON节点上执行puppet agent -vt，验证是否ok，请运行，看到mon是正常的。

```
ceph -s
```

### **3.2.4 部署OSD**

在site.pp 和hieradata中都有OSD的信息之后，通过clush在OSD节点上执行puppet agent -vt，就可以完成OSD的部署。验证是否ok，请运行

```
ceph osd tree
```

### **3.2.5 建立CRUSH**

通过运行[launchcrush.py](https://confluence.ustack.com/download/attachments/16113054/launchcrush.py?version=1&modificationDate=1466579893441&api=v2)脚本来完成crush的初始化。

获取CRUSH脚本：wget [http:\/\/uds-repo.ustack.com\/tools\/launchcrush.py](http://uds-repo.ustack.com/tools/launchcrush.py)

需要添加如下配置，将其保存成config.yaml，launchcrush.py和config.yaml.sample可以从

```
# YAML
# the number of the osd
osds: 27
# the osd weight, must be string, You must ask Rongze
# if the ssd is 480 GB, journal size is 15GB, the osd weight is '0.42'
osd_weight: '0.42'
mons:
  - 10.1.0.61
  - 10.1.0.62
  - 10.1.0.63
# the management network
racks:
  rack-01:
    - 10.1.0.70
    - 10.1.0.71
    - 10.1.0.72
  rack-02:
    - 10.1.0.76
    - 10.1.0.77
    - 10.1.0.78
  rack-03:
    - 10.1.0.82
    - 10.1.0.83
    - 10.1.0.84

# 保存为config.yaml 然后运行
launchcrush.py config.yaml
```

## 3.3 总结

通过puppet部署，需要写yaml文件（hieradata），方便状态的维护和运维，我们推荐在生产环境的部署与实施中使用。

