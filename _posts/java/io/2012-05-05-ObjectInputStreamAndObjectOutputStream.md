---
layout: post
title: "java io系列05之 ObjectInputStream和ObjectOutputStream详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-05 09:01
---

> 本章，我们学习ObjectInputStream 和 ObjectOutputStream

> **目录**  
[1. ObjectInputStream 和 ObjectOutputStream 介绍](#anchor1)   
[2. 演示程序](#anchor2)   


<a name="anchor1"></a>
# 1. ObjectInputStream 和 ObjectOutputStream 介绍

ObjectInputStream 和 ObjectOutputStream 的作用是，对基本数据和对象进行序列化操作支持。

创建“文件输出流”对应的ObjectOutputStream对象，该ObjectOutputStream对象能提供对“基本数据或对象”的持久存储；当我们需要读取这些存储的“基本数据或对象”时，可以创建“文件输入流”对应的ObjectInputStream，进而读取出这些“基本数据或对象”。

注意： 只有支持 java.io.Serializable 或 java.io.Externalizable 接口的对象才能被ObjectInputStream/ObjectOutputStream所操作！



ObjectOutputStream 函数列表

    // 构造函数
    ObjectOutputStream(OutputStream output)
    // public函数
    void     close()
    void     defaultWriteObject()
    void     flush()
    ObjectOutputStream.PutField     putFields()
    void     reset()
    void     useProtocolVersion(int version)
    void     write(int value)
    void     write(byte[] buffer, int offset, int length)
    void     writeBoolean(boolean value)
    void     writeByte(int value)
    void     writeBytes(String value)
    void     writeChar(int value)
    void     writeChars(String value)
    void     writeDouble(double value)
    void     writeFields()
    void     writeFloat(float value)
    void     writeInt(int value)
    void     writeLong(long value)
    final void     writeObject(Object object)
    void     writeShort(int value)
    void     writeUTF(String value)
    void     writeUnshared(Object object)

ObjectInputStream 函数列表

    // 构造函数
    ObjectInputStream(InputStream input)

    int     available()
    void     close()
    void     defaultReadObject()
    int     read(byte[] buffer, int offset, int length)
    int     read()
    boolean     readBoolean()
    byte     readByte()
    char     readChar()
    double     readDouble()
    ObjectInputStream.GetField     readFields()
    float     readFloat()
    void     readFully(byte[] dst)
    void     readFully(byte[] dst, int offset, int byteCount)
    int     readInt()
    String     readLine()
    long     readLong()
    final Object     readObject()
    short     readShort()
    String     readUTF()
    Object     readUnshared()
    int     readUnsignedByte()
    int     readUnsignedShort()
    synchronized void     registerValidation(ObjectInputValidation object, int priority)
    int     skipBytes(int length)

<a name="anchor2"></a>
# 2. 演示程序

源码如下(ObjectStreamTest.java)： 

    /**
     * ObjectInputStream 和 ObjectOutputStream 测试程序
     *
     * 注意：通过ObjectInputStream, ObjectOutputStream操作的对象，必须是实现了Serializable或Externalizable序列化接口的类的实例。
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
    import java.util.Map;
    import java.util.HashMap;
    import java.util.Iterator;
      
    public class ObjectStreamTest { 
        private static final String TMP_FILE = "box.tmp";
      
        public static void main(String[] args) {   
            testWrite();
            testRead();
        }
      

        /**
         * ObjectOutputStream 测试函数
         */
        private static void testWrite() {   
            try {
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                out.writeBoolean(true);
                out.writeByte((byte)65);
                out.writeChar('a');
                out.writeInt(20131015);
                out.writeFloat(3.14F);
                out.writeDouble(1.414D);
                // 写入HashMap对象
                HashMap map = new HashMap();
                map.put("one", "red");
                map.put("two", "green");
                map.put("three", "blue");
                out.writeObject(map);
                // 写入自定义的Box对象，Box实现了Serializable接口
                Box box = new Box("desk", 80, 48);
                out.writeObject(box);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * ObjectInputStream 测试函数
         */
        private static void testRead() {
            try {
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                System.out.printf("boolean:%b\n" , in.readBoolean());
                System.out.printf("byte:%d\n" , (in.readByte()&0xff));
                System.out.printf("char:%c\n" , in.readChar());
                System.out.printf("int:%d\n" , in.readInt());
                System.out.printf("float:%f\n" , in.readFloat());
                System.out.printf("double:%f\n" , in.readDouble());
                // 读取HashMap对象
                HashMap map = (HashMap) in.readObject();
                Iterator iter = map.entrySet().iterator();
                while (iter.hasNext()) {
                    Map.Entry entry = (Map.Entry)iter.next();
                    System.out.printf("%-6s -- %s\n" , entry.getKey(), entry.getValue());
                }
                // 读取Box对象，Box实现了Serializable接口
                Box box = (Box) in.readObject();
                System.out.println("box: " + box);

                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    class Box implements Serializable {
        private int width;   
        private int height; 
        private String name;   

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

运行结果：

    boolean:true
    byte:65
    char:a
    int:20131015
    float:3.140000
    double:1.414000
    two    -- green
    one    -- red
    three  -- blue
    box: [desk: (80, 48) ]

 
