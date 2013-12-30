---
layout: post
title: "Netfilter学习笔记（四）"
date: 2013-10-04 11:39
comments: true
categories: Network
keywords: nat iptables
---

前几天在看kvm网络相关的东西，看到nat，就上网搜了下相关资料，看到[鸟哥的linux私房菜](http://linux.vbird.org/)关于[防火墙和nat](http://linux.vbird.org/linux_server/0250simple_firewall.php)的介绍，里面有很详细的对netfilter和iptables的介绍，感觉学习了之后又有了更深的了解，特别是对除了[之前](http://ytliu.info/blog/2013/03/29/netfilterxue-xi-bi-ji-%28yi-%29/)介绍过的`filter`之外的另外一个`nat`表的了解。以下的内容基本上全部摘录自[这篇博文](http://linux.vbird.org/linux_server/0250simple_firewall.php)，并进行了一些整理：

我觉得最能够说明`nat`表和`filter`表的关系的是鸟哥整理的一张图：

![nat filter](http://ytliu.info/images/2013-10-04-1.png "iptalbes")

从上面的图中可以看出如果只是考虑`filter`和`nat`两个表的话，iptables可以控制三种网络封包的流向：

* 路径A：进入主机的封包，在路由判断后确定是向本主机发送的封包，主要就会透过`filter`的`INPUT`链来进行控管；
* 路径B：将本主机作为防火墙/代理的封包，在路由判断之前进行封包表头的修改之后，发现该封包主要是透过防火墙/代理而去后端主机，此时封包就会走路径B。也就是说，该封包的目标并不是我们的主机，而是要经过`filter`的`FORWARD`，以及`nat`的`POSTROUTING`链。
* 路径C：从本主机发送出去的封包，例如回应用户端的要求，或者是本机主动发送出去的封包，先是经过路由判断，决定输出的路径后，再经由`filter`的`OUTPUT`，以及`nat`的`POSTROUTING`链传送出去的。

也就是说，如果我们要处理到达本机或者从本机发出的网络包，那么我们主要是要关心`filter`表的`INPUT`和`OUTPUT`两条链，而如果我们要在另外一台主机上搭建一个防火墙，或者代理，那么我们就需要对`filter`表的`FORWARD`，以及`nat`表的`PREROUTING`和`POSTROUTING`进行配置。

<!-- more -->

------

接下来就是对`nat`用法的一些介绍：

NAT的全名是Network Address Translation，即网络地址翻译。在开始说NAT之前，我们先重新回顾一下比较简单的封包经过iptables传送到后端主机的表与链流程：

* 先经过`nat`表的`PREROUTING`链；
* 经由路由判断确定这个封包是要进入本机与否，若不进入本机，则下一步；
* 再经过`filter`表的`FORWARD`链；
* 通过`nat`表的`POSTROUTING`链，最后传送出去。

而NAT的重点就在于上面流程的第1、4两个步骤，也就是`nat`表的两条重要的链：`PREROUTING`与`POSTROUTING`。这两条链的关键在于修改包头的IP。但是这两条链修改的IP是不一样的：`POSTROUTING`修改的是来源IP，而`PREROUTING`修改的则是目标IP。 由于修改的IP 不一样，所以就称为来源NAT（Source NAT，SNAT）及目标NAT（Destination NAT，DNAT）。

我们以下图中的配置为实例，来说明SNAT和DNAT分别是如何工作的：

![nat](http://ytliu.info/images/2013-10-04-2.png "nat")

在上图中，linux主机可以作为一个防火墙，将内网与外网进行分离，并控制和过滤网络封包的进出；另外，它也可以作为内网各个主机的网关主机，可以管理内网中的主机与外界互联网的交互。它一般需要两个网络介面（比如eth1负责内网，地址为`private IP` —— 192.168.1.2，eth0负责外网，地址为`public IP`）。那么SNAT和DNAT又是什么呢？

*SNAT*：修改封包表头的”来源“项。它可以让家里好几台主机同时透过一条ADSL网路连线到Internet上面；或者对于实验室来说一般只有一个公网IP，而实验室的所有主机连接外网，就需要通过SNAT，即通过`nat`表的`POSTROUTING`来处理的。假设你的网路布线如下图所示， 那么NAT主机是如何处理这个封包的呢？

![snat 1](http://ytliu.info/images/2013-10-04-3.png "snat 1")

如上图所示，在用户端`192.168.1.100`这部主机要连线到`http://tw.yahoo.com`去时，他的封包表头会如何变化？  

* 用户端所发出的封包表头中，来源地址是`192.168.1.100`，然后传送到NAT主机；
* NAT主机的内部介面（eth1，地址`192.168.1.2`）接收到这个封包后，会主动分析表头资料，因为表头资料显示目的并非本本机，所以开始经过路由，将此封包转到可以连接到 Internet 的`public IP`处；
* 由于`private IP`与`public IP`不能互通，所以NAT主机透过iptables的`nat`表内的`POSTROUTING`链将封包表头的来源伪装成为NAT主机的`public IP`，并且将两个不同来源（`192.168.1.100`及`public IP`）的封包对应写入内存中，然后将此封包传送出去了；
 
此时 Internet 上面看到这个封包时，都只会知道这个封包来自那个`public IP`而不知道其实是来自内部。那么如果 Internet 回传封包呢？又会怎么作？

![snat 2](http://ytliu.info/images/2013-10-04-4.png "snat 2")

* 在 Internet 上面的主机接到这个封包时，会将回应资料传送给那个`public IP`的主机；
* 当NAT主机收到来自 Internet 的回应封包后，会分析该封包的序号，并比对刚刚写到内存中的资料，由于发现该封包为后端主机之前传送出去的，因此在`nat`的`PREROUTING`链中，会将目标IP修改成为后端主机，亦即`192.168.1.100`，然后发现目标已经不是本机（`public IP`），所以开始透过路由分析封包流向；
* 封包会传送到`192.168.1.2`这个内部介面，然后再传送到最终目标`192.168.1.100`机器上去。

 经过这个流程，你就可以发现，所有内网中的主机都可以透过这部NAT主机连线出去，而大家在 Internet 上面看到的都是同一个IP（就是NAT主机的`public IP`)， 

*DNAT*：修改封包表头的“目标”项。其主要用在内部主机想要架设可以让 Internet 访问的服务器。

![dnat](http://ytliu.info/images/2013-10-04-5.png "dnat")

如上图所示，假设我们的内部主机`192.168.1.210`启动了`www`服务，这个服务开启了`80端口`，那么 Internet 上面的主机（`61.xx.xx.xx`）要如何连接到我的内部服务器呢？首先，它必须要连接到我们NAT主机的`public IP`才行。  

* 外部主机想要连接到目的端的`www`服务，则必须要先连接到我们的NAT主机；
* NAT主机已经设定好`80端口`的封包对应的内部服务器的IP——`192.168.1.210`，所以当NAT主机接到这个封包后，会将目标IP由`public IP`改成`192.168.1.210`，且将该封包相关资料记录下来，等待内部服务器的回应；
* 上述的封包在经过路由后，来到private介面eth1处，然后透过内部的LAN传送到`192.168.1.210`主机；
* `192.186.1.210`主机会回应资料给`61.xx.xx.xx`，这个回应会被传送到NAT主机的`192.168.1.2`介面eth1上；
* 经过路由判断后，来到`nat`表的`POSTROUTING`链，然后通过刚刚第二步的记录，将来源IP由`192.168.1.210`改为`public IP`后，就可以传送出去了。

 其实整个步骤几乎就等于SNAT的反向传送。

#### iptables的nat用法
是这样设置的：
	
* 内部介面使用eth1，IP为`192.168.100.254`.

{% codeblock lang:sh %}
#!/bin/bash	
# 相关参数
  EXTIF="eth0"             # public IP介面
  INIF="eth1"              # private IP介面
  INNET="192.168.100.0/24"   # private IP地址
  export EXTIF INIF INNET

# 清除 NAT table 的规则
  iptables -F -t nat
  iptables -X -t nat
  iptables -Z -t nat
  iptables -t nat -P PREROUTING  ACCEPT
  iptables -t nat -P POSTROUTING ACCEPT
  iptables -t nat -P OUTPUT      ACCEPT

# 若有內部介面的存在（双网卡），开放成为路由器，且为IP分享器！
  if [ "$INIF" != "" ]; then
    iptables -A INPUT -i $INIF -j ACCEPT
    echo "1" > /proc/sys/net/ipv4/ip_forward
    if [ "$INNET" != "" ]; then
        for innet in $INNET
        do
            iptables -t nat -A POSTROUTING -s $innet -o $EXTIF -j MASQUERADE
        done
    fi
  fi
{% endcodeblock %}

其中

	echo "1" > /proc/sys/net/ipv4/ip_forward 

这一行是在让 Linux 主机具有router的能力。而

	iptables -t nat -A POSTROUTING -s $innet -o $EXTIF -j MASQUERADE 

这一行最关键！就是加入`nat`表的封包伪装，重点在那个“MASQUERADE”，这个设定值就是把IP伪装成为封包出去（-o）的那块介面上的IP。以上面的例子来说，就是 `$EXTIF`，也就是eth0。 所以封包来源只要来自`$innet`（也就是内网的其他主机），只要该封包是通过eth0传送出去，那就会自动修改表头IP的来源成为eth0的`public IP`。

这里还有一个问题，那就是对于上面所述的情况，内网中其他的主机应该要如何设定相关的网路参数？

答案其实很简单，只要将NAT主机作为内网主机的网关（GATEWAY）即可。只要记得底下的参数值：

	NETWORK 为 192.168.100.0
	NETMASK 为 255.255.255.0
	BROADCAST 为 192.168.100.255
	IP 可以设定 192.168.100.1 ~ 192.168.100.254 间，不可重复
	网关（Gateway）需要设定为 192.168.100.254 (NAT主机的private IP)

事实上，除了IP伪装（MASQUERADE）之外，我们还可以直接指定修改IP封包表头的来源IP。 举例来说，如下面这个例子：

	iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 192.168.1.100

即在不使用伪装的情况下，将对外IP固定为`192.168.1.100`。另外，还可以这样：

	iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 192.168.1.210-192.168.1.220

是在你的NAT主机有好几个公网IP的情况下，如果你想要轮流使用不同的IP，就可以这么使用。不过需要说明的是，除非你使用的是固定IP，且有多个IP可以对外连线，否则一般使用IP伪装即可，不需要使用到这个SNAT。另外，如果我们要用DNAT的话，就需要修改`nat`表的`PREROUTING`链，比如：

假设内网有服务器IP为`192.168.100.10`，该主机可对外提供`www`服务，假设`public IP`所在的介面为eth0，那么你就需要设置：

	iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.100.10:80 

那个`-j DNAT --to-destination IP[:port]`就是精髓，代表从eth0这个介面传入的，且想要使用`80端口`的服务时，将该封包重定向到`192.168.100.10:80`的IP及端口上面。其他还有一些较进阶的iptables使用方式，如下所示：

	-j REDIRECT --to-ports <port number> 

这个也挺常见的，主要就是进行本机上面端口的转换。不过，需要特别留意的是，这个操作仅能够在`nat`表的`PREROUTING`以及`OUTPUT`链上面实行。

比如，将要求与`80端口`连接的封包转递到`8080端口`：

	iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080 

------

另外真的觉得[鸟哥的Linux私房菜](http://linux.vbird.org/)很赞，做的很用心，而且还有一股台湾腔看了感觉很亲切:-) 强烈推荐！