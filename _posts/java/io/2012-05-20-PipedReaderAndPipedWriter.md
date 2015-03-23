---
layout: post
title: "java io系列20之 PipedReader和PipedWriter详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-20 09:01
---

> 本章，我们学习PipedReader和PipedWriter。它们和“PipedInputStream和PipedOutputStream”一样，都可以用于管道通信。

> **目录**  
> [1. PipedWriter和PipedReader介绍](#anchor1)   
> [2. PipedWriter和PipedReader源码分析](#anchor2)   
> [3. 示例](#anchor3)   


<a name="anchor1"></a>
# 1. PipedWriter和PipedReader介绍

PipedWriter 是字符管道输出流，它继承于Writer。  
PipedReader 是字符管道输入流，它继承于Writer。  
PipedWriter和PipedReader的作用是可以通过管道进行线程间的通讯。在使用管道通信时，必须将PipedWriter和PipedReader配套使用。



<a name="anchor2"></a>
# 2. PipedWriter和PipedReader源码分析

## 2.1 PipedWriter 源码(基于jdk1.7.40)

    package java.io;

    public class PipedWriter extends Writer {

        // 与PipedWriter通信的PipedReader对象
        private PipedReader sink;

        // PipedWriter的关闭标记
        private boolean closed = false;

        // 构造函数，指定配对的PipedReader
        public PipedWriter(PipedReader snk)  throws IOException {
            connect(snk);
        }

        // 构造函数
        public PipedWriter() {
        }

        // 将“PipedWriter” 和 “PipedReader”连接。
        public synchronized void connect(PipedReader snk) throws IOException {
            if (snk == null) {
                throw new NullPointerException();
            } else if (sink != null || snk.connected) {
                throw new IOException("Already connected");
            } else if (snk.closedByReader || closed) {
                throw new IOException("Pipe closed");
            }

            sink = snk;
            snk.in = -1;
            snk.out = 0;
            // 设置“PipedReader”和“PipedWriter”为已连接状态
            // connected是PipedReader中定义的，用于表示“PipedReader和PipedWriter”是否已经连接
            snk.connected = true;
        }

        // 将一个字符c写入“PipedWriter”中。
        // 将c写入“PipedWriter”之后，它会将c传输给“PipedReader”
        public void write(int c)  throws IOException {
            if (sink == null) {
                throw new IOException("Pipe not connected");
            }
            sink.receive(c);
        }

        // 将字符数组b写入“PipedWriter”中。
        // 将数组b写入“PipedWriter”之后，它会将其传输给“PipedReader”
        public void write(char cbuf[], int off, int len) throws IOException {
            if (sink == null) {
                throw new IOException("Pipe not connected");
            } else if ((off | len | (off + len) | (cbuf.length - (off + len))) < 0) {
                throw new IndexOutOfBoundsException();
            }
            sink.receive(cbuf, off, len);
        }

        // 清空“PipedWriter”。
        // 这里会调用“PipedReader”的notifyAll()；
        // 目的是让“PipedReader”放弃对当前资源的占有，让其它的等待线程(等待读取PipedWriter的线程)读取“PipedWriter”的值。
        public synchronized void flush() throws IOException {
            if (sink != null) {
                if (sink.closedByReader || closed) {
                    throw new IOException("Pipe closed");
                }
                synchronized (sink) {
                    sink.notifyAll();
                }
            }
        }

        // 关闭“PipedWriter”。
        // 关闭之后，会调用receivedLast()通知“PipedReader”它已经关闭。
        public void close()  throws IOException {
            closed = true;
            if (sink != null) {
                sink.receivedLast();
            }
        }
    }

## 2.2 PipedReader 源码(基于jdk1.7.40)

    package java.io;

    public class PipedReader extends Reader {
        // “PipedWriter”是否关闭的标记
        boolean closedByWriter = false;
        // “PipedReader”是否关闭的标记
        boolean closedByReader = false;
        // “PipedReader”与“PipedWriter”是否连接的标记
        // 它在PipedWriter的connect()连接函数中被设置为true
        boolean connected = false;

        Thread readSide;    // 读取“管道”数据的线程
        Thread writeSide;    // 向“管道”写入数据的线程

        // “管道”的默认大小
        private static final int DEFAULT_PIPE_SIZE = 1024;

        // 缓冲区
        char buffer[];

        //下一个写入字符的位置。in==out代表满，说明“写入的数据”全部被读取了。
        int in = -1;
        //下一个读取字符的位置。in==out代表满，说明“写入的数据”全部被读取了。
        int out = 0;

        // 构造函数：指定与“PipedReader”关联的“PipedWriter”
        public PipedReader(PipedWriter src) throws IOException {
            this(src, DEFAULT_PIPE_SIZE);
        }

        // 构造函数：指定与“PipedReader”关联的“PipedWriter”，以及“缓冲区大小”
        public PipedReader(PipedWriter src, int pipeSize) throws IOException {
            initPipe(pipeSize);
            connect(src);
        }

        // 构造函数：默认缓冲区大小是1024字符
        public PipedReader() {
            initPipe(DEFAULT_PIPE_SIZE);
        }

        // 构造函数：指定缓冲区大小是pipeSize
        public PipedReader(int pipeSize) {
            initPipe(pipeSize);
        }

        // 初始化“管道”：新建缓冲区大小
        private void initPipe(int pipeSize) {
            if (pipeSize <= 0) {
                throw new IllegalArgumentException("Pipe size <= 0");
            }
            buffer = new char[pipeSize];
        }

        // 将“PipedReader”和“PipedWriter”绑定。
        // 实际上，这里调用的是PipedWriter的connect()函数
        public void connect(PipedWriter src) throws IOException {
            src.connect(this);
        }

        // 接收int类型的数据b。
        // 它只会在PipedWriter的write(int b)中会被调用
        synchronized void receive(int c) throws IOException {
            // 检查管道状态
            if (!connected) {
                throw new IOException("Pipe not connected");
            } else if (closedByWriter || closedByReader) {
                throw new IOException("Pipe closed");
            } else if (readSide != null && !readSide.isAlive()) {
                throw new IOException("Read end dead");
            }

            // 获取“写入管道”的线程
            writeSide = Thread.currentThread();
            // 如果“管道中被读取的数据，等于写入管道的数据”时，
            // 则每隔1000ms检查“管道状态”，并唤醒管道操作：若有“读取管道数据线程被阻塞”，则唤醒该线程。
            while (in == out) {
                if ((readSide != null) && !readSide.isAlive()) {
                    throw new IOException("Pipe broken");
                }
                /* full: kick any waiting readers */
                notifyAll();
                try {
                    wait(1000);
                } catch (InterruptedException ex) {
                    throw new java.io.InterruptedIOException();
                }
            }
            if (in < 0) {
                in = 0;
                out = 0;
            }
            buffer[in++] = (char) c;
            if (in >= buffer.length) {
                in = 0;
            }
        }

        // 接收字符数组b。
        synchronized void receive(char c[], int off, int len)  throws IOException {
            while (--len >= 0) {
                receive(c[off++]);
            }
        }

        // 当PipedWriter被关闭时，被调用
        synchronized void receivedLast() {
            closedByWriter = true;
            notifyAll();
        }

        // 从管道(的缓冲)中读取一个字符，并将其转换成int类型
        public synchronized int read()  throws IOException {
            if (!connected) {
                throw new IOException("Pipe not connected");
            } else if (closedByReader) {
                throw new IOException("Pipe closed");
            } else if (writeSide != null && !writeSide.isAlive()
                       && !closedByWriter && (in < 0)) {
                throw new IOException("Write end dead");
            }

            readSide = Thread.currentThread();
            int trials = 2;
            while (in < 0) {
                if (closedByWriter) {
                    /* closed by writer, return EOF */
                    return -1;
                }
                if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
                    throw new IOException("Pipe broken");
                }
                /* might be a writer waiting */
                notifyAll();
                try {
                    wait(1000);
                } catch (InterruptedException ex) {
                    throw new java.io.InterruptedIOException();
                }
            }
            int ret = buffer[out++];
            if (out >= buffer.length) {
                out = 0;
            }
            if (in == out) {
                /* now empty */
                in = -1;
            }
            return ret;
        }

        // 从管道(的缓冲)中读取数据，并将其存入到数组b中
        public synchronized int read(char cbuf[], int off, int len)  throws IOException {
            if (!connected) {
                throw new IOException("Pipe not connected");
            } else if (closedByReader) {
                throw new IOException("Pipe closed");
            } else if (writeSide != null && !writeSide.isAlive()
                       && !closedByWriter && (in < 0)) {
                throw new IOException("Write end dead");
            }

            if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }

            /* possibly wait on the first character */
            int c = read();
            if (c < 0) {
                return -1;
            }
            cbuf[off] =  (char)c;
            int rlen = 1;
            while ((in >= 0) && (--len > 0)) {
                cbuf[off + rlen] = buffer[out++];
                rlen++;
                if (out >= buffer.length) {
                    out = 0;
                }
                if (in == out) {
                    /* now empty */
                    in = -1;
                }
            }
            return rlen;
        }

        // 是否能从管道中读取下一个数据
        public synchronized boolean ready() throws IOException {
            if (!connected) {
                throw new IOException("Pipe not connected");
            } else if (closedByReader) {
                throw new IOException("Pipe closed");
            } else if (writeSide != null && !writeSide.isAlive()
                       && !closedByWriter && (in < 0)) {
                throw new IOException("Write end dead");
            }
            if (in < 0) {
                return false;
            } else {
                return true;
            }
        }

        // 关闭PipedReader
        public void close()  throws IOException {
            in = -1;
            closedByReader = true;
        }
    }

 
<a name="anchor3"></a>
# 3. 示例

下面，我们看看多线程中通过PipedWriter和PipedReader通信的例子。例子中包括3个类：Receiver.java, Sender.java 和 PipeTest.java

    Receiver.java的代码如下：

    import java.io.IOException;   
       
    import java.io.PipedReader;   
       
    @SuppressWarnings("all")   
    /**  
     * 接收者线程  
     */   
    public class Receiver extends Thread {   
           
        // 管道输入流对象。
        // 它和“管道输出流(PipedWriter)”对象绑定，
        // 从而可以接收“管道输出流”的数据，再让用户读取。
        private PipedReader in = new PipedReader();   
       
        // 获得“管道输入流对象”
        public PipedReader getReader(){   
            return in;   
        }   
           
        @Override
        public void run(){   
            readMessageOnce() ;
            //readMessageContinued() ;
        }

        // 从“管道输入流”中读取1次数据
        public void readMessageOnce(){   
            // 虽然buf的大小是2048个字符，但最多只会从“管道输入流”中读取1024个字符。
            // 因为，“管道输入流”的缓冲区大小默认只有1024个字符。
            char[] buf = new char[2048];   
            try {   
                int len = in.read(buf);   
                System.out.println(new String(buf,0,len));   
                in.close();   
            } catch (IOException e) {   
                e.printStackTrace();   
            }   
        }

        // 从“管道输入流”读取>1024个字符时，就停止读取
        public void readMessageContinued(){
            int total=0;
            while(true) {
                char[] buf = new char[1024];
                try {
                    int len = in.read(buf);
                    total += len;
                    System.out.println(new String(buf,0,len));
                    // 若读取的字符总数>1024，则退出循环。
                    if (total > 1024)
                        break;
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            try {
                in.close(); 
            } catch (IOException e) {   
                e.printStackTrace();   
            }   
        }   
    }

Sender.java的代码如下：

    import java.io.IOException;   
       
    import java.io.PipedWriter;   
    @SuppressWarnings("all")
    /**  
     * 发送者线程  
     */   
    public class Sender extends Thread {   
           
        // 管道输出流对象。
        // 它和“管道输入流(PipedReader)”对象绑定，
        // 从而可以将数据发送给“管道输入流”的数据，然后用户可以从“管道输入流”读取数据。
        private PipedWriter out = new PipedWriter();

        // 获得“管道输出流”对象
        public PipedWriter getWriter(){
            return out;
        }   

        @Override
        public void run(){   
            writeShortMessage();
            //writeLongMessage();
        }   

        // 向“管道输出流”中写入一则较简短的消息："this is a short message" 
        private void writeShortMessage() {
            String strInfo = "this is a short message" ;
            try {
                out.write(strInfo.toCharArray());
                out.close();   
            } catch (IOException e) {   
                e.printStackTrace();   
            }   
        }
        // 向“管道输出流”中写入一则较长的消息
        private void writeLongMessage() {
            StringBuilder sb = new StringBuilder();
            // 通过for循环写入1020个字符
            for (int i=0; i<102; i++)
                sb.append("0123456789");
            // 再写入26个字符。
            sb.append("abcdefghijklmnopqrstuvwxyz");
            // str的总长度是1020+26=1046个字符
            String str = sb.toString();
            try {
                // 将1046个字符写入到“管道输出流”中
                out.write(str);
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

PipeTest.java的代码如下：

    import java.io.PipedReader;
    import java.io.PipedWriter;
    import java.io.IOException;

    @SuppressWarnings("all")   
    /**  
     * 管道输入流和管道输出流的交互程序
     */   
    public class PipeTest {   
       
        public static void main(String[] args) {   
            Sender t1 = new Sender();   
               
            Receiver t2 = new Receiver();   
               
            PipedWriter out = t1.getWriter();   
     
            PipedReader in = t2.getReader();   

            try {   
                //管道连接。下面2句话的本质是一样。
                //out.connect(in);   
                in.connect(out);   
                   
                /**  
                 * Thread类的START方法：  
                 * 使该线程开始执行；Java 虚拟机调用该线程的 run 方法。   
                 * 结果是两个线程并发地运行；当前线程（从调用返回给 start 方法）和另一个线程（执行其 run 方法）。   
                 * 多次启动一个线程是非法的。特别是当线程已经结束执行后，不能再重新启动。   
                 */
                t1.start();
                t2.start();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

 

运行结果：

    this is a short message

结果说明：
(01) in.connect(out);  
它的作用是将“管道输入流”和“管道输出流”关联起来。查看PipedWriter.java和PipedReader.java中connect()的源码；我们知道 out.connect(in); 等价于 in.connect(out);

(02)

    t1.start(); // 启动“Sender”线程
    t2.start(); // 启动“Receiver”线程

先查看Sender.java的源码，线程启动后执行run()函数；在Sender.java的run()中，调用writeShortMessage();  
writeShortMessage();的作用就是向“管道输出流”中写入数据"this is a short message" ；这条数据会被“管道输入流”接收到。下面看看这是如何实现的。  
先看write(char char的源码。PipedWriter.java继承于Writer.java；Writer.java中write(char c[])的源码如下：

    public void write(char cbuf[]) throws IOException {
        write(cbuf, 0, cbuf.length);
    }

实际上write(char c[])是调用的PipedWriter.java中的write(char c[], int off, int len)函数。查看write(char c[], int off, int len)的源码，我们发现：它会调用 sink.receive(cbuf, off, len); 进一步查看receive(char c[], int off, int len)的定义，我们知道sink.receive(cbuf, off, len)的作用就是：将“管道输出流”中的数据保存到“管道输入流”的缓冲中。而“管道输入流”的缓冲区buffer的默认大小是1024个字符。

至此，我们知道：t1.start()启动Sender线程，而Sender线程会将数据"this is a short message"写入到“管道输出流”；而“管道输出流”又会将该数据传输给“管道输入流”，即而保存在“管道输入流”的缓冲中。


接下来，我们看看“用户如何从‘管道输入流’的缓冲中读取数据”。这实际上就是Receiver线程的动作。  
t2.start() 会启动Receiver线程，从而执行Receiver.java的run()函数。查看Receiver.java的源码，我们知道run()调用了readMessageOnce()。  
而readMessageOnce()就是调用in.read(buf)从“管道输入流in”中读取数据，并保存到buf中。  
通过上面的分析，我们已经知道“管道输入流in”的缓冲中的数据是"this is a short message"；因此，buf的数据就是"this is a short message"。


为了加深对管道的理解。我们接着进行下面两个小试验。

**试验一：修改Sender.java**

将

    public void run(){   
        writeShortMessage();
        //writeLongMessage();
    }

修改为

    public void run(){   
        //writeShortMessage();
        writeLongMessage();
    }

运行程序。运行结果如下：

![img](/media/pic/java/io/PipedReaderAndPipedWriter01.jpg)

从中，我们看出，程序运行出错！抛出异常 java.io.IOException: Pipe closed

为什么会这样呢？

我分析一下程序流程。  
(01) 在PipeTest中，通过in.connect(out)将输入和输出管道连接起来；然后，启动两个线程。t1.start()启动了线程Sender，t2.start()启动了线程Receiver。  
(02) Sender线程启动后，通过writeLongMessage()写入数据到“输出管道”，out.write(str.toCharArray())共写入了1046个字符。而根据PipedWriter的源码，PipedWriter的write()函数会调用PipedReader的receive()函数。而观察PipedReader的receive()函数，我们知道，PipedReader会将接受的数据存储缓冲区。仔细观察receive()函数，有如下代码：

    while (in == out) {
        if ((readSide != null) && !readSide.isAlive()) {
            throw new IOException("Pipe broken");
        }
        /* full: kick any waiting readers */
        notifyAll();
        try {
            wait(1000);
        } catch (InterruptedException ex) {
            throw new java.io.InterruptedIOException();
        }
    }

而in和out的初始值分别是in=-1, out=0；结合上面的while(in==out)。我们知道，它的含义就是，每往管道中写入一个字符，就达到了in==out这个条件。然后，就调用notifyAll()，唤醒“读取管道的线程”。

也就是，每往管道中写入一个字符，都会阻塞式的等待其它线程读取。  
然而，PipedReader的缓冲区的默认大小是1024！但是，此时要写入的数据却有1046！所以，一次性最多只能写入1024个字符。

(03) Receiver线程启动后，会调用readMessageOnce()读取管道输入流。读取1024个字符会，会调用close()关闭，管道。

由(02)和(03)的分析可知，Sender要往管道写入1046个字符。其中，前1024个字符(缓冲区容量是1024)能正常写入，并且每写入一个就读取一个。当写入1025个字符时，依然是依次的调用PipedWriter.java中的write()；然后，write()中调用PipedReader.java中的receive()；在PipedReader.java中，最终又会调用到receive(int c)函数。 而此时，管道输入流已经被关闭，也就是closedByReader为true，所以抛出throw new IOException("Pipe closed")。

我们对“试验一”继续进行修改，解决该问题。


**试验二： 在“试验一”的基础上继续修改Receiver.java**
将

    public void run(){   
        readMessageOnce() ;
        //readMessageContinued() ;
    }

修改为

    public void run(){   
        //readMessageOnce() ;
        readMessageContinued() ;
    }

此时，程序能正常运行。运行结果为：

    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    01234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
    012345678901234567890123456789abcd
    efghijklmnopqrstuvwxyz

 
