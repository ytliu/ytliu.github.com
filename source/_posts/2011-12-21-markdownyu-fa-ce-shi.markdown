---
layout: post
title: "Markdown语法测试"
date: 2011-12-21 15:17
comments: true
categories: 杂
---

<http://ytliu.github.com>

\[mctrain](http://ytliu.github.com "mctrain's blog")    
[mctrain](http://ytliu.github.com "mctrain's blog")    
[mctrain-2][link-mctrain-2]   
[link-mctrain-2]:http://ytliu.github.com "mctrain's blog"   

        ![pic](http://ytliu.github.com/images/search.png "pic")
![pic](http://ytliu.github.com/images/search.png "pic")

        #title1
#title1
        ##title2
##title2
        ###title3
###title3
        ####title4
####title4
        #####title5
#####title5
        ######title6
######title6

        *italic*
*italic*

        **bold**
**bold**

        ***italic&bold***
***italic&bold***

        >highlight
>highlight

        <html> *a* </html>
<html> *a* </html>

* 1
* 2
* 3

16. first
* second
* third
18. fourth

hr-high
- - - - - -
hr-below

>highlight
>>highlight again

{% codeblock %}
Awesome code snippet
{% endcodeblock %}

{% codeblock lang:objc %}
[rectangle setX: 10 y: 10 width: 20 height: 20];
{% endcodeblock %}

{% codeblock Javascript Array Syntax lang:js http://j.mp/pPUUmW MDN Documentation %}
var arr1 = new Array(arrayLength);
var arr2 = new Array(element0, element1, ..., elementN);
{% endcodeblock %}

中文测试
这个测试中有很多很多很多字符
{% blockquote %}
"C"是系统默认的locale，"POSIX"是"C"的别名。所以当我们新安装完一个系统时，默认的locale就是C或POSIX。

在Debian中安装locales的方法如下：

· 通过apt-get install locales命令安装locales包
· 安装完成locales包后，系统会自动进行locale配置，你只要选择所需的locale，可以多选。最后指定一个系统默认的locale。这样系统就会帮你自动生成相应的locale和配置好系统的locale。

· 增加新的locale也很简单，用dpkg-reconfigure locales重新配置locale即可。

· 我们也可手动增加locale，只要把新的locale增加到/etc/locale.gen文件中，再运行locale-gen命令即可生成新的locale。再通过设置上面介绍的LC_*变量就可设置系统的locale了。下是一个locale.gen文件的样例。

{% endblockquote %}
