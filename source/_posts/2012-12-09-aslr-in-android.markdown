---
layout: post
title: "ASLR in android"
date: 2012-12-09 11:07
comments: true
categories: Android
---

这两篇看了两篇文章分析[aslr in androi4.0](https://blog.duosecurity.com/2012/02/a-look-at-aslr-in-android-ice-cream-sandwich-4-0/ "asli in 4.0")和[android 4.1](https://blog.duosecurity.com/2012/07/exploit-mitigations-in-android-jelly-bean-4-1/ "aslr in 4.1")，觉得蛮有趣的，这里简单介绍下。

### ASLR with Liux Kernel

首先介绍下linux中的ASLR，ASLR可以对任意内存进行随机化：

<!-- more -->

* stack：The userspace stack mapping set up by the kernel during exec(2) should be sufficiently randomized. Stack randomization is performed by the randomize\_stack\_top() function.
* Heap：The heap location returned by the brk(2) system call when a program is first exec’ed should be randomized. Heap randomization is performed by the arch\_randomize\_brk() function.
* Libs and mmap：After NX was introduced, static library mapping led to the popularity of ret-to-libc and more generic ret-to-lib attacks. The location of libraries and other mmap’ed regions should be randomized.
* Exec：Even if you’re randomized the mapping of all the shared libaries that an executable uses, you still need to randomize the location of the executable itself when it is mapped into the address space. Otherwise, the executable mapping can be used as a source for ROP gadgets.
* Linker：On most Linux systems, the ld.so dynamic linker provided by glibc can self-relocate itself, so its mapping is randomized. However, as we’ll see, this isn’t the case for all linkers.
* VDSO(Virtual Dynamically-linked Shared Object)：an executable mapping of a virtual shared library provided by the kernel for syscall transitions. However, most Android devices run on the ARM architecture, which doesn’t use a VDSO.

### ASLR in Android 2.x

在Android 2.x开始，唯一对ASLR支持的是stack（这可以通过多次查看/proc/pid/maps来发现），其通过load\_elf\_binary()函数调用randomize\_stack\_top()来实现：

{% codeblock lang:c %}
#ifndef STACK\_RND\_MASK
#define STACK\_RND\_MASK (0x7ff >> (PAGE\_SHIFT - 12)) /* 8MB of VA */
#endif
static unsigned long randomize\_stack\_top(unsigned long stack\_top)
{
    unsigned int random_variable = 0;
    if ((current->flags & PF_RANDOMIZE) &&
	            !(current->personality & ADDR_NO_RANDOMIZE)) {
            random_variable = get_random_int() & STACK_RND_MASK;
            random_variable <<= PAGE_SHIFT;
	        }
#ifdef CONFIG\_STACK\_GROWSUP
        return PAGE_ALIGN(stack\_top) + random_variable;
#else
	    return PAGE_ALIGN(stack\_top) - random_variable;
#endif
}
{% endcodeblock %}

### ASLR in Android 4.0

而在4.0，即其所谓的支持ASLR的版本上，其实ASLR也仅仅增加了对libc等一些shared libraries进行了随机化，而对于heap, executable和linker还是static的。

对于heap的随机化来说，可以通过

    echo 2 > /proc/sys/kernel/randomize_va_space

来开启。

而对于executable的随机化，由于大部分的binary没有加GCC的-pie -fPIE选项，所以编译出来的是EXEC，而不是DYN这种shared object file，因此不是PIE（Position Independent Executable），所以没有办法随机化；

同样的linker也没有做到ASLR。

### ASLR in Android 4.1

终于，在4.1 Jelly Bean中，Android终于支持了所有内存的ASLR。在第二个对4.1ASLR介绍中，作者列出了从Android 1.5开始用到的安全加强机制：

##### Android 1.5+

* ProPolice to prevent stack buffer overruns (-fstack-protector)
* safe\_iop to reduce integer overflows
* Extensions to OpenBSD dlmalloc to prevent double free() vulnerabilities and to prevent chunk consolidation attacks. Chunk consolidation attacks are a common way to exploit heap corruption.
* OpenBSD calloc to prevent integer overflows during memory allocation

##### Android 2.3+

* Format string vulnerability protections (-Wformat-security -Werror=format-security)
* Hardware-based No eXecute (NX) to prevent code execution on the stack and heap
* Linux mmap\_min\_addr to mitigate null pointer dereference privilege escalation (further enhanced in Android 4.1)
    
##### Android 4.0+

* Address Space Layout Randomization (ASLR) to randomize key locations in memory
    
##### Android 4.1+

* PIE (Position Independent Executable) support
* Read-only relocations / immediate binding (-Wl,-z,relro -Wl,-z,now)
* dmesg\_restrict enabled (avoid leaking kernel addresses)
* kptr\_restrict enabled (avoid leaking kernel addresses)

在Android 4.1中，基本上所有binary都被编译和连接成了PIE模式（可以通过readelf查看其Type）。所以，相比于4.0，4.1对Heap，executable和linker都提供了ASLR的支持。

另外，4.1还增加了几个小的安全加强机制：

* 大部分系统binary都添加了RELRO和BIND\_NOW的编译flag，起作用主要是将GOT表设置成只读，防止之前出现过的[Gingerbreak](http://jon.oberheide.org/files/bsides11-dontrootrobots.pdf "don't root robot")攻击。
* 另外，对dmesg\_restrict / kptr\_restrict的sysctl的利用，有效防止了一些低权限的用户从dmesg/klogctl中读取一些敏感信息，或者读取一些kernel内存中的敏感数据（比如很多/proc下的接口）。

### What's next

之后作者还提到一些还需要继续努力的事：

* ASLR的弱点 - 32-bit，容易破解
* 一些安全的libc调用，比如FORTIFY\_SOURCE
* PaX Hardening，虽然很多不适合手机，但是也可以cherry pick一些啦
* MAC/RBAC，其实这个现在也已经有了，比如SEAndroid...
* Mandatory Code Signing，向IPhone学习吧

另外，作者还提到一个Zygote的问题，为了性能问题，现在Android上所有的进程都是Zygote fork出来的，也就是说很多的地址空间在fork出来后是固定不变的，这样也就出现了一种可能性：a malicious app on a victim’s device leaks address mappings from its own process off to an attacker to assist in exploiting another process (eg. the browser) that might have higher privilege or valuable data.

当然，作者认为这种场景可能性比较小，所以还不算一个大问题。
