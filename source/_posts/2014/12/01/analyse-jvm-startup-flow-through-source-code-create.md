---
title: 【Hotspot源码分析】从Hotpost源码角度深入分析Java程序启动过程-创建
date: 2014-12-01 03:43:40
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
---

博主在11年到12年的时候曾连续研究过十个月的JVM，读过的相关书籍包括：

- [深入Java虚拟机](https://book.douban.com/subject/1138768/)
这本书可以说是介绍JVM内部原理的鼻祖了，于2003年出版现已绝版，不过可以再某宝买到影印版。虽然当时JDK最高仅为1.4但JVM内部的构造已大体形成，所以博主强烈推荐此书。p.s 我肯定不会告诉你这书博主看了3遍：D

- [深入理解Java虚拟机](https://book.douban.com/subject/6522893/)
国内周某人写的，鉴于博主对于国人写的书向来不怎么感兴趣还是不提了。


说起JVM它可以是以下三种：
1. 一个正在运行的Java实例
2. Java虚拟机规范
3. 一种JVM虚拟机实现

之前的研究基本上都是虚拟机规范和JVM参数调优层面的内容，但是总觉得有些意犹未尽所以决定深入研究一下Hotspot实现，由大部分C/C++和少量汇编代码构成，但清晰的结构和优雅的编码使其并不难读，不得不赞叹一句SUN的大师们的智慧。至于如何编译、调试OpenJDK&Hotspot博主在前面的文章已经介绍过，这里便不再赘述，所以直入主题。

<!-- more -->
让我们从Java程序主入口开始逐步分析，主入口文件位于 `hotspot/src/share/tools/launcher/java.c`

main方法内容如下：


```c
/*
 * Entry point.
 * JAVA程序主入口
 */
int
main(int argc, char ** argv)
{
    char *jarfile = 0;
    char *classname = 0;
    char *s = 0;
    char *main_class = NULL;
    int ret;
    InvocationFunctions ifn;
    jlong start, end;
    char jrepath[MAXPATHLEN], jvmpath[MAXPATHLEN];
    char ** original_argv = argv;

    if (getenv("_JAVA_LAUNCHER_DEBUG") != 0) {
        _launcher_debug = JNI_TRUE;
        printf("----_JAVA_LAUNCHER_DEBUG----\n");
    }

#ifndef GAMMA
    // 确保指定的版本正在运行
    SelectVersion(argc, argv, &main_class);
#endif /* ifndef GAMMA */

    /* copy original argv */
    {
      int i;
      original_argv = (char**)JLI_MemAlloc(sizeof(char*)*(argc+1));
      for(i = 0; i < argc+1; i++)
        original_argv[i] = argv[i];
    }

    // 创建运行环境，如检查系统使用的数据模型（32bit、64bit），获取使用的JRE路径，找到jvm.cfg解析已知的vm类型
    // 设置新的LD_LIBRARY_PATH变量
    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               original_argv);

    printf("Using java runtime at: %s\n", jrepath);

    ifn.CreateJavaVM = 0;
    ifn.GetDefaultJavaVMInitArgs = 0;

    if (_launcher_debug)
      start = CounterGet();
    // 通过jvmpath找到libjvm.so 并将其JNI_CreateJavaVM和JNI_GetDefaultJavaVMInitArgs方法的
    // 符号地址返回，挂载到InvocationFunctions的CreateJavaVM和GetDefaultJavaVMInitArgs以便初始化调用
    if (!LoadJavaVM(jvmpath, &ifn)) {
      exit(6);
    }
    if (_launcher_debug) {
      end   = CounterGet();
      printf("%ld micro seconds to LoadJavaVM\n",
             (long)(jint)Counter2Micros(end-start));
    }

#ifdef JAVA_ARGS  /* javac, jar and friends. */
    progname = "java";
#else             /* java, oldjava, javaw and friends */
#ifdef PROGNAME
    progname = PROGNAME;
#else
    progname = *argv;
    if ((s = strrchr(progname, FILE_SEPARATOR)) != 0) {
        progname = s + 1;
    }
#endif /* PROGNAME */
#endif /* JAVA_ARGS */
    ++argv;
    --argc;

#ifdef JAVA_ARGS
    // 转换命令行参数 如：javac -cp foo:foo/"*" -J-ms32m
    /* Preprocess wrapper arguments */
    TranslateApplicationArgs(&argc, &argv);
    /**
     * 添加了三个VM选项
     * -Denv.class.patp 用户设置的CLASSPATH变量，如果CLASSPATH显式设置了tools.jar
     *                  则可以反编译VM的工具类sun.tools.*
     * -Dapplication.home 应用程序目录
     * -Djava.class.path 应用程序的类文件目录
     */
    if (!AddApplicationOptions()) {
        exit(1);
    }
#endif

    /* Set default CLASSPATH */
    if ((s = getenv("CLASSPATH")) == 0) {
        s = ".";
    }
#ifndef JAVA_ARGS
    SetClassPath(s);
#endif

    /*
     *  解析命令行参数-cp、-version、-*path、-X*等参数
     *  Parse command line options; if the return value of
     *  ParseArguments is false, the program should exit.
     */
    if (!ParseArguments(&argc, &argv, &jarfile, &classname, &ret, jvmpath)) {
      exit(ret);
    }

    /* Override class path if -jar flag was specified */
    if (jarfile != 0) {
        SetClassPath(jarfile);
    }

    /* set the -Dsun.java.command pseudo property */
    SetJavaCommandLineProp(classname, jarfile, argc, argv);

    /* Set the -Dsun.java.launcher pseudo property */
    SetJavaLauncherProp();

    /* set the -Dsun.java.launcher.* platform properties */
    SetJavaLauncherPlatformProps();

#ifndef GAMMA
    /* Show the splash screen if needed */
    ShowSplashScreen();
#endif

    /*
     * 移除环境变量防止重复执行
     * Done with all command line processing and potential re-execs so
     * clean up the environment.
     */
    (void)UnsetEnv(ENV_ENTRY);
#ifndef GAMMA
    (void)UnsetEnv(SPLASH_FILE_ENV_ENTRY);
    (void)UnsetEnv(SPLASH_JAR_ENV_ENTRY);

    JLI_MemFree(splash_jar_entry);
    JLI_MemFree(splash_file_entry);
#endif

    /*
     * 指定线程大小
     * If user doesn't specify stack size, check if VM has a preference.
     * Note that HotSpot no longer supports JNI_VERSION_1_1 but it will
     * return its default stack size through the init args structure.
     */
    if (threadStackSize == 0) {
      struct JDK1_1InitArgs args1_1;
      memset((void*)&args1_1, 0, sizeof(args1_1));
      args1_1.version = JNI_VERSION_1_1;
      ifn.GetDefaultJavaVMInitArgs(&args1_1);  /* ignore return value */
      if (args1_1.javaStackSize > 0) {
         threadStackSize = args1_1.javaStackSize;
      }
    }

    { /* Create a new thread to create JVM and invoke main method */
      struct JavaMainArgs args;

      args.argc = argc;
      args.argv = argv;
      args.jarfile = jarfile;
      args.classname = classname;
      args.ifn = ifn;
      // block当前线程并且在新线程中继续执行
      // 至于为什么在新线程中创建JVM见如下注释引用或原文https://bugs.openjdk.java.net/browse/JDK-6316197
//      Primordial thread is created by the kernel before any program/library code
//      has a chance to run. It's stack size and location can be very different
//      from other threads created by the application. Creating JVM from primordial
//      thread and later running Java code in the primordial thread introduced
//      many problems:
//
//      1. On Windows primordial thread stack size is controlled by PE header in
//         the executable. There is no way for user to change it dynamically, which
//         means -Xss does not work for primordial thread.
//
//      2. On Solaris/Linux, primordial thread stack size is controlled by ulimit -s,
//         which is usually very large (8M). To compensate for that we set guard
//         page in the middle of stack to artificially reduce the stack size. However,
//         this may interfere with native applications.
//
//      3. Setting guard page for primordial thread is dangerous. Unlike other
//         threads, primordial thread stack can grow on demand. getrlimit()
//         tells VM the ulimit value which is the upper limit but not necessarily
//         the actual stack size. What could happen is that VM sets up the guard
//         at the theoretical limit, but because the program doesn't really use
//         that much stack, the unused space is reused for other purposes (e.g. malloc)
//         by the OS (this reuse won't occur with other threads). We ended up having
//         some C heap inserted between stack and its guard page.
//
//      4. On Linux VM bangs stack address below current SP to check for stack overflows.
//         This will trigger SEGV's if it happens in primordial thread due to a security
//         feature built into the kernel. Linux VM gets around the problem by manually
//         expanding the stack. However when VM is expanding the stack, for a very short
//         period the available stack space will be reduced to just 1 page. If a signal
//         is delivered in that window, VM could end up without space to handle the signal.
//
//      5. Some Linux kernel randomizes the starting stack address for primordial thread
//         both for stack coloring and exec-shield, but it won't tell the application.
//         This makes it impossible to reliably detect stack location and size in primordial
//         thread. VM needs the information to correctly handle stack overflows. We do
//         have some cushion which is enough most of the time, but as shown in bug reports
//         people do hit crashes because of this.
//
//      6. On Linux there is no thr_main() equivalent that can tell if current thread
//         is primordial thread, makes it even harder to have special code to handle
//         primordial thread.
//
//      I'm sure there are other issues that I didn't cover in the list. Basically
//      primordial thread has been a constant source of runtime bugs.
//
//      This proposal calls for java launcher to stop calling JNI_CreateJavaVM from
//      primordial thread. Instead, it can create a new thread and move all invocation
//      code to the new thread. Primordial thread simply waits for the new thread
//      to return and then it can terminate the process with the same exit value returned
//      by the new thread. With this change we won't see any of the above problems
//      as long as the application is started by a standard Sun launcher.
//
//      The above mentioned will still exist if VM is invoked from natvie application.
//      Which means we have to keep all current VM workarounds for primordial thread,
//      and probably need to add more. But reliability wise this is still significantly
//      better as most people are using standard launcher. Also, unlike standard java
//      launcher, customers have full control of native launcher. For example, if they
//      wish to use larger stack on Windows, they could simply rebuild their launcher
//      with larger stack size.
      return ContinueInNewThread(JavaMain, threadStackSize, (void*)&args);
    }
}

int JNICALL
JavaMain(void * _args)
{
    struct JavaMainArgs *args = (struct JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    char *jarfile = args->jarfile;
    char *classname = args->classname;
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jstring mainClassName;
    jclass mainClass;
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    /*
     * Error message to print or display; by default the message will
     * only be displayed in a window.
     */
    char * message = "Fatal exception occurred.  Program will exit.";
    jboolean messageDest = JNI_FALSE;

    /* Initialize the virtual machine */

    if (_launcher_debug)
        start = CounterGet();
    // ================================
    // 开始进行虚拟机初始化，此方法内部调用了JNI_CreateJavaVM，
    // 这里做的事情非常之多，也是JVM启动的精华部分
    // 由于这部分内容甚多，所以在下篇文章中介绍
    // ================================
    if (!InitializeJVM(&vm, &env, &ifn)) {
        ReportErrorMessage("Could not create the Java virtual machine.",
                           JNI_TRUE);
        exit(1);
    }
    // 如果输入了-version或-showversion参数
    if (printVersion || showVersion) {
        PrintJavaVersion(env);
        if ((*env)->ExceptionOccurred(env)) {
            ReportExceptionDescription(env);
            goto leave;
        }
        if (printVersion) {
            ret = 0;
            message = NULL;
            goto leave;
        }
        if (showVersion) {
            fprintf(stderr, "\n");
        }
    }

    // 如果jar文件和类名均未指定则输出默认usage信息
    /* If the user specified neither a class name nor a JAR file */
    if (jarfile == 0 && classname == 0) {
        PrintUsage();
        message = NULL;
        goto leave;
    }

#ifndef GAMMA
    FreeKnownVMs();  /* after last possible PrintUsage() */
#endif

    if (_launcher_debug) {
        end   = CounterGet();
        printf("%ld micro seconds to InitializeJVM\n",
               (long)(jint)Counter2Micros(end-start));
    }

    /* At this stage, argc/argv have the applications' arguments */
    if (_launcher_debug) {
        int i = 0;
        printf("Main-Class is '%s'\n", classname ? classname : "");
        printf("Apps' argc is %d\n", argc);
        for (; i < argc; i++) {
            printf("    argv[%2d] = '%s'\n", i, argv[i]);
        }
    }

    ret = 1;

    /*
     * 获取应用程序的主类文件
     */
    // 解析jar包并加载主类文件
    if (jarfile != 0) {
        // 如果传入的是jar文件名称则通过调用java.util.jar.JarFile加载jar包并获取主类
        mainClassName = GetMainClassName(env, jarfile);
        if ((*env)->ExceptionOccurred(env)) {
            ReportExceptionDescription(env);
            goto leave;
        }
        if (mainClassName == NULL) {
          const char * format = "Failed to load Main-Class manifest "
                                "attribute from\n%s";
          message = (char*)JLI_MemAlloc((strlen(format) + strlen(jarfile)) *
                                    sizeof(char));
          sprintf(message, format, jarfile);
          messageDest = JNI_TRUE;
          goto leave;
        }
        classname = (char *)(*env)->GetStringUTFChars(env, mainClassName, 0);
        if (classname == NULL) {
            ReportExceptionDescription(env);
            goto leave;
        }
        // 加载mainClass
        mainClass = LoadClass(env, classname);
        if(mainClass == NULL) { /* exception occured */
            const char * format = "Could not find the main class: %s. Program will exit.";
            ReportExceptionDescription(env);
            message = (char *)JLI_MemAlloc((strlen(format) +
                                            strlen(classname)) * sizeof(char) );
            messageDest = JNI_TRUE;
            sprintf(message, format, classname);
            goto leave;
        }
        (*env)->ReleaseStringUTFChars(env, mainClassName, classname);
    } else {
      // 加载主类文件
      mainClassName = NewPlatformString(env, classname);
      if (mainClassName == NULL) {
        const char * format = "Failed to load Main Class: %s";
        message = (char *)JLI_MemAlloc((strlen(format) + strlen(classname)) *
                                   sizeof(char) );
        sprintf(message, format, classname);
        messageDest = JNI_TRUE;
        goto leave;
      }
      classname = (char *)(*env)->GetStringUTFChars(env, mainClassName, 0);
      if (classname == NULL) {
        ReportExceptionDescription(env);
        goto leave;
      }
      mainClass = LoadClass(env, classname);
      if(mainClass == NULL) { /* exception occured */
        const char * format = "Could not find the main class: %s.  Program will exit.";
        ReportExceptionDescription(env);
        message = (char *)JLI_MemAlloc((strlen(format) +
                                        strlen(classname)) * sizeof(char) );
        messageDest = JNI_TRUE;
        sprintf(message, format, classname);
        goto leave;
      }
      (*env)->ReleaseStringUTFChars(env, mainClassName, classname);
    }

    // 获得主方法的ID
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    if (mainID == NULL) {
        if ((*env)->ExceptionOccurred(env)) {
            ReportExceptionDescription(env);
        } else {
          message = "No main method found in specified class.";
          messageDest = JNI_TRUE;
        }
        goto leave;
    }

    {    /* Make sure the main method is public */
        jint mods;
        jmethodID mid;
        // 通过反射获得main方法修饰符
        jobject obj = (*env)->ToReflectedMethod(env, mainClass,
                                                mainID, JNI_TRUE);

        if( obj == NULL) { /* exception occurred */
            ReportExceptionDescription(env);
            goto leave;
        }

        mid =
          (*env)->GetMethodID(env,
                              (*env)->GetObjectClass(env, obj),
                              "getModifiers", "()I");
        if ((*env)->ExceptionOccurred(env)) {
            ReportExceptionDescription(env);
            goto leave;
        }
        // 确保是public类型
        mods = (*env)->CallIntMethod(env, obj, mid);
        if ((mods & 1) == 0) { /* if (!Modifier.isPublic(mods)) ... */
            message = "Main method not public.";
            messageDest = JNI_TRUE;
            goto leave;
        }
    }

    // 构建参数数组
    mainArgs = NewPlatformStringArray(env, argv, argc);
    if (mainArgs == NULL) {
        ReportExceptionDescription(env);
        goto leave;
    }

    // 调用main方法
    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;

    /*
     * Detach the main thread so that it appears to have ended when
     * the application's main method exits.  This will invoke the
     * uncaught exception handler machinery if main threw an
     * exception.  An uncaught exception handler cannot change the
     * launcher's return code except by calling System.exit.
     */
    if ((*vm)->DetachCurrentThread(vm) != 0) {
        message = "Could not detach main thread.";
        messageDest = JNI_TRUE;
        ret = 1;
        goto leave;
    }

    message = NULL;

 leave:
    /*
     * Wait for all non-daemon threads to end, then destroy the VM.
     * This will actually create a trivial new Java waiter thread
     * named "DestroyJavaVM", but this will be seen as a different
     * thread from the one that executed main, even though they are
     * the same C thread.  This allows mainThread.join() and
     * mainThread.isAlive() to work as expected.
     */
    (*vm)->DestroyJavaVM(vm);

    if(message != NULL && !noExitErrorMessage)
      ReportErrorMessage(message, messageDest);
    return ret;
}
```



下篇文章将介绍JVM初始化部分。