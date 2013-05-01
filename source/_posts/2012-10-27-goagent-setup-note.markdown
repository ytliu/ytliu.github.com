---
layout: post
title: "GoAgent setup note"
date: 2012-10-27 12:21
comments: true
categories: 
---

这次去武汉开会的时候听翔哥提到GoAgent很好用，就尝试着去搭了下环境。

[原文地址](http://irising.me/2012/02/13376/ GoAgent)

说实话这个环境搭起来真的很方便，主要分为四步：

* 申请Google App Engine的请用，这个就不细说了，具体参看原文，唯一一个要注意的是在里面有一个Storage Schema的选项要选择High Replication (我之前注册的应用没有这一项，要转变成这个还挺麻烦的)。

* 安装GoAgent的客户端：代码的地址在[这里](http://code.google.com/p/goagent/)，然后要修改local目录下的proxy.ini文件，之后要安装一个证书（不过据说不安装也行）。最后上传到GAE上：

    $ cd goagent/server

    $ python uploader.zip update ./

会在后面提示输入appid和GAE账户信息。

* 设置代理：新建一个网络位置，例如：命名为代理，并将Web代理、安全Web代理两项勾选上，代理服务器地址均为，127.0.0.1,端口为8087。

* GoAgent使用：首先先在终端运行如下命令：

    $ cd goagent/local

    $ python proxy.python

然后再在苹果菜单下切换location成代理。

这样就可以通过GoAgent上网了。

<!-- more -->

####GoAgentX
由于这个需要每次都在终端下输入命令，切换网络位置，最关键的是如果换了一个代理服务器的地址的话还要手动更新，所以在P哥的推荐下我选择了使用GoAgentX，这个是GoAgent的图形界面版，而且可以自动更新，十分方便。

######安装
下载地址在[这里](https://github.com/downloads/ohdarling/GoAgentX/GoAgentX-v1.3.6.dmg GoAgentX)，直接点击安装就可以了。然后会在桌面上方的任务栏里有个戴帽子的小人图像出现，点击显示主窗口，在服务配置里面修改端口和appid，在代理配置里面勾选“不修改系统代理配置”，之后启动，并修改网络位置就可以了。

------

我感觉苹果电脑配置这些东西都非常的方便，而且程序运行的也很稳定，相当赞啊~
