---
layout: post
title: "Dynamic Linker"
date: 2012-12-15 13:03
comments: true
categories: Linux
---

在了解dynamic linker之前，得先对ELF文件有一个初步的了解：

在[jollen](http://www.jollen.org/blog/2006/12/enabling_dynamic_loader_1.html "elf")的博客中有一断对ELF Session的表格:

![elf section](http://ytliu.info/images/2012-12-09-1.png "elf section")

很有参考价值。其中将会涉及到的sections有*.got*, *.plt*, *.interp*。

另外在[Computer Science from the Bottom Up](http://www.bottomupcs.com/ "computer science from bottom up")中有一章对dynamic linker进行了详细的说明。

以下的内容很多来自于该文档和俞甲子的《程序员的自我修养》一书第七章。

<!-- more -->

问题的产生是这样子的：当我们使用一段shared library的时候，它并没有指定说一定要把相应的代码放在哪个内存地址，而是由dynamic linker根据当前内存情况选择一段最合适的内存区域用于放置相应的code和data。那么dynamic linker是怎么做的呢？我们举一个很简单的例子来说明；

首先，我们编写并编译一段动态链接库lib.c, lib.h：

{% codeblock lang:c %}
#include "lib.h";

void test(int i) {
    printf("in dynamic lib, i is %d\n", i);
}
{% endcodeblock %}

{% codeblock lang:c %}
Char *dylib = "Test Dynamic Linker String";
void test(int);
{% endcodeblock %}

然后将其编译为动态连接库libtest.so

    $ gcc -fPIC -shared -o libtest.so lib.c

这里 "-shared"表示产生共享对象，"-fPIC"表示地址无关代码，这在后面会说。

然后我们编写一段程序dytest.c来利用libtest.so:

{% codeblock lang:c %}
#include "lib.h";

extern char *dylib;
int main(void)
{
    printf("Hello, %s\n", dylib);
    test(5);
    return 0;
}
{% endcodeblock %}

然后对其进行编译：

    $ gcc -o dytest dytest.c ./libtest.so
    $ ./dytest
    Hello, Test Dynamic Linker String
    in dynamic lib, i is 5

就这么一段简单的测试代码中，动态链接是怎么完成的呢？它和静态链接有什么不同呢？

在静态链接中，整个程序只有一个可执行文件，在这个可执行文件中，所有变量和函数的地址都已经固定好了（这是由linker在链接时从静态链接的文件中读出来并进行地址重定位），而在动态链接中，这些地址并不会进行地址重定位。那么，链接器怎么知道一个地址是静态符号还是动态符号呢？其实在我们编译dytest的时候也将libtest.so加进去进行编译了，而在libtest.so中保存了完整的符号信息，从而linker可以知道该符号是一个动态符号。

既然动态链接库主要用于共享，那么有一个问题：共享对象在编译时不应该假设自己在进程虚拟地址空间中的位置。一种解决的方法是采用“装载时重定位”，但是这样有一个缺点，因为它要在程序装载时对指令部分进行修改，所以就无法使得指令部分在多个进程中共享，这样就失去了共享库的优势，另一种就是地址无关代码，它的基本想法就是把指令中那些需要修改的部分分离出来，和数据放在一起。

我们把地址引用分为4个部分

* 模块内部函数调用
* 模块内部数据访问
* 模块外部数据访问
* 模块外部函数调用

第一种情况应该是最简单的，因为在模块内部函数与调用者的位置是相对的，可以采取相对地址调用。

第二种情况同样采用相对地址的访问，这里有一个trick，就是如何得到数据地址和当前地址的相对值，俞子甲的书中介绍了一种方法(7.3节)。另外，在处理共享库的全局变量的时候，编译器都把它当作定义在其它模块的全局变量，相当于后面讲的类型三，使用GOT表。

第三种情况就复杂一点了，因为它要等到装载时才能决定。这里就要用到GOT（Global Offset Table）表了，ELF在数据段中建立一个指向相关地址的指针数组。对于数据变量a，在GOT表中有一个4bytes的地址项与之对应，在程序装载时，链接器会找到这个变量的地址，并将该项进行修改。

第四种情况和第三种类似，只是地址为函数地址。

其实还有第五中情况，就是模块间的全局变量，比如下面这个例子：

{% codeblock lang:c %}
extern int global;
int foo()
{
    global = 1;
}
{% endcodeblock %}

如果一个定义在共享模块内部的全局变量，编译器并不知道它是否会被其它模块使用，所以当前编译器在遇到这种全局变量的时候都会把其当做定义在其它模块中的全局变量，即上面的第三种情况，使用GOT表进行访问。

这里还要注意一点的是，在产生地址无关代码的时候参数-fpic和-fPIC的区别，-fpic产生的代码相对较小，而且较快，但是对于一些硬件平台有一些限制。另外，它也可以被用在可执行代码上，这时，就被称为PIE（using -fPIE or -fpie）。

这里需要澄清的一点是，对于一个共享库lib.so来说，它在不同的进程中都有自己独立的副本，而在同个进程不同线程中则是共享的。而对于多进程共享全局变量使用的是“共享数据段”，而多线程访问不同全局变量则被称为“线程私有存储”。

还有，对于数据段的绝对地址引用，可以用到装载时重定位的方法来解决，即对于共享对象来说，如果数据段中有绝对地址引用，如static int *p = &a，编译器和链接器会产生一个重定位表，当动态链接器装载共享对象时若发现有重定位入口，则对其进行重定位。

####延迟绑定（PLT）

在动态链接的程序开始运行的时候都会通过动态链接器寻找并装载共享对象，但是有些函数其实可能并不会被调用。为了增加性能，会采用一种被称为PLT（Procedure Linkage Table）的方式，它的基本思想就是当函数第一次被用到时才进行绑定。它采用了一些很精巧的指令来完成:

每个外部函数都有一个在PLT对应的项（bar@plt)
	
	bar@plt:
	jmp *(bar@GOT)
	push n
	push moduleID
	jmp _dl_runtime_resolve

在这里第一条指令跳转到bar在GOT中的项，该项中的初始地址即为这里第二条指令（push n）的地址，相当于没有效果，然后将bar的信息和其所在模块的信息压入栈，最后调用_dl_runtim_resolve将bar真正对的地址填入到bar@GOT中，当下次真正调用bar的时候就会跳转到真正的函数地址，并返回到调用者，而不会回到*push n*的地址了。

ELF将GOT分成了两个表“.got"和".got.plt", ".got"用来保存全局变量引用地址，".got.plt"用来保存函数引用的地址，在".got.plt"中前三项是有特殊意义的：

* 第一项保存".dynamic"段的地址，这个段描述了本模块动态链接相关的信息；
* 第二项保存的是本模块的ID；
* 第三项保存的是_dl_runtime_resolve()的地址。

而".got.plt"的其余项分别对应每个外部函数的引用。

####动态链接相关结构

在动态链接的情况下，在装载完可执行文件之后，操作系统会将控制权转交给动态链接器，动态链接器的路径在".interp"下指定。

和动态链接相关的段比如说：

#####.dynamic
动态链接器中最重要的结构就是".dynamic"段，它就像动态链接下ELF文件的”文件头“。

#####.dynsym
".dynsym"段是为了表示模块间动态链接相关符号的导入导出关系的，当然，和".symtab"段类似，它也需要一些辅助的表，如".dynstr"动态符号字符串表，".hash"符号哈希表。

#####动态链接重定位表
"rel.dyn"和"rel.plt"相当于静态链接中的"rel.data"和"rel.text"。"rel.dyn"是对数据引用的修正，它所修正的位置即".got"以及数据段，而".rel.plt"则是对函数引用的修正，即".got.plt"段。

####动态链接的步骤和实现
主要分为三步：

#####动态链接器自举
这里有两个条件：

* 本身不可以依赖于其它任何共享对象
* 本身所需要的全局和静态变量的重定位工作由其本身完成——即“自举”

#####装载共享变量
从全局符号表中开始寻找其所依赖的共享变量，即".dynamic"段中一个DT_NEEDED类型，将里面提到的所有共享对象的名字放入一个装载集合中，然后从集合中一个个读取共享变量名字，找到相对应的文件，读取里面的ELF文件头和".dynamic"段，然后将相应的代码段和数据段映射到其地址空间中，并递归地做这件事。所以当所有共享变量都被装载进来后，全局符号表里面将包含所有动态链接所需要的符号。

####重定向和初始化
装载完成后，链接器开始重新遍历可执行文件和每个共享对象的重定向表，将其GOT/PLT中需要重定向的进行修正，然后就将控制权转交给程序的入口了。由此，动态链接也就完成了。
