---
layout: post
title: 编写一个简易的Chrome扩展
comments: true
categories: Informatics
tags: [技术宅,Chrome,编程,教程]
keywords: ["Chrome Extension",编写Chrome扩展,"Content Scripts"]
description: "编写一个简易的Chrome扩展以修改页面内容"
date: 2017-05-12 17:30:00
---
时隔半年的毫无诚心的流水帐作品。
假设读者有基础的Javascript能力。

# 起因
常去的某资源站的某资源发布者喜欢把重要的内容加上花里胡哨的特殊效果并藏在页面的角落里。
虽说要尊重资源的发布者，不过这种给人添堵的行为实在令我感到不爽，于是研究了一下Chrome的扩展程序（Extension）。

# 基本思路
要干的事情有两件：

1. ~~去掉特殊效果~~ 好吧，其实这条并不重要，毕竟内容已经被提取出来了
2. 将内容移到页面的显著位置

很自然的，直接用Javascript操纵DOM树即可实现希望的效果。
那么要怎么自动载入脚本呢？
<!--more-->
# 编写扩展
感谢Chrome提供了强大的扩展系统。自动载入脚本这种功能自然是小菜一碟啦。
首先编写一个脚本`content_script.js`操纵页面元素：

``` javascript
if (page_require_modification()) {
    var content = document.getElementById("hidden-div").innerHTML;
    var clean_content = document.createElement("p");
    clean_content.append(document.createTextNode(content));
    document.getElementById("main-div").append(clean_content);
}
```
没有魔法，想干啥就写啥，就像是HTML本身引用了一个JS文件一样。也不需要考虑`document.ready`的问题，因为Chrome默认会在文档加载完成后再加载自定义的JS。

接着需要一个`manifest.json`文件，这样Chrome才能将其作为一个Extension加载。

``` json
{
  "manifest_version": 2,

  "name": "在这里填上扩展的名称",
  "description": "这里填一些描述",
  "version": "1.0",

  "content_scripts": [
      {
          "matches": ["https://www.google.com/*"],
          "js": ["content_script.js"]
      }
  ]
}
```
~~你觉得我会把实际的URL写出来嘛？肯定不会啦！~~
Chrome把这种注入到页面中的脚本称做`content_scripts`。当页面的URL符合`matches`中的pattern时，就自动加载`js`中指定的脚本。当然，脚本的文件名可以自由决定，只要前后一致即可。

最后一步，将`manifest.json`和`content_script.js`放入同一个文件夹。然后在`chrome://extensions`选择`加载已解压的扩展程序`即可加载扩展啦。

# 总结
- `manifest.json`文件指示了一个Chrome扩展
- Chrome扩展能将自定义脚本注入到符合指定URL的页面中
- 这种单纯的脚本注入任务可能Greasemonkey更适合一些，不过这次就先研究Chrome扩展啦~ 有机会再研究油猴脚本。
- Chrome扩展可以和浏览器本身做到更紧密的结合，比如提供菜单项或者是GUI之类的，不过这篇文章完全没有涉及。
- [Google的官方扩展指南](https://developer.chrome.com/extensions)永远是你的好伙伴
