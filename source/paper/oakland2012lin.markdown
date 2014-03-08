oak---
layout: page
title: Space Traveling across VM: Automatically Bridging the Semantic Gap in Virtual Machine Introspection via Online Kernel Data Redirection
date: 2014-03-08 14:00
comments: true
sharing: true
footer: true
---

#### problem: 

It is hard to write VMI code to solve the semantic gap

####Main idea: 

use Secure-VM code P to introspect in-guest VM memory data x

#### challenge: 

we need to identify the system call context we need to redirect the data and the redirectable data (some data are not redirectable like data in the stack that is used for control flow)

#### Technique: 

* syscall execution context identification
通过拦截`int 80`和`sysenter`两条指令，以及`sysexit`和`iret`两条指令，可以识别出何时进入和退出了syscall execution context。但是主要的挑战在于在这个中间会发生interrupt和exception，以及context switch。

具体方法是这样的：1. 识别每个syscall的开始，2. 在每个（除了这个syscall）的interrupt/exception handler执行之前对其进行拦截，用一个global counter来标记进入了其他的handler，退出这个syscall的context，然后在handler退出的时候counter减一，直到counter变成0则重新开始这个syscall的context。3. 直到识别出这个syscall的退出指令。另外，还需要禁止context switch（比如可以把这个进程的priority设到最高

* redirectable data identification
找到需要被重定向的数据。

方法是这样的：先找到 kernel global variables，然后采用taint技术跟踪它。

* kernel data redirection

对于哪些syscall要被redirect，作者提供了以下这张表：

![syscall table](http://ytliu.info/images/oakland2012lin.png "redirect syscall table")

对于如何redirect data，作者采用的是shadow TLB和shadow CR3的技术，即STLB和SCR3都是维护的是guest vm的kernel的值。另外对于写来说，是COW的形式。

#### Implementation:

use Qemu, dynamic binary intrumentation

#### Result:
9.3X overhead in average
for ps, only 0.06s
