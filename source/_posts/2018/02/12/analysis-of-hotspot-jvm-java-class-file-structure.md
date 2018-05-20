---
title: 【JVM源码探秘】Java中的Class文件结构
date: 2018-02-12 17:10:35
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

我们都知道JVM并不能直接运行Java源文件，而是程序猿通过JDK提供的`javac`命令将Java源文件编译成`.class`二进制文件，
然后供JVM加载并使用，也就是说class文件其实是程序猿和JVM之间交互的媒介，相当于介于用户和Linux内核之间的shell。

一个class文件完整地描述了Java源文件的各种信息，Oracle JVM规范中的[4.1 The ClassFile Structure](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-4.html#jvms-4.1) 详细定义了一个标准class文件的结构

<!-- more -->

```
ClassFile {
       u4             magic;
       u2             minor_version;
       u2             major_version;
       u2             constant_pool_count;
       cp_info        constant_pool[constant_pool_count-1];
       u2             access_flags;
       u2             this_class;
       u2             super_class;
       u2             interfaces_count;
       u2             interfaces[interfaces_count];
       u2             fields_count;
       field_info     fields[fields_count];
       u2             methods_count;
       method_info    methods[methods_count];
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}
```


# magic
class文件的魔数值，用于鉴定是否为一个合法class文件，值为`0xCAFEBABE`

# minor_version & major_version
class文件的主次版本号，随着JDK版本release递增，JVM运行时向下兼容，也就是说在JDK10上编译出的class文件在JRE7上无法运行，否则会抛出`java.lang.UnsupportedClassVersionError`
minor_version和major_version项目的值是该类文件的次版本号和主版本号。 主版本号和次版本号一起决定了类文件格式的版本。 
如果一个类文件的主版本号为M，次版本号为m，那么我们将它的类文件格式的版本表示为M.m. 因此，类文件格式版本可以按照字典顺序排列，例如，1.5 <2.0 <2.1。
Java虚拟机实现可以支持版本v的类文件格式，当且仅当v处于某个连续范围Mi.0≤v≤Mj.m. 范围基于实现符合的Java SE平台的版本。 
符合给定Java SE平台版本的实现必须支持表4.1-A中为该版本指定的范围，并且不支持其他范围。 （对于历史案例，显示的是JDK版本而不是Java SE平台版本。）

|Java SE版本| class文件格式版本号范围   |
|-------|------------------------|
|1.0.2	| 45.0 ≤ v ≤ 45.3|
|1.1	| 45.0 ≤ v ≤ 45.65535|
|1.2	| 45.0 ≤ v ≤ 46.0|
|1.3	| 45.0 ≤ v ≤ 47.0|
|1.4	| 45.0 ≤ v ≤ 48.0|
|5.0	| 45.0 ≤ v ≤ 49.0|
|6	    | 45.0 ≤ v ≤ 50.0|
|7	    | 45.0 ≤ v ≤ 51.0|
|8	    | 45.0 ≤ v ≤ 52.0|
|9	    | 45.0 ≤ v ≤ 53.0|
|10	    | 45.0 ≤ v ≤ 54.0|


# constant_pool_count
常量池元素条目数量，constant_pool_count项的值等于constant_pool表中的条目数加1。

# constant_pool[constant_pool_count-1]
constant_pool是一个结构表，它表示在ClassFile结构及其子结构中引用的各种字符串常量，类和接口名称，字段名称以及其他常量。 
每个constant_pool表项的格式由其第一个“标记”字节指示。constant_pool表的索引从1到constant_pool_count - 1。

# access_flags
access_flags项的值是用于表示对此类或接口的访问权限和属性的标志掩码。

|Flag             |值	    |说明
|-----------------|---------|----------------------------
|ACC_PUBLIC       |0x0001	|Declared public; may be accessed from outside its package.
|ACC_FINAL        |0x0010	|Declared final; no subclasses allowed.
|ACC_SUPER        |0x0020	|Treat superclass methods specially when invoked by the invokespecial instruction.
|ACC_INTERFACE    |0x0200	|Is an interface, not a class.
|ACC_ABSTRACT     |0x0400	|Declared abstract; must not be instantiated.
|ACC_SYNTHETIC    |0x1000	|Declared synthetic; not present in the source code.
|ACC_ANNOTATION   |0x2000	|Declared as an annotation type.
|ACC_ENUM         |0x4000	|Declared as an enum type.
|ACC_MODULE       |0x8000	|Is a module, not a class or interface.

ACC_MODULE标志表明这个类文件定义了一个模块，而不是类或接口。如果设置了ACC_MODULE标志，则特殊规则适用于本节末尾给出的类文件。
如果未设置ACC_MODULE标志，则当前段落下方的规则将应用于类文件。一个接口通过设置ACC_INTERFACE标志来区分。
如果未设置ACC_INTERFACE标志，则此类文件定义一个类，而不是接口或模块。
如果设置了ACC_INTERFACE标志，则还必须设置ACC_ABSTRACT标志，并且不得设置ACC_FINAL，ACC_SUPER，ACC_ENUM和ACC_MODULE标志集。
如果ACC_INTERFACE标志没有置位，除了ACC_ANNOTATION和ACC_MODULE之外，可以设置表4.1-B中的其他任何标志。但是，这样的类文件不能同时设置
其ACC_FINAL和ACC_ABSTRACT标志（JLS§8.1.1.2）。
ACC_SUPER标志指示如果它出现在这个类或接口中，则由invokespecial指令（§invokespecial）表示两个可选语义中的哪一个。 
Java虚拟机指令集的编译器应该设置ACC_SUPER标志。在Java SE 8及更高版本中，Java虚拟机认为ACC_SUPER标志将在每个类文件中设置，
而不管类文件中标志的实际值和类文件的版本如何。

ACC_SUPER标志的存在是为了与用于Java编程语言的早期编译器编译的代码向后兼容。 在1.0.2之前的JDK版本中，编译器生成了access_flags，
其中现在表示ACC_SUPER的标志没有指定的含义，并且Oracle的Java虚拟机实现忽略该标志（如果已设置）。ACC_SYNTHETIC标志表示该类或接口
是由编译器生成的，并未出现在源代码中。
注释类型（JLS§9.6）必须设置其ACC_ANNOTATION标志。 如果设置了ACC_ANNOTATION标志，则还必须设置ACC_INTERFACE标志。
ACC_ENUM标志表明该类或其超类被声明为枚举类型（JLS§8.9）。
未在表4.1-B中分配的access_flags项目的所有位保留供将来使用。 它们应该在生成的类文件中设置为零，并且应该被Java虚拟机实现忽略。

# this_class
this_class项的值必须是constant_pool表中的有效索引。 该索引处的constant_pool条目必须是表示由此类文件定义的类或接口的CONSTANT_Class_info结构。


# super_class
对于类来说，super_class项的值必须为零，或者必须是常量池表中的有效索引。 如果super_class项的值不为零，则该索引处的constant_pool条目
必须是表示由此类文件定义的类的直接超类的CONSTANT_Class_info结构。 直接超类或其任何超类都不能在其ClassFile结构的access_flags项中
设置ACC_FINAL标志。如果super_class项的值为零，那么这个类文件必须表示类Object，唯一没有直接超类的类或接口。
对于接口来说，super_class项的值必须始终是constant_pool表中的有效索引。 该索引处的constant_pool条目必须是表示类Object的
CONSTANT_Class_info结构。


# interfaces_count
interfaces_count项的值给出了此类或接口类型的直接超接口的数量。

# interfaces[interfaces_count]
interfaces数组中的每个值都必须是constant_pool表中的有效索引。 在接口[i]的每个值处的constant_pool条目（其中0≤i<interfaces_count）
必须是CONSTANT_Class_info结构，该结构表示作为该类或接口类型的直接超级接口的接口，按照从左到右的顺序 类型的来源。

# fields_count
fields_count项的值给出了field表中field_info结构的数量。 field_info结构表示由此类或接口类型声明的所有字段，包括类变量和实例变量。

# fields[fields_count]
字段表中的每个值都必须是一个field_info结构，以便对此类或接口中的字段进行完整描述。 字段表仅包含由此类或接口声明的那些字段。 
它不包含表示从超类或超接口继承的字段的项。

# methods_count
methods_count项的值给出了method_info的编号
结构在方法表中。

# methods[methods_count]
方法表中的每个值都必须是一个method_info结构（第4.6节），以提供此类或接口中方法的完整描述。 如果在method_info结构的access_flags项中
没有设置ACC_NATIVE和ACC_ABSTRACT标志，则也会提供实现该方法的Java虚拟机指令。
method_info结构表示由此类或接口类型声明的所有方法，包括实例方法，类方法，实例初始化方法）以及任何类或接口初始化方法。 
方法表不包含表示从超类或超接口继承的方法的项。

# attributes_count
attributes_count项的值给出了此类的属性表中的属性数量。

# attributes[attributes_count]

属性表的每个值必须是一个attribute_info结构

> 如果在access_flags项中设置了ACC_MODULE标志，则可以设置access_flags项中的其他标志，并且以下规则适用于ClassFile结构的其余部分：
  •major_version，minor_version：≥53.0（即Java SE 9及以上）
  •this_class：模块信息
  •super_class，interfaces_count，fields_count，methods_count：零
  •属性：必须存在一个模块属性。 除了Module，ModulePackages，ModuleMainClass，InnerClasses，SourceFile，SourceDebugExtension，
  RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations之外，可能不会出现任何预定义的属性（§4.7）。


