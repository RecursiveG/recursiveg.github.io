title: "有屏幕的地方就有烂苹果"
date: 2014-03-19 19:55:49
categories: ACG
tags: [技术宅,东方Project]
keywords: [Bad Apple,字符动画]
description: "A short tutorial on creating a character-base animation"
comments: true
---
如果你还不知道Bad Apple是什么东西，请移步[这里](http://zh.wikipedia.org/wiki/Bad_apple!!)
播放的原理很简单，就是不停的打印清屏再打印清屏。任何一个略有编程基础的人都能做到。比较令人头大的是如何把原视频转化为一个易于解析而且又不占地方的文件。
其实，借助`FFmpeg`、`ImageMagick`和一点点的编程小技巧就可以轻松完成。

第一步当然是要去下一个视频文件，我已经下好了，叫做`BadApple.mkv`。
<!--more-->
第二步要把视频变成一帧一帧的图片，请出FFmpeg来帮忙:

    ffmpeg -i BadApple.mkv -s 80x60 -r 15 Ba%d.png

然后你就会得到`Ba1.png Ba2.png Ba3.png`等一大堆文件，这就是各帧了。注意我在这一步同时把大小缩小到了80*60和把帧速率调到了15帧每秒。

第三步用ImageMagick将图像转换成黑白图，然后再转换成`xpm`格式。XPM格式本质上是一个文本文档，可以直接被`#include`。我们这一步要用到一点点脚本技巧。
``` bash
for x in *.png
do
  convert $x -monochrome `basename -s .png $x`.xpm
done
```
最后写一段C语言小程序，利用游程编码进一步缩小文件体积。

``` bad_apple.c
#include <stdio.h>
#include "xpm.h"
void main(){
  FILE *f=fopen("BA.dat","a");
  char count;
  int t=ARR[0][6]=='1'?1:2;
  for(int i=1+t;i<61+t;i++){
    count=1;
    for(int j=1;j<80;j++){
      if(ARR[i][j]==ARR[i][j-1]){
        count++;
      }else{
        if(ARR[i][j]=='.')count=-count;
        fprintf(f,"%c",count);
        count=1;
      }
    }
    if(ARR[i][79]==' ')count=-count;
    fprintf(f,"%c",count);
    fprintf(f,"%c",0);
  }
  fclose(f);
}
```

当然，离不了脚本和编译器的帮助，我这里使用了`tcc`进行编译
``` bash
for (( i=1; i<=3288; i++))
do
  cp Ba$i.xpm xpm.h
  tcc -DARR=Ba$i bad_apple.c && ./a.out
  rm xpm.h
done
```
其中，那个3288就是总帧数。这样就得到了一个`BA.dat`文件。文件内容是一堆用二进制存储的数字，正数代表连续的白色，负数代表连续的黑色，零代表换行。一帧60行，总计3288帧。这样就把一个80多兆的视频压缩到了900多K。有了数据文件剩下的就好办了。

**未完待续。。。。。。**
