---
title: 在Linux下搭建OpenJDK7源码调试环境
date: 2014-11-19 17:17:53
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - Linux
    - CentOS
---

在前面的文章中博主已经介绍过如何在Linux下编译OpenJDK7源码，现继续介绍如何在Linux下搭建基于eclipse IDE的Hotspot源码调试环境。鉴于网上关于JVM源码调试方面的文章寥寥无几并且内容参差不全，本文将把博主摸索数天的经验及成果以图文形式详细介绍Hotspot的debug过程。

# 软件环境
- OS：CentOS 6.5
- OpenJDK: OpenJDK-7u40
- JDK Version：openjdk-7u40-fcs-src-b43-26_aug_2013
- IDE：eclipse-cpp-luna-SR1-linux-gtk-x86_64

# 导入源码
首先解压JDK源码包至`/usr/local`目录，然后启动eclipse，依次选择`File` > `New` > `Makefile Project with Existing Code`(如果没有则在Other里找)

![debug-openjdk7](/images/post/2014/11/19/debug-jvm-on-linux-1.jpg)

# 配置环境变量
定位到项目名右键 > `Properties >C/C++ Build`需要修改两个地方：
- 将Builder里`Use default build command`的对勾去掉，填入参数`ARCH_DATA_MODEL=64`
- 将`Build location`的`Build directory`追加上`/make`，最终是`${workspace_lc:/hotspot}/make`，目的是告诉make编译器到该目录下寻找编译文件Makefile。

<!-- more -->


![debug-openjdk7](/images/post/2014/11/19/debug-jvm-on-linux-2.jpg)

# 开始编译
选择菜单栏`Project` > `Build Project`，如果运气不差的话会看到已经开始build了，沏杯咖啡慢慢等吧（首次build大概需要10-20m）。

![debug-openjdk7](/images/post/2014/11/19/debug-jvm-on-linux-3.jpg)

部分LOG信息：

```bash
make[4]: Entering directory `/usr/local/openjdk/hotspot/build/linux/linux_amd64_compiler2/fastdebug'
echo "**NOTICE** Dtrace support disabled: "/usr/include/sys/sdt.h not found""
**NOTICE** Dtrace support disabled: /usr/include/sys/sdt.h not found
make[4]: Leaving directory `/usr/local/openjdk/hotspot/build/linux/linux_amd64_compiler2/fastdebug'
All done.
make[3]: Leaving directory `/usr/local/openjdk/hotspot/build/linux/linux_amd64_compiler2/fastdebug'
cd linux_amd64_compiler2/fastdebug && ./test_gamma
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
OpenJDK 64-Bit Server VM (build 24.0-b56-internal-fastdebug, mixed mode)

 1. A1 B5 C8 D6 E3 F7 G2 H4 
 2. A1 B6 C8 D3 E7 F4 G2 H5 
 3. A1 B7 C4 D6 E8 F2 G5 H3 
 4. A1 B7 C5 D8 E2 F4 G6 H3 
 5. A2 B4 C6 D8 E3 F1 G7 H5 
 6. A2 B5 C7 D1 E3 F8 G6 H4

 -- 此处略去N行------


 Using java runtime at: /usr/java/jdk1.7.0_67/jre
make[2]: Leaving directory `/usr/local/openjdk/hotspot/build/linux'
make[1]: Leaving directory `/usr/local/openjdk/hotspot/make'
cd /usr/local/openjdk/hotspot/make; \
    make BUILD_FLAVOR=fastdebug VM_TARGET=fastdebug1 generic_build1 
INFO: ENABLE_FULL_DEBUG_SYMBOLS=1
INFO: /usr/bin/objcopy cmd found so will create .debuginfo files.
INFO: STRIP_POLICY=min_strip
INFO: ZIP_DEBUGINFO_FILES=1
make[1]: Entering directory `/usr/local/openjdk/hotspot/make'
mkdir -p /usr/local/openjdk/hotspot/build/linux
No compiler1 (fastdebug1) for ARCH_DATA_MODEL=64
make[1]: Leaving directory `/usr/local/openjdk/hotspot/make'
make BUILD_FLAVOR=fastdebug VM_SUBDIR=fastdebug \
      EXPORT_SUBDIR=/fastdebug \
      generic_export
INFO: ENABLE_FULL_DEBUG_SYMBOLS=1
INFO: /usr/bin/objcopy cmd found so will create .debuginfo files.
INFO: STRIP_POLICY=min_strip
INFO: ZIP_DEBUGINFO_FILES=1
make[1]: Entering directory `/usr/local/openjdk/hotspot/make'
make[1]: Nothing to be done for `generic_export'.
make[1]: Leaving directory `/usr/local/openjdk/hotspot/make'

16:02:09 Build Finished (took 8s.216ms)
```

# 配置DEBUG环境
- 编译成功之后就可以测试了，需配置如下几步
点选菜单栏`Run` > `Debug Configurations` > `New launch configuration`，在`C/C++ Application`里填入`/usr/local/openjdk/hotspot/build/linux/linux_amd64_compiler2/fastdebug/gamma`，Project选择当前项目。
- 在`Argument tab`页里`Program arguments`填入`-version`
- 在`Environment tab`页里`Environment variables to set`填入`JAVA_HOME | /usr/java/jdk1.7.0.67`
- 在`Common tab`页里勾选`Debug`
![debug-openjdk7](/images/post/2014/11/19/debug-jvm-on-linux-4.jpg)

# 进入调试模式
配置完毕后，点击Debug即可进入调试模式，Hotspot内部一览无余，
![debug-openjdk7](/images/post/2014/11/19/debug-jvm-on-linux-5.jpg)
