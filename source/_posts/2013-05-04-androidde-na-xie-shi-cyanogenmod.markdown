---
layout: post
title: "Android的那些事 - CyanogenMod"
date: 2013-05-04 22:07
comments: true
categories: Android
keywords: cyanogenmod maguro 
---

这个系列我想记录一些和Android相关但是和技术无关的东西，这次就从CyanogenMod开始讲起吧。

今天要把一个源码编译到手机上（我的测试机是Samsung Galaxy Nexus），按照以前的经历，首先要把它`lunch`成maguro，然后再开始make，但是这次的lunch发现只有一个叫做cm_maguro的选项，而且关键是它说找不到相关的配置文件，而且`repo sync`也失败了。

讲到这里，我就不想继续下去了，这个就是一个装机的过程，遇到各种错误，然后google，然后。。。这个过程我打算另开一篇，而这一篇是关于这个**cm_maguro**。

明白人都知道，这个“cm”指的就是“CyanogenMod”，而这个maguro呢？据说是Samsung Galaxy Nexus的一个代号而已，名为金枪鱼。这些在我们平时刷ROM的时候经常会出现的字眼，说出来都不好意思，我从来就没有搞清楚过他们之间具体的关系是什么，直到今天，我感觉自己终于有了一点头绪。

先声明下，以下的部分基本上都是从各种百科或者论坛或者wiki再或者是CyanogenMod的官方网站上撸下来之后进行整理的，可能会感觉似曾相识，不过没关系，就当看故事好了。

------

#### CyanogenMod

[CyanogenMod](http://wiki.cyanogenmod.org/w/About)，从人的角度来说，是一个由Android爱好者组成的团队，并且它是目前全球最大的Android第三方编译团队。而从系统的角度来说，它又是一个基于开源Android系统，供某些手机使用的二级市场固件，它提供了一些在官方Android系统或手机厂商没有提供的功能。

那么为什么会出现CyanogenMod呢？

原因很简单，我们知道Android系统从一开始发布就是一个开源的系统，当时Android有另外一个问题就是，代码是开源了，但是那么多机型Google就算再闲也某赖新菜（方言，表示没心情管它，请无视），就拿Android 2.3来说，Google开放的源码只支持Nexus S和Nexus One，而对于其它机型，比如HTC的xxx，LG的xxx，google只能说一声对不住了。虽然不同的手机制造厂商会花力气下去生产自己的ROM（简单来说，就是能把改过的代码跑在自己的机器上），但是作为一个Android用户，如果他想刷机怎么办？开源的代码不支持自己的机型，支持自己机型的ROM又由生产厂商封锁着，那些说好的“新功能”呢？那些说好的“随意刷机”呢？我觉得可以这么说，如果没有像CyanogenMod这样的团队，现在什么牛逼的第三方ROM，什么MIUI，估计都还在娘胎里没生出来吧，也就更不用说今天Android手机的千秋万代，一统江湖了。

<!-- more -->

那么，CyanogenMod这样的团队到底做了些什么呢？

问得好！其实吧，我也只是一知半解。我只是知道，相比于Google只对少数的几款机型的支持，CyanogenMod增加了对很多其它机型的支持，而这些改动主要是在内核部分。这些内核源代码都是各厂商根据GPL协议共开出来的，CM会在上面作一些改动（比如增加收音机，720P录像等）。

也就是说，CM基于Google官方发布的ASOP，每当google发布新版本的ASOP的时候，CM团队都会将它们port到不同的机型上，并且增加一些新的特性、功能和bug的修复等等。也正是因为这样，CM的ROM经常也会为ASOP带来很多好处，有时候CM加上的新特性会在ASOP的新版本中出现，CM对bug的修复也会贡献给ASOP。

另外在[这里](http://wiki.cyanogenmod.org/index.php?title=Devices)可以找到所有CM支持的机型和它们相对应的代号，非常牛逼！

##### ROM，firmware，operating system，distribution

在CM的[官方介绍](http://wiki.cyanogenmod.org/w/About)中特别说明，这四个词对于CyanogenMod来说都是指的同一个意思，都是指你装在你手机设备上的一整套软件。

##### CM版本

我们经常会看到CM7，CM8等等，这些都是CM的版本号，从[wiki](http://en.wikipedia.org/wiki/CyanogenMod#Version_history)上的一张图可以很清楚的看出这些版本都是代表些什么意思：

![CM version](http://ytliu.info/images/2013-05-04-1.png "CM version")

------

#### maguro, toro, tuna

在刷Galaxy Nexus的时候，会碰到这些代号，简单来说，tuna是Samsung Galaxy Nexus的代号，名为金枪鱼（maguro也是金枪鱼的意思）。

这里稍微跑题下，Android的很多机型都是采用和“鱼”相关的代号，比如Galaxy Nexus (tuna, 金枪鱼)，emulator (goldfish, 金鱼)，G1 (trout, 鲑鱼)， Nexus One (mahimahi, 海豚鱼)，Nexus S (herring, 鲱鱼)，Xoom (stingray, 黄貂鱼)等等。

那么maguro和toro又是什么呢？

事实上，Galaxy Nexus有两种类型的设备，一种是GSM/HSPA+的种类，代号为maguro，一种是CDMA/LTE的种类，代号为toro（还有一个它的变种叫toroplus）。对于maguro来说，在Galaxy Nexus的源码树中，一般会有两个目录，一个是`device/samsung/tuna`，一个是`device/samsung/maguro`。前者涵盖了所有maguro和toro共享的文件，而后者则存储和maguro特定相关的文件。

##### 如何识别

有两种方法，一种是查看`Settings > About phone > Model number`，看看以下哪个匹配：

* Samsung Galaxy Nexus (Maguro; GSM/HSPA+) - GT-I9250
* Samsung Galaxy Nexus (Toro; Verizon; CDMA/LTE) - SGH-I515
* Samsung Galaxy Nexus (Toro Plus; Sprint; CDMA/LTE) - SPH-L700

但是在我刷好的手机上，`Model number`显示的只有Galaxy Nexus，所以可以采用第二种更简单的办法，即看手机后盖上的图标，看看它是否有Verizon (toro) 或者 Sprint (toroplus)字样，如果都没有，那就是maguro了。