# 在Qemu中使用块设备

## 通过libvirt使用块设备

libvirt 已经成为使用最为广泛的对各种虚拟机进行管理的工具和应用程序接口（API），而且一些常用的虚拟机管理工具（如virsh、virt-install、virt-manager等）和云计算框架平台（如OpenStack、OpenNebula、Eucalyptus等）都在底层使用libvirt的应用程序接口。它提供了对虚拟化客户机和它的虚拟化设备、网络和存储的管理。

目前UDS的块存储也支持通过libvirt来管理块设备。这里假设您的虚机已经创建完成，您只需要按照以下步骤对虚机进行配置后，就可以在虚机中使用Ceph的块设备：

1 打开虚机的配置文件

```
# virsh edit {vm-name}
```

2 在配置文件中找到devices目录，添加以下信息并保存

```
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none'/>
      <source protocol='rbd' name='rbd/rbd_disk'>
        <host name='192.168.0.101' port='6789'/>
      </source>
      <backingStore/>
      <target dev='vdb' bus='virtio'/>
    </disk>
  <devices>
```

* source protocol中name指定的是镜像对应的{poolname}\/{imagename}
* host name，定义的是集群中Monitor的地址。
* dev 属性是将出现在 VM \/dev 目录下的逻辑设备名。可选的bus 属性是要模拟的磁盘类型，有效的设定值是驱动类型，如 ide 、 scsi 、 virtio 、 xen 、 usb 或 sata 。

3 如果在UDS集群中启用了Ceph认证（默认已启用），则还需要生成并定义secret。

生成一个secret文件，内容如下：

```
# cat secret.xml
<secret ephemeral='no' private='no'>
        <usage type='ceph'>
                <name>client.admin secret</name>
        </usage>
</secret>
```

定义secret

```
# virsh secret-define --file secret.xml
Secret 408abbd2-fce3-45f4-a061-6f237a3ed5e7 created
```

获取client.admin的密钥

```
# ceph auth get-key client.admin | sudo tee client.admin.key
```

根据上面生成的secret的UUID进行设置

```
# sudo virsh secret-set-value --secret 408abbd2-fce3-45f4-a061-6f237a3ed5e7 --base64 $(cat client.libvirt.key) 
```

使用virsh编辑虚拟机的配置文件，加入认证信息

```
<devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='network' device='disk'>
	  ...
      <auth username='admin'>
        <secret type='ceph' uuid='408abbd2-fce3-45f4-a061-6f237a3ed5e7'/>
      </auth>
      ...
    </disk>
 <devices>
```

4 以上配置完成后，就可以启动虚机了。虚机启动之后，可以执行如下过程确认使用是否正常。

检查Ceph集群是否正常

```
# ceph health detail
```

检查vm是否正常运行

```
# virsh list
3   instance-0002949b              running
```

检查vm是否和ceph通信

```
$ virsh qemu-monitor-command --hmp instance-0002949b 'info block'
drive-virtio-disk0: rbd:rbd/rbd_disk:id=admin:key=***:auth_supported=cephx\;none:mon_host=10.1.0.61\:6789 (raw)
drive-ide0-1-1: /var/lib/nova/instances/f4e1851f-acb2-4856-905f-6e6ada5a21aa/disk.config (raw)
    Removable device: locked, tray closed
```

登陆到vm之后，确认块设备正确的挂载到了虚拟机。

