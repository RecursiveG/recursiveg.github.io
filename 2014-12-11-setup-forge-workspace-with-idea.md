title: IDEA下建立Forge开发环境的正确姿势
tags: [Minecraft,教程,Java]
keywords: [Minecraft,Modding,Forge,ForgeGradle,IDEA,IntellijIDEA]
comments: true
date: 2014-12-11 14:17:48
categories: Informatics
description: Minecraft mod developing in seperated folders under linux with IDEA.
---
# 前言
见过不少教程都是基于Eclipse的，而基于IDEA的文章少得可怜，遂决定写此文。
本文通篇基于Linux/IntellijIDEA进行讲解，Windows/MAC/Eclipse用户请自行依葫芦画瓢。

# 设置Forge工作区
当然，你得首先去[MinecraftForge](http://files.minecraftforge.net/)下载一份源代码。我这里用的是最新的`forge-1.7.10-10.13.2.1258-src.zip`
接着，找个地方建立一个文件夹，这将是你的工程目录，我的叫做`Forge1.7.10-1258`。然后再在里面建立一个目录，比方说就叫`forge-1.7.10-10.13.2.1258-src`，把你的Forge源码解压进去。
现在，你的文件夹层次应该看起来是这样的：

    Forge1.7.10-1258
    └── forge-1.7.10-10.13.2.1258-src
         ├── build.gradle
         ├── CREDITS-fml.txt
         ├── eclipse
         ├── forge-1.7.10-10.13.2.1258-changelog.txt
         ├── gradle
         ├── gradlew
         ├── gradlew.bat
         ├── LICENSE-fml.txt
         ├── MinecraftForge-Credits.txt
         ├── MinecraftForge-License.txt
         ├── README.txt
         └── src

你可以先按自己喜好改动一下`build.gradle`。比如，我喜欢手动指定一下mappings的版本：

``` groovy
minecraft {
    version = "1.7.10-10.13.2.1258"
    runDir = "eclipse"
    mappings = "stable_12"
}
```

接着，cd到forge-1.7.10-10.13.2.1258-src目录下，执行如下两条命令：

    gradle -i setupDecompWorkspace
    gradle -i ideaModule

然后请耐心等待指令完成，可以去喝杯牛奶睡个觉什么的。
<!--more-->
# 导入到IDEA
等以上操作完成后，就可以打开IDEA，选“Create New Project”，注意要建立一个**空工程**（Empty Project）
![Create New Project](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/1.png)
"Project Name"自然可以随意填写，"Project Location"则是之前创建的目录，下方的"Project Format"推荐选"Directory Based"
点"Finish"之后应该会自动打开"Project Structure"窗口，如果没有的话可以按Ctrl+Alt+Shift+S或是从菜单栏"File --> Project Structure"
打开窗口之后我们首先要选择SDK版本：先点左边的"Project"，然后在右边"Project SDK"里选一个，我是选了Java8，你当然可以选择任何版本（不要低于Java6）
![Select Project SDK](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/2.png)
接着点左边的"Modules",再点那个绿色的“+”号，接着选"Import Module"
![Location of "Import Module"](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/3.png)
然后选forge-1.7.10-10.13.2.1258-src目录下的`forge-1.7.10-10.13.2.1258-src.iml`文件就好。
Import完了之后检查下有没有报错，如果没问题就可以点右下角OK。

重新cd到forge-1.7.10-10.13.2.1258-src目录下，执行如下三条命令让gradle自动建立运行配置。

``` bash
ln -s ../.idea .
gradle -i genIntellijRun
rm .idea
```

回到IDEA，然后有必要的话重新加载一下Project。继续选菜单栏"Run --> Edit Configurations",点左侧的"Minecraft Client"，修改"Working Directory"到`forge-1.7.10-10.13.2.1258-src/eclipse`。
![Edit Configuration](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/4.png)
你也可以像我一样指定一个username。接着对"Minecraft Server"也如法炮制。然后右下角"OK"退出。

现在Forge应该就可以运行了，在IDEA的主界面右上角有这么一片区域，选"Minecraft Client"然后点右侧那个绿色的三角箭头即可
![Execution Control Bar](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/5.png)

# 一个Mod！
如果之前的步骤都没有问题，你就可以继续了
为了更好地演示运行配置以及发布流程，我决定写一个Mod，添加一种矿石：“Xp Ore”,顾名思义，挖掉后能得到大量经验。

我们需要新建一个Module来写我们的代码：菜单栏"File --> New Module"
我就叫XpOre好了，然后继续打开"Project Structure"，将`forge-1.7.10-10.13.2.1258-src`添加成为它的依赖。
![Add to Dependencies](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/6.png)
那个菜单同样是点右边的绿色加号出来，点"Module Dependency"后在弹出来的窗口里选"forge-1.7.10-10.13.2.1258-src"然后"OK"即可。

之后就可以写Mod了！至于具体Mod怎么写我就不在这里提了，请各位参考其他文章。
这是我的代码和目录层次结构，请自行调整，我就不把每一步的细节都写出来了。
请记得把`java`和`resources`两个目录设置成代码根目录和资源根目录，具体方法是在文件夹上右键然后"Mark Directory As"
![The Module Tree](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/7.png)
（xp_ore.png是矿石的材质，其实就是拿金矿石的材质把几个像素涂成绿色让它看起来比较像附魔瓶的颜色）
为了防止图挂，我再拿文本形式列一下目录

    XpOre
    ├── src
    │   └── main
    │       ├── java (Sources Root)
    │       │   └── org
    │       │       └── devinprogress
    │       │           └── xpore
    │       │               └── XpOre.java
    │       └── resources (Resources Root)
    │           ├── assets
    │           │   └── xpore
    │           │       ├── lang
    │           │       │   └── en_US.lang
    │           │       └── textures
    │           │           └── blocks
    │           │               └── xp_ore.png
    │           └── mcmod.info
    └── XpOre.iml

要想让这个Mod在IDEA里运行起来，有两种方式。第一种比较简单，直接菜单栏"Run --> Edit Configurations --> 'Minecraft Client' --> Use classpath of mod ..."下拉列表里选"XpOre"，保存退出运行即可。这种方式比较适合只开发一个Mod的情况。
当有N个Module互相依赖的时候，我推荐创建另一个Module，比方说，叫"Run"。然后令其依赖`forge-1.7.10-10.13.2.1258-src`和你需要加载的其他Module，然后"Use classpath of mod"选择"Run"即可。

# 发布
发布其实相当方便，把forge-1.7.10-10.13.2.1258-src文件夹里的`build.gradle`复制到`XpOre`文件夹下面
按自己喜好修改其中的`version`,`group`和`archivesBaseName`即可。比方说我的是：

    version = "v0.1"
    group= "org.devinprogress.xpore"
    archivesBaseName = "XpOre-1.7.10"

然后在XpOre文件夹下`gradle build`即可。运行完成后，就能在`XpOre/build/libs/`文件夹下找到编译好的jar了。

最后来一张Mod的效果图
![XpOre Mod](http://0f4529-files.oss-cn-hangzhou.aliyuncs.com/blog/setup-forge-idea/8.png)
