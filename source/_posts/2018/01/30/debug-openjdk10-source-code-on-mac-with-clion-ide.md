---
title: 【JVM源码探秘】在Mac上搭建OpenJDK10源码调试环境
date: 2018-01-30 01:08:18
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

前面文章已经介绍了如何[在Mac上编译OpenJDK10源码](/post/2018/01/29/compile-openjdk10-source-code-on-mac/)，拥有了自己的JDK版本，

为了深入了解Java实例的创建、初始化和执行流程以及内部实现原理，DEBUG是必不可少的必杀技。

所以，本篇文章继续介绍在Mac上搭建OpenJDK10源码调试环境，黑喂狗。

<!-- more -->
# 软件环境
- OS: macOS Sierra 10.12
- IDE: Clion 2018.1
- Code: OpenJDK 10

# 下载IDE
从JetBrains官网下载[Clion](https://www.jetbrains.com/clion/)，安装。

# 导入项目
打开Clion依次选择`File` > `Import Project`

![import-hotspot](/images/post/2018/01/30/import-hotspot-src.jpg)


# 编辑配置
如下图编辑DEBUG配置信息
1. `Executable` 选择之前build出的镜像里的java可执行文件（i.e. build/macosx-x86_64-normal-server-slowdebug/jdk/bin/java）
2. `Program arguments` 填写-version，输出Java版本
3. `Before launch` 注意：这里一定要移除Build，否则会报错无法调试


![edit-configuration](/images/post/2018/01/30/edit-configuration.jpg)

# 调试源码

在`hotspot/share/runtime/thread.cpp`文件的`Threads::create_vm`方法内部打断点，

点击DEBUG按钮，不出意外会发现进入如下界面，congrats！

![openjdk](/images/post/2018/01/30/debug-openjdk10-with-clion-ide.jpg)

接下来，泡杯咖啡，Step by step慢慢DEBUG吧，后面的文章将陆续介绍JVM创建及初始化流程。
