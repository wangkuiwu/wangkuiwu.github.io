---
layout: post
title: "java io系列07之 FileInputStream和FileOutputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-07 09:01
---
 

> 本章介绍FileInputStream 和 FileOutputStream 

> **目录**  
[1. FileInputStream 和 FileOutputStream 介绍](#anchor1)   
[2. 示例程序](#anchor2)   

<a name="anchor1"></a>
# 1. FileInputStream 和 FileOutputStream 介绍

FileInputStream 是文件输入流，它继承于InputStream。  
通常，我们使用FileInputStream从某个文件中获得输入字节。

FileOutputStream 是文件输出流，它继承于OutputStream。  
通常，我们使用FileOutputStream 将数据写入 File 或 FileDescriptor 的输出流。

关于File的介绍，可以参考文章“[java io之 File总结][link_io_file]” 。  
关于FileDescriptor的介绍，可以参考文章“[java io之 FileDescriptor总结][link_io_filedescriptor]”。

**FileInputStream 函数接口**

    FileInputStream(File file)         // 构造函数1：创建“File对象”对应的“文件输入流”
    FileInputStream(FileDescriptor fd) // 构造函数2：创建“文件描述符”对应的“文件输入流”
    FileInputStream(String path)       // 构造函数3：创建“文件(路径为path)”对应的“文件输入流”

    int      available()             // 返回“剩余的可读取的字节数”或者“skip的字节数”
    void     close()                 // 关闭“文件输入流”
    FileChannel      getChannel()    // 返回“FileChannel”
    final FileDescriptor     getFD() // 返回“文件描述符”
    int      read()                  // 返回“文件输入流”的下一个字节
    int      read(byte[] buffer, int byteOffset, int byteCount) // 读取“文件输入流”的数据并存在到buffer，从byteOffset开始存储，存储长度是byteCount。
    long     skip(long byteCount)    // 跳过byteCount个字节

**FileOutputStream 函数接口**

    FileOutputStream(File file)                   // 构造函数1：创建“File对象”对应的“文件输入流”；默认“追加模式”是false，即“写到输出的流内容”不是以追加的方式添加到文件中。
    FileOutputStream(File file, boolean append)   // 构造函数2：创建“File对象”对应的“文件输入流”；指定“追加模式”。
    FileOutputStream(FileDescriptor fd)           // 构造函数3：创建“文件描述符”对应的“文件输入流”；默认“追加模式”是false，即“写到输出的流内容”不是以追加的方式添加到文件中。
    FileOutputStream(String path)                 // 构造函数4：创建“文件(路径为path)”对应的“文件输入流”；默认“追加模式”是false，即“写到输出的流内容”不是以追加的方式添加到文件中。
    FileOutputStream(String path, boolean append) // 构造函数5：创建“文件(路径为path)”对应的“文件输入流”；指定“追加模式”。

    void                    close()      // 关闭“输出流”
    FileChannel             getChannel() // 返回“FileChannel”
    final FileDescriptor    getFD()      // 返回“文件描述符”
    void                    write(byte[] buffer, int byteOffset, int byteCount) // 将buffer写入到“文件输出流”中，从buffer的byteOffset开始写，写入长度是byteCount。
    void                    write(int oneByte)  // 写入字节oneByte到“文件输出流”中

<a name="anchor2"></a>
# 2. 示例程序

关于FileInputStream和FileOutputStream的API用法，参考示例代码(FileStreamTest.java)： 

    import java.io.File;
    import java.io.FileDescriptor;
    import java.io.FileInputStream;
    import java.io.FileOutputStream;
    import java.io.BufferedInputStream;
    import java.io.BufferedOutputStream;
    import java.io.PrintStream;;
    import java.io.IOException;

    /**
     * FileInputStream 和FileOutputStream 测试程序
     *
     * @author skywang
     */
    public class FileStreamTest {

        private static final String FileName = "file.txt";

        public static void main(String[] args) {
            testWrite();
            testRead();
        }

        /**
         * FileOutputStream 演示函数
         *
         * 运行结果：
         * 在源码所在目录生成文件"file.txt"，文件内容是“abcdefghijklmnopqrstuvwxyz0123456789”
         *
         * 加入，我们将 FileOutputStream fileOut2 = new FileOutputStream(file, true);
         *       修改为 FileOutputStream fileOut2 = new FileOutputStream(file, false);
         * 然后再执行程序，“file.txt”的内容变成"0123456789"。
         * 原因是：
         * (01) FileOutputStream fileOut2 = new FileOutputStream(file, true);
         *      它是以“追加模式”将内容写入文件的。即写入的内容，追加到原始的内容之后。
         * (02) FileOutputStream fileOut2 = new FileOutputStream(file, false);
         *      它是以“新建模式”将内容写入文件的。即删除文件原始的内容之后，再重新写入。
         */
        private static void testWrite() {
            try {
                // 创建文件“file.txt”对应File对象
                File file = new File(FileName);
                // 创建文件“file.txt”对应的FileOutputStream对象，默认是关闭“追加模式”
                FileOutputStream fileOut1 = new FileOutputStream(file);
                // 创建FileOutputStream对应的PrintStream，方便操作。PrintStream的写入接口更便利
                PrintStream out1 = new PrintStream(fileOut1);
                // 向“文件中”写入26个字母
                out1.print("abcdefghijklmnopqrstuvwxyz");
                out1.close();

                // 创建文件“file.txt”对应的FileOutputStream对象，打开“追加模式”
                FileOutputStream fileOut2 = new FileOutputStream(file, true);
                // 创建FileOutputStream对应的PrintStream，方便操作。PrintStream的写入接口更便利
                PrintStream out2 = new PrintStream(fileOut2);
                // 向“文件中”写入"0123456789"+换行符
                out2.println("0123456789");
                out2.close();

            } catch(IOException e) {
                e.printStackTrace();
            }
        }

        /**
         * FileInputStream 演示程序
         */
        private static void testRead() {
            try {
                // 方法1：新建FileInputStream对象
                // 新建文件“file.txt”对应File对象
                File file = new File(FileName);
                FileInputStream in1 = new FileInputStream(file);

                // 方法2：新建FileInputStream对象
                FileInputStream in2 = new FileInputStream(FileName);

                // 方法3：新建FileInputStream对象
                // 获取文件“file.txt”对应的“文件描述符”
                FileDescriptor fdin = in2.getFD();
                // 根据“文件描述符”创建“FileInputStream”对象
                FileInputStream in3 = new FileInputStream(fdin);

                // 测试read()，从中读取一个字节
                char c1 = (char)in1.read();
                System.out.println("c1="+c1);

                // 测试skip(long byteCount)，跳过4个字节
                in1.skip(25);

                // 测试read(byte[] buffer, int byteOffset, int byteCount)
                byte[] buf = new byte[10];
                in1.read(buf, 0, buf.length);
                System.out.println("buf="+(new String(buf)));


                // 创建“FileInputStream”对象对应的BufferedInputStream
                BufferedInputStream bufIn = new BufferedInputStream(in3);
                // 读取一个字节
                char c2 = (char)bufIn.read();
                System.out.println("c2="+c2);

                in1.close();
                in2.close();
                in3.close();
            } catch(IOException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    c1=a
    buf=0123456789
    c2=a

结果说明：

运行程序，会在源码所在位置新生成一个文件“file.txt”。它的内容是“abcdefghijklmnopqrstuvwxyz0123456789”。

 

[link_io_file]: 
[link_io_filedescriptor]: 

