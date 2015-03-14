---
layout: post
title: "java io系列25之 PrintWriter详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-25 09:01
---

> **目录**  
[1. PrintWriter 介绍](#anchor1)   
[2. PrintWriter 源码](#anchor2)   
[3. 示例代码](#anchor3)   


<a name="anchor1"></a>
# 1. PrintWriter 介绍

PrintWriter 是字符类型的打印输出流，它继承于Writer。

PrintStream 用于向文本输出流打印对象的格式化表示形式。它实现在 PrintStream 中的所有 print 方法。它不包含用于写入原始字节的方法，对于这些字节，程序应该使用未编码的字节流进行写入。

 

**PrintWriter 函数列表**

    PrintWriter(OutputStream out)
    PrintWriter(OutputStream out, boolean autoFlush)
    PrintWriter(Writer wr)
    PrintWriter(Writer wr, boolean autoFlush)
    PrintWriter(File file)
    PrintWriter(File file, String csn)
    PrintWriter(String fileName)
    PrintWriter(String fileName, String csn)

    PrintWriter     append(char c)
    PrintWriter     append(CharSequence csq, int start, int end)
    PrintWriter     append(CharSequence csq)
    boolean     checkError()
    void     close()
    void     flush()
    PrintWriter     format(Locale l, String format, Object... args)
    PrintWriter     format(String format, Object... args)
    void     print(float fnum)
    void     print(double dnum)
    void     print(String str)
    void     print(Object obj)
    void     print(char ch)
    void     print(char[] charArray)
    void     print(long lnum)
    void     print(int inum)
    void     print(boolean bool)
    PrintWriter     printf(Locale l, String format, Object... args)
    PrintWriter     printf(String format, Object... args)
    void     println()
    void     println(float f)
    void     println(int i)
    void     println(long l)
    void     println(Object obj)
    void     println(char[] chars)
    void     println(String str)
    void     println(char c)
    void     println(double d)
    void     println(boolean b)
    void     write(char[] buf, int offset, int count)
    void     write(int oneChar)
    void     write(char[] buf)
    void     write(String str, int offset, int count)
    void     write(String str)

 

<a name="anchor2"></a>
# 2. PrintWriter 源码


    package java.io;

    import java.util.Objects;
    import java.util.Formatter;
    import java.util.Locale;
    import java.nio.charset.Charset;
    import java.nio.charset.IllegalCharsetNameException;
    import java.nio.charset.UnsupportedCharsetException;

    public class PrintWriter extends Writer {

        protected Writer out;

        // 自动flush
        // 所谓“自动flush”，就是每次执行print(), println(), write()函数，都会调用flush()函数；
        // 而“不自动flush”，则需要我们手动调用flush()接口。
        private final boolean autoFlush;
        // PrintWriter是否右产生异常。当PrintWriter有异常产生时，会被本身捕获，并设置trouble为true
        private boolean trouble = false;
        // 用于格式化的对象
        private Formatter formatter;
        private PrintStream psOut = null;

        // 行分割符
        private final String lineSeparator;

        // 获取csn(字符集名字)对应的Chaset
        private static Charset toCharset(String csn)
            throws UnsupportedEncodingException
        {
            Objects.requireNonNull(csn, "charsetName");
            try {
                return Charset.forName(csn);
            } catch (IllegalCharsetNameException|UnsupportedCharsetException unused) {
                // UnsupportedEncodingException should be thrown
                throw new UnsupportedEncodingException(csn);
            }
        }

        // 将“Writer对象out”作为PrintWriter的输出流，默认不会自动flush，并且采用默认字符集。
        public PrintWriter (Writer out) {
            this(out, false);
        }

        // 将“Writer对象out”作为PrintWriter的输出流，autoFlush的flush模式，并且采用默认字符集。
        public PrintWriter(Writer out, boolean autoFlush) {
            super(out);
            this.out = out;
            this.autoFlush = autoFlush;
            lineSeparator = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction("line.separator"));
        }

        // 将“输出流对象out”作为PrintWriter的输出流，不自动flush，并且采用默认字符集。
        public PrintWriter(OutputStream out) {
            this(out, false);
        }

        // 将“输出流对象out”作为PrintWriter的输出流，autoFlush的flush模式，并且采用默认字符集。
        public PrintWriter(OutputStream out, boolean autoFlush) {
            // new OutputStreamWriter(out)：将“字节类型的输出流”转换为“字符类型的输出流”
            // new BufferedWriter(...): 为输出流提供缓冲功能。
            this(new BufferedWriter(new OutputStreamWriter(out)), autoFlush);

            // save print stream for error propagation
            if (out instanceof java.io.PrintStream) {
                psOut = (PrintStream) out;
            }
        }

        // 创建fileName对应的OutputStreamWriter，进而创建BufferedWriter对象；然后将该BufferedWriter作为PrintWriter的输出流，不自动flush，采用默认字符集。
        public PrintWriter(String fileName) throws FileNotFoundException {
            this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(fileName))),
                 false);
        }

        // 创建fileName对应的OutputStreamWriter，进而创建BufferedWriter对象；然后将该BufferedWriter作为PrintWriter的输出流，不自动flush，采用字符集charset。
        private PrintWriter(Charset charset, File file)
            throws FileNotFoundException
        {
            this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file), charset)),
                 false);
        }

        // 创建fileName对应的OutputStreamWriter，进而创建BufferedWriter对象；然后将该BufferedWriter作为PrintWriter的输出流，不自动flush，采用csn字符集。
        public PrintWriter(String fileName, String csn)
            throws FileNotFoundException, UnsupportedEncodingException
        {
            this(toCharset(csn), new File(fileName));
        }

        // 创建file对应的OutputStreamWriter，进而创建BufferedWriter对象；然后将该BufferedWriter作为PrintWriter的输出流，不自动flush，采用默认字符集。
        public PrintWriter(File file) throws FileNotFoundException {
            this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file))),
                 false);
        }

        // 创建file对应的OutputStreamWriter，进而创建BufferedWriter对象；然后将该BufferedWriter作为PrintWriter的输出流，不自动flush，采用csn字符集。
        public PrintWriter(File file, String csn)
            throws FileNotFoundException, UnsupportedEncodingException
        {
            this(toCharset(csn), file);
        }

        private void ensureOpen() throws IOException {
            if (out == null)
                throw new IOException("Stream closed");
        }

        // flush“PrintWriter输出流中的数据”。
        public void flush() {
            try {
                synchronized (lock) {
                    ensureOpen();
                    out.flush();
                }
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        public void close() {
            try {
                synchronized (lock) {
                    if (out == null)
                        return;
                    out.close();
                    out = null;
                }
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        // flush“PrintWriter输出流缓冲中的数据”，并检查错误
        public boolean checkError() {
            if (out != null) {
                flush();
            }
            if (out instanceof java.io.PrintWriter) {
                PrintWriter pw = (PrintWriter) out;
                return pw.checkError();
            } else if (psOut != null) {
                return psOut.checkError();
            }
            return trouble;
        }

        protected void setError() {
            trouble = true;
        }

        protected void clearError() {
            trouble = false;
        }

        // 将字符c写入到“PrintWriter输出流”中。c虽然是int类型，但实际只会写入一个字符
        public void write(int c) {
            try {
                synchronized (lock) {
                    ensureOpen();
                    out.write(c);
                }
            }
            catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        // 将“buf中从off开始的len个字符”写入到“PrintWriter输出流”中。
        public void write(char buf[], int off, int len) {
            try {
                synchronized (lock) {
                    ensureOpen();
                    out.write(buf, off, len);
                }
            }
            catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        // 将“buf中的全部数据”写入到“PrintWriter输出流”中。
        public void write(char buf[]) {
            write(buf, 0, buf.length);
        }

        // 将“字符串s中从off开始的len个字符”写入到“PrintWriter输出流”中。
        public void write(String s, int off, int len) {
            try {
                synchronized (lock) {
                    ensureOpen();
                    out.write(s, off, len);
                }
            }
            catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        // 将“字符串s”写入到“PrintWriter输出流”中。
        public void write(String s) {
            write(s, 0, s.length());
        }

        // 将“换行符”写入到“PrintWriter输出流”中。
        private void newLine() {
            try {
                synchronized (lock) {
                    ensureOpen();
                    out.write(lineSeparator);
                    if (autoFlush)
                        out.flush();
                }
            }
            catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            }
            catch (IOException x) {
                trouble = true;
            }
        }

        // 将“boolean数据对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(boolean b) {
            write(b ? "true" : "false");
        }

        // 将“字符c对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(char c) {
            write(c);
        }

        // 将“int数据i对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(int i) {
            write(String.valueOf(i));
        }

        // 将“long型数据l对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(long l) {
            write(String.valueOf(l));
        }

        // 将“float数据f对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(float f) {
            write(String.valueOf(f));
        }

        // 将“double数据d对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(double d) {
            write(String.valueOf(d));
        }

        // 将“字符数组s”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(char s[]) {
            write(s);
        }

        // 将“字符串数据s”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(String s) {
            if (s == null) {
                s = "null";
            }
            write(s);
        }

        // 将“对象obj对应的字符串”写入到“PrintWriter输出流”中，print实际调用的是write函数
        public void print(Object obj) {
            write(String.valueOf(obj));
        }

        // 将“换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println() {
            newLine();
        }

        // 将“boolean数据对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(boolean x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“字符x对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(char x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“int数据对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(int x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“long数据对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(long x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“float数据对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(float x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“double数据对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(double x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“字符数组x+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(char x[]) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“字符串x+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(String x) {
            synchronized (lock) {
                print(x);
                println();
            }
        }

        // 将“对象o对应的字符串+换行符”写入到“PrintWriter输出流”中，println实际调用的是write函数
        public void println(Object x) {
            String s = String.valueOf(x);
            synchronized (lock) {
                print(s);
                println();
            }
        }

        // 将“数据args”根据“默认Locale值(区域属性)”按照format格式化，并写入到“PrintWriter输出流”中
        public PrintWriter printf(String format, Object ... args) {
            return format(format, args);
        }

        // 将“数据args”根据“Locale值(区域属性)”按照format格式化，并写入到“PrintWriter输出流”中
        public PrintWriter printf(Locale l, String format, Object ... args) {
            return format(l, format, args);
        }

        // 根据“默认的Locale值(区域属性)”来格式化数据
        public PrintWriter format(String format, Object ... args) {
            try {
                synchronized (lock) {
                    ensureOpen();
                    if ((formatter == null)
                        || (formatter.locale() != Locale.getDefault()))
                        formatter = new Formatter(this);
                    formatter.format(Locale.getDefault(), format, args);
                    if (autoFlush)
                        out.flush();
                }
            } catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            } catch (IOException x) {
                trouble = true;
            }
            return this;
        }

        // 根据“Locale值(区域属性)”来格式化数据
        public PrintWriter format(Locale l, String format, Object ... args) {
            try {
                synchronized (lock) {
                    ensureOpen();
                    if ((formatter == null) || (formatter.locale() != l))
                        formatter = new Formatter(this, l);
                    formatter.format(l, format, args);
                    if (autoFlush)
                        out.flush();
                }
            } catch (InterruptedIOException x) {
                Thread.currentThread().interrupt();
            } catch (IOException x) {
                trouble = true;
            }
            return this;
        }

        // 将“字符序列的全部字符”追加到“PrintWriter输出流中”
        public PrintWriter append(CharSequence csq) {
            if (csq == null)
                write("null");
            else
                write(csq.toString());
            return this;
        }

        // 将“字符序列从start(包括)到end(不包括)的全部字符”追加到“PrintWriter输出流中”
        public PrintWriter append(CharSequence csq, int start, int end) {
            CharSequence cs = (csq == null ? "null" : csq);
            write(cs.subSequence(start, end).toString());
            return this;
        }

        // 将“字符c”追加到“PrintWriter输出流中”
        public PrintWriter append(char c) {
            write(c);
            return this;
        }
    }

 
<a name="anchor3"></a>
# 3. 示例代码

关于PrintWriter中API的详细用法，参考示例代码(PrintWriterTest.java)：

    import java.io.PrintWriter;
    import java.io.File;
    import java.io.FileOutputStream;
    import java.io.IOException;

    /**
     * PrintWriter 的示例程序
     *
     * @author skywang
     */
    public class PrintWriterTest {

        public static void main(String[] args) {

            // 下面3个函数的作用都是一样：都是将字母“abcde”写入到文件“file.txt”中。
            // 任选一个执行即可！
            testPrintWriterConstrutor1() ;
            //testPrintWriterConstrutor2() ;
            //testPrintWriterConstrutor3() ;

            // 测试write(), print(), println(), printf()等接口。
            testPrintWriterAPIS() ;
        }

        /**
         * PrintWriter(OutputStream out) 的测试函数
         *
         * 函数的作用，就是将字母“abcde”写入到文件“file.txt”中
         */
        private static void testPrintWriterConstrutor1() {
            final char[] arr={'a', 'b', 'c', 'd', 'e' };
            try {
                // 创建文件“file.txt”的File对象
                File file = new File("file.txt");
                // 创建文件对应FileOutputStream
                PrintWriter out = new PrintWriter(
                        new FileOutputStream(file));
                // 将“字节数组arr”全部写入到输出流中
                out.write(arr);
                // 关闭输出流
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        /**
         * PrintWriter(File file) 的测试函数
         *
         * 函数的作用，就是将字母“abcde”写入到文件“file.txt”中
         */
        private static void testPrintWriterConstrutor2() {
            final char[] arr={'a', 'b', 'c', 'd', 'e' };
            try {
                File file = new File("file.txt");
                PrintWriter out = new PrintWriter(file);
                out.write(arr);
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        /**
         * PrintWriter(String fileName) 的测试函数
         *
         * 函数的作用，就是将字母“abcde”写入到文件“file.txt”中
         */
        private static void testPrintWriterConstrutor3() {
            final char[] arr={'a', 'b', 'c', 'd', 'e' };
            try {
                PrintWriter out = new PrintWriter("file.txt");
                out.write(arr);
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        /**
         * 测试write(), print(), println(), printf()等接口。
         */
        private static void testPrintWriterAPIS() {
            final char[] arr={'a', 'b', 'c', 'd', 'e' };
            try {
                // 创建文件对应FileOutputStream
                PrintWriter out = new PrintWriter("other.txt");

                // 将字符串“hello PrintWriter”+回车符，写入到输出流中
                out.println("hello PrintWriter");
                // 将0x41写入到输出流中
                // 0x41对应ASCII码的字母'A'，也就是写入字符'A'
                out.write(0x41);
                // 将字符串"65"写入到输出流中。
                // out.print(0x41); 等价于 out.write(String.valueOf(0x41));
                out.print(0x41);
                // 将字符'B'追加到输出流中
                out.append('B').append("CDEF");

                // 将"CDE is 5" + 回车  写入到输出流中
                String str = "GHI";
                int num = 5;
                out.printf("%s is %d\n", str, num);

                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

运行上面的代码，会在源码所在目录生成两个文件“file.txt”和“other.txt”。

file.txt的内容如下：

    abcde

other.txt的内容如下：

    hello PrintWriter
    A65BCDEFGHI is 5

