---
layout: page
title: "Towards a VMM-based Usage Control Framework for OS Kernel Integrity Protection"
date: 2013-09-17 19:50
comments: true
sharing: true
footer: true
---

一种VMM-based的access control model，用于protect kernel integrity。主要考虑了两个问题：

continuous policy enforcement: 一旦一个authorization decision被决定了，这个process接下来的访问将不会被检查。然而在一些系统中，当某些条件改变之后权限是应该要被回收的。这主要是因为接下来的一个问题：proce和object的属性是会被改变的。

mutable process and object attrubutes: 进程和object的属性是会被改变的，除了系统管理的原因之外，更重要的是一些side-effect

举个例子：对于一个系统调用表来说，普通的的用户进程是没有权限修改的，但是一些合法的特权进程是可以的，比如AV进程。对于传统的DAC来说，一个rootkit也可以修改，而对于MAC（SELinux）来说，任何进程都不能修改。？？？（这个例子看不懂。。。）

另一个例子比较好理解：本来一个被认证过的进程在访问自己的sensitive文件，如果在访问过程中它的完整性被改掉了，就应该要回收这个进程对该文件的访问权限，但是现在的ACmodel都没办法做到这种continuous security check.

提出一种框架Usage CONtrol model (UCON)，event-basedUCON-KI:

Event (access request, or a subject/object attribute change) -> if predicate true -> action execute (allowing/denying an access re- quest, revoking an ongoing access, and updating subject and/or ob- ject attribute values)

Case study: Figure 4

Implementatino: using Bochs

Rootkit (18):
        
    Syscall table: adore, knark, phide, rial, rkit, synapsys, override, maxty, kbdv3, all-root, phalanx, mood-nt
    kernel text: suckit, suckit2priv, enyelkm, phantasmagoria, superkit
    virtual file system: adore-ng

总结：
        
* 感觉在eval的时候没有感觉有关于那两个特点的东西啊？就是禁止了对syscall table，kernel text等这些不能修改的东西的修改罢了？

------
