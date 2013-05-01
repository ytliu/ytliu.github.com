---
layout: post
title: "connect smartphone to android emulator"
date: 2012-12-01 15:37
comments: true
categories: Android
---

这两天碰到一个难题，如何让手机中的某应用程序通过网络连接PC中的android emulator里面的服务。

即我在emulator里面开启了一个服务，该服务通过socket监听2012端口，那么在我的手机中，如何通过socket连接到这个服务中，也就是说，我的IP要设为什么，端口号要设为什么？

这个问题在于PC为emulator分配了一个端口号（这里为5554），所以在PC中可以很方便地和emulator进行交互，但是emulator并没有一个对外可见的IP地址。

在[stackoverflow](http://stackoverflow.com, stackoverflow)上查了好久，有一个说[如何连接两个emulator的](http://stackoverflow.com/questions/7108326/how-can-we-change-the-ip-of-android-emulator)，还有一个[介绍了一个非常复杂但经过我的测试有效的方法](http://stackoverflow.com/questions/7108326/how-can-we-change-the-ip-of-android-emulator)。这里简单介绍一下：

<!-- more -->

### 两个emulator相连

为了两个emulator能进行网络相连，首先必须满足一下三个条件：

* A为PC
* B为运行在A中的一个emulator，端口为5554
* C为运行在A中的另一个emulator，端口为5556

是不是很废话...好了，进入正题，我们要做的就是一下三步：

在B中起一个server，可以让C进行连接

    B listening on 10.0.2.15:80

telnet到B的console里面对端口进行重定向：

    $ telnet localhost 5554
    # redir add tcp:8080:90

在C中连接10.0.2.2:8080就可以和B进行交互了。

这里补充一下，你如果进入adb shell在里面查看自己的IP会发现其为10.0.2.15，其通过10.0.2.2和PC进行连接。redir相当于将到达PC的8080端口的信息全部重定向到5554端口emulator的80端口。这样就完成而来emulator之间的连接了。

------

### 手机和emulator相连

照理来说按照上面的说法，手机和emulator的连接应该也很简单，只需要将手机连接到PC的ip加上一个端口P1，然后将P1重定向到emulator中server监听的端口P2就可以了。

但是显示总是残酷的，包括redir和adb forwarding（功能似乎和redir是一样的）都只能将PC自己向端口P中写的信息forward到emulator里面的端口，而不能把其它地方从端口进来的forward。

于是在[这个问题](http://stackoverflow.com/questions/7108326/how-can-we-change-the-ip-of-android-emulator question)的回答中提到一种方法，在中间再加一个proxy，接下来我来介绍下这种方法：

这是现在的框架：

![framework](http://ytliu.github.com/images/2012-12-01-1.png "framework of smartphone and emulator")

原理和两个模拟器相连很像，但是中间加了一个proxy：

* 首先在emulator中server监听2012端口
* 在PC上通过*adb forward*命令将PC上的2012端口forward到emulator中的2012端口（*adb forward tcp:2012 tcp:2012*）
* 在proxy中将从外面来的2013端口forward到2012端口
* 而在手机端则直接向PC的2013端口发送数据

经过尝试这样是可行的。

------

### tap/tun

虽然说上面的方法是可行的，但是并没有很好地解决这个问题，因为如果是通过这种方法，则我的emulator没开启一个server，都要起一个相应的proxy，这样就显得很废。有没有什么一劳永逸的方法呢？

听斌哥说之前他在配qemu的时候通过tap/tun的技术来配置qemu的网络，于是我也尝试着配置tap/tun，不过虽然已经将其在PC中启动好了，但是由于对其原理还是不清楚，所以还是没有搞明白要如何使得emulator有一个自己独立的对外可见的IP，这块内容可能会等自己完全搞明白了再写吧。

这里推荐下wikibook中对qemu networking的配置：

http://en.wikibooks.org/wiki/QEMU/Networking
