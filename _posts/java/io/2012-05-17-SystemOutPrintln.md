---
layout: post
title: "java io系列17之 System.out.println详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-17 09:01
---


我们初学java的第一个程序是"hello world"

    public class HelloWorld {
        public static void main(String[] args) {
            System.out.println("hello world");
        }
    }

上面程序到底是怎么在屏幕上输出“hello world”的呢？这就是本来要讲解的内容，即System.out.println("hello world")的原理。


我们先看看System.out.println的流程。先看看System.java中out的定义，源码如下：

    public final class System {
        ...

        public final static PrintStream out = null;

        ...
    }

从中，我们发现，  
(01) out是System.java的静态变量。  
(02) 而且out是PrintStream对象，PrintStream.java中有许多重载的println()方法。

<br/>
OK，我们知道了out是PrintStream对象。接下来，看它是如何被初始化的，它是怎么和屏幕输出关联的？

我们还是一步步来分析，首先看看System.java的initializeSystemClass()方法。

**initializeSystemClass()的源码如下**

    private static void initializeSystemClass() {

        props = new Properties();
        initProperties(props);  // initialized by the VM

        sun.misc.VM.saveAndRemoveProperties(props);

        lineSeparator = props.getProperty("line.separator");
        sun.misc.Version.init();

        FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
        FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
        FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
        setIn0(new BufferedInputStream(fdIn));
        setOut0(new PrintStream(new BufferedOutputStream(fdOut, 128), true));
        setErr0(new PrintStream(new BufferedOutputStream(fdErr, 128), true));

        loadLibrary("zip");

        Terminator.setup();

        sun.misc.VM.initializeOSEnvironment();

        Thread current = Thread.currentThread();
        current.getThreadGroup().add(current);

        setJavaLangAccess();

        sun.misc.VM.booted();
    }

我们只需要关注下面的代码部分：即

    FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
    ...
    setOut0(new PrintStream(new BufferedOutputStream(fdOut, 128), true));

将这两句话细分，可以划分为以下几步：  
第1步 FileDescriptor fd = FileDescriptor.out;  
第2步 FileOutputStream fdOut = new FileOutputStream(fd);  
第3步 BufferedOutputStream bufOut = new BufferedOutputStream(fdOut, 128);  
第4步 PrintStream ps = new PrintStream(bufout, true);  
第5步 setOut0(ps);

说明：  
(01) 第1步，获取FileDescriptor.java中的静态成员out，out是一个FileDescriptor对象，它实际上是“标准输出(屏幕)”的标识符。关于FileDescriptor的详细介绍，可以参考博文“java io系列09之 FileDescriptor总结”。  
FileDescriptor.java中与FileDescriptor.out相关代码如下：

    public final class FileDescriptor {

        private int fd;

        public static final FileDescriptor out = new FileDescriptor(1);

        private FileDescriptor(int fd) {
            this.fd = fd;
            useCount = new AtomicInteger();
        }

        ...
    }

(02) 创建“标准输出(屏幕)”对应的“文件输出流”。  
(03) 创建“文件输出流”对应的“缓冲输出流”。目的是为“文件输出流”添加“缓冲”功能。  
(04) 创建“缓冲输出流”对应的“打印输出流”。目的是为“缓冲输出流”提供方便的打印接口，如print(), println(), printf()；使其能方便快捷的进行打印输出。  
(05) 执行setOut0(ps);


接下来，解析第5步的setOut0(ps)。查看System.java中setOut0()的声明，如下：

    private static native void setOut0(PrintStream out);

从中，我们发现setOut0()是一个native本地方法。通过openjdk，我们可以找到它对应的源码，如下：

    JNIEXPORT void JNICALL
    Java_java_lang_System_setOut0(JNIEnv *env, jclass cla, jobject stream)
    {
        jfieldID fid =
            (*env)->GetStaticFieldID(env,cla,"out","Ljava/io/PrintStream;");
        if (fid == 0)
            return;
        (*env)->SetStaticObjectField(env,cla,fid,stream);
    }

说明：这是个JNI函数，我们来对它进行简单的分析。  
(01) 函数名  

    JNIEXPORT void JNICALL Java_java_lang_System_setOut0(JNIEnv *env, jclass cla, jobject stream)

这是JNI的静态注册方法，Java_java_lang_System_setOut0(JNIEnv *env, jclass cla, jobject stream)会和System.java中的setOut0(PrintStream out)关联；而且，参数stream 对应参数out。简单来说，我们调用setOut0()，实际上是调用的Java_java_lang_System_setOut0()。  

(02) 

    jfieldID fid = (*env)->GetStaticFieldID(env,cla,"out","Ljava/io/PrintStream;");

这句话的作用是获取System.java的静态成员out的jfieldID，"Ljava/io/PrintStream;"是说明out是java.io.PrintStream对象。  
获取out的jfieldID的作用，是我们需要通过操作“out的jfielID”来改变out的值。

(03) 

    (*env)->SetStaticObjectField(env,cla,fid,stream);

这句话的作用是，设置fid(fid就是out的jfieldID)对应的静态成员的值为stream。  
stream是我们传给Java_java_lang_System_setOut0()的参数，也就是传给setOut0的参数。

总结上面的内容。我们知道，setOut0(PrintStream ps)的作用，就是将ps设置为System.java的out静态变量。

 

前面，已经说过FileDescriptor.out就是机器的“标准输出(屏幕)”的文件标识符。我们可以通俗的将文件标识符就理解为，FileDescriptor.out就是代表的“标准输出”。  
因此，在initializeSystemClass()中，上面的5步就是将“FileDescriptor.out”封装了起来。封装后的System.in既有缓冲功能；又有便利的操作接口，如print(), println(), printf()。

