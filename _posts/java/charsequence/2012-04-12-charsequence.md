---
layout: post
title: "Java 字符串系列02 StringBuilder详解"
description: "java charsequence"
category: java
tags: [java]
date: 2012-04-12 09:01
---


> 本章介绍StringBuilder以及它的API的详细使用方法。

> **目录**  
[1. StringBuilder 简介](#anchor1)   
[2. StringBuilder的API测试代码](#anchor2)   
[3. StringBuilder 完整示例](#anchor3)   


<a name="anchor1"></a>
# 1. StringBuilder 简介

StringBuilder 是一个可变的字符序列。它继承于AbstractStringBuilder，实现了CharSequence接口。  
StringBuffer 也是继承于AbstractStringBuilder的子类；但是，StringBuilder和StringBuffer不同，前者是非线程安全的，后者是线程安全的。

StringBuilder 和 CharSequence之间的关系图如下：

![img](/media/pic/java/charsequence/charsequence-02.jpg)


StringBuilder函数列表

    StringBuilder()
    StringBuilder(int capacity)
    StringBuilder(CharSequence seq)
    StringBuilder(String str)

    StringBuilder     append(float f)
    StringBuilder     append(double d)
    StringBuilder     append(boolean b)
    StringBuilder     append(int i)
    StringBuilder     append(long l)
    StringBuilder     append(char c)
    StringBuilder     append(char[] chars)
    StringBuilder     append(char[] str, int offset, int len)
    StringBuilder     append(String str)
    StringBuilder     append(Object obj)
    StringBuilder     append(StringBuffer sb)
    StringBuilder     append(CharSequence csq)
    StringBuilder     append(CharSequence csq, int start, int end)
    StringBuilder     appendCodePoint(int codePoint)
    int     capacity()
    char     charAt(int index)
    int     codePointAt(int index)
    int     codePointBefore(int index)
    int     codePointCount(int start, int end)
    StringBuilder     delete(int start, int end)
    StringBuilder     deleteCharAt(int index)
    void     ensureCapacity(int min)
    void     getChars(int start, int end, char[] dst, int dstStart)
    int     indexOf(String subString, int start)
    int     indexOf(String string)
    StringBuilder     insert(int offset, boolean b)
    StringBuilder     insert(int offset, int i)
    StringBuilder     insert(int offset, long l)
    StringBuilder     insert(int offset, float f)
    StringBuilder     insert(int offset, double d)
    StringBuilder     insert(int offset, char c)
    StringBuilder     insert(int offset, char[] ch)
    StringBuilder     insert(int offset, char[] str, int strOffset, int strLen)
    StringBuilder     insert(int offset, String str)
    StringBuilder     insert(int offset, Object obj)
    StringBuilder     insert(int offset, CharSequence s)
    StringBuilder     insert(int offset, CharSequence s, int start, int end)
    int     lastIndexOf(String string)
    int     lastIndexOf(String subString, int start)
    int     length()
    int     offsetByCodePoints(int index, int codePointOffset)
    StringBuilder     replace(int start, int end, String string)
    StringBuilder     reverse()
    void     setCharAt(int index, char ch)
    void     setLength(int length)
    CharSequence     subSequence(int start, int end)
    String     substring(int start)
    String     substring(int start, int end)
    String     toString()
    void     trimToSize()

 
由于AbstractStringBuilder和StringBuilder源码太长，这里就不列出源码了。感兴趣的读者可以自行研究。


<a name="anchor2"></a>
# 2. StringBuilder的API测试代码

## 2.1 StringBuilder 中插入(insert)相关的API

源码如下(StringBuilderInsertTest.java):

    /**
     * StringBuilder 的insert()示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderInsertTest {

        public static void main(String[] args) {
            testInsertAPIs() ;
        }

        /**
         * StringBuilder 的insert()示例
         */
        private static void testInsertAPIs() {

            System.out.println("-------------------------------- testInsertAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder();

            // 在位置0处插入字符数组
            sbuilder.insert(0, new char[]{'a','b','c','d','e'});
            // 在位置0处插入字符数组。0表示字符数组起始位置，3表示长度
            sbuilder.insert(0, new char[]{'A','B','C','D','E'}, 0, 3);
            // 在位置0处插入float
            sbuilder.insert(0, 1.414f);
            // 在位置0处插入double
            sbuilder.insert(0, 3.14159d);
            // 在位置0处插入boolean
            sbuilder.insert(0, true);
            // 在位置0处插入char
            sbuilder.insert(0, '\n');
            // 在位置0处插入int
            sbuilder.insert(0, 100);
            // 在位置0处插入long
            sbuilder.insert(0, 12345L);
            // 在位置0处插入StringBuilder对象
            sbuilder.insert(0, new StringBuilder("StringBuilder"));
            // 在位置0处插入StringBuilder对象。6表示被在位置0处插入对象的起始位置(包括)，13是结束位置(不包括)
            sbuilder.insert(0, new StringBuilder("STRINGBUILDER"), 6, 13);
            // 在位置0处插入StringBuffer对象。
            sbuilder.insert(0, new StringBuffer("StringBuffer"));
            // 在位置0处插入StringBuffer对象。6表示被在位置0处插入对象的起始位置(包括)，12是结束位置(不包括)
            sbuilder.insert(0, new StringBuffer("STRINGBUFFER"), 6, 12);
            // 在位置0处插入String对象。
            sbuilder.insert(0, "String");
            // 在位置0处插入String对象。1表示被在位置0处插入对象的起始位置(包括)，6是结束位置(不包括)
            sbuilder.insert(0, "0123456789", 1, 6);
            sbuilder.insert(0, '\n');

            // 在位置0处插入Object对象。此处以HashMap为例
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            sbuilder.insert(0, map);

            System.out.printf("%s\n\n", sbuilder);
        }
    }

运行结果：

    -------------------------------- testInsertAPIs -------------------------------
    {3=three, 2=two, 1=one}
    12345StringBUFFERStringBufferBUILDERStringBuilder12345100
    true3.141591.414ABCabcde

 
# 2.2 StringBuilder 中追加(append)相关的API

源码如下(StringBuilderAppendTest.java):

    /**
     * StringBuilder 的append()示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderAppendTest {

        public static void main(String[] args) {
            testAppendAPIs() ;
        }

        /**
         * StringBuilder 的append()示例
         */
        private static void testAppendAPIs() {

            System.out.println("-------------------------------- testAppendAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder();

            // 追加字符数组
            sbuilder.append(new char[]{'a','b','c','d','e'});
            // 追加字符数组。0表示字符数组起始位置，3表示长度
            sbuilder.append(new char[]{'A','B','C','D','E'}, 0, 3);
            // 追加float
            sbuilder.append(1.414f);
            // 追加double
            sbuilder.append(3.14159d);
            // 追加boolean
            sbuilder.append(true);
            // 追加char
            sbuilder.append('\n');
            // 追加int
            sbuilder.append(100);
            // 追加long
            sbuilder.append(12345L);
            // 追加StringBuilder对象
            sbuilder.append(new StringBuilder("StringBuilder"));
            // 追加StringBuilder对象。6表示被追加对象的起始位置(包括)，13是结束位置(不包括)
            sbuilder.append(new StringBuilder("STRINGBUILDER"), 6, 13);
            // 追加StringBuffer对象。
            sbuilder.append(new StringBuffer("StringBuffer"));
            // 追加StringBuffer对象。6表示被追加对象的起始位置(包括)，12是结束位置(不包括)
            sbuilder.append(new StringBuffer("STRINGBUFFER"), 6, 12);
            // 追加String对象。
            sbuilder.append("String");
            // 追加String对象。1表示被追加对象的起始位置(包括)，6是结束位置(不包括)
            sbuilder.append("0123456789", 1, 6);
            sbuilder.append('\n');

            // 追加Object对象。此处以HashMap为例
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            sbuilder.append(map);
            sbuilder.append('\n');

            // 追加unicode编码
            sbuilder.appendCodePoint(0x5b57);    // 0x5b57是“字”的unicode编码
            sbuilder.appendCodePoint(0x7b26);    // 0x7b26是“符”的unicode编码
            sbuilder.appendCodePoint(0x7f16);    // 0x7f16是“编”的unicode编码
            sbuilder.appendCodePoint(0x7801);    // 0x7801是“码”的unicode编码

            System.out.printf("%s\n\n", sbuilder);
        }
    }

运行结果：

    -------------------------------- testAppendAPIs -------------------------------
    abcdeABC1.4143.14159true
    10012345StringBuilderBUILDERStringBufferBUFFERString12345
    {3=three, 2=two, 1=one}
    字符编码

 
## 2.3 StringBuilder 中替换(replace)相关的API

源码如下(StringBuilderReplaceTest.java):

    /**
     * StringBuilder 的replace()示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderReplaceTest {

        public static void main(String[] args) {
            testReplaceAPIs() ;
        }

        /**
         * StringBuilder 的replace()示例
         */
        private static void testReplaceAPIs() {

            System.out.println("-------------------------------- testReplaceAPIs ------------------------------");

            StringBuilder sbuilder;

            sbuilder = new StringBuilder("0123456789");
            sbuilder.replace(0, 3, "ABCDE");
            System.out.printf("sbuilder=%s\n", sbuilder);

            sbuilder = new StringBuilder("0123456789");
            sbuilder.reverse();
            System.out.printf("sbuilder=%s\n", sbuilder);

            sbuilder = new StringBuilder("0123456789");
            sbuilder.setCharAt(0, 'M');
            System.out.printf("sbuilder=%s\n", sbuilder);

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testReplaceAPIs ------------------------------
    sbuilder=ABCDE3456789
    sbuilder=9876543210
    sbuilder=M123456789

 
## 2.4 StringBuilder 中删除(delete)相关的API

    源码如下(StringBuilderDeleteTest.java):

    /**
     * StringBuilder 的delete()示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderDeleteTest {

        public static void main(String[] args) {
            testDeleteAPIs() ;
        }

        /**
         * StringBuilder 的delete()示例
         */
        private static void testDeleteAPIs() {

            System.out.println("-------------------------------- testDeleteAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder("0123456789");
            
            // 删除位置0的字符，剩余字符是“123456789”。
            sbuilder.deleteCharAt(0);
            // 删除位置3(包括)到位置6(不包括)之间的字符，剩余字符是“123789”。
            sbuilder.delete(3,6);

            // 获取sb中从位置1开始的字符串
            String str1 = sbuilder.substring(1);
            // 获取sb中从位置3(包括)到位置5(不包括)之间的字符串
            String str2 = sbuilder.substring(3, 5);
            // 获取sb中从位置3(包括)到位置5(不包括)之间的字符串，获取的对象是CharSequence对象，此处转型为String
            String str3 = (String)sbuilder.subSequence(3, 5);

            System.out.printf("sbuilder=%s\nstr1=%s\nstr2=%s\nstr3=%s\n", 
                    sbuilder, str1, str2, str3);
        }
    }

运行结果：

    -------------------------------- testDeleteAPIs -------------------------------
    sbuilder=123789
    str1=23789
    str2=78
    str3=78

 
## 2.5 StringBuilder 中index相关的API

源码如下(StringBuilderIndexTest.java):

    /**
     * StringBuilder 中index相关API演示
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderIndexTest {

        public static void main(String[] args) {
            testIndexAPIs() ;
        }

        /**
         * StringBuilder 中index相关API演示
         */
        private static void testIndexAPIs() {
            System.out.println("-------------------------------- testIndexAPIs --------------------------------");

            StringBuilder sbuilder = new StringBuilder("abcAbcABCabCaBcAbCaBCabc");
            System.out.printf("sbuilder=%s\n", sbuilder);

            // 1. 从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.indexOf(\"bc\")", sbuilder.indexOf("bc"));

            // 2. 从位置5开始，从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.indexOf(\"bc\", 5)", sbuilder.indexOf("bc", 5));

            // 3. 从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.lastIndexOf(\"bc\")", sbuilder.lastIndexOf("bc"));

            // 4. 从位置4开始，从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.lastIndexOf(\"bc\", 4)", sbuilder.lastIndexOf("bc", 4));

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testIndexAPIs --------------------------------
    sbuilder=abcAbcABCabCaBcAbCaBCabc
    sbuilder.indexOf("bc")         = 1
    sbuilder.indexOf("bc", 5)      = 22
    sbuilder.lastIndexOf("bc")     = 22
    sbuilder.lastIndexOf("bc", 4)  = 4

 
## 2.6 StringBuilder 剩余的API

源码如下(StringBuilderOtherTest.java):

    /**
     * StringBuilder 的其它API示例
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderOtherTest {

        public static void main(String[] args) {
            testOtherAPIs() ;
        }

        /**
         * StringBuilder 的其它API示例
         */
        private static void testOtherAPIs() {

            System.out.println("-------------------------------- testOtherAPIs --------------------------------");

            StringBuilder sbuilder = new StringBuilder("0123456789");

            int cap = sbuilder.capacity();
            System.out.printf("cap=%d\n", cap);

            char c = sbuilder.charAt(6);
            System.out.printf("c=%c\n", c);

            char[] carr = new char[4];
            sbuilder.getChars(3, 7, carr, 0);
            for (int i=0; i<carr.length; i++)
                System.out.printf("carr[%d]=%c ", i, carr[i]);
            System.out.println();

            System.out.println();
        }
    }

运行结果：

    -------------------------------- testOtherAPIs --------------------------------
    cap=26
    c=6
    carr[0]=3 carr[1]=4 carr[2]=5 carr[3]=6 

 
<a name="anchor3"></a>
# 3. StringBuilder 完整示例

下面的示例是整合上面的几个示例的完整的StringBuilder演示程序，源码如下(StringBuilderTest.java):

    /**
     * StringBuilder 演示程序
     *
     * @author skywang
     */
    import java.util.HashMap;

    public class StringBuilderTest {

        public static void main(String[] args) {
            testOtherAPIs() ;
            testIndexAPIs() ;
            testInsertAPIs() ;
            testAppendAPIs() ;
            testReplaceAPIs() ;
            testDeleteAPIs() ;
        }

        /**
         * StringBuilder 的其它API示例
         */
        private static void testOtherAPIs() {

            System.out.println("-------------------------------- testOtherAPIs --------------------------------");

            StringBuilder sbuilder = new StringBuilder("0123456789");

            int cap = sbuilder.capacity();
            System.out.printf("cap=%d\n", cap);

            char c = sbuilder.charAt(6);
            System.out.printf("c=%c\n", c);

            char[] carr = new char[4];
            sbuilder.getChars(3, 7, carr, 0);
            for (int i=0; i<carr.length; i++)
                System.out.printf("carr[%d]=%c ", i, carr[i]);
            System.out.println();

            System.out.println();
        }

        /**
         * StringBuilder 中index相关API演示
         */
        private static void testIndexAPIs() {
            System.out.println("-------------------------------- testIndexAPIs --------------------------------");

            StringBuilder sbuilder = new StringBuilder("abcAbcABCabCaBcAbCaBCabc");
            System.out.printf("sbuilder=%s\n", sbuilder);

            // 1. 从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.indexOf(\"bc\")", sbuilder.indexOf("bc"));

            // 2. 从位置5开始，从前往后，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.indexOf(\"bc\", 5)", sbuilder.indexOf("bc", 5));

            // 3. 从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.lastIndexOf(\"bc\")", sbuilder.lastIndexOf("bc"));

            // 4. 从位置4开始，从后往前，找出"bc"第一次出现的位置
            System.out.printf("%-30s = %d\n", "sbuilder.lastIndexOf(\"bc\", 4)", sbuilder.lastIndexOf("bc", 4));

            System.out.println();
        }

        /**
         * StringBuilder 的replace()示例
         */
        private static void testReplaceAPIs() {

            System.out.println("-------------------------------- testReplaceAPIs ------------------------------");

            StringBuilder sbuilder;

            sbuilder = new StringBuilder("0123456789");
            sbuilder.replace(0, 3, "ABCDE");
            System.out.printf("sbuilder=%s\n", sbuilder);

            sbuilder = new StringBuilder("0123456789");
            sbuilder.reverse();
            System.out.printf("sbuilder=%s\n", sbuilder);

            sbuilder = new StringBuilder("0123456789");
            sbuilder.setCharAt(0, 'M');
            System.out.printf("sbuilder=%s\n", sbuilder);

            System.out.println();
        }

        /**
         * StringBuilder 的delete()示例
         */
        private static void testDeleteAPIs() {

            System.out.println("-------------------------------- testDeleteAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder("0123456789");
            
            // 删除位置0的字符，剩余字符是“123456789”。
            sbuilder.deleteCharAt(0);
            // 删除位置3(包括)到位置6(不包括)之间的字符，剩余字符是“123789”。
            sbuilder.delete(3,6);

            // 获取sb中从位置1开始的字符串
            String str1 = sbuilder.substring(1);
            // 获取sb中从位置3(包括)到位置5(不包括)之间的字符串
            String str2 = sbuilder.substring(3, 5);
            // 获取sb中从位置3(包括)到位置5(不包括)之间的字符串，获取的对象是CharSequence对象，此处转型为String
            String str3 = (String)sbuilder.subSequence(3, 5);

            System.out.printf("sbuilder=%s\nstr1=%s\nstr2=%s\nstr3=%s\n", 
                    sbuilder, str1, str2, str3);
        }

        /**
         * StringBuilder 的insert()示例
         */
        private static void testInsertAPIs() {

            System.out.println("-------------------------------- testInsertAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder();

            // 在位置0处插入字符数组
            sbuilder.insert(0, new char[]{'a','b','c','d','e'});
            // 在位置0处插入字符数组。0表示字符数组起始位置，3表示长度
            sbuilder.insert(0, new char[]{'A','B','C','D','E'}, 0, 3);
            // 在位置0处插入float
            sbuilder.insert(0, 1.414f);
            // 在位置0处插入double
            sbuilder.insert(0, 3.14159d);
            // 在位置0处插入boolean
            sbuilder.insert(0, true);
            // 在位置0处插入char
            sbuilder.insert(0, '\n');
            // 在位置0处插入int
            sbuilder.insert(0, 100);
            // 在位置0处插入long
            sbuilder.insert(0, 12345L);
            // 在位置0处插入StringBuilder对象
            sbuilder.insert(0, new StringBuilder("StringBuilder"));
            // 在位置0处插入StringBuilder对象。6表示被在位置0处插入对象的起始位置(包括)，13是结束位置(不包括)
            sbuilder.insert(0, new StringBuilder("STRINGBUILDER"), 6, 13);
            // 在位置0处插入StringBuffer对象。
            sbuilder.insert(0, new StringBuffer("StringBuffer"));
            // 在位置0处插入StringBuffer对象。6表示被在位置0处插入对象的起始位置(包括)，12是结束位置(不包括)
            sbuilder.insert(0, new StringBuffer("STRINGBUFFER"), 6, 12);
            // 在位置0处插入String对象。
            sbuilder.insert(0, "String");
            // 在位置0处插入String对象。1表示被在位置0处插入对象的起始位置(包括)，6是结束位置(不包括)
            sbuilder.insert(0, "0123456789", 1, 6);
            sbuilder.insert(0, '\n');

            // 在位置0处插入Object对象。此处以HashMap为例
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            sbuilder.insert(0, map);

            System.out.printf("%s\n\n", sbuilder);
        }

        /**
         * StringBuilder 的append()示例
         */
        private static void testAppendAPIs() {

            System.out.println("-------------------------------- testAppendAPIs -------------------------------");

            StringBuilder sbuilder = new StringBuilder();

            // 追加字符数组
            sbuilder.append(new char[]{'a','b','c','d','e'});
            // 追加字符数组。0表示字符数组起始位置，3表示长度
            sbuilder.append(new char[]{'A','B','C','D','E'}, 0, 3);
            // 追加float
            sbuilder.append(1.414f);
            // 追加double
            sbuilder.append(3.14159d);
            // 追加boolean
            sbuilder.append(true);
            // 追加char
            sbuilder.append('\n');
            // 追加int
            sbuilder.append(100);
            // 追加long
            sbuilder.append(12345L);
            // 追加StringBuilder对象
            sbuilder.append(new StringBuilder("StringBuilder"));
            // 追加StringBuilder对象。6表示被追加对象的起始位置(包括)，13是结束位置(不包括)
            sbuilder.append(new StringBuilder("STRINGBUILDER"), 6, 13);
            // 追加StringBuffer对象。
            sbuilder.append(new StringBuffer("StringBuffer"));
            // 追加StringBuffer对象。6表示被追加对象的起始位置(包括)，12是结束位置(不包括)
            sbuilder.append(new StringBuffer("STRINGBUFFER"), 6, 12);
            // 追加String对象。
            sbuilder.append("String");
            // 追加String对象。1表示被追加对象的起始位置(包括)，6是结束位置(不包括)
            sbuilder.append("0123456789", 1, 6);
            sbuilder.append('\n');

            // 追加Object对象。此处以HashMap为例
            HashMap map = new HashMap();
            map.put("1", "one");
            map.put("2", "two");
            map.put("3", "three");
            sbuilder.append(map);
            sbuilder.append('\n');

            // 追加unicode编码
            sbuilder.appendCodePoint(0x5b57);    // 0x5b57是“字”的unicode编码
            sbuilder.appendCodePoint(0x7b26);    // 0x7b26是“符”的unicode编码
            sbuilder.appendCodePoint(0x7f16);    // 0x7f16是“编”的unicode编码
            sbuilder.appendCodePoint(0x7801);    // 0x7801是“码”的unicode编码

            System.out.printf("%s\n\n", sbuilder);
        }
    }
 

