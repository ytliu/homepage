---
layout: page
title: "KVM Notes"
date: 2013-05-26 19:50
comments: true
sharing: true
footer: true
---

Notes about KVM study:

------

## KVM核心基础功能

### CPU配置

-smp n[,maxcpus=cpus][,cores=cores][,threads=threads][,sockets=sockets]

	n：逻辑cpu数量
	cores：每个cpu socket上的core数量
	threads：每个core上的线程数量
	sockets：cpu socket数量

-cpu help：列举qemu支持的cpu模型，默认qemu64 - QEMU Virtual CPU version 1.1.0，可以通过-cpu指定，如：

	-cpu SandyBridge

##### processor affinity

两个步骤：

* 在Linux内核启动是加上`isolcpus=`参数，使得在系统启动后普通进程默认都不会调度到被隔离的cpu上执行。比如，隔离了cpu2和cpu3的grub配置文件：

{% codeblock lang:sh %}
	...
	root=UUID=xxxx isolcpus=2,3
	...
{% endcodeblock %}

系统启动后，检查是否隔离成功：

{% codeblock lang:sh %}
$ ps -eLo psr | grep 2 | wc -l
4
$ ps -eLo psr | grep 3 | wc -l
4
$ ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==2) print $0}'
root 	10	2 	10	2 [migration/2]
root 	11	2 	11	2 [kworker/2:0]
root 	12	2 	12	2 [ksoftirqd/2]
root 	245 2 	245	2 [kworker/2:1]

{% endcodeblock %}
##### 解释

`-e`显示所有进程，`-L`将线程（LWP）显示出来，`-o`按自定义格式输出（`psr`表示分配给进程的处理器编号，`lwp`表示线程ID，`rusr`表示运行进程的用户，`args`表示运行的命令和参数。

migration：用于进程在不同CPU间迁移，kworker：用于处理workqueues，ksoftirqd：用于调度CPU软中断。

* 启动一个拥有两个vCPU的客户机并将它们绑定到host的两个cpu上。

{% codeblock lang:sh %}
# 启动虚拟机
$ qemu-system-x86_64 vm.img -smp 2 -m 512 -daemonize

# 查看代表vCPU的QEMU线程
$ ps -eLo ruser,pid,ppid,lwp,psr,args | grep qemu
root 	3963 	1 	3963 	0 	qemu-system-x86_64 -smp 2 -m 512 vm.img -daemonize
root 	3963 	1 	3967 	0 	qemu-system-x86_64 -smp 2 -m 512 vm.img -daemonize
root 	3963 	1 	3968 	1 	qemu-system-x86_64 -smp 2 -m 512 vm.img -daemonize

# 绑定QEMU进程到cpu2上
taskset -p 0x4 3963

# 绑定第一个vCPU的线程到cpu2上
taskset -p 0x4 3967

# 绑定第二个vCPU的线程到cpu3上
taskset -p 0x8 3968

# 之后可以查看绑定是否生效
$ ps -eLo ruser,pid,ppid,lwp,psr,args | grep qemu
$ ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==2) print $0}'
$ ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==3) print $0}'

{% endcodeblock %}

taskset使用：
	
	taskset -p [mask] pid

### 内存配置

	-m megs

默认值128MB

`-mem-path FILE`参数选项用于使用huge page，`-mem-prealloc`在虚拟机启动时就分配好虚拟机内存

##### 使用huge page

	$ sudo mkdir /dev/hugepages
	$ sudo mount -t hugetlbfs hugetlbfs /dev/hugepages
	$ sysctl vm.nr_hugepages=1024
	$ qemu-system-x86_64 -m 1024 -smp 2 vm.img -mem-path /dev/hugepages

### 存储配置

#### 基本配置

* -hda file (hdb, hdc, hdd)

默认选项，将file镜像作为客户机中的第一个IDE设备（序号0）：`/dev/hda`（如果客户机使用PIIX_IDE驱动）或者`/dev/sda`（如果客户机使用ata_piix）设备。也可以将host的`/dev/sdb`作为`-hda`的file参数来使用。

* -fda file (fdb)

将file镜像作为客户机中的第一个软盘设备（序号0）：`/dev/fd0`。也可以将host的`/dev/fd0`作为`-fda`的file参数来使用。

* -cdrom file

将file作为客户机中的光盘CD-ROM：`/dev/cdrom`。也可以将host的`/dev/cdrom`作为`-cdrom`的file参数来使用。注意，`-cdrom`不能和`-hdc`同时使用，因为`-cdrom`就是客户机中的第三个IDE设备。

* -mtdblock file

将file作为客户机中的一个闪存。

* -sd file

将file作为客户机中的SD卡。

* -pflash file

将file作为客户机中的平行Flash存储器。

#### 详细配置 -drive参数

具体形式：

	-drive option[,option[,option[,...]]]

* file=*file*
* if=interface (ide, scsi, sd, mtd, floopy, pflash, virtio)
* bus=bus, unit=unit (驱动器的总线编号和单元编号)
* index=index（同一接口的驱动器中的索引编号）
* media=media（disk, cdrom）
* snapshot=on|off（on表示QEMU不会把数据的修改写回镜像文件，而是写在一个临时文件中，可以在QEMU monitor中使用“commit”强行写回）
* cache=none|writeback|writethrough（默认为“writethrough”，qcow2最好使用writeback，用writethrough太慢）
* aio=threads|native（native只适用于“cache=none”，即用Linux的原生AIO
* format=format（QEMU会自动检测）
* serial=serial（分配给设备的序列号）
* addr=addr（virtio才适用）
* id=name（驱动器的ID，可以在QEMU monitor中用“info block”看到）
* readonly=on|off（设置驱动器是否只读）

#### 客户机启动顺序设置

	-boot [order=drives],once=drives][,menu=on|off][,splash=splashfile][,splash-time=sp-time]

##### 解释：

* `a`, `b`表示第一和第二个软驱，`c`表示第一个硬盘，`d`表示CD-ROM，`n`表示从网络启动，默认为`c`，要从光盘启动可设置`-boot order=d`。
* `once`表示第一次启动顺序，重启后恢复默认值。
* `menu=on|off`用于设置交互式的启动菜单选项，默认为`off`，需要客户机的BIOS支持。
* `splash=splashfile`和`splash-time=sp-time`，在`menu=on`时，设置BIOS的splash的logo图片splashfile和时间sp-time（ms）

#### qemu-img命令

* check [-f fmt] filename（一致性检查） 

只支持qcow2, qed, vdi格式文件的检查。

* create [-f fmt] [-o options] filename [size]

`-o`可以使用backing_file这个选项来指定后端镜像，其中`-o backing_file=backfile`和`-b backfile`是一样的效果。

* commit [-f fmt] filename (提交修改到后端镜像中（之前通过backing_file指定)
* convert [-c] [-f fmt] [-O output_fmt] [-o options] filename [filename2 [...]] output_filename

将fmt格式的filename镜像根据options选项转换为output_fmt的名为output_filename的镜像文件，其中`-c`表示对输出镜像进行压缩（只支持qcow2和qcow格式文件，并且压缩是只读的）。比如可以把VMWare使用的vmdk格式文件转换为qcow2文件：

	$ qemu-img convert -O qcow2 vm.vmdk vm.qcow2

* info [-f fmt] filename
* snapshot [-l | -a snapshot | -c snapshot | -d snapshot] filename

`-l`表示列出镜像文件中的所有快照；`-a snapshot`表示让镜像文件使用某个快照；`-c snapshot`表示创建一个快照；`-d`表示删除快照。

* rebase [-f fmt] [-t cache] [-p] [-u] -b backing_file [-F backing_fmt] filename

改变镜像文件的后端镜像，只支持qcow2和qed格式文件镜像。

* resize filename [+|-]size

qcow2不支持缩小镜像操作，在增加大小后，需要使用“fdisk”等分区工具进行相应操作。

#### qcow2支持的选项：

* backing_file, backing_fmt
* cluster_size (512B - 2MB)
* preallocation (off|metadata|full)
* encryption (on|off) 

用`qemu-img convert`命令是使用`-o encryption`可对镜像镜像加密，使用加密镜像时，要在QEMU monitor中输入"cont"或"c"命令唤醒客户机输入密码后才能继续执行。

### 网络配置

四种模式：bridge，nat，user mode，vt-d/SR-IOV

默认 `-net nic -net user`

“-net”参数的细节如下：

	-net nic[,vlan=n][,macaddr=mac][,model=type][,name=name][,addr=addr][,vectors=v]

#### 网桥模式

网络参数：

	-net tap[,vlan=n][,name=str][,fd=h][,ifname=name][,script=file][,downscript=dfile][,helper=helper][,sndbuf=nbytes][,vnet_hdr=on|off][,vhost=on|off][,vhostfd=h][,vhostforce=on|off]

* tap：仿真链路层的虚拟网络设备，用于创建一个网络桥；tun：仿真网络层的虚拟网络设备，与路由相关。
* sndbuf=nbytes：限制tap设备的发送缓冲区大小为n字节；默认值为0，即无限制。

#### 设置网桥链接：

* 安装bridge-utils和tunctl；
* 查看tun是否被加载：`lsmod | grep tun`，若没有则加载tun：`modprobe tun`；
* 查看`/dev/net/tun`是否有可读写权限：`ll /dev/net/tun`；
* 建立bridge，修改`/etc/network/interface`：

{% codeblock lang:sh %}
auto br0
iface br0 inet dhcp
	bridge_ports eth0
{% endcodeblock %}

* 准备`my-qemu-ifup`和`my-qemu-ifdown`脚本：

$ cat /etc/my-qemu-ifup

{% codeblock lang:sh %}
#!/bin/bash

# set your bridge name
switch=br0

# $1 is the name of tap: tap1 here
if [ -n "$1"]; then
	# start up the TAP interface
	ip link set $1 up
	sleep 1
	# add TAP interface to the bridge
	brctl addif ${switch} $1
	exit 0
else
	echo "Error: no interface specified"
	exit 1
fi
{% endcodeblock %}

{% codeblock lang:sh %}
#!/bin/bash

# set your bridge name
switch=br0

# $1 is the name of tap: tap1 here
if [ -n "$1"]; then
	# delete TAP interface from bridge
	brctl delif ${switch} $1
	# shutdown the TAP interface
	ip link set $1 down
	sleep 1
	exit 0
else
	echo "Error: no interface specified"
	exit 1
fi
{% endcodeblock %}

* 启动bridge模式的虚拟机

	qemu-system-x86_64 -enable-kvm -drive file=./vm.qcow2 -m 1024 -net nic -net tap,ifname=tap1,script=/etc/my-qemu-ifup,downscript=/etc/my-qemu-ifdown

### 图形显示

#### vnc

* -vnc display,option

display参数选项：

1. host:N（host表示哪台主机可以连接，不填表示任何主机都可以连接；N表示端口号，QEMU会建立相应的TCP端口——`5900+N`）
2. unix:path
3. none（表示VNC已经初始化，但是不在开始时启动，可在QEMU monitor中来启动：`change vnc :2`

options可选项：

1. reverse（反向链接vnc客户端，需要现在客户端开启监听：`vncviewer -listen :2`，这时可以看到开启的端口为5500，虚拟机开启式就需要`-vnc IP:5500,reverse`）
2. password（具体的密码要在QEMU monitor中用change来设置：`change vnc password`）
3. tls, x509=/path/to/certificate/dir, x509verify=/path/to/certificate/dir, sasl, acl（和安全认证相关）

#### 鼠标偏移问题

开启`-usb`和`-usbdevice tablet`选项，前者开启为客户机USB驱动的支持（默认已生效，可省略），后者表示添加一个“tablet”类型的USB设备，其是一个使用绝对坐标定位的指针设备，可以让QEMU能够在客户机不抢占鼠标的情况下获得鼠标的定位信息。

也可以用`-device piix3-usb-uhci -device usb-tablet`来代替`-usb -usbdevice tablet`。

#### 非图形模式

`-nographic`，串口被重定向到当前控制台，grub中需要添加`console=ttyS0`.

------

## KVM高级功能

### 半虚拟化驱动virtio

virtio前端驱动（客户机） -> virtio虚拟队列 -> virtio-ring（环形缓冲区） -> virtio后端驱动（qemu）

#### 安装virtio驱动

* 客户机（linux）

	$ find /lib/modules/2.6.xx/ -name "virtio*.ko"
	$ lsmod | grep virtio

加载顺序：virtio，virtio_ring，virtio_pci，virtio_net/virtio_blk...

* 客户机（windows）

下载virtio-win.iso，通过`-cdrom`加入客户虚拟机。通过如下命令启动虚拟机：

	$ qemu-system-x86_64 win7.img -smp 2 -m 2048 -cdrom /path/to/virtio-win.iso -net nic,model=virtio -net tap -balloon virtio -device virtio-serial-pci

然后在Windows的Device Manager中的Other devices会有3个设备没有合适的驱动，update driver software，找到virtio-win.iso相应的项装上驱动就好了。

而对于virtio_scsi来说，如果没有这个驱动硬盘就起不起来。方法：建立一个非启动盘，设置为使用virtio启动，起来以后装好驱动，重启启动盘即可：

	$ qemu-img create -f qcow2 fake.qcow2 10M
	$ qemu-system-x86_64 win7.img -drive file=fake.qcow2,if=virtio -smp 2 -m 2048 -cdrom /path/to/virtio_win.iso

然后在Windows的Device Manager中的Other devices中有一个没有驱动的”SCSI Controller“，用virtio_win.iso中的镜像装好驱动就好了，然后重启系统：

	$ qemu-system-x86_64 -drive file=win7.img,if=virtio -smp 2 -m 2048

#### 使用virtio_balloon

客户机中存在ballon（客户机不能访问或使用），宿主机请求内存 -> 客户机释放空余内存 -> 客户机空余内存不足 -> 内存换出swap中，气球膨胀 -> 宿主机回收气球中的内存。

KVM ballooning过程：

* KVM发送请求给客户机要求归还部分内存
* 客户机virtio_balloon驱动接收到请求
* virtio_balloon驱动使内存气球膨胀
* 将内存气球给KVM
* KVM将得到的内存分配给其它进程
* 反之，则返回内存气球

##### ballooning使用

* 命令行通过 `-balloon virtio[,addr=addr]`来使得virtio_balloon驱动加载
* 在QEMU monitor中可以使用`info balloon`和`balloon num`来查看和设置客户机内存大小（`balloon num`将客户机内存设置成512，最大只能设置成客户机启动时用`-m`设置的值）
* virsh中有`setmem`命令来动态更改客户机的可用内存

------

## libvirt

#### 下载，编译并安装libvirt源码：
	
	$ download libvirt-1.1.4
	$ ./configure --prefix=/usr
	$ make -j4
	$ make install (默认安装在/usr/local/lib和/usr/local/bin下)

#### libvirt配置文件

* /etc/libvirt/libvirt.conf

用于配置一些常用libvirt链接（通常是远程链接）别名。

{% codeblock lang:sh %}
uri_aliases = [
	"remote1=qemu+ssh://root@192.168.2.58/system",
]
{% endcodeblock %}

用remote1来代替qemu+ssh://root@192.168.2.58/system这个远程libvirt链接：

	$ service libvirtd reload
	$ virsh -c remote1

对应的python代码：

{% codeblock lang:python %}
conn = libvirt.openReadOnly('remote1')
{% endcodeblock %}

* /etc/libvirt/libvirtd.conf

libvirtd的配置文件，每次修改完需要重新加载。

* /etc/libvirt/qemu.conf

libvirt对QEMU的驱动的配置文件，包括VNC等和链接时采用的权限认证方式的配置，也包括内存大页、SELinux、Cgroups等的设置。

* /etc/libvirt/qemu/*

使用QEMU驱动的domain的配置文件。

libvirtd是一个service（/etc/init.d/libvirt-bin）：`service libvirt-bin start|restart|reload|status`

libvirtd直接启动（-d或--daemon | -f或--config | -l或--listen）

#### domain的XML配置文件

* CPU的配置

{% codeblock lang:xml %}
<domain>
	...
	<vcpu placement='static' cpuset="1-4,^3,6" current="1">2</vcpu>
	<features>
		<acpi/>
		...
	</features>
	...
</domain>
{% endcodeblock %}

cpuset表示可以运行在1，2，4，6物理CPU上，current表示启动客户机时只给一个vCPU，最多可以增加到2个vCPU。

还提供`cputune`标签对CPU分配进行更多调节，如：

{% codeblock lang:xml %}
<domain>
	...
	<cputune>
		<vcpupin vcpu="0" cpuset="1"/>
		<vcpupin vcpu="1" cpuset="2,3"/>
		<vcpupin vcpu="2" cpuset="4"/>
		<vcpupin vcpu="3" cpuset="5"/>
		<emulatorpin cpuset="1-3"/>
		<shares>2048></shares>
		<period>1000000</period>
		<quota>-1</quota>
		<emulator_period>1000000</emulator_period>
		<emulator_quota>-1</emulator_quota>
	</cputune>
	...
</domain>
{% endcodeblock %}

`emulatorpin`表示QEMU绑定到1~3号物理CPU上；`shares`表示客户机所占用CPU的时间加权配额，2048配额的domain会得到CPU执行时间为1024配额的两倍。

* 内存的配置

{% codeblock lang:xml %}
<domain>
	...
	<memory unit='KiB'>1048576</memory>
	<currentMemory unit='KiB'>1048576</currentMemory>
	...
	<device>
		...
		<memballoon model='virtio'>
			<address type='pci' domain='0x0000' bus='ox00' slot='0x06' function='0x0'/>
		</memballoon>
		...
	</device>
	...
</domain>
{% endcodeblock %}

`memory`为最大值，`currentMemory`为客户机启动时的值。

* 客户机系统类型和启动顺序

{% codeblock lang:xml %}
<domain>
	...
	<os>
		<type arch='x86_64'>hvm</type>
		<boot dev='hd'/>
		<boot dev='cdrom'/>
	...
</domain>
{% endcodeblock %}

* disk的配置

* network的配置

------
