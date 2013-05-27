---
layout: post
title: "Paper reading - Mobile security survey &amp; DS failure detection"
date: 2011-12-23 13:38
comments: true
categories: Paper
---

今天考坑爹的专题讲座，昨天组会的paper reading只能拖到现在写了。    
先稍微提下为什么要有这个section吧~我一直觉得自己进PPI一年半，开过的组会听过的paper到现在为止大部分都不记得了，效率太尼玛低下了！于是乎我就想把自己一些比较有感触的paper整理下，至少忘得会慢些吧~

好了，废话结束。进入正题。

昨天讲了两篇paper，一篇是S&P的security section的，Z神讲的，这篇其实和我们现在的方向挺相关的，不过确实没有任何创新，是一片完完全全的survey，title is 
        "Mobile Security Catching up? Revealing the Nuts and Boits of the Security of Mobile Devices"
作者是Michael Becher，它介绍了现在手机上存在的一些安全问题，包括和Desktop的比较，以及一些attack model，比如：
        Hardware-centric attack
                MITM attack - 中间人攻击
                JTAG attack
                forensic attack - 没听懂。。。
        Device-independent - 和普通服务器上的攻击差不多
        Software-centric attack
        user layer attack
其中，关于JTAG attack,
{% blockquote @Nate Lawson http://rdist.root.org/2007/04/06/jtag-attacks-and-pr-submarines %}
JTAG port is used for factory and field diagnostics and provides device-specific access to the internal flip-flops that store all the chip’s state.
Since JTAG access gives the hardware equivalent of a software debugger, attackers have been using it from the beginning. The first attackers were probably competitors reverse engineering designs to copy them or improve their own. Currently, a packaged version of this attack has been in use for years to get free satellite TV. 
{% endblockquote %}

还有关于Software-centric attack, 小Z讲了一个例子，比如说对iOS的pdf漏洞的良性利用，至于用来干嘛的，大家都懂得，jailbreak，不过其实我完全不知道这个是怎么样就得到root权限的，这几天研究下。

<!-- more -->

最后还有一点比较感兴趣的是小Z提到android的process isolation，我挺感兴趣的，它说android里面是用到context的概念进行进程间通讯，在kernel里面有一个binder进行管理，如果把kernel作为TPM那应该就没有什么安全问题了吧，不过其实还是存疑的，我不知道有没有一份关于android的安全机制的survey，上网搜了下，找到一篇宾大的"understand android security"，抽空看下。

其实这篇是一个完完全全的survey，小Z讲的也比较简单，不过其实手机的安全问题还是蛮多的，比如有提到的一个隐私的保护，有很多这方面的研究，比如通过限制控制流，比如TaintDroid，我想还有没有更强的机制呢？比如把Nicholai的histar用在手机上？

- - - - - - - 

另一篇是SOSP'11的
        "Detecting Failures in Distributed Systems with the FALCON Spy Network"
其实是一篇想法很简单的paper，只不过之前的人没有把这个问题当做一个问题罢了。想法简单来说是这样的，现在在distributed system里面判断节点是否挂掉主要用的是end-to-end timeout机制，这样的缺点是发现节点挂掉可能会有比较长的延迟，比如说60s，那么在这60s里面机器A可能已经down掉了但却还被分配了任务从而造成availability变差，而这篇的目标就是：
        Fast detection + reliability,
用的方法也很直接：gather inside information, avoid end-to-end timeout。相当于在运行应用程序主机的每一层跑一个spy（说白了就是和应用程序同个进程的一个线程），而每一层包括应用程序、OS、VMM，甚至是switch，用两个图就很容易理解：

![FALCON's Model](http://ytliu.info/images/2011-12-23-1.png "FALCON's Model")
![FALCON's Implementation](http://ytliu.info/images/2011-12-23-2.png "FALCON's Implementation")

从后来的evaluation来看它的detection速度确实变小好多，CPU的overhead也很小，不过我觉得有两个问题：    
第一，在分布式系统中一台机器挂掉的几率可能很小，真的有必要为这么小的几率做一件这么复杂的事吗？比如原来是10秒钟检查到一个一个月才挂一次的机器现在用1秒钟来检查，却要付出一直用client和spy交互的代价？      
第二，如Model里每一层的spy都要和client端交互，那么网络带宽就要占用很多吧，为什么作者不测试网络带宽的占用率呢？CPU的overhead小可能是因为本来服务器的程序就不是CPU intensive的，但是确实IO intensive，那为什么不说下网络带宽被额外占用多少呢？还有，对于作者提出的架构来说，每一层spy与client的交互是走network的，如果用PV或HVM的话都得经过最下层的switch，那么按照作者的逻辑，既然其中一层的spy挂了就相当于全部挂了，那为什么不仅仅利用最下层的switch的spy提供的信息来判断呢？这样岂不是可以节省网络带宽？

The End.
