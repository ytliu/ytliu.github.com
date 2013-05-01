---
layout: post
title: "opennebula setup note"
date: 2012-09-28 14:30
comments: true
categories: Opennebula
---

这几天搞opennebula这个项目，一个很大的感受就是对于environement setup一定要有详细的笔记，不然过了一段时间就什么也想不起来了，遇到和以前一样的问题也不懂该如何解决。实在浪费时间，于是就把它记下来。

##基本命令
首先是opennebula里面的一些基本命令，这里以kvm为例：
###Cloud Infrastructure Setup
On Cluster:
  
    $ onecluster create cluster01

On Host (kvm):
    $ onehost create host01 --im im_kvm --vm vmm_kvm --net dummy 
    $ onecluster addhost cluster01 host01

<!-- more -->

System Datastore (for running VM, not needed in front end):
    
    mount a NFS directory to /var/lib/one/datastores/0 for each host, if you would like to perform live migration.

Filesystem Datastore (for regular storage), sample config file: 'ds.conf'
    
    $ onedatastore create ds.conf 
    $ onecluster adddatastore cluster01 ds01 
    $ onedatastore update 100
    
add SAFE_DIRS="/var/lib/one/datastores/", mount actual datastore to /var/lib/one/datastores/<datastore_id>

###Set up Virtual Resource
Virtual Machine Image
    
    $ oneimage create img_debian.conf --datastore ds01

Virtual Network

    $ onevnet create net_lease.conf

Virtual Machine Template
    $ onetemplate create vm_debian.conf 
    $ onetemplate instantiate vm_debian --name debian1

Virtual Machine  
    
    $ onevm deploy debian1 brick4 
    $ onevm suspend debian1 
    $ onevm resume debian1 
    $ onevm resubmit debian1 
    $ onevm stop debian1 
    $ onevm delete debian1

###qemu/kvm

create image:
    
    $ /usr/local/kvm/bin/qemu-img create -f qcow2 &lt;img_name&gt; 10G

create image from base:

    $ /usr/local/kvm/bin/qemu-img create -f qcow2 -b &lt;base_img_name&gt; &lt;img_name&gt;

install os:

    $ sudo /usr/local/kvm/bin/qemu-system-x86_64 -k en-us -hda vdisk.img -cdrom /path/to/boot-media.iso 

run image:

    $ /usr/local/kvm/bin/qemu-system-x86_64 -k en-us -hda vdisk.img -m 512

##权限问题
###ssh 权限
需要把front的oneadmin用户的.ssh/id_rsa.pub拷贝到host的oneadmin用户的.ssh/authorized_keys里面

需要把host的oneadmin用户的.ssh/id_rsa.pub拷贝到front的oneadmin用户的.ssh/authorized_keys里面（optional）

###账户设置
两边oneadmin账户的uid，gid一定要相同！

On front:
    
    $ id oneadmin
    uid=1002(oneadmin) gid=1002(oneadmin) groups=1002(oneadmin)
    
On host:
    
    $ addgroup --gid 1002 oneadmin
    $ useradd --uid 1002 -g oneadmin -d /var/lib/one oneadmin

另外在host端的oneadmin还需要有/var/lib/one目录下所有东西的所有权，同时，oneadmin还需要加入**group kvm**（否则无法启动kvm）
    
    $ usermod -a -G kvm oneadmin
    
另外，由于/usr/bin/kvm的权限是root:root，所以这个也可能引发错误，或者将其权限改为oneadmin，或者为oneadmin添加一个root用户组:
    
    $ usermod -a -G root oneadmin

###libvirt
    
！！！！搞了我好久，一直莫名其妙地出一个错误：
    
    internal error process exited while connecting to monitor: kvm: -drive file=/var/lib/one//datastores/0,if=none,id=drive-ide0-0-0,format=raw: 
    could not open disk image /var/lib/one//datastores/0: Permission denied
    
我直接用kvm打开那个镜像都没问题，然后看到一个帖子http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=673427，再加上http://opennebula.org/documentation:archives:rel3.4:kvmg看到这样一句：
    
    Qemu should be configured to not change file ownership. Modify /etc/libvirt/qemu.conf to include dynamic_ownership = 0. To be able 
    to use the images copied by OpenNebula, change also the user and group under which the libvirtd is run to “oneadmin”.

然后去**/etc/libvirt/qemu.conf**把user改成oneadmin,group改成cloud（顺便说下oneadmin也加了一个cloud的group),瞬间就起起来了！

然后，我把oneimage换了一下，把
    NAME          = "debian_new"
    PATH          = /var/lib/one/datastores/ubuntu.11-10.x86-64.20111013.qcow2
    加了三行：
    TYPE          = OS
    PERSISTENT    = YES
    DRIVER        = qcow2

然后就起来了。。。其实我也不知道为什么，可能.qcow2结尾的镜像就是要qcow2的driver才能起起来吧

------

之后配host2的时候还会碰到一个问题就是/var/run/libvirt/libvirt-sock和/var/tmp/one目录下的permission denied，对于前者，给oneadmin再加一个libvirtd的group，对于后者，把/var/tmp/one chown一下就好了

##NFS
###On front

    $ vi /etc/hosts
    203.95.3.3 host1
    203.95.3.4 host2

    $ vi /etc/exports
    /var/lib/one/datastores/0       host1(rw,no_root_squash)   #0是system datastore的编号，是opennebula自带的
    /var/lib/one/datastores/104     host1(rw,no_root_squash)   #104是file system datastore的编号，由onedatastore自动创建的

###On host

    $ cd
    $ mkdir datasdores
    $ mkdir datastores/0
    $ mkdir datastores/104
    $ sudo mount 203.95.3.5:/var/lib/one/datastores/104 /var/lib/one/datastores/104
    $ sudo mount 203.95.3.5:/var/lib/one/datastores/0 /var/lib/one/datastores/0

