---
layout: post
title: "新浪微盘数据结构解析"
date: 2013-01-27 22:00
categories: Informatics
comments: true
tags: [Black Technology,网盘]
---
*注意!这个是[微盘](http://vdisk.weibo.com)而不是[威盘](http://www.vdisk.cn)*
研究这个问题的起因是有一次我需要从微盘上批量下载一堆文件。做为一个会写程序的人，我怎么能亲自做如此ugly的工作呢？这种事情当然要交给电脑做了！为了做到自动获取文件连接，于是就不得不研究这个问题了。
<!-- more -->
### 用户的登录
微盘内部使用一个叫做`gsid`的值来识别用户。具体获取方法如下：

1. URL:`http://vdisk.weibo.com/wap_auth`。
2. POST参数 `username`:用户名
3. POST参数 `password`:用户密码
4. 返回数据：JSON字符串。Example:`{ "message": "/?gsid=..."}`。其中`...`的部分就是`gsid`

### 文件信息的获取
得到`gsid`后，就可以用来下载文件了。当然，要先得到下载连接。

1. URL：`http://vdisk.weibo.com/share/ajaxFileinfo`
2. GET参数 `fid`：文件ID
3. Cookie参数 `gsid`:GSID
4. 浏览器标识(UA):需要设置为移动设备
5. 返回数据：JSON字符串。	Example:

```json
{"id":"406458665","name":"\u5fae\u535a\u5c01\u9762\u80cc\u666f.zip","uid":"60999569","dir_id":"0","ctime":"1358820088","ltime":"1358820088","dtime":"0","size":"1662111","is_locked":"0","type":"application\/x-zip","md5":"9e8eec4a5d2d3ac5be41bb9f34dc3e40","sha1":"6e59853b198e3eae818d5b0756f47c34d4eae6df","w":"0","h":"0","hid":"0","status":"1","app_key":"139204333","source":"2","ip":"0","rev_id":"140316098","share_status":"0","share":-1,"s3_url":"http:\/\/file.data.vdisk.me\/61099569\/6e59853b198e3eae818d5b0756f47c34d4eae6df?ip=1358945643,10.75.7.27&ssig=o9x4sQ9tz6&Expires=1358944443&KID=sae,l30zoo1wmz&fn=%E5%BE%AE%E5%8D%9A%E5%B0%81%E9%9D%A2%E8%83%8C%E6%99%AF.zip","url":"http:\/\/vdisk.weibo.com\/s\/oex4F","byte":"1662111","length":"1662111"}
```

其中，`s3_url`就是下载地址了。顺带一提，从`http://vdisk.weibo.com/file/info?fid=...`也可以得到文件信息，需要Cookie:`gsid`，但似乎不是所有文件都能成功得到信息。
### 文件下载
从上一步得到了URL就可以下载了。

1. URL：`s3_url`
2. Cookie参数 `gsid`：GSID

需要注意的是，这个URL可能会有很多`302 Redirection`，可能是出于负载平衡的原因吧。
### fid与fid64
平时我们下载都是用`http://vdisk.weibo.com/s/aMVfa`这种形式的短链接，其中`aMVfa`就是fid64,说白了就是64进制表示的fid，其0～63对应如下：

    0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_-

因此，`aR_8-`解码后就是`181920319`
### 文件列表的获取
研究完上文所提到的东西后，应该就可以进行文件批量下载了。但做为一个上进的好青年，我们不会止步与此。为了能够方便地管理自己的文件夹，当然要做进一步研究。

1. URL：`http://vdisk.weibo.com/dir/list`
2. GET参数 `dir_id`：Directionary ID
3. Cookie参数 `gsid`：GSID
4. 浏览器标识(UA):需要设置为移动设备
5. 返回数据：JSON字符串。其中`dirinfo`中的`dir_num`和`file_num`分别储存了在这个文件夹下有多少个目录和多少个文件。`data`段是一个数组，每个元素都代表了一个文件或目录，其中保存了`fid`或是`dir_id`。通过是否有`type`段来判断具体是文件还是目录。

用户根文件夹的`dir_id`是`0`
### Cookie与浏览器标识(UA)
似乎所有的链接都可以通过在Cookie中加入`device=mobile`来跳过UA检查。(就是说不用设置UA了)
