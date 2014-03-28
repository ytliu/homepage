---
layout: page
title: "Dune: Safe User-level Access to Privileged CPU Features"
date: 2014-03-08 14:50
comments: true
sharing: true
footer: true
---

#### Motivation:

provides applications with direct but safe access to hardware features such as ring protection, page tables, and tagged TLBs, while preserving the exist- ing OS interfaces for processes.

#### 一句话概括

我觉得Dune就是一个向上层提供process接口（而非VM接口）的KVM。

#### Dune的架构

![dune arch](http://ytliu.info/images/osdi2012belay.png "dune architecture")

其实从图上来看，如果了解KVM的话，就赶紧特别简单，其它都和KVM一样，只不过现在non-root模式下运行的不是一个虚拟机，而是dune process。然后：

* 对于system call，需要把它们都改成hypercall，然后通过vmexit下陷到kernel
* 对于memory management, dune process维护了VA -> PA的映射，而EPT维护了PA -> HA的映射，但是这里有一个问题，就是希望EPT和kernel page table能合并到一起。但是因为它们的格式不同，所以似乎作者也没有解决这个问题。
* 对于一些硬件的访问，比如一些寄存器。这个可以通过设置VMCS来控制。

#### 和VMM的对比

作者提出了四点：
 
* 在VMM里面用到的hypercall在这里被当做system call来使用了。
* VMM里面需要模拟很多设备来支持guest OS，而在Dune里，这些设备都可以通过OS来支持。
* VMM里面的context switch (VMExit)需要保存很多数据，而Dune里面由于只是一个类似于syscall的context switch，所以需要保存和恢复的东西很少。
* 在VMM里面，不同VM的内存地址映射都是isolate的，而在Dune里面只不过是process的内存地址空间的隔离，还是有很多share的特性保留着的。

#### Implementation:

在KVM上进行修改。

#### Result:

做了三个应用进行演示：Sandbox，Wedge和garbage collection


------
