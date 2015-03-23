---
layout: post
title: "java io系列13之 BufferedOutputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-13 09:01
---

> **目录**  
[1. BufferedOutputStream 介绍](#anchor1)   
[2. BufferedOutputStream 源码分析(基于jdk1.7.40) ](#anchor2)   
[3. 示例代码](#anchor3)   





<a name="anchor1"></a>
# 1. BufferedOutputStream 介绍

BufferedOutputStream 是缓冲输出流。它继承于FilterOutputStream。

BufferedOutputStream 的作用是为另一个输出流提供“缓冲功能”。

**BufferedOutputStream 函数列表**

    BufferedOutputStream(OutputStream out)
    BufferedOutputStream(OutputStream out, int size)

    synchronized void     close()
    synchronized void     flush()
    synchronized void     write(byte[] buffer, int offset, int length)
    synchronized void     write(int oneByte)

 
<a name="anchor2"></a>
# 2. BufferedOutputStream 源码分析(基于jdk1.7.40) 

    package java.io;

    public class BufferedOutputStream extends FilterOutputStream {
        // 保存“缓冲输出流”数据的字节数组
        protected byte buf[];

        // 缓冲中数据的大小
        protected int count;

        // 构造函数：新建字节数组大小为8192的“缓冲输出流”
        public BufferedOutputStream(OutputStream out) {
            this(out, 8192);
        }

        // 构造函数：新建字节数组大小为size的“缓冲输出流”
        public BufferedOutputStream(OutputStream out, int size) {
            super(out);
            if (size <= 0) {
                throw new IllegalArgumentException("Buffer size <= 0");
            }
            buf = new byte[size];
        }

        // 将缓冲数据都写入到输出流中
        private void flushBuffer() throws IOException {
            if (count > 0) {
                out.write(buf, 0, count);
                count = 0;
            }
        }

        // 将“数据b(转换成字节类型)”写入到输出流中
        public synchronized void write(int b) throws IOException {
            // 若缓冲已满，则先将缓冲数据写入到输出流中。
            if (count >= buf.length) {
                flushBuffer();
            }
            // 将“数据b”写入到缓冲中
            buf[count++] = (byte)b;
        }

        public synchronized void write(byte b[], int off, int len) throws IOException {
            // 若“写入长度”大于“缓冲区大小”，则先将缓冲中的数据写入到输出流，然后直接将数组b写入到输出流中
            if (len >= buf.length) {
                flushBuffer();
                out.write(b, off, len);
                return;
            }
            // 若“剩余的缓冲空间 不足以 存储即将写入的数据”，则先将缓冲中的数据写入到输出流中
            if (len > buf.length - count) {
                flushBuffer();
            }
            System.arraycopy(b, off, buf, count, len);
            count += len;
        }

        // 将“缓冲数据”写入到输出流中
        public synchronized void flush() throws IOException {
            flushBuffer();
            out.flush();
        }
    }

说明： BufferedOutputStream的源码非常简单，这里就BufferedOutputStream的思想进行简单说明：BufferedOutputStream通过字节数组来缓冲数据，当缓冲区满或者用户调用flush()函数时，它就会将缓冲区的数据写入到输出流中。

 
<a name="anchor3"></a>
# 3. 示例代码

关于BufferedOutputStream中API的详细用法，参考示例代码(BufferedOutputStreamTest.java)：

    import java.io.BufferedOutputStream;
    import java.io.File;
    import java.io.OutputStream;
    import java.io.FileOutputStream;
    import java.io.IOException;
    import java.io.FileNotFoundException;
    import java.lang.SecurityException;
    import java.util.Scanner;

    /**
     * BufferedOutputStream 测试程序
     *
     * @author skywang
     */
    public class BufferedOutputStreamTest {

        private static final int LEN = 5;
        // 对应英文字母“abcddefghijklmnopqrsttuvwxyz”
        private static final byte[] ArrayLetters = {
            0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68, 0x69, 0x6A, 0x6B, 0x6C, 0x6D, 0x6E, 0x6F,
            0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x7A
        };

        public static void main(String[] args) {
            testBufferedOutputStream() ;
        }

        /**
         * BufferedOutputStream的API测试函数
         */
        private static void testBufferedOutputStream() {

            // 创建“文件输出流”对应的BufferedOutputStream
            // 它对应缓冲区的大小是16，即缓冲区的数据>=16时，会自动将缓冲区的内容写入到输出流。
            try {
                File file = new File("out.txt");
                OutputStream out =
                      new BufferedOutputStream(
                          new FileOutputStream(file), 16);

                // 将ArrayLetters数组的前10个字节写入到输出流中
                out.write(ArrayLetters, 0, 10);
                // 将“换行符\n”写入到输出流中
                out.write('\n');

                // TODO!
                //out.flush();

                readUserInput() ;

                out.close();
           } catch (FileNotFoundException e) {
               e.printStackTrace();
           } catch (SecurityException e) {
               e.printStackTrace();
           } catch (IOException e) {
               e.printStackTrace();
           }
        }

        /**
         * 读取用户输入
         */
        private static void readUserInput() {
            System.out.println("please input a text:");
            Scanner reader=new Scanner(System.in);
            // 等待一个输入
            String str = reader.next();
            System.out.printf("the input is : %s\n", str);
        }
    }

**运行结果**

生成文件“out.txt”，文件的内容是“abcdefghij”。

**分步测试**：
分别按照下面3种步骤测试程序,来查看缓冲区大小以及flush()的作用。

 

**第1种**：原始程序

(01) 运行程序。在程序等待用户输入时，查看“out.txt”的文本内容；发现：内容为空。  
(02) 运行程序。在用户输入之后，查看“out.txt”的文本内容；发现：内容为“abcdefghij”。  
从中，我们发现(01)和(02)的结果不同；之所以(01)中的out.txt内容为空，是因为out.txt对应的缓冲区大小是16字节，而我们只写入了11个字节，所以，它不会执行清空缓冲操作(即，将缓冲数据写入到输出流中)。  
而(02)对应out.txt的内容是“abcdefghij”，是因为执行了out.close()，它会关闭输出流；在关闭输出流之前，会将缓冲区的数据写入到输出流中。

注意：重新测试时，要先删除out.txt。

 

**第2种**：在readUserInput()前添加如下语句

    out.flush();

这句话的作用，是将“缓冲区的内容”写入到输出流中。  
(01) 运行程序。在程序等待用户输入时，查看“out.txt”的文本内容；发现：内容为“abcdefghij”。  
(02) 运行程序。在用户输入之后，查看“out.txt”的文本内容；发现：内容为“abcdefghij”。  
从中，我们发现(01)和(02)结果一样，对应out.txt的内容都是“abcdefghij”。这是因为执行了flush()操作，它的作用是将缓冲区的数据写入到输出流中。  

注意：重新测试时，要先删除out.txt！


**第3种**：在第1种的基础上，将

    out.write(ArrayLetters, 0, 10);

修改为

    out.write(ArrayLetters, 0, 20);

(01) 运行程序。在程序等待用户输入时，查看“out.txt”的文本内容；发现：内容为“abcdefghijklmnopqrst”(不含回车)。  
(02) 运行程序。在用户输入之后，查看“out.txt”的文本内容；发现：内容为“abcdefghijklmnopqrst”(含回车)。  
从中，我们发现(01)运行结果是“abcdefghijklmnopqrst”(不含回车)。这是因为，缓冲区的大小是16，而我们通过out.write(ArrayLetters, 0, 20)写入了20个字节，超过了缓冲区的大小；这时，会直接将全部的输入都写入都输出流中，而不经过缓冲区。  
(03)运行结果是“abcdefghijklmnopqrst”(含回车)，这是因为执行out.close()时，将回车符号'\n'写入了输出流中。

注意：重新测试时，要先删除out.txt！

 
