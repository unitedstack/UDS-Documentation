# 3 UDS 自动化部署指南

## 3.1 部署规划



在实际的部署过程中，需要选择一下部署方式，网络规划，和角色。不同的部署方式将导致不同的部署行为。网络规划对于后续的性能和稳定性也是有很大影响的。如果条件允许请使用两个万兆网卡。

### **3.1.1 部署方式**

目前部署方式可以有Puppet和Ceph-deploy, 目前生产环境中主要使用Puppet来进行部署。Ceph-deploy的部署方式主要是为了满足快速验证的需求。一般推荐在生产环境中使用Puppet来部署，这样方便后续环境的维护。而且Ceph-deploy如果操作不当具有很强的破坏性，如purge，所以除非你非常清除自己在做什么。另外随着Ceph-deploy的越来越完善，osd等系统逐渐将配置信息转移到osd自身，使用Ceph-deploy部署一个Ceph集群变的越来越方便，后面很有可能在受限生产集群（比如离线）上尝试使用Anisble组合Ceph-deploy来部署。

### **3.1.2 网络规划**

Ceph的网络分为public-nework，cluster-network，所以需要规划一下，如果成本允许，可以使用两块万兆。一块用来作为public-network，一块用来作为cluster-network。如果成本紧张，则可以使用一块万兆，这样public-network和cluster-network都使用同一块网卡。



fdsg gsgs

