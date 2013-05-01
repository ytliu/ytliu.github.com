---
layout: post
title: "getSystemService() in android"
date: 2012-11-11 20:18
comments: true
categories: Android
---

在之前研究了那么久的bindService()这个API，一直没搞清楚一个问题：

为什么我看到的基本上都是和AMS相关的代码，而之前所学到说如果application要和service打交道都是需要通过ServiceManager获得某个service的binder才可以。那么AMS和ServiceManager到底是什么关系呢？如果AMS是通过ServiceManager获得的service binder，那么相关的代码又是在哪里呢？

这个问题困扰了我很久，直到我看到一个[博客](http://blog.csdn.net/windskier/article/details/6625883 "杜文涛的专栏")，我才发现其实我把一个很重要的概念混起来了：

Android中主要通过2种方法来获得service binder：

* 通过一些API如bindService，来获得application service的binder。因为app service都是直接和AMS注册的，AMS运行在system\_server进程；
* 通过ServiceManager.getService(String Descriptor)来获得Service Manager管理的service的binder，ServiceManager运行在servicemanager进程。

也就是说，尼玛bindService只是用来bind application level的service！！！

<!-- more -->

而我们更在意的system service应该是由ServiceManager.getService()来获得的！！！

发现这个问题之后我整个人都有些凌乱了。。。不过静下来想想其实自己从bindService这里入手也学到很多东西，也就不纠结了！

那么现在的问题是，应用程序到底是如何获得system service的binder的？

在这个问题的探索过程初期，我又一次惊奇地发现android.os.ServiceManager竟然是@hide的！！！！

尼玛hide的啊！！！也就是如果不是和framework一起编译的话是找不到的啊！我在网上看到一个很无语的[解决方案](http://www.oschina.net/question/54100_32232?sort=time "隐藏类的使用")，但是，我还是不知道应用程序到底是怎么调用system service的啊？总不可能每个人都懂得什么变态的“隐藏类的使用”吧？

后来不知道是怎么回事，我突然发现在android里面竟然有一个API叫getSystemService()！我问了下乃正，他竟然和我说“你们干嘛不问我，这不是很显然的吗？应用里面就是调用这个获得system service的啊”！尼玛伤不起啊，一个没写过android应用程序的还要研究android framework的人伤不起啊！！

------

###getSystemService()

好了，还是言归正传吧，到底getSystemService()是如何得到system service的呢？其实这个也没有想象的那么好理解。而且我查了下网上的资料，大部分还是讲ServiceManager.getService()是如何得到system service的，而基本上都没有涉及到getSystemService()是怎么和ServiceManager.getService()搭上关系的。

于是经过一番研究，我终于搞清楚了这里面所涉及的种种“复杂而又充满基情”的关系，请听我娓娓道来：

------

在一个月黑风高的夜晚，一个郁郁不得志的少年Activity调用了一个具有扭转乾坤概率的大API---getSystemService()。

算了，没有写小说的潜质。。。还是老实点正常写着吧。。。早点写完早点洗洗睡吧。。。

在调用Activity.getSystemService()之后，就进入ContextImpl.getSystemService()。

至于是怎么进来的，其实也很简单，Activity继承ContextThemeWrapper，再继承ContextWrapper。里面会调用mBase.getSystemService()。这个mBase是一个ContextImpl实例变量，于是就调到了。

于是在ContextImpl.getSystemService()是这样的：

{% codeblock lang:java %}
@Override
public Object getSystemService(String name) {
    ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
    return fetcher == null ? null : fetcher.getService(this);
}
{% endcodeblock %}

话说这个SYSTEM_SERVICE_MAP是怎么来的呢？这就要扯得远点了：

在每个Activity启动的时候都要运行一大段的static的代码（在android.app.ContextImpl.java）里面：

{% codeblock lang:java %}
static {
    registerService(ACCESSIBILITY_SERVICE, new ServiceFetcher() {
	    public Object getService(ContextImpl ctx) {
	    return AccessibilityManager.getInstance(ctx);
	    }});

    registerService(ACCOUNT_SERVICE, new ServiceFetcher() {
	    public Object createService(ContextImpl ctx) {
	    IBinder b = ServiceManager.getService(ACCOUNT_SERVICE);
	    IAccountManager service = IAccountManager.Stub.asInterface(b);
	    return new AccountManager(ctx, service);
	    }});

    registerService(ACTIVITY_SERVICE, new ServiceFetcher() {
	    public Object createService(ContextImpl ctx) {
	    return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
	    }});

    ...

{% endcodeblock %}

它会在自己的SYSTEM\_SERVICE\_MAP为每一个系统服务注册一个ServiceFetcher的类，在这个类中，大部分的服务为重写一个函数叫*createService(ContextImpl)*，这个方法之后会用到，现在只需要知道我们从SYSTEM_SERVICE_MAP获得了某个服务的ServiceFetcher对象fetcher，并通过fetcher.getService(this)获得了该服务的对象。

fetcher.getService(ContextImpl):

{% codeblock lang:java %}
public Object getService(ContextImpl ctx) {
    ArrayList<Object> cache = ctx.mServiceCache;
    Object service;
    synchronized (cache) {
	if (cache.size() == 0) {
	    // Initialize the cache vector on first access.
	    // At this point sNextPerContextServiceCacheIndex
	    // is the number of potential services that are
	    // cached per-Context.
	    for (int i = 0; i < sNextPerContextServiceCacheIndex; i++) {
		cache.add(null);
	    }
	} else {
	    service = cache.get(mContextCacheIndex);
	    if (service != null) {
		return service;
	    }
	}
	service = createService(ctx);
	cache.set(mContextCacheIndex, service);
	return service;
    }
}
{% endcodeblock %}

也就是说每个Activity都有一个mServiceCache，它cache了所有用到的service的ServiceFetcher类。如果hit了，那么直接返回该对象，否则会调用这个fetcher的createService()方法，这也就是我们刚才提到的每一个ServiceFetcher在注册的时候会重写的那个方法。

可以看到，大部分重写的方式都是类似于这样的：
{% codeblock lang:java %}
public Object createService(ContextImpl ctx) {
    IBinder b = ServiceManager.getService(XXX_SERVICE);
    IXXXManager service = IXXXManager.Stub.asInterface(b);
    return new XXXManager(ctx, service);
}});
{% endcodeblock %}
于是乎，getSystemService()就在这里和ServiceManager.getService()无缝地结合在了一起！

------

对于ServiceManager.getService()，在网上有很多关于它的说明和讨论，这里推荐其中两篇：

在[老罗](http://blog.csdn.net/Luoshengyang "老罗的android之旅")的博客中两篇，

第一篇是[这篇文章](http://blog.csdn.net/luoshengyang/article/details/6642463 "java API")，主要介绍了怎么从java里面的ServiceManager.getService()开始的流程；

还有一篇是[这篇文章](http://blog.csdn.net/luoshengyang/article/details/6633311 "getService")，深入介绍了在C++和driver层是如何调用getService的
