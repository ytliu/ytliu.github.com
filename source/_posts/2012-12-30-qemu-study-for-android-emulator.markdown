---
layout: post
title: "Qemu study for Android emulator"
date: 2012-12-30 19:10
comments: true
categories: Android Qemu
---

这两天看了下android emulator的源代码，位置在`android-src/external/qemu`里面，

编译和启动的方式很简单；
	$ ./android-configure.sh
	$ make
	$ export ANDROID_SDK_ROOT=/path/to/androdi-sdk
	$ emulator-arm @4.2

你可以对源码进行修改，然后重新编译、使用，而这里主要要讲的是qemu的运行原理，资料来源是[Qemu detailed study](http://lists.gnu.org/archive/html/qemu-devel/2011-04/pdfhC5rVdz7U8.pdf)。要说明的一点是，android emulator原理基本上是和qemu一样的，只是加了一些android specific的东西在里面。

<!-- more -->

首先是qemu整体流程：

![qemu process](http://ytliu.github.com/images/2012-12-30-1.png "qemu process")

首先将guest code（这里即为arm code）被TCG（Tiny Code Generator)转换成一个中间表达，然后再转换成host code（这里即为x86 code），具体来说分为两步：

* 一个TB（Translation Block）被翻译成TCG ops
* TCG ops被翻译成host code

我们先来看下qemu的code base：

![qemu code base](http://ytliu.github.com/images/2012-12-30-2.png "qemu code base")

* `vl.c/vl-android.c`: 这个是整个qemu的入口函数，主要是初始化qemu环境，然后进入`main_loop`；
* `target-xyz/translate.c`: 将guest code翻译成TCG ops；
* `tcg/*/tcg-target.c`: 将TCG ops翻译成host code；
* `tcg/tcg.c`: TCG的主函数；
* `cpu-exec.c`: 寻找下一个TB（如果没找到则调用tcg.c生成TB），然后执行。

在qemu中也很好地利用了locality，即没产生一段code（TCG ops或host code），就将其存在一个code cache中，然后用LRU进行替换。

#### 运行流程（code perspective）

主要分为两部分： *代码生成*和*代码运行*

##### 代码生成

这是主要部分，流程是这样的：

![qemu process from code perspective](http://ytliu.github.com/images/2012-12-30-3.png "qemu process from code perspective")

其中函数`cpu_exec()`相当于主要的执行循环函数，它将TB第一次初始化，在两个嵌套无限for循环中通过`tb_find_fast()`来获得host code TB，然后通过`tcg_qemu_tb_exec()`来执行相应代码。

`tb_find_fast`会首先查看code cache中是否有TB存在了，有则直接执行`tcg_qemu_tb_exec()`，否则通过`tb_find_slow()`来查找或者生成TB，后者通过一系列调用，最后到达`disas_insn()`，该函数执行了实际的guest code到TCG ops的翻译，并将其加入TCG ops的code buffer，最后调用`tcg_gen_code()`来生成host code。

##### 代码运行

代码运行就是通过`tcg_qem_tb_exec()`来实现的，可以看到，其实这是一个宏，定义在`tcg/tcg.h`里面：

{% codeblock lang:c %}
#define tcg_qemu_tb_exec(tb_ptr) \
	((long REGPARM __attribute__ ((longcall)) (*)(void *))code_gen_prologue)(tb_ptr)
{% endcodeblock %}

这是一个感觉非常复杂的调用，我们知道， `((long REGPARM (*)(void *))` 是一个指向函数的指针，`void *`是它的参数，返回值为`long`；而在这里 `REGPARAM(*)`是一个GCC选项，表示函数是通过寄存器传参而不是通过栈传参的。

而一个数组的名字表示的是指向这个数组的基地址，于是，`(function_pointer) array_name`则会将这个基地址cast成一个函数地址。

另外，一个函数可以通过`(*pointer_to_func)(args)`被调用，所以`((long REGPARM (*)(void *))code_gen_prologue)(tc_ptr)`进行了一次函数调用，似乎在这里少了一个`*`号，不过其实只要测试下可以发现 `(*pointer_to_func)(args)`和`(pointer_to_func)(args)`是一样的。

所以上面`tcg_qemu_tb_exec(tb_ptr)`翻译的宏可以表示为一个数组`code_gen_prologue`被cast成一个函数指针，参数为`tc_ptr`，返回值为`long`（指向下一个TB），并且被调用。其实，被`code_gen_prologue`指向的函数就是`Function Prologue`，将控制流转到`tc_ptr`指向的host code开头部分。



