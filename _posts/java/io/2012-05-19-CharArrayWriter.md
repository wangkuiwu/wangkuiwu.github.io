---
layout: post
title: "java io系列19之 CharArrayWriter详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-19 09:01
---

> 本章，我们学习CharArrayWriter。学习时，我们先对CharArrayWriter有个大致了解，然后深入了解一下它的源码，最后通过示例来掌握它的用法。

> **目录**  
> [1. CharArrayWriter 介绍](#anchor1)   
> [2. Writer和CharArrayWriter源码分析](#anchor2)   
> [3. 示例代码](#anchor3)   


<a name="anchor1"></a>
# 1. CharArrayWriter 介绍

CharArrayReader 用于写入数据符，它继承于Writer。操作的数据是以字符为单位！

**CharArrayWriter 函数列表**

    CharArrayWriter()
    CharArrayWriter(int initialSize)

    CharArrayWriter     append(CharSequence csq, int start, int end)
    CharArrayWriter     append(char c)
    CharArrayWriter     append(CharSequence csq)
    void     close()
    void     flush()
    void     reset()
    int     size()
    char[]     toCharArray()
    String     toString()
    void     write(char[] buffer, int offset, int len)
    void     write(int oneChar)
    void     write(String str, int offset, int count)
    void     writeTo(Writer out)

 
<a name="anchor2"></a>
# 2. Writer和CharArrayWriter源码分析

Writer是CharArrayWriter的父类，我们先看看Writer的源码，然后再学CharArrayWriter的源码。

## 2.1 Writer源码分析(基于jdk1.7.40)

    package java.io;

    public abstract class Writer implements Appendable, Closeable, Flushable {

        private char[] writeBuffer;

        private final int writeBufferSize = 1024;

        protected Object lock;

        protected Writer() {
            this.lock = this;
        }

        protected Writer(Object lock) {
            if (lock == null) {
                throw new NullPointerException();
            }
            this.lock = lock;
        }

        public void write(int c) throws IOException {
            synchronized (lock) {
                if (writeBuffer == null){
                    writeBuffer = new char[writeBufferSize];
                }
                writeBuffer[0] = (char) c;
                write(writeBuffer, 0, 1);
            }
        }

        public void write(char cbuf[]) throws IOException {
            write(cbuf, 0, cbuf.length);
        }

        abstract public void write(char cbuf[], int off, int len) throws IOException;

        public void write(String str) throws IOException {
            write(str, 0, str.length());
        }

        public void write(String str, int off, int len) throws IOException {
            synchronized (lock) {
                char cbuf[];
                if (len <= writeBufferSize) {
                    if (writeBuffer == null) {
                        writeBuffer = new char[writeBufferSize];
                    }
                    cbuf = writeBuffer;
                } else {    // Don't permanently allocate very large buffers.
                    cbuf = new char[len];
                }
                str.getChars(off, (off + len), cbuf, 0);
                write(cbuf, 0, len);
            }
        }

        public Writer append(CharSequence csq) throws IOException {
            if (csq == null)
                write("null");
            else
                write(csq.toString());
            return this;
        }

        public Writer append(CharSequence csq, int start, int end) throws IOException {
            CharSequence cs = (csq == null ? "null" : csq);
            write(cs.subSequence(start, end).toString());
            return this;
        }

        public Writer append(char c) throws IOException {
            write(c);
            return this;
        }

        abstract public void flush() throws IOException;

        abstract public void close() throws IOException;
    }

 

## 2.2 CharArrayWriter 源码分析(基于jdk1.7.40)

    package java.io;

    import java.util.Arrays;

    public class CharArrayWriter extends Writer {
        // 字符数组缓冲
        protected char buf[];

        // 下一个字符的写入位置
        protected int count;

        // 构造函数：默认缓冲区大小是32
        public CharArrayWriter() {
            this(32);
        }

        // 构造函数：指定缓冲区大小是initialSize
        public CharArrayWriter(int initialSize) {
            if (initialSize < 0) {
                throw new IllegalArgumentException("Negative initial size: "
                                                   + initialSize);
            }
            buf = new char[initialSize];
        }

        // 写入一个字符c到CharArrayWriter中
        public void write(int c) {
            synchronized (lock) {
                int newcount = count + 1;
                if (newcount > buf.length) {
                    buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
                }
                buf[count] = (char)c;
                count = newcount;
            }
        }

        // 写入字符数组c到CharArrayWriter中。off是“字符数组b中的起始写入位置”，len是写入的长度
        public void write(char c[], int off, int len) {
            if ((off < 0) || (off > c.length) || (len < 0) ||
                ((off + len) > c.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return;
            }
            synchronized (lock) {
                int newcount = count + len;
                if (newcount > buf.length) {
                    buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
                }
                System.arraycopy(c, off, buf, count, len);
                count = newcount;
            }
        }

        // 写入字符串str到CharArrayWriter中。off是“字符串的起始写入位置”，len是写入的长度
        public void write(String str, int off, int len) {
            synchronized (lock) {
                int newcount = count + len;
                if (newcount > buf.length) {
                    buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
                }
                str.getChars(off, off + len, buf, count);
                count = newcount;
            }
        }

        // 将CharArrayWriter写入到“Writer对象out”中
        public void writeTo(Writer out) throws IOException {
            synchronized (lock) {
                out.write(buf, 0, count);
            }
        }

        // 将csq写入到CharArrayWriter中
        // 注意：该函数返回CharArrayWriter对象
        public CharArrayWriter append(CharSequence csq) {
            String s = (csq == null ? "null" : csq.toString());
            write(s, 0, s.length());
            return this;
        }

        // 将csq从start开始(包括)到end结束(不包括)的数据，写入到CharArrayWriter中。
        // 注意：该函数返回CharArrayWriter对象！ 
        public CharArrayWriter append(CharSequence csq, int start, int end) {
            String s = (csq == null ? "null" : csq).subSequence(start, end).toString();
            write(s, 0, s.length());
            return this;
        }

        // 将字符c追加到CharArrayWriter中！
        // 注意：它与write(int c)的区别。append(char c)会返回CharArrayWriter对象。
        public CharArrayWriter append(char c) {
            write(c);
            return this;
        }

        // 重置
        public void reset() {
            count = 0;
        }

        // 将CharArrayWriter的全部数据对应的char[]返回
        public char toCharArray()[] {
            synchronized (lock) {
                return Arrays.copyOf(buf, count);
            }
        }

        // 返回CharArrayWriter的大小
        public int size() {
            return count;
        }

        public String toString() {
            synchronized (lock) {
                return new String(buf, 0, count);
            }
        }

        public void flush() { }

        public void close() { }
    }

说明： CharArrayWriter实际上是将数据写入到“字符数组”中去。  
(01) 通过CharArrayWriter()创建的CharArrayWriter对应的字符数组大小是32。  
(02) 通过CharArrayWriter(int size) 创建的CharArrayWriter对应的字符数组大小是size。  
(03) write(int oneChar)的作用将int类型的oneChar换成char类型，然后写入到CharArrayWriter中。  
(04) write(char[] buffer, int offset, int len) 是将字符数组buffer写入到输出流中，offset是从buffer中读取数据的起始偏移位置，len是读取的长度。  
(05) write(String str, int offset, int count) 是将字符串str写入到输出流中，offset是从str中读取数据的起始位置，count是读取的长度。  
(06) append(char c)的作用将char类型的c写入到CharArrayWriter中，然后返回CharArrayWriter对象。  
&nbsp;&nbsp;&nbsp;&nbsp; 注意：append(char c)与write(int c)都是将单个字符写入到CharArrayWriter中。它们的区别是，append(char c)会返回CharArrayWriter对象，但是write(int c)返回void。  
(07) append(CharSequence csq, int start, int end)的作用将csq从start开始(包括)到end结束(不包括)的数据，写入到CharArrayWriter中。  
&nbsp;&nbsp;&nbsp;&nbsp; 注意：该函数返回CharArrayWriter对象！  
(08) append(CharSequence csq)的作用将csq写入到CharArrayWriter中。  
&nbsp;&nbsp;&nbsp;&nbsp; 注意：该函数返回CharArrayWriter对象！  
(09) writeTo(OutputStream out) 将该“字符数组输出流”的数据全部写入到“输出流out”中。

 
<a name="anchor3"></a>
# 3. 示例代码

关于CharArrayWriter中API的详细用法，参考示例代码(CharArrayWriterTest.java)：

    import java.io.CharArrayReader;
    import java.io.CharArrayWriter;
    import java.io.IOException;


    /**
     * CharArrayWriter 测试程序
     *
     * @author skywang
     */
    public class CharArrayWriterTest {

        private static final int LEN = 5;
        // 对应英文字母“abcdefghijklmnopqrstuvwxyz”
        private static final char[] ArrayLetters = new char[] {'a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z'};

        public static void main(String[] args) {

            tesCharArrayWriter() ;
        }

        /**
         * CharArrayWriter的API测试函数
         */
        private static void tesCharArrayWriter() {
            try {
                // 创建CharArrayWriter字符流
                CharArrayWriter caw = new CharArrayWriter();

                // 写入“A”个字符
                caw.write('A');
                // 写入字符串“BC”个字符
                caw.write("BC");
                //System.out.printf("caw=%s\n", caw);
                // 将ArrayLetters数组中从“3”开始的后5个字符(defgh)写入到caw中。
                caw.write(ArrayLetters, 3, 5);
                //System.out.printf("caw=%s\n", caw);

                // (01) 写入字符0
                // (02) 然后接着写入“123456789”
                // (03) 再接着写入ArrayLetters中第8-12个字符(ijkl)
                caw.append('0').append("123456789").append(String.valueOf(ArrayLetters), 8, 12);

                System.out.printf("caw=%s\n", caw);

                // 计算长度
                int size = caw.size();
                System.out.printf("size=%s\n", size);

                // 转换成byte[]数组
                char[] buf = caw.toCharArray();
                System.out.printf("buf=%s\n", String.valueOf(buf));

                // 将caw写入到另一个输出流中
                CharArrayWriter caw2 = new CharArrayWriter();
                caw.writeTo(caw2);
                System.out.printf("caw2=%s\n", caw2);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    caw=ABCdefgh0123456789ijkl
    size=22
    buf=ABCdefgh0123456789ijkl
    caw2=ABCdefgh0123456789ijkl

