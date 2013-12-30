---
layout: page
title: "EagleEye: Towards Mandatory Security Monitoring in Virtualized Datacenter Environment"
date: 2013-10-19 19:40
comments: true
sharing: true
footer: true
---

do both memory and disk introspection

用到的几个技术：

* Stealthy hook：用CPUIDoverwrite想要下陷的指令，在VMM里面再还原被overwrite的指令。另外，用一个shadow copy来解决integrity check的问题（这个有个疑问，因为它把相应的EPT设置成not read，那么在执行的时候难道不需要读权限吗？）
* deferred memory introspection：通过把相对应的PTE设成not write，可以使得原本会产生page fault的内存延迟被检查
* replication of high-level representation: 相比于每次直接introspect memory和disk，EagleEye采用了一种不一样的方法（有点像状态机），现在最初维护一个一致的状态Ri，然后截获所有syscall，直接把syscall apply到Ri，得到新的状态Rj（它必须得知道这些syscall对状态产生了哪些影响）
* In-VM Idle loop可选，用于优化。由于当执行流进入VMM后，guest的VCPU应该要block，但是单纯地block一个VCPU会造成一些问题，如果block所有的VCPU会造成比较大的overhead，所以改成将陷入VMM的thread设成idle loop

实现:

* 采用了libvmi+xen
* 主要考虑的是disk introspection，并结合write buffer和memory map view来实现：
* 维护一个per-VM table，记录（path, fd, memory-mapped view, write buffer）；
* 当secure monitor访问一个file的时候，先默认用disk introspection得到content，同时会把write buffer里面的内容pull出来，有的时候还要考虑memory-mapped view。

问题：

* 什么时候做的stealthy hook（好像没讲？）
* 如何结合disk block和write buffer（很含糊）
* consistency的问题描述的不是很清楚
* 对于non-standard的syscall不能解决
* memory和CPU的introspection讲的很简单？
* 里面提到的一个"the use of virtualization to separate instruction execution and data access contexts on memory pages"，挺有趣的

------
