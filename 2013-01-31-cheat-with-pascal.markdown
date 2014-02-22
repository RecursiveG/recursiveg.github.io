---
layout: post
title: "Pascal中鲜为人知的那些技巧"
date: 2013-01-31 21:46
comments: true
categories: Informatics
tags: [Pascal,ACM,编程]
description: Some special skills programming with pascal. Useful in ACM.
---
做为一个搞信息学竞赛这么长时间的人，再加上估计很快就要转C++了，我觉得我有必要留下一些关于Pascal语言的资料，于是就有了这篇文章。我只负责解释用法，对基础概念不了解的请自行Google。所有这篇文章里的东西应该都能在Free Pascal自带的文档里找到，我写出来是为了众多不喜欢看英文的同学们，如果你愿意自己去看一下，一定会收益匪浅。
###不同进制的表示
平时我们写的常量都是十进制数，但我们有时需要写一个比如十六进制数怎么办呢？我们当然可以手动计算一下，但还有更优雅的方法。

	writeln($Ff,#32,&10,#32,%100);
	
你觉得它会输出什么呢？它输出`255 8 4`!所以以`$`开头的是16进制数，`&`开头的是8进制数，`%`开头的是二进制数。顺带一提的是，以`#`开头的数会转变成对应ASCII码的字符，其实它可以和前面的三个符号共同使用，即`#$20`和`#%100000`都代表了空格。
<!-- more -->
###函数内联
关于内联的解释请自己找资料，写法如下：

	function foo(bar:Type):ReturnType;inline;
	
即在函数头后加`inline;`即可。测试证明确实有效，不过建议只用于诸如`min`或`max`这种函数。
###函数的重载
Pascal也是支持重载的，甚至可以重载系统函数！演示如下：

	procedure sort(var a:TArray;l,r:longint);
	begin
		...
	end;
	procedure sort(var a:TArray;r:longint);
	begin
		sort(a,1,r)
	end;
	
这样，调用`sort(arr,top)`就相当于调用`sort(arr,1,top)`.
###操作符重载
是不是对写高精度时的`plus(a,b)`感到厌倦？是不是想换一种更帅的书写方式？没问题，操作符重载能满足你的愿望！它可以让你用`a+b`的形式对高精度进行计算！

	operator + (a,b:Type) c:Type;
	operator := (a:Type1) b:Type2;
	operator > (a,b:Type) c:boolean;
	
需要注意的是

- 比较操作符的返回值只能是`Boolean`
- 二元操作符和赋值操作符如果两端类型不同不能随意交换位置
- 重载后优先级不变

为了解决不能随意交换位置的问题，你可以这样写：

	operator + (a:Type1;b:Type2)c:ReturnType;
	begin
		...
	end;

	operator + (a:Type2;b:Type1)c:ReturnType;
	begin
		c:=b+a
	end;
	
###想重载str()和val()？
看完前面的函数重载，你是不是迫不及待地想要重载`val()`和`str()`这两个你看着不爽很久的函数了？但是却发现不能调用系统原来的函数了，Pascal把它当成了递归！解决方法很简单，在要用原始系统函数的地方加上`system.`即可。
``` pascal
function str(x:longint):string;
begin
	//str(x,str)<--This is completely wrong!
	system.str(x,str);//<--This is the right form
end;
```
###动态数组
担心数组太大爆内存？但心数组太小存不下？动态数组解除你的忧虑！主要操作如下：

1. `a:array of Type;`：变量声明。
2. `setlength(a,length)`：设定数组下标，范围为`[0..length-1]`，会自动清零。
3. `b:=a`:看上去像是赋值，但其实不是赋值，只是复制地址而已，因此对`b`的修改就是对`a`的修改。
4. `c:=copy(a,0,length(a))`：这就是真正的赋值了！还记得`copy()`和`length()`函数么？现在它们可以用于动态数组了！
5. `d:array of array of Type`:二维动态数组声明。
6. `setlength(d,length1,length2)`:不解释。
7. `a:=d[1]`:`a`是一维动态数组，对`a[x]`的修改就是对`d[1][x]`的修改。
8. `copy(d[x],0,length(d[x]))`:取出第`x`个一维动态数组。
9. 二维动态数组可以用`d[x,y]`的方式访问，也可以用`d[x][y]`的方式访问。

###取地址操作符
想用C语言中的`&`操作符？在Pascal中它是`@`!估计某年的NOIP坑了不少人。
###输入输出重定向
你还在定义`text`类型么？你还在用查找替换功能批量替换你的`readln()`么？赶快试试这个！

	assign(input,'foo.in');reset(input);
	assign(output,'foo.out');rewrite(output);
	...
	close(input);close(output);
	
再也不用担心输入输出了！
###用动态数组实现伪变参
是不是很羡慕C中`printf`的变参？是不是很羡慕`writeln`可以有好多好多参数？利用动态数组可以实现类似的功能！
``` pascal
function average(arr:array of longint):real;
var i:longint;
begin
  average:=0;
  for i:=0 to high(arr) do
    average:=average+arr[i];
  average:=average/length(arr) 
end;
```
请注意其中`high()`和`length()`的区别。合法调用如下：

	var A:array[1..MAX]of longint;
	average([3]);
	average([1,2,3,4]);
	average(A);
	average(A[1..5]);
	
###函数也是变量？！
或许你对C++中的`sort()`已有所耳闻，或许你已经知道，它的比较函数是做为参数传进去的。配合`@`操作符，Pascal可以做到相同的效果。
``` pascal
type TMyCompareFunc=Function(a,b:MyType):boolean;
function largerthan(a,b:MyType):boolean;
begin
  //...
end;
procedure sort(var a:array of MyType;l,r:longint;f:TMyCompareFunc);
begin
  //...
  //Use `f(a[x],a[y])` to compare
end;
begin
compare(arr,1,max,@largerthan);
end.
```
既然是变量，就能互相赋值，但是请注意：
``` pascal
type TNoArgFunc=Function():integer; 
var
  F:TNoArgFunc;
  N:integer;
function MyFunc():integer;
begin
  MyFunc:=3; 
end;
begin
F:=MyFunc; //F()成为MyFunc()的别名
N:=MyFunc; //N被赋值为3

//if F=MyFunc then
//  writeln('You will never see this');
//这个判断将导致`类型不匹配`编译错误

if F=@MyFunc then
  writeln('这是同一个函数');

if F()=MyFunc then
  writeln('这两个函数的返回值相同');
end;
```
###C风格的操作符
我就不多介绍了`+=`、`-=`、`*=`、`/=`...考试时先试试能不能用。使用有风险，偷懒须谨慎。
###强制类型转换
知道C中的`(int)a`或是`int(a)`么？不知道没关系，Pascal中的强制类型转换是这样写的`TypeIdentifier(Variable)`。下面给几个例子：
```pascal
type IntPtr=^integer;
IntVal:=97;
RealVal:=3.7; 
longint(IntVal);//长版的IntVal
real(IntVal);//实数版IntVal，效果和赋值一样
pointer(IntVal);//一个指向内存地址97的无类型指针
IntPtr(IntVal);//一个指向内存地址97的Integer指针
longint(@IntVal);//IntVal地址的Longint版。注意，它是一个数，所以可以用writeln()输出
writeln(IntPtr(@RealVal)^);//你可以猜猜这句话输出什么（写程序时请绝对不要这么做）
```
###无类型变量与无类型指针
标题写着`无类型变量`，其实应该叫做`多类型变量`更准确。他的主要工作原理就是在内部进行类型转换，因此效率极其低下，占用空间还特别大，连官方手册都不建议用。类型名为`Variant`。
无类型指针就是上一节提到的`Pointer`了。任何指针都可以赋值给它，它也可以赋值给任何指针，例子如下：
```pascal
var
p:pointer;
r:real;
i:^integer;
begin
r:=3.7; 
p:=@r; 
i:=p;
writeln(i^); 
end.
```
看出来了么？这段文字和上一段的最后一句话等效。
###这玩意儿是类？！
这是我最近才看到的一种写法，从来没用过。有愿意尝试的请自行研究。
``` pascal
type 
TGetSum=object
a,b:longint;
procedure Init(x,y:longint);
function GetSum():longint;
end;
procedure TGetSum.Init(x,y:longint);
begin
a:=x;b:=y;
end;
function TGetSum.GetSum():longint;
begin
GetSum:=a+b; 
end;
var
sum:TGetSum;
begin
sum.Init(2,3);
writeln(sum.GetSum())
end.
```
关于Object Pascal,推荐一本书:[Start Programming Using Object Pascal](http://code-sd.com/books/startprog/)
###后记
Pascal是不少OIer最开始使用的一种语言，仅以此文献给众多正在使用和曾经使用过Pascal的OIer。
**做人要厚道，转载请注明出处！**

