---
layout: post
title: "Notes about Ruby"
date: 2012-04-01 19:58
comments: true
categories: Ruby
---

已经决定要学ruby和rails了，就从最实际的开始吧~

CSE要求TA每次recitation完了以后给小朋友们发邮件告知成绩，就尝试用ruby写了个脚本，其实这件事最简单的应该是直接用shell脚本来做的，既然在学ruby嘛~就逮着机会用咯~而且还和小Z学到很多东西呢。

其实要做的事情很简单，就把学生的成绩按照*"学号 分数"*一行行写在文件里，分数是按照*"1.5/0.5/0.5"*的顺序排放的，最后要通过一行行读文件，然后用gmail发给相应学生的学号邮箱里面，标题为REC SCORE，内容是

    2.5 （summary/qa/discussion = 1.5/0.5/0.5）
    
其它也没什么好多说的，先贴代码吧：

<!-- more -->

{% codeblock lang:ruby %}
require 'gmail'

def sendgmail(filename, username, password)
  p "in sendgmail"
  Gmail.new(username, password) do |gmail|
    lines = IO.readlines(filename)
    lines.each do |line|
      to_mail = "#{line.split[0]}@fudan.edu.cn"
      score = line.split[1]
      tscore = format("%.2f", score.split('/').map(&:to_f).inject(0) { |a, e| a + e })
      content = "#{tscore} (summary/qa/discussion = #{score})"
      p "#{to_mail} #{content}"
      
      gmail.deliver do
	to to_mail
	subject 'REC2 Score'
	text_part do
	  body content
	end
      end
    end
  end
end

username = 'mctrain016@gmail.com'
password = ARGV[0]
#sendgmail(username, password)

sendgmail('rec2_score', username, password)
{% endcodeblock %}

这里主要是用了一个gem库*ruby-gmail*，要先

    gem install ruby-gmail
    gem install mime
    
里面封装好了用ruby发gmail的方法，这个没有什么好说的，唯一想说的是小Z教我的map和reduce的用法：

{% codeblock lang:ruby %}
      tscore = format("%.2f", score.split('/').map(&:to_f).inject(0) { |a, e| a + e })
{% endcodeblock %}

这里*.map*是对Array里面的所有元素都做一遍to_f操作变成一个新的Array，*.inject(0)*则是在为之后的reduce赋一个初始值0，之后每一次reduce都进行一次*a + e*的操作，将a作为输出，然后再将*a, e*作为输入。举一个例子：

![example-1](http://ytliu.info/images/2012-04-01-1.png "reduce example")

希望之后还有什么需要的都可以用ruby来尝试下，然后再慢慢地学习ruby on rails，不会忘记我的梦想滴~
