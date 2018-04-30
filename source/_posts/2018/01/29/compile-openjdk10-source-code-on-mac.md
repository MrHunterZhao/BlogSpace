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

以前尝试编译过OpenJDK7、8和9，由于源代码中存在诸多BUG，导致各种编译问题，解决来解决去，还是很难编译通过，很是不爽。

今天抱着试试看的态度又服用了一个疗程，拿起桌上的OpenJDK10，反手就是一次编译，居然像德芙一样丝滑，一次通过。

<!-- more -->
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
此处需要耐心等待一段时间，必要情况下需要多尝试几次才能拉去成功。
为了方便我把源码上传到了GitHub上一份，所以也可以直接克隆
```
$ git clone https://github.com/MrHunterZhao/OpenJDK10.git
```



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


# 测试

```bash
$ ./build/macosx-x86_64-normal-server-slowdebug/jdk/bin/java -version
openjdk version "10-internal"
OpenJDK Runtime Environment (slowdebug build 10-internal+0-adhoc.hunterzhao.OpenJDK10)
OpenJDK 64-Bit Server VM (slowdebug build 10-internal+0-adhoc.hunterzhao.OpenJDK10, mixed mode)
```


本篇文章至此结束，下篇介绍搭建OpenJDK10 DEBUG环境。