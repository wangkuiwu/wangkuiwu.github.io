---
layout: post
title: "java io系列12之 BufferedInputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-12 09:01
---

> **目录**  
[1. BufferedInputStream 介绍](#anchor1)   
[2. BufferedInputStream 源码分析(基于jdk1.7.40)](#anchor2)   
[3. 示例代码](#anchor3)   


<a name="anchor1"></a>
# 1. BufferedInputStream 介绍

BufferedInputStream 是缓冲输入流。它继承于FilterInputStream。

BufferedInputStream 的作用是为另一个输入流添加一些功能，例如，提供“缓冲功能”以及支持“mark()标记”和“reset()重置方法”。  
BufferedInputStream 本质上是通过一个内部缓冲区数组实现的。例如，在新建某输入流对应的BufferedInputStream后，当我们通过read()读取输入流的数据时，BufferedInputStream会将该输入流的数据分批的填入到缓冲区中。每当缓冲区中的数据被读完之后，输入流会再次填充数据缓冲区；如此反复，直到我们读完输入流数据位置。


BufferedInputStream 函数列表

    BufferedInputStream(InputStream in)
    BufferedInputStream(InputStream in, int size)

    synchronized int     available()
    void     close()
    synchronized void     mark(int readlimit)
    boolean     markSupported()
    synchronized int     read()
    synchronized int     read(byte[] buffer, int offset, int byteCount)
    synchronized void     reset()
    synchronized long     skip(long byteCount)

 
<a name="anchor2"></a>
# 2. BufferedInputStream 源码分析(基于jdk1.7.40)

    package java.io;
    import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

    public class BufferedInputStream extends FilterInputStream {

        // 默认的缓冲大小是8192字节
        // BufferedInputStream 会根据“缓冲区大小”来逐次的填充缓冲区；
        // 即，BufferedInputStream填充缓冲区，用户读取缓冲区，读完之后，BufferedInputStream会再次填充缓冲区。如此循环，直到读完数据...
        private static int defaultBufferSize = 8192;

        // 缓冲数组
        protected volatile byte buf[];

        // 缓存数组的原子更新器。
        // 该成员变量与buf数组的volatile关键字共同组成了buf数组的原子更新功能实现，
        // 即，在多线程中操作BufferedInputStream对象时，buf和bufUpdater都具有原子性(不同的线程访问到的数据都是相同的)
        private static final
            AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
            AtomicReferenceFieldUpdater.newUpdater
            (BufferedInputStream.class,  byte[].class, "buf");

        // 当前缓冲区的有效字节数。
        // 注意，这里是指缓冲区的有效字节数，而不是输入流中的有效字节数。
        protected int count;

        // 当前缓冲区的位置索引
        // 注意，这里是指缓冲区的位置索引，而不是输入流中的位置索引。
        protected int pos;

        // 当前缓冲区的标记位置
        // markpos和reset()配合使用才有意义。操作步骤：
        // (01) 通过mark() 函数，保存pos的值到markpos中。
        // (02) 通过reset() 函数，会将pos的值重置为markpos。接着通过read()读取数据时，就会从mark()保存的位置开始读取。
        protected int markpos = -1;

        // marklimit是标记的最大值。
        // 关于marklimit的原理，我们在后面的fill()函数分析中会详细说明。这对理解BufferedInputStream相当重要。
        protected int marklimit;

        // 获取输入流
        private InputStream getInIfOpen() throws IOException {
            InputStream input = in;
            if (input == null)
                throw new IOException("Stream closed");
            return input;
        }

        // 获取缓冲
        private byte[] getBufIfOpen() throws IOException {
            byte[] buffer = buf;
            if (buffer == null)
                throw new IOException("Stream closed");
            return buffer;
        }

        // 构造函数：新建一个缓冲区大小为8192的BufferedInputStream
        public BufferedInputStream(InputStream in) {
            this(in, defaultBufferSize);
        }

        // 构造函数：新建指定缓冲区大小的BufferedInputStream
        public BufferedInputStream(InputStream in, int size) {
            super(in);
            if (size <= 0) {
                throw new IllegalArgumentException("Buffer size <= 0");
            }
            buf = new byte[size];
        }

        // 从“输入流”中读取数据，并填充到缓冲区中。
        // 后面会对该函数进行详细说明！
        private void fill() throws IOException {
            byte[] buffer = getBufIfOpen();
            if (markpos < 0)
                pos = 0;            /* no mark: throw away the buffer */
            else if (pos >= buffer.length)  /* no room left in buffer */
                if (markpos > 0) {  /* can throw away early part of the buffer */
                    int sz = pos - markpos;
                    System.arraycopy(buffer, markpos, buffer, 0, sz);
                    pos = sz;
                    markpos = 0;
                } else if (buffer.length >= marklimit) {
                    markpos = -1;   /* buffer got too big, invalidate mark */
                    pos = 0;        /* drop buffer contents */
                } else {            /* grow buffer */
                    int nsz = pos * 2;
                    if (nsz > marklimit)
                        nsz = marklimit;
                    byte nbuf[] = new byte[nsz];
                    System.arraycopy(buffer, 0, nbuf, 0, pos);
                    if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                        throw new IOException("Stream closed");
                    }
                    buffer = nbuf;
                }
            count = pos;
            int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
            if (n > 0)
                count = n + pos;
        }

        // 读取下一个字节
        public synchronized int read() throws IOException {
            // 若已经读完缓冲区中的数据，则调用fill()从输入流读取下一部分数据来填充缓冲区
            if (pos >= count) {
                fill();
                if (pos >= count)
                    return -1;
            }
            // 从缓冲区中读取指定的字节
            return getBufIfOpen()[pos++] & 0xff;
        }

        // 将缓冲区中的数据写入到字节数组b中。off是字节数组b的起始位置，len是写入长度
        private int read1(byte[] b, int off, int len) throws IOException {
            int avail = count - pos;
            if (avail <= 0) {
                // 加速机制。
                // 如果读取的长度大于缓冲区的长度 并且没有markpos，
                // 则直接从原始输入流中进行读取，从而避免无谓的COPY（从原始输入流至缓冲区，读取缓冲区全部数据，清空缓冲区， 
                //  重新填入原始输入流数据）
                if (len >= getBufIfOpen().length && markpos < 0) {
                    return getInIfOpen().read(b, off, len);
                }
                // 若已经读完缓冲区中的数据，则调用fill()从输入流读取下一部分数据来填充缓冲区
                fill();
                avail = count - pos;
                if (avail <= 0) return -1;
            }
            int cnt = (avail < len) ? avail : len;
            System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
            pos += cnt;
            return cnt;
        }

        // 将缓冲区中的数据写入到字节数组b中。off是字节数组b的起始位置，len是写入长度
        public synchronized int read(byte b[], int off, int len)
            throws IOException
        {
            getBufIfOpen(); // Check for closed stream
            if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }

            // 读取到指定长度的数据才返回
            int n = 0;
            for (;;) {
                int nread = read1(b, off + n, len - n);
                if (nread <= 0)
                    return (n == 0) ? nread : n;
                n += nread;
                if (n >= len)
                    return n;
                // if not closed but no bytes available, return
                InputStream input = in;
                if (input != null && input.available() <= 0)
                    return n;
            }
        }

        // 忽略n个字节
        public synchronized long skip(long n) throws IOException {
            getBufIfOpen(); // Check for closed stream
            if (n <= 0) {
                return 0;
            }
            long avail = count - pos;

            if (avail <= 0) {
                // If no mark position set then don't keep in buffer
                if (markpos <0)
                    return getInIfOpen().skip(n);

                // Fill in buffer to save bytes for reset
                fill();
                avail = count - pos;
                if (avail <= 0)
                    return 0;
            }

            long skipped = (avail < n) ? avail : n;
            pos += skipped;
            return skipped;
        }

        // 下一个字节是否存可读
        public synchronized int available() throws IOException {
            int n = count - pos;
            int avail = getInIfOpen().available();
            return n > (Integer.MAX_VALUE - avail)
                        ? Integer.MAX_VALUE
                        : n + avail;
        }

        // 标记“缓冲区”中当前位置。
        // readlimit是marklimit，关于marklimit的作用，参考后面的说明。
        public synchronized void mark(int readlimit) {
            marklimit = readlimit;
            markpos = pos;
        }

        // 将“缓冲区”中当前位置重置到mark()所标记的位置
        public synchronized void reset() throws IOException {
            getBufIfOpen(); // Cause exception if closed
            if (markpos < 0)
                throw new IOException("Resetting to invalid mark");
            pos = markpos;
        }

        public boolean markSupported() {
            return true;
        }

        // 关闭输入流
        public void close() throws IOException {
            byte[] buffer;
            while ( (buffer = buf) != null) {
                if (bufUpdater.compareAndSet(this, buffer, null)) {
                    InputStream input = in;
                    in = null;
                    if (input != null)
                        input.close();
                    return;
                }
                // Else retry in case a new buf was CASed in fill()
            }
        }
    }

说明：  
要想读懂BufferedInputStream的源码，就要先理解它的思想。BufferedInputStream的作用是为其它输入流提供缓冲功能。创建BufferedInputStream时，我们会通过它的构造函数指定某个输入流为参数。BufferedInputStream会将该输入流数据分批读取，每次读取一部分到缓冲中；操作完缓冲中的这部分数据之后，再从输入流中读取下一部分的数据。

为什么需要缓冲呢？原因很简单，效率问题！缓冲中的数据实际上是保存在内存中，而原始数据可能是保存在硬盘或NandFlash等存储介质中；而我们知道，从内存中读取数据的速度比从硬盘读取数据的速度至少快10倍以上。   
那干嘛不干脆一次性将全部数据都读取到缓冲中呢？第一，读取全部的数据所需要的时间可能会很长。第二，内存价格很贵，容量不像硬盘那么大。

下面，我就BufferedInputStream中最重要的函数fill()进行说明。其它的函数很容易理解，我就不详细介绍了，大家可以参考源码中的注释进行理解。


fill() 源码如下：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos < 0)
            pos = 0;
        else if (pos >= buffer.length) {
            if (markpos > 0) {  /* can throw away early part of the buffer */
                int sz = pos - markpos;
                System.arraycopy(buffer, markpos, buffer, 0, sz);
                pos = sz;
                markpos = 0;
            } else if (buffer.length >= marklimit) {
                markpos = -1;   /* buffer got too big, invalidate mark */
                pos = 0;        /* drop buffer contents */
            } else {            /* grow buffer */
                int nsz = pos * 2;
                if (nsz > marklimit)
                    nsz = marklimit;
                byte nbuf[] = new byte[nsz];
                System.arraycopy(buffer, 0, nbuf, 0, pos);
                if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                    // Can't replace buf if there was an async close.
                    // Note: This would need to be changed if fill()
                    // is ever made accessible to multiple threads.
                    // But for now, the only way CAS can fail is via close.
                    // assert buf == null;
                    throw new IOException("Stream closed");
                }
                buffer = nbuf;
            }
        }

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }


根据fill()中的if...else...，下面我们将fill分为5种情况进行说明。

 

## 情况1：读取完buffer中的数据，并且buffer没有被标记

执行流程如下，  
(01) read() 函数中调用 fill()  
(02) fill() 中的 if (markpos < 0) ...  
为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos < 0)
            pos = 0;

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

说明：  
这种情况发生的情况是 — — 输入流中有很长的数据，我们每次从中读取一部分数据到buffer中进行操作。每次当我们读取完buffer中的数据之后，并且此时输入流没有被标记；那么，就接着从输入流中读取下一部分的数据到buffer中。  
> 判断是否读完buffer中的数据，是通过 if (pos >= count) 来判断的；  
  判断输入流有没有被标记，是通过 if (markpos < 0) 来判断的。

理解这个思想之后，我们再对这种情况下的fill()的代码进行分析，就特别容易理解了。  
(01) if (markpos < 0) 它的作用是判断“输入流是否被标记”。若被标记，则markpos大于/等于0；否则markpos等于-1。  
(02) 在这种情况下：通过getInIfOpen()获取输入流，然后接着从输入流中读取buffer.length个字节到buffer中。  
(03) count = n + pos; 这是根据从输入流中读取的实际数据的多少，来更新buffer中数据的实际大小。

 

## 情况2：读取完buffer中的数据，buffer的标记位置>0，并且buffer中没有多余的空间

执行流程如下，  
(01) read() 函数中调用 fill()  
(02) fill() 中的 else if (pos >= buffer.length) ...  
(03) fill() 中的 if (markpos > 0) ...

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos >= 0 && pos >= buffer.length) {
            if (markpos > 0) {
                int sz = pos - markpos;
                System.arraycopy(buffer, markpos, buffer, 0, sz);
                pos = sz;
                markpos = 0;
            }
        }

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

说明：  
这种情况发生的情况是 — — 输入流中有很长的数据，我们每次从中读取一部分数据到buffer中进行操作。当我们读取完buffer中的数据之后，并且此时输入流存在标记时；那么，就发生情况2。此时，我们要保留“被标记位置”到“buffer末尾”的数据，然后再从输入流中读取下一部分的数据到buffer中。  
>  其中，判断是否读完buffer中的数据，是通过 if (pos >= count) 来判断的；  
  判断输入流有没有被标记，是通过 if (markpos < 0) 来判断的。  
  判断buffer中没有多余的空间，是通过 if (pos >= buffer.length) 来判断的。

理解这个思想之后，我们再对这种情况下的fill()代码进行分析，就特别容易理解了。  
(01) int sz = pos - markpos; 作用是“获取‘被标记位置’到‘buffer末尾’”的数据长度。  
(02) System.arraycopy(buffer, markpos, buffer, 0, sz); 作用是“将buffer中从markpos开始的数据”拷贝到buffer中(从位置0开始填充，填充长度是sz)。接着，将sz赋值给pos，即pos就是“被标记位置”到“buffer末尾”的数据长度。  
(03) int n = getInIfOpen().read(buffer, pos, buffer.length - pos); 从输入流中读取出“buffer.length - pos”的数据，然后填充到buffer中。  
(04) 通过第(02)和(03)步组合起来的buffer，就是包含了“原始buffer被标记位置到buffer末尾”的数据，也包含了“从输入流中新读取的数据”。

注意：**执行过情况2之后，markpos的值由“大于0”变成了“等于0”！**

 

## 情况3：读取完buffer中的数据，buffer被标记位置=0，buffer中没有多余的空间，并且buffer.length>=marklimit

执行流程如下，  
(01) read() 函数中调用 fill()  
(02) fill() 中的 else if (pos >= buffer.length) ...  
(03) fill() 中的 else if (buffer.length >= marklimit) ...

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos >= 0 && pos >= buffer.length) {
            if ( (markpos <= 0) && (buffer.length >= marklimit) ) {
                markpos = -1;   /* buffer got too big, invalidate mark */
                pos = 0;        /* drop buffer contents */
            }
        }

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

说明：这种情况的处理非常简单。首先，就是“取消标记”，即 markpos = -1；然后，设置初始化位置为0，即pos=0；最后，再从输入流中读取下一部分数据到buffer中。

 

## 情况4：读取完buffer中的数据，buffer被标记位置=0，buffer中没有多余的空间，并且buffer.length<marklimit

执行流程如下，  
(01) read() 函数中调用 fill()  
(02) fill() 中的 else if (pos >= buffer.length) ...  
(03) fill() 中的 else { int nsz = pos * 2; ... }

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos >= 0 && pos >= buffer.length) {
            if ( (markpos <= 0) && (buffer.length < marklimit) ) {
                int nsz = pos * 2;
                if (nsz > marklimit)
                    nsz = marklimit;
                byte nbuf[] = new byte[nsz];
                System.arraycopy(buffer, 0, nbuf, 0, pos);
                if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                    throw new IOException("Stream closed");
                }
                buffer = nbuf;
            }
        }

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

说明：  
这种情况的处理非常简单。  
(01) 新建一个字节数组nbuf。nbuf的大小是“pos*2”和“marklimit”中较小的那个数。

    int nsz = pos * 2;
    if (nsz > marklimit)
        nsz = marklimit;
    byte nbuf[] = new byte[nsz];

(02) 接着，将buffer中的数据拷贝到新数组nbuf中。通过System.arraycopy(buffer, 0, nbuf, 0, pos)  
(03) 最后，从输入流读取部分新数据到buffer中。通过getInIfOpen().read(buffer, pos, buffer.length - pos);  

注意：**在这里，我们思考一个问题，“为什么需要marklimit，它的存在到底有什么意义？”我们结合“情况2”、“情况3”、“情况4”的情况来分析。**

假设，marklimit是无限大的，而且我们设置了markpos。当我们从输入流中每读完一部分数据并读取下一部分数据时，都需要保存markpos所标记的数据；这就意味着，我们需要不断执行情况4中的操作，要将buffer的容量扩大……随着读取次数的增多，buffer会越来越大；这会导致我们占据的内存越来越大。所以，我们需要给出一个marklimit；当buffer>=marklimit时，就不再保存markpos的值了。

 

## 情况5：除了上面4种情况之外的情况

执行流程如下，  
(01) read() 函数中调用 fill()  
(02) fill() 中的 count = pos...

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();

        count = pos;
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

说明：这种情况的处理非常简单。直接从输入流读取部分新数据到buffer中。

 
<a name="anchor3"></a>
# 3. 示例代码

关于BufferedInputStream中API的详细用法，参考示例代码(BufferedInputStreamTest.java)：

    import java.io.BufferedInputStream;
    import java.io.ByteArrayInputStream;
    import java.io.File;
    import java.io.InputStream;
    import java.io.FileInputStream;
    import java.io.IOException;
    import java.io.FileNotFoundException;
    import java.lang.SecurityException;

    /**
     * BufferedInputStream 测试程序
     *
     * @author skywang
     */
    public class BufferedInputStreamTest {

        private static final int LEN = 5;

        public static void main(String[] args) {
            testBufferedInputStream() ;
        }

        /**
         * BufferedInputStream的API测试函数
         */
        private static void testBufferedInputStream() {

            // 创建BufferedInputStream字节流，内容是ArrayLetters数组
            try {
                File file = new File("bufferedinputstream.txt");
                InputStream in =
                      new BufferedInputStream(
                          new FileInputStream(file), 512);

                // 从字节流中读取5个字节。“abcde”，a对应0x61，b对应0x62，依次类推...
                for (int i=0; i<LEN; i++) {
                    // 若能继续读取下一个字节，则读取下一个字节
                    if (in.available() >= 0) {
                        // 读取“字节流的下一个字节”
                        int tmp = in.read();
                        System.out.printf("%d : 0x%s\n", i, Integer.toHexString(tmp));
                    }
                }

                // 若“该字节流”不支持标记功能，则直接退出
                if (!in.markSupported()) {
                    System.out.println("make not supported!");
                    return ;
                }
                  
                // 标记“当前索引位置”，即标记第6个位置的元素--“f”
                // 1024对应marklimit
                in.mark(1024);

                // 跳过22个字节。
                in.skip(22);

                // 读取5个字节
                byte[] buf = new byte[LEN];
                in.read(buf, 0, LEN);
                // 将buf转换为String字符串。
                String str1 = new String(buf);
                System.out.printf("str1=%s\n", str1);

                // 重置“输入流的索引”为mark()所标记的位置，即重置到“f”处。
                in.reset();
                // 从“重置后的字节流”中读取5个字节到buf中。即读取“fghij”
                in.read(buf, 0, LEN);
                // 将buf转换为String字符串。
                String str2 = new String(buf);
                System.out.printf("str2=%s\n", str2);

                in.close();
           } catch (FileNotFoundException e) {
               e.printStackTrace();
           } catch (SecurityException e) {
               e.printStackTrace();
           } catch (IOException e) {
               e.printStackTrace();
           }
        }
    }

程序中读取的bufferedinputstream.txt的内容如下：

    abcdefghijklmnopqrstuvwxyz
    0123456789
    ABCDEFGHIJKLMNOPQRSTUVWXYZ

运行结果：

    0 : 0x61
    1 : 0x62
    2 : 0x63
    3 : 0x64
    4 : 0x65
    str1=01234
    str2=fghij

