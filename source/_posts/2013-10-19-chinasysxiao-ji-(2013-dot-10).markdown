---
layout: post
title: "ChinaSys小记（2013.10）"
date: 2013-10-18 10:32
comments: true
categories: System Conference
---

这两天去苏州参加第五届[ChinaSys](http://www.chinasys.org/)，这应该是我第三次参加ChinaSys；这一届由中科大主办，依然主要是MSRA赞助。粗略算了下大约有六七十个人参加，学校主要还是那几所，企业来了五家（MSRA，Intel, Baidu, AMD, NVIDIA）。一天半的时间有将近6个session，2个keynote，22个report，时间安排的非常紧凑（当然每一届应该都差不多啦）。这两天听下来，总的感觉就是做大数据、分布式的比较多，而和我自己现在研究领域（虚拟化/安全）相关的很少。

另外这次还碰到了前两年到清华一起写本子的两个同学，和他们实验室几个人聊的比较多。不管怎么说，听听报告，互相交流学习，了解下别人都在做些哪方面的研究，对于自己的视野和今后的学习还是很有帮助的。

由于这次和自己相关的topic很少，就简单记下自己听到并听懂，而且比较感兴趣的内容，整理下自己的笔记：

<!-- more -->

------

#### 2013.10.17 周四 平江府

第一个keynote是来自NVIDIA的赖俊杰讲的关于CUDA目前的情况和未来的发展方向，演讲者太高端，一上来就英文开讲，加上自己本来对这方面实在欠缺，只能默默地打酱油了。。。

第一个session是关于EFFICIENT SYSTEM/PLATFORM，两个来自MSRA，两个来自北大：

* `xCloudified` by [郭振宇](http://research.microsoft.com/en-us/people/zhenyug/) from MSRA：和上一届的ChinaSys一样，问题依然来自他们开发过程中发现的一些现象：在一个系统中，对于service, library, tool这三个"Mr. Competence"，因为是不同人开发的，如果他们走的方向不同，就可能会各自影响其他方面（fragmented effort may harm each other）；而他们的目标就是开发一个系统，能够保证：`efforts do not harm each other`，具体如何做的这里就不剧透了，毕竟还是人家正在研究的东西。其实我觉得这东西做出来主要是要能协调一个大公司各个部门的人的分工与合作，使得整个公司就算是不同的部门，研究都是朝同一个方向前进的。
* `Minerva` by 王敏捷 from MSRA： 关于deep learning，似乎是微软最近蛮重视的一个项目，和敏捷讨论，他说微软有好多部门都在做这个方向，比如语音识别（这个好像做的很好），他的presentation很不错，讲的很清晰，不过自己实在对这方面不是很懂，就不在这里献丑了。
* 北大的[肖臻](http://zhenxiao.com/)教授关于map-reduce上的一些研究，一个大家都知道的常识是：”map-reduce的完成时间取决于最后一个完成的任务“，于是他们考虑了很多方面（比如input, phase time...），来找到比较慢的那个machine（straggler machine)，又考虑了很多方面，来把这些比较慢的机器上跑的task放在另外机器上跑（backing up on alternative machine），再做了很多实验，来验证他们系统的有效性，基本就是这样，我的感觉评测还是很solid的。
* 还是来自北大的杨智做的是图计算的研究，我听下来感觉他们的问题是说现在图计算中计算节点和数据节点是放在一起，每次都要load一个很大的图，造成性能瓶颈？（其实自己也不是很理解），于是他们就想了个法子为jobs共享一些静态的graph。。。好吧，就这么多，其它我也lost掉了，实在惭愧！

之后一个session是关于memory management的，现在还有印象的有两个工作：

* 清华的郭维超（就是那个一起写本子的兄弟:-)）的一个关于手机应用程序启动时间的研究；主要问题是由于内存的限制，手机应用程序如果没有在内存中的话启动时间会很长（好像平均要个几十秒吧？），然后他们主要希望通过对Dalvik的修改来管理cache机制，因为他们也是还在实践的项目，这里就不讨论太多细节了。其实我个人觉得这是一个蛮有意义的事情，做出来也可以直接在android上使用，而且这个问题也不是一个新问题，已经有一些相应的但是还不完善的方法，希望他们能早点搞一个比较完美的解决方案出来。
* 计算所的蒋德钧的一个蛮有意思的工作（还有一张poster贴出来），关于有效利用hybrid memory，比如如何同时使用DRAM和NVM（Non-Volatile Memory，比如SSD，PCM...），他们之前开发了一个HMSim（Hybrid Memory Simulator），然后他们想开发一个对下面hybrid memory感知的OS —— HMOS，比如现在他们已经开发了一套叫EPFI（Energy-aware Page Frame Initialization）的系统，可以利用一些energy的knowledge来合理调度对下面存储单元的读写，从而达到节约能源的效果。当然他们也在探究除了energy之外的其它参数。

#### 2013.10.18 周五 平江府

第二天能说的内容更少了，就写在一起吧：

* keynote是来自Intel的刘新铭关于IoC（Internet of Things，物联网）的介绍，这东西最近很火，前几年关于RFID技术的发展，加上其它方式，IoC已经慢慢走进并且开始影响人们的生活了。Intel也开发了一个叫Quark的小型芯片用于IoT，产品叫X1000，是一个ultra low power，single core, single thread的芯片，为了可靠性它还包括了ECC的功能，据说马上就要投入市场了。当然IoC和IT中其它领域一样，都面临一些比如power，performance，security的问题。
* 来自北大的孙广宇介绍了一个他们关于SSD的研究，他们发现LSM（Log-structured merge-tree）based KV store有一个特点就是它能够消除随机写，而SSD的随机读性能非常高，但是随机写就差很多，于是他们希望把这两点结合起来。但是单纯的结合不管从LSM还是从SSD的角度都会遇到一些问题，于是他们就着眼于这些问题，并且基于百度开发的open channel SSD，把 LSM-tree KV整合进去了。我突然发现这个思路蛮有趣的：找到两个有相反特性的东西，然后结合起来！（naive结合肯定有问题，就可以解决其中遇到的challenge）。
* 来自百度的欧阳剑则主要介绍了百度这几年做的一些工作（cloud storage, mobile search, app distribution, mobile video）以及他们在SSD中的研究——SDF（Software-Defined Flash)，主要是针对SSD的一些限制（比如low capacity utilisation, low bandwidth utilisation, unsustained performance），扩展了其硬件架构，把硬件channel接口暴露给软件，让程序员能对其做更合适的控制。

其实还有一些应该蛮有趣的工作，比如Intel做的XenGT，似乎是在Xen上做了GPU的硬件虚拟化支持？不过由于自己实在太累，非常不争气地睡着了（掩面。。。）。

------

最后的最后，这次来苏州最大的一个遗憾就是竟然没有吃到大闸蟹。。。到现在都还是好伤心！
