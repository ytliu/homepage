---
layout: post
title: "Netfilter学习笔记（三）"
date: 2013-04-09 21:21
comments: true
categories: Network Android
---

这次的内容主要是关于在Android上如何使用iptables。

其实在android源码中已经有对iptables进行了支持，如何确定可以参看[这里](http://www.roman10.net/how-to-build-and-use-libnetfilter_queue-for-android/)。

这里简单说下如何采用nfqueue对其进行控制：

首先说下一个很简单的场景：我们需要将从手机端发到192.168.1.2服务器的所有TCP包都拦截下来，将包的信息打印出来，并将其发送出去。

<!-- more -->

具体的步骤如下：

* 首先将手机连到PC上，运行：

	$ adb shell
	$ su
	#

进入root模式，这个时候可以配置iptables：

	# iptables -A OUTPUT -t tcp -d 192.168.1.2 -j NFQUEUE

这样到192.168.1.2的所有TCP包都会进入NFQUEUE等着被处理。

* 此时我们需要写程序来处理NFQUEUE里面的包，首先我们在PC上新建一个Android项目：

	$ android create project --target <target_ID> -name IptablesTest --path ./IptablesTest --activity IptablesTestActivity  --package com.iptables.test
	$ cd IptablesTest
	$ mkdir jni

接下来，要下载两个包（[libnfnetlink](http://www.netfilter.org/projects/libnfnetlink/downloads.html)和[libnetfilter_queue](http://www.netfilter.org/projects/libnetfilter_queue/downloads.html)）到jni目录下。

然后在jni目录下创建一个文件叫做nfqnl_test.c：

{% codeblock lang:c %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>
#include <linux/types.h>
#include <linux/netfilter.h>		/* for NF_ACCEPT */
#include <errno.h>

#include <libnetfilter_queue/libnetfilter_queue.h>

/* returns packet id */
static u_int32_t print_pkt (struct nfq_data *tb)
{
	int id = 0;
	struct nfqnl_msg_packet_hdr *ph;
	struct nfqnl_msg_packet_hw *hwph;
	u_int32_t mark,ifi; 
	int ret;
	unsigned char *data;

	ph = nfq_get_msg_packet_hdr(tb);
	if (ph) {
		id = ntohl(ph->packet_id);
		printf("hw_protocol=0x%04x hook=%u id=%u ",
			ntohs(ph->hw_protocol), ph->hook, id);
	}

	hwph = nfq_get_packet_hw(tb);
	if (hwph) {
		int i, hlen = ntohs(hwph->hw_addrlen);

		printf("hw_src_addr=");
		for (i = 0; i < hlen-1; i++)
			printf("%02x:", hwph->hw_addr[i]);
		printf("%02x ", hwph->hw_addr[hlen-1]);
	}

	mark = nfq_get_nfmark(tb);
	if (mark)
		printf("mark=%u ", mark);

	ifi = nfq_get_indev(tb);
	if (ifi)
		printf("indev=%u ", ifi);

	ifi = nfq_get_outdev(tb);
	if (ifi)
		printf("outdev=%u ", ifi);
	ifi = nfq_get_physindev(tb);
	if (ifi)
		printf("physindev=%u ", ifi);

	ifi = nfq_get_physoutdev(tb);
	if (ifi)
		printf("physoutdev=%u ", ifi);

	ret = nfq_get_payload(tb, &data);
	if (ret >= 0)
		printf("payload_len=%d ", ret);

	fputc('\n', stdout);

	return id;
}
	

static int cb(struct nfq_q_handle *qh, struct nfgenmsg *nfmsg,
	      struct nfq_data *nfa, void *data)
{
	u_int32_t id = print_pkt(nfa);
	printf("entering callback\n");
	return nfq_set_verdict(qh, id, NF_ACCEPT, 0, NULL);
}

int main(int argc, char **argv)
{
	struct nfq_handle *h;
	struct nfq_q_handle *qh;
	struct nfnl_handle *nh;
	int fd;
	int rv;
	char buf[4096] __attribute__ ((aligned));

	printf("opening library handle\n");
	h = nfq_open();
	if (!h) {
		fprintf(stderr, "error during nfq_open()\n");
		exit(1);
	}

	printf("unbinding existing nf_queue handler for AF_INET (if any)\n");
	if (nfq_unbind_pf(h, AF_INET) < 0) {
		fprintf(stderr, "error during nfq_unbind_pf()\n");
		exit(1);
	}

	printf("binding nfnetlink_queue as nf_queue handler for AF_INET\n");
	if (nfq_bind_pf(h, AF_INET) < 0) {
		fprintf(stderr, "error during nfq_bind_pf()\n");
		exit(1);
	}

	printf("binding this socket to queue '0'\n");
	qh = nfq_create_queue(h,  0, &cb, NULL);
	if (!qh) {
		fprintf(stderr, "error during nfq_create_queue()\n");
		exit(1);
	}

	printf("setting copy_packet mode\n");
	if (nfq_set_mode(qh, NFQNL_COPY_PACKET, 0xffff) < 0) {
		fprintf(stderr, "can't set packet_copy mode\n");
		exit(1);
	}

	fd = nfq_fd(h);

	for (;;) {
		if ((rv = recv(fd, buf, sizeof(buf), 0)) >= 0) {
			printf("pkt received\n");
			nfq_handle_packet(h, buf, rv);
			continue;
		}
		if (rv < 0 && errno == ENOBUFS) {
			printf("losing packets!\n");
			continue;
		}
		perror("recv failed");
		break;
	}

	printf("unbinding from queue 0\n");
	nfq_destroy_queue(qh);

	printf("closing library handle\n");
	nfq_close(h);

	exit(0);
}
{% endcodeblock %}

这里的原理很简单，就是通过：

{% codeblock lang:c %}
qh = nfq_create_queue(h,  0, &cb, NULL);
{% endcodeblock %}

注册了一个callback函数cb，在

{% codeblock lang:c %}
nfq_handle_packet(h, buf, rv);
{% endcodeblock %}

的时候就会将收到的包传给该函数进行处理，当然了，还可以像[这里](http://ytliu.info/blog/2013/03/29/netfilterxue-xi-bi-ji-%28er-%29/)一样通过`struct iphdr`和`struct tcphdr`结构体来获得payload里面的IP包头和TCP包的信息，对其进行处理。

* 之后在jni目录下创建`Android.mk`文件：

{% codeblock lang:c %}
#LOCAL_PATH is used to locate source files in the development tree.

#the macro my-dir provided by the build system, indicates the path of the current directory

LOCAL_PATH:=$(call my-dir)
 

#####################################################################

#            build libnflink                    #

#####################################################################

include $(CLEAR_VARS)

LOCAL_MODULE:=nflink

LOCAL_C_INCLUDES := $(LOCAL_PATH)/libnfnetlink-1.0.0/include

LOCAL_SRC_FILES:=\

    libnfnetlink-1.0.0/src/iftable.c \

    libnfnetlink-1.0.0/src/rtnl.c \

    libnfnetlink-1.0.0/src/libnfnetlink.c

include $(BUILD_STATIC_LIBRARY)

#include $(BUILD_SHARED_LIBRARY)
 

#####################################################################

#            build libnetfilter_queue            #

#####################################################################

include $(CLEAR_VARS)

LOCAL_C_INCLUDES := $(LOCAL_PATH)/libnfnetlink-1.0.0/include \

    $(LOCAL_PATH)/libnetfilter_queue-1.0.0/include

LOCAL_MODULE:=netfilter_queue

LOCAL_SRC_FILES:=libnetfilter_queue-1.0.0/src/libnetfilter_queue.c

LOCAL_STATIC_LIBRARIES:=libnflink

include $(BUILD_STATIC_LIBRARY)

#include $(BUILD_SHARED_LIBRARY)
 

#####################################################################

#            build our code                    #

#####################################################################

include $(CLEAR_VARS)

LOCAL_C_INCLUDES := $(LOCAL_PATH)/libnfnetlink-1.0.0/include \

    $(LOCAL_PATH)/libnetfilter_queue-1.0.0/include

LOCAL_MODULE:=nfqnltest

LOCAL_SRC_FILES:=nfqnl_test.c

LOCAL_STATIC_LIBRARIES:=libnetfilter_queue

LOCAL_LDLIBS:=-llog -lm

#include $(BUILD_SHARED_LIBRARY)

include $(BUILD_EXECUTABLE)
{% endcodeblock %}

* 之后，调用ndk-build来创建可执行文件nfqnltest，它位于libs目录下。

* 将nfqnltest传进Android中：

	$ adb shell
	$ su
	# mkdir /data/data/nfqnltest
	# chmod 777 /data/data/nfqnltest

打开一个shell：

	$ adb push libs/nfqnltest /data/data/nfqnltest/

转回刚刚那个shell

	# cd /data/data/nfqnltest
	# ./nfqnltest

这样整个过程就完成了。
