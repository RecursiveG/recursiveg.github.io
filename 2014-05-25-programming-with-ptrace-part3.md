title: "Programming with PTRACE, Part3 - 进程的终止与信号"
date: 2014-05-25 18:26:20
categories: Informatics
tags: [Linux,PTRACE,教程]
keywords: [ptrace,signal,return-value]
description: "Third part of serial of tutorial: Programming with PTRACE"
comments: true
---
在Part2中，我们粗略了解了如何使用`ptrace`获得系统调用信息，即在一个大循环里不断获取程序信息，如果程序退出则停止循环。当然，那个判断异常简陋，几乎无法处理任何特殊情况。我将在本Part中详细解说各种异常情况的处理，同时讲解各种信号相关的问题。

## 一些重要的宏
在使用`wait4`后，程序的信息被存储在`sta`变量中，这些信息被存储在这个整数的不同二进制位上，这儿有一系列宏用于帮我们提取这些信息。以下信息是我对`man 3 wait`中相关部分的翻译,同时参考了[这个](http://www.manualpages.de/OpenBSD/OpenBSD-5.0/man2/WIFEXITED.2.html)页面

    WIFEXITED   如果进程正常退出，返回一个非0值(通常是进程调用了`exit()`或是`_exit()`)
    WIFSIGNALED 如果进程由于一个未被捕获的信号而被终止，返回一个非0值
    WIFSTOPPED  当进程被停止(非终止)时，返回一个非0值(通常发生在当进程处于`traced`状态时)

    WEXITSTATUS 当`WIFEXITED`为非0值，获得进程`main()`函数的返回值
    WTERMSIG    如果`WIFSIGNALED`为非0值，获得引起进程终止的信号代码
    WSTOPSIG    如果`WIFSTOPPED`为非0值，获得引起进程停止的信号代码

除了这六个，还有`WIFCONTINUED`和`WCOREDUMP`两个宏，不过我们用不到，我也没仔细研究，就不说了。
当进程自行终止时，`WIFEXITED`即为`true`，配套使用`WEXITSTATUS`获得返回值，不做过多解释。当子进程进行系统调用时，`WIFSTOPPED`为`true`,同时`WSTOPSIG`等于`SIGTRAP`(信号代码为7),我们可以用这种方法区分`syscall-stop`和`signal-delivery-stop`。当有一个外部信号要发送给子进程，这个信号会先到达父进程，使`WIFSTOPPED`为`true`，同时`WSTOPSIG`等于该信号的信号代码。父进程可以选择将这个信号继续传递或是不传递，甚至传递另一个信号给子进程。一旦信号真正到达子进程，就进入子进程自己的处理流程或是系统默认动作，可能触发`WIFSIGNALED`，比如`SIGINT`。
在所有信号中，`SIGKILL`是一个例外，它不会经过父进程引发`WIFSTOPPED`，而是直接传递到子进程，引发`WIFSIGNALED`。
<!--more-->
## 信号的传递与修改
之前提到，父进程需要将信号传递给子进程，这是由`ptrace(PTRACE_SYSCALL,pid,0,0)`的第四个参数决定的。如果为0,就不传递信号，否则传递对应代码的信号，比如`ptrace(PTRACE_SYSCALL,pid,0,9)`就将信号9(SIGKILL)传递给了子进程。
修改信号简直信手拈来，传一个你想要传的信号即可。

## strsignal()和代码
`strsignal()`接受一个整数参数，返回`const char*`，用于把信号代码变为对应的、人类可读的字符串描述，定义于`string.h`。下面给出判断程序退出的代码：
```c
wait4(pid,&sta,0,&ru);
if(WIFEXITED(sta)){printf("Exited with code %d",WEXITSTATUS(sta));break;}
if(WIFSIGNALED(sta)){printf("Terminated by signal: %s",strsignal(WTERMSIG(sta)));break;}
int sig_no;
if(WIFSTOPPED(sta))sig_no=WSTOPSIG(sta);
if(sig_no==SIGTRAP)sig_no=0;
......
ptrace(PTRACE_SYSCALL,pid,0,sig_no);
```
## 来自ptrace的高级选项
你也许会纠结，如果外部传递了一个`SIGTRAP`信号，那么如何分辨呢？答案是使用`PTRACE_SETOPTIONS`设置`PTRACE_O_TRACESYSGOOD`标记，即在while之前，第一个wait之后，第一个`PTRACE_SYSCALL`之前，使用`ptrace(PTRACE_SETOPTIONS,pid,0,PTRACE_O_TRACESYSGOOD)`。这会使得`syscall-stop`导致的`WSTOPSIG`从`SIGTRAP`变为`SIGTRAP|0x80`，而普通的来自外部的`SIGTRAP`依然是`SIGTRAP`。
