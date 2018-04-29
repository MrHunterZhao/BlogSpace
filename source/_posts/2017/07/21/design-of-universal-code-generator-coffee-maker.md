---
title: 关于MyBatis通用代码生成器CoffeeMaker的设计思路
date: 2017-07-21 15:30:02
categories: MyBatis
tags:
    - MyBatis
---

CoffeeMaker是一款经过良好设计的代码生成器，可快速生成通用CRUD代码模板，使用方便且易于扩展。于2017年7月开发完成，源代码托管于[GitHub](https://github.com/MrHunterZhao/CoffeeMaker)仓库。

<!-- more -->
# 引言
在整个软件开发过程中有一大部分内容是相当有共性的，比如模型对象、配置文件、基本CRUD操作方法等等，

**程序员是用来思考问题的，不是用来执行重复任务的**，所以那些重复性的工作交给机器或工具去做就好了。

而开源社区里不乏一些代码生成器，但生成代码的格式、风格往往并不符合团队开发规范，去修改的话还得先去去读懂别人代码，为何不写一个出来沉淀为自己团队的产物呢。

# 设计
1. 通过JDBC读取DB元数据封装成数据对象。
2. 使用`DefinitionConverter`转换成模型文件定义对象。
3. 将模型文件对象封装成所需文件封装器`FileWrapper`。
4. 使用`FileParser`将`FileWrapper`渲染至对应的模板文件。
5. 输出最终目标文件。

# 流程
```
                              < Workflow diagram of CoffeeMaker >
                                       
+----------+  JDBC     +---------------------+    TableMetadata      +------------------------+
|          | ------>   |                     |   ---------------->   |  DefinitionConverter   |
| Database |           |  MetadataProvider   |                       +------------------------+
|          |           |                     |                         |
|          |           |                     |                         | FileDefinition
+----------+           +---------------------+                         v
                                                                   +- - - - - - - - - - - - - - +
                                                                   ' Various of file wrappers   '
                                                                   '                            '
                                                                   ' +------------------------+ '
                                                                   ' |      FileWrapper       | '
                                                                   ' +------------------------+ '
                                                                   '                            '
                                                                   +- - - - - - - - - - - - - - +
                                                                       |
                                                                       |
                                                                       |
                     +- - - - - - - - - - - - -+                       |
                     ' OUTPUT:                 '                       |
                     '                         '                       |
                     ' XxxEntity.java          '                       |
                     ' XxxMapper.xml           '                       |
                     ' XxxDao.java             '                       |
                     ' XxxService.java         '                       |
                     ' XxxServiceImpl.java     '                       |
                     ' XxxController.java      '                       |
                     ' XxxVo.java              '                       |
                     '                         '                       v
                     ' +---------------------+ '  Parse & export     +------------------------+
                     ' | Ultimate Code Files | ' <----------------   |                        |
                     ' +---------------------+ '                     |       FileParser       |
                     '                         '                     |                        |
                     +- - - - - - - - - - - - -+                     +------------------------+
                     
```


# 配置
1. 配置数据源文件`src/main/resource/config.properties`
2. 配置代码生成规则

```java
Configuration configuration = new Configuration();
        configuration.setTableName("t_user")
            .setTablePrefix("")
            .setPackageName("com.workholiday")
            .setPagerPackageName("com.workholiday.base.core.page")
            .setWithPager(true)
            .setOutputPath("/Users/hunterzhao/tmp/output");
```

# 执行
通过`CoffeeMakerLauncher`类的main方法执行代码生成器


# 输出
生成的代码模板如下：
- Entity文件
- DAO文件
- MyBatis mapper文件
- Service文件
- Service实现类
- VO文件
- Controller文件


# 扩展
如果需要定制化CoffeeMaker，可以通过修改（新增）`FileWrapper`和`FileTemplate`来轻松实现。
