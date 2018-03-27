---
title: 在Linux上编译OpenJDK7源码
date: 2014-11-08 05:50:46
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - CentOS
---


# 软件环境
- OS：CentOS 6.5
- JDK: OpenJDK-7u40

# 准备工作

## 下载源码包
[OpenJDK Source Releases](https://jdk7.java.net/source.html)

## 解压源码包
```bash
# unzip openjdk-7u40-fcs-src-b43-26_aug_2013.zip
```

- hotspot 虚拟机实现，大部分是C/C++代码
- jdk Java核心类库目录，位于 jdk/src/share/classes
- langtools 一些编译工具

## 安装依赖
```bash
yum -y install gcc gcc-c++ alsa-lib alsa-lib-devel libXrender libXrender-devel libXi-devel libXt-devel libXtst-devel cups cups-devel
```

<!-- more -->

## 安装freetype
```bash
# ./configure && make && make install
```

## 安装Ant
```bash
ln -s /usr/local/apache-ant-1.9.4/bin/ant /usr/bin/ant
```

## 安装JDK（如果已有则不必重新安装） 
`Sun JDK或Open JDK均可，过程略.`

# 环境配置
编辑文件`vim ~/.bash_profile`加入以下变量

```
export LANG="C"
export ALT_BOOTDIR="/usr/java/jdk1.7.0_67/"
export ANT_HOME="/usr/local/apache-ant-1.9.4"
export ALT_FREETYPE_HEADERS_PATH="/usr/local/include/freetype2"
export ALT_FREETYPE_LIB_PATH="/usr/local/lib"
export ALLOW_DOWNLOADS=true
export SKIP_DEBUG_BUILD=false
export SKIP_FASTDEBUG_BUILD=true
export DEBUG_NAME=debug
unset JAVA_HOME
unset CLASSPATH
```

使变量生效
```
# source ~/.bash_profile
```

# 编译源码

## 测试环境是否健全
```
# make sanity
```

## 如果输出以下内容则表示通过，可以进行编译。
```
Sanity check passed.
```

## 开始编译
```
# make ARCH_DATA_MODEL=64
```

## 看到如下输出则为编译成功
```
>>>Finished making images @ Sat Nov  8 00:45:16 EST 2014 ...
make[2]: Leaving directory `/usr/local/openjdk/jdk/make'
########################################################################
##### Leaving jdk for target(s) sanity all docs images             #####
########################################################################
##### Build time 00:10:15 jdk for target(s) sanity all docs images #####
########################################################################

#-- Build times ----------
Target debug_build
Start 2014-11-08 00:26:41
End   2014-11-08 00:45:16
00:02:11 corba
00:04:36 hotspot
00:00:24 jaxp
00:00:30 jaxws
00:10:15 jdk
00:00:39 langtools
00:18:35 TOTAL
-------------------------
make[1]: Leaving directory `/usr/local/openjdk'
[root@BobServerStation openjdk]#
```

# 测试验证

## 一个测试类
```java
/**
* Author: HunterZhao
* Date: 2014-11-08
*/
public class Test{
  public static void main(String[] args){
       System.out.println("Hello OpenJDK~");
  }
}
```

使用刚生成的JDK编译
```
# ./build/linux-amd64/bin/javac Test.java
```
在当前目录下会生成Test.class文件，然后运行便会看到输出。
```
# ./build/linux-amd64/bin/java Test
Hello OpenJDK~
```

## 另一个测试类
进入目录`jdk/src/share/classes/java/io`，然后修改`PrintStream.java`

```java
/**
   * Prints a string.  If the argument is <code>null</code> then the string
   * <code>"null"</code> is printed.  Otherwise, the string's characters are
   * converted into bytes according to the platform's default character
   * encoding, and these bytes are written in exactly the manner of the
   * <code>{@link #write(int)}</code> method.
   *
   * @param      s   The <code>String</code> to be printed
   */
  public void print(String s) {
      if (s == null) {
          s = "null";
      }
      s = s + " This is OpenJdk7 compiled by Bob.Z!!!";  // 重新赋值
      write(s);
  }
```

接下来重新编译JDK，重新编译刚才的Test.java文件并运行会看到如下输出：


![compile-openjdk7](/images/post/2014/11/08/compile-openjdk7-on-mac.png)

Enjoy them~


# Reference
- OpenJDK: https://jdk7.java.net/
- freetype: http://download.savannah.gnu.org/releases/freetype/
- Ant: http://archive.apache.org/dist/ant/binaries/