---
layout: post
title: "Redexer and Ocaml"
date: 2013-03-10 09:53
comments: true
categories: Android OCaml
---

在[上一篇博客](http://ytliu.github.com/blog/2013/03/01/dr-android-and-mr-hide-xidashiyin-fa-de-gu-shi/)是一个用ruby和OCaml写的Dalvik binary rewriter，它提供了unparse, htmlunparse, id, combine, info, classes, api, opstat, cg, cfg, dom, pdom, dump_method等功能，还可以自己扩展相应的功能，具体的可以参看它的[github主页](https://github.com/plum-umd/redexer)。

redexer主要原理就是将`.dex`文件按照[Dalvik Executable Format](http://source.android.com/tech/dalvik/dex-format.html)映射到内存中的数据结构，然后根据这些数据结果进行分析，可以达到比较高的性能。

这里主要想要说说[OCaml](http://en.wikipedia.org/wiki/OCaml)这个语言。

<!-- more -->

OCaml的全称是Object Caml，是一个有OO扩展的函数式语言（Functional Language），看了看它的语法，和我们程序语言理论里面的那个simPL非常的像，也和当时学FP的时候讲的Haskell很像。

查了查functional language和imperative language有什么不同，在wiki上似乎有一个最大的区别在于：

	It emphasizes the application of functions, in contrast to the imperative programming style, 
	which emphasizes changes in state. The most significant differences stem from the fact that 
	functional programming avoids side effects, which are used in imperative programming to implement 
	state and I/O. 

然后在[stack overflow](http://stackoverflow.com/questions/2078978/functional-programming-vs-object-oriented-programming)里面有一个蛮经典的回答”When do you choose functional programming over object oriented ?“

![stackoverflowans](http://ytliu.github.com/images/2013-03-10-1.png "answer")

另外，在OCaml里面有很多FP的特性，比如 static type system, type inference, parametric polymorphism, tail recursion, pattern matching, first class lexical closures, functors (parametric modules), exception handling, and incremental generational automatic garbage collection等等，这些会在之后对OCaml的学习中慢慢弄清楚来吧~