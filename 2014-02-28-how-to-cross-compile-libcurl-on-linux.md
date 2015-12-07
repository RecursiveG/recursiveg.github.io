title: "在Linux下使用MinGW静态交叉编译带有zlib的libcurl"
date: 2014-02-28 21:17:41
categories: Informatics
tags: [Linux,MinGW,交叉编译]
keywords: [Cross-Compile,Static Library,zlib,libcurl,MinGW]
description: "A short memo on how to cross-compile libcurl statically on linux."
comments: true
---
[libcurl](http://curl.haxx.se/)是一个跨平台的、易用的、强大的网络库。在大部分Linux发行版中都有编译好的二进制包可供使用，Mac系统更是将其作为了一个核心部件。但是在Windows平台上却需要手工编译，更不必说一些有特殊洁癖的人（比如说我）还特别讨厌多出来几个DLL,非要静态链接不可。本文作为我两个晚上折腾经历的一个小小总结，讲解如何在Linux下使用MinGW编译给Windows使用的libcurl静态库。

### STEP1 安装MinGW编译器
这步我不打算多说，大部分Linux发行版的仓库应该都有，以我的ArchLinux为例，执行：

    ~# pacman -S mingw-w64

即可。如果你不需要交叉编译，要在Windows上直接编译，请自行去SourceForge上下载Windows版本。不要担心那个`w64`是不是64位版本，它既可以编译32位又可以编译64位程序。还是以我的版本为例:

    ~# pacman -Ql mingw-w64-gcc| grep '/usr/bin/.*gcc$'
    mingw-w64-gcc /usr/bin/i686-w64-mingw32-gcc
    mingw-w64-gcc /usr/bin/x86_64-w64-mingw32-gcc

可以看到有两个gcc,用`i686-w64-mingw32-gcc`编译出来的程序就是32位的，而`x86_64-w64-mingw32-gcc`编译出来的就是64位的。现在，随便写个Hello World（你可以用我的[Hello World代码](http://www.devinprogress.org/2012/12/helloworld/) ^_^），然后编译试试：

    i686-w64-mingw32-gcc hello_world.c -o hello_world.exe

把它拿到虚拟机或扔进Wine里，如果能正常运行，那么恭喜你，第一步完成了。
<!--more-->
### STEP2 下载源码
很简单的步骤，如果自己搞不定的建议直接右上角。

- [LibCurl](http://curl.haxx.se/download.html):最上面的Source Archives
- [zLib](http://www.zlib.net/):请下Source Code
- [OpenSSL](https://www.openssl.org/source/):可选，如果没有必要就不要编译，会极大地增加文件体积

把`curl-7.35.0`和`zlib-1.2.8`(可能还有`openssl-1.0.1f`)这几个文件夹放在同一个目录下，然后进行下一步。

### STEP3 编译源码
先打开`zlib/win32`文件夹下的`Makefile.gcc`文件,把`PREFIX = `这行改成STEP1里的gcc前缀，对于我来说就是`PREFIX = i686-w64-mingw32-`。把这个文件拷贝到`zlib`文件夹下，然后在`zlib`文件夹下`make -f Makefile.gcc`，你就应该能看到`libz.a`这个文件了。

如果你要编译OpenSSL,那么就去openssl文件夹下

    $ ./Configure no-shared --cross-compile-prefix=i686-w64-mingw32- mingw
    $ make

即可，记得改prefix。生成`libssl.a`和`libcrypto.a`

最后去libcurl里的lib文件夹里修改`Makefile.m32`文件，在`CC	= $(CROSSPREFIX)gcc`上加一行`CROSSPREFIX=i686-w64-mingw32-`（请按需修改），然后把下面`CFLAGS`那行改成这样`CFLAGS	= -g -O2 -Wall -DCURL_DISABLE_LDAP`，最后

    make -f Makefile.m32 CFG=-zlib

或是

    make -f Makefile.m32 CFG=-zlib-ssl

make到最后时会报个错，是因为文件没放对地方，手动挪一下即可

``` bash
for x in vtls/openssl.o vtls/gtls.o vtls/vtls.o vtls/nss.o vtls/qssl.o vtls/polarssl.o vtls/polarssl_threadlock.o vtls/axtls.o vtls/cyassl.o vtls/curl_schannel.o vtls/curl_darwinssl.o vtls/gskit.o
do
    mv `basename $x` vtls
done
```
然后再make一下，`libcurl.a`文件应该就出现了。
如果生成dll出错也不要紧，我们要的是`.a`文件

### STEP4 测试
现在，你可以找一段libcurl的demo来测试了。注意要加上宏定义`CURL_STATICLIB`

    i686-w64-mingw32-gcc -I. -L. -DCURL_STATICLIB curl_demo.c -lcurl -lz -lws2_32 -o curl_demo.exe

如果你因为不知道gcc`-I`和`-L`选项的用法而编译不过，请自行Google。如果你加了ssl支持，你需要链接更多的库，具体请根据错误信息自行Google。最后提醒一点:**请把`-lcurl`选项放在源文件后面**，我当初就是因为这个死活链接不过。最后把`curl_demo.exe`拖进虚拟机里，如果一切正常，那么恭喜你，你成功了。
