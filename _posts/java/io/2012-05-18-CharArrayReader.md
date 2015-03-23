---
layout: post
title: "java io系列18之 CharArrayReader详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-18 09:01
---

> 从本章开始，我们开始对“字符流”进行学习。首先，要学习的是CharArrayReader。学习时，我们先对CharArrayReader有个大致了解，然后深入了解一下它的源码，最后通过示例来掌握它的用法。

> **目录**  
[1. CharArrayReader 介绍](#anchor1)   
[2. Reader和CharArrayReader源码分析](#anchor2)   
[3. 示例代码](#anchor3)   



<a name="anchor1"></a>
# 1. CharArrayReader 介绍

CharArrayReader 是字符数组输入流。它和ByteArrayInputStream类似，只不过ByteArrayInputStream是字节数组输入流，而CharArray是字符数组输入流。CharArrayReader 是用于读取字符数组，它继承于Reader。操作的数据是以字符为单位！


**CharArrayReader 函数列表**

    CharArrayReader(char[] buf)
    CharArrayReader(char[] buf, int offset, int length)

    void      close()
    void      mark(int readLimit)
    boolean   markSupported()
    int       read()
    int       read(char[] buffer, int offset, int len)
    boolean   ready()
    void      reset()
    long      skip(long charCount)

 
<a name="anchor2"></a>
# 2. Reader和CharArrayReader源码分析

Reader是CharArrayReader的父类，我们先看看Reader的源码，然后再学CharArrayReader的源码。

## 2.1 Reader源码分析(基于jdk1.7.40)

    package java.io;

    public abstract class Reader implements Readable, Closeable {

        protected Object lock;

        protected Reader() {
            this.lock = this;
        }

        protected Reader(Object lock) {
            if (lock == null) {
                throw new NullPointerException();
            }
            this.lock = lock;
        }

        public int read(java.nio.CharBuffer target) throws IOException {
            int len = target.remaining();
            char[] cbuf = new char[len];
            int n = read(cbuf, 0, len);
            if (n > 0)
                target.put(cbuf, 0, n);
            return n;
        }

        public int read() throws IOException {
            char cb[] = new char[1];
            if (read(cb, 0, 1) == -1)
                return -1;
            else
                return cb[0];
        }

        public int read(char cbuf[]) throws IOException {
            return read(cbuf, 0, cbuf.length);
        }

        abstract public int read(char cbuf[], int off, int len) throws IOException;

        private static final int maxSkipBufferSize = 8192;

        private char skipBuffer[] = null;

        public long skip(long n) throws IOException {
            if (n < 0L)
                throw new IllegalArgumentException("skip value is negative");
            int nn = (int) Math.min(n, maxSkipBufferSize);
            synchronized (lock) {
                if ((skipBuffer == null) || (skipBuffer.length < nn))
                    skipBuffer = new char[nn];
                long r = n;
                while (r > 0) {
                    int nc = read(skipBuffer, 0, (int)Math.min(r, nn));
                    if (nc == -1)
                        break;
                    r -= nc;
                }
                return n - r;
            }
        }

        public boolean ready() throws IOException {
            return false;
        }

        public boolean markSupported() {
            return false;
        }

        public void mark(int readAheadLimit) throws IOException {
            throw new IOException("mark() not supported");
        }

        public void reset() throws IOException {
            throw new IOException("reset() not supported");
        }

         abstract public void close() throws IOException;
    }

## 2.2 CharArrayReader 源码分析(基于jdk1.7.40)

    package java.io;

    public class CharArrayReader extends Reader {
        // 字符数组缓冲
        protected char buf[];

        // 下一个被获取的字符的位置
        protected int pos;

        // 被标记的位置
        protected int markedPos = 0;

        // 字符缓冲的长度
        protected int count;

        // 构造函数
        public CharArrayReader(char buf[]) {
            this.buf = buf;
            this.pos = 0;
            this.count = buf.length;
        }

        // 构造函数
        public CharArrayReader(char buf[], int offset, int length) {
            if ((offset < 0) || (offset > buf.length) || (length < 0) ||
                ((offset + length) < 0)) {
                throw new IllegalArgumentException();
            }
            this.buf = buf;
            this.pos = offset;
            this.count = Math.min(offset + length, buf.length);
            this.markedPos = offset;
        }

        // 判断“CharArrayReader是否有效”。
        // 若字符缓冲为null，则认为其无效。
        private void ensureOpen() throws IOException {
            if (buf == null)
                throw new IOException("Stream closed");
        }

        // 读取下一个字符。即返回字符缓冲区中下一位置的值。
        // 注意：读取的是字符！
        public int read() throws IOException {
            synchronized (lock) {
                ensureOpen();
                if (pos >= count)
                    return -1;
                else
                    return buf[pos++];
            }
        }

        // 读取数据，并保存到字符数组b中从off开始的位置中，len是读取长度。
        public int read(char b[], int off, int len) throws IOException {
            synchronized (lock) {
                ensureOpen();
                if ((off < 0) || (off > b.length) || (len < 0) ||
                    ((off + len) > b.length) || ((off + len) < 0)) {
                    throw new IndexOutOfBoundsException();
                } else if (len == 0) {
                    return 0;
                }

                if (pos >= count) {
                    return -1;
                }
                if (pos + len > count) {
                    len = count - pos;
                }
                if (len <= 0) {
                    return 0;
                }
                System.arraycopy(buf, pos, b, off, len);
                pos += len;
                return len;
            }
        }

        // 跳过n个字符
        public long skip(long n) throws IOException {
            synchronized (lock) {
                ensureOpen();
                if (pos + n > count) {
                    n = count - pos;
                }
                if (n < 0) {
                    return 0;
                }
                pos += n;
                return n;
            }
        }

        // 判断“是否能读取下一个字符”。能的话，返回true。
        public boolean ready() throws IOException {
            synchronized (lock) {
                ensureOpen();
                return (count - pos) > 0;
            }
        }

        public boolean markSupported() {
            return true;
        }

        // 保存当前位置。readAheadLimit在此处没有任何实际意义
        // mark()必须和reset()配合使用才有意义！
        public void mark(int readAheadLimit) throws IOException {
            synchronized (lock) {
                ensureOpen();
                markedPos = pos;
            }
        }

        // 重置“下一个读取位置”为“mark所标记的位置”
        public void reset() throws IOException {
            synchronized (lock) {
                ensureOpen();
                pos = markedPos;
            }
        }

        public void close() {
            buf = null;
        }
    }

说明：CharArrayReader实际上是通过“字符数组”去保存数据。

(01) 通过 CharArrayReader(char[] buf) 或 CharArrayReader(char[] buf, int offset, int length) ，我们可以根据buf数组来创建CharArrayReader对象。  
(02) read()的作用是从CharArrayReader中“读取下一个字符”。  
(03) read(char[] buffer, int offset, int len)的作用是从CharArrayReader读取字符数据，并写入到字符数组buffer中。offset是将字符写入到buffer的起始位置，len是写入的字符的长度。  
(04) markSupported()是判断CharArrayReader是否支持“标记功能”。它始终返回true。  
(05) mark(int readlimit)的作用是记录标记位置。记录标记位置之后，某一时刻调用reset()则将“CharArrayReader下一个被读取的位置”重置到“mark(int readlimit)所标记的位置”；也就是说，reset()之后再读取CharArrayReader时，是从mark(int readlimit)所标记的位置开始读取。  

 
<a name="anchor3"></a>
# 3. 示例代码

关于CharArrayReader中API的详细用法，参考示例代码(CharArrayReaderTest.java)： 

    /**
     * CharArrayReader 演示程序
     *
     * @author skywang
     */

    import java.io.CharArrayReader;
    import java.io.CharArrayWriter;
    import java.io.IOException;

    public class CharArrayReaderTest {

        private static final int LEN = 5;
        // 对应英文字母“abcdefghijklmnopqrstuvwxyz”
        private static final char[] ArrayLetters = new char[] {'a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z'};
        
        public static void main(String[] args) {
            tesCharArrayReader() ;
        }

        /**
         * CharArrayReader的API测试函数
         */
        private static void tesCharArrayReader() {
            try {
                // 创建CharArrayReader字符流，内容是ArrayLetters数组
                CharArrayReader car = new CharArrayReader(ArrayLetters);

                // 从字符数组流中读取5个字符
                for (int i=0; i<LEN; i++) {
                    // 若能继续读取下一个字符，则读取下一个字符
                    if (car.ready() == true) {
                        // 读取“字符流的下一个字符”
                        char tmp = (char)car.read();
                        System.out.printf("%d : %c\n", i, tmp);
                    }
                }

                // 若“该字符流”不支持标记功能，则直接退出
                if (!car.markSupported()) {
                    System.out.println("make not supported!");
                    return ;
                }

                // 标记“字符流中下一个被读取的位置”。即--标记“f”，因为因为前面已经读取了5个字符，所以下一个被读取的位置是第6个字符”
                // (01), CharArrayReader类的mark(0)函数中的“参数0”是没有实际意义的。
                // (02), mark()与reset()是配套的，reset()会将“字符流中下一个被读取的位置”重置为“mark()中所保存的位置”
                car.mark(0);

                // 跳过5个字符。跳过5个字符后，字符流中下一个被读取的值应该是“k”。
                car.skip(5);

                // 从字符流中读取5个数据。即读取“klmno”
                char[] buf = new char[LEN];
                car.read(buf, 0, LEN);
                System.out.printf("buf=%s\n", String.valueOf(buf));
            
                // 重置“字符流”：即，将“字符流中下一个被读取的位置”重置到“mark()所标记的位置”，即f。
                car.reset();
                // 从“重置后的字符流”中读取5个字符到buf中。即读取“fghij”
                car.read(buf, 0, LEN);
                System.out.printf("buf=%s\n", String.valueOf(buf));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    0 : a
    1 : b
    2 : c
    3 : d
    4 : e
    buf=klmno
    buf=fghij

