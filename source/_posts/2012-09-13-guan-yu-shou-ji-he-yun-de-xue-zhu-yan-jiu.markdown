---
layout: post
title: "关于手机和云的学术研究"
date: 2012-09-13 14:00
comments: true
categories: Android System
---
这两天看了两篇关于手机和云结合的文章，两篇的motivation不同，所以所用的方法也就自然不一样了。下面对这两篇进行分别的介绍：

第一篇叫《*CloneCloud: Elastic Execution between Mobile Device and Cloud*》

分开来说就是用了三种技术，application partitioning, thread migration, remote execution，主要是前面两个：

**application partitioning**: 主要用了static和dynamic相结合的方式，先用static找出合法的可分割点，再用dynamic计算出cost model，然后得出最合适的分割方案，生成partition configuration，主要是要注意两点：

1.这些都是在offline做的，在程序实际运行的时候直接获得configuration，然后在migrate point的时候进行migrate。

2.按照作者的意思是不需要source code的，是通过分析二进制文件来得到合法的分割点，但需要重写二进制文件，加入一些annotation比如migrate point之类的。另外，合法的分割点作者提出了几个限制：

    a.首先，它必须是method entry and exit point，不能是core-system library，当然在method里面调用的core 
    library是允许的;
    b.Methods that access specific features of a machine motivationust be pinned to the machine, 比如说
    location-based的调用或者camera这些没有被virtualized的服务都不能被分到远端，这也是我后面会详细说的；
    c.Methods that share native state must be collocated andt the same machine.也就是说那些调用native 
    code同时share了native state的方法也不能被migrate
    d.Prevent nested migration.

<!-- more -->

**thread migrate**: 在Dalvik VM执行到migrate point的时候，会有一个migrator thread将thread suspend住，并将当前的thread state打包，把它所需要的信息通过网络发给cloud端的clone。这里的state主要包括execution stack frames and relevant data objects in the process heap, and register contents at the migration point. Virtualized stack frames…在cloud端的clone接收到这些信息后会有一个migrator thread来将这些信息装入VM的内存中，并继续执行，当遇到reintegration的时候用同样的方式将该thread的state打包发回去，mobile端的thread继续执行~这些看起来很容易，但是有一个问题，因为每一个object都是通过内存地址进行reference的，在不同的platform中这些reference可能会不一样，这里作者用了一种叫做object mapping table的方式，用一张图（Figure 7）就可以很方便的说明：

![object mapping table example](http://ytliu.info/images/2012-09-13.png "object mapping table example")

另外，clonecloud系统是实现在AOS和Android x86 virtual machine上的，修改了大约8000行的Dalvik代码，用JChord工具进行static analysis，用Monsoon power monitor [Mon]测量了energy consumption来计算cost model，通过hprof(HPR)进行thread state的capture。最后evaluation就不细说了，据说提速20X，降低了20X的能耗，当然这个是怎么测的我也没细看。

最后来说说clonecloud的限制：

首先是之前提到的platform-pinned的调用不能migrate到远程，其实从合法partition的限制来看就可以发现要partition不是那么容易的，其实作者也提出：We consider this alternative a complementary point in the design space, and plan to pursue it in conjunction with thread granularity migration in the future.从这个角度来看和我们的remote-binder的motivation还是很像的。除了这一点，两地之间不能内存共享也是一个限制partition很重要的方面。另外，作者提到的perfunctory concurrency其实应该不是一个太大的问题，本来这段code就是顺序执行的。

其实关于application的partition问题并没有结束，作者的method力度的partition也没有证明就是最好的，这一块应该还有研究的余地

----------------

另外一篇叫《*Paranoid Android: Versatile Protection for Smartphones*》

这又是一篇mobile和cloud结合的paper，和clonecloud不同的是它强调的是security，所以它使用的方法也就不是thread migration，而是replica。按照作者的说法，cloud端的emulator中有一个mobile端系统的replica，而它的做法这是控制那些所谓的non-deterministic的因素，按照作者的说法，只有某些system call和来自OS的signal。为了达到mobile和cloud端的synchronization，主要分为两部分，mobile端的record和cloud端的replay。

**Record**：主要是在mobile端，为了减小trace数据的传输，这里有几个原则和方法来缩减trace的大小：

* 1.Record only system calls that introduce non-layoutdeterminism. 对于比如创建socket，读文件，甚至是IPC的syscall都不需要记录，因为这些都是replica那里也可以直接做的；
* 2.Use a network proxy so that inbound data are not logged in the trace. 这样可以避免网络数据传到mobile再从mobile传到replica，减小mobile端的负担和trace的大小；
* 3.Compress data using three andlgorithms. 这个就没什么好说的，就是把trace进行加密，减小其大小。

在mobile端，整个系统的启动过程是这样的：

* 1.Android的init进程先启动一个tracer进程，打开一个FIFO，等待其它进程的连接;
* 2.之后在init启动其它进程的时候，先启动一个exec stub，它会把该进程的pid写入tracer的FIFO中，表示需要被trace，然后pause；
* 3.tracer对这个进程进行attach，然后恢复exec stub的运行，stub运行exec命令执行进程的binary。

其实总的来说record还是一个比较好理解的过程，但是对于replay，我觉得这是这篇文章中完全忽略的一个部分，它只是说有很多之前的工作对record和replay进行了研究，但是竟然一点都没有说明在replica端是如何进行replay的，这也就使得我对它所说的synchronization的效果产生了极大的怀疑。

另外，文中所说的record和replay只是对system call进行了记录和重放，但是对于一个这么复杂的系统，当当记录这些真的够吗？特别是对于手机这种特殊的设备来说，用户输入大部分是触控事件，而这些在文中（不管是paper还是technical report）都没有详细提到。而关于进程间的IPC通讯，作者只是在谈到ioctl的时候简单的提到了一下binder机制，并且在最后说这些进程间的通讯都是不需要被记录的，再加上在最后的evaluation部分，作者只是对trace的大小，已经performance的overhead进行了简单的测试，并没有对它所实现的原型的可行性的分析，这就让我对它们所说的mobile的replica产生了极大的怀疑。另外还有最后一个问题就是调用mobile特定的服务，比如location, camera的服务等，这些文中都没有合理的解释。仅仅是一个system call的record和replay移植到手机上其实并不是一个很有说服力的研究。

不过在这篇paper中我学到一点，就是paranoid对trace的保护，这也是我一开始就有的一个疑问。既然attacker可以攻击你的手机，自然就可以伪造trace，而且它又可以知道存在你手机中的key，那么该如何对这个关键的trace进行保护呢？虽然我不知道tamper-evidency这个方法是否作者自己提出来的，但是确实很有意思：它用了一个secure storage，将trace和一个HMAC code存在这个storage中，而存的方法是：
    
    STORE(message + HMAC(key,message)) 
    key′ = HASH(key) 
    key = key′

也就是说每一次的key都是上一次的key做hash得来的，由于attacker只能知道当前的key是什么，而无法知道之前的key，所以它就无法伪造entry，只能对其进行删除操作，那么在cloud端的replica就能通过计算发现trace的历史记录的修改。而做到这一点唯一需要保护的就是初始化的时候key在mobile和replica中的安全传递，这可以在整个系统启动的时候就做好了。
    
除此之外我也想过用append-only memory的方法，这样也能防止attacker对trace的修改，不过由于自己没有实际做个这个方面的研究，也不清楚它在手机中是否有实现。
    
总的来说，这是一篇提出概念的文章，通过cloud来对mobile进行安全检测，作者说实现的prototype可能真的是一个prototype，我都不确定是否真的实现了replica，是否真的运行起来了。加上作者对replica端基本没有做出什么介绍，所以感觉还是比较不靠谱的。

----------------

现在cloud和mobile相结合的概念提出的越来越多了，但真正运用到世纪生活中的还很少很少，一方面是很多东西还停留在research阶段，没有比较稳定的工程实现，另一方面就是关于流量的考虑，mobile和cloud的结合离不开稳定的网络给予保障，而这些在现在的实际生活中还不能很完美的实现。不过我相信在不久之后，这两个当前及其流行的设备一定能非常好的结合在一起，方便大家的生活，并为mobile提供非常强有力的能源，性能和安全保障的。
