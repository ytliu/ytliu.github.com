---
layout: post
title: "get root in my android"
date: 2012-12-08 16:28
comments: true
categories: Android
---

首先申明，这其实并不是一篇android root教程，因为我并没有用什么exploit的方法，也没有用CWM等这些第三方ROM，而是该了ASOP，然后把它刷到机子上，然后对其进行所谓的root，装了一个dSploit和busybox。

具体的做法是这样的：

* 更新到ASOP的最新版 - 4.2.4
* lunch full-maguro, make
* fastboot to smartphone
* 把手机连到电脑上，通过adb shell上去，进入su模式
* 通过[这里](http://www.cypherpunk.at/2011/10/08/manual-rooting-android-on-linux-2/ "manual rooting")的方法进行root

即：

* mount -o remount,rw /dev/block/.../system /system
* mv path/to/modified/su /system/xbin/su
* mount -o remount,ro /dev/block/.../system /system

但是发现失败了，我怀疑是su这个文件不兼容，于是，我想了一个更贱的方法：

* 把ASOP代码中的/system/extras/su/su.c该掉，把检查的部分全部去掉
* 然后刷机

这样就直接root了！不过这只能用于我的测试机啦，正常的手机千万不敢那么做，太危险了！

------

另外，我用了两款软件，[dSploit](https://github.com/evilsocket/dsploit "dsploit")和[Mercury](github.com/mwrlabs/mercury "mercury")，而且都有源码，准备之后两周研究下，学习下android编程和如何进行penetration


