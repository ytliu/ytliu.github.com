---
layout: post
title: "progit reading 1"
date: 2012-09-16 22:54
comments: true
categories: Git
---

花了大概两周的时间吧，把《progit》那本书看完了（我看书实在是比较慢，特别是这种英文书）。发觉git实在是一个太强大的工具了，以至于我看完了一遍又把前面所说的功能忘记了。。。orz。。。于是乎决定花一周时间重新回顾一下，顺便把一些牛逼的地方记下来。

##Plumbing & Porcelain
因为这是progit的最后一章，也是我刚刚看完的一章，还比较有印象。更重要的是这是git的internal的机制，对于深入理解git有很大的帮助，所以想先把这章啃下来。

porcelain是瓷器的意思，在这里是指git中比较user-friendly的命令，比如文中介绍的将近30条git命令，包括checkout, brance, commit, 以及所有的remote命令等等。而plumbing是水管的意思，和porcelain相对的，指的是一些和unix style类似的low-level的可以直接在脚本中执行的命令（其实我也没搞懂为什么要交porcelain和plumbing两个名字，感觉没什么关系啊？）。事实上，如果我没有理解错的话，porcelain在git中应该就是由一系列plumbing命令组成的。比如git commit命令就是由一个叫做“git commit-tree”的plumbing命令完成的，至于什么是commit-tree，以及这个tree是怎么形成的，这个会再接下来慢慢解释。

首先来看下.git目录下都有些什么。

    $ ls .git
    HEAD
    config
    description
    hooks/
    index
    info/
    objects/
    refs/

这些是在*git init*的时候初始化就默认产生的，其中description现在还不需要考虑，config主要用来配置一些program-specific的参数选项，info是一个目录，包含了一些需要被ignore的文件模式，而hooks定义了一些client或server端用于用户进行脚本定制的功能，这会在接下来详细介绍。而在这一节中主要描述了以下四个对象：**HEAD**，**index**，**objects**，**refs**。这是git最internal的部分。

<!-- more -->

##git objects
object主要由两种组成：*tree object*和*commit object*，在介绍这两个object之前首先要说明下git的文件系统，git是一个content-addressable文件系统，换句话来说，对于git的核心存储来说仅仅是一个key-value数据库，你能向里面插入任何数据，并得到一个相对应的hash值用于之后的访问。这里有两个plumbing命令*hash-object*和*cat-file*，比如在新建的git仓库中输入以下命令；
    $ echo 'test content' | git hash-object -w --stdin
**-w**表示对该object进行存储，将会返回：
    d670460b4b4aece5915caf5c68d12f560a9fe3e4
这个时候你查看.git/objects目录下的内容将会看到：
    $ find .git/objects -type f
      .git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
其中d6是这个hash值的前两个数字，如果运行*cat-file*将会得到该object：
    $ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
      test content
**-p**表示把里面的内容打印出来，而另一个参数**-t**这是将该object的类型打印出来：
    $ git cat-file -t d670460b4b4aece5915caf5c68d12f560a9fe3e4
      blob
注意这里的*blob*是一种type，它是属于整个*tree object*的叶子节点的类型，而那些中间节点都可以叫做tree。于是现在就可以详细来说说*tree object*了：

###tree object
首先我们来运行以下的命令：
    $ vi test1
    $ vi test2
    $ find .git/objects/ -type f
    $ git add .
    $ find .git/objects/ -type f
      .git/objects//18/0cf8328022becee9aaa2577a8f84ea2b9f3827
      .git/objects//9f/71d140ff7712ec3a6dda42c09078fd290a3a61
    $ git ci -m "first commit"
    $ find .git/objects/ -type f
      .git/objects//18/0cf8328022becee9aaa2577a8f84ea2b9f3827
      .git/objects//92/bfc1480834507340a9bb30ac05fb4965785875
      .git/objects//9f/71d140ff7712ec3a6dda42c09078fd290a3a61
      .git/objects//dc/7a09861107e178fa0016fb48300b569de5c64c
    $ git cat-file -p 92bf
      tree dc7a09861107e178fa0016fb48300b569de5c64c
      author ytliu <mctrain016@gmail.com> 1347865347 +0800
      committer ytliu <mctrain016@gmail.com> 1347865347 +0800

      first commit
    $ git cat-file -p dc7a
      100644 blob 9f71d140ff7712ec3a6dda42c09078fd290a3a61 test1
      100644 blob 180cf8328022becee9aaa2577a8f84ea2b9f3827 test2
    $ git cat-file -t dc7a
      tree

到目前为止，在first commit之后，整个数据存储的树结构是这样的：

![tree object 1](http://ytliu.github.com/images/2012-09-16-1.png "tree object 1")

记下来我们新建一个目录dir，并再次commit一次：
    $ mkdir dir
    $ vi dir/test3
    $ git add .                 
    $ find .git/objects/ -type f
      .git/objects//18/0cf8328022becee9aaa2577a8f84ea2b9f3827
      .git/objects//92/bfc1480834507340a9bb30ac05fb4965785875
      .git/objects//9f/71d140ff7712ec3a6dda42c09078fd290a3a61
      .git/objects//dc/7a09861107e178fa0016fb48300b569de5c64c
      .git/objects//df/6b0d2bcc76e6ec0fca20c227104a4f28bac41b
    $ git ci -m "second commit" 
    $ find .git/objects/ -type f
      .git/objects//18/0cf8328022becee9aaa2577a8f84ea2b9f3827
      .git/objects//41/77687db29c641515b10f13536dd70fae4ed142
      .git/objects//84/ddd13670be5f3636586915421cd98035ad9c66
      .git/objects//92/bfc1480834507340a9bb30ac05fb4965785875
      .git/objects//9f/71d140ff7712ec3a6dda42c09078fd290a3a61
      .git/objects//c5/04b82867a9a4104974edd54c56f01856d9426b
      .git/objects//dc/7a09861107e178fa0016fb48300b569de5c64c
      .git/objects//df/6b0d2bcc76e6ec0fca20c227104a4f28bac41b
    $ git cat-file -t 4177      
      tree
    $ git cat-file -p 4177
      040000 tree c504b82867a9a4104974edd54c56f01856d9426b dir
      100644 blob 9f71d140ff7712ec3a6dda42c09078fd290a3a61 test1
      100644 blob 180cf8328022becee9aaa2577a8f84ea2b9f3827 test2
    $ git cat-layoutfile -p 84dd
      tree 4177687db29c641515b10f13536dd70fae4ed142
      parent 92bfc1480834507340a9bb30ac05fb4965785875
      author ytliu <mctrain016@gmail.com> 1347865426 +0800
      committer ytliu <mctrain016@gmail.com> 1347865426 +0800

	second commit
    $ git cat-file -p c504
      100644 blob df6b0d2bcc76e6ec0fca20c227104a4f28bac41b test3
可以看出，现在多了两个*tree object*，当前的树结构是这样的：

![tree object1 2](http://ytliu.github.com/images/2012-09-16-2.png "tree object 2")

也就是说在新建一个dir的时候会新建一个*tree object*，而它指向的是这个dir下的blob或其它tree，另外，在进行一次commit的时候也会新建一个*tree object*，其包含的内容是staging area里面的所有东西。另外，git也提供了和*tree object*相关的plumbing命令：*write-tree*和*read-tree*。*write-tree*用于新建一个tree，把staging area里面的object就涵盖进来，而*read-tree*则是将一个tree读入staging area，比如运行以下命令：
    $ git write-tree
      4177687db29c641515b10f13536dd70fae4ed142
    $ git cat-file -p 4177
      040000 tree c504b82867a9a4104974edd54c56f01856d9426b dir
      100644 blob 9f71d140ff7712ec3a6dda42c09078fd290a3a61 test1
      100644 blob 180cf8328022becee9aaa2577a8f84ea2b9f3827 test2
    $ git read-tree --prefix=bak 4177
    $ git write-tree
      389a980f31bfb78f9bf7e41d85fb3a1736a54f8c
    $ git cat-file -p 389a
      040000 tree 4177687db29c641515b10f13536dd70fae4ed142 bak
      040000 tree c504b82867a9a4104974edd54c56f01856d9426b dir
      100644 blob 9f71d140ff7712ec3a6dda42c09078fd290a3a61 test1
      100644 blob 180cf8328022becee9aaa2577a8f84ea2b9f3827 test2
    $ ls                  
      dir test1 test2
可以看出*write-tree*新建了一个*tree object*，并通过read-tree被标为bak，成为另一个tree的subtree，但是我们通过**ls**并不能显示出来这个tree——bak。

###commit object
在之前的cat-file命令中可以看到有另一类object
    $ git cat-file -p 84dd
      tree 4177687db29c641515b10f13536dd70fae4ed142
      parent 92bfc1480834507340a9bb30ac05fb4965785875
      author ytliu <mctrain016@gmail.com> 1347865426 +0800
      committer ytliu <mctrain016@gmail.com> 1347865426 +0800

      second commit
    $ git cat-file -t 84dd
      commit
这就是一个*commit object*，是在每一次commit的时候产生的。可以看到，它所指向的*tree object*为4177，即：
    $ git cat-file -p 4177
      040000 tree c504b82867a9a4104974edd54c56f01856d9426b dir
      100644 blob 9f71d140ff7712ec3a6dda42c09078fd290a3a61 test1
      100644 blob 180cf8328022becee9aaa2577a8f84ea2b9f3827 test2
这个很好理解，当然了，相应与*write-tree*，同样也有一个*commit object*相关的plumbing命令：*commit-tree*，用法大概是这样的：
    $ echo 'first commit' | git commit-tree 4177
      d8c7554eb5ee1a0eca359f3d58b99529ac94529c
    $ echo 'second commit' | git commit-tree 389a -p d8c7 
      fc45c76849a24fe3e6b98fec5f17194c0c5f52a3
    ......
具体的就不详说了，前面的echo是commit message，**-p**选项表示parent。这个时候如果你运行git log:
    $ git log
      commit 84ddd13670be5f3636586915421cd98035ad9c66
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 15:03:46 2012 +0800

      second commit

      commit 92bfc1480834507340a9bb30ac05fb4965785875
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 15:02:27 2012 +0800

      first commit
它只显示了之前commit的记录，那么新commit的third commit和fourth commit呢？原因很简单，因为我在commit-tree third commit的时候并没有指定**-p**，所以它并没有接着second commit下去，而是自己新开了一个：
    $ git log fc45
      commit fc45c76849a24fe3e6b98fec5f17194c0c5f52a3
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 16:20:10 2012 +0800

      fourth commit

      commit d8c7554eb5ee1a0eca359f3d58b99529ac94529c
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 16:18:13 2012 +0800

      third commit
当然，我也没有指定它们属于那个branch，所以它现在是属于一个没有被记录的detached HEAD状态，不属于任何一个branch。如果需要为它加一个branch，可以用：
    $ git co fc4f -b new_branch

到目前为止，整个git仓库的objects的关系可以用下图来表示：

![object 1](http://ytliu.github.com/images/2012-09-16-3.png "object 1")

##git references
其实.git/refs的目的主要是为了更方便用户记忆object，而不用每次都用一个那么长的SHA-1，比如：
    $ cat .git/refs/heads/master 
      84ddd13670be5f3636586915421cd98035ad9c66
这个就是传说中的master是怎么被关联到最新的commit的。你可以用git提供的plumbing命令*update-ref*来更新不同的ref：
    $ git update-ref refs/heads/master fc45
这个时候master就指向fourth commit了。当然你也可以用这个命令新建ref：
    $ git update-ref refs/heads/test 84dd
    $ git co test
    $ git log
      commit 84ddd13670be5f3636586915421cd98035ad9c66
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 15:03:46 2012 +0800

      second commit

      commit 92bfc1480834507340a9bb30ac05fb4965785875
      Author: ytliu <mctrain016@gmail.com>
      Date:   Mon Sep 17 15:02:27 2012 +0800

      first commit

还有一种reference是remote *reference*，可以用remote add来添加：
    $ git remote add origin git@github.com:something.git
然后吧本地的master分支push上去。
    $ git push origin master
然后你就可以在refs/remotes/origin/master下看到当前最新的分支情况了~

##HEAD
HEAD其实就是一个reference指向当前branch的引用：
    $ cat .git/HEAD
      ref: refs/heads/test
我们可以直接修改这个文件，也可以用git提供的命令*symbolic-ref*来修改：
    $ git symbolic-ref HEAD refs/heads/test
    $ cat .git/HEAD
      ref: refs/heads/test

##packfiles
接下来是一个很重要的概念——packfile。具体的场景是这样的：

假设我们有一个很大的文件largefile，它的hash是fb699e017d85f1f0d037f0417a7e17a449533ecc：
    $ git cat-file -s fb69
      132480
**-s**表示object的大小，这个时候我们对其进行了一个小小的修改，并重新commit：
    $ echo "modify a little" >> largefile
    $ git add largefile
    $ git ci -m "modify largefile"
这个时候largefile的hash就变成了084d9fa99e4558d38cba7006e3b28f6c87a8fd86；
    $ git cat-file -s 084d
      132496

可以看出，在git里面存了两个基本差不多的largefile的object，这样是非常浪费空间的。其实git在disk上存的object是一种叫做*loose object*的格式，而在一段时间之后git会将这些*loose object*打包。当然这种情况一般会在两种情况下发生: 执行*git gc*命令，以及push到remote server：
    $ git gc
    $ find .git/objects/ -type f
      .git/objects//1e/f02bee3de76100febdefb8c55bf99fcfbdf714
      .git/objects//45/699e25f45a743e08c0909ce1925641f9c03e2e
      .git/objects//info/packs
      .git/objects//pack/pack-4ba1c4a110a95c95d7fc1a33d0c5916bb4c10a34.idx
      .git/objects//pack/pack-4ba1c4a110a95c95d7fc1a33d0c5916bb4c10a34.pack
可以看到，现在只剩下5行了，若我们用plumbing命令verify-pack来查看的话可以看到：
    $ git verify-pack -v .git/objects/pack/pack-4ba1c4a110a95c95d7fc1a33d0c5916bb4c10a34.pack 
      eb5efdaed6e57b4356a6758e77c998f1efd009ed commit 221 151 12
      5a937dca309316f541a433e624868dfe5196c165 commit 218 150 163
      fc45c76849a24fe3e6b98fec5f17194c0c5f52a3 commit 218 149 313
      84ddd13670be5f3636586915421cd98035ad9c66 commit 218 149 462
      d8c7554eb5ee1a0eca359f3d58b99529ac94529c commit 43 53 611 1 84ddd13670be5f3636586915421cd98035ad9c66
      92bfc1480834507340a9bb30ac05fb4965785875 commit 169 119 664
      902c38d7fbadea10287a581b9c557fea63d8b00c tree   133 131 783
      c504b82867a9a4104974edd54c56f01856d9426b tree   33 44 914
      df6b0d2bcc76e6ec0fca20c227104a4f28bac41b blob   6 15 958
      084d9fa99e4558d38cba7006e3b28f6c87a8fd86 blob   132496 316 973
      9f71d140ff7712ec3a6dda42c09078fd290a3a61 blob   7 16 1289
      180cf8328022becee9aaa2577a8f84ea2b9f3827 blob   6 15 1305
      f5adadb9b7cf88c0aa57bc4c810d5c2a68d93c5c tree   133 131 1320
      fb699e017d85f1f0d037f0417a7e17a449533ecc blob   13 22 1451 1 084d9fa99e4558d38cba7006e3b28f6c87a8fd86
      389a980f31bfb78f9bf7e41d85fb3a1736a54f8c tree   126 126 1473
      4177687db29c641515b10f13536dd70fae4ed142 tree   96 98 1599
      dc7a09861107e178fa0016fb48300b569de5c64c tree   66 70 1697
      non delta: 15 objects
      chain length = 1: 2 objects
      .git/objects/pack/pack-4ba1c4a110a95c95d7fc1a33d0c5916bb4c10a34.pack: ok
原来第一个largefile的object fb69 现在指向了084d，而其大小也变成了13，而084d作为修改过的largefile，大小还是132496。另外，git还把其它的一些类似的object进行了pack，更充分地缩减了空间。

##refspec
这个是和remote branch相关的，即当你运行了*git remote add*命令后，在.git/config下面会有一个诸如：
    [remote "origin"]
	url = git@github.com:something.git
	fetch = +refs/heads/*:refs/remotes/origin/*
的entry，其中origin是remote端在本地的reference，url是remote端的地址，fetch是你在执行fetch命令时的操作，格式为(+)src:dst，其中src是remote side的匹配模式，dst是local side的匹配模式，+为可选，表示即使不是fast-forward也要更新reference。当然你也可以在每次fetch的时候手动指定。另外对于push同样有这种模式，只需要在config中加上一个push行就行了。


##remove object
最后一个想讲的是如何真正地删除一个object。比如你有一个很大的object，你用*git rm*把他删除了，但是你并没有真正地把它从怎个历史中删除，任何一个其他人要fetch你的git仓库，都会把这个很大的object也一起fetch过去。那么，要如何才能真正意义上地删除一个大的object呢？

/* fix me */

------

接下来我会从头开始回顾：*git basic*, *git branch*, *git distribution*, *git tools*, 以及*git customization*。

