---
layout: post
title: "Sync with iDevice on Linux"
date: 2012-12-31 21:01
comments: true
categories: Informatics
tags: [Written In English,iDevice,Linux]
---
It's a bit hard to connect an iDevice with Linux because Apple is not so open and we have to use iTunes to sync with our iDevice for a long time. Luckily we now have a set of tool so that we can control our device on linux. The most important two library are [libimobiledevice](http://www.libimobiledevice.org/)(libiphone) and libgpod.

libimobiledevice, like it's name, is a library who provides the interface to access the iDevice. It provides a higher level of access such as photo, bookmark, install/uninstall softwares and even sync music. And it doesn't need jailbreak.

What I want to mention is how musics synchronized with an iDevice. Under the iTunes folder (You may never seen that before. That's ordinary.), there's a file called iTunesDB. That's the file which libgpod really works with. This file contains the name of songs, singers' names, your play lists and so on. Unfortunately, because Apple don't want it be modified by any programs except iTunes, they add some hash info into the file. If iPod found the hash is incorrect, it refused to display the songs. There was once a project called iPodHash, but it seems to be die due to a DMCA notice. Apple engineers have changed the hash algorithm for several times and the latest version haven't been reverse-engineering, as a result, now we can only sync with a old version of iOS.

If your iDevice is jailbreaked, you can change a key called DBVersion(Sorry, I forgot where it is.). It tells iPod which version of hash algorithm it should use so we could use a known hash on new iOS. This process depends on libimobiledevice too. It only support to sync with iOS 4 or older. That means it's useless even if you changed DBVersion on your iOS5 device. By the way, you may will not find a iTunesDB file but a iTunesCDB instead. It's a compressed version of iTunesDB using zlib.

I feel so sad that such a project is closed and now I can only sync with my iPod on Windows.
