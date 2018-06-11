---
title: 【JVM源码探秘】细说Class的装载、链接和初始化
date: 2018-05-17 19:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)


接上篇[《【JVM源码探秘】细说Class.forName()底层实现》](/post/2018/05/15/hotspot-explore-java-lang-class-forName/)执行Class.forName("com.xxx.Xxx")通过`JavaCalls`组件调用了
Java方法`ClassLoader.loadClass()`对该类进行加载，而最终会调用ClassLoader的native方法`defineClass1()`，其实现位于`src/java.base/share/native/libjava/ClassLoader.c`。

<!-- more -->

# ClassLoader.c # Java_java_lang_ClassLoader_defineClass1()
```c
// 加载指定类
JNIEXPORT jclass JNICALL
Java_java_lang_ClassLoader_defineClass1(JNIEnv *env,
                                        jclass cls,
                                        jobject loader,
                                        jstring name,
                                        jbyteArray data,
                                        jint offset,
                                        jint length,
                                        jobject pd,
                                        jstring source)
{
    jbyte *body;
    char *utfName;
    jclass result = 0;
    char buf[128];
    char* utfSource;
    char sourceBuf[1024];

    if (data == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return 0;
    }

 
    ....
    

    // ==========================================
    // 查找并加载class
    // ==========================================
    result = JVM_DefineClassWithSource(env, utfName, loader, body, length, pd, utfSource);

    if (utfSource && utfSource != sourceBuf)
        free(utfSource);

 free_utfName:
    if (utfName && utfName != buf)
        free(utfName);

 free_body:
    free(body);
    return result;
}
```

# jvm.cpp # JVM_DefineClassWithSource()
方法跳转到`src/hotspot/share/prims/jvm.cpp`
```c
JVM_ENTRY(jclass, JVM_DefineClassWithSource(JNIEnv *env, const char *name, jobject loader, const jbyte *buf, jsize len, jobject pd, const char *source))
  JVMWrapper("JVM_DefineClassWithSource");

  return jvm_define_class_common(env, name, loader, buf, len, pd, source, THREAD);
JVM_END
```

# jvm.cpp # jvm_define_class_common()
```c
// common code for JVM_DefineClass() and JVM_DefineClassWithSource()
static jclass jvm_define_class_common(JNIEnv *env, const char *name,
                                      jobject loader, const jbyte *buf,
                                      jsize len, jobject pd, const char *source,
                                      TRAPS) {
  if (source == NULL)  source = "__JVM_DefineClass__";

  // 如果设置了-XX:+UsePerfData
  if (UsePerfData) {
    ClassLoader::perf_app_classfile_bytes_read()->inc(len);
  }

  // Since exceptions can be thrown, class initialization can take place
  // if name is NULL no check for class name in .class stream has to be made.
  TempNewSymbol class_name = NULL;
  if (name != NULL) {
    const int str_len = (int)strlen(name);

    // 如果全限定名超过了最大限制则抛出java_lang_NoClassDefFoundError()
    if (str_len > Symbol::max_length()) {
      // It's impossible to create this class;  the name cannot fit
      // into the constant pool.
      Exceptions::fthrow(THREAD_AND_LOCATION,
                         vmSymbols::java_lang_NoClassDefFoundError(),
                         "Class name exceeds maximum length of %d: %s",
                         Symbol::max_length(),
                         name);
      return 0;
    }
    // 将该类写入符号表
    class_name = SymbolTable::new_symbol(name, str_len, CHECK_NULL);
  }

  ResourceMark rm(THREAD);
  // 创建文件流
  ClassFileStream st((u1*)buf, len, source, ClassFileStream::verify);
  Handle class_loader (THREAD, JNIHandles::resolve(loader));
  if (UsePerfData) {
    is_lock_held_by_thread(class_loader,
                           ClassLoader::sync_JVMDefineClassLockFreeCounter(),
                           THREAD);
  }
  Handle protection_domain (THREAD, JNIHandles::resolve(pd));
  // =============================================================================
  //
  // 解析class文件流并返回对应的Klass
  //
  // =============================================================================
  Klass* k = SystemDictionary::resolve_from_stream(class_name,
                                                   class_loader,
                                                   protection_domain,
                                                   &st,
                                                   CHECK_NULL);

  if (log_is_enabled(Debug, class, resolve) && k != NULL) {
    trace_class_resolution(k);
  }

  return (jclass) JNIHandles::make_local(env, k->java_mirror());
}
```


# systemDictionary.cpp # resolve_from_stream()

```c
// 把一个klass加入系统（systemDictionary）
InstanceKlass* SystemDictionary::resolve_from_stream(Symbol* class_name,
                                                     Handle class_loader,
                                                     Handle protection_domain,
                                                     ClassFileStream* st,
                                                     TRAPS) {
  HandleMark hm(THREAD);

  ClassLoaderData* loader_data = register_loader(class_loader, CHECK_NULL);

  // 确认同步操作
  Handle lockObject = compute_loader_lock_object(class_loader, THREAD);
  check_loader_lock_contention(lockObject, THREAD);
  ObjectLocker ol(lockObject, THREAD, DoObjectLock);

 InstanceKlass* k = NULL;

  if (k == NULL) {
    if (st->buffer() == NULL) {
      return NULL;
    }
    // ===============================================
    //
    // 解析class文件流并创建Klass
    //
    // ===============================================
    k = KlassFactory::create_from_stream(st,
                                         class_name,
                                         loader_data,
                                         protection_domain,
                                         NULL, // host_klass
                                         NULL, // cp_patches
                                         CHECK_NULL);
  }

  // 如果支持并行加载
  if (is_parallelCapable(class_loader)) {
    InstanceKlass* defined_k = find_or_define_instance_class(h_name, class_loader, k, THREAD);
    if (!HAS_PENDING_EXCEPTION && defined_k != k) {
      
      loader_data->add_to_deallocate_list(k);
      k = defined_k;
    }
  } else {
  
    // ================================================================
    //
    // 如果不支持并行加载则直接更新systemDictionary，这里主要做两件事：
    // 
    // 1、把新的class加入到systemDictionary
    // 2、链接并初始化klass
    //
    // ================================================================
    define_instance_class(k, THREAD);
  }

  return k;
}
```

# klassFactory.cpp # create_from_stream()
这里是整个类加载流程中最为关键的两个步骤，实现逻辑较为复杂，先看`KlassFactory::create_from_stream()`，位于`src/hotspot/share/classfile/klassFactory.cpp`

```c
// =====================================================================================================================
//                            解析class文件流并创建InstanceKlass
// =====================================================================================================================
// 入口1：| classLoader.cpp      # ClassLoader::load_class()               |
// 入口2：| systemDictionary.cpp # SystemDictionary::parse_stream()        |  与resolve_from_stream()方法类似，但不会把解析的klass添加至systemDictionary
// 入口3：| systemDictionary.cpp # SystemDictionary::resolve_from_stream() |  called by jni_DefineClass and JVM_DefineClass).
// =====================================================================================================================
InstanceKlass* KlassFactory::create_from_stream(ClassFileStream* stream,
                                                Symbol* name,
                                                ClassLoaderData* loader_data,
                                                Handle protection_domain,
                                                const InstanceKlass* host_klass,
                                                GrowableArray<Handle>* cp_patches,
                                                TRAPS) {

  ClassFileStream* old_stream = stream;

  // ============================================================================
  // 
  // 实例化class文件解析器并执行解析操作，包含：
  //
  // ============================================================================
  // 1. 魔数
  // 2. 版本号
  // 3. 常量池
  // 4. 访问标识
  // 5. 类的修饰符合法性验证
  // 6. 父类
  // 7. 接口列表
  // 8. 域列表
  // 9. 方法列表
  // 10. 属性和注解列表
  //
  // ============================================================================
  ClassFileParser parser(stream,
                         name,
                         loader_data,
                         protection_domain,
                         host_klass,
                         cp_patches,
                         ClassFileParser::BROADCAST, // publicity level
                         CHECK_NULL);

  // ============================================================================
  //
  // 创建InstanceKlass，并把classFileParser解析的文件流内容填充至InstanceKlass返回
  //
  // ============================================================================
  InstanceKlass* result = parser.create_instance_klass(old_stream != stream, CHECK_NULL);

  if (result == NULL) {
    return NULL;
  }

  return result;
}
```

先实例化class文件解析器并执行解析操作
# classFileParser.cpp # ClassFileParser::ClassFileParser()
```c
ClassFileParser::ClassFileParser(ClassFileStream* stream,
                                 Symbol* name,
                                 ClassLoaderData* loader_data,
                                 Handle protection_domain,
                                 const InstanceKlass* host_klass,
                                 GrowableArray<Handle>* cp_patches,
                                 Publicity pub_level,
                                 TRAPS) :
  _stream(stream),
  _requested_name(name),
  _loader_data(loader_data),
  _host_klass(host_klass),
  _cp_patches(cp_patches),
  _num_patched_klasses(0),
  _max_num_patched_klasses(0),
  _orig_cp_size(0),
  _first_patched_klass_resolved_index(0),
  _super_klass(),
  _cp(NULL),
  _fields(NULL),
  _methods(NULL),
  _inner_classes(NULL),
  _local_interfaces(NULL),
  _transitive_interfaces(NULL),
  _combined_annotations(NULL),
  _annotations(NULL),
  _type_annotations(NULL),
  _fields_annotations(NULL),
  _fields_type_annotations(NULL),
  _klass(NULL),
  _klass_to_deallocate(NULL),
  _parsed_annotations(NULL),
  _fac(NULL),
  _field_info(NULL),
  _method_ordering(NULL),
  _all_mirandas(NULL),
  _vtable_size(0),
  _itable_size(0),
  _num_miranda_methods(0),
  _rt(REF_NONE),
  _protection_domain(protection_domain),
  _access_flags(),
  _pub_level(pub_level),
  _bad_constant_seen(0),
  _synthetic_flag(false),
  _sde_length(false),
  _sde_buffer(NULL),
  _sourcefile_index(0),
  _generic_signature_index(0),
  _major_version(0),
  _minor_version(0),
  _this_class_index(0),
  _super_class_index(0),
  _itfs_len(0),
  _java_fields_count(0),
  _need_verify(false),
  _relax_verify(false),
  _has_nonstatic_concrete_methods(false),
  _declares_nonstatic_concrete_methods(false),
  _has_final_method(false),
  _has_finalizer(false),
  _has_empty_finalizer(false),
  _has_vanilla_constructor(false),
  _max_bootstrap_specifier_index(-1) {

  _class_name = name != NULL ? name : vmSymbols::unknown_class_name();

  // synch back verification state to stream
  stream->set_verify(_need_verify);

  // ================================================================
  // 执行解析操作
  // ================================================================
  parse_stream(stream, CHECK);

  // 执行一些解析的后续操作（计算该类实现的接口列表、对方法列表进行排序等）
  post_process_parsed_stream(stream, _cp, CHECK);

}
```


# classFileParser.cpp # parse_stream()
```c
// 解析文件流
void ClassFileParser::parse_stream(const ClassFileStream* const stream,
                                   TRAPS) {

  // BEGIN STREAM PARSING
  stream->guarantee_more(8, CHECK);  // magic, major, minor
  
  // 魔数
  const u4 magic = stream->get_u4_fast();
  guarantee_property(magic == JAVA_CLASSFILE_MAGIC,
                     "Incompatible magic value %u in class file %s",
                     magic, CHECK);

  // 版本号
  _minor_version = stream->get_u2_fast();
  _major_version = stream->get_u2_fast();
  

  // 检查版本号是否支持，如果不支持抛出java_lang_UnsupportedClassVersionError()
  if (!is_supported_version(_major_version, _minor_version)) {
    ResourceMark rm(THREAD);
    Exceptions::fthrow(
      THREAD_AND_LOCATION,
      vmSymbols::java_lang_UnsupportedClassVersionError(),
      "%s has been compiled by a more recent version of the Java Runtime (class file version %u.%u), "
      "this version of the Java Runtime only recognizes class file versions up to %u.%u",
      _class_name->as_C_string(),
      _major_version,
      _minor_version,
      JAVA_MAX_SUPPORTED_VERSION,
      JAVA_MAX_SUPPORTED_MINOR_VERSION);
    return;
  }

  stream->guarantee_more(3, CHECK); // length, first cp tag
  u2 cp_size = stream->get_u2_fast();

  guarantee_property(
    cp_size >= 1, "Illegal constant pool size %u in class file %s",
    cp_size, CHECK);

  _orig_cp_size = cp_size;
  if (int(cp_size) + _max_num_patched_klasses > 0xffff) {
    THROW_MSG(vmSymbols::java_lang_InternalError(), "not enough space for patched classes");
  }
  cp_size += _max_num_patched_klasses;

  // 分配常量池空间
  _cp = ConstantPool::allocate(_loader_data,
                               cp_size,
                               CHECK);

  ConstantPool* const cp = _cp;

  // 解析常量池
  parse_constant_pool(stream, cp, _orig_cp_size, CHECK);

  // ACCESS FLAGS
  stream->guarantee_more(8, CHECK);  // flags, this_class, super_class, infs_len

  // 访问标识
  jint flags;
  // 定义了JVM_ACC_MODULE in JDK-9 and later.
  if (_major_version >= JAVA_9_VERSION) {
    flags = stream->get_u2_fast() & (JVM_RECOGNIZED_CLASS_MODIFIERS | JVM_ACC_MODULE);
  } else {
    flags = stream->get_u2_fast() & JVM_RECOGNIZED_CLASS_MODIFIERS;
  }

  // 验证类的修饰符合法性
  verify_legal_class_modifiers(flags, CHECK);

  short bad_constant = class_bad_constant_seen();
  if (bad_constant != 0) {
    // Do not throw CFE until after the access_flags are checked because if
    // ACC_MODULE is set in the access flags, then NCDFE must be thrown, not CFE.
    classfile_parse_error("Unknown constant tag %u in class file %s", bad_constant, CHECK);
  }

  _access_flags.set_flags(flags);

  
  // 修复匿名类名
  // if this is an anonymous class fix up its name if it's in the unnamed
  // package.  Otherwise, throw IAE if it is in a different package than
  // its host class.
  if (_host_klass != NULL) {
    fix_anonymous_class_name(CHECK);
  }

  // 解析父类
  _super_class_index = stream->get_u2_fast();
  _super_klass = parse_super_class(cp,
                                   _super_class_index,
                                   _need_verify,
                                   CHECK);

  // 解析接口
  _itfs_len = stream->get_u2_fast();
  parse_interfaces(stream,
                   _itfs_len,
                   cp,
                   &_has_nonstatic_concrete_methods,
                   CHECK);

  // 解析Field
  // Fields (offsets are filled in later)
  _fac = new FieldAllocationCount();
  parse_fields(stream,
               _access_flags.is_interface(),
               _fac,
               cp,
               cp_size,
               &_java_fields_count,
               CHECK);

  // ======================================================
  //
  // 解析方法列表
  // 包含栈、字节码表、异常表、局部变量表、运行指针等
  //
  // ======================================================
  AccessFlags promoted_flags;
  parse_methods(stream,
                _access_flags.is_interface(),
                &promoted_flags,
                &_has_final_method,
                &_declares_nonstatic_concrete_methods,
                CHECK);

  // promote flags from parse_methods() to the klass' flags
  _access_flags.add_promoted_flags(promoted_flags.as_int());

  if (_declares_nonstatic_concrete_methods) {
    _has_nonstatic_concrete_methods = true;
  }

  // 解析attributes/annotations
  _parsed_annotations = new ClassAnnotationCollector();
  parse_classfile_attributes(stream, cp, _parsed_annotations, CHECK);

  // Finalize the Annotations metadata object,
  // now that all annotation arrays have been created.
  create_combined_annotations(CHECK);

  // all bytes in stream read and parsed
}
```

# classFileParser.cpp # parse_methods()

```c
void ClassFileParser::parse_methods(const ClassFileStream* const cfs,
                                    bool is_interface,
                                    AccessFlags* promoted_flags,
                                    bool* has_final_method,
                                    bool* declares_nonstatic_concrete_methods,
                                    TRAPS) {
  cfs->guarantee_more(2, CHECK);  // length
  const u2 length = cfs->get_u2_fast();
  if (length == 0) {
    _methods = Universe::the_empty_method_array();
  } else {
    _methods = MetadataFactory::new_array<Method*>(_loader_data,
                                                   length,
                                                   NULL,
                                                   CHECK);

    for (int index = 0; index < length; index++) {
      // ===============================================
      // 遍历解析所有方法
      // ===============================================
      Method* method = parse_method(cfs,
                                    is_interface,
                                    _cp,
                                    promoted_flags,
                                    CHECK);

      if (method->is_final()) {
        *has_final_method = true;
      }
      // declares_nonstatic_concrete_methods: declares concrete instance methods, any access flags
      // used for interface initialization, and default method inheritance analysis
      if (is_interface && !(*declares_nonstatic_concrete_methods)
        && !method->is_abstract() && !method->is_static()) {
        *declares_nonstatic_concrete_methods = true;
      }
      
      // 放入方法表
      _methods->at_put(index, method);
    }
  }
}
```


# classFileParser.cpp # post_process_parsed_stream()
```c
void ClassFileParser::post_process_parsed_stream(const ClassFileStream* const stream,
                                                 ConstantPool* cp,
                                                 TRAPS) {
  assert(stream != NULL, "invariant");
  assert(stream->at_eos(), "invariant");
  assert(cp != NULL, "invariant");
  assert(_loader_data != NULL, "invariant");

  if (_class_name == vmSymbols::java_lang_Object()) {
    check_property(_local_interfaces == Universe::the_empty_klass_array(),
                   "java.lang.Object cannot implement an interface in class file %s",
                   CHECK);
  }
  // We check super class after class file is parsed and format is checked
  if (_super_class_index > 0 && NULL ==_super_klass) {
    Symbol* const super_class_name = cp->klass_name_at(_super_class_index);
    if (_access_flags.is_interface()) {
      // Before attempting to resolve the superclass, check for class format
      // errors not checked yet.
      guarantee_property(super_class_name == vmSymbols::java_lang_Object(),
        "Interfaces must have java.lang.Object as superclass in class file %s",
        CHECK);
    }
    Handle loader(THREAD, _loader_data->class_loader());
    // 解析父类
    _super_klass = (const InstanceKlass*)
                       SystemDictionary::resolve_super_or_fail(_class_name,
                                                               super_class_name,
                                                               loader,
                                                               _protection_domain,
                                                               true,
                                                               CHECK);
  }

  if (_super_klass != NULL) {
    if (_super_klass->has_nonstatic_concrete_methods()) {
      _has_nonstatic_concrete_methods = true;
    }

    // 确保不是接口
    if (_super_klass->is_interface()) {
      ResourceMark rm(THREAD);
      Exceptions::fthrow(
        THREAD_AND_LOCATION,
        vmSymbols::java_lang_IncompatibleClassChangeError(),
        "class %s has interface %s as super class",
        _class_name->as_klass_external_name(),
        _super_klass->external_name()
      );
      return;
    }
    // 确保父类不是final修饰
    if (_super_klass->is_final()) {
      THROW_MSG(vmSymbols::java_lang_VerifyError(), "Cannot inherit from final class");
    }
  }

  // 计算由此类实现的所有唯一接口的传递列表
  // Compute the transitive list of all unique interfaces implemented by this class
  _transitive_interfaces =
    compute_transitive_interfaces(_super_klass,
                                  _local_interfaces,
                                  _loader_data,
                                  CHECK);

  assert(_transitive_interfaces != NULL, "invariant");

  // sort methods
  _method_ordering = sort_methods(_methods);

  _all_mirandas = new GrowableArray<Method*>(20);

  Handle loader(THREAD, _loader_data->class_loader());
  
  // 计算虚拟表大小
  klassVtable::compute_vtable_size_and_num_mirandas(&_vtable_size,
                                                    &_num_miranda_methods,
                                                    _all_mirandas,
                                                    _super_klass,
                                                    _methods,
                                                    _access_flags,
                                                    _major_version,
                                                    loader,
                                                    _class_name,
                                                    _local_interfaces,
                                                    CHECK);

  // Size of Java itable (in words)
  _itable_size = _access_flags.is_interface() ? 0 :
    klassItable::compute_itable_size(_transitive_interfaces);

  _field_info = new FieldLayoutInfo();
  layout_fields(cp, _fac, _parsed_annotations, _field_info, CHECK);

  // Compute reference typ
  _rt = (NULL ==_super_klass) ? REF_NONE : _super_klass->reference_type();

}
```

# classFileParser.cpp # create_instance_klass()

接下来是创建InstanceKlass，并把classFileParser解析的文件流内容填充至InstanceKlass返回

```c
InstanceKlass* ClassFileParser::create_instance_klass(bool changed_by_loadhook, TRAPS) {
  if (_klass != NULL) {
    return _klass;
  }

  // 为新建的InstanceKlass分配内存空间
  InstanceKlass* const ik =
    InstanceKlass::allocate_instance_klass(*this, CHECK_NULL);

  // 把classFileParser解析的文件流内容填充至InstanceKlass
  fill_instance_klass(ik, changed_by_loadhook, CHECK_NULL);

  return ik;
}
```

# instanceKlass.cpp # allocate_instance_klass()
```c
// 为新建的InstanceKlass分配内存空间
InstanceKlass* InstanceKlass::allocate_instance_klass(const ClassFileParser& parser, TRAPS) {
  const int size = InstanceKlass::size(parser.vtable_size(),
                                       parser.itable_size(),
                                       nonstatic_oop_map_size(parser.total_oop_map_count()),
                                       parser.is_interface(),
                                       parser.is_anonymous(),
                                       should_store_fingerprint());

  const Symbol* const class_name = parser.class_name();
  assert(class_name != NULL, "invariant");
  ClassLoaderData* loader_data = parser.loader_data();
  assert(loader_data != NULL, "invariant");

  InstanceKlass* ik;

  // Allocation
  if (REF_NONE == parser.reference_type()) {
    if (class_name == vmSymbols::java_lang_Class()) {
      // mirror
      ik = new (loader_data, size, THREAD) InstanceMirrorKlass(parser);
    }
    else if (is_class_loader(class_name, parser)) {
      // class loader
      ik = new (loader_data, size, THREAD) InstanceClassLoaderKlass(parser);
    }
    else {

      // 普通类
      ik = new (loader_data, size, THREAD) InstanceKlass(parser, InstanceKlass::_misc_kind_other);
    }
  }
  else {
    // reference
    ik = new (loader_data, size, THREAD) InstanceRefKlass(parser);
  }

  const bool publicize = !parser.is_internal();

  // 把当前类加入到classloader
  // Add all classes to our internal class loader list here,
  // including classes in the bootstrap (NULL) class loader.
  loader_data->add_class(ik, publicize);
  Atomic::inc(&_total_instanceKlass_count);

  return ik;
}

```

# systemDictionary.cpp # define_instance_class()
至此，通过KlassFactory解析并创建完毕InstanceKlass，接下来就是调用`define_instance_class()`进行连接和初始化，

```c
void SystemDictionary::define_instance_class(InstanceKlass* k, TRAPS) {

  

  // 调用Java方法ClassLoader#addClass()把该类注册到对应的classloader
  //
  if (k->class_loader() != NULL) {
    methodHandle m(THREAD, Universe::loader_addClass_method());
    JavaValue result(T_VOID);
    JavaCallArguments args(class_loader_h);
    args.push_oop(Handle(THREAD, k->java_mirror()));
    JavaCalls::call(&result, m, &args, CHECK);
  }

  // 把新的class加入到systemDictionary
  //
  // Add the new class. We need recompile lock during update of CHA.
  {
    unsigned int p_hash = placeholders()->compute_hash(name_h);
    int p_index = placeholders()->hash_to_index(p_hash);

    MutexLocker mu_r(Compile_lock, THREAD);

    // 添加到class结构目录，初始化虚拟表，并把状态标记为loaded
    //
    // Add to class hierarchy, initialize vtables, and do possible
    // deoptimizations.
    add_to_hierarchy(k, CHECK); // No exception, but can block

    // 执行更新操作
    //
    // Add to systemDictionary - so other classes can see it.
    // Grabs and releases SystemDictionary_lock
    update_dictionary(d_index, d_hash, p_index, p_hash,
                      k, class_loader_h, THREAD);
  }
  
  // ============================================
  // 链接并初始化klass
  // ============================================
  k->eager_initialize(THREAD);

  // 通知 jvmti
  if (JvmtiExport::should_post_class_load()) {
      assert(THREAD->is_Java_thread(), "thread->is_Java_thread()");
      JvmtiExport::post_class_load((JavaThread *) THREAD, k);

  }
  class_define_event(k, loader_data);
}
```

# instanceKlass.cpp # eager_initialize()
```c
void InstanceKlass::eager_initialize(Thread *thread) {
  if (!EagerInitialization) return;

  if (this->is_not_initialized()) {
    // abort if the the class has a class initializer
    if (this->class_initializer() != NULL) return;

    // abort if it is java.lang.Object (initialization is handled in genesis)
    Klass* super_klass = super();
    if (super_klass == NULL) return;

    // abort if the super class should be initialized
    if (!InstanceKlass::cast(super_klass)->is_initialized()) return;

    // ============================================
    // 执行初始化该指针
    // ============================================
    eager_initialize_impl();
  }
}
```

# instanceKlass.cpp # eager_initialize_impl()
```c
void InstanceKlass::eager_initialize_impl() {
  EXCEPTION_MARK;
  HandleMark hm(THREAD);
  Handle h_init_lock(THREAD, init_lock());
  ObjectLocker ol(h_init_lock, THREAD, h_init_lock() != NULL);

  // abort if someone beat us to the initialization
  if (!is_not_initialized()) return;  // note: not equivalent to is_initialized()

  ClassState old_state = init_state();
  // =====================================================
  //
  //     链接class
  //
  // =====================================================
  link_class_impl(true, THREAD);
  
  if (HAS_PENDING_EXCEPTION) {
    CLEAR_PENDING_EXCEPTION;
   
    if (old_state != _init_state)
      set_init_state(old_state);
  } else {

    // =====================================================
    // class状态标记为fully_initialized
    // =====================================================
    set_init_state(fully_initialized);
    fence_and_clear_init_lock();
}
```



# instanceKlass.cpp # link_class_impl()
```c
bool InstanceKlass::link_class_impl(bool throw_verifyerror, TRAPS) {

  // return if already verified
  if (is_linked()) {
    return true;
  }
  
  JavaThread* jt = (JavaThread*)THREAD;

  // 在链接该类之前先链接父类
  Klass* super_klass = super();
  if (super_klass != NULL) {
    if (super_klass->is_interface()) {  // check if super class is an interface
      ResourceMark rm(THREAD);
      Exceptions::fthrow(
        THREAD_AND_LOCATION,
        vmSymbols::java_lang_IncompatibleClassChangeError(),
        "class %s has interface %s as super class",
        external_name(),
        super_klass->external_name()
      );
      return false;
    }

    InstanceKlass* ik_super = InstanceKlass::cast(super_klass);
    // 递归调用当前方法进行链接
    ik_super->link_class_impl(throw_verifyerror, CHECK_false);
  }

  // 在链接该类之前先链接该类实现的所有接口
  Array<Klass*>* interfaces = local_interfaces();
  int num_interfaces = interfaces->length();
  for (int index = 0; index < num_interfaces; index++) {
    InstanceKlass* interk = InstanceKlass::cast(interfaces->at(index));
    interk->link_class_impl(throw_verifyerror, CHECK_false);
  }

  // in case the class is linked in the process of linking its superclasses
  if (is_linked()) {
    return true;
  }

  // 验证 & 重写
  // verification & rewriting
  {
    HandleMark hm(THREAD);
    Handle h_init_lock(THREAD, init_lock());
    ObjectLocker ol(h_init_lock, THREAD, h_init_lock() != NULL);
    // rewritten will have been set if loader constraint error found
    // on an earlier link attempt
    // don't verify or rewrite if already rewritten
    //

    if (!is_linked()) {
      if (!is_rewritten()) {
        {
          bool verify_ok = verify_code(throw_verifyerror, THREAD);
          if (!verify_ok) {
            return false;
          }
        }

        // Just in case a side-effect of verify linked this class already
        // (which can sometimes happen since the verifier loads classes
        // using custom class loaders, which are free to initialize things)
        if (is_linked()) {
          return true;
        }

        // also sets rewritten
        rewrite_class(CHECK_false);
      } else if (is_shared()) {
        SystemDictionaryShared::check_verification_constraints(this, CHECK_false);
      }

      // 重写之后链接方法
      // relocate jsrs and link methods after they are all rewritten
      link_methods(CHECK_false);

      // 方法重写后初始化虚拟表和接口表
      // Initialize the vtable and interface table after
      // methods have been rewritten since rewrite may
      // fabricate new Method*s.
      // also does loader constraint checking
      //
      // initialize_vtable and initialize_itable need to be rerun for
      // a shared class if the class is not loaded by the NULL classloader.
      ClassLoaderData * loader_data = class_loader_data();
      if (!(is_shared() &&
            loader_data->is_the_null_class_loader_data())) {
        ResourceMark rm(THREAD);
        // 初始化虚拟表
        vtable().initialize_vtable(true, CHECK_false);
        // 初始化接口表
        itable().initialize_itable(true, CHECK_false);
      }
      
      // class状态标记为linked
      set_init_state(linked);
      if (JvmtiExport::should_post_class_prepare()) {
        Thread *thread = THREAD;
        assert(thread->is_Java_thread(), "thread->is_Java_thread()");
        JvmtiExport::post_class_prepare((JavaThread *) thread, this);
      }
    }
  }
  return true;
}
```


链接完成以后最终在方法`eager_initialize_impl()`内调用`set_init_state(fully_initialized);`把klass状态标记为
`初始化完成`状态，并返回一个可用的实例对象，至此类的`装载`、`验证`、`准备`、`解析`、`初始化`所有操作全部完成。




