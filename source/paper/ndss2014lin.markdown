---
layout: page
title: "HYBRID-BRIDGE: Efficiently Bridging the Semantic Gap in Virtual Machine Introspection via Decoupled Execution and Training Memoization"
date: 2014-03-08 15:50
comments: true
sharing: true
footer: true
---

#### Motivation:

在VMI中，为了narrow semantic gap，有两种方法可以利用本地的程序来做out-of-vm的VMI，但是VMST（参见[这里]()）和Virtuoso都太慢了。

#### 方法：

于是，作者就提出了一个Hybrid的方法，见下图：

![hybrid bridge](http://ytliu.info/images/ndss2014lin.png "hybrid bridge")

具体步骤是这样的：

现在对于一个VMI的应用，先在slow bridge那里用VMST的方法跑一遍，生成一些Metadata，存到fallback那里，之后这个应用就能在fast bridge那里跑了。

其中，slow bridge那里用的是Qemu虚拟机，而fast bridge那里用的是KVM虚拟机！

在fast bridge模式下，所有指令中和redirectable data相关的指令都被dynamic patching的方法改成了int 3指令，会被下陷到VMM里面，而EPT中需要被重定向的地址映射也被映射到了被监控的虚拟机的地址。

#### Result：

效果是这样的：

![hybrid bridge result](http://ytliu.info/images/ndss2014lin-1.png "hybrid bridge result")

它跑了每个VMI程序5遍，其中大部分程序在第二遍就基本上能全部在KVM的虚拟机里面跑了！

#### Limitation：

这里有好多个limitation，我觉得最让我不能接受的是下面这一条：

* 一个虚拟机需要有两个和它相同操作系统的虚拟机（一个Qemu，一个KVM）来监控！


------
