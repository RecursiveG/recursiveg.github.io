title: "Programming with PTRACE, Part1 - 起步"
date: 2014-04-17 21:16:29
categories: Informatics
tags: [Linux,PTRACE,教程]
keywords: [ptrace,fork,execve,linux process,]
description: "First part of serial of tutorial: Programming with PTRACE"
comments: true
---
## 前言
本人作为一个信息学竞赛的参与者，在很久之前曾经试图自己写过一个Online Judge系统（允许用户上传源代码并在服务器上编译运行），考虑到安全因素，必须要对程序的行为进行限制，因此对ptrace进行了一番研究。网上有一份关于ptrace的很好的教程（[Playing with ptrace](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/)）,但是时间有点久了，而且没有涉及64位操作系统。因此，我决定写这份教程，基于64位Linux，尽力介绍一些新加入的功能，同时兼顾一下32位系统。另外，由于一开始的目的是“对程序的行为进行*限制*”，所以不会涉及到诸如设置断点之类的内容，相反，可能会涉及到其他关于系统资源管理的内容。
`ptrace()`是一个由Linux内核提供的系统调用。它允许一个用户态进程检查、修改另一个进程的内存和寄存器。这种技术被广泛用于`gdb`等调试器中。尽管这系列文章的标题叫做“Programming with PTRACE”，但在第一部分中，我将着重介绍Linux的进程和相关的几个重要函数。
## fork(), vfork() 与 clone()
在Linux中，每一个进程都有一个唯一的编号，被称作`pid`(Process ID)。在Linux中，进程不能凭空产生（`init`进程是个例外），只能从一个已有进程衍生出来。原来的进程被称做父进程，衍生出来的进程叫子进程。一个系统中所有进程以父子关系相连接，形成一棵树，这棵“树”的树根就是`init`进程，它是在系统启动时被直接启动的，因此它没有父进程。并且系统中所有其他进程都直接或间接地是它的子进程。在Linux系统中，实现“把一个进程变成两个”这一功能的有三个系统调用，即`fork()`、`vfork()`和`clone()`。

`fork()`的工作流程的确和叉子有几分相似之处，它将当前进程所有数据复制一份，产生一个和父进程一模一样的子进程。并在两个进程中返回不同的返回值。比如这段代码：
``` c demo1.c
#include <stdio.h>
#include <unistd.h>
int main(int argc,char *argv[]){
    int return_val;
    puts("Program started.");
    return_val=fork();
    printf("fork() returned %d\n",return_val);
    return 0;
}
```
将会输出

    Program started.
    fork() returned 5768
    fork() returned 0
很明显地可以看到，`puts()`只被调用了一次而`printf()`被调用了两次，这说明在`fork()`前的一个进程变成了两个，而且`fork()`在两个进程中有不同的返回值（这就是“调用一次，返回两次”的来历）。`fork()`会返回0给子进程，返回子进程的pid给父进程，因此，我们很容易判断出`fork() returned 0`是由子进程打印的。在实际应用中，也通过`if`语句判断返回值的方法来决定执行不同的代码：

    int pid=fork();
    if (pid==0){
      //子进程的工作
    }else{
      //父进程的工作
    }

一般来说，子进程的工作就是调用`exec`族函数，启动另一个程序(把自己替换掉)。如果子进程还在执行而父进程已结束，那么它就成为“孤儿”进程，成为`init`进程的子进程。另外，请不要纠结那个`if`判断带来的性能损失，Linux的内核开发者都不纠结，你纠结什么呢？
<!--more-->

`vfork()`的存在是一个历史遗留问题，在很久很久以前，`fork()`调用是没有[CoW](http://en.wikipedia.org/wiki/Copy_on_write)机制的，如果fork出的一个子进程又立即调用了`exec`族函数，那么辛辛苦苦拷贝出来的内存又立马被扔进了废纸篓里（这个比喻可能不太恰当，毕竟被从内存里抹去的数据是捡不回来的）。Linux的开发者当然不会允许效率如此低下的事情发生，于是他们创造出了`vfork()`。它和`fork()`最大的差别在于，vfork出的子进程，在执行`exec`族函数前和父进程**共享同一块内存**。也就是说，子进程对内存的修改也会体现在父进程上。只有当子进程执行了`exec`族函数，它才真正拥有一块属于自己的内存。这样就节省了`fork()`中那个无意义的内存拷贝。现在因为有了CoW，`fork()`和`vfork()`已经几乎没有性能差异了。
``` c demo2.c
#include <stdio.h>
#include <unistd.h>
int main(int argc,char *argv[]){
    int pid,x=1;
    printf("X=%d\n",x);
    pid=vfork();
    if (pid==0){
        x+=1;
        printf("Child-X=%d\n",x);
    }else{
        x+=1;
        printf("Parent-X=%d\n",x);
    }
    _exit(0);
}
```
这段代码输出，而且一定输出

    X=1
    Child-X=2
    Parent-X=3
很好地说明了内存的共享，如果换成`fork()`，那么父子进程就都输出X=2了。
也许有人会问，为什么不可能是父进程先输出呢？这涉及到`vfork()`的另一个特点。如果使用`vfork()`创建进程，那么在子进程使用`exec`族函数或是`_exit()`(这就是我为什么不用`return 0`的原因，但没有详细研究过原因，求大神指教)之前，父进程会始终等待vfork返回。比如以下代码：
``` c demo3.c
#include <stdio.h>
#include <unistd.h>
int main(int argc,char *argv[]){
    int pid;
    pid=vfork();
    if (pid==0){
        puts("Child Sleeping...");
        sleep(3);
        puts("Child Exit.");
    }else{
        puts("Parent Exit.");
    }
    _exit(0);
}
```
输出

    Child Sleeping...
    //这里等了3秒
    Child Exit.
    Parent Exit.
而改成`fork()`后输出

    Parent Exit.
    Child Sleeping...
    ~$
    //这里等了3秒
    Child Exit.
可以明显看出两者差别。(给Windows用户的Tip: 那个`~$`是Linux终端的提示符，类似cmd)

`clone()`函数提供了更多的控制选项，可自由决定要执行哪个代码片段甚至是哪些内存共享，哪些内存要复制。但我没怎么用过，不敢乱说，有兴趣的读者可以自行实验。

## 令人困惑的exec族函数
我在这篇文章之前的部分N次提到了一个叫`exec族函数`的东西，如果我们man手册里查找(`man 3 exec`)，我们会得到一大堆函数（是不是开始感到困惑了？）：

       int execl  (const char *path, const char *arg, ...);
       int execlp (const char *file, const char *arg, ...);
       int execle (const char *path, const char *arg, ..., char * const envp[]);
       int execv  (const char *path, char *const argv[]);
       int execve (const char *path, char *const argv[], char *const envp[]);
       int execvp (const char *file, char *const argv[]);
       int execvpe(const char *file, char *const argv[], char *const envp[]);

`exec族函数`就是这一“族”函数，全部以exec打头，他们都是对系统调用`execve()`的包装。他们的作用就是把某个进程（通常是fork出来的子进程）从里到外，完完整整，包括代码、堆栈，全部换成另一个程序，然后从头开始运行。它们的调用效果是一样的，区别在于调用方式。总的来说，大致的参数顺序是这样的：`exec*(可执行文件路径，程序参数表[,环境变量表])`，其中环境变量表是可选的。
去掉打头的exec，带`l`（代表list）的函数使用了一种比较接近人类方法来表示程序参数表，即以`NULL`作为结尾（man手册推荐使用`(char *)0`）的变参列表；而带`v`(代表vector)的则使用一个字符串数组来表示程序参数表，就像`int main(int argc,char *argv[])`里的`argv`一样。
如果结尾带`e`（environment），则该函数接受一个字符串数组表示的环境变量表；反之，则会默认传递所有当前环境变量。如果带有`p`，那么你就不必在第一个参数中列出完整路径，系统会自动检查当前目录和`PATH`环境变量（如果你非要手贱加个路径分割符进去，那么系统就会把它当成完整路径）。
值得一提的是，不管你使用那种方法表示程序参数表，第0个参数（C的数组下标从0开始，记得么？）都应当和可执行文件路径保持一致，虽然不一致依然可以正确运行，但有可能出现奇奇怪怪的问题。（博主继续偷懒，欢迎各位读者当小白鼠自行实验）。如果你已经混乱了，或是直接跳过了上面的一大堆说明直接到了这，那么我推荐你直接使用`execlp()`函数，比如说，你要运行一个叫`foo`的程序：

    execlp("foo","foo",NULL);
    
或是列举出根目录下所有文件：

    execlp("ls","ls","/",NULL);

## 继续之前的其他一些准备
从本系列的下一篇开始，我将要开始讨论`ptraec()`这一强大的工具。但是，如果你有一下现象之一的，我建议你*不要*继续阅读并且从头学习有关`*nix`系列系统的知识：
1. 基本看不懂这篇文章的
2. 不会C语言的
3. 狂热的Windows爱好者
4. 不会使用[Google](https://www.google.com)的
5. 没有IDE就不会编译程序的
6. 没有听说过`寄存器`，`堆栈`的

另外，`ptrace()`相当接近系统底层，对内核版本，系统构架，指令长度，库头文件等有相当大的依赖性，如果你还在使用2.x系列的内核，你可能在之后遇到问题，因为一些功能在新版本内核才被加入。我在这里列出我的编程环境：

- 系统: ArchLinux x86_64
- 内核: Linux 3.14.1
- glibc 2.19
- gcc 4.8.2

另外，这里有更多关于进程的文章

- [Linux进程基础](http://www.cnblogs.com/vamei/archive/2012/09/20/2694466.html) （我觉得其中关于食谱的那个比喻不太恰当，也许程序和进程的关系更像类和类的实例的关系？）
- [Linux 进程管理剖析](http://www.ibm.com/developerworks/cn/linux/l-linux-process-management/) (IBM developerWorks是我超喜欢的一个网站，有相当多高质量的文章)
- [Linux 的僵尸(zombie)进程](http://coolshell.cn/articles/656.html)







