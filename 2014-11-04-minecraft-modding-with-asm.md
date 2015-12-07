title: "Minecraft Coremod开发杂事记"
tags: [教程,Minecraft,Java]
keywords: [Minecraft,ASM,Forge,Coremod,AccessTransformer,szszss,srg]
comments: true
date: 2014-11-04 21:48:53
categories: Informatics
description: "Some notes on developing coremods for Minecraft, mainly about ASM & AccessTransformer"
---
表示最近时间荒废得厉害，主要都是耗在了[Minecraft](http://www.minecraft.net)这款游戏上。
Minecraft的一大魅力在于其几乎无穷的MODs，于是我也小试了一下Mod开发，顺便学习一下Java。<del>于是掉入了万劫不复的深坑</del>
当然，我要做点和加个方块、改个合成表之类的不一样的事。
（教程中不少内容都参考了szszss的[博客](http://www.hakugyokurou.net/wordpress/?page_id=126)，在此表示深深的感谢）

# 基础知识
关于这篇文章，不适合特别特别新的新人，我假设各位读者都有一些基础的编程经验。如果你是入门级别的，在MCBBS论坛的[编程开发板块](http://www.mcbbs.net/thread-54579-1-1.html)有不少不错的入门教程。
我假设各位读者都具备以下能力:

- 会安装软件
- 了解基本的程序流程控制，比如判断、循环等
- 了解基本的OOP概念，比如类，继承等 （其实这条不是那么重要，Java看多了就自然会了 <del>一个原C程序员如是说</del>）
- 了解命令行、终端的基本使用方法
- 有方法正常访问国际互联网，如Facebook等
- 了解基本英语单词（这条似乎也不是那么重要，主要是希望大家能够在遇到问题时不要怕阅读英文资料）
- （本教程面向Linux用户，Mac用户大同小异，Windows用户自己看着办）

<!--more-->
然后再来介绍一下要用到的工具:

- [MCP](http://mcp.ocean-labs.de/)(Minecraft Coders' Pack)主要负责反混淆Minecraft的代码，同时向Forge提供对应的文档
- [FML](http://files.minecraftforge.net/fml/)(Forge Mod Loader)提供了一些底层功能，如Mod加载，ASM等。
- [Forge](http://www.minecraftforge.net)提供了更高级的接口，如增加方块，修改合成表等。
- ForgeGradle帮助建立开发环境和发布

一般来说，FML都会附带在Forge里，在某些情况下，比如现在（2014年11月5日）1.8的Forge还未完成，但FML已放出，就可以单独只安装FML，先开始Coremod的开发。

# Intellij IDEA配置教程
网上大部分教程都是讲Eclipse的，但是个人偏好IDEA，所以讲一下IDEA的配置流程。

1. 安装IDEA，没有必要找破解版，免费的Community Edition足够
2. [下载](http://files.minecraftforge.net/)Forge代码，就是Src那个链接。请选择自己需要的版本，我以`1.7.10-Recommended`为例
3. 解压到一个你看着顺眼的地方，然后依次执行以下命令

       gradle setupDecompWorkspace
       gradle idea
       gradle genIntellijRun

   没有装gradle也不想装的，可以用`./gradlew`
   强烈建议挂着代理或VPN做这事，否则将是极端痛苦的过程。
   你也可以加上`-i`选项看滚滚的数据输出以不至于那么无聊。
4. 打开IDEA，直接Open Project，选择目录下的.ipr文件应该就好了，你可以试着Run一下看看有没有什么问题。

注：直接`gradle idea`现在是不被推荐的，可以尝试用`gradle ideaModule`代替，具体方法在[我的另一篇日志](http://www.devinprogress.org/2014/12/setup-forge-workspace-with-idea/)里有讲。
如果需要Socks代理的，可以这么来`gradle -DsocksProxyHost={代理服务器地址} -DsocksProxyPort={代理端口}`

# 代码管理与开发
~~源代码和资源文件都是放在`src`文件夹里的，有时要在多个不同的Mod间切换开发，我目前的解决方法是将代码统一放在别处，将src文件夹做软链接进来。同时我也非常推荐也用这种方法处理`build.gradle`文件。将代码放在别处还有个好处，就是可以用`git`来管理版本，而且可以用分支方便地管理对不同Minecraft版本做的修改。不管什么方式，自己习惯就好。~~

用了`gradle ideaModule`后，代码本身就分开放置了，不再需要这种方法了。

对于开发这一部分，自己深感无力(其实就是懒)，请参阅szszss的系列教程。
另外，现在已经没有Coremod文件夹了，所以所有Mod都放在Mods文件夹下，不同之处只在于`MANIFEST.MF`文件。

# 发布
感谢ForgeGradle，打包发布不再需要手动拷贝压缩一大堆文件了。首先，你需要修改下`build.gradle`文件，这也是我为什么推荐用软链接来管理它的原因。
以原始的文件为例：

``` groovy
version = "1.0"
group= "com.yourname.modid" // http://maven.apache.org/guides/mini/guide-naming-conventions.html
archivesBaseName = "modid"

minecraft {
    version = "1.7.10-10.13.2.1230"
    runDir = "eclipse"
}
```

`archivesBaseName`和第一行`version`都可以自由修改，只会影响输出的jar文件的名字，关于`group`用途不明，有了解的求留言告知。
如果你打算把Mod升级到一个新的Forge版本，请务必修改`minecraft.version`和你的开发环境一致，否则会出现奇奇怪怪的问题。
修改好后，就可以用`gradle build`来编译了，同样建议开代理。编译好的jar在`build/libs`下。

如果是需要对MANIFEST进行修改的，比如Coremod，需要在`build.gradle`中minecraft块之后添加jar块:

``` groovy
jar {
    manifest {
        attributes 'FMLCorePlugin': 'org.devinprogress.uniskinmod.SkinCore'
    }
}
```
如果你的一个jar包里既有普通Mod（以`@Mod`作Annotation的）又有Coremod，你还需要

    attributes 'FMLCorePluginContainsFMLMod': true

否则普通Mod不会被载入。

# ASMTransformer
因为MCP坑爹的反混淆机制，开发者在处理Method或Field时需要对付三种不同的名字：

1. 形似`a`这样的混淆名，`obfName`
2. 形似`func_xxxx_a`这样的半混淆名，有时也称作`srgName`
3. 形似`doTick`这样的反混淆名，或称`mcpName`

关于为什么要有srgName，MCP是这么解释的：因为mcpName是任何人都可以贡献的（这是真的），所以会出现这么一种情况，有时为了更好地描述某个函数的功能，在次要版本升级时（比如1.7.1升级1.7.2），mcpName会发生变化，如果直接以mcpName进行编译，那么为1.7.1编译的Mod就无法在1.7.2上使用，即使其他方面都没有问题。于是为了解决这个问题，引入了相对固定的srgName。FML是这么处理名称的，在`gradle build`时，代码中所有的mcpName均会被混淆成srgName。然后在玩家运行游戏时，所有的obfName均被反混淆成srgName，即`RuntimeDeobfuscation`，运行时反混淆。

在你修改某个方法之前，首先必须定位它（废话）。定位一个方法需要四个信息：

- 方法所在的类的完全限定名（这里这里指的是反混淆了的类名）
- 方法的srgName
- 方法的mcpName(用于在开发时确定方法，有时和srgName相同)
- 方法的Description，或者说，参数列表。


类很好确定，每当一个新的类被加载时，都会调用`IClassTransformer`接口的`transform`方法：

``` java
public interface IClassTransformer {
    byte[] transform(String obfuscatedClassName, String transformedClassName, byte[] bytes);
}
```

第二个参数就是反混淆了的类名，并且是点分割，大小写正确的，可以直接用`equals()`来判断。
但是方法名的判断就比较复杂，因为在ASM转换时，运行时反混淆还没有被执行，所以方法名全部都是obfName。更要命的是，如果方法的参数里有Minecraft的类，那么这个类名也是被混淆了的类名。

不过谢天谢地，我们有`FMLDeobfuscatingRemapper`，你可以用`FMLDeobfuscatingRemapper.INSTANCE`来取得实例。这个类提供了几个重要的方法：

``` java
String mapMethodName(String obfedClassName, String obfedMethodName, String obfedMethodDescription)
String mapFieldName(String obfedClassName, String obfedFieldName, String obfedFieldDescription)
String mapMethodDesc(String obfedMethodDescription)
```

其中`mapMethodName`和`mapFieldName`返回对应方法和字段的srgName，`mapMethodDesc`返回反混淆了的Description，也就是将`(Lbee;F)V`这种变为`(Lnet/minecraft/client/gui/GuiMainMenu;F)V`。

如果你需要在某个方法中添加大段的代码，我极度不推荐写长长的代码将所有这些操作码全部加到目标方法里去，这种方法枯燥至极，又不直观，还难于调试。我一般的方法是，使用`INVOKESTATIC`调用自己写好的函数，并将需要修改的变量作为参数传递，这样需要的代码不多，也易于调试和维护。

在向代码中添加操作，尤其是费时的操作时（比如网络IO）请务必谨慎选择插入的位置。因为大部分代码会在主线程中执行，一旦卡住轻则界面冻结，重则直接被服务器超时踢出。

我不是非常推荐让ASM自动计算栈大小和本地变量区大小，因为碰到一些比较复杂的类时会悲剧，比如`AbstractClientPlayer`

如果你想清空一个方法让它什么都不做，请还是不要忘记加上RETURN

有时，开发环境下编译出的class和原始的class会有区别，所以还是建议用javap之类的工具看一下原始的字节码。

# AccessTransformer
有时，我们需要频繁调用某个private的Method或是Field，使用反射会有性能损失，而ASMTransformer也无效(因为无法通过编译)，这就到了AccessTransformer大显身手的时候了。
AccessTransformer用于将private或是protected的Method和Field变为public，这样在代码中就可以直接使用了。
你需要首先创建一个`*_at.cfg`的配置文件放在resources目录（就是放`mcmod.info`的目录）下，然后将其软链接到根目录下（和`build.gradle`同目录）。
配置文件的语法类似这样（这是`fml_at.cfg`的一部分）

``` java
public net.minecraft.entity.EntityList func_75618_a(Ljava/lang/Class;Ljava/lang/String;I)V
public net.minecraft.entity.EntityList field_75625_b #nameToClassMap
public net.minecraft.item.crafting.CraftingManager func_92103_a(Lnet.minecraft.item.ItemStack;[Ljava/lang/Object;)Lnet.minecraft.item.crafting.ShapedRecipes;
```

接着重建工作区：

    gradle clean setupDecompWorkspace idea --refresh-dependencies

同样建议挂代理，用eclipse的同学把`idea`换成`eclipse`，有强迫症的同学可以加上`-i`选项。这样，开发环境下代码的改动就完成了。我在挂着代理的情况下大约需要8分钟。

为了让其在混淆环境下也能正常工作，你需要创建一个新的类，继承`AccessTransformer`:

``` java
public class MyATransformer extends AccessTransformer{
    public MyATransformer(){
        super("mymod_at.cfg");
    }
}
```
其中的字符串就是配置文件名，然后在实现了`IFMLLoadingPlugin`的类里：

``` java
@Override
public String getAccessTransformerClass() {
    return MyATransformer.class.getName();
}
```

另外需要注意的是，AccessTransformer不会自动转换衍生类，所以在转换基类时请务必当心，否则会编译不通过。

在1.7.10及以上的版本中，可以在build.gradle文件的中加入以下内容，这样就不必再写IFMLLoadingPlugin了。其中的`mymod_at.cfg`文件要放在META-INF文件夹下。

``` groovy
jar {
    manifest {
        attributes 'FMLAT': 'mymod_at.cfg'
    }
}
```

# 其他杂七杂八的东西
- 如果你想给MCP贡献mcpName，可以去IRC esper#mcp 找 MCPBot_Reborn
- 最新的反混淆名对应表可以在[MCPBot Export](http://export.mcpbot.bspk.rs/)找到
- 如果你厌倦了用文本编辑器搜索字符串来找srgName的话，可以试试[MCPMappingViewer](https://github.com/bspkrs/MCPMappingViewer/)
- [Procyon](https://bitbucket.org/mstrobel/procyon)是个极好的Java反编译器
