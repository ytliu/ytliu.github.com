---
layout: post
title: "Dr. Android and Mr. Hide - xidashi引发的故事"
date: 2013-03-01 20:00
comments: true
categories: Android Security
---

故事得从几天前利小哥发现的[xidashi](www.xidashi.com)说起：

![xidashi](http://ytliu.info/images/2013-03-01-1.png "xidashi")

洗大师是一个android的安全应用软件+第三方应用市场，它的功能和特点简单来说就是：

> 在不root的情况下，允许用户对已安装的应用程序实行权限的细粒度动态控制

这里的细粒度动态控制是指用户可以随时允许或禁止该应用程序某一个权限，并对利用该权限调用API的行为进行log。

洗大师的流程是这样子的：

上传apk -> 服务器对apk进行”清洗“ -> 返回”清洗“过后的apk并安装 -> 在apk运行过程中进行动态权限控制

当然还有一种模式就是”直接下载洗大师market提供的应用“。

<!-- more -->

咋一看，这真的是一个非常牛逼的应用，在没有root的情况下可以对每个应用进行细粒度动态权限控制，这样用户就再也不用担心恶意程序利用过渡申请的权限做坏事了！

而且从实现上来看，利小哥刚开始写了一个很简单的测试程序，用洗大师”洗“了洗，发现它就加了一个包，而没有对原来的程序进行任何修改，我们都觉得不可思议！amazing！！！讨论了下觉得这是不可能的，然后斌哥就在网上找到了这一篇文章[Dr. Android and Mr. Hide: Fine-grained security policies on unmodiﬁed Android](http://www.cs.umd.edu/~jfoster/papers/acplib.pdf)，这是一篇基于spsm2012的technical report，然后再重新写了个测试程序用洗大师”洗“了洗，发现其实它还是改了应用程序原来的binary的。于是乎，就感觉被骗了一样。不过仔细想来，其实洗大师这种方式不失为一种解决恶意程序权限泛滥的好办法，不知道洗大师的作者是看了这篇论文做出的洗大师还是自己想出来的，这种创业产品还是比较有效的，特别是如果它的第三方市场能够更加普及一点的话。当然啦，我们都觉得这个技术并不是一个高不可攀的技术，对于腾讯、360这种xx公司来说，要实现类似的功能应该还是挺快的。我去关注了下洗大师的新浪微博，发现它现在的粉丝并不多，不知道洗大师最后能牛逼到什么程度，祝君好运吧。

------

扯了那么多闲话，开始进入正题，这一篇《Dr. Android and Mr. Hide》从效果上来说和洗大师还是有一点不同的，但是我觉得实现原理应该不会差太远，这里简单介绍下吧：

它的motivation是这样的：

现在Android的权限系统粒度太粗了，举个例子：如果一个应用程序，比如”大众点评“（点评躺着中枪了），申请了一个INTERNET权限，那么它就可以访问所有的网络了，但是实际上它只需要访问www.dianping.com这一个域名，那么一个恶意程序或者repackage的程序就可以利用它的网络权限窃取一些隐私数据传到某个服务器，这种方式并没有违反Android的权限系统。那么这篇paper的目的就是在不修改Android Framework的情况下将权限细化，比如把INTERNET改成InternetURL(d)，使得有后者权限的应用只能访问`d`这个URL。

这里需要说明的是，在这种安全防护模式下有两种方法可以做：

* 修改Android Framework，换一种方式，从用户的角度来说也就是root，或者刷机；
* 对应用程序进行instrumentation，也就是将应用程序改一改，然后repackage一下。当然这个步骤应该是可以自动化的。

对于前者来说，需要google或者一些设备提供商的支持，或者用户进行root；对于后者来说，谁来进行instrumentation，instrumentation产生的side effect都是需要考虑的问题。

这篇文章采用了第二种方式，而它的方法其实是很直观的：

![mrhide](http://ytliu.info/images/2013-03-01-2.png "Mr. Hide")

![drandroid](http://ytliu.info/images/2013-03-01-3.png "Dr. Android")

* 首先，将apk中的粗粒度权限换成细粒度权限；
* 然后在apk中插入一个library（文中为hidelib），并通过一个自动化工具Dr. Android找出应用程序中所有对sensitive API的调用，将其换成hidelib中对应的API调用；
* hidelib中的API会首先对应用的权限进行一个检查，然后和Mr. Hide进行交互，Mr. Hide是一个运行在另一个进程中的Service，它可以根据hidelib传送过来的信息进行相应的调用，并将结果返回。

当然，这里面会牵扯到很多细节问题，比如应用程序怎么样和Mr. Hide进行binding，如何保护Mr. Hide不被恶意程序利用，以及如何精确而又高效地对原始apk进行instrumentation，还有很多Android中签名机制的问题等等，这里就不一一阐述了。

------

值得一提的是，这篇paper中提到的Dr. Android是一个叫做[redexer](http://www.cs.umd.edu/projects/PL/redexer/about.html)的开源项目，也就是作者开发的。redexer是一个用ruby和OCaml写的Dalvik Binary Rewriter，效率很高，功能也挺强大的。我在之后的博客中会对其进行一个介绍。