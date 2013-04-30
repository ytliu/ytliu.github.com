---
layout: post
title: "Traceview in Android"
date: 2013-02-23 15:54
comments: true
categories: Android
---

这两天尝试了下用Android自带的[Traceview](http://developer.android.com/tools/debugging/debugging-tracing.html)进行profile，感觉蛮有意思的，而且发现它的文档上对数据格式的说明和我实现时候碰到的情况有出入，于是在google code上发了封[帖](https://code.google.com/p/android/issues/detail?id=51286)（[前一封](https://code.google.com/p/android/issues/detail?id=50930#makechanges)被华丽丽地无视掉了），但是到最后还是不明白到底是什么问题，维护文档的人好不热情啊...

反正这里还是按照我的实验情况来解释吧:-)

首先得解释下什么是Traceview。最简单而且最普遍的Traceview的流程是这样子的：

* 用户首先要对源代码进行一些instrumentation，如下所示：

{% codeblock lang:java %}
	// start tracing to "/sdcard/calc.trace"
    Debug.startMethodTracing("calc");
    // ...
    // stop tracing
    Debug.stopMethodTracing();
{% endcodeblock %}

* 之后当运行到`Debug.startMethodTracing`时，系统就会进入Trace模式，在`/sdcard`目录下创建一个`calc.trace`文件，并动态地将之后的所有函数调用都trace进这个文件中（当然也包括framework中的API调用）；
* 然后就可以利用Android提供的Traceview工具或者dmtracedump工具对这个文件进行分析了。

其中通过Traceview可以获得很多非常有用的信息，包括Timeline Panel和Profile Panel，具体的可以参见Traceview的[document](http://developer.android.com/tools/debugging/debugging-tracing.html)，这里就不阐述了。而通过dmtracedump工具可以得到整个运行过程中的call-stack diagram。

不过目前的dmtracedump的call-graph功能还没有实现，于是我从网上找到一个用python脚本写的[dmtracedump替代](http://blog.csdn.net/zjujoe/article/details/6080738)，一开始不能用，于是debug了一通，从而发现了前面说的和Traceview官方文档的出入。下面主要就是解释一下有何出入：

首先要介绍下官方文档中由Debug产生的Traceview的文件格式是怎样的：

cal.trace主要分为两个部分 —— key和data。

key部分的格式是这样的：

{% codeblock %}
*version
1
clock=global
*threads
1 main
6 JDWP Handler
5 Async GC
4 Reference Handler
3 Finalizer
2 Signal Handler
*methods
0x080f23f8 java/io/PrintStream write ([BII)V
0x080f25d4 java/io/PrintStream print (Ljava/lang/String;)V
0x080f27f4 java/io/PrintStream println (Ljava/lang/String;)V
0x080da620 java/lang/RuntimeException   <init>    ()V
[...]
0x080f630c android/os/Debug startMethodTracing ()V
0x080f6350 android/os/Debug startMethodTracing (Ljava/lang/String;Ljava/lang/String;I)V
*end
{% endcodeblock %}

它分为了3个sections，每一个section由`*`开头：

* version section;
* threads section：主要记录了thread ID和thread Name，其中thread ID会在data部分中被引用；
* methods section：主要记录了method ID和method signature，其中method ID用于被data部分的record引用。

data部分的格式是这样的：

{% codeblock %}
* File format:
* header
* record 0
* record 1
* ...
*
* Header format:
* u4 magic 0x574f4c53 ('SLOW')
* u2 version
* u2 offset to data
* u8 start date/time in usec
*
* Record format:
* u1 thread ID
* u4 method ID | method action
* u4 time delta since start, in usec
{% endcodeblock %}

它由heade和一系列的records组成，其中在header里面有一个域叫offset to data，由此可以找到第一个record的地址，而每一个record由三部分组成：

* thread ID：就是在key里面的那个thread ID；
* method ID | method action：就是在key里面的那个method ID，method action有四种情况: {0->method entry; 1->method exit; 2->method exited by exception handling; 3->reversed};
* time：就是这个action发生的时间，正常情况下时间都是按照升序排序的。

有了这个trace文件以及事先协议好的格式，就可以进行分析了。即对每一个record进行遍历，得到每个时间点的函数调用，并将它们串起来，就可以得到很多信息。

但是在我的实验中，发现data部分中每个record不是像文档中描述的那样是由9个bytes组成的，而是14个bytes组成：

{% codeblock %}
* Record format:
* u2 thread ID
* u4 method ID | method action
* u8 time delta since start, in usec
{% endcodeblock %}

我的猜想是和产生trace文件时机器的配置有关，我的系统是Debian 6.0，x86_64，维护文档的人似乎没有回答我，不过根据后面一个回答，好像确实在新的版本中就是14个bytes。

Anyway，不管怎么样，这个trace文件是可以提供非常多动态信息的，但是需要对源代码进行修改，并且可能会产生比较大的运行时overhead（自己猜的，没有验证）。不过按照文档上的说法还有一种方式是采用[DDMS](http://developer.android.com/tools/debugging/ddms.html)，不过我也没有尝试过，就不在这里阐述了。

------

另外，Python脚本我就不帖出来了，我用ruby重新实现了一遍，这是可以在我的机器上work的版本：

{% codeblock lang:ruby %}
# dmtracedump.rb
# turn the traceview data into a png pic, showing methods call relationship

################################################################################
########################  Global Variable  #####################################
################################################################################
@target_thread = 1
@all_threads = {}
@all_methods = {}
@all_records = []
@parent_methods = {}
@child_methods = {}
@method_calls = {}

################################################################################
##############################   Methods   #####################################
################################################################################
def add_one_thread(line)
    fields = line.split("\t")
    @all_threads[fields[0].to_i]=fields
end

def add_one_method(line)
    fields = line.split("\t")
    @all_methods[fields[0].to_i(16)]=fields
end

def add_one_record(one)
    record = one.unpack("SLQ")
    if record[0] == @target_thread
        method_id = (record[1] / 4) * 4
        method_action = record[1] % 4
        @all_records.push([record[0], method_id, method_action, record[2]])
    end
end

def handle_one_call(parent_method_id, method_id)
    # p "#{parent_method_id} -> #{method_id}"

    if !@parent_methods.has_key?(parent_method_id)
        @parent_methods[parent_method_id] = 1
    end

    if  !@child_methods.has_key?(method_id)
        @child_methods[method_id] = 1
    end

    if @method_calls.has_key?(parent_method_id)
        if @method_calls[parent_method_id].has_key?(method_id)
            @method_calls[parent_method_id][method_id] += 1
        else
            @method_calls[parent_method_id][method_id] = 1
        end
    else
        @method_calls[parent_method_id] = {}
        @method_calls[parent_method_id][method_id] = 1
    end
end

def gen_funcname(method_id)
    cla_med = @all_methods[method_id][1].gsub(/[\/\$<>]/, '_')
    sub_med = @all_methods[method_id][2].gsub(/[\/\$<>]/, '_')
    "#{cla_med}_#{sub_med}"
end

def gen_dot_script_file
    ofile = File.open("graph.dot", "w")
    ofile.write("digraph vanzo {\n\n")

    @all_methods.keys.each { |m|
        if @parent_methods.has_key?(m)
            ofile.write("#{gen_funcname(m)}  [shape=rectangle];\n")
        else
            if @child_methods.has_key?(m)
                ofile.write("#{gen_funcname(m)}  [shape=ellipse];\n")
            end
        end
    }

    @method_calls.keys.each { |p|
        @method_calls[p].keys.each { |c|
        	ofile.write("#{gen_funcname(p)} -> #{gen_funcname(c)} [label='#{@method_calls[p][c].to_s}' fontsize='10'];\n")
        }
    }

    ofile.write("\n}\n");
    ofile.close
end

################################################################################
########################## Script starts from here #############################
################################################################################

if ARGV.size < 1
    print "No input file specified."
    exit
end

if !File.exists?(ARGV[0])
    p "input file not exists"
    exit
end

#Now handle the text part
current_section = 0
lines = File.readlines(ARGV[0])
lines.each { |line|
	line.strip!
    if line.start_with?("*")
        if line.start_with?("*version")
            current_section = 1
        elsif line.start_with?("*threads")
        	current_section = 2
        elsif line.start_with?("*methods")
        	current_section = 3
		elsif line.start_with?("*end")
			current_section = 4
			break
		end
    elsif current_section == 2
        add_one_thread(line)
    elsif current_section == 3
        add_one_method(line)
    end
}
    
#Now handle the binary part
file = File.open(ARGV[0], 'rb')
alldata = file.read()
#alldata = File.read(ARGV[0])
pos = alldata.index("SLOW")
offset = alldata[pos+6, 2].unpack("S")[0]
pos += offset #where the record begin
record_num = alldata.length - pos
record_num /= 14
0.upto(record_num-1) { |i|
	add_one_record(alldata[pos+i*14, 14])
}

my_stack = [0]

@all_records.each { |onerecord|
	thread_id = onerecord[0];    
    method_id = onerecord[1];
    action = onerecord[2];
    time = onerecord[3];
    if action == 0
        if my_stack.size > 1
            parent_method_id = my_stack[-1]
            handle_one_call(parent_method_id, method_id)
        end
        my_stack.push(method_id)
    else
        if action == 1
            if my_stack.size > 1
                my_stack.pop()
            end
        end
    end
}
    
gen_dot_script_file
os.system("dot -Tjpg graph.dot -o output.png;rm -f graph.dot");

{% endcodeblock %}

用法：

    $ ruby dmtracedump.rb calc.trace
