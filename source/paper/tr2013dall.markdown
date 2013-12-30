---
layout: page
title: "KVM/ARM: Experiences Building the Linux ARM Hypervisor"
date: 2013-10-29 19:50
comments: true
sharing: true
footer: true
---

利用ARM提供的硬件虚拟化支持实现ARM体系架构上的virtualization。

ARM提供的虚拟化支持：

#### CPU：

Non-secure state: 除了User和Kernel mode之外，多了一个Hyp mode, Hyp mode支持trap，可以配置一些敏感指令trap到Hyp mode里面。另外ARM允许配置trap,是直接trap到Kernel mode（比如syscall）还是Hyp mode。

#### Memory：

和EPT一样，多了一个stage-2 page table (GPA->HPA)。可以通过Hyp Configuration Register（HCR）中的一个bit来开启或禁止stage-2 translation，stage-2 page table的CR3是存在VirtualizationTranslation Table Base  Register（VTTBR）上的。这两个寄存器都只能在Hyp mode下配置

#### Interrupt：

有一个Generic Interrupt Controller（GIC）架构。GIC有一个distributor，一个interrupt来了之后，由distributor负责把它发到相应的CPU interface上，每个CPU都有一个CPU interface，它会raise interrupt signal到CPU上，CPU通过访问MMIO来发送ACK和EOI。

由CPU interface发起的interrupt可以被配置成trap到kernel mode还是Hyp mode（只能二选一，all or nothing），如果全部trap到kernel，则hypervisor就不能管理虚拟机了，如果全部trap到Hyp，则性能太差。

为了支持虚拟化，ARM引入了一个virtual interrupt的支持来减小trap到Hyp mode的interrupt数量。这样，由硬件产生的中断都被trap到Hyp mode，而virtual interrupt则trap到kernel mode，这样guest OS就可以ACK，mask和EOI？每个CPU都有一个virtual CPU interface，guest OS可以通过MMIO配置，而virtual CPU control interface（list registers）只能在Hyp mode下配置。一般来说，模拟出来的设备发出的interrupt都是virtual interrupt，VM直接在kernel mode就可以处理和写回ACK/EOI。另外直接分配给VM的设备发出的interrupt也可以被当做virtual interrupt

另外无论何时写distributor都会trap到Hyp mode，比如IPI会写distributor的SGI register，就需要hypervisor为每个VM模拟一个virtual distributor。
Timer：