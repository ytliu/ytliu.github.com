---
layout: post
title: "paper reading: TCP RE and incremental MR"
date: 2011-12-26 20:57
comments: true
categories: Paper
---

今天组会两篇paper，Naruil讲的第一篇听了蛮有收获的，DX讲的第二篇感觉是自己英文太差而且领域不熟最后听的不是很懂，就大致记录下吧。

第一篇是一个做数据中心之间数据传递重复消除的系统，叫End-to-end RE(redundancy elimination)，是发在NSDI'11上的，题目叫"EndRE: An End-System Redundancy Elimination Service for Enterprises"，主要由微软的人做的。简单来说它的motivation就是现在在网络中传递的数据有很大一部分是重复的，比如两个包有大量的重复数据，而这些重复数据占用了大量的带宽，而这篇paper的工作就是如何在减少重复数据的同时也不减少太多的性能。

现在在用的关于这种重复数据消除的技术主要是"Middlebox-based"，它有几个缺点：    
*       关于安全方面的问题，网络数据在middlebox里面是明文，这样才能更好地判断重复性，这样就势必减弱了网络数据的安全级别；      
*       对于一些终端的手机设备，它需要和PC终端传递数据，在这过程中同样有数据的重复，而这种情况就不适宜用Middlebox技术了；        
*       代价比较大，现在用的middlebox都是很强大的服务器，配备巨大的内存来存储内容的cache。

于是作者就提出一种end-to-end的RE技术。通篇听下来最大的收获是对传统的fingerprint算法和作者提出的一种新的SAMPLEBYTE fingerprint算法的理解。

这篇paper里面提到了两种传统的fingerprint算法：
        ModP fingerprint：
                这是一种content-based的fingerprint算法，用一种特殊的hash算法
                （每一个window的hash值 = 上一个window的hash值+上一个window的第一个byte+下一个byte）
                这样可以快速地得出每个window的hash值，之后将该hash值mod一个P，
                如果结果为0，则将该hash值作为一个sample的fingerprint。             
        Fixed fingerprint：
                这是一个position-based的fingerprint算法，即每隔P个byte算一个hash
                （window size大小的byte），之后将其作为一个sample的fingerprint。
从这两个fingerprint算法可以很容易地看出对于ModP，由于它是一个content-based的算法，所以不管是不是有偏移都可以比较完整地得出内容上的重复性，但是它的效率太低了，因为它要算每一个byte的hash；而对于Fixed，它的效率远远大于ModP，但消除重复性的能力也相应地变小很多。

<!-- more -->

于是作者提出提出的一种新的算法，叫SAMPLEBYTE fingerprint：
        提供一个256bit的数组A，遍历要发送的包的每一个字节，比如第一个字节是0x23，那么查找
        数组A的第23个bit看它是否为1，如果为0则表示miss，继续查看后一个字节，如果为1则代表hit，
        即将这个字节以及后window size个大小算一个hash作为fingerprint，然后跳过（p/2）个字节
        （为了防止计算重复的hit）。
也就是说SAMPLEBYTE也是一个content-based的算法，而且跳过了每个byte都要检查的低效率从而达到更合理而又大粒度的sample机制。

这是这篇paper的核心算法，至于最后如何利用得出的fingerprint，则可以通过下张图看出：

![RE Overview](http://ytliu.github.com/images/2011-12-26-1.png "the overview of EndRE")
![Look up in fingerprint hash table](http://ytliu.github.com/images/2011-12-26-2.png "Look up in fingerprint hash table")

在服务器端和客户端都要维护一个同步的cache，同时在服务器端有一个hash table，里面的key即为之前算出来的fingerprint，指向的是cache中的offset，当要传递一个包时，将算出的fingerprint在hash table里面查找，如果hit了，则从该fingerprint的第一个byte开始找出最大匹配的字符长度，从而省去了该重复内容的传递。而在客户端也只需要将接收到的内容按时间顺序写入cache中，保持和服务器的cache的同步性就好了。

另外，这个数据重复性消除的问题还需要考虑的很重要的一点就是：在客户端不该有很复杂的计算，否则直接将包压缩传递岂不更高效？

还有关于fingerprint算法和差抄袭算法的关系，问了下Naruil，他之前写的差抄袭算法也是用了fingerprint，不过用的是一个更健壮性的fingerprint算法，是基于一篇Sigmod'03的paper：[Winnowing](http://dl.acm.org/citation.cfm?id=872770)。有机会可以去瞻仰下，据说效果相当好，反正我们这里的抄袭都是这么被检查出来的~

- - - - - - 

另外一篇的题目是"Incoop: MapReduce for Incremental Computation"，发表在Socc'11上，做的是MapReduce上的Incremental computation。也就是说现在的MR架构对于两个输入，即使是有许多重复的输入也是从头开始重新计算一遍，而作者希望的是对于两个输入，对于不变的输入可以不再重复计算，而仅仅是变化的数据。因为DX是用英文说的，很多内容不是搞得太懂，只知道它用了"content-base chunking"和"reduce combiners"来分别解决stability和granularity的问题。至于具体的细节就搞不来了。

- - - - - -


The End.
