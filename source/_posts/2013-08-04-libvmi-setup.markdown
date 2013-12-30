---
layout: post
title: "Libvmi Setup"
date: 2013-08-04 10:50
comments: true
categories: Security Tool
---

These days I was busy leanrning and trying one of the famous Virtual Machine Introspection (VMI) framework —— [libvmi](https://github.com/bdpayne/libvmi), it is a library which provides lots of serviceable APIs for programmer to develop introspection tools.

As we know, one of the foremost problems of VMI is bridging semantics gap between protected and security VMs, *libvmi* provides APIs 
helping you to access the memory of a running virtual machine, more specifically, it provides primatives for accessing this memory using physical or virtual addresses and kernel symbols. I will discuss about that in the coming blog introducing libvmi usage.

------

Now it's the main topic of this blog: how to setup libvmi?

Since libvmi currently support Xen and KVM, and Xen is my preference, following I will take xen as example platform.

<!-- more -->

#### Xen install

Before setup libvmi, we need to install Xen first:

One of the most straight forward way to install xen is using Debian's aptitude:

	$ sudo aptitude install xen-linux-system-2.6 libc6-xen bridge-utils xen-tools

However, I cannot compile libvmi successfully in such environment, and after seeking in libvmi's group solution, I need to compile xen from source code. Before that, I need to uninstall the previous installed xen environment:
	
	$sudo dpkg -l | grep xen
	purge them except Dom0 kernel:
	$ sudo dpkg -P libxenstore3.0
	…

Then download the latest 4.3.0 source tarball from [xen.org](http://www.xenproject.org/) and do the usual:

	$ make xen
	$ ./configure

Error: unable to find xgettext, please install xgettext

	$ sudo aptitude install gettext

Error: unable to find as86, please install as86

	$ sudo aptitude install bcc

Error: unable to find iasl, please install iasl

	$ sudo aptitude install sail

Error: unable to find a uuid library

	$ sudo aptitude install uuid-dev

Error: unable to find yawl, please install yajl

	$ sudo aptitude install libyajl-dev
	$ sudo aptitude install libpixman-1.dev

 After install all these pre-required library:

	$ make tools	
	$ make stubdom
	$ sudo make install-xen
	$ sudo make install-tools PYTHON_PREFIX_ARG=
	$ sudo make install-stubdom

Then Xen is successfully compiled.

#### Domain 0 install

After xen is installed, we need to compile the domain 0 ourselves.

I download the latest linux kernel (here is 3.10.3) from [kernel.org](https://www.kernel.org), then:

	$ make menuconfig

in the menuconfig, I choose the virtualization config options and some required device driver (specifically the SATA and SCSI ones), and then:
	
	$ make -j4
	$ make modules
	$ make modules_install
	$ sudo make install

Then domain 0 is also successfully compiled, and after we use 

	$ update-grub

the `grub.cfg` file in `/boot/grub/` will traverse the `/boot/` directory and fill all choiceable Xen and kernel image in the grub config during booting.

Here I did not create the initrd of the kernel, because I've already compiled the essential driver in my kernel.

#### Domain U setup

After the above done, we reboot, and enter into the Xen environment we just compiled, and are ready to setup our DomU.

For our HVM DomU setup, we first need to provide a ISO image for DomU install —— `ubuntu.iso`, and `dd` for a 10G image:

	$ dd if=/dev/zero of=ubuntu.img bs=1000 count=0 seek=$[1000*1000*10]

Then edit our HVM config file: 

{% codeblock lang:xml %}
kernel = "hvmloader"
builder='hvm'
memory = 2048
name = "ubuntu"
vif = [ 'bridge=xenbr0' ]
disk = [ 'file:diretory-to-domu/ubuntu.img,hda,w', 'file:diretory-to-domu/ubuntu.iso,hdc:cdrom,r' ]
sdl=0
opengl=1
vnc=1
vncpasswd=''
stdvga=0
serial='pty'
tsc_mode=0
{% endcodeblock %}

Then in the terminal, run following command:

	$ sudo xm create ubuntu.hvm

After that, we can connect to our DomU using vnc:

	$ gvncviewer localhost

------

#### libvmi setup

After all the above environment is ready, we can now compile our libvmi and try to use some of its examples:

We first download the source code from [here](https://github.com/bdpayne/libvmi), and `cd` enter it,
	
	./autogen.sh

Error: could not find libtoolize or glibtoolize

	$ sudo aptitude install libtool

Error: aclocal not found

	$ sudo aptitude install automake autoconf

Then:

	./configure

Error: Package requirements (glib-2.0 >= 2.16) were not met

	$ sudo aptitude install libglib2.0-dev

Error: Package requirements (check >= 0.9.4) are not met:

	$ sudo aptitude install check

Then:
	
	$ make
	$ sudo ldconfig
	$ sudo make install  (optional)

Actually, after we successfully `make`, we can already use it. Before that, we firstly need to provide a config file: `/etc/libvmi.conf`:

{% codeblock lang:xml %}
ubuntu {
    sysmap      = "directory-to-sysmap/System.map-3.5.0-23-generic";
    ostype      = "Linux";
    linux_tasks = 0x240;
    linux_name  = 0x460;
    linux_mm    = 0x278;
    linux_pid   = 0x2b4;
    linux_pgd   = 0x48;
}
{% endcodeblock %}

These options are:

Option   			| Description
:-----------: 		| :-----------
ostype		  		| Linux or Windows guests are supported
sysmap	      		| The path to the System.map file for the VM
linux_tasks	      	| Offset to task_struct->tasks in linux/include/linux/sched.h
linux_mm   			| Offset to task_struct->mm
linux_pid      		| Offset to task_struct->pid
linux_name 			| Offset to task_struct->name
linux_pgd			| Offset to mm_struct->pgd

Also, libvmi provide a tool in `libvmi/tools/linux-offset-finder/`, you can copy this directory to the DomU, compile it, and then:

	$ sudo insmod findoffsets.ko

then, look the log of the system:

	$ dmesg

to get these offsets automatically.

Meanwhile, libvmi provide some straight forward examples, like `process-list`, `dump-memory`, etc., and we can use them (process-list as example):

	$ sudo ./examples/process-list ubuntu

Here `ubuntu` means the name of the DomU, the same as shown when we run:

	$ sudo xm list

Then it will list all of the processes running in the ubuntu DomU.

------

Other usages of libvmi, as well as how to write our own introspection tools is introduced [here](http://ytliu.info/blog/2013/08/14/write-introspection-tools-using-libvmi/).