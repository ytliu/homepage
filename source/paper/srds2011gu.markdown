---
layout: page
title: "Process Implanting: A New Active Introspection Framework for Virtualization"
date: 2013-10-11 21:20
comments: true
sharing: true
footer: true
---

4 security requirements: stealthiness, isolation, robustness and completeness.

Techniques used:

* Random selection of victim process
* Single virtual CPU for guest OS: implanted process只跑在一个vcpu上，但是这样有一个问题: 会被这个guest VM的其它vcpu上的process发现。所以需要disable SMP!
* Camouflage of implanted process: 重用原来victim process的所有数据结构。而且不允许fork（否则会多一个process，留下fingerprint）
* Self-contained executable of implanted process: 在编译implanted process executable的时候是静态地链接library的，为了防止恶意的库造成的影响
* Invisible memory space: 分配了三块内存: code, data和stack，这三块内存区域是在guest VM之外的内存区域
* Timing of implanting: 三个原则: 1.如果是syscall产生的context switch，在执行完implant process之后会重新执行一遍syscall；2. 如果是victim process把自己设成uninterruptible，则不能implant; 3. 如果是由kernel抢占造成的context switch，则不implant?
* Frequent scene restoration: 提供一种机制来恢复implant process运行过程中对系统状态造成的修改
* Checkpoint and restart: checkpoint机制使得implant process可以从一个checkpoint恢复继续执行
* Coordination between implanted process and hypervisor: 像strace这样的程序可能会attach到其它的进程，或者一些multi-thread的implant process在退出后会有一些线程还在运行，所以需要一个covert channel来设置一个control bit来告诉implant process什么时候可以exit；另外一种是要修改一些tool的源代码使得它们的打印是被send到hypervisor而不是直接打印在guest VM里面。
* Graceful exiting: 在正常的exit中会free一些内存，由于implant process的内存不属于guest VM，所以会有问题。所以这里通过设置debug register来intercept exit的行为。
* Protection from the hypervisor: root + unkillable flag
* Multi-threaded program implanting: 这个有点复杂，主要是要在implant process和normal process的线程间切换是管理context信息，但是其实是没有简单的办法分辨哪些是implant process的线程。这里的假设是每次选的都是主线程，而基本上都是主线程会产生新线程，所以它根据新线程的pid的大小来判断（pid大于之前maxpid的都是implant process产生的线程）

------
