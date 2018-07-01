---
title: 【JVM源码探秘】JVM线程模型概览
date: 2018-05-20 15:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)


HotSpot中的线程模型是Java线程（java.lang.Thread）与本地操作系统线程一一映射，本地线程在Java线程启动（调用start()）时创建，
并在终止时回收。操作系统负责调度本地线程给可用的CPU来执行。Java线程优先级和操作系统优先级之间的关系是相当复杂的，并且因操作系统而异。


<!-- more -->

# JVM线程类结构

JVM中线程模型是相当重要的一部分，如执行虚拟机指令的VM线程、执行内存垃圾回收的GC线程、采集虚拟机内存及CPU状况的监控线程，
还有专门用于执行用户线程的Java线程，如此多的线程之间是怎样一种联系和区别，我们一起来看一下。

JVM内部主要的线程主要分为以下几类：

```c
// ==================================================
//              线程实现类结构
// ==================================================

// Class hierarchy
// - Thread
//   - NamedThread               支持命名的非Java线程
//     - VMThread                   VM原始线程，用于执行VM操作
//     - ConcurrentGCThread         并发GC线程
//     - WorkerThread               工作线程
//       - GangWorker               一组线程，类似线程池
//       - GCTaskThread             GC任务线程
//   - JavaThread                C++层面的Java线程实现
//     - various subclasses eg CompilerThread, ServiceThread
//                                  各种子类，如：编译器线程，服务线程
//   - WatcherThread             监视器线程，用于模拟计时器中断
```

比较重要的还有OSThread操作系统层面的本地线程并且包含跟踪线程状态所需的附加操作系统级信息。OSThread还包含一个特定于平台的“句柄”，用于标识操作系统的实际线程


# 线程的创建与销毁
在JVM内部产生一个线程的基本方法有两种
- 调用java.lang.Thread的start()方法
- 通过JNI attach到一个已经存在的本地线程上

在java.lang.Thread启动时，JVM会分别创建一个相关联的JavaThread和OSThread对象，最终创建本地线程。 
在准备完所有的VM状态（例如线程本地存储和分配缓冲区，同步对象等等）之后，本地线程启动。 本地线程完成初始化，
然后执行启动方法，该方法导致执行java.lang.Thread对象的run()方法，然后在返回时在处理任何未捕获的异常之后
终止该线程，并且交互 与VM一起检查此线程的终止是否需要终止整个VM。 线程终止释放所有分配的资源，从一组已知线程中
删除JavaThread，调用OSThread和JavaThread的析构函数，并最终在初始启动方法完成时停止执行。


# C++层面的JavaThread状态
源码位于`src/hotspot/share/utilities/globalDefinitions.hpp`

```c
enum JavaThreadState {
  _thread_uninitialized     =  0, // should never happen (missing initialization)
  _thread_new               =  2, // just starting up, i.e., in process of being initialized
  _thread_new_trans         =  3, // corresponding transition state (not used, included for completness)
  _thread_in_native         =  4, // running in native code
  _thread_in_native_trans   =  5, // corresponding transition state
  _thread_in_vm             =  6, // running in VM
  _thread_in_vm_trans       =  7, // corresponding transition state
  _thread_in_Java           =  8, // running in Java or in stub code
  _thread_in_Java_trans     =  9, // corresponding transition state (not used, included for completness)
  _thread_blocked           = 10, // blocked in vm
  _thread_blocked_trans     = 11, // corresponding transition state
  _thread_max_state         = 12  // maximum thread state+1 - used for statistics allocation
};
```
最主要的有四个：
- _thread_new         : 刚启动但还没有初始化
- _thread_in_native   : 在执行本地代码
- _thread_in_vm       : 在执行JVM本身的代码
- _thread_in_Java     : 在执行解释的或编译的Java代码



# Java语言层面的线程状态
源码位于`src/hotspot/share/classfile/javaClasses.hpp`

这些线程状态主要用于JVMTI和MANGEMENT状态输出。


```c
enum ThreadStatus {
    NEW                      = 0,
    RUNNABLE                 = JVMTI_THREAD_STATE_ALIVE +          // runnable / running
                               JVMTI_THREAD_STATE_RUNNABLE,
    SLEEPING                 = JVMTI_THREAD_STATE_ALIVE +          // Thread.sleep()
                               JVMTI_THREAD_STATE_WAITING +
                               JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT +
                               JVMTI_THREAD_STATE_SLEEPING,
    IN_OBJECT_WAIT           = JVMTI_THREAD_STATE_ALIVE +          // Object.wait()
                               JVMTI_THREAD_STATE_WAITING +
                               JVMTI_THREAD_STATE_WAITING_INDEFINITELY +
                               JVMTI_THREAD_STATE_IN_OBJECT_WAIT,
    IN_OBJECT_WAIT_TIMED     = JVMTI_THREAD_STATE_ALIVE +          // Object.wait(long)
                               JVMTI_THREAD_STATE_WAITING +
                               JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT +
                               JVMTI_THREAD_STATE_IN_OBJECT_WAIT,
    PARKED                   = JVMTI_THREAD_STATE_ALIVE +          // LockSupport.park()
                               JVMTI_THREAD_STATE_WAITING +
                               JVMTI_THREAD_STATE_WAITING_INDEFINITELY +
                               JVMTI_THREAD_STATE_PARKED,
    PARKED_TIMED             = JVMTI_THREAD_STATE_ALIVE +          // LockSupport.park(long)
                               JVMTI_THREAD_STATE_WAITING +
                               JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT +
                               JVMTI_THREAD_STATE_PARKED,
    BLOCKED_ON_MONITOR_ENTER = JVMTI_THREAD_STATE_ALIVE +          // (re-)entering a synchronization block
                               JVMTI_THREAD_STATE_BLOCKED_ON_MONITOR_ENTER,
    TERMINATED               = JVMTI_THREAD_STATE_TERMINATED
  };
```

- SLEEPING              调用Thread.sleep()进入
- IN_OBJECT_WAIT        调用Object.wait()进入
- IN_OBJECT_WAIT_TIMED  调用Object.wait(long)进入
- PARKED                调用LockSupport.park()进入
- PARKED_TIMED          调用LockSupport.park(long)进入


# Java线程状态名称

```c
const char* java_lang_Thread::thread_status_name(oop java_thread) {
  assert(_thread_status_offset != 0, "Must have thread status");
  ThreadStatus status = (java_lang_Thread::ThreadStatus)java_thread->int_field(_thread_status_offset);
  switch (status) {
    case NEW                      : return "NEW";
    case RUNNABLE                 : return "RUNNABLE";
    case SLEEPING                 : return "TIMED_WAITING (sleeping)";
    case IN_OBJECT_WAIT           : return "WAITING (on object monitor)";
    case IN_OBJECT_WAIT_TIMED     : return "TIMED_WAITING (on object monitor)";
    case PARKED                   : return "WAITING (parking)";
    case PARKED_TIMED             : return "TIMED_WAITING (parking)";
    case BLOCKED_ON_MONITOR_ENTER : return "BLOCKED (on object monitor)";
    case TERMINATED               : return "TERMINATED";
    default                       : return "UNKNOWN";
  };
}
```


# 实例验证

随便实现一个Java线程并在内部sleep，


```java
public class Main {

    public static void main(String[] args) throws IOException {

        Thread t = new DemoThread();
        // 启动线程
        t.start();
        System.in.read();
    }
    
    static class DemoThread extends Thread {

        @Override
        public void run() {
            try {
                TimeUnit.HOURS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

然后通过jstack命令获取其运行状态：


```bash
$ jps
82005 Jps
81869 Main
81868 Launcher
```

```bash
$ jstack 81869
2018-07-02 00:28:52
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.151-b12 mixed mode):

"Attach Listener" #12 daemon prio=9 os_prio=31 tid=0x00007f99af000000 nid=0x1307 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007f99ae817000 nid=0x5a03 waiting on condition [0x0000700011471000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.workholiday.demoapp.Main$DemoThread.run(Main.java:30)

"Service Thread" #10 daemon prio=9 os_prio=31 tid=0x00007f99ae800800 nid=0x5603 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE


...


"Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007f99ab814000 nid=0x3903 in Object.wait() [0x0000700010ad3000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x000000076ab08ec8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x000000076ab08ec8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"main" #1 prio=5 os_prio=31 tid=0x00007f99ae802000 nid=0x1c03 runnable [0x000070000ffb2000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileInputStream.readBytes(Native Method)
	at java.io.FileInputStream.read(FileInputStream.java:255)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
	- locked <0x000000076ab20660> (a java.io.BufferedInputStream)
	at com.workholiday.demoapp.Main.main(Main.java:19)
```

通过日志输出会发现线程中的状态
- RUNNABLE
- TIMED_WAITING (sleeping)
- WAITING (on object monitor)

正是通过java_lang_Thread::thread_status_name()方法获取的。



# Reference
[HotSpot Threading Model](http://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Thread%20Management%7Coutline)