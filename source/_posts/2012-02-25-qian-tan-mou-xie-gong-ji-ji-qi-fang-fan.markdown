---
layout: post
title: "浅谈某些攻击及其防范"
date: 2012-02-25 21:03
comments: true
categories: Security Paper Attack
---

很多时候我觉得攻击和防范的历史就像一部电视连续剧，不断出现的魔高一尺道高一丈，此消彼长，生生不息……

这篇博文的初衷来源于6.858中lecture2里面最后的一篇references:

        INTERPRETER EXPLOITATION: POINTER INFERENCE AND JIT SPRAYING

这是一篇介绍两个技术*pointer inference*和*JIT spraying*的文章，关于这篇文章我应该会在之后详细介绍，这里想说的又是里面引出的一篇报告：

[Bypassing Browser Memory Protections](http://www.azimuthsecurity.com/resources/bh2008_dowd_sotirov.pdf "bypassing")

里面介绍了微软在软硬件层面对5种攻击所作的防范，以及攻击者可以如何绕开这些防范。这里的描述方式也按文中一样按照防范措施来分类感觉会比较清晰点吧

####**前言**
对于一个成功的攻击，有很大一部分是需要用到*buffer overflow*技术的，而*bf*分为很多，主要看你要覆盖什么内容，之后如何使得控制流变成你想要执行的代码，对于之后的这些攻击技术，也就出现了相应的防范策略和机制。但不管怎么说，这里所提到的所有攻击都源自于*bf*，没有它的攻击属于另外的范畴，这篇暂且不谈。

<!-- more -->

####**stack cookies(canaries) & variable reordering**
*bf*最简单的攻击就是修改函数调用的return address，canaries即是在进入函数栈时在压入的return address之后再加入一个4位的随机数（canaries），如果攻击者改了return address那么canaries也会被改，那么系统在返回时将会报错。但是也有可能在函数返回之前调用一些函数变量，variable reordering则是将变量的顺序进行调整使得攻击者无法对本地变量进行overwrite，这两个技术都是靠编译器支持的，如果开启了这两种保护，栈上的结构将会变成这样：

![stack layout with or without GS protection](http://ytliu.info/images/2012-02-25-1.png "stack layout")

这两种机制的缺点很明显，对于canaries，它只能保护函数返回时的控制流变化，而且*canaries*也有可能被破解，对于*vr*它只支持有限种类的变量reordering，对于一些结构体的保护就不是很好。

####**SafeSEH**
这是微软系统对于它们独特的*Exception handler*机制的保护，叫*Structured Exception Handler*，这里不想详细介绍，简单说就是在每个函数体的栈上除了一些本地变量之外还会存有一个叫做*Exception Handler Record*的东西，里面指向一个Exception Handler的链结构，当在函数执行过程中如果发生Exception，则会从这个record开始查找对应的handler。如果攻击者在overflow时将这个结构的handler的地址改成攻击代码的地址，则会在发生Exception时调用恶意代码。而SafeSEH用了两种方法*SEH handler validation*（维护一张表，在调用handler时检查是否是表中的合法handler）和*SEH chain validation*（enforce chain的一些特征使其在违反时能够被检查出来）。

####**Heap Protection**
这是这篇文章的重点，因为我在这之前虽然听说过*heap overflow*，但也不清楚里面的细节，[这篇文章](http://www.h-online.com/security/features/A-Heap-of-Risk-747161.html "a heap of risk")对此做了一个很详细的描述，这里简单介绍下:

文中举了个简单但是明了的例子，虽然不同系统中对heap的实现大同小异，但原理都是一样的。在本文的例子中，堆的结构是这样的：

![heap layout](http://ytliu.info/images/2012-02-25-2.png "heap layout")

每一个被回收的堆都有一个带有信息的头结构，里面包含了next, prev, size, used的信息，而在free操作过后都会有一个merge的操作，即将相邻的两个free的堆merge起来从而避免fragmentation，在这个merge操作中有一个非常关键的操作：

{% codeblock %}
hdr->next->next->prev = hdr->next->prev;
{% endcodeblock %}

而系统在计算*next->prev*时会先得到next的地址然后加4，于是攻击者就可以通过buffer overflow将堆的头结构改成以下内容（这里要说明下，heap的overflow可以通过integer的overflow来实现，具体可参阅文章前半部分）：

![heap header layout](http://ytliu.info/images/2012-02-25-3.png "heap header layout")

将next中的地址指向栈中存放*return address + 4*的地址，然后*next + 4*即为return address，将其地址通过那步关键操作赋值成攻击者注入的恶意代码的地址，即可达到目的。

heap overflow有一个最大的缺陷就是需要攻击者对系统堆的实现，以及一些内存信息非常了解，而且如果heap若也被标志为NX，则该方法也没有用了。

####**DEP（NX）**
*DEP*（Data Execution Prevention）即为通常所说的*NX*（Non-eXecutable），让除了代码段之外其它地方的数据都不能执行，这在很大程度上防止了攻击者注入的恶意代码的执行，但其有以下几点缺陷：

        * 有些程序就是需要除了代码段的其它section的指令运行，如果用了该技术则会不兼容；
        * return-oriented & return-to-libc攻击。

####**ASLR**
*ASLR*（Address Space Layout Randomization）对一些object的地址做随机化，使得攻击者很难准确地overflow正确的信息，但攻击者并非对此毫无办法：

        * 可以猜测，或者用穷举的办法（JIT spraying就是一个很好的例子）；
        * 可以运行一些代码来获得随机规律；
        * 有时候并不需要知道确切的地址，只要知道相对值就行了。

- - - - - - -


以上是对一些防范和攻击的简单介绍，这周还看了一篇paper叫

        XFI: SOftware Guards for System Address Space

是由微软和UCB在OSDI'06发表的，应该说这是一篇很牛逼的文章，但是可能是因为我功力太浅，实在没怎么看懂，这里就将里面提出的7个properties列出来，如果有兴趣也可以参考MIT的[6.858课程lecture3](http://pdos.csail.mit.edu/6.858/2011/lec/l03-xfi.txt)中对这篇文章提出的几个问题。

**external properties**:

*P1. memory-access constraints*: memory accesses are either into a). the memory of the XFI module, or b). into what host system has granted. read/write/execute handled separately, no write to XFI module's code

*P2. interface restrictions*: control cannot flow out of XFI's code, except via calls to a set of prescribed support routines, and via returns to external call-site.

*P3. scoped-stack integrity*: a). stack register points to at least a fixed amount of writable stack memory? b). accurately reflect function calls, returns and exception; c). Windows stack exception frames are well formed, and linked to each other.

*P4. simplified instruction semantics*: certain machine-code instructions (dangerous, privileged instructions) can never be executed, certain other machine-code instructions may be executed only in a context taht constrains their effects.

*P5. system-environment integrity*: certain aspects of system environment are subject to invariants.

**internal properties**:

*P6. control-flow integrity*: execution must follow a static, expected control-flow graph, even on computed calls and jumps.

*P7. program-data integrity*: certain module-global and function-local variables can be accessed only via static references from the proper instructions in the XFI module.


文中之后所说的细节很多也都是通过*P6*和*P7*来保证整个系统7个properties的，所以说*control-flow integrity*和*data integrity*还是非常重要的，关于CFI，相信这一篇也是理解XFI的关键一文：

        Contro-flow Integrity: Principles, Implementations, and Applications

打算在接下来的一周认真读一下。

另外还有一个由Robert C. Seacord做的presentation：
        
        Pointer Subterfuge: Secure Coding in C and C++

也详细地介绍了在C和C++语言中可能出现的通过篡改pointer来实现攻击的例子，主要介绍了*GOT（Global Offset Table） Entries*, *The .dtors Section*, C++中的*Virtual Pointer*, *atexit()*和*on_exit()*, *setjump()*和*longjump()*，还有就是之前说的*SEH（Structured Exception Handler）*，在这个presentation里面详细介绍了它们的机制和用法，以及攻击者如何利用它们进行攻击的手段等等，是一篇很有趣的报告。

- - - - - - - -


然后简单讲下Oakland'11吧，上周把*IEEE Oakland'11*（四大安全会议之一，其余三个为ACM CCS, USENIX Security and ISOC NDSS）的paper简单浏览的一遍，根据题目挑选了20来篇看了下abstract和introduction，安全会议果然和系统相关会议有蛮大区别的，大部分paper主要是针对一个很小但是很具体的问题进行阐述并解决，不过说实话，我对里面的大部分都不是很有兴趣，但是里面一些paper提出来的一些概念，确实挺让我焕然一新，比如在Hardware Security的Section里面提到的硬件厂商或第三方的恶意backdoor，比如*HomeAlone*中利用Side-Channel来进行防范，还有之前说过的*Virtuoso*，*PRISM*中提到的和我们之前想过的细粒度权限控制的想法有着相似但不同称法的*Multi-Level Secure（MLS）*，以及*RePriv*中关于用户隐私性和个性化的平衡，还有那些当今比较热门的话题，如*Cashier-as-a-Service（CaaS）*、*Sybil Attack*、*Mobile Privacy（location privacy）*、*Side-Channel Attack*，甚至是我不是很感兴趣的*Formalization*，都让我对安全这个领域有了更进一步的了解。之后我又翻阅了下NDSS'12的paper，发现里面一些更让我感兴趣的知识，这也算是我下周的打算。

- - - - - - - 


总之下周注定要成为更忙的一周，重新奋斗hc，加上各种计划各种paper，凡事尽力吧，也无需勉强自己。

最后用这周译言上的那篇文章结尾：[Top five regrets of the dying](http://select.yeeyan.org/view/216596/248984 "top five regrets of the dying")

{% blockquote %}
1. I wish I'd had the courage to live a life true to myself, not the life others expected of me;
2. I wish I hadn't worked so hard;
3. I wish I'd had the courge to express my feelings;
4. I wish I had stayed in touch with my friends;
5. I wish that I had let myself be happier.
{% endblockquote %}

人生真的是如此短暂，"死而无憾"绝非易事，只希望在活着的时候对得起自己，不辜负亲人、朋友，其它的一切又算得了什么呢？
