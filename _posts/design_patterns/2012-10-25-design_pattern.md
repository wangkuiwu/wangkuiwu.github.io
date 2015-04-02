---
layout: post
title: "设计模式15之 不变(Immutable)模式(行为模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-25 09:01
---
 

> 本章介绍"不变模式"。

> **目录**  
[1. 不变模式简介](#anchor1)  
[2. 不变模式示例](#anchor2)  
[3. 不变模式的优缺点](#anchor3)  

 
<a name="anchor1"></a>
# 1. 不变模式简介

一个对象的状态在对象被创建之后就不再变化，这就是所谓的不变(Immutable)模式。
 

## 不变模式的结构

　　不变模式可增强对象的强壮型(robustness)。不变模式允许多个对象共享某一个对象，降低了对该对象进行并发访问时的同步化开销。如果需要修改一个不变对象的状态，那么就需要建立一个新的同类型对象，并在创建时将这个新的状态存储在新对象里。

　　不变模式只涉及到一个类。一个类的内部状态创建后，在整个生命周期都不会发生变化时，这样的类称作不变类。这种使用不变类的做法叫做不变模式。不变模式有两种形式：一种是弱不变模式，另一种是强不变模式。

 

## 弱不变模式

　　一个类的实例的状态是不可改变的；但是这个类的子类的实例具有可能会变化的状态。这样的类符合弱不变模式的定义。要实现弱不变模式，一个类必须满足下面条件：

(01) 所考虑的对象没有任何方法会修改对象的状态；这样一来，当对象的构造函数将对象的状态初始化之后，对象的状态便不再改变。  
(02) 所有属性都应当是私有的。不要声明任何的公开的属性，以防客户端对象直接修改任何的内部状态。  
(03) 这个对象所引用到的其他对象如何是可变对象的话，必须设法限制外界对这些可变对象的访问，以防止外界修改这些对象。如何可能，应当尽量在不变对象内部初始化这些被引用的对象，而不要在客户端初始化，然后再传入到不变对象内部来。如果某个可变对象必须在客户端初始化，然后再传入到不变对象里的话，就应当考虑在不变对象初始化的时候，将这个可变对象复制一份，而不要使用原来的拷贝。

弱不变模式的缺点是：  
第一, 一个弱不变对象的子对象可以是可变对象；换言之，一个弱不变对象的子对象可能是可变的。  
第二, 这个可变的子对象可能可以修改父对象的状态，从而可能会允许外界修改父对象的状态。

 

## 强不变模式

　　一个类的实例不会改变，同时它的子类的实例也具有不可变化的状态。这样的类符合强不变模式。要实现强不变模式，一个类必须首先满足弱不变模式所要求的所有条件，并且还有满足下面条件之一：  
(01) 所考虑的类所有的方法都应当是final，这样这个类的子类不能够置换掉此类的方法。  
(02) 这个类本身就是final的，那么这个类就不可能会有子类，从而也就不可能有被子类修改的问题。



 
<a name="anchor2"></a>
# 2. 不变模式示例

通过以下几个示例来看看"弱不变模式"和"强不变模式"。

## 2.1 可变模式

    class Person {
        private String name;
        private int age;
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        public void setName(String name) { this.name = name; }
        public String getName() { return name; }
        public void setAge(int age) { this.age = age; }
        public int getAge() { return age; }

        public String toString() {
            return "name:"+name+", age:"+age;
        }
    }

    // Complex不是"弱不变类"。
    // 因为它包含了可变类对象person，并且person是在外部初始化的。
    class Complex {
        private Person person;
        private int id;

        public Complex (Person person, int id) {
            this.person = person;
            this.id = id;
        }

        public int getId() { return id; }
        public Person getPerson() { return person; }

        public String toString() {
            return "id:"+id+", person("+person+")";
        }
    }

    public class MutableDemo {
        public static void main(String[] args) {
            // 新建Complex对象，在"外部(即，非Complex内部)"初始化person。
            Person person = new Person("张三", 18);
            Complex com = new Complex(person, 1);
            System.out.println(com);

            // 修改person的名字为"李四"，年龄为20
            person.setName("李四");
            person.setAge(20);
            System.out.println(com);
        }
    }

运行结果：

    id:1, person(name:张三, age:18)
    id:1, person(name:李四, age:20)

结果说明：  
MutableDemo中的Complex类满足"弱不变模式"的前2个条件：  
(01) Complex中没有方法可以修改对象的状态。Complex包含了person和id两个属性值，但是没有方法修改这两个树形。  
(02) Complex中的person和id属性都是私有的。  
　　但是，Complex不满足"若不变模式"的第3个条件。Complex引用了"可变类对象person"，person是Person类的对象，而Person是一个可变类！  
　　所以，在MutableDemo中，我们初始化了Complex的person值为("张三",18)之后；在后面修改了person的名字为"李四",年龄为18。Complex中的person的值也会相应的改变！所以，Complex是"可变类"。

　　若想将Complex修改为"不变类"，需要将Complex修改的满足"若不变模式"的第3个条件。方法是，在Complex中初始化person的时候，创建一个person的拷贝即可。详细代码请看"2. 弱不变模式"。

 

## 2.2 弱不变模式

    class Person {
        private String name;
        private int age;
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        public void setName(String name) { this.name = name; }
        public String getName() { return name; }
        public void setAge(int age) { this.age = age; }
        public int getAge() { return age; }

        public String toString() {
            return "name:"+name+", age:"+age;
        }
    }

    // Complex是"弱不变类"。
    // 它虽然包含了可变的外部对象person，但是Complex内部的person是不会改变的。因为它保存的是person的拷贝。
    class Complex {
        private Person person;
        private int id;

        public Complex (Person person, int id) {
            // 新建一个person的拷贝。
            this.person = new Person(person.getName(), person.getAge());
            this.id = id;
        }

        public int getId() { return id; }
        public Person getPerson() { return person; }

        public String toString() {
            return "id:"+id+", person("+person+")";
        }
    }

    public class WeakImmutable {
        public static void main(String[] args) {
            // 新建Complex对象。
            // id是1, person的名字是"张三"，年龄是18
            Person person = new Person("张三", 18);
            Complex com = new Complex(person, 1);
            System.out.println(com);

            // 修改person的名字为"李四"，年龄为20
            person.setName("李四");
            person.setAge(20);
            System.out.println(com);
        }
    }

运行结果：

    id:1, person(name:张三, age:18)
    id:1, person(name:张三, age:18)

结果说明：WeakImmutable相比MutableDemo，只对Complex的构造函数进行了修改。Complex同时满足"弱不变模式"的3个条件，因此，它就是个不变模式。

 

## 2.3 强不变模式

    class Person {
        private String name;
        private int age;
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        public void setName(String name) { this.name = name; }
        public String getName() { return name; }
        public void setAge(int age) { this.age = age; }
        public int getAge() { return age; }

        public String toString() {
            return "name:"+name+", age:"+age;
        }
    }

    // Complex是"强不变类"。
    final class Complex {
        private final Person person;
        private final int id;

        public Complex (Person person, int id) {
            // 新建一个person的拷贝。
            this.person = new Person(person.getName(), person.getAge());
            this.id = id;
        }

        public final int getId() { return id; }
        public final Person getPerson() { return person; }

        public String toString() {
            return "id:"+id+", person("+person+")";
        }
    }

    public class StrongImmutable {
        public static void main(String[] args) {
            // 新建Complex对象。
            // id是1, person的名字是"张三"，年龄是18
            Person person = new Person("张三", 18);
            Complex com = new Complex(person, 1);
            System.out.println(com);

            // 修改person的名字为"李四"，年龄为20
            person.setName("李四");
            person.setAge(20);
            System.out.println(com);
        }
    }

运行结果：

    id:1, person(name:张三, age:18)
    id:1, person(name:张三, age:18)

结果说明：  
StrongImmutable中的Complex满足"强不变模式"的要求。StrongImmutable相比WeakImmutable，进行了两个修改：  
第一，将Complex中的属性和方法都修改为final类型。  
第二，将Complex类本身修改为final类型。

 


<a name="anchor3"></a>
# 3. 不变模式的优缺点

不变模式有很明显的优点：  
(1) 因为不能修改一个不变对象的状态，所以可以避免由此引起的不必要的程序错误；换言之，一个不变的对象要比可变的对象更加容易维护。  
(2) 因为没有任何一个线程能够修改不变对象的内部状态，一个不变对象自动就是线程安全的，这样就可以省掉处理同步化的开销。一个不变对象可以自由地被不同的客户端共享。

不变模式的缺点：  
不变模式唯一的缺点是：一旦需要修改一个不变对象的状态，就只好创建一个新的同类对象。在需要频繁修改不变对象的环境里，会有大量的不变对象作为中间结果被创建出来，再被JAVA垃圾收集器收集走。这是一种资源上的浪费。

在设计任何一个类的时候，应当慎重考虑其状态是否有需要变化的可能性。除非其状态有变化的必要，不然应当将它设计成不变类。

 
