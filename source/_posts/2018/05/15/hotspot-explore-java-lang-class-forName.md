---
title: 【JVM源码探秘】细说Class.forName()底层实现
date: 2018-05-15 19:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

JVM允许在运行时动态装载类，这为开发者提供了极大方便，使用`Class.forName("com.xxx.Xxx")`，
装载完成后可以通过调用其`newInstance()`完成对象的创建，然后便可以正常操作该类。

接下来我们就细说说Class.forName()在JVM层面所做的事情。

<!-- more -->

# java.lang.Class
```java
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }
```

这里调用了native方法forName0()
```java
/** Called after security check for system loader access checks have been made. */
    private static native Class<?> forName0(String name, boolean initialize,
                                            ClassLoader loader,
                                            Class<?> caller)
       throws ClassNotFoundException;
```

# Class.c # Java_java_lang_Class_forName0()
源码实现位于src/java.base/share/native/libjava/Class.c
```c
// 动态装载类型入口
JNIEXPORT jclass JNICALL
Java_java_lang_Class_forName0(JNIEnv *env, jclass this, jstring classname,
                              jboolean initialize, jobject loader, jclass caller)
{
    char *clname;
    jclass cls = 0;
    char buf[128];
    jsize len;
    jsize unicode_len;

    if (classname == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return 0;
    }

    // 把类全限定名里的'.'翻译成'/'
    if (VerifyFixClassname(clname) == JNI_TRUE) {
        /* slashes present in clname, use name b4 translation for exception */
        (*env)->GetStringUTFRegion(env, classname, 0, unicode_len, clname);
        JNU_ThrowClassNotFoundException(env, clname);
        goto done;
    }

    // 验证类全限定名名合法性（是否以'/'分隔）
    if (!VerifyClassname(clname, JNI_TRUE)) {  /* expects slashed name */
        JNU_ThrowClassNotFoundException(env, clname);
        goto done;
    }

    // 从指定的加载器查找该类
    cls = JVM_FindClassFromCaller(env, clname, initialize, loader, caller);

 done:
    if (clname != buf) {
        free(clname);
    }
    return cls;
}
```


# jvm.cpp # JVM_FindClassFromCaller()
`JVM_FindClassFromCaller`方法位于`src/hotspot/share/prims/jvm.cpp`
```c
// 从指定的加载器查找该类
// Find a class with this name in this loader, using the caller's protection domain.
JVM_ENTRY(jclass, JVM_FindClassFromCaller(JNIEnv* env, const char* name,
                                          jboolean init, jobject loader,
                                          jclass caller))

  // 把当前类加入符号表（一个哈希表实现）
  TempNewSymbol h_name = SymbolTable::new_symbol(name, CHECK_NULL);

  // 获取加载器和调用类
  oop loader_oop = JNIHandles::resolve(loader);
  oop from_class = JNIHandles::resolve(caller);
  oop protection_domain = NULL;

  if (from_class != NULL && loader_oop != NULL) {
    protection_domain = java_lang_Class::as_Klass(from_class)->protection_domain();
  }
  
  // 查找该类
  jclass result = find_class_from_class_loader(env, h_name, init, h_loader,
                                               h_prot, false, THREAD);

  // 返回结果
  return result;
JVM_END

```

# symbolTable.cpp # lookup()
把当前类加入符号表，实现`src/hotspot/share/classfile/symbolTable.cpp`
```c
  // Symbol creation
  static Symbol* new_symbol(const char* utf8_buffer, int length, TRAPS) {
    assert(utf8_buffer != NULL, "just checking");
    return lookup(utf8_buffer, length, THREAD);
  }
  
  
  Symbol* SymbolTable::lookup(const char* name, int len, TRAPS) {
    unsigned int hashValue = hash_symbol(name, len);
    int index = the_table()->hash_to_index(hashValue);
  
    Symbol* s = the_table()->lookup(index, name, len, hashValue);
  
    // 找到则直接返回
    // Found
    if (s != NULL) return s;
  
    // 先获取SymbolTable_lock
    MutexLocker ml(SymbolTable_lock, THREAD);
  
    // 然后把该类加入符号表
    return the_table()->basic_add(index, (u1*)name, len, hashValue, true, THREAD);
  }
```

# jvm.cpp # find_class_from_class_loader()
加入符号表后紧接着在指定的classloader中查找该类，`/src/hotspot/share/prims/jvm.cpp`
```c
// Shared JNI/JVM entry points //////////////////////////////////////////////////////////////
// 从指定的classloader中查找类
jclass find_class_from_class_loader(JNIEnv* env, Symbol* name, jboolean init,
                                    Handle loader, Handle protection_domain,
                                    jboolean throwError, TRAPS) {

  //==========================================
  //
  // 根据指定的类名和加载器返回一个Klass对象，必要情况下需要加载该类。
  // 如果未找到该类则抛出NoClassDefFoundError或ClassNotFoundException
  //
  //=========================================
  Klass* klass = SystemDictionary::resolve_or_fail(name, loader, protection_domain, throwError != 0, CHECK_NULL);

  // Check if we should initialize the class
  if (init && klass->is_instance_klass()) {
    klass->initialize(CHECK_NULL);
  }
  return (jclass) JNIHandles::make_local(env, klass->java_mirror());
}
```

# systemDictionary.cpp # resolve_or_fail()
方法`SystemDictionary::resolve_or_fail()`位于`src/hotspot/share/classfile/systemDictionary.cpp`
```c
// Forwards to resolve_or_null

Klass* SystemDictionary::resolve_or_fail(Symbol* class_name, Handle class_loader, Handle protection_domain, bool throw_error, TRAPS) {
  Klass* klass = resolve_or_null(class_name, class_loader, protection_domain, THREAD);
  if (HAS_PENDING_EXCEPTION || klass == NULL) {
    // can return a null klass
    klass = handle_resolution_exception(class_name, throw_error, klass, THREAD);
  }
  return klass;
}
```

# systemDictionary.cpp # resolve_or_null()
```c
// Forwards to resolve_instance_class_or_null

Klass* SystemDictionary::resolve_or_null(Symbol* class_name, Handle class_loader, Handle protection_domain, TRAPS) {
  
  if (FieldType::is_array(class_name)) {
    return resolve_array_class_or_null(class_name, class_loader, protection_domain, THREAD);
  } else if (FieldType::is_obj(class_name)) {
    ResourceMark rm(THREAD);
    // Ignore wrapping L and ;.
    TempNewSymbol name = SymbolTable::new_symbol(class_name->as_C_string() + 1,
                                   class_name->utf8_length() - 2, CHECK_NULL);
    return resolve_instance_class_or_null(name, class_loader, protection_domain, THREAD);
  } else {
    // 解析实例类
    return resolve_instance_class_or_null(class_name, class_loader, protection_domain, THREAD);
  }
}
```
# systemDictionary.cpp # resolve_instance_class_or_null()

```c
Klass* SystemDictionary::resolve_instance_class_or_null(Symbol* name,
                                                        Handle class_loader,
                                                        Handle protection_domain,
                                                        TRAPS) {
  Handle lockObject = compute_loader_lock_object(class_loader, THREAD);
  check_loader_lock_contention(lockObject, THREAD);
  // 获取对象锁
  ObjectLocker ol(lockObject, THREAD, DoObjectLock);

  {
    MutexLocker mu(SystemDictionary_lock, THREAD);
    // 查找类
    InstanceKlass* check = find_class(d_index, d_hash, name, dictionary);
    if (check != NULL) {
      // Klass is already loaded, so just return it
      class_has_been_loaded = true;
      k = check;
    } else {
      // 查找该类是否在placeholder table中
      placeholder = placeholders()->get_entry(p_index, p_hash, name, loader_data);
      if (placeholder && placeholder->super_load_in_progress()) {
         super_load_in_progress = true;
         if (placeholder->havesupername() == true) {
           superclassname = placeholder->supername();
           havesupername = true;
         }
      }
    }
  }

  // 如果该类在placeholder table中，则说明类加载进行中
  if (super_load_in_progress && havesupername==true) {
    k = handle_parallel_super_load(name,
                                   superclassname,
                                   class_loader,
                                   protection_domain,
                                   lockObject, THREAD);
    if (HAS_PENDING_EXCEPTION) {
      return NULL;
    }
    if (k != NULL) {
      class_has_been_loaded = true;
    }
  }

  bool throw_circularity_error = false;
  if (!class_has_been_loaded) {
    bool load_instance_added = false;

    if (!class_has_been_loaded) {

      // =====================================
      //
      //      执行实例加载动作
      //
      // =====================================
      k = load_instance_class(name, class_loader, THREAD);

      if (!HAS_PENDING_EXCEPTION && k != NULL &&
        k->class_loader() != class_loader()) {

        check_constraints(d_index, d_hash, k, class_loader, false, THREAD);

        // Need to check for a PENDING_EXCEPTION again; check_constraints
        // can throw and doesn't use the CHECK macro.
        if (!HAS_PENDING_EXCEPTION) {
          { // Grabbing the Compile_lock prevents systemDictionary updates
            // during compilations.
            MutexLocker mu(Compile_lock, THREAD);
            update_dictionary(d_index, d_hash, p_index, p_hash,
              k, class_loader, THREAD);
          }

          // 通知JVMTI类加载事件
          if (JvmtiExport::should_post_class_load()) {
            Thread *thread = THREAD;
            assert(thread->is_Java_thread(), "thread->is_Java_thread()");
            JvmtiExport::post_class_load((JavaThread *) thread, k);
          }
        }
      }
    } // load_instance_class
  }

  ...

  return k;
}
```

# systemDictionary.cpp # load_instance_class()
```c
// ===================================================================================
//
//              加载实例class，这里有两种方式：
// ===================================================================================
//
// 1、如果classloader为null则说明是加载系统类，使用bootstrap loader
//    调用方式：直接调用ClassLoader::load_class()加载该类
//
// 2、如果classloader不为null则说明是非系统类，使用ext/app/自定义 classloader
//    调用方式：通过JavaCalls::call_virtual()调用Java方法ClassLoader.loadClass()加载该类
//
// ===================================================================================
InstanceKlass* SystemDictionary::load_instance_class(Symbol* class_name, Handle class_loader, TRAPS) {

  // 使用bootstrap加载器加载
  if (class_loader.is_null()) {

    // 根据全限定名获取包名
    // Find the package in the boot loader's package entry table.
    TempNewSymbol pkg_name = InstanceKlass::package_from_name(class_name, CHECK_NULL);
    if (pkg_name != NULL) {
      pkg_entry = loader_data->packages()->lookup_only(pkg_name);
    }

    InstanceKlass* k = NULL;

    if (k == NULL) {
      // Use VM class loader
      PerfTraceTime vmtimer(ClassLoader::perf_sys_classload_time());
      // =================================================================
      //
      //        使用bootstrap loader加载该类
      //
      // =================================================================
      k = ClassLoader::load_class(class_name, search_only_bootloader_append, CHECK_NULL);
    }


    return k;
  } else {
    // =======================================================================================
    //
    // 使用用户指定的加载器加载该类，调用class_loader的loadClass操作方法，
    // 最终返回一个标准的InstanceKlass，流程如下
    //
    // +-----------+  loadClass()   +---------------+  get_jobject()   +-------------+
    // | className | -------------> |   JavaValue   | ---------------> |     oop     |
    // +-----------+                +---------------+                  +-------------+
    //                                                                       |
    //                                                                       | as_Klass()
    //                                                                       v
    //                               +---------------+  cast()          +-------------+
    //                               | InstanceKlass | <--------------- |    Klass    |
    //                               +---------------+                  +-------------+
    //
    // =======================================================================================  
    ResourceMark rm(THREAD);

    assert(THREAD->is_Java_thread(), "must be a JavaThread");
    JavaThread* jt = (JavaThread*) THREAD;

    PerfClassTraceTime vmtimer(ClassLoader::perf_app_classload_time(),
                               ClassLoader::perf_app_classload_selftime(),
                               ClassLoader::perf_app_classload_count(),
                               jt->get_thread_stat()->perf_recursion_counts_addr(),
                               jt->get_thread_stat()->perf_timers_addr(),
                               PerfClassTraceTime::CLASS_LOAD);

    Handle s = java_lang_String::create_from_symbol(class_name, CHECK_NULL);
    // Translate to external class name format, i.e., convert '/' chars to '.'
    Handle string = java_lang_String::externalize_classname(s, CHECK_NULL);

    JavaValue result(T_OBJECT);

    InstanceKlass* spec_klass = SystemDictionary::ClassLoader_klass();

    // Added MustCallLoadClassInternal in case we discover in the field
    // a customer that counts on this call
    if (MustCallLoadClassInternal && has_loadClassInternal()) {
      JavaCalls::call_special(&result,
                              class_loader,
                              spec_klass,
                              vmSymbols::loadClassInternal_name(),
                              vmSymbols::string_class_signature(),
                              string,
                              CHECK_NULL);
    } else {
      // ===============================================================
      //
      // 调用ClassLoader.loadClass()方法加载该类，而最终会调用ClassLoader的native方法defineClass1()
      // 其实现位于ClassLoader.c # Java_java_lang_ClassLoader_defineClass1()
      //
      // ===============================================================
      JavaCalls::call_virtual(&result,
                              class_loader,
                              spec_klass,
                              vmSymbols::loadClass_name(),
                              vmSymbols::string_class_signature(),
                              string,
                              CHECK_NULL);
    }

    assert(result.get_type() == T_OBJECT, "just checking");
    // 获取oop对象
    oop obj = (oop) result.get_jobject();

    // 如果不是基本类，则转换成对应的InstanceKlass
    if ((obj != NULL) && !(java_lang_Class::is_primitive(obj))) {
      InstanceKlass* k = InstanceKlass::cast(java_lang_Class::as_Klass(obj));
      
      if (class_name == k->name()) {
        // 返回最终InstanceKlass
        return k;
      }
    }
    // Class is not found or has the wrong name, return NULL
    return NULL;
  }
}
```


至此，JVM便完成了类型的InstanceKlass实例创建，这里两种加载方式中不管是通过bootstrap loader还是app(or自定义) loader均是
殊途同归，都会经历class文件的`装载`、`验证`、`准备`、`解析`、`初始化`等操作。具体流程在[下一篇](/post/2018/05/17/hotspot-explore-class-loading-linking-and-initializing/)文章中详细介绍。

