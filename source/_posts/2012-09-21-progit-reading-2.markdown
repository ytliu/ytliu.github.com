---
layout: post
title: "progit reading 2"
date: 2012-09-21 16:50
comments: true
categories: Git
---

上章介绍了git internal的内容，接下里会从git更接近用户的角度来说。

------

#git basic
git中的文件有以下几种状态，而所有的命令也就是对于这几种状态的查看和转换：

![git file status](http://ytliu.github.com/images/2012-09-16-4.png "git file status")

一般情况下，在新建一个文件后，需要*git add*将其变成tracked file，如果修改了一个该文件，则同样需要使用*git add*将其变成staged file，只有staged的文件才会在*git commit*的时候commit成功。

**git status**

这个可以用来查看当前文件的状态。如果我们在commit的参数中加上*-a*，就可以自动将tracked file变成staged file了。当然也可以忽略一些文件，这些都是写在.gitignore文件下的，那么，如何unstage一个文件呢？其实在你使用*git stage*命令的时候就会有提示: 

    $ git reset HEAD <file>

同样的，我们也可以把一个modified file变成unmodified file:
    
    $ git checkout -- <file>

<!-- more -->

**git diff**

由于*git status*只能告诉我们哪些文件被修改了，而不能告诉我们都修改了哪些具体内容，所以有一个*git diff*命令来补充这个功能。

*git diff*显示的是changed but not staged的文件，而*git diff --cached*显示的是staged的文件

**git commit**

这个就是将stage area里面的东西进行提交，在提交的时候需要指定*-m*参数。另外还有一个比较有用的命令叫*git commit --amend**用来返回到最近的一次commit之前，然后加一些其它文件，然后再自动提交一遍，这样就不会有两个相似的commit了。

**git rm**

如果只是简单地用*rm*命令，那么它还是处于unstage的状态，用*git rm*会将其变成untrack状态。另外，如果你加了参数*-f*则可以将其从index中删除，这样如果之前没有commit这个文件的话之后也就无法恢复了。

还有一个比较有用的命令是:
    
    $ git rm --cached <file>

这个命令可以在硬盘上保留file，但是将其从working tree中删除。比如说你忘记把file添加到.gitignore文件中的话就可以用这个命令。

同样，这个命令也支持正则表达式。

**git mv**

在git中，如果你用*git mv*命令，在git history中应该是没有renaming这个记录的，那么为什么还有mv这个命令呢？在git中：*git mv file_from file_to*这一个命令相当于

    $ mv file_from file_to
    $ git rm file_from
    $ git add file_to

这三条命令。

**git log**

用来查看git commit history，有许多有用的参数，比如*--stat*可以查看一些简略的信息，*-p -2*可以列出最近的两条entris，*--pretty=oneline*可以将history简略成一行，同样可以指定format——*--pretty=format:"%h - %an, %ar : %s*，而这些参数的意义如下: 

    Option  Description of Output
    %H  Commit hash
    %h  Abbreviated commit hash
    %T  Tree hash
    %t  Abbreviated tree hash
    %P  Parent hashes
    %p  Abbreviated parent hashes
    %an  Author name
    %ae  Author e-mail
    %ad  Author date (format respects the –date= option)
    %ar  Author date, relative
    %cn  Committer name
    %ce  Committer email
    %cd  Committer date
    %cr  Committer date, relative
    %s  Subject

另外还有*--graph*用来列出一个history graph，还有一些和时间相关的log，比如*--since=2.weeks*，列出两周内的commit信息...

    Option  Description
    -(n)  Show only the last n commits
    --since, --after  Limit the commits to those made after the specified date.
    --until, --before  Limit the commits to those made before the specified date.
    --author  Only show commits in which the author entry matches the specified string.
    --committer  Only show commits in which the committer entry matches the specified strin

**git tag**

这个先不说了，感觉用不太到...

**git remote**

主要就7个命令：

    $ git remote add local_name url
    $ git remote -v
    $ git remote show local_name
    $ git fetch [local_name | url]
    $ git push [local_name | url] [branch_name]
    $ git remote rename old_name new_name
    $ git remote rm local_name

**tips and tricks**

*git aliases*: 可以将很多命令缩写，比如:
    
    $ git config --global alias.co checkout
    ......
    $ git config --global alias.unstage 'reset HEAD --'
    $ git config --global alias.last 'log -1 HEAD'

#git branch

git branch的抽象是这样子的，每次commit都会产生一个commit object:

![commit object](http://ytliu.github.com/images/2012-09-16-5.png "commit object")

而一个branch则有一个个commit object通过pointer串起来的:

![branch](http://ytliu.github.com/images/2012-09-16-6.png "branch")

在整个git目录中有一个很特别的index叫做HEAD，它指向了当前的branch:

![current branch](http://ytliu.github.com/images/2012-09-16-7.png "current branch")

之后，就是对branch的一些操作:

    $ git checkout -b new_branch

这个等价于:

    $ git branch new_branch
    $ git checkout new_branch

**merge**

merge需要先指定base的branch，然后再和一个新的branch进行合并：

    $ git checkout base_branch
    $ git merge another_branch

如果两个branch不在一个history中(如图所示):
    
![merge branch 1](http://ytliu.github.com/images/2012-09-16-8.png "merge branch 1")

则会找到一个common ancestor，之后创建一个新的commit object，它的parents为两个branch，

![merge branch 2](http://ytliu.github.com/images/2012-09-16-9.png "merge branch 2")

如果两个branch有conflic，则需要通过diff工具进行merge，merge完之后用*git add*和*git commit*进行确认。

**delete branch**

删除一个branch的命令为:

    $ git branch -d branch_name

**remote branch**

主要是个命令的使用:

    $ git clone url
    $ git fetch local_name
    $ git push local_name branch_name
    $ git checkout --track local_name/branch_name 	# set up a tracking branch
    $ git checkout -b new_branch_name local_name/branch_name
    $ git push local_name :branch_name 	# delete a remote branch from the server

**rebasing**

和merge不同，rebase是将一个branch的改动replay到另一个branch上，比如说，一个简单的例子:

![rebase branch 1](http://ytliu.github.com/images/2012-09-16-10.png "rebase branch 1")

以experiment为current branch，*git merge master*的结果是这样的:

![rebase branch 2](http://ytliu.github.com/images/2012-09-16-11.png "rebase branch 2")

而以experiment为current branch，*git rebase master*的结果是这样的:

![rebase branch 3](http://ytliu.github.com/images/2012-09-16-12.png "rebase branch 3")

它的过程是这样的：先找到一个common ancestor，将两个branch的diff结果保存到一个文件里面，将当前的branch重新设成新的branch，之后将这些diff都应用到这个branch中。

对于rebase，有一点要注意的:

{% blockquote %}
Do not rebase commits that you have pushed to a public repository.
{% endblockquote %}

否则会造成很多你意想不到的结果。

#git distribution

/* fix me */

#git tools

**revision selection**

主要是几条命令:
    $ git log --abbrev-commit --pretty=oneline
    $ git rev-parse topic1 
    ca82a6dff817ec66f44342007202690a93763949
    $ git reflog
    $ git log -g branch 	# see reflog of master
    $ git show HEAD^ 	# see the parent commit of HEAD
    $ git show HEAD~2 	# see the grandparent commit of HEAD...

另外也可以看commit range，比如如图所示:

![commit range](http://ytliu.github.com/images/2012-09-16-13.png "commit range")

branch1..branch2的意思是: all commits reachable by branch2 that aren't reachable by branch1。

branch1...branch2的意思是: all commits that are reachable by either of two references but not by both of them。

所以:
    $ git log master..experiment
    D 
    C

    $ git log experiment..master
    F
    E

    $ git log master...experiment
    F
    E
    D
    C

另外下面三个命令是等价的:

    $ git log refA..refB
    $ git log ^refA refB
    $ git log refB --not refA

**interactive staging**

如果为*git add*加一个*-i*选项，则可以进入interactive的模式，之后会有几个命令让你选择:

    Commands * 1: status 2: update 3: revert 4: add untracked 5: patch 6: diff 7: quit 8: help 

之后就可以交互式地stage每个文件，同时，还可以stage文件的一部分（5：patch），很方便。

**stashing**

stashing的作用是将当前目录的状态（不管是modified还是staged的文件）都保存起来，然后做其它的工作，在之后可以从中恢复过来。

另外，可以用*git stash list*列出当前被stash的内容，之后用*git stash apply*来应用某个，或者*git stash drop*来丢弃某个，或者直接用*git stash pop*将最近的一个apply然后drop。

还有一种比较牛逼但比较用不到的场景，你apply了一个stash，do some work，然后想unapply之前的工作（指originally come from the stash），可以用这个命令:

    $ git stash show -p stash@{0} | git apply -R

**rewriting history**

之前有说过*git commit --amend*，这里要注意的是这个命令就像一个小的rebase，如果你已经push到remote的话最后就不要用这条命令了。

如果你想要改之前的commit，那么就需要用*git rebase*了，加上*-i*选项:

    $ git rebase -i HEAD~3

之后会有几个选项让你选:

    p, pick = use commit
    r, reword = use commit, but edit the commit message
    e, edit = use commit, but stop for amending
    s, squash = use commit, but meld into previous commit
    f, fixup = like "squash", but discard this commit's log message
    x, exec = run command (the rest of the line) using shell

这里我遇到一个问题: 在退出编辑器的时候报错——could not execute editor，解决的办法是:

    $ git config --global core.editor "/usr/bin/vim"

总的来说这是一个很强大的功能，可以修改commit message，可以reordering commit，可以squashing commit，也可以splitting commit，还有一个非常牛逼的filter-branch功能，可以用来把一个文件从每次的commit中删除。在这里就不详细介绍了。

**debugging with git**

/* fix me */

**submodules**

submodule的作用是将某段代码目录作为子目录导入你的主工程中，让别人可以下载，另外，由于这个子工程可能是别人开发的，所以需要分开来commit。

可以通过以下命令来建立submodule:
 
    $ git submodule add url local_module_name

但是如果你直接用*git clone*是不会把这个submodule下下来的，需要运行以下两个命令:

    $ git submodule init
    $ git submodule update

**subtree merging**

/* fix me */

#git customization

**git configuration**

一般，git的配置参数在/etc/gitconfig和~/.gitconfig两个文件里面。

git可以设置默认的commit message:

    $ git config --global commit.template <file>

还有其它的:

    $ git config --global core.pager [more | less | '']
    $ git config --global help.autocorrect [0 | 1]
    $ git config --global color.ui [true | false]
    $ git config --global color.branch [true | false | always]
    $ git config --global color.diff [true | false | always]
    ......

**git attributes**

可以在.gitattributes中加入**.b binary*来表示以b结尾的文件都是binary file，这样在*git diff*的时候就不会diff它了，还可以通过*\*.doc diff=word*来表示用word来进行diff，而word可以通过:

    $ git config diff.word.textconv strings

来进行设置。

**git hooks**

这是一个很强大的工具，你可以将所有可执行脚本通过合法的命名方式放在.git/hooks目录下，主要分为client-side和server-side的hooks，比如在client-side中，可以规定在commit之前运行一个脚本进行检查...

这是一个蛮有趣的话题，我会单独写一篇文章来说明下~

------

以上基本上就是progit的全部内容，还有一些我觉得暂时还用不到就没有详细写，有一些我可能还会再单独更详细地描述。

总之，git真的是一个很神奇的用于提高开发效率和可靠性的工具，如果能熟练地掌握它的用法将能极大地提高工作效率。

