title: "Programming with PTRACE, Part6 - 时间控制"
date: 2014-07-07 16:51:03
categories: Informatics
tags: [Linux,PTRACE,教程]
keywords: [ptrace,time,signal,setrlimit,getrlimit,RLIMIT_CPU,gettimeofday,timeval]
description: "Sixth part of serial of tutorial: Programming with PTRACE"
comments: true
---
# 不同的时间计算方法
程序运行会占用一小段时间（废话），事实上，我们有不止一种方法来表示一个程序运行了多长时间。最直观的应该是“墙上时间”，也就是说，你掐个秒表，看看程序从开始到结束用了多长时间。除此之外，还有“用户态时间”和“内核态时间”，这两个时间都是以CPU实际运算的时间，也就是CPU周期，来计数的。“用户态时间”就是程序在用户态执行的时间，包括程序所引用的库中的代码（比如STL），“内核态时间”就是指程序在内核态执行的时间，一般是各种系统调用（比如各种IO操作）。这两种时间和墙上时间的区别在于，因为CPU其实是在多个程序中快速切换的，所以在运行某个程序的时间里，CPU也处理了属于其他进程的任务，而且CPU切换任务也需要一定的时间（真的很短）。如果处于被调试状态，tracer的运行时间也会被计算在内，这些不属于这个进程的时间片也会被计算在这个进程的“墙上时间”里。所以一般以用户态时间和内核态时间的总和作为进程的运行时间。

在Linux系统里有一个叫`time`的命令可以查看一个命令执行了多长时间。这个命令有两个版本，一个是shell内置的，另一个是独立的可执行文件，可以用`type time`命令查看。虽然可执行版本功能更强一点，但内置的功能足够，这一点区别可以不管。用法是： `time [命令] <参数>`。给个例子：

    time ffmpeg -i sample.mp4 target.mp3
    ...
    5.42s user
    0.10s system
    100% cpu
    5.520 total

<!--more-->
# 动手写个带时限的time
还在对上个PART的`setrlimit`耿耿于怀么？我们现在就来用它！相关的定义位于`sys/resource.h`头文件里。我们这次要用到`RLIMIT_CPU`,这个选项限制进程所能占用的CPU时间，以秒为单位，可以把它理解为用户态时间和内核态时间的和。我们首先要使用`getrlimit`获得当前的限制：

    struct rlimit TimeL;
    getrlimit(RLIMIT_CPU,&TimeL);

`rlimit`结构有两个成员:

- `rlim_cur` 软限制
- `rlim_max` 硬限制

系统一般会用比较平和的方式对待那些达到软限制的进程，比如发个SIGSEGV什么的。而那些达到硬限制的进程会被直接SIGKILL。我们接下来要修改软限制，注意单位是秒。

    TimeL.rlim_cur=Timeout;

以上工作都要在`fork()`之前完成，之后要在*子进程*里应用这个限制（没错就是exec那里）

    setrlimit(RLIMIT_CPU,&TimeL);

这样，如果子进程超过软限制，系统就会发送`SIGXCPU`信号给子进程。当然，因为ptrace的原因，信号会被先发送给父进程，这样就可以用part3里介绍的方法进行处理。这样子进程是要清蒸还是油炸就都由父进程决定了。
当然，我们还有别的方法获取时间信息。一是用`gettimeofday()`函数配合`timeval`结构，可以获得当前时间，精确到微秒（百万分之一秒）。在程序开始时调用下，结束时调用下，相减即可得到墙上时间。另一种方法是利用`wait4`里的`ru`参数，它其实是个`rusage`结构,成员[见此](http://man7.org/linux/man-pages/man2/getrusage.2.html)。其中的`ru_utime`和`ru_stime`成员是`timeval`结构，分别记录了用户态时间和内核态时间，同样精确到微秒。

# 程序睡着了
`RLIMIT_CPU`大多数情况下都能正常工作，配合`timeval`结构甚至能进一步提高精度。但是有两个例外（如果有更多请务必告诉我）：

1. 程序主动调用`sleep()`
2. 交互状态下`scanf()`一类的函数等待键盘输入

在这两种情况下：进程不占用CPU时间，`RLIMIT_CPU`管不着；没有系统调用，`wait4()`不返回。为了能够在这种情况下依然能够限制时间，我想出了两种方法。一是限制和`sleep()`相关的系统调用，二是父进程设置ALARM。我在这里讲一下第二种方法。
Linux提供了一个`alarm()`函数，可以在指定的秒数**(墙上时间)**后给这个进程本身发送`SIGALRM`信号。而且，我们可以给信号绑定一个处理函数（就是当信号到达时调用的函数），在这个处理函数里，可以用`kill`命令给子进程发送信号（比如`SIGUSR1`），这样就能使父进程里的`wait4()`返回，就可以控制子进程了。以下是一个简要指导：
首先我们需要一个信号处理函数,记得把pid改成全局变量：

    void AlarmIn(int sig){
        if(sig==SIGALRM)
        kill(pid,SIGUSR1);
    }

然后在子程序开始执行的时候绑定信号并设置Alarm，我在这设置超时一秒:

    signal(SIGALRM,AlarmIn);
    alarm(1);

然后请根据part3所讲的内容在while循环里正确处理`SIGUSR1`。最后记得取消Alarm，如果没超时的话：

    alarm(0);

# 完整代码？
表示完整代码太长了，放这儿太不美观，<del>我会稍后贴到gist上去。</del>代码被幽幽子吃掉了大家自己写把。

# 拓展阅读
- [Wikipedia - 数量级 (时间)](http://zh.wikipedia.org/wiki/%E6%95%B0%E9%87%8F%E7%BA%A7_%28%E6%97%B6%E9%97%B4%29) 我把这个链接放这儿是因为老有人把微秒缩写成`ms`然后和毫秒搞混
- [协同式多任务与抢占式多任务](http://blog.csdn.net/xwdok/article/details/542109)
- [Linux时间管理](http://www.cnblogs.com/iceocean/articles/1650929.html)
- [浅谈时间函数gettimeofday的成本](http://russelltao.iteye.com/blog/1405353)
