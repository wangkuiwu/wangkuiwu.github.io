---
layout: post
title: "java io系列15之 DataOutputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-15 09:01
---


> 本章介绍DataOutputStream。我们先对DataOutputStream有个大致认识，然后再深入学习它的源码，最后通过示例加深对它的了解。

> **目录**  
[1. DataOutputStream 介绍](#anchor1)   
[2. DataOutputStream 源码分析(基于jdk1.7.40)](#anchor2)   
[3. 示例代码](#anchor3)   



<a name="anchor1"></a>
# 1. DataOutputStream 介绍

DataOutputStream 是数据输出流。它继承于FilterOutputStream。

DataOutputStream 是用来装饰其它输出流，将DataOutputStream和DataInputStream输入流配合使用，“允许应用程序以与机器无关方式从底层输入流中读写基本 Java 数据类型”。


<a name="anchor2"></a>
# 2. DataOutputStream 源码分析(基于jdk1.7.40)

    package java.io;

    public class DataOutputStream extends FilterOutputStream implements DataOutput {
        // “数据输出流”的字节数
        protected int written;

        // “数据输出流”对应的字节数组
        private byte[] bytearr = null;

        // 构造函数
        public DataOutputStream(OutputStream out) {
            super(out);
        }

        // 增加“输出值”
        private void incCount(int value) {
            int temp = written + value;
            if (temp < 0) {
                temp = Integer.MAX_VALUE;
            }
            written = temp;
        }

        // 将int类型的值写入到“数据输出流”中
        public synchronized void write(int b) throws IOException {
            out.write(b);
            incCount(1);
        }

        // 将字节数组b从off开始的len个字节，都写入到“数据输出流”中
        public synchronized void write(byte b[], int off, int len)
            throws IOException
        {
            out.write(b, off, len);
            incCount(len);
        }

        // 清空缓冲，即将缓冲中的数据都写入到输出流中
        public void flush() throws IOException {
            out.flush();
        }

        // 将boolean类型的值写入到“数据输出流”中
        public final void writeBoolean(boolean v) throws IOException {
            out.write(v ? 1 : 0);
            incCount(1);
        }

        // 将byte类型的值写入到“数据输出流”中
        public final void writeByte(int v) throws IOException {
            out.write(v);
            incCount(1);
        }

        // 将short类型的值写入到“数据输出流”中
        // 注意：short占2个字节
        public final void writeShort(int v) throws IOException {
            // 写入 short高8位 对应的字节
            out.write((v >>> 8) & 0xFF);
            // 写入 short低8位 对应的字节
            out.write((v >>> 0) & 0xFF);
            incCount(2);
        }

        // 将char类型的值写入到“数据输出流”中
        // 注意：char占2个字节
        public final void writeChar(int v) throws IOException {
            // 写入 char高8位 对应的字节
            out.write((v >>> 8) & 0xFF);
            // 写入 char低8位 对应的字节
            out.write((v >>> 0) & 0xFF);
            incCount(2);
        }

        // 将int类型的值写入到“数据输出流”中
        // 注意：int占4个字节
        public final void writeInt(int v) throws IOException {
            out.write((v >>> 24) & 0xFF);
            out.write((v >>> 16) & 0xFF);
            out.write((v >>>  8) & 0xFF);
            out.write((v >>>  0) & 0xFF);
            incCount(4);
        }

        private byte writeBuffer[] = new byte[8];

        // 将long类型的值写入到“数据输出流”中
        // 注意：long占8个字节
        public final void writeLong(long v) throws IOException {
            writeBuffer[0] = (byte)(v >>> 56);
            writeBuffer[1] = (byte)(v >>> 48);
            writeBuffer[2] = (byte)(v >>> 40);
            writeBuffer[3] = (byte)(v >>> 32);
            writeBuffer[4] = (byte)(v >>> 24);
            writeBuffer[5] = (byte)(v >>> 16);
            writeBuffer[6] = (byte)(v >>>  8);
            writeBuffer[7] = (byte)(v >>>  0);
            out.write(writeBuffer, 0, 8);
            incCount(8);
        }

        // 将float类型的值写入到“数据输出流”中
        public final void writeFloat(float v) throws IOException {
            writeInt(Float.floatToIntBits(v));
        }

        // 将double类型的值写入到“数据输出流”中
        public final void writeDouble(double v) throws IOException {
            writeLong(Double.doubleToLongBits(v));
        }

        // 将String类型的值写入到“数据输出流”中
        // 实际写入时，是将String对应的每个字符转换成byte数据后写入输出流中。
        public final void writeBytes(String s) throws IOException {
            int len = s.length();
            for (int i = 0 ; i < len ; i++) {
                out.write((byte)s.charAt(i));
            }
            incCount(len);
        }

        // 将String类型的值写入到“数据输出流”中
        // 实际写入时，是将String对应的每个字符转换成char数据后写入输出流中。
        public final void writeChars(String s) throws IOException {
            int len = s.length();
            for (int i = 0 ; i < len ; i++) {
                int v = s.charAt(i);
                out.write((v >>> 8) & 0xFF);
                out.write((v >>> 0) & 0xFF);
            }
            incCount(len * 2);
        }

        // 将UTF-8类型的值写入到“数据输出流”中
        public final void writeUTF(String str) throws IOException {
            writeUTF(str, this);
        }

        // 将String数据以UTF-8类型的形式写入到“输出流out”中
        static int writeUTF(String str, DataOutput out) throws IOException {
            //获取String的长度
            int strlen = str.length();
            int utflen = 0;
            int c, count = 0;

            // 由于UTF-8是1～4个字节不等；
            // 这里，根据UTF-8首字节的范围，判断UTF-8是几个字节的。
            for (int i = 0; i < strlen; i++) {
                c = str.charAt(i);
                if ((c >= 0x0001) && (c <= 0x007F)) {
                    utflen++;
                } else if (c > 0x07FF) {
                    utflen += 3;
                } else {
                    utflen += 2;
                }
            }

            if (utflen > 65535)
                throw new UTFDataFormatException(
                    "encoded string too long: " + utflen + " bytes");

            // 新建“字节数组bytearr”
            byte[] bytearr = null;
            if (out instanceof DataOutputStream) {
                DataOutputStream dos = (DataOutputStream)out;
                if(dos.bytearr == null || (dos.bytearr.length < (utflen+2)))
                    dos.bytearr = new byte[(utflen*2) + 2];
                bytearr = dos.bytearr;
            } else {
                bytearr = new byte[utflen+2];
            }

            // “字节数组”的前2个字节保存的是“UTF-8数据的长度”
            bytearr[count++] = (byte) ((utflen >>> 8) & 0xFF);
            bytearr[count++] = (byte) ((utflen >>> 0) & 0xFF);

            // 对UTF-8中的单字节数据进行预处理
            int i=0;
            for (i=0; i<strlen; i++) {
               c = str.charAt(i);
               if (!((c >= 0x0001) && (c <= 0x007F))) break;
               bytearr[count++] = (byte) c;
            }

            // 对预处理后的数据，接着进行处理
            for (;i < strlen; i++){
                c = str.charAt(i);
                // UTF-8数据是1个字节的情况
                if ((c >= 0x0001) && (c <= 0x007F)) {
                    bytearr[count++] = (byte) c;

                } else if (c > 0x07FF) {
                    // UTF-8数据是3个字节的情况
                    bytearr[count++] = (byte) (0xE0 | ((c >> 12) & 0x0F));
                    bytearr[count++] = (byte) (0x80 | ((c >>  6) & 0x3F));
                    bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
                } else {
                    // UTF-8数据是2个字节的情况
                    bytearr[count++] = (byte) (0xC0 | ((c >>  6) & 0x1F));
                    bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
                }
            }
            // 将字节数组写入到“数据输出流”中
            out.write(bytearr, 0, utflen+2);
            return utflen + 2;
        }

        public final int size() {
            return written;
        }
    }

<a name="anchor3"></a>
# 3. 示例代码

关于DataOutStream中API的详细用法，参考示例代码(DataInputStreamTest.java)：

    import java.io.DataInputStream;
    import java.io.DataOutputStream;
    import java.io.ByteArrayInputStream;
    import java.io.File;
    import java.io.InputStream;
    import java.io.FileInputStream;
    import java.io.FileOutputStream;
    import java.io.IOException;
    import java.io.FileNotFoundException;
    import java.lang.SecurityException;

    /**
     * DataInputStream 和 DataOutputStream测试程序
     *
     * @author skywang
     */
    public class DataInputStreamTest {

        private static final int LEN = 5;

        public static void main(String[] args) {
            // 测试DataOutputStream，将数据写入到输出流中。
            testDataOutputStream() ;
            // 测试DataInputStream，从上面的输出流结果中读取数据。
            testDataInputStream() ;
        }

        /**
         * DataOutputStream的API测试函数
         */
        private static void testDataOutputStream() {

            try {
                File file = new File("file.txt");
                DataOutputStream out =
                      new DataOutputStream(
                          new FileOutputStream(file));

                out.writeBoolean(true);
                out.writeByte((byte)0x41);
                out.writeChar((char)0x4243);
                out.writeShort((short)0x4445);
                out.writeInt(0x12345678);
                out.writeLong(0x0FEDCBA987654321L);

                out.writeUTF("abcdefghijklmnopqrstuvwxyz严12");

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
         * DataInputStream的API测试函数
         */
        private static void testDataInputStream() {

            try {
                File file = new File("file.txt");
                DataInputStream in =
                      new DataInputStream(
                          new FileInputStream(file));

                System.out.printf("byteToHexString(0x8F):0x%s\n", byteToHexString((byte)0x8F));
                System.out.printf("charToHexString(0x8FCF):0x%s\n", charToHexString((char)0x8FCF));

                System.out.printf("readBoolean():%s\n", in.readBoolean());
                System.out.printf("readByte():0x%s\n", byteToHexString(in.readByte()));
                System.out.printf("readChar():0x%s\n", charToHexString(in.readChar()));
                System.out.printf("readShort():0x%s\n", shortToHexString(in.readShort()));
                System.out.printf("readInt():0x%s\n", Integer.toHexString(in.readInt()));
                System.out.printf("readLong():0x%s\n", Long.toHexString(in.readLong()));
                System.out.printf("readUTF():%s\n", in.readUTF());

                in.close();
           } catch (FileNotFoundException e) {
               e.printStackTrace();
           } catch (SecurityException e) {
               e.printStackTrace();
           } catch (IOException e) {
               e.printStackTrace();
           }
        }

        // 打印byte对应的16进制的字符串
        private static String byteToHexString(byte val) {
            return Integer.toHexString(val & 0xff);
        }

        // 打印char对应的16进制的字符串
        private static String charToHexString(char val) {
            return Integer.toHexString(val);
        }

        // 打印short对应的16进制的字符串
        private static String shortToHexString(short val) {
            return Integer.toHexString(val & 0xffff);
        }
    }

运行结果：

    byteToHexString(0x8F):0x8f
    charToHexString(0x8FCF):0x8fcf
    readBoolean():true
    readByte():0x41
    readChar():0x4243
    readShort():0x4445
    readInt():0x12345678
    readLong():0xfedcba987654321
    readUTF():abcdefghijklmnopqrstuvwxyz严12

