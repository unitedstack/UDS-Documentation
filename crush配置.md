# 5 CRUSH配置

## 5.1 CRUSH介绍

Ceph中通过CRUSH算法来计算数据的存储位置，而非从一个（多个）中心节点查询数据的存放位置。CRUSH算法的计算需要两个输入：CRUSH map和CRUSH rule。

CRUSH map维护了集群内OSD的组织方式，这种组织方式以树型结构存在，所有OSD都位于树的叶子节点，而树的非叶子节点被称为桶（Bucket）。CRUSH rule定义了一系列操作。CRUSH map好比是一张帮助CRUSH寻找OSD的地图，并且通过这张地图能够有很多种方法找到OSD，而CRUSH rule就是定义好的寻找OSD的一种方法。

从集群中将CRUSH map dump出来之后，可以发现它主要包括5个段落

* tunables
* 设备，由对象存储设备组成，即对应一个 ceph-osd 进程的存储器。 Ceph 配置文件里的每个 OSD 都应该有一个设备。
* 桶类型，定义了 CRUSH 分级结构里要用的桶类型（types ），桶由逐级汇聚的存储位置（如行、机柜、机箱、主机等等）及其权重组成。
* 桶及其层级结构，集群中已经定义好的Bucket。
* rule，选择OSD的方法。

**tunables**： 定义了CRUSH算法暴露出来的一些可调参数，这个一般不需要进行修改。

**设备**： 为把归置组映射到 OSD ， CRUSH 图需要 OSD 列表（即配置文件所定义的 OSD 守护进程名称），所以它们首先出现在 CRUSH 图里。要在 CRUSH 图里声明一个设备，在设备列表后面新建一行，输入 device 、之后是唯一的数字 ID 、之后是相应的ceph-osd 守护进程例程名字。例如：

```
# devices
device 0 osd.0
```

**桶类型**： CRUSH 图里的第二个列表定义了 bucket （桶）类型，桶简化了节点和叶子层次。节点（或非叶子）桶在分级结构里一般表示物理位置，节点汇聚了其它节点或叶子，叶桶表示 ceph-osd 守护进程及其对应的存储媒体。默认情况下，集群中会定义以下类型的bucket。

```
# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root
```

您也可以添加您自定义的bucket类型。

**桶及其层级结构**：该部分显示了集群中已经创建出来的bucket，并且这些bucket按树型层级结构存在。引入桶层级结构的目的是按故障域隔离叶节点，像主机、机箱、机柜、电力分配单元、机群、行、房间、和数据中心。除了表示叶节点的 OSD ，其它分级结构都是任意的，您可以按需定义。

```
# buckets
host ceph-02 {
        id -2           # do not change unnecessarily
        # weight 0.020
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 0.020
}
host ceph-01 {
        id -3           # do not change unnecessarily
        # weight 0.030
        alg straw
        hash 0  # rjenkins1
        item osd.3 weight 0.030
}
host ceph-03 {
        id -4           # do not change unnecessarily
        # weight 0.030
        alg straw
        hash 0  # rjenkins1
        item osd.4 weight 0.030
}
root default {
        id -1           # do not change unnecessarily
        # weight 0.080
        alg straw
        hash 0  # rjenkins1
        item ceph-02 weight 0.020
        item ceph-01 weight 0.030
        item ceph-03 weight 0.030
}
```

* id，bucket的编号，除了OSD的id为正数之外，所有bucket的id均为负数。
* weight，该bucket的权重。CRUSH在选择bucket时，是根据bucket的权重来决定的，权重越大被选中的概率也越大。
* alg，桶的类型。Ceph目前支持四种桶，分别是Uniform、List、Tree和Straw。每种都是性能和组织简易间的折衷，如果您不确定用哪种桶，我们建议 straw。
* hash，每个桶都采用了一种哈希算法，当前ceph仅支持rjenkins1。
* item，表示这个桶里包含的内容。

**rule**：CRUSH 规则定义了数据归置和复制策略、或分布策略，用它可以规定 CRUSH 如何放置数据。通过它能够从层级结构的桶中选出对应的OSD。例如，你也许想创建一条规则用以选择一对目的地做双路复制；另一条规则用以选择位于两个数据中心的三个目的地做三路镜像；又一条规则用 6 个设备做纠删编码。

```
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
```

* ruleset，规则的编号
* type，规则应用于副本策略或是纠删码策略
* min\_size，如果一个归置组副本数小于此数， CRUSH 将**不**应用此规则。
* max\_size，如果一个归置组副本数大于此数， CRUSH 将**不**应用此规则。
* step take &lt; bucket name&gt;，根据桶名，选择对应的子树。
* step chooseleaf firstn {num} type {bucket-type}，选择 {bucket-type} 类型的一堆桶，并从各桶的子树里选择一个叶子节点。集合内桶的数量通常是存储池的副本数（即 pool size ）。如果num=0，则选择副本数个，如果num &gt; 0，选择num个桶，如果num &lt; 0， 就选择 副本数 － num个。
* step emit，输出当前值并清空堆栈。通常用于规则末尾，也适用于相同规则应用到不同树的情况。

CRUSH map具体是应用在Ceph的存储池中的，通过设置存储池的crush\_ruleset来完成。

```
# ceph osd pool set <poolname> crush_ruleset 0
```

通过合理的设计CRUSH map能够实现以不同的故障域进行数据容灾，保证数据的安全和可用。UDS的CRUSH map是经过精心设计的，能够实现跨机柜容灾。对于普通采用3副本机制的集群来说，能够保证在挂掉两个机柜之后，UDS仍能够对外提供存储服务。

## 5.2 编辑CRUSH map

编辑CRUSH map可以通过两种方式完成，一种是完全通过命令行完成，另一种是通过编辑CRUSH map文件完成。对于编辑文件的方式，需要完成以下几步

* 获取当前CRUSH map文件
* 反编译map成可编辑文件
* 修改内容
* 重新编译CRUSH map
* 往集群中注入CRUSH map

### **5.2.1 编辑CRUSH map文件**

1）获取当前CRUSH map文件

```
# ceph osd getcrushmap -o map.bin
```

2） 反编译成可编辑文件

```
# crushtool -d map.bin -o map.txt
```

3）编辑文件，比如编辑一个设备、桶或者规则

4）重新编译

```
# crushtool -c map.txt -o newmap.bin
```

如果编译出错，需要检查修改项是否合法。

5）往集群中注入CRUSH map

```
# ceph osd setcrushmap -i newmap.bin
```

设置成功之后，集群就应用了新的CRUSH map。

### **5.2.2 通过命令编辑行修改CRUSH map。**

目前通过命令行只能完成bucket和OSD的添加、删除、移动以及相应权重的调整。

1）增加bucket或OSD

```
# ceph osd crush add-bucket {bucket-name} {bucket-type}
# ceph osd crush add {id} {name} {weight} [{bucket-type}={bucket-name} ...]
```

例如添加一个host 类型的bucket，并在其中加入一个OSD

```
# ceph osd crush add-bucket ceph-05 host
added bucket ceph-05 type host to crush map
# ceph osd crush add osd.3 0.03 host=ceph-05
add item id 3 name 'osd.3' weight 0.03 at location {host=ceph-05} to crush map
```

2）移动一个bucket和OSD

```
# ceph osd crush move {bucket-name} {bucket-type}={bucket-name}, [...]
# ceph osd crush set {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
```

例如，将ceph-05移动到default的bucket下面，将osd.3移动到 ceph-03下面

```
# ceph osd crush move ceph-05 root=default
moved item id -5 name 'ceph-05' to location {root=default} in crush map
# ceph osd crush set osd.3 0.03 host=ceph-03
set item id 3 name 'osd.3' weight 0.03 at location {host=ceph-03} to crush map
```

3）删除bucket和OSD

```
# ceph osd crush remove {bucket-name}
# ceph osd crush remove {osd-name}
```

例如，删除掉ceph-05和osd.3

```
# ceph osd crush remove osd.3
removed item id 3 name 'osd.3' from crush map
# ceph osd crush remove ceph-05
removed item id -5 name 'ceph-05' from crush map
```

注意，在删除bucket之前，它必须不能包含其他内容。

4）调整bucket和OSD的权重

```
# ceph osd crush reweight {name} {weight}
```

例如，调整osd.3的权重，

```
# ceph osd crush reweight osd.3 0.03
reweighted item id 3 name 'osd.3' to 0.03 in crush map
```

