---
layout: post
title: "Android Partition Explaination"
date: 2012-11-12 21:16
comments: true
categories: Android
---

这两天看了AndroidForums上的一个Thread，主要是介绍Android Partition的，感觉写的非常好，很有必要做个简单的笔记。

具体的信息在以下两个链接地址中：

http://androidforums.com/evo-4g-all-things-root/278898-android-partitions-kernels-explained.html

http://androidforums.com/evo-4g-all-things-root/279261-more-information-about-android-partitions.html

这个thread由两个人来写，[novox77](http://androidforums.com/members/novox77.html)主要介绍了下android中partition的大概组织结构，[akazabam](http://androidforums.com/members/akazabam.html)则详细地介绍了android partition的内部机制和原理，以及如何利用其来将application从RAM中移到sdcard上。

以下做一些节选和注释吧：

------

#### Android partitions, kernels explained

    Here are the standard partitions on an Android phone:
    
    /misc     - not sure what this is for.
    /boot     - bootloader, kernel
    /recovery - holds the recovery program (either clockworkmod or RA recovery for a rooted Evo)
    /system   - operating system goes here: Android, Sense, boot animation, Sprint crapware, busybox, etc
    /cache    - cached data from OS usage
    /data     - user applications, data, settings, etc.
    
    The below partitions are not android-specific. They are tied to the hardware of the phone, but the kernel may have code allowing Android to interact with said hardware.
    
    /radio - the phone's radio firmware, controls cellular, data, GPS, bluetooth.
    /wimax - firmware for Sprint's flavor of 4G, WiMax.

    During the rooting process, a critical piece of the process is disabling a security system built into the bootloader that protects these partitions 
    from accidental (or intentional) modification. This is what's referred to as "unlocking NAND." 
    The security system can be set to active or inactive. S-ON means the security is in place (NAND locked). 
    S-OFF means the security is off (NAND unlocked). When S-OFF, you have the ability to modify all partitions. 
    With S-ON, you only have write access to /cache and /data. Everything else is read-only.

也就是说root的过程就是NAND unlock，使得partition可以被任意修改。否则你只能修改/cache和/data文件夹，而不能修改更为关键的/system

    When you flash a custom ROM, that ROM typically includes a kernel and an OS. That means the /boot and /system partitions will be modified at a minimum.

在这里作者把kernel和OS区别开来，在我看来，它这里的kernel主要是一些boot的信息，而OS则指framework和传统意义上的kernel？

    When you do a factory reset (AKA: wipe, hard reset, factory wipe, etc.), you are erasing the /data and /cache partitions. 
    Note that a factory reset does NOT put your phone back to its factory state from an OS standpoint. 
    If you've upgraded to froyo, you will stay on froyo, because the OS lives in /system, and that is not touched during a factory reset. 
   
所以所谓的"factory reset"并不是真正意义上的reset，而是"data reset"。只有root了才有可能做到真正的reset吧？

    The SD card can also be partitioned to include a section dedicated to storing user apps. To create the partition, your SD card needs to be formatted. 
   
这个会在后来akazabam的文章中详细的描述。

    Onto kernels....

之后novox77扯了点关于kernel的东西，这里就不讲了，具体的可以看原贴。

------

#### More information about Android partitions

##### The Basics

    Linux/UNIX type systems have one top level file structure. The top level is called root, and is designated by /. 
    There are no drive letters, and the files and folders under / are not necessarily all stored in the same physical location. 
    
    From this point on, the root of the file system will just be referred to as /. There really isn't much under / by itself. 
    It's generally a small partition. In order to use other partitions, Android must make these partitions available under /. 
    There is a special directory called /dev. In order to use partitions found in /dev, the system must mount them under /. 

也就是说，不过我们mount哪一个设备，它都把mount point作为它自己文件系统的root，即"/"。

##### Mounting

    There are two very important conclusions:
	1) The system mounts and unmounts partitions at boot and shutdown. 
	If you pull the battery out while the system is running or use a poorly written app to reboot the phone, partitions are still mounted, 
	and if the system is writing to them, you could easily corrupt something. 
	Granted, sometimes this is necessary if the phone becomes unresponsive. It's more likely to be a problem if repeatedly done.
	2) Ever wonder why you cannot access the sdcard while the phone is connected to a computer as a disk drive? 
	It's because the computer is mounting the sdcard partition so that you can see it there. What does that mean? 
	It means Windows, Linux, etc. (whatever OS you have on your computer) has direct access to that physical media. 
	No two operating systems can have direct control over physical media at the same time. It would result in massive data corruption. 
	You may be wondering how it's possible to share drives or partitions in the networking world. 
	You can do so because the Computer that has the physical media is still the only host that can physically read and write to the media. 
	Sharing of the data is done at a much higher level and is controlled by the operating system.

第一点告诉我们，不能随便插拔电池，做多了会有问题的。第二点解释了为什么通常我们在把手机插到电脑上时在手机上就没办法访问sdcard上的数据了：因为这时候sdcard被mount在了电脑上，而一个设备不可能被同时mount在两个系统上。

##### Mount Options

    Linux based systems have a file called fstab. That file is a mapping of physical partitions and their mountpoints, 
    along with options it needs to know when mounting said partitions. It uses this file to mount partitions at boot time. 
    So, the fact that you don't have to mount /system, /boot, /data, etc. yourself is because the system does it for you. 

这段话解释了我们linux系统上的fstab，系统在启动的时候会自动在这个文件中查找需要mount的设备和mountpoint，并自动帮我们mount了该mount的东西。

    The partition mounted as /data is mounted as rw. It has to be, otherwise the system would be all but unusable. 
    You wouldn't be able to install apps, change settings, etc. 
    Do not confuse this with file permissions. That is a different discussion for a different time, but at least understand this - 
    file and directories have certain permissions that allow the file owner, the group the file owner belongs to, 
    and everyone else specific access to said file. 
    The point is that a partition must be mounted as rw in order for write permissions to work. 
    If you have permission to write to a file on a partition, but it is mounted as ro, it will not work.

这个区分了mount option和file permission之间的关系。这是两种不同的限制方式，mount option是针对整个文件系统的，而file permission只针对一个文件或是文件夹。也就是说，即使你某个文件有write权限，如果整个文件系统的mount option是ro，那么文件也是不可写的。

##### The System Partition

    The partition mounted as /system is automatically mounted as read only. It's like this because, even with root 
    (the unlocked ability to make changes to the partition mounted as /system) it's dangerous to make changes there 
    if you do not know what you are doing. When you flash a ROM from recovery, it wipes that partition, and writes new contents to it. 
    Recovery scripts, however, are smart enough to mount this partition in rw mode. 
    If you are going to make changes to /system while booted up in Android, you must have /system mounted as rw. 
    Otherwise, you will just get permission errors even thought you have root level permissions.

所以说在android系统中，/system分区是默认mount为ro的，对于那种刷机的程序它可以通过某些tricky的手段将其设为rw模式，然后将里面的数据覆盖掉。

    To do this, you must remount the partition under /system as rw. There are many ways of doing this. 
    All of them are system-wide. What that means is, if you use an app to remount /system as rw, 
    the entire system and any other app will see it such until you reboot the phone or remount it as ro. 
    Root explorer, for example, has an option in the top right corner to mount whatever file system you are currently browsing 
    as either rw or ro depending on what it's currently mounted as. Basically, it toggles between rw and ro. 
    There is also an app in the marketplace called "Mount /system (rw /ro)", which will do that as well if you don't have root explorer. 
    Let's look at it in a little more detail, though.

一般来说，如果你要将/system分区remount成rw的话，一般是要有root权限的，上文中说有一个app叫"Mount /system (rw /ro)"的不要root explorer，这个我没有搞太清楚。

    Should you want to remount the partition mounted under /system as rw using the command line from a terminal emulator, you would run this:

    $ su mount -o remount,rw -t yaffs2 /dev/block/mtdblock4 /system

这里解释下几个参数：
    -o is the option used to specify certain special mount options. 
    a comma separated list following -o are the options you want to specify for mounting.

    remount means that the file system you are mounting is already mounted, and you want to specify some other options. 

    rw means that you want /system to be mounted in read/write mode. 
    
    -t yaffs2 is the filesystem type of the partition. 

    /dev/block/mtdblock4 is the logical location of the partition itself under /dev as previously explained. 

    /system is the location under / where the files belonging to the mtdblock4 partition will be made accessible. 

当用户可以用root权限的账户在模拟器上执行完以上操作，这/system分区就被remount了。这个root权限是指你前面的su，或者某个可以运行root用户的app。

##### Current Mounts

    To see how /system is currently mounted:

    $ mount | grep /system

    You will see something like:

    /dev/block/mtdblock4 /system yaffs2 ro 0 0

    Having an understanding of how this works will help you determine ways of saving space and making the most of your available storage. 
    The three major partitions are /data, /system, and /cache. A majority of your 1 GB of internal storage is used by these. 
    The partitions sizes are set when the partitions are created. By default, this is how they are partitioned:

    /system – 350 MB
    /data – 430 MB
    /cache – 160 MB

    As you can see, you only have 430 MB for apps and data. That’s less than half of the advertised space. 
    There are numerous apps on the market that will display the free space available on each partition, 
    but it can be done from the command line as well by typing this:

    $ df -h

    You will probably notice that both /system and /cache have a lot of free space. As it turns out, 
    most of that free space rarely gets used, and cannot be used by you for apps, at all. It’s wasted space. 
    Cache, of course, will show mostly free space, but it’s necessary for things like OTAs, which will download to that directory. 
    It does need to be somewhat large, but not that big. In any case, there are ways of having more space available than just 430 MB. 
    The best place to start is a2sd.

现在的系统中，如果我们有1G的内存，那么它会将大于一半的分给/system和/cache分区，而真正给应用程序使用的只有很少的一部分。而大部分时候我们是不需要那么多分区的。那么怎么办呢，接下来介绍的a2sd可以部分地解决这些个问题：

##### A2sd, Apps2sd, and File System Types

    With all of this knowledge in mind, you can probably get a better understanding of how something like a2sd works. 
    A2sd is a system devised to move all installed applications to the sdcard. This is by no means the same thing as the built-in froyo apps2sd.

说白了，a2sd就是将一些应用程序移到sdcard上，而不要占用少量的/data分区，但是它和froyo内置的apps2sd又有很大的区别：

    With Froyo apps2sd:
    The developer of a certain app specify that it is allowed to be moved to the sdcard.
    Even when it is, if the app has widgets, those widgets will not be available once the app is on the sdcard. 
    Why? Because the sdcard is unmounted once the phone is connected to a computer as a disk drive. 
    If widgets belonging to such an app were on the homescreen at that time, they would stop working. 
    Google designed Android to avoid such a case by just making those widgets unavailable.

如果我们把一部分应用通过apps2sd移动到sdcard上，有时候是会有问题的，比如如果这个app有widgets，那么如果我们把手机连接到电脑上，这个时候sdcard就被mount到电脑上了，但那些widgets还在屏幕中，这样就会有问题。为了防止这些问题，google会直接将那些个widgets禁掉。

    Only a part of the app is moved, not the entire thing. 
    If you've ever looked at at an app that has been moved to the sdcard in manage applications, 
    you will see that it still is taking up space on internal storage (in /data). 
    The reason this happens goes back to file system types.

另一个更关键的问题是用apps2sd只能将应用的一部分移到sdcard上，而很大一部分会留下来，这是有文件系统类型决定的：

    Linux based systems have a certain number of file system types that it can use. 
    Windows has its own as well. The sdcard needs to be formatted in a file system that is basically universal. 
    This means that no matter what kind of computer you plug the phone into, plus the phone itself, 
    you need to be able to view/modify the contents of the card. That file system is fat32. 
    Both Linux and Windows can view/modify said contents. BUT, Linux (Android) can't execute anything off of a fat32 partition. 
    Its use of it is somewhat limited. That being said, Android cannot move an entire app to the sdcard in its stock condition, 
    as it would be moving it to a fat32 file system, where it would not be able to execute it.

因为一个手机上的sdcard可能会被mount在不同的系统中（比如windows, mac, linux），所以sdcard上的文件系统必须得足够universal。所以sdcard的文件系统一般是fat32，因为它可以被linux和windows都看到，但它有一个很大的限制就是不能再上面运行任何东西。也就是说应用程序运行的部分是不能被移到sdcard上的，否则它就无法被运行了。

    A2sd has none of these issues, and gets around them in a fairly creative way. 
    The sdcard, in stock state, has one partition, which is this fat32 partition. 
    You still need a majority of the card to have this fat32 partition for the purpose of using it normally, 
    but a2sd must use a partition type that is native to Linux. 
    So, the first thing you must do to use a2sd is partition the card into two separate partitions. 
    This can be done in recovery. Once it is done partitioning, it formats the two partitions using a particular file system. 
    The bigger partition, which the user will continue to see as /sdcard and keep all of there data, remains fat32. 
    The smaller partition (usually made between 512 MB and 1 GB) is formatted in the ext3 file system. 
    This ext3 file system type is native to Linux. What does this mean? 
    It means that Android can use that partition on the sdcard the exact same way it could internal storage.

a2sd的原理是这样的，它将sdcard分成了两个区，一个比较大的继续当fat32用，存放正常数据，另一个将其格式化为ext3格式，就可以达到所谓的"native to linux"了。

    Doesn't Android have to mount this new ext3 partition just like it does internal storage partitions and the normal fat32 sdcard partition? 
    Why, yes it does. It mounts the partition just like any other partition, but it makes the mount point /system/sd. 
    Once you've created the ext3 partition, you can browse /system/sd. It will look just like a directory in internal storage, 
    but since it's a mountpoint, when you view it, you're looking on the sdcard, just in the smaller, ext3 partition. 
    Having done this, you've basically fooled the system into thinking you have more internal storage. 
    The issue is that Android will look for apps in two main places - /data/app and /system/app. 
    if you just stuck an apk (Android application) in /system/sd, Android system would never find it, as it will never look there. 
    For those interested in seeing how the sdcard partitions are mounted, run these two commands:

    $ mount | grep /sdcard
    $ mount | grep /system/sd

    The output for /system/sd, for example, looks like:

    /dev/block/mmcblk0p2 on /system/sd type ext3 (rw,noatime,nodiratime,errors=continue,data=writeb ack)

    Both of those commands, though, will show how the fat32 sdcard partition is mounted (/mnt/sdcard) and 
    how the ext3 sdcard partition is mounted (/system/sd). As you can see, mmcblk0p2 is the ext partition, 
    while the majority of the card (the fat32 partition) is mmcblk0p1. 
    Another quick important point is that /system/sd is mounted as rw so that you can make changes. 
    Remember this - if a partition is mounted as ro (/system) but a directory under it is used as a mountpoint (/system/sd) 
    you will still be able to write to whatever is mounted under that second directory as long as the partition is mounted as rw. 
    That being said, you can leave /system mounted as ro, and still always make changes to /system/sd.

所以，sdcard被分出来的那个区被mount在了/system/sd上，虽然/system是ro模式的，但并不影响它下面的子目录是可写的。但是有一个问题，android一般是在/data/app和/system/app中查找有没有应用，我们将其mount到/system/sd上就算有应用也不能被发现啊？

    The first thing a2sd does is move all applications from /data/app and /data/app-secure to /system/sd/app and /system/sd/app-secure. 
    Remember that with just this step, the system would not see apps anymore. At this point, a2sd makes use of something called a symlink. 
    a2sd removes the /data/app and /data/app-secure directories, then creates symlinks (shortcuts) called /data/app and /data/app-secure 
    that point to directories in /system/sd for apps. This means that the system will continue to look in /data/app and /data/app-secure for apps, 
    but will be directed to /system/sd. Basically, the system doesn't care where files actually are. 
    It only cares that it can find them where it knows to look.

这里就用了symbol link的方式将/data/app和/system/app链接到/system/sd/app和/system/sd/app-secure上，这样就解决了上面说的问题了。

    A2sd can also be used to move dalvik cache to the sdcard. It does this in exactly the same manner as moving apps. 
    Dalvik cache is normally stored in /data/dalvik-cache. A2sd creates a directory in /system/sd for dalvik-cache, 
    then creates a /data/dalvik-cache symlink that points to the real location.

同样也可以用同样的方法将dalvik-cache移到sdcard上。

    As dalvik cache is stored in /data by default, it takes up your usable storage, needlessly. 
    That is why it can be moved to the sdcard ext3 partition. If you choose to, though, it can also be moved to the /cache partition. 
    /cache is normally just unused space on internal storage that is way bigger than it needs to be. 
    Thus, dalvik cache can be moved there instead, too. The idea is the same, but it doesn't use symlinks. 
    It does some creative work with the mount command to make the system look there for it. 

dalvik-cache也可以被移到/cache里面，虽然cache需要留一些空间，但是当前系统留的大于它需要的。而且移到/cache里就不需要用symbolic-link了。


##### Other Space Saving Options

    Some people do not want to use a2sd, as they do not have good enough sdcards and are not interested in buying a new one. 
    A2sd will not work well with a slower card, such as the one that comes stock with the Evo. 
    However, it is possible to reclaim some of the unusable internal storage. 
    If you remember, /data, /system, and /cache make up a majority of your internal storage. They can be resized with this mod. 

很显然，将app移到sdcard上必然会减慢app的运行速度，特别是如果你的sdcard很弱的话。不过把一些程序从/data移到/system或/cache倒是一个可行的方法。
