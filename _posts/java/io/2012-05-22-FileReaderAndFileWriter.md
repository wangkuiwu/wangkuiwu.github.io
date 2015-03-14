---
layout: post
title: "java io系列22之 FileReader和FileWriter详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-22 09:01
---

> **目录**  
[1. FileReader和FileWriter 介绍](#anchor1)   
[2. FileReader和FileWriter源码分析](#anchor2)   
[3. 示例程序](#anchor3)   


<a name="anchor1"></a>
# 1. FileReader和FileWriter 介绍

FileReader 是用于读取字符流的类，它继承于InputStreamReader。要读取原始字节流，请考虑使用 FileInputStream。  
FileWriter 是用于写入字符流的类，它继承于OutputStreamWriter。要写入原始字节流，请考虑使用 FileOutputStream。


 

<a name="anchor2"></a>
# 2. FileReader和FileWriter源码分析

## 2.1 FileReader 源码(基于jdk1.7.40)

    package java.io;

    public class FileReader extends InputStreamReader {

        public FileReader(String fileName) throws FileNotFoundException {
            super(new FileInputStream(fil java io系列21之 InputStreamReader和OutputStreamWritereName));
        }

        public FileReader(File file) throws FileNotFoundException {
            super(new FileInputStream(file));
        }

        public FileReader(FileDescriptor fd) {
            super(new FileInputStream(fd));
        }
    }

从中，我们可以看出FileReader是基于InputStreamReader实现的。 

## 2.2 FileWriter 源码(基于jdk1.7.40)

    package java.io;

    public class FileWriter extends OutputStreamWriter {

        public FileWriter(String fileName) throws IOException {
            super(new FileOutputStream(fileName));
        }

        public FileWriter(String fileName, boolean append) throws IOException {
            super(new FileOutputStream(fileName, append));
        }

        public FileWriter(File file) throws IOException {
            super(new FileOutputStream(file));
        }

        public FileWriter(File file, boolean append) throws IOException {
            super(new FileOutputStream(file, append));
        }

        public FileWriter(FileDescriptor fd) {
            super(new FileOutputStream(fd));
        }
    }

从中，我们可以看出FileWriter是基于OutputStreamWriter实现的。


<a name="anchor3"></a>
# 3. 示例程序

    import java.io.File;
    import java.io.FileInputStream;
    import java.io.FileOutputStream;
    import java.io.FileWriter;;
    import java.io.FileReader;
    import java.io.IOException;

    /**
     * FileReader 和 FileWriter 测试程序
     *
     * @author skywang
     */
    public class FileReaderWriterTest {

        private static final String FileName = "file.txt";
        private static final String CharsetName = "utf-8";

        public static void main(String[] args) {
            testWrite();
            testRead();
        }

        /**
         * OutputStreamWriter 演示函数
         *
         */
        private static void testWrite() {
            try {
                // 创建文件“file.txt”对应File对象
                File file = new File(FileName);
                // 创建FileOutputStream对应FileWriter：将字节流转换为字符流，即写入out1的数据会自动由字节转换为字符。
                FileWriter out1 = new FileWriter(file);
                // 写入10个汉字
                out1.write("字节流转为字符流示例");
                // 向“文件中”写入"0123456789"+换行符
                out1.write("0123456789\n");

                out1.close();

            } catch(IOException e) {
                e.printStackTrace();
            }
        }

        /**
         * InputStreamReader 演示程序
         */
        private static void testRead() {
            try {
                // 方法1：新建FileInputStream对象
                // 新建文件“file.txt”对应File对象
                File file = new File(FileName);
                FileReader in1 = new FileReader(file);

                // 测试read()，从中读取一个字符
                char c1 = (char)in1.read();
                System.out.println("c1="+c1);

                // 测试skip(long byteCount)，跳过4个字符
                in1.skip(6);

                // 测试read(char[] cbuf, int off, int len)
                char[] buf = new char[10];
                in1.read(buf, 0, buf.length);
                System.out.println("buf="+(new String(buf)));

                in1.close();
            } catch(IOException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    c1=字
    buf=流示例0123456
