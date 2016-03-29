# lab1实验报告
## 1.实验思考题
### thinking 1.1
##### 问题描述：
>思考以下几条指令有何作用？
+ ls -l
+ mv test1.c test2.c
+ cp test1.c test2.c
+ cd ..

##### 问题解答：

根据官方manual pages文档，
> ls - list directory contents.</br>
> -l use a long listing format
>
> mv - Rename SOURSE to DEST, or move SOURSE(s) to DIRECTORY.
>
> cp - Copy SOURCE to DEST, or multiple SOURCE(s) to DIRECTORY.
>
> cd - Change the current directory to dir.

所以，</br>
**ls -l** 就是用长列表格式（详细信息）打印出当前列表内容；</br>
**mv test1.c test2.c** 将文件test1.c重命名为test2.c；</br>
**cp test1.c test2.c** 复制test1.c为test2.c,相当于另存为；</br>
**cd ..** 返回上级目录。

### thinking 1.2
##### 问题描述：
> 思考 grep 指令的用法，例如如何查找项目中所有的宏？如何查找指定的函数名？

##### 问题解答：
grep是一种流过滤器，可以在输入流中抓取指定的正则表达式匹配,用以处理文本十分方便，比ctrl+f更强大；</br>
manual中有详细的grep用法，grep确实有很强大的功能；</br>
-a,-i,-r三个选项使用率最高；
+ -a 将二进制文件作为文本文件处理
+ -i 匹配时忽略大小写
+ -r 递归目录中的所有文件

1. 查找项目中的所有宏(以我的lab1项目为例）：grep -r '#define' 14061213-lab/
2. 查找指定的函数名(以main函数为例)：grep -r 'main' 14061213-lab/

### thinking 1.3
##### 问题描述：
> 思考 gcc 的 -Werror 和 -Wall 两个参数在项目开发中的意义。

##### 问题解答：
+ -Wall 功能：在标准错误输出上输出所有可选的警告级错误信息。
+ -Werror 功能：使用该选项后，GCC发现可疑之处时不会简单的发出警告就算完事，而是将警告作为一个错误而中断编译过程。

编译器警告很多时候可以帮助我们发现一些代码上的错误，有些程序员在编程时忽略warning是一个非常坏的习惯，很多bug就隐藏在warning中，列出所有警告信息有助于我们发现大部分警告信息；而-Werror选项则强制程序员消除warning，对写出高质量代码很有帮助。善用编译器警告可以大大加快我们调试程序的进度。

### thinking 2.1
##### 问题描述：
> 1.深夜，小明在做操作系统实验。困意一阵阵袭来，小明睡倒在了键盘上。等到小明早上醒来的时候，他惊恐地发现，他把一个重要的代码文件 printf.c 删除掉了。苦恼的小明向你求助，你该怎样帮他把代码文件恢复？
>
>正在小明苦恼的时候，小红主动请缨帮小明解决问题。小红很爽快地在键盘上敲下了 git rm printf.c，这下事情更复杂了，现在你又该如何处理才能弥补小红的过错呢？
>
>处理完代码文件，你正打算去找小明说他的文件已经恢复了，但突然发现小明的仓库里有一个叫 Tucao.txt，你好奇地打开一看，发现是吐槽操作系统实验的，且该文件已经被添加到暂存区了，面对这样的情况，你该如何设置才能使 Tucao.txt 在不从工作区删除的情况下不会被 git commit 指令提交到版本库？

##### 问题解答：
1. git checkout printf.c
2. git reset HEAD printf.c
3. 在工作区编辑（没有的话新建）文件.gitignore，将Tucao.txt加到文件中。

### thinking 2.2
##### 问题描述：
> 思考下面四个描述，你觉得哪些正确，哪些错误，请给出你参考的资料或实验证据。</br>
> 1. 克隆时所有分支均被克隆，但只有 HEAD 指向的分支被检出。</br>
> 2. 克隆出的工作区中执行 git log、 git status、 git checkout、 git commit 等操作不会去访问远程版本库。</br>
> 3. 克隆时只有远程版本库 HEAD 指向的分支被克隆。</br>
> 4. 克隆后工作区的默认分支处于 master 分支。</br>

##### 问题解答：
为了解答这些问题，我特意重新在新目录下使用了克隆，以下实验证据也均是我自己亲自测试过的。

实验证据：
> 14061213@ubuntu:~/learnGit/14061213-lab$ git branch -a</br>
    * master</br>
      remotes/origin/HEAD -> origin/master</br>
      remotes/origin/lab1</br>
      remotes/origin/lab1-result</br>
      remotes/origin/lab2</br>
      remotes/origin/master</br>
      remotes/origin/repair_lab2</br>

1. 正确；</br>
由实验证据可见，确实所有的分支均被克隆，且只有HEAD指向的分支(remotes/origin/master)被检出；
而且git-clone 的 Manual Page 中的 DESCRIPTION 也有这样的话 creates and checks out an initial branch that is forked from the cloned repository’s currently active branch（创造并检出一个初始分支，这个分支是克隆仓库的当前活动分支的叉）。
2. 正确;</br>git log,git status,git checkout,git commit均不会访问远程版本库。</br>
实验证据：在断网的情况下，这些命令依旧可以使用，可见这些命令不会访问远程服务器。
3. 错误；</br>解释及实验证据同回答1。
4. 错误；</br>git-clone 的 Manual Page 中的 DESCRIPTION 也有这样的话 creates and checks out an initial branch that is forked from the cloned repository’s currently active branch（创造并检出一个初始分支，这个分支是克隆仓库的当前活动分支的叉），可见默认的初始分支是远程仓库当前的活动分支的检出，不一定是 master。</p>
我也做了相应的实验来创造反例,工作区的默认分支为tutor：</br>
> vcb@bm  ~/Documents/GitHub/Tutorial (tutor)</br>
> $ git clone https://github.com/YoungForest/Tutorial.git</br>
> vcb@bm  ~/Documents/GitHub/Tutorial (tutor)</br>
> $ git branch -a</br>
> \* tutor</br>
>   remotes/origin/HEAD -> origin/tutor</br>
>   remotes/origin/branch1</br>
>   remotes/origin/tutor</br>


## 2.实验难点图示
![Alt lab1](https://github.com/YoungForest/BUAA_MIPS_OS_DOC/blob/master/lab1难点图示.png)

<https://github.com/YoungForest/BUAA_MIPS_OS_DOC/blob/master/lab1难点图示.png>
## 3.体会与感想
第一次实验总体感觉难度还是比较低的，但学习的新知识比较多。
除了我们操作系统必须要用的知识，为了完成实验，我们还不得不学习：ubuntu的命令行、ssh、git、markdown等一系列工具的使用。在短短的三周里，回头看到自己学到了如此多的知识，才感觉老师及助教的用心良苦。

我也简单看了一下指导书上实验二的部分，感觉难度上比实验一提高了很多，并不是轻松可以完成的，当然我也相信自己在爬过很多坑之后，最终可以完满地完成实验二。

## 4.残留难点
还是不能熟练地使用git的很多命令，对git的理解不是很清楚，在做thinking2.2时花了很多功夫。
