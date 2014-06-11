title: Programming with PTRACE, Part4 - 系统调用进阶
date: 2014-05-26 10:00:31
categories: Informatics
tags: [Linux,PTRACE,教程]
keywords: [ptrace,syscall,arguments,kernel,ABI]
description: Fourth part of serial of tutorial: Programming with PTRACE
comments: true
---
这个part是[Part2](/2014/04/programming-with-ptrace-part2/)的延续，所以我强烈建议你弄明白Part2中的内容后再来看本part。那么进入正题，我将在这个部分讲解系统调用的参数传递顺序以及如何利用ptrace系统调用获得用户空间的数据。

## 参数与寄存器
我在Part2中提到过，系统调用的参数是以一定顺序保存在寄存器里的，那么这个顺序是什么呢？在`man 2 syscall`中有两张表格解释了这个问题，你也可以在[这里](http://man7.org/linux/man-pages/man2/syscall.2.html)看到，就在"Architecture calling conventions"下面。我知道很多人很懒，所以我就把这两张表格复制过来了。

| arch/ABI | instruction         |syscall #|retval|Notes                 |
| -------- | --------------------|---------|------|----------------------|
| arm/OABI | swi NR              |-        |a1    |NR is syscall #       |
| arm/EABI | swi 0x0             |r7       |r0    |                      |
| blackfin | excpt 0x0           |P0       |R0    |                      |
| i386     | int $0x80           |eax      |eax   |                      |
| ia64     | break 0x100000      |r15      |r10/r8|bool error/errno value|
| parisc   | ble 0x100(%sr2, %r0)|r20      |r28   |                      |
| s390     | svc 0               |r1       |r2    |See below             |
| s390x    | svc 0               |r1       |r2    |See below             |
| sparc/32 | t 0x10              |g1       |o0    |                      |
| sparc/64 | t 0x6d              |g1       |o0    |                      |
| x86_64   | syscall             |rax      |rax   |                      |

| arch/ABI | arg1 | arg2 | arg3 | arg4 | arg5 | arg6 | arg7 |
| -------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| arm/OABI | a1   | a2   | a3   | a4   | v1   | v2   | v3   |
| arm/EABI | r0   | r1   | r2   | r3   | r4   | r5   | r6   |
| blackfin | R0   | R1   | R2   | R3   | R4   | R5   | -    |
| i386     | ebx  | ecx  | edx  | esi  | edi  | ebp  | -    |
| ia64     | out0 | out1 | out2 | out3 | out4 | out5 | -    |
| parisc   | r26  | r25  | r24  | r23  | r22  | r21  | -    |
| s390     | r2   | r3   | r4   | r5   | r6   | r7   | -    |
| s390x    | r2   | r3   | r4   | r5   | r6   | r7   | -    |
| sparc/32 | o0   | o1   | o2   | o3   | o4   | o5   | -    |
| sparc/64 | o0   | o1   | o2   | o3   | o4   | o5   | -    |
| x86_64   | rdi  | rsi  | rdx  | r10  | r8   | r9   | -    |

<!--more-->
表格内容虽多，但其实我们关心的只有i386和x86_64（32位和64位）一共4行（因为有两张表格嘛）。精简提炼下，一共就两句话

> 对于32位系统，系统调用号存放在EAX寄存器，参数依次放入EBX、ECX、EDX、ESI ... 返回值位于EAX寄存器
  对于64位系统，系统调用号存放在RAX寄存器，参数依次放入RDI、RSI、RDX、R10 ... 返回值位于RAX寄存器

以64位系统下的`write()`调用为例:

    ssize_t write(int fd, const void *buf, size_t count);
    
那么RAX是1(write的调用号)，RDI一般为1(stdout),RSI存储着指向用户空间中将要被输出的字符串的地址，RDX自然就是字符串长度啦。

## 获取那个字符串
理论讲完了，进入实战。这次我们拿`open()`系统调用开刀，一是因为监视程序打开了什么文件比得知输出了什么更常用，二是因为传递给`open()`的字符串没有长度信息，只能自己通过`\0`判断，更有挑战性。我们这次要使用ptrace的一个新功能`PTRACE_PEEKTEXT`,其实还有另外一个叫做`PTRACE_PEEKDATA`的，不过根据man手册的描述，这两个的功能是一样的。它的用法是这样的

    data = ptrace(PTRACE_PEEKTEXT,pid,addr,0);
    
即从子进程（由pid标识）的addr内存地址处取出对应字长(64位为8字节，32位4字节)的数据，做为返回值。也就是说，读取一次能得到八个字符。现在如果我们要取得从`base_addr`地址开始的一个字符串，那么我们只要8个字节8个字节读取，直到碰到`\0`为止。把这个功能写成函数就是这样：（32位系统不要忘记改那个define）

    #define WORD_LEN 8
    void peek_str(pid_t pid,long base_addr,char target[]){
      union{
        long word;
        char str[WORD_LEN];
      } data;/*利用union把WORD_LEN字节的整数变为字符数组*/
      long offset=0;
      int done=0,i;
      target[0]='\0';
      while(!done){/*循环读取*/
        data.word=ptrace(PTRACE_PEEKTEXT,pid,base_addr+offset,0);
        strncat(target,data.str,WORD_LEN);/*追加至多WORD_LEN个字符*/
        for(i=0;i<WORD_LEN;i++)/*检查是否有'\0'*/
          if(data.str[i]=='\0')
            done=1;
        offset+=WORD_LEN;/*准备读取下一个WORD_LEN字节*/
      }
    }
    
主程序的大`while()`循环里的代码是这样的（我已经设置了`PTRACE_O_TRACESYSGOOD`标记）：

      wait4(pid,&sta,0,&ru);
      if(WIFEXITED(sta)){printf("Exited with code %d",WEXITSTATUS(sta));break;}
      if(WIFSIGNALED(sta)){printf("Terminated by signal: %s",strsignal(WTERMSIG(sta)));break;}
      int sig_no;if(WIFSTOPPED(sta))sig_no=WSTOPSIG(sta);else{puts("Unknown Status");break;} 
      if(sig_no!=(SIGTRAP|0x80)){ptrace(PTRACE_SYSCALL,pid,0,sig_no);continue;}
      if (intocall){
        struct user_regs_struct reg;
        ptrace(PTRACE_GETREGS,pid,0,&reg);
        if (reg.orig_rax==SYS_open){
          char file[255];
          peek_str(pid,reg.rdi,file);
          printf("open() opened: %s\n",file);
        }      
      }
      intocall^=1;
      ptrace(PTRACE_SYSCALL,pid,0,0);

在这里我使用的`PTRACE_GETREGS`和`user_regs_struct`结构来一次性获得所有寄存器的值，该结构定义于`sys/user.h`头文件中。另外，我还使用了`SYS_open`来判断系统调用号，避免了Magic Number。`SYS_*`宏定义于`sys/syscall.h`头文件中。传递RDI寄存器也很容易理解，查询`man 2 open`可知open系统调用的路径是第一个参数。现在，重新编译你的`target`，不要加`-static`,然后运行，你应该能看到类似这样的输出。

    Parent started
    Child PiD == 4717
    Child exec...
    Child execve() returned with 0
    open() opened: /usr/lib/tls/x86_64/libc.so.6
    open() opened: /usr/lib/tls/libc.so.6
    open() opened: /usr/lib/x86_64/libc.so.6
    open() opened: /usr/lib/libc.so.6
    Hello World!
    Exited with code 0

可以很明显的看到程序搜索动态链接库的过程。
如果你觉得这还不够过瘾，那么你可以看[Playing with ptrace, part1](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/),后面提供了一个配合使用`PTRACE_PEEKTEXT`和`PTRACE_POKETEXT`来将`write`输出的字符串反转的例子

## 其他资料
* [调试器是怎样工作的](http://godorz.info/2011/02/how-debuggers-work-part-1)
* [Process Tracing Using Ptrace](http://linuxgazette.net/issue81/sandeep.html)

## 不是后记的后记
不知不觉已经写到Part4了，期间一边查资料一边写代码做验证一边写这篇文章，又发现了好多好多之前遗漏的信息和好文章。同时深深感觉自己真是个蒟蒻，好多东西觉得很重要，想讲却心有余而力不足，而且越来越像是在翻译man手册了……我是不是一开始就应该去翻译手册而不是写这系列文章呢？（笑）
下个part开始，估计就要暂时和ptrce说再见，然后和内存管理开始较劲了。
