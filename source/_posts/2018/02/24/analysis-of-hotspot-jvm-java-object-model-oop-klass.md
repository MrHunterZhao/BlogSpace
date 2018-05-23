---
title: 【JVM源码探秘】Java对象模型OOP-Klass
date: 2018-02-24 20:10:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

一个Java类在JVM中是如何描述的？创建一个Java对象在JVM中数据又是如何存储的？

在Hotspot VM中，设计者设计了OOP-Klass模型用来描述class的属性和行为，这里的OOP并不是面向对象编程（Object-oriented programming），
而是Ordinary Object Pointer（普通对象指针），之所以设计为OOP和Klass两部分是因为不希望每个对象都有一个C ++ vtbl指针，
因此，普通的oops没有任何虚拟功能。 相反，他们将所有“虚拟”函数转发到它们的klass，它具有vtbl并根据对象的实际类型执行C ++调度。 

<!-- more -->
# OOP

oopDesc是对象类的最高基类。 {name}Desc类描述了Java对象的格式，因此可以从C++访问这些字段。 oopDesc是抽象的。 

```c
 class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  // 对象头  
  volatile markOop _mark;
  // 元数据
  union _metadata {
    // 对应的Klass对象  
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
```

有关完整的类层次结构，请参见`src/hotspot/share/oops/oopsHierarchy.hpp`

OOP体系如下：
```c
typedef class oopDesc*                            oop;
typedef class   instanceOopDesc*            instanceOop;
typedef class   arrayOopDesc*                    arrayOop;
typedef class     objArrayOopDesc*            objArrayOop;
typedef class     typeArrayOopDesc*            typeArrayOop;
```


# Klass


Klass体系如下：
```c
class Klass;
class   InstanceKlass;
class     InstanceMirrorKlass;
class     InstanceClassLoaderKlass;
class     InstanceRefKlass;
class   ArrayKlass;
class     ObjArrayKlass;
class     TypeArrayKlass;
```

Klass对象提供：
- 1：语言级别的类对象（方法字典等）
- 2：为对象提供虚拟机调度行为




```c++
    class Klass : public Metadata {
      friend class VMStructs;
      friend class JVMCIVMStructs;
     protected:
      // If you add a new field that points to any metaspace object, you
      // must add this field to Klass::metaspace_pointers_do().
      enum { _primary_super_limit = 8 };
    
      // The "layout helper" is a combined descriptor of object layout.
      // For klasses which are neither instance nor array, the value is zero.
      jint        _layout_helper;
    
      // 类名
      // Class name.  Instance classes: java/lang/String, etc.  Array classes: [I,
      // [Ljava/lang/String;, etc.  Set to zero for all other kinds of classes.
      Symbol*     _name;
    
      // Cache of last observed secondary supertype
      Klass*      _secondary_super_cache;
      // Array of all secondary supertypes
      Array<Klass*>* _secondary_supers;
      // Ordered list of all primary supertypes
      Klass*      _primary_supers[_primary_super_limit];
    
      // java.lang.Class镜像类
      // java/lang/Class instance mirroring this class
      oop       _java_mirror;
      // 父类
      // Superclass
      Klass*      _super;
      // First subclass (NULL if none); _subklass->next_sibling() is next one
      Klass*      _subklass;
      // Sibling link (or NULL); links all subklasses of a klass
      Klass*      _next_sibling;
    
      // All klasses loaded by a class loader are chained through these links
      Klass*      _next_link;
    
      // 加载该类的类加载器
      // The VM's representation of the ClassLoader used to load this class.
      // Provide access the corresponding instance java.lang.ClassLoader.
      ClassLoaderData* _class_loader_data;
    
      // 修饰符
      jint        _modifier_flags;  // Processed access flags, for use by Class.getModifiers.
      // 访问权限
      AccessFlags _access_flags;    // Access flags. The class/interface distinction is stored here.
    
      TRACE_DEFINE_TRACE_ID_FIELD;
    
      // 偏向锁实现
      // Biased locking implementation and statistics
      // (the 64-bit chunk goes first, to avoid some fragmentation)
      jlong    _last_biased_lock_bulk_revocation_time;
      markOop  _prototype_header;   // Used when biased locking is both enabled and disabled for this type
      jint     _biased_lock_revocation_count;
    
      // 虚拟表长度
      // vtable length
      int _vtable_len;
    
      // Remembered sets support for the oops in the klasses.
      jbyte _modified_oops;             // Card Table Equivalent (YC/CMS support)
      jbyte _accumulated_modified_oops; // Mod Union Equivalent (CMS support)

```

