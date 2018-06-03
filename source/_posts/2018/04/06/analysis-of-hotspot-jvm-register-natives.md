---
title: 【JVM源码探秘】深入registerNatives()底层实现
date: 2018-04-06 19:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

 
在Java的系统包下如：

- java.lang.System
- java.lang.Object
- java.lang.Class

等类中均有一个静态块用来执行一个叫做`registerNatives()`的native方法，这个native方法里究竟都做了啥？我们进去瞧瞧。


```java
    private static native void registerNatives();
    static {
        registerNatives();
    }
```


<!-- more -->

# System.c
```c
/* Only register the performance-critical methods */
static JNINativeMethod methods[] = {
    {"currentTimeMillis", "()J",              (void *)&JVM_CurrentTimeMillis},
    {"nanoTime",          "()J",              (void *)&JVM_NanoTime},
    {"arraycopy",     "(" OBJ "I" OBJ "II)V", (void *)&JVM_ArrayCopy},
};

#undef OBJ

JNIEXPORT void JNICALL
Java_java_lang_System_registerNatives(JNIEnv *env, jclass cls)
{
    // 注册本地方法
    (*env)->RegisterNatives(env, cls,
                            methods, sizeof(methods)/sizeof(methods[0]));
}
```



# Object.c

```c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};

JNIEXPORT void JNICALL
Java_java_lang_Object_registerNatives(JNIEnv *env, jclass cls)
{
    // 注册本地方法
    (*env)->RegisterNatives(env, cls,
                            methods, sizeof(methods)/sizeof(methods[0]));
}
```

# Class.c
```c
static JNINativeMethod methods[] = {
    {"getName0",         "()" STR,          (void *)&JVM_GetClassName},
    {"getSuperclass",    "()" CLS,          NULL},
    {"getInterfaces0",   "()[" CLS,         (void *)&JVM_GetClassInterfaces},
    {"isInterface",      "()Z",             (void *)&JVM_IsInterface},
    {"getSigners",       "()[" OBJ,         (void *)&JVM_GetClassSigners},
    {"setSigners",       "([" OBJ ")V",     (void *)&JVM_SetClassSigners},
    {"isArray",          "()Z",             (void *)&JVM_IsArrayClass},
    {"isPrimitive",      "()Z",             (void *)&JVM_IsPrimitiveClass},
    {"getModifiers",     "()I",             (void *)&JVM_GetClassModifiers},
    {"getDeclaredFields0","(Z)[" FLD,       (void *)&JVM_GetClassDeclaredFields},
    {"getDeclaredMethods0","(Z)[" MHD,      (void *)&JVM_GetClassDeclaredMethods},
    {"getDeclaredConstructors0","(Z)[" CTR, (void *)&JVM_GetClassDeclaredConstructors},
    {"getProtectionDomain0", "()" PD,       (void *)&JVM_GetProtectionDomain},
    {"getDeclaredClasses0",  "()[" CLS,      (void *)&JVM_GetDeclaredClasses},
    {"getDeclaringClass0",   "()" CLS,      (void *)&JVM_GetDeclaringClass},
    {"getSimpleBinaryName0", "()" STR,      (void *)&JVM_GetSimpleBinaryName},
    {"getGenericSignature0", "()" STR,      (void *)&JVM_GetClassSignature},
    {"getRawAnnotations",      "()" BA,        (void *)&JVM_GetClassAnnotations},
    {"getConstantPool",     "()" CPL,       (void *)&JVM_GetClassConstantPool},
    {"desiredAssertionStatus0","("CLS")Z",(void *)&JVM_DesiredAssertionStatus},
    {"getEnclosingMethod0", "()[" OBJ,      (void *)&JVM_GetEnclosingMethodInfo},
    {"getRawTypeAnnotations", "()" BA,      (void *)&JVM_GetClassTypeAnnotations},
};



JNIEXPORT void JNICALL
Java_java_lang_Class_registerNatives(JNIEnv *env, jclass cls)
{
    // 注册本地方法
    methods[1].fnPtr = (void *)(*env)->GetSuperclass;
    (*env)->RegisterNatives(env, cls, methods,
                            sizeof(methods)/sizeof(JNINativeMethod));
}
```

# jni.cpp # jni_RegisterNatives()
通过以上源码发现均调用的`(*env)->RegisterNatives(env, cls, methods...`，这里的`*env`为`JNI`环境，
方法进入`jni_RegisterNatives()`

```c
  // 注册Java系统类中的本地方法
JNI_ENTRY(jint, jni_RegisterNatives(JNIEnv *env, jclass clazz,
                                    const JNINativeMethod *methods,
                                    jint nMethods))
  JNIWrapper("RegisterNatives");
  HOTSPOT_JNI_REGISTERNATIVES_ENTRY(env, clazz, (void *) methods, nMethods);
  jint ret = 0;
  DT_RETURN_MARK(RegisterNatives, jint, (const jint&)ret);

  // 加载对应的类并转换成Klass对象
  Klass* k = java_lang_Class::as_Klass(JNIHandles::resolve_non_null(clazz));

  for (int index = 0; index < nMethods; index++) {
    const char* meth_name = methods[index].name;
    const char* meth_sig = methods[index].signature;
    int meth_name_len = (int)strlen(meth_name);

    // The class should have been loaded (we have an instance of the class
    // passed in) so the method and signature should already be in the symbol
    // table.  If they're not there, the method doesn't exist.
    // 方法名
    TempNewSymbol  name = SymbolTable::probe(meth_name, meth_name_len);
    // 方法签名
    TempNewSymbol  signature = SymbolTable::probe(meth_sig, (int)strlen(meth_sig));

    // 如果没找到该方法则抛出java.lang.NoSuchMethodError()
    if (name == NULL || signature == NULL) {
      ResourceMark rm;
      stringStream st;
      st.print("Method %s.%s%s not found", k->external_name(), meth_name, meth_sig);
      // Must return negative value on failure
      THROW_MSG_(vmSymbols::java_lang_NoSuchMethodError(), st.as_string(), -1);
    }

    // 执行注册本地方法
    bool res = register_native(k, name, signature,
                               (address) methods[index].fnPtr, THREAD);
    if (!res) {
      ret = -1;
      break;
    }
  }
  return ret;
JNI_END
```


# jni.cpp # register_native()

```c
static bool register_native(Klass* k, Symbol* name, Symbol* signature, address entry, TRAPS) {
  // 找到对应的方法
  Method* method = k->lookup_method(name, signature);
  if (method == NULL) {
    ResourceMark rm;
    stringStream st;
    st.print("Method %s name or signature does not match",
             Method::name_and_sig_as_C_string(k, name, signature));
    THROW_MSG_(vmSymbols::java_lang_NoSuchMethodError(), st.as_string(), false);
  }
  if (!method->is_native()) {
    // 检查JVMTI是否指定native方法前缀
    // trying to register to a non-native method, see if a JVM TI agent has added prefix(es)
    method = find_prefixed_native(k, name, signature, THREAD);
    if (method == NULL) {
      ResourceMark rm;
      stringStream st;
      st.print("Method %s is not declared as native",
               Method::name_and_sig_as_C_string(k, name, signature));
      THROW_MSG_(vmSymbols::java_lang_NoSuchMethodError(), st.as_string(), false);
    }
  }

  if (entry != NULL) {
    // 设置本为本地方法
    method->set_native_function(entry,
      Method::native_bind_event_is_interesting);
  } else {
    method->clear_native_function();
  }
  if (PrintJNIResolving) {
    ResourceMark rm(THREAD);
    tty->print_cr("[Registering JNI native method %s.%s]",
      method->method_holder()->external_name(),
      method->name()->as_C_string());
  }
  return true;
}
```