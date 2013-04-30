---
layout: post
title: "Cross Reference Graph of Android Component"
date: 2013-03-24 13:35
comments: true
categories: Android
---

由于项目的需要，这周写了一个ruby project——[apk-static-xref](https://github.com/ytliu/apk-static-xref)。

虽然现在Android静态分析的项目很多，但是我都没有找到一个看上去很简单的功能：给定一个component，找出它所有调用到的函数，然后画出call graph。

现在的大部分call graph都只能限制在一个class文件里，这里所说的cross reference graph (XRG)就是指函数调用是跨class文件的，需要cross reference直到调用的是Android本身的API或者Java API。

这其实是一个很简单的功能，但我不知道为什么一直找不到工具可以满足我的需求，直到找到了spark的一个[slide](http://appsrv.cse.cuhk.edu.hk/~mzheng/DroidTrace.pdf)，里面提到一个cross reference graph。其实就是一个对smali文件的DFS算法。

于是我把[smali-cfg]()和[redexer]()结合了一下，写了这个ruby project，具体做法就是：

* 用apktool unpack apk：

	$ java -jar apktool.jar -d source.apk target.dir

* 用nokogiri parse AndroidManefest.xml 找出所有的service和activity：

	Nokogiri::XML(f)

* 用DFS遍历所有的smali文件，得出整个call graph的结点和边：

{% codeblock lang:ruby %}
def run_assign(apk, cls, mtd, pty, flag, ori_cls)
  match = 0
  caller = nil
  callee = nil
  vclass = nil
  filename = apk.smali + cls + ".smali"
  if File.file?(filename)
    fh = File.open(filename, "r")
  else
    return
  end
  lines = fh.readlines
  lines.each { |line|
    line = line[0...-1]
    if line.start_with?(".class")
      vclass = line.split(' ')[-1][1...-1]
    elsif line.start_with?(".method")
      caller = Invoker.new(vclass, line)
      if (caller.mtd.eql?(mtd)) and (caller.pty.eql?(pty))
        match = 1
      end
    elsif line.start_with?(".end method")
      match = 0
    elsif line.include?("invoke-")
      callee = Invoked.new(line)
      if (match == 1) or (flag == 1)
        if $caller_methods.include?(callee.str)
          if !$caller_methods.include?(caller.str)
            addNodes(caller.str, callee.str) if $graph_need
          end
        else
          addNodes(caller.str, callee.str) if $graph_need
          $caller_methods << caller.str
          run_assign(apk, callee.cls, callee.mtd, callee.pty, 0, ori_cls)
        end
      end
    end
  }
end
{% endcodeblock %}

* 用ruby-graphviz画出call graph

	$graph.output(:png => "#{to}.png")

其实整个过程非常的简单，不过可能自己代码写得比较烂，会不太好理解。另外，现在是先用apktool把apk翻译成smali之后再对smali进行操作，效率可能会比较慢，接下来准备把实现改成直接对dex文件进行分析，打算基于Android自带的dexdump或者开源的baksmali，顺便把dex的格式搞得更加清楚一点。

------

由于下周一门课的原因，这周又把OSDI 2012的[AppInsight](http://research.microsoft.com/pubs/173922/appinsight.pdf)的论文仔细看了一遍，觉得确实是做得很好很有用的一个系统。我希望自己能把它实现在Android上。其实整个原理也不是非常的难，我想最难的部分在于Windows本来就有一个Detour可以直接利用来做instrumentation和interception，非常的方便，但是Java本身好像没有一个相当的库，需要自己实现一遍。

不知道这个计划需要多久才能完成，我可能要先找个和Detour功能类似的库改一改，希望这个作为自己第一个比较正式的开源项目不会中途破产吧。



