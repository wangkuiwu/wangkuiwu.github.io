---
layout: post
title: "Java Annotation认知(包括框架图、详细介绍、示例说明)"
description: "java"
category: java
tags: [java]
date: 2012-03-03 09:01
---

> Java Annotation是JDK5.0引入的一种注释机制。

> 网上很多关于Java Annotation的文章，看得人眼花缭乱。Java Annotation本来很简单的，结果说的人没说清楚；弄的看的人更加迷糊。

> 我按照自己的思路，对Annotation进行了整理。理解 Annotation 的关键，是理解Annotation的语法和用法，对这些内容，我都进行了详细说明；理解Annotation的语法和用法之后，再看Annotation的框架图，可能有更深刻体会。废话就说这么多，下面开始对Annotation进行说明。若您发现文章中存在错误或不足的地方，希望您能指出！

> **目录**  
[第1部分 Annotation架构](#anchor1)  
[第2部分 Annotation组成部分](#anchor2)  
[第3部分 java自带的Annotation](#anchor3)  
[第4部分 Annotation 的作用](#anchor4)  
 

 

<a name="anchor1"></a>
# 第1部分 Annotation架构

先看看Annotation的架构图：

![img](/media/pic/java/basic/annotation01.jpg)

从中，我们可以看出：  
(01) 1个Annotation 和 1个RetentionPolicy关联。  
&nbsp;&nbsp;&nbsp;&nbsp; 可以理解为：每1个Annotation对象，都会有唯一的RetentionPolicy属性。

(02) 1个Annotation 和 1~n个ElementType关联。  
&nbsp;&nbsp;&nbsp;&nbsp; 可以理解为：对于每1个Annotation对象，可以有若干个ElementType属性。

(03) Annotation 有许多实现类，包括：Deprecated, Documented, Inherited, Override等等。  
&nbsp;&nbsp;&nbsp;&nbsp; Annotation 的每一个实现类，都“和1个RetentionPolicy关联”并且“和1~n个ElementType关联”。


下面，我先介绍框架图的左半边(如下图)，即Annotation, RetentionPolicy, ElementType；然后在就Annotation的实现类进行举例说明。

![img](/media/pic/java/basic/annotation02.jpg)
 

<a name="anchor2"></a>
# 第2部分 Annotation组成部分

## 1. annotation组成成分

java annotation 的组成中，有3个非常重要的主干类。它们分别是：

**(01) Annotation.java**

    package java.lang.annotation;
    public interface Annotation {

        boolean equals(Object obj);

        int hashCode();

        String toString();

        Class<? extends Annotation> annotationType();
    }

**(02) ElementType.java**

    package java.lang.annotation;

    public enum ElementType {
        TYPE,               /* 类、接口（包括注释类型）或枚举声明  */

        FIELD,              /* 字段声明（包括枚举常量）  */

        METHOD,             /* 方法声明  */

        PARAMETER,          /* 参数声明  */

        CONSTRUCTOR,        /* 构造方法声明  */

        LOCAL_VARIABLE,     /* 局部变量声明  */

        ANNOTATION_TYPE,    /* 注释类型声明  */

        PACKAGE             /* 包声明  */
    }

**(03) RetentionPolicy.java**

    package java.lang.annotation;
    public enum RetentionPolicy {
        SOURCE,            /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该Annotation信息了  */

        CLASS,             /* 编译器将Annotation存储于类对应的.class文件中。默认行为  */

        RUNTIME            /* 编译器将Annotation存储于class文件中，并且可由JVM读入 */
    }

说明：  
(01) Annotation 就是个接口。  
&nbsp;&nbsp;&nbsp;&nbsp; “每1个Annotation” 都与 “1个RetentionPolicy”关联，并且与 “1～n个ElementType”关联。可以通俗的理解为：每1个Annotation对象，都会有唯一的RetentionPolicy属性；至于ElementType属性，则有1~n个。  

(02) ElementType 是Enum枚举类型，它用来指定Annotation的类型。  
&nbsp;&nbsp;&nbsp;&nbsp; “每1个Annotation” 都与 “1～n个ElementType”关联。当Annotation与某个ElementType关联时，就意味着：Annotation有了某种用途。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，若一个Annotation对象是METHOD类型，则该Annotation只能用来修饰方法。  

(03) RetentionPolicy 是Enum枚举类型，它用来指定Annotation的策略。通俗点说，就是不同RetentionPolicy类型的Annotation的作用域不同。  
&nbsp;&nbsp;&nbsp;&nbsp; “每1个Annotation” 都与 “1个RetentionPolicy”关联。  
&nbsp;&nbsp;&nbsp;&nbsp; a) 若Annotation的类型为 SOURCE，则意味着：Annotation仅存在于编译器处理期间，编译器处理完之后，该Annotation就没用了。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，“ @Override ”标志就是一个Annotation。当它修饰一个方法的时候，就意味着该方法覆盖父类的方法；并且在编译期间会进行语法检查！编译器处理完后，“@Override”就没有任何作用了。  
&nbsp;&nbsp;&nbsp;&nbsp; b) 若Annotation的类型为 CLASS，则意味着：编译器将Annotation存储于类对应的.class文件中，它是Annotation的默认行为。  
&nbsp;&nbsp;&nbsp;&nbsp; c) 若Annotation的类型为 RUNTIME，则意味着：编译器将Annotation存储于class文件中，并且可由JVM读入。  

这时，只需要记住“每1个Annotation” 都与 “1个RetentionPolicy”关联，并且与 “1～n个ElementType”关联。学完后面的内容之后，再回头看这些内容，会更容易理解。

 

<a name="anchor3"></a>
# 第3部分 java自带的Annotation

理解了上面的3个类的作用之后，我们接下来可以讲解Annotation实现类的语法定义了。

## 1 Annotation通用定义

    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface MyAnnotation1 {
    }

说明：  
上面的作用是定义一个Annotation，它的名字是MyAnnotation1。定义了MyAnnotation1之后，我们可以在代码中通过“@MyAnnotation1”来使用它。

其它的，@Documented, @Target, @Retention, @interface都是来修饰MyAnnotation1的。下面分别说说它们的含义：  
(01) @interface  
&nbsp;&nbsp;&nbsp;&nbsp; 使用@interface定义注解时，意味着它实现了java.lang.annotation.Annotation接口，即该注解就是一个Annotation。  
&nbsp;&nbsp;&nbsp;&nbsp; 定义Annotation时，@interface是必须的。  
&nbsp;&nbsp;&nbsp;&nbsp; 注意：它和我们通常的implemented实现接口的方法不同。Annotation接口的实现细节都由编译器完成。通过@interface定义注解后，该注解不能继承其他的注解或接口。  

(02) @Documented  
&nbsp;&nbsp;&nbsp;&nbsp; 类和方法的Annotation在缺省情况下是不出现在javadoc中的。如果使用@Documented修饰该Annotation，则表示它可以出现在javadoc中。  
&nbsp;&nbsp;&nbsp;&nbsp; 定义Annotation时，@Documented可有可无；若没有定义，则Annotation不会出现在javadoc中。  

(03) @Target(ElementType.TYPE)  
&nbsp;&nbsp;&nbsp;&nbsp; 前面我们说过，ElementType 是Annotation的类型属性。而@Target的作用，就是来指定Annotation的类型属性。  
&nbsp;&nbsp;&nbsp;&nbsp; @Target(ElementType.TYPE) 的意思就是指定该Annotation的类型是ElementType.TYPE。这就意味着，MyAnnotation1是来修饰“类、接口（包括注释类型）或枚举声明”的注解。  
&nbsp;&nbsp;&nbsp;&nbsp; 定义Annotation时，@Target可有可无。若有@Target，则该Annotation只能用于它所指定的地方；若没有@Target，则该Annotation可以用于任何地方。  

(04) @Retention(RetentionPolicy.RUNTIME)  
&nbsp;&nbsp;&nbsp;&nbsp; 前面我们说过，RetentionPolicy 是Annotation的策略属性，而@Retention的作用，就是指定Annotation的策略属性。  
&nbsp;&nbsp;&nbsp;&nbsp; @Retention(RetentionPolicy.RUNTIME) 的意思就是指定该Annotation的策略是RetentionPolicy.RUNTIME。这就意味着，编译器会将该Annotation信息保留在.class文件中，并且能被虚拟机读取。  
&nbsp;&nbsp;&nbsp;&nbsp; 定义Annotation时，@Retention可有可无。若没有@Retention，则默认是RetentionPolicy.CLASS。


## 2 java自带的Annotation

通过上面的示例，我们能理解：@interface用来声明Annotation，@Documented用来表示该Annotation是否会出现在javadoc中， @Target用来指定Annotation的类型，@Retention用来指定Annotation的策略。

理解这一点之后，我们就很容易理解java中自带的Annotation的实现类，即Annotation架构图的右半边。如下图：

![img](/media/pic/java/basic/annotation03.jpg)

java 常用的Annotation：

|      标注        |                说明                |
| ---------------- | ---------------------------------- |
| @Deprecated  | @Deprecated 所标注内容，不再被建议使用。 |
| @Override    | @Override 只能标注方法，表示该方法覆盖父类中的方法。 |
| @Documented  | @Documented 所标注内容，可以出现在javadoc中。 |
| @Inherited   | @Inherited只能被用来标注“Annotation类型”，它所标注的Annotation具有继承性。 |
| @Retention   | @Retention只能被用来标注“Annotation类型”，而且它被用来指定Annotation的RetentionPolicy属性。 |
| @Target      | @Target只能被用来标注“Annotation类型”，而且它被用来指定Annotation的ElementType属性。 |
| @SuppressWarnings | @SuppressWarnings 所标注内容产生的警告，编译器会对这些警告保持静默。 |

 

由于“@Deprecated和@Override”类似，“@Documented, @Inherited, @Retention, @Target”类似；下面，我们只对@Deprecated, @Inherited, @SuppressWarnings 这3个Annotation进行说明。

### 2.1 @Deprecated

@Deprecated 的定义如下：

    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Deprecated {
    }

说明：  
(01) @interface -- 它的用来修饰Deprecated，意味着Deprecated实现了java.lang.annotation.Annotation接口；即Deprecated就是一个注解。  

(02) @Documented -- 它的作用是说明该注解能出现在javadoc中。  

(03) @Retention(RetentionPolicy.RUNTIME) -- 它的作用是指定Deprecated的策略是RetentionPolicy.RUNTIME。这就意味着，编译器会将Deprecated的信息保留在.class文件中，并且能被虚拟机读取。  

(04) @Deprecated 所标注内容，不再被建议使用。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，若某个方法被 @Deprecated 标注，则该方法不再被建议使用。如果有开发人员试图使用或重写被@Deprecated标示的方法，编译器会给相应的提示信息。示例如下:

![img](/media/pic/java/basic/annotation04.jpg)

源码如下(DeprecatedTest.java)：

    package com.skywang.annotation;

    import java.util.Date;
    import java.util.Calendar;

    public class DeprecatedTest {
        // @Deprecated 修饰 getString1(),表示 它是建议不被使用的函数
        @Deprecated
        private static void getString1(){
            System.out.println("Deprecated Method");
        }
        
        private static void getString2(){
            System.out.println("Normal Method");
        }
        
        // Date是日期/时间类。java已经不建议使用该类了
        private static void testDate() {
            Date date = new Date(113, 8, 25);
            System.out.println(date.getYear());
        }
        // Calendar是日期/时间类。java建议使用Calendar取代Date表示“日期/时间”
        private static void testCalendar() {
            Calendar cal = Calendar.getInstance();
            System.out.println(cal.get(Calendar.YEAR));
        }
        
        public static void main(String[] args) {
            getString1(); 
            getString2();
            testDate(); 
            testCalendar();
        }
    }

说明：  
上面是eclipse中的截图，比较类中 “getString1() 和 getString2()” 以及 “testDate() 和 testCalendar()” 。

(01) getString1() 被@Deprecated标注，意味着建议不再使用getString1()；所以getString1()的定义和调用时，都会一横线。这一横线是eclipse()对@Deprecated方法的处理。  
getString2() 没有被@Deprecated标注，它的显示正常。  

(02) testDate() 调用了Date的相关方法，而java已经建议不再使用Date操作日期/时间。因此，在调用Date的API时，会产生警告信息，途中的warnings。  
testCalendar() 调用了Calendar的API来操作日期/时间，java建议用Calendar取代Date。因此，操作Calendar不回产生warning。

 

### 2.2 @Inherited

@Inherited 的定义如下：

    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.ANNOTATION_TYPE)
    public @interface Inherited {
    }

说明：  
(01) @interface -- 它的用来修饰Inherited，意味着Inherited实现了java.lang.annotation.Annotation接口；即Inherited就是一个注解。  

(02) @Documented -- 它的作用是说明该注解能出现在javadoc中。  

(03) @Retention(RetentionPolicy.RUNTIME) -- 它的作用是指定Inherited的策略是RetentionPolicy.RUNTIME。这就意味着，编译器会将Inherited的信息保留在.class文件中，并且能被虚拟机读取。  

(04) @Target(ElementType.ANNOTATION_TYPE) -- 它的作用是指定Inherited的类型是ANNOTATION_TYPE。这就意味着，@Inherited只能被用来标注“Annotation类型”。  

(05) @Inherited 的含义是，它所标注的Annotation将具有继承性。  
&nbsp;&nbsp;&nbsp;&nbsp; 假设，我们定义了某个Annotaion，它的名称是MyAnnotation，并且MyAnnotation被标注为@Inherited。现在，某个类Base使用了MyAnnotation，则Base具有了“具有了注解MyAnnotation”；现在，Sub继承了Base，由于MyAnnotation是@Inherited的(具有继承性)，所以，Sub也“具有了注解MyAnnotation”。

@Inherited的使用示例  
源码如下(InheritableSon.java)：

    /**
     * @Inherited 演示示例
     * 
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    package com.skywang.annotation;

    import java.lang.annotation.Target;
    import java.lang.annotation.ElementType;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;
    import java.lang.annotation.Inherited;

    /**
     * 自定义的Annotation。
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Inherited
    @interface Inheritable
    {
    }

    @Inheritable
    class InheritableFather
    {
        public InheritableFather() {
            // InheritableBase是否具有 Inheritable Annotation
            System.out.println("InheritableFather:"+InheritableFather.class.isAnnotationPresent(Inheritable.class));
        }
    }

    /**
     * InheritableSon 类只是继承于 InheritableFather，
     */
    public class InheritableSon extends InheritableFather
    {
        public InheritableSon() {
            super();    // 调用父类的构造函数
            // InheritableSon类是否具有 Inheritable Annotation
            System.out.println("InheritableSon:"+InheritableSon.class.isAnnotationPresent(Inheritable.class));
        }
        
        public static void main(String[] args)
        {
            InheritableSon is = new InheritableSon();
        }
    }

运行结果：

    InheritableFather:true
    InheritableSon:true

现在，我们对InheritableSon.java进行修改：注释掉“Inheritable的@Inherited注解”。  
源码如下(InheritableSon.java)：

    /**
     * @Inherited 演示示例
     * 
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    package com.skywang.annotation;

    import java.lang.annotation.Target;
    import java.lang.annotation.ElementType;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;
    import java.lang.annotation.Inherited;

    /**
     * 自定义的Annotation。
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    //@Inherited
    @interface Inheritable
    {
    }

    @Inheritable
    class InheritableFather
    {
        public InheritableFather() {
            // InheritableBase是否具有 Inheritable Annotation
            System.out.println("InheritableFather:"+InheritableFather.class.isAnnotationPresent(Inheritable.class));
        }
    }

    /**
     * InheritableSon 类只是继承于 InheritableFather，
     */
    public class InheritableSon extends InheritableFather
    {
        public InheritableSon() {
            super();    // 调用父类的构造函数
            // InheritableSon类是否具有 Inheritable Annotation
            System.out.println("InheritableSon:"+InheritableSon.class.isAnnotationPresent(Inheritable.class));
        }
        
        public static void main(String[] args)
        {
            InheritableSon is = new InheritableSon();
        }
    }

运行结果：

    InheritableFather:true
    InheritableSon:false

对比上面的两个结果，我们发现：当注解Inheritable被@Inherited标注时，它具有继承性。否则，没有继承性。

 

### 2.3 @SuppressWarnings

@SuppressWarnings 的定义如下：

    @Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
    @Retention(RetentionPolicy.SOURCE)
    public @interface SuppressWarnings {
        String[] value();
    }

说明：  
(01) @interface -- 它的用来修饰SuppressWarnings，意味着SuppressWarnings实现了java.lang.annotation.Annotation接口；即SuppressWarnings就是一个注解。  

(02) @Retention(RetentionPolicy.SOURCE) -- 它的作用是指定SuppressWarnings的策略是RetentionPolicy.SOURCE。这就意味着，SuppressWarnings信息仅存在于编译器处理期间，编译器处理完之后SuppressWarnings就没有作用了。  

(03) @Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE}) -- 它的作用是指定SuppressWarnings的类型同时包括TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE。  
> TYPE意味着，它能标注“类、接口（包括注释类型）或枚举声明”。  
FIELD意味着，它能标注“字段声明”。  
METHOD意味着，它能标注“方法”。  
PARAMETER意味着，它能标注“参数”。  
CONSTRUCTOR意味着，它能标注“构造方法”。  
LOCAL_VARIABLE意味着，它能标注“局部变量”。

(04) String[] value(); 意味着，SuppressWarnings能指定参数  

(05) SuppressWarnings 的作用是，让编译器对“它所标注的内容”的某些警告保持静默。例如，"@SuppressWarnings(value={"deprecation", "unchecked"})" 表示对“它所标注的内容”中的 “SuppressWarnings不再建议使用警告”和“未检查的转换时的警告”保持沉默。示例如下：

![img](/media/pic/java/basic/annotation05.jpg)

源码如下(SuppressWarningTest.java)：

    package com.skywang.annotation;

    import java.util.Date;

    public class SuppressWarningTest {

        //@SuppressWarnings(value={"deprecation"})
        public static void doSomething(){
            Date date = new Date(113, 8, 26);
            System.out.println(date);
        }

        public static void main(String[] args) {
            doSomething();
        }
    }

说明：  
(01) 左边的图中，没有使用 @SuppressWarnings(value={"deprecation"}) , 而Date属于java不再建议使用的类。因此，调用Date的API时，会产生警告。  
而右边的图中，使用了 @SuppressWarnings(value={"deprecation"})。因此，编译器对“调用Date的API产生的警告”保持沉默。

补充：SuppressWarnings 常用的关键字的表格

|    关键字    |                说明                |
| ------------ | ---------------------------------- |
| deprecation  | 使用了不赞成使用的类或方法时的警告 |
| unchecked    | 执行了未检查的转换时的警告，例如当使用集合时没有用泛型 (Generics) 来指定集合保存的类型。 |
| fallthrough  | 当 Switch 程序块直接通往下一种情况而没有 Break 时的警告。 |
| path         | 在类路径、源文件路径等中有不存在的路径时的警告。 |
| serial       | 当在可序列化的类上缺少 serialVersionUID 定义时的警告。 |
| finally      | 任何 finally 子句不能正常完成时的警告。
| all          | 关于以上所有情况的警告。 |

 

<a name="anchor4"></a>
# 第4部分 Annotation 的作用

Annotation 是一个辅助类，它在Junit、Struts、Spring等工具框架中被广泛使用。

我们在编程中经常会使用到的Annotation作用有：

## 1 编译检查

Annotation具有“让编译器进行编译检查的作用”。

例如，@SuppressWarnings, @Deprecated和@Override都具有编译检查作用。  

(01) 关于@SuppressWarnings和@Deprecated，已经在“第3部分”中详细介绍过了。这里就不再举例说明了。  

(02) 若某个方法被 @Override的 标注，则意味着该方法会覆盖父类中的同名方法。如果有方法被@Override标示，但父类中却没有“被@Override标注”的同名方法，则编译器会报错。示例如下：

![img](/media/pic/java/basic/annotation06.jpg)

源码(OverrideTest.java):

    package com.skywang.annotation;

    /**
     * @Override测试程序
     * 
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    public class OverrideTest {

        /**
         * toString() 在java.lang.Object中定义；
         * 因此，这里用 @Override 标注是对的。
         */
        @Override
        public String toString(){
            return "Override toString";
        }

        /**
         * getString() 没有在OverrideTest的任何父类中定义；
         * 但是，这里却用 @Override 标注，因此会产生编译错误！
         */
        @Override
        public String getString(){
            return "get toString";
        }
        
        public static void main(String[] args) {
        }
    }

上面是该程序在eclipse中的截图。从中，我们可以发现“getString()”函数会报错。这是因为“getString() 被@Override所标注，但在OverrideTest的任何父类中都没有定义getString1()函数”。  
“将getString() 上面的@Override注释掉”，即可解决该错误。

 

## 2 在反射中使用Annotation

在反射的Class, Method, Field等函数中，有许多于Annotation相关的接口。  
这也意味着，我们可以在反射中解析并使用Annotation。  
源码如下(AnnotationTest.java)：

    package com.skywang.annotation;

    import java.lang.annotation.Annotation;
    import java.lang.annotation.Target;
    import java.lang.annotation.ElementType;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;
    import java.lang.annotation.Inherited;
    import java.lang.reflect.Method;

    /**
     * Annotation在反射函数中的使用示例
     * 
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    @Retention(RetentionPolicy.RUNTIME)
    @interface MyAnnotation {
        String[] value() default "unknown";
    }

    /**
     * Person类。它会使用MyAnnotation注解。
     */
    class Person {
        
        /**
         * empty()方法同时被 "@Deprecated" 和 “@MyAnnotation(value={"a","b"})”所标注 
         * (01) @Deprecated，意味着empty()方法，不再被建议使用
         * (02) @MyAnnotation, 意味着empty() 方法对应的MyAnnotation的value值是默认值"unknown"
         */
        @MyAnnotation
        @Deprecated
        public void empty(){
            System.out.println("\nempty");
        }
        
        /**
         * sombody() 被 @MyAnnotation(value={"girl","boy"}) 所标注，
         * @MyAnnotation(value={"girl","boy"}), 意味着MyAnnotation的value值是{"girl","boy"}
         */
        @MyAnnotation(value={"girl","boy"})
        public void somebody(String name, int age){
            System.out.println("\nsomebody: "+name+", "+age);
        }
    }

    public class AnnotationTest {

        public static void main(String[] args) throws Exception {
            
            // 新建Person
            Person person = new Person();
            // 获取Person的Class实例
            Class<Person> c = Person.class;
            // 获取 somebody() 方法的Method实例
            Method mSomebody = c.getMethod("somebody", new Class[]{String.class, int.class});
            // 执行该方法
            mSomebody.invoke(person, new Object[]{"lily", 18});
            iteratorAnnotations(mSomebody);
            

            // 获取 somebody() 方法的Method实例
            Method mEmpty = c.getMethod("empty", new Class[]{});
            // 执行该方法
            mEmpty.invoke(person, new Object[]{});        
            iteratorAnnotations(mEmpty);
        }
        
        public static void iteratorAnnotations(Method method) {

            // 判断 somebody() 方法是否包含MyAnnotation注解
            if(method.isAnnotationPresent(MyAnnotation.class)){
                // 获取该方法的MyAnnotation注解实例
                MyAnnotation myAnnotation = method.getAnnotation(MyAnnotation.class);
                // 获取 myAnnotation的值，并打印出来
                String[] values = myAnnotation.value();
                for (String str:values)
                    System.out.printf(str+", ");
                System.out.println();
            }
            
            // 获取方法上的所有注解，并打印出来
            Annotation[] annotations = method.getAnnotations();
            for(Annotation annotation : annotations){
                System.out.println(annotation);
            }
        }
    }

运行结果：

    somebody: lily, 18
    girl, boy,
    @com.skywang.annotation.MyAnnotation(value=[girl, boy])

    empty
    unknown,
    @com.skywang.annotation.MyAnnotation(value=[unknown])
    @java.lang.Deprecated()

 

## 3 根据Annotation生成帮助文档

通过给Annotation注解加上@Documented标签，能使该Annotation标签出现在javadoc中。

 

## 4 能够帮忙查看查看代码

通过@Override, @Deprecated等，我们能很方便的了解程序的大致结构。  
另外，我们也可以通过自定义Annotation来实现一些功能。

