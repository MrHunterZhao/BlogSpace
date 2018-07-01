---
title: 【JVM源码探秘】深入理解Thread.run()底层实现
date: 2018-06-11 13:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

对于Java程序猿来说可以通过new java.lang.Thread.start()来启动一个线程，只需要将业务逻辑放在run()方法里即可，
如此高效且易用的Java线程在JVM层面是怎样的呢？本文将从源码角度深入解读。

<!-- more -->

# java.lang.Thread # start()

我们尝试去启动一个Java线程，调用`start()`方法，然后跟随源码逐渐dive into进去。

```java
public synchronized void start() {
    /**
     * This method is not invoked for the main method thread or "system"
     * group threads created/set up by the VM. Any new functionality added
     * to this method in the future may have to also be added to the VM.
     *
     * A zero status value corresponds to state "NEW".
     */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
     * so that it can be added to the group's list of threads
     * and the group's unstarted count can be decremented. */
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}
```


在启动一个线程时会调用`start0()`这个native方法，关于本地方法的注册请参照之前的文章[【JVM源码探秘】深入registerNatives()底层实现](/post/2018/04/06/hotspot-explore-register-natives/)

# Thread.c
Thread类源码位于`src/java.base/share/native/libjava/Thread.c`，可见，start0对应`JVM_StartThread`

```c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};
```

# jvm.cpp # JVM_StartThread()
JVM_StartThread方法位于`src/hotspot/share/prims/jvm.cpp`

```c
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  bool throw_illegal_thread_state = false;

  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {
    // 获取互斥锁
    MutexLocker mu(Threads_lock);

    // 线程状态检查，确保尚未启动
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
    
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));

      // 创建本地线程
      NOT_LP64(if (size > SIZE_MAX) size = SIZE_MAX;)
      size_t sz = size > 0 ? (size_t) size : 0;

      // ===============================================================
      // 创建C++级别的本地线程，&thread_entry为线程run方法执行入口
      // ===============================================================
      native_thread = new JavaThread(&thread_entry, sz);

      // 检查该本地线程中是否包含OSThread，因为可能出现由于内存不足导致OSThread未创建成功的情况
      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        // ===============================================================
        // 准备Java本地线程，链接Java线程 <-> C++线程
        // ===============================================================
        native_thread->prepare(jthread);
      }
    }
  }

  // ===============================================================
  // 启动Java本地线程
  // ===============================================================
  Thread::start(native_thread);

JVM_END
```


# thread.cpp #  JavaThread::JavaThread()

代码native_thread = new JavaThread(&thread_entry, sz);用于创建JavaThread实例，位于src/hotspot/share/runtime/thread.cpp

```c
// C++级别Java线程构造方法
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread()
#if INCLUDE_ALL_GCS
                       , _satb_mark_queue(&_satb_mark_queue_set),
                       _dirty_card_queue(&_dirty_card_queue_set)
#endif // INCLUDE_ALL_GCS
{
  // 初始化实例变量
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  
  // ============================================= 
  // 设置Java执行线程入口，最终会调用
  // =============================================  
  set_entry_point(entry_point);
  // 创建系统级本地线程
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;

  // ============================================= 
  // 调用系统库创建线程
  // =============================================
  os::create_thread(this, thr_type, stack_sz);
}

```


# os_linux.cpp # os::create_thread()
通过OS创建线程，位于src/hotspot/os/linux/os_linux.cpp
```c
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {
  // 创建操作系统线程
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }
  
  // 把osthread状态设置为已分配
  osthread->set_state(ALLOCATED);
  // 绑定至JavaThread
  thread->set_osthread(osthread);
  // 初始化线程数形
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

  ThreadState state;
  {
    pthread_t tid;
    // ========================================================
    // 调用系统库创建线程，thread_native_entry为本地Java线程执行入口
    // ========================================================
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);

    if (ret != 0) {
      // Need to clean up stuff we've allocated so far
      thread->set_osthread(NULL);
      delete osthread;
      return false;
    }

    // Store pthread info into the OSThread
    osthread->set_pthread_id(tid);

    // Wait until child thread is either initialized or aborted
    {
      Monitor* sync_with_child = osthread->startThread_lock();
      MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
      while ((state = osthread->get_state()) == ALLOCATED) {
        sync_with_child->wait(Mutex::_no_safepoint_check_flag);
      }
    }
  }
  
  return true;
}
```

# os_linux.cpp # *thread_native_entry()
thread_native_entry为本地Java线程执行入口
```c
// 线程执行入口
// Thread start routine for all newly created threads
static void *thread_native_entry(Thread *thread) {

  // 初始化当前线程，把当前线程加入到TLS里
  thread->initialize_thread_current();

  OSThread* osthread = thread->osthread();
  // 获取同步锁
  Monitor* sync = osthread->startThread_lock();
  osthread->set_thread_id(os::current_thread_id());

  // handshaking with parent thread
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);

    // notify parent thread
    osthread->set_state(INITIALIZED);
    sync->notify_all();

    // 等待调用os::start_thread()，然后继续执行
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }

  

  // =======================================================
  // 调用JavaThread的run方法以便触发执行java.lang.Thread.run()
  // =======================================================
  thread->run();

  return 0;
}
```

# thread.cpp # Thread::start()
初始化工作完成之后当前线程wait，等待调用`Thread::start(native_thread);`
```c
void Thread::start(Thread* thread) {

  if (!DisableStartThread) {
    if (thread->is_Java_thread()) {

      // 设置线程状态为RUNNABLE
      java_lang_Thread::set_thread_status(((JavaThread*)thread)->threadObj(),
                                          java_lang_Thread::RUNNABLE);
    }
    // 启动本地线程
    os::start_thread(thread);
  }
}
```

# os.cpp # os::start_thread()
将java.lang.Thread对象设为RUNNABLE后启动本地线程，位于src/hotspot/share/runtime/os.cpp

```c
void os::start_thread(Thread* thread) {
  // guard suspend/resume
  MutexLockerEx ml(thread->SR_lock(), Mutex::_no_safepoint_check_flag);
  OSThread* osthread = thread->osthread();
  // osthread状态设为运行中
  osthread->set_state(RUNNABLE);
  // 最终启动线程
  pd_start_thread(thread);
}
```


# os_linux.cpp # os::pd_start_thread()
最终启动线程，位于src/hotspot/os/linux/os_linux.cpp，通知子线程JavaThread继续往下执行

```c
void os::pd_start_thread(Thread* thread) {
  OSThread * osthread = thread->osthread();
  assert(osthread->get_state() != INITIALIZED, "just checking");
  Monitor* sync_with_child = osthread->startThread_lock();
  MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
  // 通知子线程继续往下执行
  sync_with_child->notify();
}
```


# thread.cpp # JavaThread::run()

这里调用JavaThread的run方法以便执行java.lang.Thread.run()用户逻辑代码，位于src/hotspot/share/runtime/thread.cpp

```c
void JavaThread::run() {


  // 执行run方法前的初始化和缓存工作
  this->initialize_tlab();

  ...

  // 通知JVMTI
  if (JvmtiExport::should_post_thread_life()) {
    JvmtiExport::post_thread_start(this);
  }

  EventThreadStart event;
  if (event.should_commit()) {
    event.set_thread(THREAD_TRACE_ID(this));
    event.commit();
  }

  // ==================================================
  // 执行Java级别Thread类run()方法内容
  // ==================================================
  thread_main_inner();

}


void JavaThread::thread_main_inner() {

  if (!this->has_pending_exception() &&
      !java_lang_Thread::is_stillborn(this->threadObj())) {
    {
      ResourceMark rm(this);
      this->set_native_thread_name(this->get_thread_name());
    }
    HandleMark hm(this);

    // ==========================================
    // 执行线程入口java.lang.Thread # run()方法
    // ==========================================
    this->entry_point()(this, this);
  }

  DTRACE_THREAD_PROBE(stop, this);

  // 退出并释放空间
  this->exit(false);
  // 释放资源
  delete this;
}
```

# jvm.cpp # thread_entry()

最终执行实例化JavaThread时设置的入口方法entry_point，代表了Java代码级别Java线程执行入口，
这里通过JavaCalls组件调用java.lang.Thread.run()方法，执行真正的用户逻辑代码。

```c
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  // 执行Java调用
  JavaCalls::call_virtual(&result,
                          obj,
                          SystemDictionary::Thread_klass(),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
```

