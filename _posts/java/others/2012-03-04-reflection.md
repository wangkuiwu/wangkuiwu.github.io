---
layout: post
title: "Java 反射机制(包括组成、结构、示例说明等内容)"
description: "java"
category: java
tags: [java]
date: 2012-03-04 09:02
---


> **目录**  
[第1部分 Java 反射机制介绍](#anchor1)  
[第2部分 Class 详细说明](#anchor2)  
 


<a name="anchor1"></a>
# 第1部分 Java 反射机制介绍

Java 反射机制。  
通俗来讲呢，就是在运行状态中，我们可以根据“类的部分已经的信息”来还原“类的全部的信息”。这里“类的部分已经的信息”，可以是“类名”或“类的对象”等信息。“类的全部信息”就是指“类的属性，方法，继承关系和Annotation注解”等内容。

举个简单的例子：假设对于类ReflectionTest.java，我们知道的唯一信息是它的类名是“com.skywang.Reflection”。这时，我们想要知道ReflectionTest.java的其它信息(比如它的构造函数，它的成员变量等等)，要怎么办呢？

这就需要用到“反射”。通过反射，我们可以解析出ReflectionTest.java的完整信息，包括它的构造函数，成员变量，继承关系等等。

在了解了“java 反射机制”的概念之后，接下来思考一个问题：如何根据类的类名，来获取类的完整信息呢？

这个过程主要分为两步：  
第1步：根据“类名”来获取对应类的Class对象。  
第2步：通过Class对象的函数接口，来读取“类的构造函数，成员变量”等信息。

下面，我们根据示例来加深对这个概念的理解。示例如下(Demo1.java):

    package com.skywang.test;

    import java.lang.Class;

    public class Demo1 {

        public static void main(String[] args) {

            try {
                // 根据“类名”获取 对应的Class对象
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 新建对象。newInstance()会调用类不带参数的构造函数
                Object obj = cls.newInstance();

                System.out.println("cls="+cls);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    class Person {
        public Person() {
            System.out.println("create Person");
        }
    }

运行结果：

    create Person
    cls=class com.skywang.test.Person

说明：  
(01) Person类的完整包名是"com.skywang.test.Person"。而 Class.forName("com.skywang.test.Person"); 这一句的作用是，就是根据Person的包名来获取Person的Class对象。  
(02) 接着，我们调用Class对象的newInstance()方法，创建Person对象。


现在，我们知道了“java反射机制”的概念以及它的原理。有了这个总体思想之后，接下来，我们可以开始对反射进行深入研究了。

 

<a name="anchor2"></a>
# 第2部分 Class 详细说明

## 1. 获取Class对象的方法

我这里总结了4种常用的“获取Class对象”的方法：  
**方法1**：Class.forName("类名字符串") （注意：类名字符串必须是全称，包名+类名）  
**方法2**：类名.class  
**方法3**：实例对象.getClass()  
**方法4**："类名字符串".getClass()

下面，我们通过示例演示这4种方法。示例如下(Demo2.java):

    package com.skywang.test;

    import java.lang.Class;

    public class Demo2 {

        public static void main(String[] args) {

            try {
                // 方法1：Class.forName("类名字符串")  （注意：类名字符串必须是全称，包名+类名）
                //Class cls1 = Class.forName("com.skywang.test.Person");
                Class<?> cls1 = Class.forName("com.skywang.test.Person");
                //Class<Person> cls1 = Class.forName("com.skywang.test.Person");

                // 方法2：类名.class
                Class cls2 = Person.class; 

                // 方法3：实例对象.getClass()
                Person person = new Person();
                Class cls3 = person.getClass();

                // 方法4："类名字符串".getClass()
                String str = "com.skywang.test.Person"; 
                Class cls4 = str.getClass();

                System.out.printf("cls1=%s, cls2=%s, cls3=%s, cls4=%s\n", cls1, cls2, cls3, cls4);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    class Person {
        public Person() {
            System.out.println("create Person");
        }
    }

运行结果：

    create Person
    cls1=class com.skywang.test.Person, cls2=class com.skywang.test.Person, cls3=class com.skywang.test.Person, cls4=class java.lang.String

 

## 2 Class的API说明

Class的全部API如下表:

    public static Class    forName(String className)
    public static Class    forName(String name, boolean initialize, ClassLoader loader)
    public Constructor    getConstructor(Class[] parameterTypes)
    public Constructor[]    getConstructors()
    public Constructor    getDeclaredConstructor(Class[] parameterTypes)
    public Constructor[]    getDeclaredConstructors()
    public Constructor    getEnclosingConstructor()
    public Method    getMethod(String name, Class[] parameterTypes)
    public Method[]    getMethods()
    public Method    getDeclaredMethod(String name, Class[] parameterTypes)
    public Method[]    getDeclaredMethods()
    public Method    getEnclosingMethod()
    public Field    getField(String name)
    public Field[]    getFields()
    public Field    getDeclaredField(String name)
    public Field[]    getDeclaredFields()
    public Type[]    getGenericInterfaces()
    public Type    getGenericSuperclass()
    public Annotation<A>    getAnnotation(Class annotationClass)
    public Annotation[]    getAnnotations()
    public Annotation[]    getDeclaredAnnotations()
    public boolean    isAnnotation()
    public boolean    isAnnotationPresent(Class annotationClass)
    public boolean    isAnonymousClass()
    public boolean    isArray()
    public boolean    isAssignableFrom(Class cls)
    public boolean    desiredAssertionStatus()
    public Class<U>    asSubclass(Class clazz)
    public Class    getSuperclass()
    public Class    getComponentType()
    public Class    getDeclaringClass()
    public Class    getEnclosingClass()
    public Class[]    getClasses()
    public Class[]    getDeclaredClasses()
    public Class[]    getInterfaces()
    public boolean    isEnum()
    public boolean    isInstance(Object obj)
    public boolean    isInterface()
    public boolean    isLocalClass()
    public boolean    isMemberClass()
    public boolean    isPrimitive()
    public boolean    isSynthetic()
    public String    getSimpleName()
    public String    getName()
    public String    getCanonicalName()
    public String    toString()
    public ClassLoader    getClassLoader()
    public Package    getPackage()
    public int    getModifiers()
    public ProtectionDomain    getProtectionDomain()
    public URL    getResource(String name)
    public InputStream    getResourceAsStream(String name)
    public Object    cast(Object obj)
    public Object    newInstance()
    public Object[]    getSigners()
    public Object[]    getEnumConstants()
    public TypeVariable[]    getTypeParameters()

我们根据类的特性，将Class中的类分为4部分进行说明：构造函数，成员方法，成员变量，类的其它信息(如注解、包名、类名、继承关系等等)。Class中涉及到Annotation(注解)的相关API，可以点击查看前一章节关于Annotation的详细介绍。

### 2.1 构造函数

“构造函数”相关API

    // 获取“参数是parameterTypes”的public的构造函数
    public Constructor    getConstructor(Class[] parameterTypes)
    // 获取全部的public的构造函数
    public Constructor[]    getConstructors()
    // 获取“参数是parameterTypes”的，并且是类自身声明的构造函数，包含public、protected和private方法。
    public Constructor    getDeclaredConstructor(Class[] parameterTypes)
    // 获取类自身声明的全部的构造函数，包含public、protected和private方法。
    public Constructor[]    getDeclaredConstructors()
    // 如果这个类是“其它类的构造函数中的内部类”，调用getEnclosingConstructor()就是这个类所在的构造函数；若不存在，返回null。
    public Constructor    getEnclosingConstructor()

接下来，我们通过示例对这些API进行说明。示例代码(DemoClassContructor.java)如下：

    package com.skywang.test;

    import java.lang.Class;
    import java.lang.reflect.Constructor;

    /**
     * java Class类的Constructor相关API的测试函数
     *
     * @author skywang
     */
    public class DemoClassContructor {

        public static void main(String[] args) {

            // getDeclaredConstructor() 的测试函数
            testGetDeclaredConstructor() ;

            // getConstructor() 的测试函数
            testGetConstructor() ;

            // getEnclosingConstructor() 的测试函数
            testGetEnclosingConstructor() ;
        }

        /**
         * getDeclaredConstructor() 的测试函数
         */
        public static void testGetDeclaredConstructor() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 根据class，获取构造函数
                Constructor cst1 = cls.getDeclaredConstructor();
                Constructor cst2 = cls.getDeclaredConstructor(new Class[]{String.class});
                Constructor cst3 = cls.getDeclaredConstructor(new Class[]{String.class, int.class, Gender.class});

                // 根据构造函数，创建相应的对象
                cst1.setAccessible(true); // 因为Person中Person()是private的，所以这里要设置为可访问
                Object p1 = cst1.newInstance();
                Object p2 = cst2.newInstance("Juce");
                Object p3 = cst3.newInstance("Jody", 34, Gender.MALE);

                System.out.printf("%-30s: p1=%s, p2=%s, p3=%s\n", 
                        "getConstructor()", p1, p2, p3);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }


        /**
         * getConstructor() 的测试函数
         */
        public static void testGetConstructor() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 根据class，获取构造函数
                //Constructor cst1 = cls.getConstructor(); // 抛出异常，因为默认构造函数是private权限。
                //Constructor cst2 = cls.getConstructor(new Class[]{String.class});// 抛出异常，因为该构造函数是protected权限。
                Constructor cst3 = cls.getConstructor(new Class[]{String.class, int.class, Gender.class});

                // 根据构造函数，创建相应的对象
                //Object p1 = cst1.newInstance();
                //cst1.setAccessible(true); // 因为Person中Person()是private的，所以这里要设置为可访问
                //Object p1 = cst1.newInstance();
                //Object p2 = cst2.newInstance("Kim");
                Object p3 = cst3.newInstance("Katter", 36, Gender.MALE);

                System.out.printf("%-30s: p3=%s\n", 
                        "getConstructor()", p3);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        /**
         * getEnclosingConstructor() 的测试函数
         */
        public static void testGetEnclosingConstructor() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 根据class，调用Person类中有内部类InnerA的构造函数
                Constructor cst = cls.getDeclaredConstructor(new Class[]{String.class, int.class});

                // 根据构造函数，创建相应的对象
                Object obj = cst.newInstance("Ammy", 18);

                System.out.printf("%-30s: obj=%s\n", 
                        "getEnclosingConstructor()", obj);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    // 枚举类型。表示“性别”
    enum Gender{  
        MALE, FEMALE
    } 
    // 人
    class Person {
        private Gender gender;  // 性别
        private int age;        // 年龄
        private String name;    // 姓名

        private Person() {
            this.name = "unknown";
            this.age = 0;
            this.gender = Gender.FEMALE;
            System.out.println("call--\"private Person()\"");
        }
        protected Person(String name) {
            this.name = name;
            this.age = 0;
            this.gender = Gender.FEMALE;
            System.out.println("call--\"protected Person(String name)\"");
        }
        public Person(String name, int age, Gender gender) {
            this.name = name;
            this.age = age;
            this.gender = gender;
            System.out.println("call--\"public Person(String name, int age, Gender gender)\"");
        }

        public Person(String name, int age) {
            this.name = name;
            this.age = age;
            this.gender = Gender.FEMALE;
            //内部类在构造方法中
            class InnerA{
            }
            // 获取InnerA的Class对象
            Class cls = InnerA.class;

            // 获取“封闭该内部类(InnerA)”的构造方法
            Constructor cst = cls.getEnclosingConstructor();
            
            System.out.println("call--\"public Person(String name, int age)\" cst="+cst);
        }
        @Override
        public String toString() {
            return "("+name+", "+age+", "+gender+")";
        }
    }

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    call--"private Person()"
    call--"protected Person(String name)"
    call--"public Person(String name, int age, Gender gender)"
    getConstructor() : p1=(unknown, 0, FEMALE), p2=(Juce, 0, FEMALE), p3=(Jody, 34, MALE)
    call--"public Person(String name, int age, Gender gender)"
    getConstructor() : p3=(Katter, 36, MALE)
    call--"public Person(String name, int age)" cst=public com.skywang.test.Person(java.lang.String,int)
    getEnclosingConstructor() : obj=(Ammy, 18, FEMALE)

说明：  
(01) 首先，要搞清楚Person类，它是我们自定义的类。专门用来测试这些API的。Person中有一个成员变量gender；它是Gender对象，Gender是一个枚举类。取值可以是MALE或者FEMALE。

(02) testGetDeclaredConstructor() 是“getDeclaredConstructor() 的测试函数”。getDeclaredConstructor()可以“获取类中任意的构造函数，包含public、protected和private方法”。

(03) testGetConstructor() 是“getConstructor() 的测试函数”。getConstructor()只能“获取类中public的构造函数”。

(04) testGetEnclosingConstructor() 是“getEnclosingConstructor() 的测试函数”。   
关于getEnclosingConstructor()的介绍，官方说法是“如果该 Class 对象表示构造方法中的一个本地或匿名类，则返回 Constructor 对象，它表示底层类的立即封闭构造方法。否则返回 null。” 通俗点来说，就是***“如果一个类A的构造函数中定义了一个内部类InnerA，则通过InnerA的Class对象调用getEnclosingConstructor()方法，可以获取类A的这个构造函数”。***


### 2.2 成员方法

 “成员方法”相关API

    // 获取“名称是name，参数是parameterTypes”的public的函数(包括从基类继承的、从接口实现的所有public函数)
    public Method    getMethod(String name, Class[] parameterTypes)
    // 获取全部的public的函数(包括从基类继承的、从接口实现的所有public函数)
    public Method[]    getMethods()
    // 获取“名称是name，参数是parameterTypes”，并且是类自身声明的函数，包含public、protected和private方法。
    public Method    getDeclaredMethod(String name, Class[] parameterTypes)
    // 获取全部的类自身声明的函数，包含public、protected和private方法。
    public Method[]    getDeclaredMethods()
    // 如果这个类是“其它类中某个方法的内部类”，调用getEnclosingMethod()就是这个类所在的方法；若不存在，返回null。
    public Method    getEnclosingMethod()

接下来，我们通过示例对这些API进行说明。示例代码(DemoClassMethod.java)如下：

    package com.skywang.test;

    import java.lang.Class;
    import java.lang.reflect.Method;

    /**
     * java Class类的Method相关API的测试函数
     *
     * @author skywang
     */
    public class DemoClassMethod {

        public static void main(String[] args) {

                // getDeclaredMethod() 的测试函数
                testGetDeclaredMethod() ;

                // getMethod() 的测试函数
                testGetMethod() ;

                // getEnclosingMethod() 的测试函数
                testGetEnclosingMethod() ;
        }

        /**
         * getDeclaredMethod() 的测试函数
         */
        public static void testGetDeclaredMethod() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");
                // 根据class，调用类的默认构造函数(不带参数)
                Object person = cls.newInstance();

                // 获取Person中的方法
                Method mSetName = cls.getDeclaredMethod("setName", new Class[]{String.class});
                Method mGetName = cls.getDeclaredMethod("getName", new Class[]{});
                Method mSetAge  = cls.getDeclaredMethod("setAge", new Class[]{int.class});
                Method mGetAge  = cls.getDeclaredMethod("getAge", new Class[]{});
                Method mSetGender = cls.getDeclaredMethod("setGender", new Class[]{Gender.class});
                Method mGetGender = cls.getDeclaredMethod("getGender", new Class[]{});

                // 调用获取的方法
                mSetName.invoke(person, new Object[]{"Jimmy"});
                mSetAge.invoke(person, new Object[]{30});
                mSetGender.setAccessible(true);    // 因为Person中setGender()是private的，所以这里要设置为可访问
                mSetGender.invoke(person, new Object[]{Gender.MALE});
                String name = (String)mGetName.invoke(person, new Object[]{});
                Integer age = (Integer)mGetAge.invoke(person, new Object[]{});
                mGetGender.setAccessible(true);    // 因为Person中getGender()是private的，所以这里要设置为可访问
                Gender gender = (Gender)mGetGender.invoke(person, new Object[]{});

                // 打印输出
                System.out.printf("%-30s: person=%s, name=%s, age=%s, gender=%s\n", 
                        "getDeclaredMethod()", person, name, age, gender);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }


        /**
         * getMethod() 的测试函数
         */
        public static void testGetMethod() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");
                // 根据class，调用类的默认构造函数(不带参数)
                Object person = cls.newInstance();

                // 获取Person中的方法
                Method mSetName = cls.getMethod("setName", new Class[]{String.class});
                Method mGetName = cls.getMethod("getName", new Class[]{});
                //Method mSetAge  = cls.getMethod("setAge", new Class[]{int.class});         // 抛出异常，因为setAge()是protected权限。
                //Method mGetAge  = cls.getMethod("getAge", new Class[]{});                  // 抛出异常，因为getAge()是protected权限。
                //Method mSetGender = cls.getMethod("setGender", new Class[]{Gender.class}); // 抛出异常，因为setGender()是private权限。
                //Method mGetGender = cls.getMethod("getGender", new Class[]{});             // 抛出异常，因为getGender()是private权限。

                // 调用获取的方法
                mSetName.invoke(person, new Object[]{"Phobe"});
                //mSetAge.invoke(person, new Object[]{38});
                //mSetGender.invoke(person, new Object[]{Gender.FEMALE});
                String name = (String)mGetName.invoke(person, new Object[]{});
                //Integer age = (Integer)mGetAge.invoke(person, new Object[]{});
                //Gender gender = (Gender)mGetGender.invoke(person, new Object[]{});

                // 打印输出
                System.out.printf("%-30s: person=%s\n", 
                        "getMethod()", person);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }


        /**
         * getEnclosingMethod() 的测试函数
         */
        public static void testGetEnclosingMethod() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");
                // 根据class，调用类的默认构造函数(不带参数)
                Object person = cls.newInstance();

                // 根据class，调用Person类中有内部类InnerB的函数
                Method mGetInner = cls.getDeclaredMethod("getInner", new Class[]{});

                // 根据构造函数，创建相应的对象
                mGetInner.invoke(person, new Object[]{});

                System.out.printf("%-30s: person=%s\n",
                           "getEnclosingMethod", person);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }


    }

    enum Gender{  
        MALE, FEMALE
    } 
    class Person {
        private Gender gender;  // 性别
        private int age;        // 年龄
        private String name;    // 姓名

        public Person() {
            this("unknown", 0, Gender.FEMALE);
        }
        public Person(String name, int age, Gender gender) {
            this.name = name;
            this.age = age;
            this.gender = gender;
        }

        // 获取”姓名“。权限是 public
        public String getName() {
            return name;
        }
        // 设置”姓名“。权限是 public
        public void setName(String name) {
            this.name = name;
        }
        // 获取”年龄“。权限是 protected
        protected int getAge() {
            return age;
        }
        // 设置”年龄“。权限是 protected
        protected void setAge(int age) {
            this.age = age;
        }
        // 获取“性别”。权限是 private
        private Gender getGender() {
            return gender;
        }
        // 设置“性别”。权限是 private
        private void setGender(Gender gender) {
            this.gender = gender;
        }


        // getInner() 中有内部类InnerB，用来测试getEnclosingMethod()
        public void getInner() {
            // 内部类
            class InnerB{
            }
            // 获取InnerB的Class对象
            Class cls = InnerB.class;

            // 获取“封闭该内部类(InnerB)”的构造方法
            Method cst = cls.getEnclosingMethod();
            
            System.out.println("call--\"getInner()\" cst="+cst);
        }


        @Override
        public String toString() {
            return "("+name+", "+age+", "+gender+")";
        }
    }

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    getDeclaredMethod() : person=(Jimmy, 30, MALE), name=Jimmy, age=30, gender=MALE
    getMethod() : person=(Phobe, 0, FEMALE)
    call--"getInner()" cst=public void com.skywang.test.Person.getInner()
    getEnclosingMethod : person=(unknown, 0, FEMALE)

### 2.3 成员变量

“成员变量”的相关API

    // 获取“名称是name”的public的成员变量(包括从基类继承的、从接口实现的所有public成员变量)
    public Field    getField(String name)
    // 获取全部的public成员变量(包括从基类继承的、从接口实现的所有public成员变量)
    public Field[]    getFields()
    // 获取“名称是name”，并且是类自身声明的成员变量，包含public、protected和private成员变量。
    public Field    getDeclaredField(String name)
    // 获取全部的类自身声明的成员变量，包含public、protected和private成员变量。
    public Field[]    getDeclaredFields()

接下来，我们通过示例对这些API进行说明。示例代码(DemoClassField.java)如下：

    package com.skywang.test;

    import java.lang.Class;
    import java.lang.reflect.Field;

    /**
     * java Class类的"成员变量"相关API的测试函数
     *
     * @author skywang
     */
    public class DemoClassField {

        public static void main(String[] args) {
            // getDeclaredField() 的测试函数
            testGetDeclaredField() ;

            // getField() 的测试函数
            testGetField() ;
        }

        /**
         * getDeclaredField() 的测试函数
         * getDeclaredField() 用于获取的是类自身声明的所有成员遍历，包含public、protected和private方法。
         */
        public static void testGetDeclaredField() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");
                // 根据class，调用类的默认构造函数(不带参数)
                Object person = cls.newInstance();

                // 根据class，获取Filed
                Field fName = cls.getDeclaredField("name");
                Field fAge = cls.getDeclaredField("age");
                Field fGender = cls.getDeclaredField("gender");

                // 根据构造函数，创建相应的对象
                fName.set(person, "Hamier");
                fAge.set(person, 31);
                fGender.setAccessible(true);  // 因为"flag"是private权限，所以要设置访问权限为true；否则，会抛出异常。
                fGender.set(person, Gender.FEMALE);

                System.out.printf("%-30s: person=%s\n", 
                        "getDeclaredField()", person);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        /**
         * getField() 的测试函数
         * getField() 用于获取的是public的“成员”
         */
        public static void testGetField() {
            try {
                // 获取Person类的Class
                Class<?> cls = Class.forName("com.skywang.test.Person");
                // 根据class，调用类的默认构造函数(不带参数)
                Object person = cls.newInstance();

                // 根据class，获取Filed
                Field fName = cls.getField("name");
                Field fAge = cls.getDeclaredField("age");       // 抛出异常，因为Person中age是protected权限。 
                Field fGender = cls.getDeclaredField("gender"); // 抛出异常，因为Person中gender是private权限。 

                // 根据构造函数，创建相应的对象
                fName.set(person, "Grace");
                //fAge.set(person, 26);
                //fGender.set(person, Gender.FEMALE);

                System.out.printf("%-30s: person=%s\n", 
                        "getField()", person);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    // 枚举类型。表示“性别”
    enum Gender{  
        MALE, FEMALE
    } 
    // 人
    class Person {
        // private。性别
        private Gender gender;
        // protected。 年龄
        protected int age;
        // public。 姓名
        public String name;

        public Person() {
            this("unknown", 0, Gender.FEMALE);
        }

        public Person(String name, int age, Gender gender) {
            this.name = name;
            this.age = age;
            this.gender = gender;
        }

        @Override
        public String toString() {
            return "("+name+", "+age+", "+gender+")";
        }
    }

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    getDeclaredField() : person=(Hamier, 31, FEMALE)
    getField() : person=(Grace, 0, FEMALE)

 

### 2.4 类的其它信息

**2.4.1 “注解”相关的API**

    // 获取类的"annotationClass"类型的注解 (包括从基类继承的、从接口实现的所有public成员变量)
    public Annotation<A>    getAnnotation(Class annotationClass)
    // 获取类的全部注解 (包括从基类继承的、从接口实现的所有public成员变量)
    public Annotation[]    getAnnotations()
    // 获取类自身声明的全部注解 (包含public、protected和private成员变量)
    public Annotation[]    getDeclaredAnnotations()

接下来，我们通过示例对这些API进行说明。示例代码(DemoClassAnnotation.java)如下：

    package com.skywang.test;

    import java.lang.Class;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;

    /**
     * java Class类getAnnotation()的测试程序
     */
    public class DemoClassAnnotation {

        public static void main(String[] args) {
            try {
                // 根据“类名”获取 对应的Class对象
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 获取“Person类”的注解
                MyAnnotation myann = cls.getAnnotation(MyAnnotation.class);

                System.out.println("myann="+myann);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * MyAnnotation是自定义个一个Annotation
     */
    @Retention(RetentionPolicy.RUNTIME)
    @interface MyAnnotation {
    }

    /**
     * MyAnnotation 是Person的注解。
     */
    @MyAnnotation 
    class Person {
    }

 

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    myann=@com.skywang.test.MyAnnotation()

说明：  
(01) MyAnnotation 是我们自定义个一个Annotation注解。若读者不明白“注解”，可以参考博文“[Java Annotation认知(包括框架图、详细介绍、示例说明)][link_java_annotation]”  
(02) getAnnotation()就是获取这个类的注解。


**2.4.2 “父类”和“接口”相关的API**

    // 获取实现的全部接口
    public Type[]    getGenericInterfaces()
    // 获取父类
    public Type    getGenericSuperclass()

示例代码(DemoClassInterface.java)如下：

    package com.skywang.test;

    import java.io.Serializable;
    import java.lang.Runnable;
    import java.lang.Thread;
    import java.lang.Class;
    import java.lang.reflect.Type;

    /**
     * java Class类的有关父类和接口的测试
     */
    public class DemoClassInterface {

        public static void main(String[] args) {

            try {
                // 根据“类名”获取 对应的Class对象
                Class<?> cls = Class.forName("com.skywang.test.Person");

                // 获取“Person”的父类
                Type father = cls.getGenericSuperclass();
                // 获取“Person”实现的全部接口
                Type[] intfs = cls.getGenericInterfaces();

                System.out.println("father="+father);
                for (Type t:intfs)
                    System.out.println("t="+t);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * Person 继承于 Object，并且实现了Serializable和Runnable接口
     */
    class Person extends Object implements Serializable, Runnable{

        @Override
        public void run() {
        }

    }

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    father=class java.lang.Object
    t=interface java.io.Serializable
    t=interface java.lang.Runnable

 

**2.4.3 剩余的API**

    // 获取“类名”
    public String    getSimpleName()
    // 获取“完整类名”
    public String    getName()
    // 类是不是“枚举类”
    public boolean    isEnum()
    // obj是不是类的对象
    public boolean    isInstance(Object obj)
    // 类是不是“接口”
    public boolean    isInterface()
    // 类是不是“本地类”。本地类,就是定义在方法内部的类。
    public boolean    isLocalClass()
    // 类是不是“成员类”。成员类,是内部类的一种，但是它不是“内部类”或“匿名类”。
    public boolean    isMemberClass()
    // 类是不是“基本类型”。 基本类型，包括void和boolean、byte、char、short、int、long、float 和 double这几种类型。
    public boolean    isPrimitive()
    // 类是不是“复合类”。 JVM中才会产生复合类，在java应用程序中不存在“复合类”！
    public boolean    isSynthetic()

示例代码(DemoClassOtherAPIs.java)如下：

    package com.skywang.test;

    import java.lang.Class;
    import java.lang.Runnable;
    import java.lang.annotation.ElementType;
    import java.util.TreeMap;

    /**
     * java Class类的getName(), isInterface()等测试程序
     *
     * @author skywang
     */
    public class DemoClassOtherAPIs {
        public static void main(String[] args) {

            Class cls = DemoClassOtherAPIs.class;
            // 获取“类名”
            System.out.printf("%-50s:getSimpleName()=%s\n", cls, cls.getSimpleName());
            // 获取“完整类名”
            System.out.printf("%-50s:getName()=%s\n", cls, cls.getName());

            // 测试其它的API
            testOtherAPIs() ;
        }

        public static void testOtherAPIs() {
            // 本地类
            class LocalA {
            }

            // 测试枚举类型。ElementType是一个枚举类
            Class elementtypeCls = ElementType.class;
            System.out.printf("%-50s:isEnum()=%s\n", 
                    elementtypeCls, elementtypeCls.isEnum());

            // 判断是不是类的对象
            Class demoCls = DemoClassOtherAPIs.class;
            DemoClassOtherAPIs demoObj = new DemoClassOtherAPIs();
            System.out.printf("%-50s:isInstance(obj)=%s\n", 
                    demoCls, demoCls.isInstance(demoObj));

            // 类是不是“接口”
            Class runCls = Runnable.class;
            System.out.printf("%-50s:isInterface()=%s\n", 
                    runCls, runCls.isInterface());

            // 类是不是“本地类”。本地类,就是定义在方法内部的类。
            Class localCls = LocalA.class;
            System.out.printf("%-50s:isLocalClass()=%s\n", 
                    localCls, localCls.isLocalClass());

            // 类是不是“成员类”。成员类,是内部类的一种，但是它不是“内部类”或“匿名类”。
            Class memCls = MemberB.class;
            System.out.printf("%-50s:isMemberClass()=%s\n", 
                    memCls, memCls.isMemberClass());

            // 类是不是“基本类型”。 基本类型，包括void和boolean、byte、char、short、int、long、float 和 double这几种类型。
            Class primCls = int.class;
            System.out.printf("%-50s:isPrimitive()=%s\n", 
                    primCls, primCls.isPrimitive());

            // 类是不是“复合类”。 JVM中才会产生复合类，在java应用程序中不存在“复合类”！
            Class synCls = DemoClassOtherAPIs.class;
            System.out.printf("%-50s:isSynthetic()=%s\n", 
                    synCls, synCls.isSynthetic());
        }

        // 内部成员类
        class MemberB {
        }
    }

注意：若程序无法运行，请检查“forName()”中的包名是否正确！forName()的参数必须是，Person类的完整包名。

运行结果：

    class com.skywang.test.DemoClassOtherAPIs :getSimpleName()=DemoClassOtherAPIs
    class com.skywang.test.DemoClassOtherAPIs :getName()=com.skywang.test.DemoClassOtherAPIs
    class java.lang.annotation.ElementType :isEnum()=true
    class com.skywang.test.DemoClassOtherAPIs :isInstance(obj)=true
    interface java.lang.Runnable :isInterface()=true
    class com.skywang.test.DemoClassOtherAPIs$1LocalA :isLocalClass()=true
    class com.skywang.test.DemoClassOtherAPIs$MemberB :isMemberClass()=true
    int :isPrimitive()=true
    class com.skywang.test.DemoClassOtherAPIs :isSynthetic()=false

说明：isSynthetic()是用来判断Class是不是“复合类”。这在java应用程序中只会返回false，不会返回true。因为，JVM中才会产生复合类，在java应用程序中不存在“复合类”！

 

[link_java_annotation]: /2012/03/03/annotation
