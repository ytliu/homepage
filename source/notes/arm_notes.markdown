---
layout: page
title: "ARM Notes"
date: 2014-1-19 19:50
comments: true
sharing: true
footer: true
---

Notes about ARM architecture study:

------

#### CPU模式

CPU ARM架构指定了以下的CPU模式。在任何时刻，CPU可以在只有一种模式，但由于外部事件（中断）或编程方式可以能够切换模式。

mode 		| Description
:---------- | :-----------
用户模式(User)   | The only non-privileged mode.
系统模式(System) 	| The only privileged mode that is not entered by an exception. It can only be entered by executing an instruction that explicitly writes to the mode bits of the CPSR.
Supervisor模式(SVC)	| A privileged mode entered whenever the CPU is reset or when a SWI instruction is executed.
Abort模式(Abort) 	| A privileged mode that is entered whenever a prefetch abort or data abort exception occurs.
未定义模式(Undef) 	| A privileged mode that is entered whenever an undefined instruction exception occurs.
干预模式(IRQ) 		| A privileged mode that is entered whenever the processor accepts an IRQ interrupt.
快速干预模式(FIQ)		| A privileged mode that is entered whenever the processor accepts an FIQ interrupt.
Hyp模式(Hyp) 	    | A hypervisor mode introduced in armv-7a for cortex-A15 processor for providing hardware virtualization support.

#### 寄存器

![register](http://ytliu.info/images/arm-note-1.png "register")

寄存器 R0-R7 对于所有CPU模式都是相同的; 它们不会被分块。
对于所有的特权CPU模式，除了系统CPU模式之外，R13和R14都是分块的。也就是说，每个因为一个异常(exception)而可以进入模式，有其自己的R13和R14。这些寄存器通常分别包含堆栈指针和函数调用的返回地址。

![register2](http://ytliu.info/images/arm-note-2.png "register2")

note:

* R13 也被指为 SP, the Stack Pointer.
* R14 也被指为 LR, the Link Register.
* R15 也被指为 PC, the Program Counter.

#### Thumb指令集（16bit）

Thumb指令集可以看作是ARM指令压缩形式的子集，它是为减小代码量而提出，具有16bit的代码密度。Thumb指令体系并不完整，只支持通用功能，必要时仍需要使用ARM指令，如进入异常时。

在一般的情况下，Thumb指令与ARM指令的时间效率和空间效率关系为：

* Thumb代码所需的存储空间约为ARM代码的60％～70％
* Thumb代码使用的指令数比ARM代码多约30％～40％
* 若使用32位的存储器，ARM代码比Thumb代码快约40％
* 若使用16位的存储器，Thumb代码比ARM代码快约40％～50％
* 与ARM代码相比较，使用Thumb代码，存储器的功耗会降低约30％
