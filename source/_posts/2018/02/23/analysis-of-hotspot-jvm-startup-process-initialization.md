---
title: 【JVM源码探秘】HotSpot启动流程分析-初始化
date: 2018-02-23 17:10:35
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

接[上篇](/post/2018/02/23/analysis-of-hotspot-jvm-startup-process-creation/)，HotSpot在启动流程完成了参数的解析、JNI入口的定位、环境变量的设置等一系列操作，

最终在`JavaMain()`中调用了`InitializeJVM()`方法，用于完成虚拟机所需的内存申请、挂载和初始化，本文我们就一起一探究竟。

<!-- more -->

# java.c # InitializeJVM()

```c
/*
 * Initializes the Java Virtual Machine. Also frees options array when
 * finished.
 */
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

    memset(&args, 0, sizeof(args));
    args.version  = JNI_VERSION_1_2;
    args.nOptions = numOptions;
    args.options  = options;
    args.ignoreUnrecognized = JNI_FALSE;

    if (JLI_IsTraceLauncher()) {
        int i = 0;
        printf("JavaVM args:\n    ");
        printf("version 0x%08lx, ", (long)args.version);
        printf("ignoreUnrecognized is %s, ",
               args.ignoreUnrecognized ? "JNI_TRUE" : "JNI_FALSE");
        printf("nOptions is %ld\n", (long)args.nOptions);
        for (i = 0; i < numOptions; i++)
            printf("    option[%2d] = '%s'\n",
                   i, args.options[i].optionString);
    }

    // 调用JNI_CreateJavaVM方法
    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    JLI_MemFree(options);
    return r == JNI_OK;
}
```
# jni.cpp # JNI_CreateJavaVM()
此处调用了之前加载的JNI_CreateJavaVM方法，位于`src/hotspot/share/prims/jni.cpp`


```c
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
  jint result = JNI_ERR;
  // On Windows, let CreateJavaVM run with SEH protection
#ifdef _WIN32
  __try {
#endif
    result = JNI_CreateJavaVM_inner(vm, penv, args);
#ifdef _WIN32
  } __except(topLevelExceptionFilter((_EXCEPTION_POINTERS*)_exception_info())) {
    // Nothing to do.
  }
#endif
  return result;
}
```

# jni.cpp # JNI_CreateJavaVM_inner()
`JNI_CreateJavaVM()`调用了内部方法`JNI_CreateJavaVM_inner()`


```c
static jint JNI_CreateJavaVM_inner(JavaVM **vm, void **penv, void *args) {
  HOTSPOT_JNI_CREATEJAVAVM_ENTRY((void **) vm, penv, args);

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

  // =================================
  //           创建虚拟机
  // =================================
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
  
  // 如果创建成功
  if (result == JNI_OK) {
    JavaThread *thread = JavaThread::current();
    assert(!thread->has_pending_exception(), "should have returned not OK");
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
      event.set_thread(THREAD_TRACE_ID(thread));
      event.commit();
    }

    // Since this is not a JVM_ENTRY we have to set the thread state manually before leaving.
    ThreadStateTransition::transition_and_fence(thread, _thread_in_vm, _thread_in_native);
  } 
  // 如果未创建成功
  else {
    
    ....
    
  }

  return result;

}
```

# thread.cpp # Threads::create_vm()
`Threads::create_vm()`方法位于`src/hotspot/share/runtime/thread.cpp`中



```c
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  extern void JDK_Version_init();

  // 版本信息初始化
  VM_Version::early_initialize();

  // 检查JNI版本
  if (!is_supported_jni_version(args->version)) return JNI_EVERSION;

  // 初始化TLS
  // Initialize library-based TLS
  ThreadLocalStorage::init();

  // 初始化系统输出流模块
  // Initialize the output stream module
  ostream_init();

  // 处理Java启动参数，如-Dsun.java.launcher*
  // Process java launcher properties.
  Arguments::process_sun_java_launcher_properties(args);

  // 初始化操作系统模块，如页大小，处理器数量，系统时钟等
  // Initialize the os module
  os::init();

  // 启动VM创建计时器
  // Record VM creation timing statistics
  TraceVmCreationTime create_vm_timer;
  create_vm_timer.start();

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
  // Initialize system properties.
  Arguments::init_system_properties();

  // JDK版本初始化
  // So that JDK version can be used as a discriminator when parsing arguments
  JDK_Version_init();

  // 设置java.vm.specification.vendor
  // java.vm.specification.version和java.vm.vendor属性
  // Update/Initialize System properties after JDK version number is known
  Arguments::init_version_specific_system_properties();

  // 初始化日志配置
  // Make sure to initialize log configuration *before* parsing arguments
  LogConfiguration::initialize(create_vm_timer.begin_time());

  // 解析启动参数，如-XX:Flags=、-XX:+PrintVMOptions、-XX:+PrintFlagsInitial etc.
  // Parse arguments
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;

  os::init_before_ergo();

  // 初始化GC日志输出流，用来处理-Xloggc参数
  // Initialize output stream logging
  ostream_init_log();

  // Convert -Xrun to -agentlib: if there is no JVM_OnLoad
  // Must be before create_vm_init_agents()
  if (Arguments::init_libraries_at_startup()) {
    convert_vm_init_libraries_to_agents();
  }

  // 初始化agent
  // Launch -agentlib/-agentpath and converted -Xrun agents
  if (Arguments::init_agents_at_startup()) {
    create_vm_init_agents();
  }

  // Initialize Threads state
  _thread_list = NULL;
  _number_of_threads = 0;
  _number_of_non_daemon_threads = 0;

  /*
   * ========================================
   * 初始化VM全局数据结构及系统类
   * ========================================
   * 初始化Java基础类型
   * 初始化对象OOP大小
   * 初始化锁
   * 初始化chunkpool
   * 初始化性能数据统计模块
   * ========================================
   */
  // Initialize global data structures and create system classes in heap
  vm_init_globals();

  // 初始化Java级别的对象同步器子系统
  // Initialize Java-Level synchronization subsystem
  ObjectMonitor::Initialize();

  /*
   * 初始化全局模块
   * ========================================
   * 1. 初始化management模块
   * 2. 初始化字节码/操作符表
   * 3. 初始化ClassLoader
   * 4. 根据命令行参数决定编译策略
   * 5. 代码缓存初始化
   * 6. 虚拟机版本初始化
   * 7. OS全局初始化
   * ....
   *
   * ========================================
   */
  jint status = init_globals();
  if (status != JNI_OK) {
    delete main_thread;
    *canTryAgain = false; // don't let caller call JNI_CreateJavaVM again
    return status;
  }

  if (TRACE_INITIALIZE() != JNI_OK) {
    vm_exit_during_initialization("Failed to initialize tracing backend");
  }

  // Should be done after the heap is fully created
  main_thread->cache_global_variables();

  HandleMark hm;

  { MutexLocker mu(Threads_lock);
    Threads::add(main_thread);
  }

  Thread* THREAD = Thread::current();

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_early_start_phase();

  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_early_vm_start();

  // 初始化Java的lang包
  initialize_java_lang_classes(main_thread, CHECK_JNI_ERR);

  // We need this for ClassDataSharing - the initial vm.info property is set
  // with the default value of CDS "sharing" which may be reset through
  // command line options.
  reset_vm_info_property(CHECK_JNI_ERR);

  quicken_jni_functions();

  // No more stub generation allowed after that point.
  StubCodeDesc::freeze();

  // Set flag that basic initialization has completed. Used by exceptions and various
  // debug stuff, that does not work until all basic classes have been initialized.
  set_init_completed();

  LogConfiguration::post_initialize();
  Metaspace::post_initialize();

  HOTSPOT_VM_INIT_END();

  // record VM initialization completion time
#if INCLUDE_MANAGEMENT
  Management::record_vm_init_completed();
#endif // INCLUDE_MANAGEMENT

  // 启动一个叫做“信号分发器”的线程用来处理进程间的信号
  // 比如通过jstack获取一个jvm实例的栈信息
  // Signal Dispatcher needs to be started before VMInit event is posted
  os::signal_init(CHECK_JNI_ERR);

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

  // 通知JVMTI agents虚拟机初始化开始
  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_vm_start();

  // Final system initialization including security manager and system class loader
  call_initPhase3(CHECK_JNI_ERR);

  // cache the system class loader
  SystemDictionary::compute_java_system_loader(CHECK_(JNI_ERR));


  if (MemProfiling)                   MemProfiler::engage();
  StatSampler::engage();
  if (CheckJNICalls)                  JniPeriodicChecker::engage();

  // 初始化偏向锁
  BiasedLocking::init();

  return JNI_OK;
}
```

# init.cpp # init_globals()
`init_globals()`方法用于初始化虚拟机全局模块，位于调用了`src/hotspot/share/runtime/init.cpp`


```c
jint init_globals() {
  HandleMark hm;
  // 初始化各子系统的监控及管理服务
  // JMX、线程和同步子系统、类加载子系统的监控和管理
  management_init();
  // 初始化字节码表，如istore、iload、iadd
  bytecodes_init();
  // 类加载器初始化
  classLoader_init1();
  // 初始化编译策略（根据启动参数决定编译策略）
  compilationPolicy_init();
  // 代码缓存池初始化
  codeCache_init();
  // 虚拟机版本初始化
  VM_Version_init();
  // OS全局初始化
  os_init_globals();
  stubRoutines_init1();
  // ============================
  // 初始化堆以及决定所使用GC策略
  // ============================
  jint status = universe_init();  // dependent on codeCache_init and
                                  // stubRoutines_init1 and metaspace_init.
  if (status != JNI_OK)
    return status;

  // 初始化解析器
  interpreter_init();  // before any methods loaded
  // 初始化动作触发器
  invocationCounter_init();  // before any methods loaded
  // 初始化MarkSweep
  marksweep_init();
  // 初始化访问标识
  accessFlags_init();
  // 初始化操作码模板表
  templateTable_init();
  // 接口支持提供了VM_LEAF_BASE和VM_ENTRY_BASE宏
  InterfaceSupport_init();
  SharedRuntime::generate_stubs();
  // 初始化语法表及系统字典等
  universe2_init();  // dependent on codeCache_init and stubRoutines_init1
  // 初始化软引用时间戳表并设定软引用清除策略
  referenceProcessor_init();
  jni_handles_init();
#if INCLUDE_VM_STRUCTS
  // 代码数据结构的必要性检查（仅限debug版本）
  vmStructs_init();
#endif // INCLUDE_VM_STRUCTS

  vtableStubs_init();
  InlineCacheBuffer_init();
  // oracle编译器初始化（oracle编译器是一个编译器开关接口）
  compilerOracle_init();
  dependencyContext_init();

  if (!compileBroker_init()) {
    return JNI_EINVAL;
  }
  VMRegImpl::set_regName();
  // 执行初始化
  if (!universe_post_init()) {
    return JNI_ERR;
  }
  javaClasses_init();   // must happen after vtable initialization
  stubRoutines_init2(); // note: StubRoutines need 2-phase init
  MethodHandles::generate_adapters();

#if INCLUDE_NMT
  // Solaris stack is walkable only after stubRoutines are set up.
  // On Other platforms, the stack is always walkable.
  NMT_stack_walkable = true;
#endif // INCLUDE_NMT

  // All the flags that get adjusted by VM_Version_init and os::init_2
  // have been set so dump the flags now.
  if (PrintFlagsFinal || PrintFlagsRanges) {
    CommandLineFlags::printFlags(tty, false, PrintFlagsRanges);
  }

  return JNI_OK;
}
```

# universe.cpp # universe_init()
`universe_init()`方法初始化堆以及决定所使用GC策略，位于`src/hotspot/share/memory/universe.cpp`

```c
jint universe_init() {
  assert(!Universe::_fully_initialized, "called after initialize_vtables");
  guarantee(1 << LogHeapWordSize == sizeof(HeapWord),
         "LogHeapWordSize is incorrect.");
  guarantee(sizeof(oop) >= sizeof(HeapWord), "HeapWord larger than oop?");
  guarantee(sizeof(oop) % sizeof(HeapWord) == 0,
            "oop size is not not a multiple of HeapWord size");

  TraceTime timer("Genesis", TRACETIME_LOG(Info, startuptime));

  JavaClasses::compute_hard_coded_offsets();

  /*
   * ==============================================================
   *                  初始化堆空间
   * ==============================================================
   * 在JDK7以前的版本中默认使用CMS收集器，这里会创建及初始化各分区代，设定空间比例大小，回收策略等
   * 流程：根据启动参数决定使用的回收策略，初始化回收策略时会指定所使用的代规范，
   * 	  最后根据规范创建对应类型的回收堆。
   *      i.e. arguments -> policy -> spec -> heap
   *
   * 在最新的JDK10中默认使用G1作为默认收集器，在JEP248里就提议，参见http://openjdk.java.net/jeps/248，
   * 虽然也采用分代算法，但由连续内存的年轻（老）代改为非连续的小块region（单个region连续）
   * ==============================================================
   */
  jint status = Universe::initialize_heap();
  if (status != JNI_OK) {
    return status;
  }

  // 初始化元数据空间
  // 在JDK8里移除了PermGen，就是加入了它
  Metaspace::global_initialize();

  // 初始化AOT loader
  AOTLoader::universe_init();

  // Checks 'AfterMemoryInit' constraints.
  if (!CommandLineFlagConstraintList::check_constraints(CommandLineFlagConstraint::AfterMemoryInit)) {
    return JNI_EINVAL;
  }

  // 为元数据申请内存空间
  // Create memory for metadata.  Must be after initializing heap for
  // DumpSharedSpaces.
  ClassLoaderData::init_null_class_loader_data();

  // We have a heap so create the Method* caches before
  // Metaspace::initialize_shared_spaces() tries to populate them.
  Universe::_finalizer_register_cache = new LatestMethodCache();
  Universe::_loader_addClass_cache    = new LatestMethodCache();
  Universe::_pd_implies_cache         = new LatestMethodCache();
  Universe::_throw_illegal_access_error_cache = new LatestMethodCache();
  Universe::_do_stack_walk_cache = new LatestMethodCache();

#if INCLUDE_CDS
  if (UseSharedSpaces) {
    // Read the data structures supporting the shared spaces (shared
    // system dictionary, symbol table, etc.).  After that, access to
    // the file (other than the mapped regions) is no longer needed, and
    // the file is closed. Closing the file does not affect the
    // currently mapped regions.
    MetaspaceShared::initialize_shared_spaces();
    StringTable::create_table();
  } else
#endif
  {
    // 创建符号表
    SymbolTable::create_table();
    // 创建字符串缓存池
    StringTable::create_table();

#if INCLUDE_CDS
    if (DumpSharedSpaces) {
      MetaspaceShared::prepare_for_dumping();
    }
#endif
  }
  if (strlen(VerifySubSet) > 0) {
    Universe::initialize_verify_flags();
  }

  // 创建方法表
  ResolvedMethodTable::create_table();

  return JNI_OK;
}
```

# universe.cpp # initialize_heap()
`initialize_heap()`方法用于初始化堆空间，位于`src/hotspot/share/memory/universe.cpp`
```c
// Choose the heap base address and oop encoding mode
// when compressed oops are used:
// Unscaled  - Use 32-bits oops without encoding when
//     NarrowOopHeapBaseMin + heap_size < 4Gb
// ZeroBased - Use zero based compressed oops with encoding when
//     NarrowOopHeapBaseMin + heap_size < 32Gb
// HeapBased - Use compressed oops with heap base + encoding.

jint Universe::initialize_heap() {
  jint status = JNI_ERR;


  // 根据GC策略创建堆空间
  _collectedHeap = create_heap_ext();
  if (_collectedHeap == NULL) {
    _collectedHeap = create_heap();
  }

  /*
   * ==========================================
   *        初始化堆空间
   * ==========================================
   * 这里会调用G1CollectedHeap::initialize()方法，
   * 真正向操作系统申请内存
   * ==========================================
   */
  status = _collectedHeap->initialize();
  if (status != JNI_OK) {
    return status;
  }
  log_info(gc)("Using %s", _collectedHeap->name());

  ThreadLocalAllocBuffer::set_max_size(Universe::heap()->max_tlab_size());

#ifdef _LP64
  // 在LP64数据模型下是否开启对象指针压缩
  if (UseCompressedOops) {
    // Subtract a page because something can get allocated at heap base.
    // This also makes implicit null checking work, because the
    // memory+1 page below heap_base needs to cause a signal.
    // See needs_explicit_null_check.
    // Only set the heap base for compressed oops because it indicates
    // compressed oops for pstack code.
    if ((uint64_t)Universe::heap()->reserved_region().end() > UnscaledOopHeapMax) {
      // Didn't reserve heap below 4Gb.  Must shift.
      Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
    }
    if ((uint64_t)Universe::heap()->reserved_region().end() <= OopEncodingHeapMax) {
      // Did reserve heap below 32Gb. Can use base == 0;
      Universe::set_narrow_oop_base(0);
    }

    Universe::set_narrow_ptrs_base(Universe::narrow_oop_base());

    LogTarget(Info, gc, heap, coops) lt;
    if (lt.is_enabled()) {
      ResourceMark rm;
      LogStream ls(lt);
      Universe::print_compressed_oops_mode(&ls);
    }

    // Tell tests in which mode we run.
    Arguments::PropertyList_add(new SystemProperty("java.vm.compressedOopsMode",
                                                   narrow_oop_mode_to_string(narrow_oop_mode()),
                                                   false));
  }
  // Universe::narrow_oop_base() is one page below the heap.
  assert((intptr_t)Universe::narrow_oop_base() <= (intptr_t)(Universe::heap()->base() -
         os::vm_page_size()) ||
         Universe::narrow_oop_base() == NULL, "invalid value");
  assert(Universe::narrow_oop_shift() == LogMinObjAlignmentInBytes ||
         Universe::narrow_oop_shift() == 0, "invalid value");
#endif

  // We will never reach the CATCH below since Exceptions::_throw will cause
  // the VM to exit if an exception is thrown during initialization
  // 如果使用TLAB
  if (UseTLAB) {
    assert(Universe::heap()->supports_tlab_allocation(),
           "Should support thread-local allocation buffers");
    ThreadLocalAllocBuffer::startup_initialization();
  }
  return JNI_OK;
}
```
# universe.cpp # create_heap()
`create_heap()`用于根据GC策略创建堆空间，位于`src/hotspot/share/memory/universe.cpp`

```c
CollectedHeap* Universe::create_heap() {
  assert(_collectedHeap == NULL, "Heap already created");
#if !INCLUDE_ALL_GCS
  if (UseParallelGC) {
    fatal("UseParallelGC not supported in this VM.");
  } else if (UseG1GC) {
    fatal("UseG1GC not supported in this VM.");
  } else if (UseConcMarkSweepGC) {
    fatal("UseConcMarkSweepGC not supported in this VM.");
#else
  if (UseParallelGC) {
    return Universe::create_heap_with_policy<ParallelScavengeHeap, GenerationSizer>();
  } else if (UseG1GC) {
    // 此处默认使用G1
    return Universe::create_heap_with_policy<G1CollectedHeap, G1CollectorPolicy>();
  } else if (UseConcMarkSweepGC) {
    return Universe::create_heap_with_policy<GenCollectedHeap, ConcurrentMarkSweepPolicy>();
#endif
  } else if (UseSerialGC) {
    return Universe::create_heap_with_policy<GenCollectedHeap, MarkSweepPolicy>();
  }

  ShouldNotReachHere();
  return NULL;
}
```

# g1CollectedHeap.cpp # G1CollectedHeap::initialize()

堆空间创建完毕，接下来是初始化，从上面`return Universe::create_heap_with_policy<G1CollectedHeap, G1CollectorPolicy>();`可以看出，
`_collectedHeap`对应的堆实现是`G1CollectedHeap`，位于`src/hotspot/share/gc/g1/g1CollectedHeap.cpp`，
对应上面的`_collectedHeap->initialize()`，

```c
// G1CollectedHeap初始化
jint G1CollectedHeap::initialize() {
  CollectedHeap::pre_initialize();
  os::enable_vtime();

  // Necessary to satisfy locking discipline assertions.

  MutexLocker x(Heap_lock);

  size_t init_byte_size = collector_policy()->initial_heap_byte_size();
  size_t max_byte_size = collector_policy()->max_heap_byte_size();
  size_t heap_alignment = collector_policy()->heap_alignment();

  // 申请Java堆内存及确定CompressedOops模式
  ReservedSpace heap_rs = Universe::reserve_heap(max_byte_size,
                                                 heap_alignment);

  // 初始化申请的内存区域
  initialize_reserved_region((HeapWord*)heap_rs.base(), (HeapWord*)(heap_rs.base() + heap_rs.size()));

  // 为整个保留区域创建barrier
  // Create the barrier set for the entire reserved region.
  G1SATBCardTableLoggingModRefBS* bs
    = new G1SATBCardTableLoggingModRefBS(reserved_region());
  bs->initialize();
  assert(bs->is_a(BarrierSet::G1SATBCTLogging), "sanity");
  set_barrier_set(bs);

  // 创建热卡缓存
  // Create the hot card cache.
  _hot_card_cache = new G1HotCardCache(this);

  // Carve out the G1 part of the heap.
  ReservedSpace g1_rs = heap_rs.first_part(max_byte_size);
  size_t page_size = UseLargePages ? os::large_page_size() : os::vm_page_size();
  // 创建mapper
  G1RegionToSpaceMapper* heap_storage =
    G1RegionToSpaceMapper::create_mapper(g1_rs,
                                         g1_rs.size(),
                                         page_size,
                                         HeapRegion::GrainBytes,
                                         1,
                                         mtJavaHeap);
  os::trace_page_sizes("Heap",
                       collector_policy()->min_heap_byte_size(),
                       max_byte_size,
                       page_size,
                       heap_rs.base(),
                       heap_rs.size());
  heap_storage->set_mapping_changed_listener(&_listener);

  FreeRegionList::set_unrealistically_long_length(max_regions() + 1);

  _bot = new G1BlockOffsetTable(reserved_region(), bot_storage);

  {
    HeapWord* start = _hrm.reserved().start();
    HeapWord* end = _hrm.reserved().end();
    size_t granularity = HeapRegion::GrainBytes;

    _in_cset_fast_test.initialize(start, end, granularity);
    _humongous_reclaim_candidates.initialize(start, end, granularity);
  }

  // 创建G1ConcurrentMark数据结构和线程
  // Create the G1ConcurrentMark data structure and thread.
  // (Must do this late, so that "max_regions" is defined.)
  _cm = new G1ConcurrentMark(this, prev_bitmap_storage, next_bitmap_storage);
  if (_cm == NULL || !_cm->completed_initialization()) {
    vm_shutdown_during_initialization("Could not create/initialize G1ConcurrentMark");
    return JNI_ENOMEM;
  }
  _cmThread = _cm->cmThread();

  // Now expand into the initial heap size.
  if (!expand(init_byte_size, _workers)) {
    vm_shutdown_during_initialization("Failed to allocate initial heap.");
    return JNI_ENOMEM;
  }

  // 执行委托给内存（G1）策略的所有初始化操作
  // Perform any initialization actions delegated to the policy.
  g1_policy()->init(this, &_collection_set);

  JavaThread::satb_mark_queue_set().initialize(SATB_Q_CBL_mon,
                                               SATB_Q_FL_lock,
                                               G1SATBProcessCompletedThreshold,
                                               Shared_SATB_Q_lock);

  jint ecode = initialize_concurrent_refinement();
  if (ecode != JNI_OK) {
    return ecode;
  }

  JavaThread::dirty_card_queue_set().initialize(DirtyCardQ_CBL_mon,
                                                DirtyCardQ_FL_lock,
                                                (int)concurrent_g1_refine()->yellow_zone(),
                                                (int)concurrent_g1_refine()->red_zone(),
                                                Shared_DirtyCardQ_lock,
                                                NULL,  // fl_owner
                                                true); // init_free_ids

  dirty_card_queue_set().initialize(DirtyCardQ_CBL_mon,
                                    DirtyCardQ_FL_lock,
                                    -1, // never trigger processing
                                    -1, // no limit on length
                                    Shared_DirtyCardQ_lock,
                                    &JavaThread::dirty_card_queue_set());

  // Here we allocate the dummy HeapRegion that is required by the
  // G1AllocRegion class.
  HeapRegion* dummy_region = _hrm.get_dummy_region();

  // We'll re-use the same region whether the alloc region will
  // require BOT updates or not and, if it doesn't, then a non-young
  // region will complain that it cannot support allocations without
  // BOT updates. So we'll tag the dummy region as eden to avoid that.
  dummy_region->set_eden();
  // Make sure it's full.
  dummy_region->set_top(dummy_region->end());
  G1AllocRegion::setup(this, dummy_region);

  _allocator->init_mutator_alloc_region();

  // Do create of the monitoring and management support so that
  // values in the heap have been properly initialized.
  _g1mm = new G1MonitoringSupport(this);

  G1StringDedup::initialize();

  _preserved_marks_set.init(ParallelGCThreads);

  _collection_set.initialize(max_regions());

  return JNI_OK;
}
```


至此，JVM的整个初始化工作完成，关于GC策略的空间分配具体细节在以后的文章中再详细介绍。

