---
title: 【JVM源码探秘】在Mac上编译OpenJDK10源码
date: 2018-01-29 01:03:13
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk-logo.jpg)


博主在11年到12年的时候曾连续研究过十个月的JVM，读过的相关书籍包括：

- [深入Java虚拟机](https://book.douban.com/subject/1138768/)
这本书可以说是介绍JVM内部原理的鼻祖了，于2003年出版现已绝版，不过可以再某宝买到影印版。虽然当时JDK最高仅为1.4但JVM内部的构造已大体形成，所以博主强烈推荐此书。p.s 我肯定不会告诉你这书博主看了3遍：D

- [深入理解Java虚拟机](https://book.douban.com/subject/6522893/)
国内周某人写的，鉴于博主对于国人写的书向来不怎么感兴趣还是不提了。

- [HotSpot实战](https://book.douban.com/subject/25847620/)

- [揭秘Java虚拟机 JVM设计原理与实现](https://book.douban.com/subject/27086821/)

- [自己动手写Java虚拟机](https://book.douban.com/subject/26802084/)

之前的研究基本上都是虚拟机规范和JVM参数调优层面的内容，但是总觉得有些意犹未尽所以决定深入研究一下Hotspot实现，
由大部分C/C++和少量汇编代码构成，但清晰的结构和优雅的编码使其并不难读，不得不赞叹一句SUN的大师们的智慧。
今天就从编译OpenJDK开始我们的《JVM源码探秘》系列文章之旅。

<!-- more -->

以前尝试编译过OpenJDK7、8和9，由于源代码中存在诸多BUG，导致各种编译问题，解决来解决去，还是很难编译通过，很是不爽。
今天抱着试试看的态度又服用了一个疗程，拿起桌上的OpenJDK10，反手就是一次编译，居然像德芙一样丝滑，一次通过。


# 软件环境
- OS：macOS Sierra 10.12
- Xcode: 8.3.3
- Oracle JDK: 1.8.0_151
- freetype: 2.9
- ccache: 3.3.5(Optional)

# 前置条件
先确保系统已安装freetype和ccache
```bash
$ brew install freetype ccache
```

# 拉取代码
```
$ hg clone http://hg.openjdk.java.net/jdk10/master openjdk10
```
此处需要耐心等待一段时间，必要情况下需要多尝试几次才能拉取成功。



# 配置参数

接下来配置编译参数，以下是相关选项说明
- `--with-debug-level=slowdebug` 启用slowdebug级别调试
- `--enable-dtrace` 启用dtrace
- `--with-jvm-variants=server` 编译server类型JVM
- `--with-target-bits=64` 指定JVM为64位
- `--enable-ccache` 启用ccache，加快编译
- `--with-num-cores=8` 编译使用CPU核心数
- `--with-memory-size=8000` 编译使用内存
- `--disable-warnings-as-errors` 忽略警告 

```bash
$ bash configure --with-debug-level=slowdebug --enable-dtrace --with-jvm-variants=server --with-target-bits=64 --enable-ccache --with-num-cores=8 --with-memory-size=8000  --disable-warnings-as-errors

...

====================================================
A new configuration has been successfully created in
/Users/hunterzhao/CLionProjects/OpenJDK10/build/macosx-x86_64-normal-server-slowdebug
using configure arguments '--with-debug-level=slowdebug --enable-dtrace --with-jvm-variants=server --with-target-bits=64 --enable-ccache --with-num-cores=8 --with-memory-size=8000 --disable-warnings-as-errors'.

Configuration summary:
* Debug level:    slowdebug
* HS debug level: debug
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: macosx, CPU architecture: x86, address length: 64
* Version string: 10-internal+0-adhoc.hunterzhao.OpenJDK10 (10-internal)

Tools summary:
* Boot JDK:       java version "1.8.0_151" Java(TM) SE Runtime Environment (build 1.8.0_151-b12) Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)  (at /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home)
* Toolchain:      clang (clang/LLVM from Xcode 8.3.3)
* C Compiler:     Version 8.1.0 (at /usr/bin/clang)
* C++ Compiler:   Version 8.1.0 (at /usr/bin/clang++)

Build performance summary:
* Cores to use:   7
* Memory limit:   8000 MB
* ccache status:  Active (3.3.5)
```




# 执行编译
```
$ make images

...

Creating support/demos/image/jfc/CodePointIM/CodePointIM.jar
Creating support/demos/image/applets/MoleculeViewer/MoleculeViewer.jar
Creating support/demos/image/applets/WireFrame/WireFrame.jar
Creating support/demos/image/jfc/SwingApplet/SwingApplet.jar
Creating support/demos/image/jfc/FileChooserDemo/FileChooserDemo.jar
Creating support/demos/image/jfc/Font2DTest/Font2DTest.jar
Creating support/demos/image/jfc/Metalworks/Metalworks.jar
Creating support/demos/image/jfc/Notepad/Notepad.jar
Creating support/demos/image/jfc/SampleTree/SampleTree.jar
Creating support/demos/image/jfc/TableExample/TableExample.jar
Creating support/demos/image/jfc/TransparentRuler/TransparentRuler.jar
Creating support/classlist.jar
Creating images/jmods/jdk.jlink.jmod
Creating images/jmods/java.base.jmod
Creating jre jimage
Creating jdk jimage
WARNING: Using incubator modules: jdk.incubator.httpclient
WARNING: Using incubator modules: jdk.incubator.httpclient
Stopping sjavac server
Finished building target 'images' in configuration 'macosx-x86_64-normal-server-slowdebug'
```
大约10分钟后编译完成，由于机器配置不同可能会导致编译时长有所差异。

# 测试

```bash
$ ./build/macosx-x86_64-normal-server-slowdebug/jdk/bin/java -version
openjdk version "10-internal"
OpenJDK Runtime Environment (slowdebug build 10-internal+0-adhoc.hunterzhao.OpenJDK10)
OpenJDK 64-Bit Server VM (slowdebug build 10-internal+0-adhoc.hunterzhao.OpenJDK10, mixed mode)
```


本篇文章至此结束，下篇介绍搭建OpenJDK10 DEBUG环境。