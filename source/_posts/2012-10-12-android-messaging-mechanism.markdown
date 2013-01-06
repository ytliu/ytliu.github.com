---
layout: post
title: "android messaging mechanism"
date: 2012-10-12 21:43
comments: true
categories: Android
---

这两天做remote binder遇到一个bug，具体是什么就不细说了，总之是和android的message机制相关，在Handler.java里面通过mQueue.enqueueMessage()成功后却没有办法从Looper.java里面的mQueue.next()中读出该条message。搞了半天，打印了一堆log，最后发现竟然是因为Looper的线程被我自己block住了。这个主要是由于自己对其理解的错误造成的，之前我一直以为对于每个Activity（或Service）来说，除了主线程之外，都会有一个专用的Looper线程进行消息队列的轮询，今天和乃正讨论了下，其实不是这样的。对于一个进程来说，如果你没有需求说需要有某个线程做某些特定的事（比如socket监听），那么你的主线程就会进入Looper循环进行消息队列的轮询，否则你就需要自己再新建一个线程，或者重新开一个looper，或者进行socket监听...

此外还有一点，looper是thread local的，对于这个的理解，应该是这样的：在每个线程新建之后，都会有一个原来线程的mQueue的引用，而如果你要自己做一些特定的事，比如重新开一个新的Looper进行其它消息的轮询，那么没有问题，你新建一个looper，但是要注意，你的mQueue是唯一的，也就是说，你之前的对原来线程的mQueue的引用就没了，也就是说，对于一个线程来说只可能有唯一的mQueue。

之后就是message的流程。有两种可能的情况：

* sendMessage(): 这是最常见的形式，这个msg就是一个普通的message，它会依次调用sendMessageDelayed(), sendMessageAtTime(), queue.enqueueMessage()，最后将消息插到队列中去。
* post(): 我这次遇到的就是这种形式，在ServiceDispatcher中用到的就是它，它传递的message是一个Runnable对象，叫RunConnection，在post里面会逐步调用sendMessageDelayed(), sendMessageAtTime(), queue.enqueueMessage()，并且会将msg.callback设成RunConnection对象。


之后在Looper中会进入while循环，每次取出一个msg(*queue.next()*)，然后调用dispatchMessage()，在里面会有下面这段代码：

{% codeblock %}
if (msg.callback != null) {
    msg.callback.run();
}
else {
    ...
}
{% endcodeblock %}

也就是说如果是post传进来的RunConnection对象的msg的话将会直接运行它的run函数，否则会直接调用handleMessage()进行处理。

------

android的message机制其实就是一种异步消息处理机制，进程通过Handler的sendMessage或post将msg进行enqueueMessage，然后通过该线程自己的Looper进行消息队列轮询并根据相应的情况进行handle。

