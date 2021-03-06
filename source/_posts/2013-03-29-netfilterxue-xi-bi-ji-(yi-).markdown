---
layout: post
title: "Netfilter学习笔记（一）"
date: 2013-03-29 18:39
comments: true
categories: Network
---

这两天在学习`iptables`，感觉这个东西实在是太牛逼！想用两篇博文来介绍一番。

第一篇会介绍下netfilter和iptables的关系，以及iptables的原理；第二篇希望通过`libipq`和`nfqueue-bindings-python`来介绍下user态如何利用C和Python来调用iptables的接口获得封包的信息。

### netfilter

简单地说，netfilter是一套在计算机网络栈中过滤和修改封包的框架，它的做法是在Linux Kernel中插入了一系列的hook，并允许kernel在不同层的网络栈中注册回调函数，这些回调函数会在封包进入相应的hook的时候被调用到。

<!-- more -->

网络栈中对netfilter的支持如图所示：

![netfilter](http://ytliu.info/images/2013-03-29-1.png "package flow in netfiter and general networking")

可以看到在链路层和网络层中按照封包流的路径有五种类型的hooks: prerouting, input, forward, output, postrouting。这五种hooks会在封包到达之时按照封包流的顺序调用相应的回调函数，对四种类型的表中的chain（会在iptables中描述）进行过滤和修改：filter, nat, mangle, raw。而定义这些过滤的规则则是由一个用户态的命令`iptables`进行配置，也就是我接下来要详细描述的命令。
 
### iptables

[这里](http://itzone.hk/article/index.php?tid=14)有6篇系列的文章介绍iptables的，讲的挺清楚，蛮适合入门学习的。

前面说过iptables有四种类型的表：filter，nat，mangle，raw，这里只是对filter表进行介绍：

filter表主要用于对封包进行过滤，在该表中有三条默认的chain：INPUT，FORWARD，OUTPUT。

chain是做什么的呢？(转自[这里](http://itzone.hk/article/article.php?aid=200502091507054036))

>所謂chain就是一組封包過濾規則，您可以在INPUT chain中加入一條防止所有外界封包進入的規則；您可以在OUTPUT chain中加入一條防止用戶連接某網頁伺服器。準備進入網絡的封包，會順著chain內的過濾規則被稽核，若果該封包並不符合任何規則，則會直接進入網 內，因為INPUT和OUTPUT在Linux kernel裡被預設為ACCEPT，而FORWARD則被預設為DROP。

比如说，如果要把发到某个地址（如192.168.1.2）的包丢弃，可以这样做：

	$ sudo iptables -A OUTPUT -d 192.168.1.2 -j DROP

这里`-A`指对OUTPUT这条链的规则进行修改，`-d`表示这个封包的destination，`-j`后加的是target，有五种选择：ACCEPT（不作任何操作，让封包流过），DROP（将封包丢弃），QUEUE（将封包插入队列，传递到用户态处理，这个会在第二篇中详细描述），RETURN（直接从该chain中返回，到前一个chain的下一条规则继续执行），以及自己定义的CHAIN，如下：

	$ sudo iptables -N SELF_CHAIN

也就是说我们可以按照不同的源地址、目标地址、端口、协议等规则分类，由不同的chain进行处理，可以更合理地对过滤条件进行管理。

另外，有几个常用的选项这里提一下：

	-A chain 				# modify a chain
	-D chain rulenum 		# delete a specific rule of chain
	-I chain rulenum rule 	# insert rule in rulenum of chain
	-R chain rulenum rule 	# replace rule with rulenum of chain
	-L [chain]				# list rules of chain
	-F [chain] 				# flush the rules of chain
	-N chain 				# new a chain
	-X [chain]				# delete chain

	-p protocol 			# e.g., tcp, udp, icmp...
	-s source address 		# e.g., 192.168.1.22
	-d destination address 	# e.g., 192.168.1.13
	-j target 				# e.g., ACCEPT, DROP, RETURN, QUEUE, other-chain
	-i in-interface 		# e.g., eth0
	-o out-interface		# e.g., eth0


------

下一篇主要介绍当`iptables`的target是QUEUE或者NFQUEUE时，用户态要如何调用相关接口。