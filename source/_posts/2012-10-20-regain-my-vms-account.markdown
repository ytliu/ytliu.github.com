---
layout: post
title: "regain my VM's account"
date: 2012-10-20 13:53
comments: true
categories: 
---

这两天在做opennebula项目的时候遇到一个问题，我忘记掉之前那几个虚拟机的用户密码了！试了好多可能的密码都不对，只能想办法把密码改掉了。

虚拟机的好处在于我可以把它直接mount到物理主机上对其进行操作，不过镜像文件不是一个普通的设备块，不能通过一般的方法mount，只能用不一般的方法了：

首先我有两种镜像，一个是普通的qemu镜像，以.img结尾的文件，一种是KVM的qcow2文件。

##### .img镜像

对于第一种，飞机教了我一个办法：

    $ fdisk -l vm.img
    Disk datastores/vm.img: 8589 MB, 8589934592 bytes
    255 heads, 63 sectors/track, 1044 cylinders, total 16777216 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x0004bacc
    Device Boot      Start         End      Blocks   Id  System
    datastores/vm3.img1   *        2048    15958015     7977984   83  Linux
    datastores/vm3.img2        15960062    16775167      407553    5  Extended
    datastores/vm3.img5        15960064    16775167      407552   82  Linux swap / Solaris

<!-- more -->

注意里面的一个数2048，然后执行下面一条指令：

    $ mount -o offset=$((2048*512)),loop vm.img /mnt

然后就mount到/mnt上去了。

##### .qcow2镜像

第二种镜像得用另外一个方法，这个是从[这里](http://blog.loftninjas.org/2008/10/27/mounting-kvm-qcow2-qemu-disk-images/ "")看到的：

首先要先aptitude install 一个nbd-client，通过

    $ sudo modprobe nbd max_part=8
    $ sudo qemu-nbd --connect=/dev/nbd0 vm.qcow2
    $ sudo fdisk /dev/nbd0
    $ sudo mount /dev/nbd0p1 /mnt

即完成了mount，当然我对其中的原理也不太了解就是了。

##### chroot

最后是如何改密码，我是痛过chroot来做的，这里想简单介绍下这个命令：

刚开始我直接运行：

    $ sudo chroot .

然后报了一个这个错误：

    chroot: failed to run command `/bin/zsh': No such file or directory`

之后了解了chroot的[工作流程](http://www.ibm.com/developerworks/cn/linux/l-cn-chroot/ "")才知道这是怎么回事：

一个相对完整的chroot流程是这样的：
{% codeblock lang:c %}
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main(int argc, char *argv[])
{
    if(argc<2){
	printf("Usage: chroot NEWROOT [COMMAND...] \n");
	return 1;
    }

    printf("newroot = %s\n", argv[1]);
    if(chroot(argv[1])) {
	perror("chroot");
	return 1;
    }

    if(chdir("/")) {
	perror("chdir");
	return 1;
    }

    if(argc == 2) {
	argv[0] = getenv("SHELL");
	if(!argv[0])
	    argv[0] = (char *)"/bin/sh";

	argv[1] = (char *) "-i";
	argv[2] = NULL;
    } else {
	argv += 2;
    }

    execvp (argv[0], argv);
    printf("chroot: cannot run command `%s`\n", *argv);

    return 0;
}
{% endcodeblock %}

在chdir之后会先看看用户有没有指定特定的shell，如果没有则根据环境变量中的SHELL来指定相同的shell，否则会使用用户指定的shell运行（argv += 2)。由于我原来用户的shell是*/bin/zsh*, 且我没有指定特定的shell，所以在chroot之后就默认使用*/bin/zsh*，但是在这个镜像的文件系统中并没有*/bin/zsh*，所以就会报错。

所以我只需要（1）将环境变量中的shell设成*/bin/bash*或者（2）指定一个shell（$ sudo chroot . /bin/bash）就可以了

之后就可以通过运行

    $ passwd root

来改变root的密码了。
