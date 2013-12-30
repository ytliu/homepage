---
layout: post
title: "Share mouse and keyboard using Synergy"
date: 2013-09-12 13:04
comments: true
categories: Tool
---

Today Han shows me an interesting functionality of one software called [Synergy](http://synergy-foss.org/):

> Share your mouse and keyboard between multiple computers on your desk

Since I have at least 2 machine in my Desktop (One Windows, one Linux, not mentioned of my Mac), so I've 2 mouses, 2 keyboards, and the wires are really not easy to place in clearness. So Synergy is exactly the fantastic software I need, you only need one mouse, one keyboard, and share them among all these machines and OSes!

So the way how to realize this is as follows:

<!-- more -->

First you need to download the install files of both windows and linux versions from [here](http://synergy-foss.org/download/?list), Han tells me that the 64bit version of windows has some problems in his trial, I just download the 32bit version for Windows, and 64bit version for Linux.

##### Windows installation 

Then in Windows, install it, choose the `Server` mode, configure that using any of the encryption mode, and in the popup of `Configure Server`, drag the Computer icon in the right-top to the center, and assign the hostname of the Linux client to it:

![Synergy-Windows](http://ytliu.info/images/2013-09-12-1.png "Synergy in windows")

Then click OK, and just start!

##### Linux installation 

In Linux, install the deb file using:

	$ sudo dpkg -i syner-gy-1.4.12-Linux-x86_64.deb

then run synergy in background:
	
	$ synergy &

In the popup configuration, choose the `Client` mode, configure to the same encryption mode as in Windows, and fill the Windows IP address in, finally start!

------

After all of the above done, your mouse can move between the two monitors seamlessly, as well as your only keyboard can be shared! How amazing and usefull of Synergy! Many thanks to the author Nick Bolton!