---
layout: page
title: "Countering Persistent Kernel Rootkits Through Systematic Hook Discovery"
date: 2013-09-26 19:10
comments: true
sharing: true
footer: true
---

identify kernel data hook

HAP: hook attach points

context-aware execution monitor:  

* monitor internal process events (syscall…)
* intercept "iret" to handle interrupt (shadow interrupt stack for nested interrupt)
* 在monitor里面跑tool，然后有monitor记录system call和syscall path里面的指令
* kenel hook identifier(KHI): 当收集了这些指令，KHI会根据call或jmp指令识别出HAP

implementation:

* logging: in lifetime (int 80 & sysenter) of syscall event, 记录，而且要考虑interrupt的影响（avoid noises）
* 最后得到syscall和其对应的kernel instructions
* hook identify: 当遇到比如“call \*%edx”，需要一种方法trace到这个地址到底是多少？

    dynamic program slicing：给定一段执行流和一个变量，找到所有访问这个变量的指令。

总结：

感觉方法都很简单，就一个dynamic program slicing有点技术含量，但是也是直接用以前的办法。感觉和他提到的HookFinder应该很像，只不过是找一些security tool的HAP。不过应该还是挺实用的。
里面提到一个related work说96%的rootkit都是hook-based，有点夸张啊！

------
