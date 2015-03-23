---
layout: post
title: "java io系列23之 BufferedReader详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-23 09:01
---

> **目录**  
[1. BufferedReader 介绍](#anchor1)   
[2. BufferedReader 源码分析(基于jdk1.7.40)](#anchor2)   
[3. 示例代码](#anchor3)   


<a name="anchor1"></a>
# 1. BufferedReader 介绍

BufferedReader 是缓冲字符输入流。它继承于Reader。   
BufferedReader 的作用是为其他字符输入流添加一些缓冲功能。

**BufferedReader 函数列表**

    BufferedReader(Reader in)
    BufferedReader(Reader in, int size)

    void     close()
    void     mark(int markLimit)
    boolean  markSupported()
    int      read()
    int      read(char[] buffer, int offset, int length)
    String   readLine()
    boolean  ready()
    void     reset()
    long     skip(long charCount)


<a name="anchor2"></a>
# 2. BufferedReader 源码分析(基于jdk1.7.40)

    package java.io;

    public class BufferedReader extends Reader {

        private Reader in;

        // 字符缓冲区
        private char cb[];
        // nChars 是cb缓冲区中字符的总的个数
        // nextChar 是下一个要读取的字符在cb缓冲区中的位置
        private int nChars, nextChar;

        // 表示“标记无效”。它与UNMARKED的区别是：
        // (01) UNMARKED 是压根就没有设置过标记。
        // (02) 而INVALIDATED是设置了标记，但是被标记位置太长，导致标记无效！
        private static final int INVALIDATED = -2;
        // 表示没有设置“标记”
        private static final int UNMARKED = -1;
        // “标记”
        private int markedChar = UNMARKED;
        // “标记”能标记位置的最大长度
        private int readAheadLimit = 0; /* Valid only when markedChar > 0 */

        // skipLF(即skip Line Feed)是“是否忽略换行符”标记
        private boolean skipLF = false;

        // 设置“标记”时，保存的skipLF的值
        private boolean markedSkipLF = false;

        // 默认字符缓冲区大小
        private static int defaultCharBufferSize = 8192;
        // 默认每一行的字符个数
        private static int defaultExpectedLineLength = 80;

        // 创建“Reader”对应的BufferedReader对象，sz是BufferedReader的缓冲区大小
        public BufferedReader(Reader in, int sz) {
            super(in);
            if (sz <= 0)
                throw new IllegalArgumentException("Buffer size <= 0");
            this.in = in;
            cb = new char[sz];
            nextChar = nChars = 0;
        }

        // 创建“Reader”对应的BufferedReader对象，默认的BufferedReader缓冲区大小是8k
        public BufferedReader(Reader in) {
            this(in, defaultCharBufferSize);
        }

        // 确保“BufferedReader”是打开状态
        private void ensureOpen() throws IOException {
            if (in == null)
                throw new IOException("Stream closed");
        }

        // 填充缓冲区函数。有以下两种情况被调用：
        // (01) 缓冲区没有数据时，通过fill()可以向缓冲区填充数据。
        // (02) 缓冲区数据被读完，需更新时，通过fill()可以更新缓冲区的数据。
        private void fill() throws IOException {
            // dst表示“cb中填充数据的起始位置”。
            int dst;
            if (markedChar <= UNMARKED) {
                // 没有标记的情况，则设dst=0。
                dst = 0;
            } else {
                // delta表示“当前标记的长度”，它等于“下一个被读取字符的位置”减去“标记的位置”的差值；
                int delta = nextChar - markedChar;
                if (delta >= readAheadLimit) {
                    // 若“当前标记的长度”超过了“标记上限(readAheadLimit)”，
                    // 则丢弃标记！
                    markedChar = INVALIDATED;
                    readAheadLimit = 0;
                    dst = 0;
                } else {
                    if (readAheadLimit <= cb.length) {
                        // 若“当前标记的长度”没有超过了“标记上限(readAheadLimit)”，
                        // 并且“标记上限(readAheadLimit)”小于/等于“缓冲的长度”；
                        // 则先将“下一个要被读取的位置，距离我们标记的置符的距离”间的字符保存到cb中。
                        System.arraycopy(cb, markedChar, cb, 0, delta);
                        markedChar = 0;
                        dst = delta;
                    } else {
                        // 若“当前标记的长度”没有超过了“标记上限(readAheadLimit)”，
                        // 并且“标记上限(readAheadLimit)”大于“缓冲的长度”；
                        // 则重新设置缓冲区大小，并将“下一个要被读取的位置，距离我们标记的置符的距离”间的字符保存到cb中。
                        char ncb[] = new char[readAheadLimit];
                        System.arraycopy(cb, markedChar, ncb, 0, delta);
                        cb = ncb;
                        markedChar = 0;
                        dst = delta;
                    }
                    // 更新nextChar和nChars
                    nextChar = nChars = delta;
                }
            }

            int n;
            do {
                // 从“in”中读取数据，并存储到字符数组cb中；
                // 从cb的dst位置开始存储，读取的字符个数是cb.length - dst
                // n是实际读取的字符个数；若n==0(即一个也没读到)，则继续读取！
                n = in.read(cb, dst, cb.length - dst);
            } while (n == 0);

            // 如果从“in”中读到了数据，则设置nChars(cb中字符的数目)=dst+n，
            // 并且nextChar(下一个被读取的字符的位置)=dst。
            if (n > 0) {
                nChars = dst + n;
                nextChar = dst;
            }
        }

        // 从BufferedReader中读取一个字符，该字符以int的方式返回
        public int read() throws IOException {
            synchronized (lock) {
                ensureOpen();
                for (;;) {
                    // 若“缓冲区的数据已经被读完”，
                    // 则先通过fill()更新缓冲区数据
                    if (nextChar >= nChars) {
                        fill();
                        if (nextChar >= nChars)
                            return -1;
                    }
                    // 若要“忽略换行符”，
                    // 则对下一个字符是否是换行符进行处理。
                    if (skipLF) {
                        skipLF = false;
                        if (cb[nextChar] == '\n') {
                            nextChar++;
                            continue;
                        }
                    }
                    // 返回下一个字符
                    return cb[nextChar++];
                }
            }
        }

        // 将缓冲区中的数据写入到数组cbuf中。off是数组cbuf中的写入起始位置，len是写入长度
        private int read1(char[] cbuf, int off, int len) throws IOException {
            // 若“缓冲区的数据已经被读完”，则更新缓冲区数据。
            if (nextChar >= nChars) {
                if (len >= cb.length && markedChar <= UNMARKED && !skipLF) {
                    return in.read(cbuf, off, len);
                }
                fill();
            }
            // 若更新数据之后，没有任何变化；则退出。
            if (nextChar >= nChars) return -1;
            // 若要“忽略换行符”，则进行相应处理
            if (skipLF) {
                skipLF = false;
                if (cb[nextChar] == '\n') {
                    nextChar++;
                    if (nextChar >= nChars)
                        fill();
                    if (nextChar >= nChars)
                        return -1;
                }
            }
            // 拷贝字符操作
            int n = Math.min(len, nChars - nextChar);
            System.arraycopy(cb, nextChar, cbuf, off, n);
            nextChar += n;
            return n;
        }

        // 对read1()的封装，添加了“同步处理”和“阻塞式读取”等功能
        public int read(char cbuf[], int off, int len) throws IOException {
            synchronized (lock) {
                ensureOpen();
                if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                    ((off + len) > cbuf.length) || ((off + len) < 0)) {
                    throw new IndexOutOfBoundsException();
                } else if (len == 0) {
                    return 0;
                }

                int n = read1(cbuf, off, len);
                if (n <= 0) return n;
                while ((n < len) && in.ready()) {
                    int n1 = read1(cbuf, off + n, len - n);
                    if (n1 <= 0) break;
                    n += n1;
                }
                return n;
            }
        }

        // 读取一行数据。ignoreLF是“是否忽略换行符”
        String readLine(boolean ignoreLF) throws IOException {
            StringBuffer s = null;
            int startChar;

            synchronized (lock) {
                ensureOpen();
                boolean omitLF = ignoreLF || skipLF;

                bufferLoop:
                for (;;) {

                    if (nextChar >= nChars)
                        fill();
                    if (nextChar >= nChars) { /* EOF */
                        if (s != null && s.length() > 0)
                            return s.toString();
                        else
                            return null;
                    }
                    boolean eol = false;
                    char c = 0;
                    int i;

                    /* Skip a leftover '\n', if necessary */
                    if (omitLF && (cb[nextChar] == '\n'))
                        nextChar++;
                    skipLF = false;
                    omitLF = false;

                charLoop:
                    for (i = nextChar; i < nChars; i++) {
                        c = cb[i];
                        if ((c == '\n') || (c == '\r')) {
                            eol = true;
                            break charLoop;
                        }
                    }

                    startChar = nextChar;
                    nextChar = i;

                    if (eol) {
                        String str;
                        if (s == null) {
                            str = new String(cb, startChar, i - startChar);
                        } else {
                            s.append(cb, startChar, i - startChar);
                            str = s.toString();
                        }
                        nextChar++;
                        if (c == '\r') {
                            skipLF = true;
                        }
                        return str;
                    }

                    if (s == null)
                        s = new StringBuffer(defaultExpectedLineLength);
                    s.append(cb, startChar, i - startChar);
                }
            }
        }

        // 读取一行数据。不忽略换行符
        public String readLine() throws IOException {
            return readLine(false);
        }

        // 跳过n个字符
        public long skip(long n) throws IOException {
            if (n < 0L) {
                throw new IllegalArgumentException("skip value is negative");
            }
            synchronized (lock) {
                ensureOpen();
                long r = n;
                while (r > 0) {
                    if (nextChar >= nChars)
                        fill();
                    if (nextChar >= nChars) /* EOF */
                        break;
                    if (skipLF) {
                        skipLF = false;
                        if (cb[nextChar] == '\n') {
                            nextChar++;
                        }
                    }
                    long d = nChars - nextChar;
                    if (r <= d) {
                        nextChar += r;
                        r = 0;
                        break;
                    }
                    else {
                        r -= d;
                        nextChar = nChars;
                    }
                }
                return n - r;
            }
        }

        // “下一个字符”是否可读
        public boolean ready() throws IOException {
            synchronized (lock) {
                ensureOpen();

                // 若忽略换行符为true；
                // 则判断下一个符号是否是换行符，若是的话，则忽略
                if (skipLF) {
                    if (nextChar >= nChars && in.ready()) {
                        fill();
                    }
                    if (nextChar < nChars) {
                        if (cb[nextChar] == '\n')
                            nextChar++;
                        skipLF = false;
                    }
                }
                return (nextChar < nChars) || in.ready();
            }
        }

        // 始终返回true。因为BufferedReader支持mark(), reset()
        public boolean markSupported() {
            return true;
        }

        // 标记当前BufferedReader的下一个要读取位置。关于readAheadLimit的作用，参考后面的说明。
        public void mark(int readAheadLimit) throws IOException {
            if (readAheadLimit < 0) {
                throw new IllegalArgumentException("Read-ahead limit < 0");
            }
            synchronized (lock) {
                ensureOpen();
                // 设置readAheadLimit
                this.readAheadLimit = readAheadLimit;
                // 保存下一个要读取的位置
                markedChar = nextChar;
                // 保存“是否忽略换行符”标记
                markedSkipLF = skipLF;
            }
        }

        // 重置BufferedReader的下一个要读取位置，
        // 将其还原到mark()中所保存的位置。
        public void reset() throws IOException {
            synchronized (lock) {
                ensureOpen();
                if (markedChar < 0)
                    throw new IOException((markedChar == INVALIDATED)
                                          ? "Mark invalid"
                                          : "Stream not marked");
                nextChar = markedChar;
                skipLF = markedSkipLF;
            }
        }

        public void close() throws IOException {
            synchronized (lock) {
                if (in == null)
                    return;
                in.close();
                in = null;
                cb = null;
            }
        }
    }

说明： 想读懂BufferReader的源码，就要先理解它的思想。BufferReader的作用是为其它Reader提供缓冲功能。创建BufferReader时，我们会通过它的构造函数指定某个Reader为参数。BufferReader会将该Reader中的数据分批读取，每次读取一部分到缓冲中；操作完缓冲中的这部分数据之后，再从Reader中读取下一部分的数据。

为什么需要缓冲呢？原因很简单，效率问题！缓冲中的数据实际上是保存在内存中，而原始数据可能是保存在硬盘或NandFlash中；而我们知道，从内存中读取数据的速度比从硬盘读取数据的速度至少快10倍以上。  
那干嘛不干脆一次性将全部数据都读取到缓冲中呢？第一，读取全部的数据所需要的时间可能会很长。第二，内存价格很贵，容量不想硬盘那么大。


下面，我就BufferReader中最重要的函数fill()进行说明。其它的函数很容易理解，我就不详细介绍了，大家可以参考源码中的注释进行理解。我们先看看fill()的源码：

    private void fill() throws IOException {
        int dst;
        if (markedChar <= UNMARKED) {
            /* No mark */
            dst = 0;
        } else {
            /* Marked */
            int delta = nextChar - markedChar;
            if (delta >= readAheadLimit) {
                /* Gone past read-ahead limit: Invalidate mark */
                markedChar = INVALIDATED;
                readAheadLimit = 0;
                dst = 0;
            } else {
                if (readAheadLimit <= cb.length) {
                    /* Shuffle in the current buffer */
                    System.arraycopy(cb, markedChar, cb, 0, delta);
                    markedChar = 0;
                    dst = delta;
                } else {
                    /* Reallocate buffer to accommodate read-ahead limit */
                    char ncb[] = new char[readAheadLimit];
                    System.arraycopy(cb, markedChar, ncb, 0, delta);
                    cb = ncb;
                    markedChar = 0;
                    dst = delta;
                }
                nextChar = nChars = delta;
            }
        }

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);
        } while (n == 0);
        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }

根据fill()中的if...else...，我将fill()分为4种情况进行说明。


**情况1**：读取完缓冲区的数据，并且缓冲区没有被标记

执行流程如下，  
(01) 其它函数调用 fill()，来更新缓冲区的数据  
(02) fill() 执行代码 if (markedChar <= UNMARKED) { ... }

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        int dst;
        if (markedChar <= UNMARKED) {
            /* No mark */
            dst = 0;
        } 

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);
        } while (n == 0);

        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }

说明：

这种情况发生的情况是 — — Reader中有很长的数据，我们每次从中读取一部分数据到缓冲中进行操作。每次当我们读取完缓冲中的数据之后，并且此时BufferedReader没有被标记；那么，就接着从Reader(BufferReader提供缓冲功能的Reader)中读取下一部分的数据到缓冲中。  
其中，判断是否读完缓冲区中的数据，是通过“比较nextChar和nChars之间大小”来判断的。其中，nChars 是缓冲区中字符的总的个数，而 nextChar 是缓冲区中下一个要读取的字符的位置。  
判断BufferedReader有没有被标记，是通过“markedChar”来判断的。  
理解这个思想之后，我们再对这种情况下的fill()的代码进行分析，就特别容易理解了。

(01) if (markedChar <= UNMARKED) 它的作用是判断“BufferedReader是否被标记”。若被标记，则dst=0。  
(02) in.read(cb, dst, cb.length - dst) 等价于 in.read(cb, 0, cb.length)，意思是从Reader对象in中读取cb.length个数据，并存储到缓冲区cb中，而且从缓冲区cb的位置0开始存储。该函数返回值等于n，也就是n表示实际读取的字符个数。若n=0(即没有读取到数据)，则继续读取，直到读到数据为止。  
(03) nChars=dst+n 等价于 nChars=n；意味着，更新缓冲区数据cb之后，设置nChars(缓冲区的数据个数)为n。  
(04) nextChar=dst 等价于 nextChar=0；意味着，更新缓冲区数据cb之后，设置nextChar(缓冲区中下一个会被读取的字符的索引值)为0。

 

**情况2**：读取完缓冲区的数据，缓冲区的标记位置>0，并且“当前标记的长度”超过“标记上限(readAheadLimit)”

执行流程如下，  
(01) 其它函数调用 fill()，来更新缓冲区的数据   
(02) fill() 执行代码 if (delta >= readAheadLimit) { ... }   

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        int dst;
        if (markedChar > UNMARKED) {
            int delta = nextChar - markedChar;
            if (delta >= readAheadLimit) {
                markedChar = INVALIDATED;
                readAheadLimit = 0;
                dst = 0;
            } 
        }

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);
        } while (n == 0);
        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }

说明：  
这种情况发生的情况是 — — BufferedReader中有很长的数据，我们每次从中读取一部分数据到缓冲区中进行操作。当我们读取完缓冲区中的数据之后，并且此时，BufferedReader存在标记时，同时，“当前标记的长度”大于“标记上限”；那么，就发生情况2。此时，我们会丢弃“标记”并更新缓冲区。  
(01) delta = nextChar - markedChar；其中，delta就是“当前标记的长度”，它是“下一个被读取字符的位置”减去“被标记的位置”的差值。  
(02) if (delta >= readAheadLimit)；其中，当delta >= readAheadLimit，就意味着，“当前标记的长度”>=“标记上限”。为什么要有标记上限，即readAheadLimit的值到底有何意义呢？  

我们标记一个位置之后，更新缓冲区的时候，被标记的位置会被保存；当我们不停的更新缓冲区的时候，被标记的位置会被不停的放大。然后内存的容量是有效的，我们不可能不限制长度的存储标记。所以，需要readAheadLimit来限制标记长度！  
(03) in.read(cb, dst, cb.length - dst) 等价于 in.read(cb, 0, cb.length)，意思是从Reader对象in中读取cb.length个数据，并存储到缓冲区cb中，而且从缓冲区cb的位置0开始存储。该函数返回值等于n，也就是n表示实际读取的字符个数。若n=0(即没有读取到数据)，则继续读取，直到读到数据为止。  
(04) nChars=dst+n 等价于 nChars=n；意味着，更新缓冲区数据cb之后，设置nChars(缓冲区的数据个数)为n。  
(05) nextChar=dst 等价于 nextChar=0；意味着，更新缓冲区数据cb之后，设置nextChar(缓冲区中下一个会被读取的字符的索引值)为0。

 

**情况3**：读取完缓冲区的数据，缓冲区的标记位置>0，“当前标记的长度”没超过“标记上限(readAheadLimit)”，并且“标记上限(readAheadLimit)”小于/等于“缓冲的长度”；

执行流程如下，  
(01) 其它函数调用 fill()，来更新缓冲区的数据  
(02) fill() 执行代码 if (readAheadLimit <= cb.length) { ... }

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        int dst;
        if (markedChar > UNMARKED) {
            int delta = nextChar - markedChar;
            if ((delta < readAheadLimit) &&  (readAheadLimit <= cb.length) ) {
                System.arraycopy(cb, markedChar, cb, 0, delta);
                markedChar = 0;
                dst = delta;

                nextChar = nChars = delta;
            }
        }

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);
        } while (n == 0);
        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }

说明：这种情况发生的情况是 — — BufferedReader中有很长的数据，我们每次从中读取一部分数据到缓冲区中进行操作。当我们读取完缓冲区中的数据之后，并且此时，BufferedReader存在标记时，同时，“当前标记的长度”小于“标记上限”，并且“标记上限”小于/等于“缓冲区长度”；那么，就发生情况3。此时，我们保留“被标记的位置”(即，保留被标记位置开始的数据)，并更新缓冲区(将新增的数据，追加到保留的数据之后)。

 

**情况4**：读取完缓冲区的数据，缓冲区的标记位置>0，“当前标记的长度”没超过“标记上限(readAheadLimit)”，并且“标记上限(readAheadLimit)”大于“缓冲的长度”；

执行流程如下，  
(01) 其它函数调用 fill()，来更新缓冲区的数据  
(02) fill() 执行代码 else { char ncb[] = new char[readAheadLimit]; ... }

为了方便分析，我们将这种情况下fill()执行的操作等价于以下代码：

    private void fill() throws IOException {
        int dst;
        if (markedChar > UNMARKED) {
            int delta = nextChar - markedChar;
            if ((delta < readAheadLimit) &&  (readAheadLimit > cb.length) ) {
                char ncb[] = new char[readAheadLimit];
                System.arraycopy(cb, markedChar, ncb, 0, delta);
                cb = ncb;
                markedChar = 0;
                dst = delta;
                
                nextChar = nChars = delta;
            }
        }

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);
        } while (n == 0);
        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }

说明：这种情况发生的情况是 — — BufferedReader中有很长的数据，我们每次从中读取一部分数据到缓冲区中进行操作。当我们读取完缓冲区中的数据之后，并且此时，BufferedReader存在标记时，同时，“当前标记的长度”小于“标记上限”，并且“标记上限”大于“缓冲区长度”；那么，就发生情况4。此时，我们要先更新缓冲区的大小，然后再保留“被标记的位置”(即，保留被标记位置开始的数据)，并更新缓冲区数据(将新增的数据，追加到保留的数据之后)。

 
<a name="anchor3"></a>
# 3. 示例代码

关于BufferedReader中API的详细用法，参考示例代码(BufferedReaderTest.java)： 

    import java.io.BufferedReader;
    import java.io.ByteArrayInputStream;
    import java.io.File;
    import java.io.InputStream;
    import java.io.FileReader;
    import java.io.IOException;
    import java.io.FileNotFoundException;
    import java.lang.SecurityException;

    /**
     * BufferedReader 测试程序
     *
     * @author skywang
     */
    public class BufferedReaderTest {

        private static final int LEN = 5;

        public static void main(String[] args) {
            testBufferedReader() ;
        }

        /**
         * BufferedReader的API测试函数
         */
        private static void testBufferedReader() {

            // 创建BufferedReader字符流，内容是ArrayLetters数组
            try {
                File file = new File("bufferedreader.txt");
                BufferedReader in =
                      new BufferedReader(
                          new FileReader(file));

                // 从字符流中读取5个字符。“abcde”
                for (int i=0; i<LEN; i++) {
                    // 若能继续读取下一个字符，则读取下一个字符
                    if (in.ready()) {
                        // 读取“字符流的下一个字符”
                        int tmp = in.read();
                        System.out.printf("%d : %c\n", i, tmp);
                    }
                }

                // 若“该字符流”不支持标记功能，则直接退出
                if (!in.markSupported()) {
                    System.out.println("make not supported!");
                    return ;
                }
                  
                // 标记“当前索引位置”，即标记第6个位置的元素--“f”
                // 1024对应marklimit
                in.mark(1024);

                // 跳过22个字符。
                in.skip(22);

                // 读取5个字符
                char[] buf = new char[LEN];
                in.read(buf, 0, LEN);
                System.out.printf("buf=%s\n", String.valueOf(buf));
                // 读取该行剩余的数据
                System.out.printf("readLine=%s\n", in.readLine());

                // 重置“输入流的索引”为mark()所标记的位置，即重置到“f”处。
                in.reset();
                // 从“重置后的字符流”中读取5个字符到buf中。即读取“fghij”
                in.read(buf, 0, LEN);
                System.out.printf("buf=%s\n", String.valueOf(buf));

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

程序中读取的bufferedreader.txt的内容如下：

    abcdefghijklmnopqrstuvwxyz
    0123456789
    ABCDEFGHIJKLMNOPQRSTUVWXYZ

运行结果：

    0 : a
    1 : b
    2 : c
    3 : d
    4 : e
    buf=01234
    readLine=56789
    buf=fghij

 
