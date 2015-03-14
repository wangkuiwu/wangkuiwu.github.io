---
layout: post
title: "java io系列24之 BufferedWriter详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-24 09:01
---

> **目录**  
[1. BufferedWriter 介绍](#anchor1)   
[2. BufferedWriter 源码分析(基于jdk1.7.40)](#anchor2)   
[3. 示例代码](#anchor3)   


<a name="anchor1"></a>
# 1. BufferedWriter 介绍

BufferedWriter 是缓冲字符输出流。它继承于Writer。  
BufferedWriter 的作用是为其他字符输出流添加一些缓冲功能。

**BufferedWriter 函数列表**

    // 构造函数
    BufferedWriter(Writer out) 
    BufferedWriter(Writer out, int sz) 
     
    void    close()                              // 关闭此流，但要先刷新它。
    void    flush()                              // 刷新该流的缓冲。
    void    newLine()                            // 写入一个行分隔符。
    void    write(char[] cbuf, int off, int len) // 写入字符数组的某一部分。
    void    write(int c)                         // 写入单个字符。
    void    write(String s, int off, int len)    // 写入字符串的某一部分。

 
<a name="anchor2"></a>
# 2. BufferedWriter 源码分析(基于jdk1.7.40)

    package java.io;

    public class BufferedWriter extends Writer {

        // 输出流对象
        private Writer out;

        // 保存“缓冲输出流”数据的字符数组
        private char cb[];

        // nChars 是cb缓冲区中字符的总的个数
        // nextChar 是下一个要读取的字符在cb缓冲区中的位置
        private int nChars, nextChar;

        // 默认字符缓冲区大小
        private static int defaultCharBufferSize = 8192;

        // 行分割符
        private String lineSeparator;

        // 构造函数，传入“Writer对象”，默认缓冲区大小是8k
        public BufferedWriter(Writer out) {
            this(out, defaultCharBufferSize);
        }

        // 构造函数，传入“Writer对象”，指定缓冲区大小是sz
        public BufferedWriter(Writer out, int sz) {
            super(out);
            if (sz <= 0)
                throw new IllegalArgumentException("Buffer size <= 0");
            this.out = out;
            cb = new char[sz];
            nChars = sz;
            nextChar = 0;

            lineSeparator = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction("line.separator"));
        }

        // 确保“BufferedWriter”是打开状态
        private void ensureOpen() throws IOException {
            if (out == null)
                throw new IOException("Stream closed");
        }

        // 对缓冲区执行flush()操作，将缓冲区的数据写入到Writer中
        void flushBuffer() throws IOException {
            synchronized (lock) {
                ensureOpen();
                if (nextChar == 0)
                    return;
                out.write(cb, 0, nextChar);
                nextChar = 0;
            }
        }

        // 将c写入到缓冲区中。先将c转换为char，然后将其写入到缓冲区。
        public void write(int c) throws IOException {
            synchronized (lock) {
                ensureOpen();
                // 若缓冲区满了，则清空缓冲，将缓冲数据写入到输出流中。
                if (nextChar >= nChars)
                    flushBuffer();
                cb[nextChar++] = (char) c;
            }
        }

        // 返回a，b中较小的数
        private int min(int a, int b) {
            if (a < b) return a;
            return b;
        }

        // 将字符数组cbuf写入到缓冲中，从cbuf的off位置开始写入，写入长度是len。
        public void write(char cbuf[], int off, int len) throws IOException {
            synchronized (lock) {
                ensureOpen();
                if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                    ((off + len) > cbuf.length) || ((off + len) < 0)) {
                    throw new IndexOutOfBoundsException();
                } else if (len == 0) {
                    return;
                }

                if (len >= nChars) {
                    /* If the request length exceeds the size of the output buffer,
                       flush the buffer and then write the data directly.  In this
                       way buffered streams will cascade harmlessly. */
                    flushBuffer();
                    out.write(cbuf, off, len);
                    return;
                }

                int b = off, t = off + len;
                while (b < t) {
                    int d = min(nChars - nextChar, t - b);
                    System.arraycopy(cbuf, b, cb, nextChar, d);
                    b += d;
                    nextChar += d;
                    if (nextChar >= nChars)
                        flushBuffer();
                }
            }
        }

        // 将字符串s写入到缓冲中，从s的off位置开始写入，写入长度是len。
        public void write(String s, int off, int len) throws IOException {
            synchronized (lock) {
                ensureOpen();

                int b = off, t = off + len;
                while (b < t) {
                    int d = min(nChars - nextChar, t - b);
                    s.getChars(b, b + d, cb, nextChar);
                    b += d;
                    nextChar += d;
                    if (nextChar >= nChars)
                        flushBuffer();
                }
            }
        }

        // 将换行符写入到缓冲中
        public void newLine() throws IOException {
            write(lineSeparator);
        }

        // 清空缓冲区数据
        public void flush() throws IOException {
            synchronized (lock) {
                flushBuffer();
                out.flush();
            }
        }

        public void close() throws IOException {
            synchronized (lock) {
                if (out == null) {
                    return;
                }
                try {
                    flushBuffer();
                } finally {
                    out.close();
                    out = null;
                    cb = null;
                }
            }
        }
    }

说明： BufferedWriter的源码非常简单，这里就BufferedWriter的思想进行简单说明：BufferedWriter通过字符数组来缓冲数据，当缓冲区满或者用户调用flush()函数时，它就会将缓冲区的数据写入到输出流中。

 
<a name="anchor3"></a>
# 3. 示例代码

关于BufferedWriter中API的详细用法，参考示例代码(BufferedWriterTest.java)：

    import java.io.BufferedWriter;
    import java.io.File;
    import java.io.OutputStream;
    import java.io.FileWriter;
    import java.io.IOException;
    import java.io.FileNotFoundException;
    import java.lang.SecurityException;
    import java.util.Scanner;

    /**
     * BufferedWriter 测试程序
     *
     * @author skywang
     */
    public class BufferedWriterTest {

        private static final int LEN = 5;
        // 对应英文字母“abcdefghijklmnopqrstuvwxyz”
        //private static final char[] ArrayLetters = "abcdefghijklmnopqrstuvwxyz";
        private static final char[] ArrayLetters = new char[] {'a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z'};

        public static void main(String[] args) {
            testBufferedWriter() ;
        }

        /**
         * BufferedWriter的API测试函数
         */
        private static void testBufferedWriter() {

            // 创建“文件输出流”对应的BufferedWriter
            // 它对应缓冲区的大小是16，即缓冲区的数据>=16时，会自动将缓冲区的内容写入到输出流。
            try {
                File file = new File("bufferwriter.txt");
                BufferedWriter out =
                      new BufferedWriter(
                          new FileWriter(file));

                // 将ArrayLetters数组的前10个字符写入到输出流中
                out.write(ArrayLetters, 0, 10);
                // 将“换行符\n”写入到输出流中
                out.write('\n');

                out.flush();
                //readUserInput() ;

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

运行结果： 生成文件“bufferwriter.txt”，文件的内容是“abcdefghij”。
