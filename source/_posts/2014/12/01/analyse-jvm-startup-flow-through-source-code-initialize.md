---
title: 【Hotspot源码分析】从Hotpost源码角度深入分析Java程序启动过程-初始化  
date: 2014-12-01 04:20:08
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
---

接上篇文章从Hotpost源码角度深入分析Java程序启动过程-创建 ，本文将继续介绍JVM启动过程的初始化部分。

在上篇文章中在执行LoadJavaVM方法的时候将libjvm.so内的方法`JNI_CreateJavaVM`和`JNI_GetDefaultJavaVMInitArgs`符号引用挂载到了结构体`InvocationFunctions`上，并且在执行InitializeJVM方法的时候进行了调用。

这里执行了JNI调用`JNI_CreateJavaVM`，文件位于`hotspot/src/share/vm/prims/jni.cpp`。方法内容如下：

```c
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {

   // 略去部分非重要内容

  /**
   * Certain errors during initialization are recoverable and do not
   * prevent this method from being called again at a later time
   * (perhaps with different arguments).  However, at a certain
   * point during initialization if an error occurs we cannot allow
   * this function to be called again (or it will crash).  In those
   * situations, the 'canTryAgain' flag is set to false, which atomically
   * sets safe_to_recreate_vm to 1, such that any new call to
   * JNI_CreateJavaVM will immediately fail using the above logic.
   */
  bool can_try_again = true;
  //========================================
  // 通过Threads模块初始化VM并创建VM线程
  //========================================
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
  if (result == JNI_OK) {
    JavaThread *thread = JavaThread::current();
    /* thread is thread_in_vm here */
    *vm = (JavaVM *)(&main_vm);
    *(JNIEnv**)penv = thread->jni_environment();

    // Tracks the time application was running before GC
    RuntimeService::record_application_start();

    // Notify JVMTI
    if (JvmtiExport::should_post_thread_life()) {
       JvmtiExport::post_thread_start(thread);
    }

    EventThreadStart event;
    if (event.should_commit()) {
      event.set_javalangthread(java_lang_Thread::thread_id(thread->threadObj()));
      event.commit();
    }

    // 略去部分内容

  return result;
}
```

<!-- more -->

这里调用了`hotspot/src/share/vm/runtime/thread.cpp`的`create_vm`方法：

```c
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {

  extern void JDK_Version_init();

  // Check version
  if (!is_supported_jni_version(args->version)) return JNI_EVERSION;

  // Initialize the output stream module
  // 初始化输出流
  ostream_init();

  // Process java launcher properties.
  // 处理Java启动参数，如-Dsun.java.launcher*
  Arguments::process_sun_java_launcher_properties(args);

  // Initialize the os module before using TLS
  // 初始化操作系统模块，如页大小，处理器数量，系统时钟等
  os::init();

  // Initialize system properties.
  // 初始化系统属性，其中分为【可读属性】和【可读写属性】
  // 可读属性：
  // java.vm.specification.name
  // java.vm.version
  // java.vm.name
  // java.vm.info
  // 可读写属性：
  // java.ext.dirs
  // java.endorsed.dirs
  // sun.boot.library.path
  // java.library.path
  // java.home
  // sun.boot.class.path
  // java.class.path
  Arguments::init_system_properties();

  // So that JDK version can be used as a discrimintor when parsing arguments
  JDK_Version_init();

  // Update/Initialize System properties after JDK version number is known
  // 设置java.vm.specification.vendor属性（1.6之前是Sun Microsystems Inc. 1.7之后是Oracle Corporation）
  // 设置java.vm.specification.version和java.vm.vendor属性
  Arguments::init_version_specific_system_properties();

  // Parse arguments
  // 解析启动参数，如-XX:Flags=、-XX:+PrintVMOptions、-XX:+PrintFlagsInitial etc.
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;

  if (PauseAtStartup) {
    os::pause();
  }

#ifndef USDT2
  HS_DTRACE_PROBE(hotspot, vm__init__begin);
#else /* USDT2 */
  HOTSPOT_VM_INIT_BEGIN();
#endif /* USDT2 */

  // Record VM creation timing statistics
  TraceVmCreationTime create_vm_timer;
  create_vm_timer.start();

  // Timing (must come after argument parsing)
  TraceTime timer("Create VM", TraceStartupTime);

  // Initialize the os module after parsing the args
  jint os_init_2_result = os::init_2();
  if (os_init_2_result != JNI_OK) return os_init_2_result;

  // intialize TLS
  ThreadLocalStorage::init();

  // Bootstrap native memory tracking, so it can start recording memory
  // activities before worker thread is started. This is the first phase
  // of bootstrapping, VM is currently running in single-thread mode.
  MemTracker::bootstrap_single_thread();

  // Initialize output stream logging
  // 初始化GC日志输出流，用来处理-Xloggc参数
  ostream_init_log();

  // Convert -Xrun to -agentlib: if there is no JVM_OnLoad
  // Must be before create_vm_init_agents()
  if (Arguments::init_libraries_at_startup()) {
    convert_vm_init_libraries_to_agents();
  }

  // Launch -agentlib/-agentpath and converted -Xrun agents
  // 加载agent库
  if (Arguments::init_agents_at_startup()) {
    create_vm_init_agents();
  }

  // Initialize Threads state
  _thread_list = NULL;
  _number_of_threads = 0;
  _number_of_non_daemon_threads = 0;

  // Initialize global data structures and create system classes in heap
  // 初始化全局数据数据结构及系统类，包括：
  // 初始化Java基础类型
  // 初始化时间队列
  // 初始化锁
  // 初始化chunkpool
  // 初始化性能数据统计模块
  vm_init_globals();

  // Attach the main thread to this os thread
  JavaThread* main_thread = new JavaThread();
  main_thread->set_thread_state(_thread_in_vm);
  main_thread->record_stack_base_and_size();
  main_thread->initialize_thread_local_storage();

  main_thread->set_active_handles(JNIHandleBlock::allocate_block());

  if (!main_thread->set_as_starting_thread()) {
    vm_shutdown_during_initialization(
      "Failed necessary internal allocation. Out of swap space");
    delete main_thread;
    *canTryAgain = false; // don't let caller call JNI_CreateJavaVM again
    return JNI_ENOMEM;
  }

  // Enable guard page *after* os::create_main_thread(), otherwise it would
  // crash Linux VM, see notes in os_linux.cpp.
  main_thread->create_stack_guard_pages();

  // Initialize Java-Level synchronization subsystem
  ObjectMonitor::Initialize() ;

  // Second phase of bootstrapping, VM is about entering multi-thread mode
  MemTracker::bootstrap_multi_thread();

  // Initialize global modules
  // ========================================
  // IMPORTANT!!! 初始化全局模块
  // ========================================
  jint status = init_globals();
  if (status != JNI_OK) {
    delete main_thread;
    *canTryAgain = false; // don't let caller call JNI_CreateJavaVM again
    return status;
  }

  // Should be done after the heap is fully created
  main_thread->cache_global_variables();

  HandleMark hm;

  { MutexLocker mu(Threads_lock);
    Threads::add(main_thread);
  }

  // Any JVMTI raw monitors entered in onload will transition into
  // real raw monitor. VM is setup enough here for raw monitor enter.
  JvmtiExport::transition_pending_onload_raw_monitors();

  // Fully start NMT
  MemTracker::start();

  // Create the VMThread
  { TraceTime timer("Start VMThread", TraceStartupTime);
    VMThread::create();
    Thread* vmthread = VMThread::vm_thread();

    if (!os::create_thread(vmthread, os::vm_thread))
      vm_exit_during_initialization("Cannot create VM thread. Out of system resources.");

    // Wait for the VM thread to become ready, and VMThread::run to initialize
    // Monitors can have spurious returns, must always check another state flag
    {
      MutexLocker ml(Notify_lock);
      os::start_thread(vmthread);
      while (vmthread->active_handles() == NULL) {
        Notify_lock->wait();
      }
    }
  }

  assert (Universe::is_fully_initialized(), "not initialized");
  if (VerifyBeforeGC && VerifyGCStartAt == 0) {
    Universe::heap()->prepare_for_verify();
    Universe::verify();   // make sure we're starting with a clean slate
  }

  EXCEPTION_MARK;

  // At this point, the Universe is initialized, but we have not executed
  // any byte code.  Now is a good time (the only time) to dump out the
  // internal state of the JVM for sharing.

  if (DumpSharedSpaces) {
    Universe::heap()->preload_and_dump(CHECK_0);
    ShouldNotReachHere();
  }

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_start_phase();

  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_vm_start();

  {
    TraceTime timer("Initialize java.lang classes", TraceStartupTime);

    if (EagerXrunInit && Arguments::init_libraries_at_startup()) {
      create_vm_init_libraries();
    }

    if (InitializeJavaLangString) {
      initialize_class(vmSymbols::java_lang_String(), CHECK_0);
    } else {
      warning("java.lang.String not initialized");
    }

    // Initialize java_lang.System (needed before creating the thread)
    if (InitializeJavaLangSystem) {
      initialize_class(vmSymbols::java_lang_System(), CHECK_0);
      initialize_class(vmSymbols::java_lang_ThreadGroup(), CHECK_0);
      Handle thread_group = create_initial_thread_group(CHECK_0);
      Universe::set_main_thread_group(thread_group());
      initialize_class(vmSymbols::java_lang_Thread(), CHECK_0);
      oop thread_object = create_initial_thread(thread_group, main_thread, CHECK_0);
      main_thread->set_threadObj(thread_object);
      // Set thread status to running since main thread has
      // been started and running.
      java_lang_Thread::set_thread_status(thread_object,
                                          java_lang_Thread::RUNNABLE);

      // The VM preresolve methods to these classes. Make sure that get initialized
      initialize_class(vmSymbols::java_lang_reflect_Method(), CHECK_0);
      initialize_class(vmSymbols::java_lang_ref_Finalizer(),  CHECK_0);
      // The VM creates & returns objects of this class. Make sure it's initialized.
      initialize_class(vmSymbols::java_lang_Class(), CHECK_0);
      call_initializeSystemClass(CHECK_0);

      // get the Java runtime name after java.lang.System is initialized
      JDK_Version::set_runtime_name(get_java_runtime_name(THREAD));
      JDK_Version::set_runtime_version(get_java_runtime_version(THREAD));
    } else {
      warning("java.lang.System not initialized");
    }

    // an instance of OutOfMemory exception has been allocated earlier
    if (InitializeJavaLangExceptionsErrors) {
      initialize_class(vmSymbols::java_lang_OutOfMemoryError(), CHECK_0);
      initialize_class(vmSymbols::java_lang_NullPointerException(), CHECK_0);
      initialize_class(vmSymbols::java_lang_ClassCastException(), CHECK_0);
      initialize_class(vmSymbols::java_lang_ArrayStoreException(), CHECK_0);
      initialize_class(vmSymbols::java_lang_ArithmeticException(), CHECK_0);
      initialize_class(vmSymbols::java_lang_StackOverflowError(), CHECK_0);
      initialize_class(vmSymbols::java_lang_IllegalMonitorStateException(), CHECK_0);
      initialize_class(vmSymbols::java_lang_IllegalArgumentException(), CHECK_0);
    } else {
      warning("java.lang.OutOfMemoryError has not been initialized");
      warning("java.lang.NullPointerException has not been initialized");
      warning("java.lang.ClassCastException has not been initialized");
      warning("java.lang.ArrayStoreException has not been initialized");
      warning("java.lang.ArithmeticException has not been initialized");
      warning("java.lang.StackOverflowError has not been initialized");
      warning("java.lang.IllegalArgumentException has not been initialized");
    }
  }


  initialize_class(vmSymbols::java_lang_Compiler(), CHECK_0);

  reset_vm_info_property(CHECK_0);

  quicken_jni_functions();

  // Must be run after init_ft which initializes ft_enabled
  if (TRACE_INITIALIZE() != JNI_OK) {
    vm_exit_during_initialization("Failed to initialize tracing backend");
  }

  // Set flag that basic initialization has completed. Used by exceptions and various
  // debug stuff, that does not work until all basic classes have been initialized.
  set_init_completed();

#ifndef USDT2
  HS_DTRACE_PROBE(hotspot, vm__init__end);
#else /* USDT2 */
  HOTSPOT_VM_INIT_END();
#endif /* USDT2 */

  // record VM initialization completion time
  // 向VM管理模块发送初始化完成信号
  Management::record_vm_init_completed();

  // Compute system loader. Note that this has to occur after set_init_completed, since
  // valid exceptions may be thrown in the process.
  // Note that we do not use CHECK_0 here since we are inside an EXCEPTION_MARK and
  // set_init_completed has just been called, causing exceptions not to be shortcut
  // anymore. We call vm_exit_during_initialization directly instead.
  // 载入classloader
  SystemDictionary::compute_java_system_loader(THREAD);
  if (HAS_PENDING_EXCEPTION) {
    vm_exit_during_initialization(Handle(THREAD, PENDING_EXCEPTION));
  }

#ifndef SERIALGC
  // Support for ConcurrentMarkSweep. This should be cleaned up
  // and better encapsulated. The ugly nested if test would go away
  // once things are properly refactored. XXX YSR
  if (UseConcMarkSweepGC || UseG1GC) {
    if (UseConcMarkSweepGC) {
      ConcurrentMarkSweepThread::makeSurrogateLockerThread(THREAD);
    } else {
      ConcurrentMarkThread::makeSurrogateLockerThread(THREAD);
    }
    if (HAS_PENDING_EXCEPTION) {
      vm_exit_during_initialization(Handle(THREAD, PENDING_EXCEPTION));
    }
  }
#endif // SERIALGC

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_live_phase();

  // Signal Dispatcher needs to be started before VMInit event is posted
  // 启动一个叫做“信号分发器”的线程用来处理进程间的信号
  // 比如通过jstack获取一个jvm实例的栈信息
  os::signal_init();

  // Start Attach Listener if +StartAttachListener or it can't be started lazily
  if (!DisableAttachMechanism) {
    AttachListener::vm_start();
    if (StartAttachListener || AttachListener::init_at_startup()) {
      AttachListener::init();
    }
  }

  // Launch -Xrun agents
  // Must be done in the JVMTI live phase so that for backward compatibility the JDWP
  // back-end can launch with -Xdebug -Xrunjdwp.
  if (!EagerXrunInit && Arguments::init_libraries_at_startup()) {
    create_vm_init_libraries();
  }

  // Notify JVMTI agents that VM initialization is complete - nop if no agents.
  JvmtiExport::post_vm_initialized();

  if (TRACE_START() != JNI_OK) {
    vm_exit_during_initialization("Failed to start tracing backend.");
  }

  if (CleanChunkPoolAsync) {
    Chunk::start_chunk_pool_cleaner_task();
  }

  // initialize compiler(s)
  CompileBroker::compilation_init();
  // 加载sun.management.Agent类并调用startAgent方法开启管理服务
  Management::initialize(THREAD);
  if (HAS_PENDING_EXCEPTION) {
    // management agent fails to start possibly due to
    // configuration problem and is responsible for printing
    // stack trace if appropriate. Simply exit VM.
    vm_exit(1);
  }

  if (Arguments::has_profile())       FlatProfiler::engage(main_thread, true);
  if (Arguments::has_alloc_profile()) AllocationProfiler::engage();
  if (MemProfiling)                   MemProfiler::engage();
  StatSampler::engage();
  if (CheckJNICalls)                  JniPeriodicChecker::engage();

  BiasedLocking::init();

  if (JDK_Version::current().post_vm_init_hook_enabled()) {
    call_postVMInitHook(THREAD);
    // The Java side of PostVMInitHook.run must deal with all
    // exceptions and provide means of diagnosis.
    if (HAS_PENDING_EXCEPTION) {
      CLEAR_PENDING_EXCEPTION;
    }
  }

  {
      MutexLockerEx ml(PeriodicTask_lock, Mutex::_no_safepoint_check_flag);
      // Make sure the watcher thread can be started by WatcherThread::start()
      // or by dynamic enrollment.
      WatcherThread::make_startable();
      // Start up the WatcherThread if there are any periodic tasks
      // NOTE:  All PeriodicTasks should be registered by now. If they
      //   aren't, late joiners might appear to start slowly (we might
      //   take a while to process their first tick).
      if (PeriodicTask::num_tasks() > 0) {
          WatcherThread::start();
      }
  }

  // Give os specific code one last chance to start
  os::init_3();

  create_vm_timer.end();
  return JNI_OK;
}
```


其中`init_globals()`方法位于`hotspot/src/share/vm/runtime/init.cpp`用来初始化全局模块:
```c
jint init_globals() {

  HandleMark hm;
  // 初始化各子系统的监控及管理服务
  // JMX、线程和同步子系统、类加载子系统的监控和管理
  management_init();
  // 初始化字节码表，如istore、iload、iadd
  bytecodes_init();
  // 类加载器初始化
  classLoader_init();
  // 代码缓存池初始化
  codeCache_init();
  // VM版本初始化
  VM_Version_init();
  // 系统初始化
  os_init_globals();
  stubRoutines_init1();
  // ============================
  // 初始化堆以及决定所使用GC策略
  // ============================
  jint status = universe_init();  // dependent on codeCache_init and
                                  // stubRoutines_init1
  if (status != JNI_OK)
    return status;
  // 初始化解析器
  interpreter_init();  // before any methods loaded
  invocationCounter_init();  // before any methods loaded
  // 初始化MarkSweep
  marksweep_init();
  accessFlags_init();
  // 初始化操作码模板表
  templateTable_init();
  InterfaceSupport_init();
  SharedRuntime::generate_stubs();
  // 初始化语法表及系统字典等
  universe2_init();  // dependent on codeCache_init and stubRoutines_init1
  // 初始化软引用时间戳表并设定软引用清除策略
  referenceProcessor_init();
  jni_handles_init();
  // 代码数据结构的必要性检查（仅限debug版本）
  vmStructs_init();
  vtableStubs_init();
  InlineCacheBuffer_init();
  // oracle编译器初始化（oracle编译器是一个编译器开关接口）
  compilerOracle_init();
  // 初始化编译策略（根据启动参数决定编译策略）
  compilationPolicy_init();
  compileBroker_init();
  VMRegImpl::set_regName();

  if (!universe_post_init()) {
    return JNI_ERR;
  }
  javaClasses_init();   // must happen after vtable initialization
  stubRoutines_init2(); // note: StubRoutines need 2-phase init

  // All the flags that get adjusted by VM_Version_init and os::init_2
  // have been set so dump the flags now.
  if (PrintFlagsFinal) {
    CommandLineFlags::printFlags(tty, false);
  }

  return JNI_OK;
}
```

其中`universe_init()`方法位于`hotspot/src/share/vm/memory/universe.cpp`

```c
jint universe_init() {
  assert(!Universe::_fully_initialized, "called after initialize_vtables");
  guarantee(1 << LogHeapWordSize == sizeof(HeapWord),
         "LogHeapWordSize is incorrect.");
  guarantee(sizeof(oop) >= sizeof(HeapWord), "HeapWord larger than oop?");
  guarantee(sizeof(oop) % sizeof(HeapWord) == 0,
            "oop size is not not a multiple of HeapWord size");
  TraceTime timer("Genesis", TraceStartupTime);
  GC_locker::lock();  // do not allow gc during bootstrapping
  JavaClasses::compute_hard_coded_offsets();

  // Get map info from shared archive file.
  if (DumpSharedSpaces)
    UseSharedSpaces = false;

  FileMapInfo* mapinfo = NULL;
  if (UseSharedSpaces) {
    mapinfo = NEW_C_HEAP_OBJ(FileMapInfo, mtInternal);
    memset(mapinfo, 0, sizeof(FileMapInfo));

    // Open the shared archive file, read and validate the header. If
    // initialization files, shared spaces [UseSharedSpaces] are
    // disabled and the file is closed.

    if (mapinfo->initialize()) {
      FileMapInfo::set_current_info(mapinfo);
    } else {
      assert(!mapinfo->is_open() && !UseSharedSpaces,
             "archive file not closed or shared spaces not disabled.");
    }
  }

  //===================================
  // 初始化堆
  // 包括创建及初始化各分区代，设定空间比例大小，回收策略等
  // 流程：根据启动参数决定使用的回收策略，初始化回收策略时会指定所使用的代规范，
  //       最后根据规范创建对应类型的回收堆。i.e.
  //      arguments -> policy -> spec -> heap
  //===================================
  jint status = Universe::initialize_heap();
  if (status != JNI_OK) {
    return status;
  }

  // We have a heap so create the methodOop caches before
  // CompactingPermGenGen::initialize_oops() tries to populate them.
  Universe::_finalizer_register_cache = new LatestMethodOopCache();
  Universe::_loader_addClass_cache    = new LatestMethodOopCache();
  Universe::_pd_implies_cache         = new LatestMethodOopCache();
  Universe::_reflect_invoke_cache     = new ActiveMethodOopsCache();

  if (UseSharedSpaces) {

    // Read the data structures supporting the shared spaces (shared
    // system dictionary, symbol table, etc.).  After that, access to
    // the file (other than the mapped regions) is no longer needed, and
    // the file is closed. Closing the file does not affect the
    // currently mapped regions.

    CompactingPermGenGen::initialize_oops();
    mapinfo->close();

  } else {
    SymbolTable::create_table();
    StringTable::create_table();
    ClassLoader::create_package_info_table();
  }

  return JNI_OK;
}
```


initialize_heap()方法如下：
```c
jint Universe::initialize_heap() {
  // 如果使用并行GC
  if (UseParallelGC) {
#ifndef SERIALGC
    // 回收堆类型使用并行回收堆
    Universe::_collectedHeap = new ParallelScavengeHeap();
#else  // SERIALGC
    fatal("UseParallelGC not supported in java kernel vm.");
#endif // SERIALGC

  } else if (UseG1GC) {
#ifndef SERIALGC
    // 如果使用G1回收，设定回收器策略和回收堆类型为G1CollectorPolicy和G1CollectedHeap
    G1CollectorPolicy* g1p = new G1CollectorPolicy();
    G1CollectedHeap* g1h = new G1CollectedHeap(g1p);
    Universe::_collectedHeap = g1h;
#else  // SERIALGC
    fatal("UseG1GC not supported in java kernel vm.");
#endif // SERIALGC

  } else {
    GenCollectorPolicy *gc_policy;
    // 使用串行回收
    if (UseSerialGC) {
      gc_policy = new MarkSweepPolicy();
    // 使用并发回收
    } else if (UseConcMarkSweepGC) {
#ifndef SERIALGC
      // 是否使用自适应策略
      // ASConcurrentMarkSweepPolicy继承自ConcurrentMarkSweepPolicy，
      if (UseAdaptiveSizePolicy) {
        gc_policy = new ASConcurrentMarkSweepPolicy();
      } else {
        gc_policy = new ConcurrentMarkSweepPolicy();
      }
#else   // SERIALGC
    fatal("UseConcMarkSweepGC not supported in java kernel vm.");
#endif // SERIALGC
    // 默认使用标记清除算法
    } else { // default old generation
      gc_policy = new MarkSweepPolicy();
    }
    // 回收策略类型体系图
    // AllocatedObj
    //    └── CHeapObj
    //        └── CollectorPolicy
    //            └── GenCollectorPolicy
    //                └── TwoGenerationCollectorPolicy
    //                    └── ConcurrentMarkSweepPolicy
    //                        └── ASConcurrentMarkSweepPolicy
    Universe::_collectedHeap = new GenCollectedHeap(gc_policy);
  }
  //===================================
  // 初始化堆空间
  // 这里调用GenCollectedHeap::initialize()方法， 真正向操作系统申请内存
  //===================================
  jint status = Universe::heap()->initialize();
  if (status != JNI_OK) {
    return status;
  }

#ifdef _LP64
  // 在LP64数据模型下是否开启对象指针压缩
  if (UseCompressedOops) {
    // Subtract a page because something can get allocated at heap base.
    // This also makes implicit null checking work, because the
    // memory+1 page below heap_base needs to cause a signal.
    // See needs_explicit_null_check.
    // Only set the heap base for compressed oops because it indicates
    // compressed oops for pstack code.
    bool verbose = PrintCompressedOopsMode || (PrintMiscellaneous && Verbose);
    if (verbose) {
      tty->cr();
      tty->print("heap address: " PTR_FORMAT ", size: " SIZE_FORMAT " MB",
                 Universe::heap()->base(), Universe::heap()->reserved_region().byte_size()/M);
    }
    if ((uint64_t)Universe::heap()->reserved_region().end() > OopEncodingHeapMax) {
      // Can't reserve heap below 32Gb.
      Universe::set_narrow_oop_base(Universe::heap()->base() - os::vm_page_size());
      Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
      if (verbose) {
        tty->print(", %s: "PTR_FORMAT,
            narrow_oop_mode_to_string(HeapBasedNarrowOop),
            Universe::narrow_oop_base());
      }
    } else {
      Universe::set_narrow_oop_base(0);
      if (verbose) {
        tty->print(", %s", narrow_oop_mode_to_string(ZeroBasedNarrowOop));
      }
#ifdef _WIN64
      if (!Universe::narrow_oop_use_implicit_null_checks()) {
        // Don't need guard page for implicit checks in indexed addressing
        // mode with zero based Compressed Oops.
        Universe::set_narrow_oop_use_implicit_null_checks(true);
      }
#endif //  _WIN64
      if((uint64_t)Universe::heap()->reserved_region().end() > NarrowOopHeapMax) {
        // Can't reserve heap below 4Gb.
        Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
      } else {
        Universe::set_narrow_oop_shift(0);
        if (verbose) {
          tty->print(", %s", narrow_oop_mode_to_string(UnscaledNarrowOop));
        }
      }
    }
    if (verbose) {
      tty->cr();
      tty->cr();
    }
  }
  assert(Universe::narrow_oop_base() == (Universe::heap()->base() - os::vm_page_size()) ||
         Universe::narrow_oop_base() == NULL, "invalid value");
  assert(Universe::narrow_oop_shift() == LogMinObjAlignmentInBytes ||
         Universe::narrow_oop_shift() == 0, "invalid value");
#endif

  // We will never reach the CATCH below since Exceptions::_throw will cause
  // the VM to exit if an exception is thrown during initialization
  // 如果使用TLAB则对其进行初始化
  if (UseTLAB) {
    assert(Universe::heap()->supports_tlab_allocation(),
           "Should support thread-local allocation buffers");
    ThreadLocalAllocBuffer::startup_initialization();
  }
  return JNI_OK;
}
```

在上面的代码中`Universe::heap()->initialize()`会调用GenCollectedHeap的`initialize()`方法：
```c
jint GenCollectedHeap::initialize() {
  CollectedHeap::pre_initialize();

  int i;
  _n_gens = gen_policy()->number_of_generations();

  // While there are no constraints in the GC code that HeapWordSize
  // be any particular value, there are multiple other areas in the
  // system which believe this to be true (e.g. oop->object_size in some
  // cases incorrectly returns the size in wordSize units rather than
  // HeapWordSize).
  guarantee(HeapWordSize == wordSize, "HeapWordSize must equal wordSize");

  // The heap must be at least as aligned as generations.
  size_t alignment = Generation::GenGrain;

  _gen_specs = gen_policy()->generations();
  PermanentGenerationSpec *perm_gen_spec =
                                collector_policy()->permanent_generation();

  // Make sure the sizes are all aligned.
  for (i = 0; i < _n_gens; i++) {
    _gen_specs[i]->align(alignment);
  }
  perm_gen_spec->align(alignment);

  // If we are dumping the heap, then allocate a wasted block of address
  // space in order to push the heap to a lower address.  This extra
  // address range allows for other (or larger) libraries to be loaded
  // without them occupying the space required for the shared spaces.

  if (DumpSharedSpaces) {
    uintx reserved = 0;
    uintx block_size = 64*1024*1024;
    while (reserved < SharedDummyBlockSize) {
      char* dummy = os::reserve_memory(block_size);
      reserved += block_size;
    }
  }

  // Allocate space for the heap.

  char* heap_address;
  size_t total_reserved = 0;
  int n_covered_regions = 0;
  ReservedSpace heap_rs(0);
  //分配区域，三个区域:YoungGen,OldGen,PermGen
  heap_address = allocate(alignment, perm_gen_spec, &total_reserved,
                          &n_covered_regions, &heap_rs);

  if (UseSharedSpaces) {
    if (!heap_rs.is_reserved() || heap_address != heap_rs.base()) {
      if (heap_rs.is_reserved()) {
        heap_rs.release();
      }
      FileMapInfo* mapinfo = FileMapInfo::current_info();
      mapinfo->fail_continue("Unable to reserve shared region.");
      allocate(alignment, perm_gen_spec, &total_reserved, &n_covered_regions,
               &heap_rs);
    }
  }

  if (!heap_rs.is_reserved()) {
    vm_shutdown_during_initialization(
      "Could not reserve enough space for object heap");
    return JNI_ENOMEM;
  }
  //_reserved区域包括Y,O,P三个区域
  _reserved = MemRegion((HeapWord*)heap_rs.base(),
                        (HeapWord*)(heap_rs.base() + heap_rs.size()));

  // It is important to do this in a way such that concurrent readers can't
  // temporarily think somethings in the heap.  (Seen this happen in asserts.)
  _reserved.set_word_size(0);
  _reserved.set_start((HeapWord*)heap_rs.base());
  size_t actual_heap_size = heap_rs.size() - perm_gen_spec->misc_data_size()
                                           - perm_gen_spec->misc_code_size();
  _reserved.set_end((HeapWord*)(heap_rs.base() + actual_heap_size));

  _rem_set = collector_policy()->create_rem_set(_reserved, n_covered_regions);
  set_barrier_set(rem_set()->bs());

  _gch = this;

  for (i = 0; i < _n_gens; i++) {
    ReservedSpace this_rs = heap_rs.first_part(_gen_specs[i]->max_size(),
                                              UseSharedSpaces, UseSharedSpaces);
    _gens[i] = _gen_specs[i]->init(this_rs, i, rem_set());
    // tag generations in JavaHeap
    MemTracker::record_virtual_memory_type((address)this_rs.base(), mtJavaHeap);
    heap_rs = heap_rs.last_part(_gen_specs[i]->max_size());
  }
  _perm_gen = perm_gen_spec->init(heap_rs, PermSize, rem_set());
  // tag PermGen
  MemTracker::record_virtual_memory_type((address)heap_rs.base(), mtJavaHeap);

  clear_incremental_collection_failed();

#ifndef SERIALGC
  // If we are running CMS, create the collector responsible
  // for collecting the CMS generations.
  if (collector_policy()->is_concurrent_mark_sweep_policy()) {
    bool success = create_cms_collector();
    if (!success) return JNI_ENOMEM;
  }
#endif // SERIALGC

  return JNI_OK;
}
```


至此JVM的初始化全部完成，至于内存策略的计算部分详细内容请阅读Hotspot源码，这里只作抛砖引玉。