---
title: apk加固原理
date: 2018-02-27 10:04:48
tags: Android
subtitle: Android
categories: Android
---

本文主要通过学习姜维大神的[Android中的Apk的加固(加壳)原理解析和实现](http://www.wjdiankong.cn/android%e4%b8%ad%e7%9a%84apk%e7%9a%84%e5%8a%a0%e5%9b%ba%e5%8a%a0%e5%a3%b3%e5%8e%9f%e7%90%86%e8%a7%a3%e6%9e%90%e5%92%8c%e5%ae%9e%e7%8e%b0/)和雪一梦大神的[根据”so劫持”过360加固详细分析](https://bbs.pediy.com/thread-223796.htm) 来记录自己的学习过程。

> ### 为什么要对APK加固

目前的软件发展速度非常快，尤其是移动端软件越来越深入人们的生活之中，我们时时刻刻都需要这些软件来进行支付，查询以及浏览信息。我们的隐私信息以及经济安全可谓是都在app之中。所以这也加快了Apk加固技术的发展，加固后的apk虽然增大了不法分子的破解难度，但是也明显降低了app的运行效率，所以有一些加固是针对于apk的主要逻辑代码进行的。加固也分为dex加固和so加固，但是呢，dex加固的重要性应该是更加高一点的，因为反编译后的java代码的可读性更高。

> ### 加固原理

图片盗用姜维大神

![](http://ousaim1qx.bkt.clouddn.com/download.png)

从上图我们可以明白，apk加固主要步骤为：

将一个需要加固的apk文件，一个负责解密的壳程序，先将apk文件进行加密，然后将解密的壳程序与之合并，得到新的dex文件，再将壳的dex文件替换，并得到一个全新的apk文件。将某个被加固的apk解压，如下图：

![assets文件夹](http://ousaim1qx.bkt.clouddn.com/jiagua.png)

经过加密后的文件的assets文件夹中多出了两个libjiagu.so和libjiagu_x86.so文件，这就是apk被加固的标志。

![dex文件](http://ousaim1qx.bkt.clouddn.com/jadx.png)

我们用jadx打开加压后的dex文件，看到和加固的原理一样，dex文件被替换。以上就是加固的原理。

> ### Dex文件解析

这里我需要盗用一张非虫大神的图：

![Android dex文件格式结构图](http://ousaim1qx.bkt.clouddn.com/Android%20dex%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E7%BB%93%E6%9E%84%E5%9B%BE.png)

Dex是Android平台上(Dalvik虚拟机)的可执行文件。其实所有的加固，加密措施都是针对DexHeader中的checksum，signature和filesize来进行的，因为加固需要对dex文件进行改动，所以dex文件校验的数据也必须进行改动，知道了这个点之后，也就明白了，我们只要掌握dexheader其他的也就万变不离其宗了。

首先根据图中所标注的，前8个字节是整个dex文件的magic（魔法数），其中低地址的4个字节的数据是dex文件

的标识符，所有的dex文件的标志符都是一样的。高地址的4个字节数据是dex文件的版本，图中的dex文件版本是035。`0x08h-0x0Bh`这段地址上存放的是checksum，使用adler32加密算法，用来检验dex文件除magic和checksum以外的所有文件区域，检查错误。`0x0Ch-0x1Fh`这段地址上存放的是signature，使用SHA-1 hash算法，识别dex文件的唯一性。使用双重校验保证文件的安全性以及效率。`0x20h-0x23h`这段地址上存放的是这个dex文件的大小，`0x24h-0x27h`这段地址存放的是dexheader的大小，我发现好像所有的dexheader的大小都一样都是0x70，`0x28h-0x2Bh`这段地址上标志的是dex文件的字节序，像图中的0x78563412就是默认为小尾模式，跟C/C++中的小端模式一样**即高地址存放低字节，低地址存放高字节**，`0x2ch-0x2fh`中说明的是连接段的大小，默认为0表示静态连接。