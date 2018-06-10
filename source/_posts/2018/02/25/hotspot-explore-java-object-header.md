---
title: 【JVM源码探秘】Java对象模型之对象头
date: 2018-02-25 21:10:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

一个Java对象在JVM中是由一个对应角色的`oop`对象来描述的，比如`instanceOopDesc`用来描述普通实例对象，`arrayOopDesc`用来描述数组对象，而这些类型的oop对象均是继承自`oopDesc`。 

<!-- more -->

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

oopDesc主要包含两部分，一部分是`_mark`，一部分是`_metadata`，

- `_mark` _mark是一个`markOop`实例，它描述了一个对象的头信息，用于存储对象的运行时记录信息，如哈希值、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等：

```
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```


- `_metadata` 包含一个普通`_klass`和一个压缩后的`_compressed_klass`，详细信息参见[OpenJDK Wiki](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)。


