---
layout: page
title: "Eliminating the Hypervisor Attack Surface for a More Secure Cloud"
date: 2013-10-26 19:50
comments: true
sharing: true
footer: true
---

### 问题：

在ISCA2010提出的NoHype架构，是一个构想出来的体系架构，希望在没有hypervisor的情况下，完全通过硬件的特性实现一个有虚拟化功能的系统。而这一篇这是在软件上（Xen 4.0）根据isca'10的思路实现出来的一个NoHype原型。它还有一个很小的temporary hypervisor在创建和启动虚拟机的时候要被用到。

其实做法和在ISCA'10中提出来的基本上是一样的，通过：

* pre-allocating memory and cores
* using virtualized I/O device

来做。

另外，它还需要cache一些硬件的信息，在之后虚拟机运行调用比如CPUID指令的时候直接返回这些信息。

为了避免indirection（比如中断处理，需要virtual中断到实际硬件中断的mappign，再比如IPI，正常情况下，每个VM看到的processor ID都是从“0”开始的，所以需要hypervisor做一个mapping进行转换），对于IPI，在NoHype上由于core是VM独享的，每个VM都可以看到processor真实的ID。处理中断也一样。

其它就是VM的lifecycle了，创建，启动（discover device， discover processor capabilities）都是需要temporary hypervisor帮忙的，然后在VM真正开始运行之前，temporary hypervisor就要退出了（用过VM initrd里面的一个hypercall来控制），虚拟机完全独立运行（绑定了core，EPT和device（这里只用网卡）），虚拟机关机（VM自己发起的关机会产生VMExit，然后被kill，Dom0也可以发一个NMI，这个的实现方式是配置VM的VMCS，让它在收到NMI的时候发生VMExit）
