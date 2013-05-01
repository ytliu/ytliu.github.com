---
layout: post
title: "Netfilter学习笔记（二）"
date: 2013-03-29 20:26
comments: true
categories: Network
---

之前讲了关于netfilter和iptables的一些简单的原理和用法，对于iptables来说，如果只能对封包进行DROP、ACCEPT操作，那么就显得太弱了，其实在我看来iptables里面filter表中最牛逼的就在于QUEUE（NFQUEUE）这个target了。

那么当iptables将封包插入QUEUE后，用户态又能用什么方法才能读到queue中的数据呢？

这里介绍两种方法，一种是C语言中使用的`libipq`，一种是python中使用的`nfqueue`。

<!-- more -->

### libipq

`libipq`是一个对开发者提供的用于读取iptables queue的C库，具体的用法可以参看[linux man page](http://linux.die.net/man/3/libipq)和[这里](http://www.imchris.org/projects/libipq.html)的用法，需要注意的是在我的机器中必须得再加三个头文件：
	
	#include <netinet/in.h>
	#include <linux/ip.h>
	#include <linux/tcp.h>

另外编译出来的文件必须得用`sudo`执行！！！

这里主要用了一个数据结构

{% codeblock lang:c %}
struct ipq_handle *h;

h = ipq_create_handle(0, PF_INET);
{% endcodeblock %}

另外需要设置一个模式，这里是`IPQ_COPY_PACKET`，即是将queue中的封包的payload和header一起拷贝到用户空间：

{% codeblock lang:c %}
status = ipq_set_mode(h, IPQ_COPY_PACKET, BUFSIZE);
{% endcodeblock %}

之后通过：

{% codeblock lang:c %}
status = ipq_read(h, buf, BUFSIZE, 0);
{% endcodeblock %}

将queueu中的封包一个一个拷贝到用户空间，由用户进行操作，用户可以通过：

{% codeblock lang:c %}
ipq_message_type(buf)
{% endcodeblock %}

得到包的类型，可能是`NLMSG_ERROR`，也有可能是`IPQM_PACKET`，如果是后者，即为一个正常的封包，可以通过类似于如下的代码对封包进行操作：

{% codeblock lang:c %}
case IPQM_PACKET: {
	ipq_packet_msg_t *m = ipq_get_packet(buf);

	struct iphdr *ip = (struct iphdr*) m->payload;

	struct tcphdr *tcp = (struct tcphdr*) (m->payload + (4 * ip->ihl));

	int port = htons(tcp->dest);        	

	status = ipq_set_verdict(h, m->packet_id, NF_ACCEPT, 0, NULL);
	if (status < 0)
		die(h);
		break;
	}
}
{% endcodeblock %}

上段代码的意思是先从`buf`中获得整个封包m，之后可以通过`struct iphdr`和`struct tcphdr`获得ip包头和tcp包头，最后有一个最关键的代码：

{% codeblock lang:c %}
status = ipq_set_verdict(h, m->packet_id, NF_ACCEPT, 0, NULL);
{% endcodeblock %}

这里的意思相当于原来在规则中target设为ACCEPT，当然也可以设置成`NF_DROP`等。

在这里我有一个疑问，就是这里除了获得`struct tcphdr`之外，我没有找到和TCP的payload相关的结构，不知道该如何获得。

------

### nfqueue

和C的libipq比起来，支持python的nfqueue会显得强大很多，特别是和[scapy](http://www.secdev.org/projects/scapy/)结合起来用的时候。

首先需要说明的是在iptables中的target除了之前提到的五项（ACCEPT，DROP，RETURN，QUEUE，other_chain）之外，还有一个叫`NFQUEUE`，它是QUEUE的扩展。相比于QUEUE，它可以由用户指定不同的queue number。

在使用nfqueue之前，需要安装如下的包：

	$ sudo aptitude install libnetfilter-queue-dev
	$ sudo aptitude install nfqueue-bindings-python
	$ sudo aptitude install python-scapy

之后就可以采用python对NFQUEUE进行操作了。

假设我们将封包从主机A（`192.168.1.1`）传输到主机B（`192.168.1.2`）时，需要对封包进行分析，如果是TCP协议的包，并且其flags为 ACK|PSH 的话，则将其payload进行修改（比如替换成“hack”）：

首先，需要先在主机A中对iptables进行操作：

	$ sudo iptables -A OUTPUT -d 192.168.1.2 -p tcp -j NFQUEUE

然后利用下面的代码：

{% codeblock lang:python %}
import os,sys,nfqueue,socket
from scapy.all import *

def ch_payload_and_send(pkt):
	pkt[TCP].payload == "hack"
	send(pkt, verbose=0)

def process(i, payload):
	data = payload.get_data()
	pkt = IP(data)

	# Check if TCP flags is ACK|PSH
	if pkt[TCP].flags == 24:
		# Dropping the packet
		payload.set_verdict(nfqueue.NF_DROP)
		ch_payload_and_send(pkt)
	else:
		# Accepting the packet
		payload.set_verdict(nfqueue.NF_ACCEPT)
	
def main():
	q = nfqueue.queue()
	q.open()
	q.unbind(socket.AF_INET)
	q.bind(socket.AF_INET)
	q.set_callback(process)
	q.create_queue(0)

	try:
		q.try_run()
	except KeyboardInterrupt:
		print "Exiting..."
		q.unbind(socket.AF_INET)
		q.close()
		sys.exit(1)

main()
{% endcodeblock %}

这里用到了`scapy`这个非常牛逼的模块，它可以直接通过如`IP()`，`TCP()`等直接对包进行解释和操作，非常方便，具体的可以参看它的[文档](http://www.secdev.org/projects/scapy/doc/)。这里只是说明下它的安装方式：

	$ wget scapy.net
	$ mv index.html scapy-latest.zip
	$ chmod +x scapy-latest.zip
	$ mv scapy-latest.zip /usr/local/bin/scapy

然后就可以运行：

	$ sudo scapy

直接开启scapy的交互模式了。


