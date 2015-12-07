title: "Programming with PTRACE, Part2 - 系统调用入门"
date: 2014-04-20 20:00:19
categories: Informatics
tags: [Linux,PTRACE,教程]
keywords: [ptrace,ABI,syscall,register,signal]
description: "Second part of serial of tutorial: Programming with PTRACE"
comments: true
---
在这部分，我会介绍如何使用ptrace监控子进程的系统调用。我先将完整代码列在开头，你现在十有八九看不懂它，但我希望你在看完这篇文章后能彻底理解这段代码。（这段代码在64位系统上有效，32位系统请参照最后`给32位系统的Tip`手动修改源代码）
``` c demo4.c
#include <stdio.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <sys/resource.h>
#include <sys/reg.h>
int main(){
  puts("Parent started");
  pid_t pid;
  pid=fork();
  if (pid<0){
    puts("fork() failed");
    return(-1);
  }
  if (pid==0){
    ptrace(PTRACE_TRACEME,0,0,0);
    puts("Child sleeping...");
    sleep(1);
    puts("Child exec...");
    execlp("./target","target",NULL);
  }else{
    printf("Child PiD == %d\n",pid);
    int sta=0;
    struct rusage ru;
    wait4(pid,&sta,0,&ru);
    long rax_rt=ptrace(PTRACE_PEEKUSER,pid,8*RAX,0);
    printf("Child execve() returned with %ld\n",rax_rt);
    ptrace(PTRACE_SYSCALL,pid,0,0);
    int intocall=1;
    while(1){
      wait4(pid,&sta,0,&ru);
      if (WIFEXITED(sta)){
        puts("Child Exited");
        break;
      }
      long _ORIG_RAX=ptrace(PTRACE_PEEKUSER,pid,8*ORIG_RAX,0);
      long _RAX=ptrace(PTRACE_PEEKUSER,pid,8*RAX,0);
      if (intocall){
        printf("Entering SYSCALL %ld .... ",_ORIG_RAX);
        intocall=0;
      }else{
        printf("Exited with %ld\n",_RAX);
        intocall=1;
      }
      ptrace(PTRACE_SYSCALL,pid,0,0);
    }
  }
}
```
<!--more-->
## 运行我们的程序
当然，如果你试图直接编译并运行上面这段程序肯定是失败的，因为你缺少一个用于被执行的“target”（就是execlp里的那个）。在这里，我们的第一个target是最经典的“Hello World!”程序：
``` c target.c
#include <stdio.h>
int main()
{
  puts("Hello World!");
  return 0;
}
```
然后，我建议你静态方式进行链接：

    gcc -static target.c -o target

注意到我这里使用了`-static`参数，它的作用是将c运行时库静态链接入可执行文件中。你可以比较一下用两种方式编译的文件大小（几K和几百K的区别）。虽然用动态链接也可以，但是会和我之后的输出有一点点出入（因为动态链接文件需要根据环境变量搜索动态库）。现在把`target`和`demo4.c`放在同一目录下，然后

    gcc demo4.c -o demo4 && ./demo4

如果运行正确，你应该看到类似如下的输出

    Parent started
    Child PiD == 9702
    Child sleeping...
    Child exec...
    Child execve() returned with 0
    Entering SYSCALL 63 .... Exited with 0
    Entering SYSCALL 12 .... Exited with 31248384
    Entering SYSCALL 12 .... Exited with 31252928
    Entering SYSCALL 158 .... Exited with 0
    Entering SYSCALL 89 .... Exited with 55
    Entering SYSCALL 12 .... Exited with 31388096
    Entering SYSCALL 12 .... Exited with 31391744
    Entering SYSCALL 5 .... Exited with 0
    Entering SYSCALL 9 .... Exited with 140378408579072
    Hello World!
    Entering SYSCALL 1 .... Exited with 13
    Entering SYSCALL 231 .... Child Exited

## 深入这段代码
我先介绍一下各个头文件的用途：

- `stdio.h`：（如果你不知道这个文件是干嘛的请重学C语言）
- `unistd.h`：提供`fork()`、`pid_t`、`execlp()`、`sleep()`等
- `sys/ptrace.h`：提供ptrace相关函数和宏定义
- `sys/wait.h`：提供`wait4()`和`WIFEXITED`宏
- `sys/resource.h`：提供`rusage`结构定义
- `sys/reg.h`：提供寄存器系列宏定义（`ORIG_RAX`等）

看到代码的第15行，一个巨大的`if...else...`将代码清晰地分成了父子进程两个部分，16行的`ptrace(PTRACE_TRACEME,0,0,0);`首先吸引了我们的注意力。（为什么有一种在写春游作文的感觉）这个调用使得子进程被标记为`TRACED`并且使系统内核在子进程调用exec族函数*之后*通知父进程，这也是为什么17到19行的系统调用没有被追踪到的原因。
再看父进程部分，由于系统调用是一个从用户态到内核态再到用户态的过程，所以每进行一次系统调用都会触发两次`syscall_stop`,分别是进入时的`syscall_enter_stop`和离开内核时的`syscall_exit_stop`。这种子进程的状态的变化可以在父进程中使用`wait()`、`waitpid()`、`wait4()`等一票函数完成（还记得part1课后阅读中的僵尸进程么？）。值得注意的是，第一次的状态变化是由`execve()`调用返回导致的`syscall_exit_stop`，所以我在25到29行单独做了处理。我喜欢使用`wait4()`的原因是它还可以获得子进程的当前资源占用情况（就是那个`rusage`结构），这对于了解进程资源使用情况非常有用，只不过现在还用不到（我应该会在之后专门开几个part来讲系统资源的限制），所以我们只要关注那个`sta`就可以了。
注意下面那个大的`while`循环，在每次循环的开头等待，一旦`wait4()`返回，子进程就已经进入了暂停的状态（其实是内核给子进程发送了`SIGTRAP`信号，但因为子进程处于`TRACED`状态，所以这个信号被转交给了父进程，使父进程的`wait4()`返回，但这也意味着由其他方式引起的信号(比如`kill`命令)也会引起wait4的返回）。接着在32行使用`WIFEXITED`宏加上wait4收集的状态信息`sta`判断子进程是否已经退出，如果已退出，那么父进程也从循环中退出。当然这是一个非常粗糙的处理方式，更具完整的处理流程将在之后的part里介绍。接着，我们就可以使用各种各样的命令来调戏子进程了，这里我们只是简单的取得系统调用号和返回值。最后，在45行，让子进程继续执行，并要求子进程在下一个系统调用（进入或返回）停住，然后父进程开始等待下一次的`syscall_stop`。因为一次系统调用会导致两次`syscall_stop`，所以我使用变量`intocall`来分辨，并且在38到44行打印出不同的提示信息。顺带提一下，在输出中，`Hello World!`应该输出在`Entering SYSCALL 1`和`Exited with 13`中间，但因为缓冲区刷新的问题所以被输出到了前面。

终于到最激动人心的部分了！36、37两行代码是最重要的部分，可以看出，他们做的工作是差不多的，都是从子进程的内存空间中取一些数据。为了解释好这两行，我要讲一讲系统调用的调用过程。系统调用和普通的函数调用差不多，函数调用是将参数以约定好的顺序压入栈中，而系统调用则发生了一个类似上下文切换的过程：程序将需要调用的系统调用的调用号以及参数存入寄存器中，然后将所有寄存器存入栈中，进入内核态后，内核从栈中取得调用号和调用参数，并将返回值写入栈中对应寄存器的位置，最后还原寄存器的值并返回用户态，于是返回值就这样被“还原”到了寄存器里。在x86-64平台上，负责传递系统调用号和返回值的都是`RAX`寄存器，也就是说返回值会覆盖调用号，为了在系统调用返回时也能知道调用号，`RAX`寄存器在保存时被入栈两遍，一个是用于保存返回值的`RAX`，另一个是负责保存调用号的`ORIG_RAX`。现在，我们要获得寄存器的值，只要访问栈中的对应位置就可以了。而系统内核又会在系统调用时将栈中的这些信息复制一遍到一个叫做`u-area`(USER Area)的内存区域。在`sys/reg.h`头文件中定义了各寄存器保存时在`u-area`中的顺序，乘以每个寄存器的长度（64位系统自然就是8了嘛~~）就得到了我们所要访问的字节偏移量，`PTRACE_PEEKUSER`要求ptrace从指定偏移取出一个寄存器长度的数据（也就是8字节）作为返回值，于是`ptrace(PTRACE_PEEKUSER,pid,8*ORIG_RAX,0)`就能获得系统调用号啦！
## 给32位系统的Tip
要让程序通过编译，需要做两个改动：

1. 在`int main()`之前加入这两个预处理命令：

        #define RAX EAX
        #define ORIG_RAX ORIG_EAX
2. 把26、36、37行ptrace第三个参数中的8全部改成4

这是因为32位系统的寄存器长度是4字节，而且负责传递系统调用号和返回值的是`EAX`寄存器。
## 其他资料
* [strace](http://sourceforge.net/projects/strace/)是一个用于监视并输出某程序系统调用情况的工具，比如`strace ./target`
* [Linux信号代码](http://man7.org/linux/man-pages/man7/signal.7.html)：也可以通过查man手册得到`man 7 signal`
* [Playing with ptrace](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/)：Pradeep Padala 的关于ptrace的文章。
* [x86_64 系统调用号列表](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64)：也可以在`asm/unistd_64.h`头文件中找到，`asm/unistd_32.h`就是32位的
* [Process and Process Memory](http://www.hep.wisc.edu/~pinghc/Process_Memory.htm)
* **有问题或意见请务必留言啊啊啊啊~**没人留言都不知道评论系统是否工作正常
