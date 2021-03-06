---
layout: post
title: "同步原语学习笔记 - Lock，RCU和Transaction Memory"
date: 2013-04-14 14:24
comments: true
categories: System
---

这周最大的收获就是周四那天，感觉被洗脑了一样，上午P哥给我们讲他的[ReEmu](http://ipads.se.sjtu.edu.cn/lib/exe/fetch.php?media=publications:reemu-ppopp13.pdf)，里面讲的最多的就是在它的replay系统中如何记录shared memory access order。还有一个就是Naruil下午讲的“The myth of lock & scalability“，讲了lock，RCU，Transaction memory的一些基本原理和它们的发展过程，以及其中的一些scalability的问题。

关于ReEmu，主要是说需要记录哪些non-determinism：里面主要讲了interrupt，DMA handling，在不同cores发生page fault时的走页表顺序问题，对于一些non-deterministic的指令，比如in/out的处理，以及在共享内存访问中不同核的访问顺序。其中最重要的就是memory access order，即如何记录read-after-write, write-after-write和write-after-read的方法。这篇paper最大的贡献在于它利用了CREW（Concurrent Read，Exclusive Write）和seqlock的特性，利用version极大地减小了原子操作的数量，减小了log的大小，提高了efficiency和scalability。具体如何做的还是直接看paper吧，这里就不详述了。

这次主要想记的是Naruil关于一些同步原语的讲解。之前一直对Read-Write Lock, RCU，Transaction Memory等都只是一知半解，这几周由于海波和大爷的课，以及组会对这些都提到了好多，因此希望能把它们记下来做个笔记，省的自己又很快就忘记了。

<!-- more -->

slides在[这里](http://ipads.se.sjtu.edu.cn/courses/ads/slides/lec7-lock.pdf)。

刚开始有个印象很深的提法，他说在计算机中不同核（或者在分布式系统中不同机器）之间的交流一般只有两种方式：**message passing**和**shared memory**，而对于第二种shared memory的方式，其本质上也是一种message passing（比如在多核之间的SM其实就是inter-processor间cache的message passing）。

还有一个印象很深的知识点就是对于我们平常提到的原子指令（比如test_and_set），只要是对于一个共享变量的原子指令，都会涉及到cache invalidation，即要把其它核的cache line都invalidate，这就是造成在多核环境下scalability上不去的原因，因为过多的原子指令将会造成in-flight的message太多（这个也是那篇[Non-scalable locks are dangerous](http://pdos.csail.mit.edu/papers/linux:lock.pdf)强调的问题）。因此在同步原语中要想获得高的scalability，就必须减少对共享变量的原子操作（其实在ReEmu中它的目的也正是如此）。

### Read-Write Lock

读写锁，顾名思义，就是将读锁和写锁分开来处理，对于读锁，它可以允许多个读者同时对一个共享变量进行读取，但是需要维护一个全局counter来记录读者的数量。当某个核产生一个写的请求，那么就不能再允许有新的读者进入critical section（CS），而写者也需要所有读者结束（即counter降为0）后才可以进入写的CS。在这里，一般的实现方式是这样的：

{% codeblock lang:c %}
Read object begin
	P(object.lock)
	AtomicAdd(object.activeReader, 1)
	V(object.lock)
	Do Actual Read
	AtomicAdd(object.activeReaders, −1)
end
Write object begin
	P(object.lock)
	while object.activeReaders != 0 do delay
	Do Actual Write
	V(object.lock)
end
{% endcodeblock %}

这里值得注意的是，虽然它减小了读者和读者之间信号量（P/V）的竞争，但是读者在读之前需要对共享变量activeReader进行原子加，安装前面所说的，这就会造成cache line invalidate message数量的增加，不具有很好的可扩展性。

为了减小等待message的时间，有一种解决方法叫GOLL:

![GOLL](http://ytliu.info/images/2013-04-14-01.png "GOLL")

即构造一棵二叉树，两个reader共享一个activeReader，而对于writer来说，只要其子节点存在activeReader，则它就为1。

另外一种方法叫BR lock：

![BRLock](http://ytliu.info/images/2013-04-14-02.png "big reader lock")

和GOLL相比，它为每一个reader都维护一个变量，这样在reader之间都完全不会存在message，但是对于writer来说，它的overhead则增大了很多，在writer进入CS之前，需要所有的reader都发message给它。

另外，如果我们为了增加公平性，采用一个ticket lock的形式，那么同样的，如果这个ticket是全局共享的，那么也会有同样的问题，于是就提出了一种叫做MCS lock的机制。

### MCS Lock

MCS是由两个人(Mellor-Crummey and Scott)发明的算法，相比于需要维护一个全局变量的ticket based lock，在MCS lock中，每个进程只需要各自维护一个local variable，这个局部变量可以是一个用于spin的flag，用于标识自己是否拿到锁了。另外，存在一个全局的队列，拿锁的过程相当于向队列中插入一个结构体qnode，这个结构体包含了某个进程的flag——locked，和这个qnode指向的下一个qnode的地址：

{% codeblock lang:c %}
procedure acquire_lock( L: *lock, I: *qnode )
	var pred: *qnode
	I->next := nil // Initially, no successor
	pred := swap(L, I) // Queue for lock
	if pred 6= nil // Lock was not free
		i->locked := true // Prepare to spin
		pred->next := I // Link behind predecessor
		repeat while I->locked // Busy wait for lock
{% endcodeblock %}

在这里，有一个变量`L`，它也是一个qnode，表示全局队列中最后一个拿锁插入的qnode，通过`swap(L, I)`（将I的值赋给L，并返回L的值），得到前一个拿锁的qnode pred，将它的next指向自己，并spin在自己的这个locked局部变量中。

而释放锁的过程也就显而易见了：

{% codeblock lang:c %}
procedure release_lock( L: *lock, I: *qnode )
	if I->next = nil // No known successor
		if compare_and_swap(L, I, nil)
			return // No successor, lock free
		repeat while I->next = nil // Wait for successor
	I->next->locked := false // Pass lock
{% endcodeblock %}

先判断自己是不是最后一个拿锁的人，如果不是，则将下一个qnode的locked设成false，即将锁传递给下一个qnode。

由于MCS lock是spin在自己的局部变量上的，所以就没有前面说过的scalability的问题。不过MCS对于核数比较少的情况似乎性能不是很好（这个我也不是很理解）。

### RCU (Read-Copy-Updata)

除了lock之后，还有一种同步原语叫RCU：

![RCU](http://ytliu.info/images/2013-04-14-03.png "RCU")

RCU最大的特点在于writer可以在reader进行读取操作的时候直接修改，不过需要注意的是，这里的修改是这样的：

* 首先对Shared_data进行一次拷贝（到一个新的地址B）；
* 更新B指向的对象；
* 将Shared_data原来的地址赋给一个新的变量Old_data；
* 将Shared_data赋值为B。

这样有一个问题就是，几个reader可能会在一段时间内读到的值是不一样的，有些读到的是old value，有些读到的是new value。但是可以保证的是，当执行完

{% codeblock lang:c %}
Shared_data = B
{% endcodeblock %}

之后，所有的reader读到的值都是改过后的值！

另外这里还要考虑一个问题，那就是什么时候释放这个Old_data？

这里注意在Reader的代码中，`rcu_read_lock()`会关闭中断，禁止在一个核中其它进程的抢断，而`rcu_read_unlock()`则会打开中断，恢复抢断机制。于是writer就可以根据reader进程是否进行过context switch来判断说是否对Shared_data的读过程是否全部结束了：

![RCU figure 2](http://ytliu.info/images/2013-04-14-04.png "RCU Figure")

RCU还要考虑的一个问题是在reader中是否能允许sleep？因为如果一个reader在读的过程中如果碰到一段IO操作，那么它如果不sleep的话，则会很大地浪费CPU资源，而如果sleep的话，则会发生一次context switch，而此时，读的过程其实并没有结束，与RCU的语义就不符了。

对于这个问题，按照Naruil的说法，也产生了一系列研究，比如允许sleep的RCU等，这里就不展开了（其实是我也不懂啦）。

和之前提到的RW lock等相比，RCU没有scalability的问题，性能会好很多。所以当前RCU被运用到Linux的很多地方，这里有一个插曲是说当年发明RCU的是作者是IBM的一个员工，它就靠着RCU这么一个东西在IBM待了20多年，而RCU也被申请了专利，像FreeBSD这些系统是不被允许使用RCU的。

当然RCU还有很多问题，比如它有很多semantic的限制，对于那些不允许在一段时间内多个reader会读到不同值的情况就不能使用RCU。另外，RCU只能应用在一些比较简单的数据结构中，如果遇到比较复杂的数据结果就挂掉了，因为它需要程序员自己处理很多corner cases。

### Transaction Memory

最后要讲的一个同步原语就是Transaction Memory（TM），这个最近几年很火的一个研究方向，特别是当Hardware TM（HTM）问世之后。

安装Naruil的说法，TM的原理大概是这样的：在一个Transaction中，可以访问不同的对象（这个粒度可以自己控制，可以是几个cache line，几个page...)，而当访问了某个对象之后，它对应的cache line上就会有一个flag标识，当一个进程进行commit，则会将自己对应的那些cache line上的flag清楚，而对于其它访问了这些cache line的transaction，则会abort，并invalidate 这些cache line。

TM的好处在于它可以非常细粒度地进行同步，并且避免了对lock这样一个额外的同步变量维护产生的cache message。

------

同步原语真的是一个非常需要智商的领域，我最近对此感觉特别深刻。我想在这个领域里搞研究的应该都是一群智商超群的人吧:)
