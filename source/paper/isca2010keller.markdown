---
layout: page
title: "NoHype: Virtualized Cloud Infrastructure without the Virtualization"
date: 2013-10-20 19:50
comments: true
sharing: true
footer: true
---

### 问题：

现在虚拟化环境中hypervisor太大，不安全，需要把hyperviosr去掉，但是同样拥有和虚拟化一样的功能和隔离性

### 方法：

利用现有硬件的特性和设计新的硬件特性，通过完全硬件的方法来支持虚拟化，虚拟机完全通过硬件进行隔离，而不需要中间加一层软件层（hypervisor）。

### 技术：

* CPU：每个虚拟机运行在单独的core上，通过mask out每个core的local APIC来禁止core与core之间的IPI
* Memory：在启动虚拟机之前为虚拟机分配好物理内存（比如2G），然后在每个绑定的core对应的EPT中写死GPA到HPA的映射，这样也就不会产生EPT Violation了（如果发生了，直接把虚拟机杀掉），对于memory bus的fairness问题，则需要修改memory controller来实现
* I/O：由于有了内存隔离，所以虚拟机DMA到设备和设备DMA到虚拟机都可以通过写这个隔离的内存来实现，而设备的中断可以通过MSI写隔离的内存实现。而对于设备本身的虚拟化，则需要通过SRIOV的设备才可能实现了。对于I/O带宽的fairness，可以通过PCIe的credit-based flow control来实现。
