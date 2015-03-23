---
layout: post
title: "java io系列06之 序列化(Serializable和Externalizable)详解"
description: "java io"
category: java
tags: [java]
date: 2012-05-06 09:01
---
 

> 本章，我们对序列化进行深入的学习和探讨。学习内容，包括序列化的作用、用途、用法，以及对实现序列化的2种方式Serializable和Externalizable的深入研究。

> **目录**  
[1. 序列化是的作用和用途](#anchor1)   
[2. 演示程序1](#anchor2)   
[3. 演示程序2](#anchor3)   
[4. 演示程序3](#anchor4)   
[5. 演示程序4](#anchor5)   
[6. 演示程序5](#anchor6)   
[7. Externalizable和完全定制序列化过程](#anchor7)   



<a name="anchor1"></a>
# 1. 序列化是的作用和用途

序列化，就是为了保存对象的状态；而与之对应的反序列化，则可以把保存的对象状态再读出来。  
简言之：序列化/反序列化，是Java提供一种专门用于的保存/恢复对象状态的机制。

一般在以下几种情况下，我们可能会用到序列化：  
a）当你想把的内存中的对象状态保存到一个文件中或者数据库中时候；  
b）当你想用套接字在网络上传送对象的时候；  
c）当你想通过RMI传输对象的时候。

<a name="anchor2"></a>
# 2. 演示程序1

下面，我们先通过一则简单示例来查看序列化的用法。

源码如下(SerialTest1.java)： 

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
      
    public class SerialTest1 {
        private static final String TMP_FILE = ".serialtest1.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
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

    testWrite box: [desk: (80, 48) ]
    testRead box: [desk: (80, 48) ]

源码说明：

(01) 程序的作用很简单，就是演示：先将Box对象，通过对象输出流保存到文件中；之后，再通过对象输入流，将文件中保存的Box对象读取出来。

(02) Box类说明。Box是我们自定义的演示类，它被用于序列化的读写。Box实现了Serialable接口，因此它支持序列化操作；即，Box支持通过ObjectOutputStream去写入到输出流中，并且支持通过ObjectInputStream从输入流中读取出来。

(03) testWrite()函数说明。testWrite()的作用就是，新建一个Box对象，然后将该Box对象写入到文件中。  
&nbsp;&nbsp;&nbsp;&nbsp; 首先，新建文件TMP_FILE的文件输出流对象(即FileOutputStream对象)，再创建该文件输出流的对象输出流(即ObjectOutputStream对象)。  
&nbsp;&nbsp;&nbsp;&nbsp; a) 关于FileInputStream和FileOutputStream的内容，可以参考“java io系列07之 FileInputStream和FileOutputStream”。  
&nbsp;&nbsp;&nbsp;&nbsp; b) 关于ObjectInputStream和ObjectOutputStream的的更多知识，可以参考“java io系列05之 ObjectInputStream 和 ObjectOutputStream”  
&nbsp;&nbsp;&nbsp;&nbsp; 然后，新建Box对象。  
&nbsp;&nbsp;&nbsp;&nbsp; 最后，通过out.writeObject(box) 将box写入到对象输出流中。实际上，相当于将box写入到文件TMP_FILE中。  

(04) testRead()函数说明。testRead()的作用就是，从文件中读出Box对象。  
&nbsp;&nbsp;&nbsp;&nbsp; 首先，新建文件TMP_FILE的文件输入流对象(即FileInputStream对象)，再创建该文件输入流的对象输入流(即ObjectInputStream对象)。  
&nbsp;&nbsp;&nbsp;&nbsp; 然后，通过in.readObject() 从对象输入流中读取出Box对象。实际上，相当于从文件TMP_FILE中读取Box对象。

通过上面的示例，我们知道：我们可以自定义类，让它支持序列化(即实现Serializable接口)，从而能支持对象的保存/恢复。  
若要支持序列化，除了“自定义实现Serializable接口的类”之外；java的“基本类型”和“java自带的实现了Serializable接口的类”，都支持序列化。我们通过下面的示例去查看一下。


<a name="anchor3"></a>
# 3. 演示程序2

源码如下(SerialTest2.java)：

    /**
     * “基本类型” 和 “java自带的实现Serializable接口的类” 对序列化的支持
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
      
    public class SerialTest2 {
        private static final String TMP_FILE = ".serialabletest2.txt";
      
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
                out.writeBoolean(true);    // 写入Boolean值
                out.writeByte((byte)65);// 写入Byte值
                out.writeChar('a');     // 写入Char值
                out.writeInt(20131015); // 写入Int值
                out.writeFloat(3.14F);  // 写入Float值
                out.writeDouble(1.414D);// 写入Double值
                // 写入HashMap对象
                HashMap map = new HashMap();
                map.put("one", "red");
                map.put("two", "green");
                map.put("three", "blue");
                out.writeObject(map);

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

                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
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

源码说明：

(01) 程序的作用很简单，就是演示：先将“基本类型数据”和“HashMap对象”，通过对象输出流保存到文件中；之后，再通过对象输入流，将这些保存的数据读取出来。

(02) testWrite()函数说明。testWrite()的作用就是，先将“基本类型数据”和“HashMap对象”，通过对象输出流保存到文件中。  
&nbsp;&nbsp;&nbsp;&nbsp; 首先，新建文件TMP_FILE的文件输出流对象(即FileOutputStream对象)，再创建该文件输出流的对象输出流(即ObjectOutputStream对象)。  
&nbsp;&nbsp;&nbsp;&nbsp; 然后，通过 writeBoolean(), writeByte(), ... , writeDouble() 等一系列函数将“Boolean, byte, char, ... , double等基本数据类型”写入到对象输出流中。实际上，相当于将这些内容写入到文件TMP_FILE中。  
&nbsp;&nbsp;&nbsp;&nbsp; 最后，新建HashMap对象map，并通过out.writeObject(map) 将map写入到对象输出流中。实际上，相当于map写入到文件TMP_FILE中。  
&nbsp;&nbsp;&nbsp;&nbsp; 关于HashMap的更多知识，可以参考“[Java 集合系列10之 HashMap详细介绍(源码解析)和使用示例][link_java_collection_10]”。

(03) testRead()函数说明。testRead()的作用就是，从文件中读出testWrite()写入的对象。  
&nbsp;&nbsp;&nbsp;&nbsp; 首先，新建文件TMP_FILE的文件输入流对象(即FileInputStream对象)，再创建该文件输入流的对象输入流(即ObjectInputStream对象)。  
&nbsp;&nbsp;&nbsp;&nbsp; 然后，通过in.readObject() 从对象输入流中读取出testWrite()对象。实际上，相当于从文件TMP_FILE中读取出这些对象。

在前面，我们提到过：若要支持序列化，除了“自定义实现Serializable接口的类”之外；java的“基本类型”和“java自带的实现了Serializable接口的类”，都支持序列化。为了验证这句话，我们看看HashMap是否实现了Serializable接口。  

HashMap是java.util包中定义的类，它的接口声明如下：

    public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {} 

至此，我们对序列化的认识已经比较深入了：即知道了“序列化的作用和用法”，也知道了“基本类型”、“java自带的支持Serializable接口的类”和“自定义实现Serializable接口的类”都能支持序列化。  
应付序列化的简单使用应该足够了。但是，我们的目的是对序列化有更深层次的了解！更何况，写此文的作者(也就是区区在下)，应该比各位看官要累(既要写代码，又要总结，还得注意排版和用词，讲的通俗易懂，让各位看得轻松自在)；我这个菜鸟都能做到这些，何况对知识极其渴望的您呢？所以，请深吸一口气，然后继续……

我们在介绍序列化定义时，说过“序列化/反序列化，是专门用于的保存/恢复对象状态的机制”。  
从中，我们知道：序列化/反序列化，只支持保存/恢复对象状态，即仅支持保存/恢复类的成员变量，但不支持保存类的成员方法！  
但是，序列化是不是对类的所有的成员变量的状态都能保存呢？

答案当然是否定的！  
(01) 序列化对static和transient变量，是不会自动进行状态保存的。  
&nbsp;&nbsp;&nbsp;&nbsp; transient的作用就是，用transient声明的变量，不会被自动序列化。  
(02) 对于Socket, Thread类，不支持序列化。若实现序列化的接口中，有Thread成员；在对该类进行序列化操作时，编译会出错！  
&nbsp;&nbsp;&nbsp;&nbsp; 这主要是基于资源分配方面的原因。如果Socket，Thread类可以被序列化，但是被反序列化之后也无法对他们进行重新的资源分配；再者，也是没有必要这样实现。  

下面，我们还是通过示例来查看“序列化对static和transient的处理”。



<a name="anchor4"></a>
# 4. 演示程序3

我们对前面的SerialTest1.java进行简单修改，得到源文件(SerialTest3.java)如下：

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
      
    public class SerialTest3 { 
        private static final String TMP_FILE = ".serialtest3.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
    class Box implements Serializable {
        private static int width;   
        private transient int height; 
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

SerialTest3.java 相比于 SerialTest1.java。仅仅对Box类中的 width 和 height 变量的定义进行了修改。  
SerialTest1.java 中width和height定义

    private int width;   
    private int height; 

SerialTest3.java 中width和height定义

    private static int width; 
    private transient int height; 

在看后面的结果之前，我们建议大家对程序进行分析，先自己得出一个结论。

运行结果：

    testWrite box: [desk: (80, 48) ]
    testRead  box: [desk: (80, 0) ] 

结果分析：

我们前面说过，“序列化不对static和transient变量进行状态保存”。因此，testWrite()中保存Box对象时，不会保存width和height的值。这点是毋庸置疑的！但是，为什么testRead()中读取出来的Box对象的width=80，而height=0呢？  
先说，为什么height=0。因为Box对象中height是int类型，而int类型的默认值是0。  
再说，为什么width=80。这是因为height是static类型，而static类型就意味着所有的Box对象都共用一个height值；而在testWrite()中，我们已经将height初始化为80了。因此，我们通过序列化读取出来的Box对象的height值，也被就是80。  

理解上面的内容之后，我们应该可以推断出下面的代码的运行结果。

源码如下(SerialTest4.java)： 

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
      
    public class SerialTest4 { 
        private static final String TMP_FILE = ".serialtest4.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);
                // 修改box的值
                box = new Box("room", 100, 50);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
    class Box implements Serializable {
        private static int width;   
        private transient int height; 
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

SerialTest4.java 相比于 SerialTest3.java，在testWrite()中添加了一行代码box = new Box("room", 100, 50);

运行结果：

    testWrite box: [desk: (80, 48) ]
    testRead  box: [desk: (100, 0) ]

现在，我们更加确认“序列化不对static和transient变量进行状态保存”。但是，若我们想要保存static或transient变量，能不能办到呢？  
当然可以！我们在类中重写两个方法writeObject()和readObject()即可。下面程序演示了如何手动保存static和transient变量。



<a name="anchor5"></a>
# 5. 演示程序4

我们对前面的SerialTest4.java进行简单修改，以达到：序列化存储static和transient变量的目的。

源码如下(SerialTest5.java)： 

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
    import java.io.IOException;   
    import java.lang.ClassNotFoundException;   
      
    public class SerialTest5 { 
        private static final String TMP_FILE = ".serialtest5.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);
                // 修改box的值
                box = new Box("room", 100, 50);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
    class Box implements Serializable {
        private static int width;   
        private transient int height; 
        private String name;   

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        private void writeObject(ObjectOutputStream out) throws IOException{ 
            out.defaultWriteObject();//使定制的writeObject()方法可以利用自动序列化中内置的逻辑。 
            out.writeInt(height); 
            out.writeInt(width); 
            //System.out.println("Box--writeObject width="+width+", height="+height);
        }

        private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException{ 
            in.defaultReadObject();//defaultReadObject()补充自动序列化 
            height = in.readInt(); 
            width = in.readInt(); 
            //System.out.println("Box---readObject width="+width+", height="+height);
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

运行结果：

    testWrite box: [desk: (80, 48) ]
    testRead  box: [desk: (80, 48) ]

程序说明：

“序列化不会自动保存static和transient变量”，因此我们若要保存它们，则需要通过writeObject()和readObject()去手动读写。  
(01) 通过writeObject()方法，写入要保存的变量。writeObject的原始定义是在ObjectOutputStream.java中，我们按照如下示例覆盖即可：

    private void writeObject(ObjectOutputStream out) throws IOException{ 
        out.defaultWriteObject();// 使定制的writeObject()方法可以利用自动序列化中内置的逻辑。 
        out.writeInt(ival);      // 若要保存“int类型的值”，则使用writeInt()
        out.writeObject(obj);    // 若要保存“Object对象”，则使用writeObject()
    }

(02) 通过readObject()方法，读取之前保存的变量。readObject的原始定义是在ObjectInputStream.java中，我们按照如下示例覆盖即可：

    private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException{ 
        in.defaultReadObject();       // 使定制的readObject()方法可以利用自动序列化中内置的逻辑。 
        int ival = in.readInt();      // 若要读取“int类型的值”，则使用readInt()
        Object obj = in.readObject(); // 若要读取“Object对象”，则使用readObject()
    }

至此，我们就介绍完了“序列化对static和transient变量的处理”。  
接下来，我们来研究“对于Socket, Thread类，不支持序列化”。还是通过示例来查看。


<a name="anchor6"></a>
# 6. 演示程序5

我们修改SerialTest5.java的源码，在Box类中添加一个Thread成员。

源码如下(SerialTest6.java)：

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
    import java.lang.Thread;
    import java.io.IOException;   
    import java.lang.ClassNotFoundException;   
      
    public class SerialTest6 { 
        private static final String TMP_FILE = ".serialtest6.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);
                // 修改box的值
                box = new Box("room", 100, 50);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
    class Box implements Serializable {
        private static int width;   
        private transient int height; 
        private String name;   
        private Thread thread = new Thread() {
            @Override
            public void run() {
                System.out.println("Serializable thread");
            }
        };

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        private void writeObject(ObjectOutputStream out) throws IOException{ 
            out.defaultWriteObject();//使定制的writeObject()方法可以利用自动序列化中内置的逻辑。 
            out.writeInt(height); 
            out.writeInt(width); 
            //System.out.println("Box--writeObject width="+width+", height="+height);
        }

        private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException{ 
            in.defaultReadObject();//defaultReadObject()补充自动序列化 
            height = in.readInt(); 
            width = in.readInt(); 
            //System.out.println("Box---readObject width="+width+", height="+height);
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

结果是，编译出错！  
事实证明，不能对Thread进行序列化。若希望程序能编译通过，我们对Thread变量添加static或transient修饰即可！如下，是对Thread添加transient修饰的源码(SerialTest7.java)：

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */

    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.Serializable;   
    import java.lang.Thread;
    import java.io.IOException;   
    import java.lang.ClassNotFoundException;   
      
    public class SerialTest7 { 
        private static final String TMP_FILE = ".serialtest7.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象，Box实现了Serializable序列化接口
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);
                // 修改box的值
                box = new Box("room", 100, 50);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类“支持序列化”。因为Box实现了Serializable接口。
     *
     * 实际上，一个类只需要实现Serializable即可实现序列化，而不需要实现任何函数。
     */
    class Box implements Serializable {
        private static int width;   
        private transient int height; 
        private String name;   
        private transient Thread thread = new Thread() {
            @Override
            public void run() {
                System.out.println("Serializable thread");
            }
        };

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        private void writeObject(ObjectOutputStream out) throws IOException{ 
            out.defaultWriteObject();//使定制的writeObject()方法可以利用自动序列化中内置的逻辑。 
            out.writeInt(height); 
            out.writeInt(width); 
            //System.out.println("Box--writeObject width="+width+", height="+height);
        }

        private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException{ 
            in.defaultReadObject();//defaultReadObject()补充自动序列化 
            height = in.readInt(); 
            width = in.readInt(); 
            //System.out.println("Box---readObject width="+width+", height="+height);
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

至此，关于“Serializable接口”来实现序列化的内容，都说完了。为什么这么说？因为，实现序列化，除了Serializable之外，还有其它的方式，就是通过实现Externalizable来实现序列化。整理下心情，下面继续对Externalizable进行了解。



<a name="anchor7"></a>
# 7. Externalizable和完全定制序列化过程

如果一个类要完全负责自己的序列化，则实现Externalizable接口，而不是Serializable接口。

Externalizable接口定义包括两个方法writeExternal()与readExternal()。需要注意的是：声明类实现Externalizable接口会有重大的安全风险。writeExternal()与readExternal()方法声明为public，恶意类可以用这些方法读取和写入对象数据。如果对象包含敏感信息，则要格外小心。

下面，我们修改之前的SerialTest1.java测试程序；将其中的Box由“实现Serializable接口” 改为 “实现Externalizable接口”。   
修改后的源码如下( ExternalizableTest1.java)：

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */
    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.ObjectOutput;   
    import java.io.ObjectInput;   
    import java.io.Serializable;   
    import java.io.Externalizable;   
    import java.io.IOException;   
    import java.lang.ClassNotFoundException;   
      
    public class ExternalizableTest1 { 
        private static final String TMP_FILE = ".externalizabletest1.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类实现Externalizable接口
     */
    class Box implements Externalizable {
        private int width;   
        private int height; 
        private String name;   

        public Box() {
        }

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

运行结果：

    testWrite box: [desk: (80, 48) ]
    testRead  box: [null: (0, 0) ]

说明：

(01) 实现Externalizable接口的类，不会像实现Serializable接口那样，会自动将数据保存。  
(02) 实现Externalizable接口的类，必须实现writeExternal()和readExternal()接口！  
否则，程序无法正常编译！  
(03) 实现Externalizable接口的类，必须定义不带参数的构造函数！  
否则，程序无法正常编译！  
(04) writeExternal() 和 readExternal() 的方法都是public的，不是非常安全！


接着，我们修改上面的ExternalizableTest1.java测试程序；实现Box类中的writeExternal()和readExternal()接口！  
修改后的源码如下( ExternalizableTest2.java)：

    /**
     * 序列化的演示测试程序
     *
     * @author skywang
     */
    import java.io.FileInputStream;   
    import java.io.FileOutputStream;   
    import java.io.ObjectInputStream;   
    import java.io.ObjectOutputStream;   
    import java.io.ObjectOutput;   
    import java.io.ObjectInput;   
    import java.io.Serializable;   
    import java.io.Externalizable;   
    import java.io.IOException;   
    import java.lang.ClassNotFoundException;   
      
    public class ExternalizableTest2 { 
        private static final String TMP_FILE = ".externalizabletest2.txt";
      
        public static void main(String[] args) {   
            // 将“对象”通过序列化保存
            testWrite();
            // 将序列化的“对象”读出来
            testRead();
        }
      

        /**
         * 将Box对象通过序列化，保存到文件中
         */
        private static void testWrite() {   
            try {
                // 获取文件TMP_FILE对应的对象输出流。
                // ObjectOutputStream中，只能写入“基本数据”或“支持序列化的对象”
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(TMP_FILE));
                // 创建Box对象
                Box box = new Box("desk", 80, 48);
                // 将box对象写入到对象输出流out中，即相当于将对象保存到文件TMP_FILE中
                out.writeObject(box);
                // 打印“Box对象”
                System.out.println("testWrite box: " + box);

                out.close();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
     
        /**
         * 从文件中读取出“序列化的Box对象”
         */
        private static void testRead() {
            try {
                // 获取文件TMP_FILE对应的对象输入流。
                ObjectInputStream in = new ObjectInputStream(
                        new FileInputStream(TMP_FILE));
                // 从对象输入流中，读取先前保存的box对象。
                Box box = (Box) in.readObject();
                // 打印“Box对象”
                System.out.println("testRead  box: " + box);
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * Box类实现Externalizable接口
     */
    class Box implements Externalizable {
        private int width;   
        private int height; 
        private String name;   

        public Box() {
        }

        public Box(String name, int width, int height) {
            this.name = name;
            this.width = width;
            this.height = height;
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(name);
            out.writeInt(width);
            out.writeInt(height);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            name = (String) in.readObject();
            width = in.readInt();
            height = in.readInt();
        }

        @Override
        public String toString() {
            return "["+name+": ("+width+", "+height+") ]";
        }
    }

运行结果：

    testWrite box: [desk: (80, 48) ]
    testRead  box: [desk: (80, 48) ]


[link_java_collection_10]: /2012/02/10/collection-10-hashmap
