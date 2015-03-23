---
layout: post
title: "java io系列02之 ByteArrayInputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-02 09:01
---

> 我们以ByteArrayInputStream，拉开对字节类型的“输入流”的学习序幕。  
本章，我们会先对ByteArrayInputStream进行介绍，然后深入了解一下它的源码，最后通过示例来掌握它的用法。

> **目录**  
[1. ByteArrayInputStream 介绍](#anchor1)   
[2. InputStream和ByteArrayInputStream源码分析](#anchor2)   
[3. 示例代码](#anchor3)   

 
<a name="anchor1"></a>
# 1. ByteArrayInputStream 介绍

ByteArrayInputStream 是字节数组输入流。它继承于InputStream。

它包含一个内部缓冲区，该缓冲区包含从流中读取的字节；通俗点说，它的内部缓冲区就是一个字节数组，而ByteArrayInputStream本质就是通过字节数组来实现的。  
我们都知道，InputStream通过read()向外提供接口，供它们来读取字节数据；而ByteArrayInputStream 的内部额外的定义了一个计数器，它被用来跟踪 read() 方法要读取的下一个字节。


InputStream 函数列表

    // 构造函数
    InputStream()

                 int     available()
                 void    close()
                 void    mark(int readlimit)
                 boolean markSupported()
                 int     read(byte[] buffer)
    abstract     int     read()
                 int     read(byte[] buffer, int offset, int length)
    synchronized void    reset()
                 long    skip(long byteCount)

ByteArrayInputStream 函数列表

    // 构造函数
    ByteArrayInputStream(byte[] buf)
    ByteArrayInputStream(byte[] buf, int offset, int length)

    synchronized int         available()
                 void        close()
    synchronized void        mark(int readlimit)
                 boolean     markSupported()
    synchronized int         read()
    synchronized int         read(byte[] buffer, int offset, int length)
    synchronized void        reset()
    synchronized long        skip(long byteCount)



<a name="anchor2"></a>
# 2. InputStream和ByteArrayInputStream源码分析

InputStream是ByteArrayInputStream的父类，我们先看看InputStream的源码，然后再学ByteArrayInputStream的源码。

**1. InputStream.java源码分析(基于jdk1.7.40)**

    package java.io;

    public abstract class InputStream implements Closeable {

        // 能skip的大小
        private static final int MAX_SKIP_BUFFER_SIZE = 2048;

        // 从输入流中读取数据的下一个字节。
        public abstract int read() throws IOException;

        // 将数据从输入流读入 byte 数组。
        public int read(byte b[]) throws IOException {
            return read(b, 0, b.length);
        }

        // 将最多 len 个数据字节从此输入流读入 byte 数组。
        public int read(byte b[], int off, int len) throws IOException {
            if (b == null) {
                throw new NullPointerException();
            } else if (off < 0 || len < 0 || len > b.length - off) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }

            int c = read();
            if (c == -1) {
                return -1;
            }
            b[off] = (byte)c;

            int i = 1;
            try {
                for (; i < len ; i++) {
                    c = read();
                    if (c == -1) {
                        break;
                    }
                    b[off + i] = (byte)c;
                }
            } catch (IOException ee) {
            }
            return i;
        }

        // 跳过输入流中的n个字节
        public long skip(long n) throws IOException {

            long remaining = n;
            int nr;

            if (n <= 0) {
                return 0;
            }

            int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
            byte[] skipBuffer = new byte[size];
            while (remaining > 0) {
                nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
                if (nr < 0) {
                    break;
                }
                remaining -= nr;
            }

            return n - remaining;
        }

        public int available() throws IOException {
            return 0;
        }

        public void close() throws IOException {}

        public synchronized void mark(int readlimit) {}

        public synchronized void reset() throws IOException {
            throw new IOException("mark/reset not supported");
        }

        public boolean markSupported() {
            return false;
        }
    }

**2. ByteArrayInputStream.java源码分析(基于jdk1.7.40)**

    package java.io;

    public class ByteArrayInputStream extends InputStream {

        // 保存字节输入流数据的字节数组
        protected byte buf[];

        // 下一个会被读取的字节的索引
        protected int pos;

        // 标记的索引
        protected int mark = 0;

        // 字节流的长度
        protected int count;

        // 构造函数：创建一个内容为buf的字节流
        public ByteArrayInputStream(byte buf[]) {
            // 初始化“字节流对应的字节数组为buf”
            this.buf = buf;
            // 初始化“下一个要被读取的字节索引号为0”
            this.pos = 0;
            // 初始化“字节流的长度为buf的长度”
            this.count = buf.length;
        }

        // 构造函数：创建一个内容为buf的字节流，并且是从offset开始读取数据，读取的长度为length
        public ByteArrayInputStream(byte buf[], int offset, int length) {
            // 初始化“字节流对应的字节数组为buf”
            this.buf = buf;
            // 初始化“下一个要被读取的字节索引号为offset”
            this.pos = offset;
            // 初始化“字节流的长度”
            this.count = Math.min(offset + length, buf.length);
            // 初始化“标记的字节流读取位置”
            this.mark = offset;
        }

        // 读取下一个字节
        public synchronized int read() {
            return (pos < count) ? (buf[pos++] & 0xff) : -1;
        }

        // 将“字节流的数据写入到字节数组b中”
        // off是“字节数组b的偏移地址”，表示从数组b的off开始写入数据
        // len是“写入的字节长度”
        public synchronized int read(byte b[], int off, int len) {
            if (b == null) {
                throw new NullPointerException();
            } else if (off < 0 || len < 0 || len > b.length - off) {
                throw new IndexOutOfBoundsException();
            }

            if (pos >= count) {
                return -1;
            }

            int avail = count - pos;
            if (len > avail) {
                len = avail;
            }
            if (len <= 0) {
                return 0;
            }
            System.arraycopy(buf, pos, b, off, len);
            pos += len;
            return len;
        }

        // 跳过“字节流”中的n个字节。
        public synchronized long skip(long n) {
            long k = count - pos;
            if (n < k) {
                k = n < 0 ? 0 : n;
            }

            pos += k;
            return k;
        }

        // “能否读取字节流的下一个字节”
        public synchronized int available() {
            return count - pos;
        }

        // 是否支持“标签”
        public boolean markSupported() {
            return true;
        }

        // 保存当前位置。readAheadLimit在此处没有任何实际意义
        public void mark(int readAheadLimit) {
            mark = pos;
        }

        // 重置“字节流的读取索引”为“mark所标记的位置”
        public synchronized void reset() {
            pos = mark;
        }

        public void close() throws IOException {
        }
    }

说明：  
ByteArrayInputStream实际上是通过“字节数组”去保存数据。  
(01) 通过ByteArrayInputStream(byte buf[]) 或 ByteArrayInputStream(byte buf[], int offset, int length) ，我们可以根据buf数组来创建字节流对象。  
(02) read()的作用是从字节流中“读取下一个字节”。  
(03) read(byte[] buffer, int offset, int length)的作用是从字节流读取字节数据，并写入到字节数组buffer中。offset是将字节写入到buffer的起始位置，length是写入的字节的长度。  
(04) markSupported()是判断字节流是否支持“标记功能”。它一直返回true。  
(05) mark(int readlimit)的作用是记录标记位置。记录标记位置之后，某一时刻调用reset()则将“字节流下一个被读取的位置”重置到“mark(int readlimit)所标记的位置”；也就是说，reset()之后再读取字节流时，是从mark(int readlimit)所标记的位置开始读取。



<a name="anchor3"></a>
# 3. 示例代码

关于ByteArrayInputStream中API的详细用法，参考示例代码(ByteArrayInputStreamTest.java)：

    import java.io.ByteArrayInputStream;
    import java.io.ByteArrayOutputStream;

    /**
     * ByteArrayInputStream 测试程序
     *
     * @author skywang
     */
    public class ByteArrayInputStreamTest {

        private static final int LEN = 5;
        // 对应英文字母“abcddefghijklmnopqrsttuvwxyz”
        private static final byte[] ArrayLetters = {
            0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68, 0x69, 0x6A, 0x6B, 0x6C, 0x6D, 0x6E, 0x6F,
            0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x7A
        };

        public static void main(String[] args) {
            String tmp = new String(ArrayLetters);
            System.out.println("ArrayLetters="+tmp);

            tesByteArrayInputStream() ;
        }

        /**
         * ByteArrayInputStream的API测试函数
         */
        private static void tesByteArrayInputStream() {
            // 创建ByteArrayInputStream字节流，内容是ArrayLetters数组
            ByteArrayInputStream bais = new ByteArrayInputStream(ArrayLetters);

            // 从字节流中读取5个字节
            for (int i=0; i<LEN; i++) {
                // 若能继续读取下一个字节，则读取下一个字节
                if (bais.available() >= 0) {
                    // 读取“字节流的下一个字节”
                    int tmp = bais.read();
                    System.out.printf("%d : 0x%s\n", i, Integer.toHexString(tmp));
                }
            }

            // 若“该字节流”不支持标记功能，则直接退出
            if (!bais.markSupported()) {
                System.out.println("make not supported!");
                return ;
            }

            // 标记“字节流中下一个被读取的位置”。即--标记“0x66”，因为因为前面已经读取了5个字节，所以下一个被读取的位置是第6个字节”
            // (01), ByteArrayInputStream类的mark(0)函数中的“参数0”是没有实际意义的。
            // (02), mark()与reset()是配套的，reset()会将“字节流中下一个被读取的位置”重置为“mark()中所保存的位置”
            bais.mark(0);

            // 跳过5个字节。跳过5个字节后，字节流中下一个被读取的值应该是“0x6B”。
            bais.skip(5);

            // 从字节流中读取5个数据。即读取“0x6B, 0x6C, 0x6D, 0x6E, 0x6F”
            byte[] buf = new byte[LEN];
            bais.read(buf, 0, LEN);
            // 将buf转换为String字符串。“0x6B, 0x6C, 0x6D, 0x6E, 0x6F”对应字符是“klmno”
            String str1 = new String(buf);
            System.out.printf("str1=%s\n", str1);
        
            // 重置“字节流”：即，将“字节流中下一个被读取的位置”重置到“mark()所标记的位置”，即0x66。
            bais.reset();
            // 从“重置后的字节流”中读取5个字节到buf中。即读取“0x66, 0x67, 0x68, 0x69, 0x6A”
            bais.read(buf, 0, LEN);
            // 将buf转换为String字符串。“0x66, 0x67, 0x68, 0x69, 0x6A”对应字符是“fghij”
            String str2 = new String(buf);
            System.out.printf("str2=%s\n", str2);
        }
    }

运行结果：

    ArrayLetters=abcdefghijklmnopqrstuvwxyz
    0 : 0x61
    1 : 0x62
    2 : 0x63
    3 : 0x64
    4 : 0x65
    str1=klmno
    str2=fghij

结果说明：

(01) ArrayLetters 是字节数组。0x61对应的ASCII码值是a，0x62对应的ASCII码值是b，依次类推...  
(02) ByteArrayInputStream bais = new ByteArrayInputStream(ArrayLetters); 这句话是创建“字节流bais”，它的内容就是ArrayLetters。  
(03) for (int i=0; i<LEN; i++) ; 这个for循环的作用就是从字节流中读取5个字节。每次调用bais.read()就从字节流中读取一个字节。  
(04) bais.mark(0); 这句话就是“设置字节流的标记”，此时标记的位置对应的值是0x66。  
(05) bais.skip(5); 这句话是跳过5个字节。跳过5个字节后，对应的字节流中下一个被读取的字节的值是0x6B。  
(06) bais.read(buf, 0, LEN); 这句话是“从字节流中读取LEN个数据写入到buf中，0表示从buf的第0个位置开始写入”。  
(07) bais.reset(); 这句话是将“字节流中下一个被读取的位置”重置到“mark()所标记的位置”，即0x66。  

学完了ByteArrayInputStream输入流。下面，我们学习与之对应的输出流ByteArrayOutputStream。
