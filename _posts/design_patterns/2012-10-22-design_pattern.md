---
layout: post
title: "设计模式12之 享元(Flyweight)模式(结构模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-22 09:01
---
 
> 本章介绍"享元模式"。

> **目录**  
[1. 享元模式简介](#anchor1)  
[2. 单纯享元模式](#anchor2)  
[3. 复合享元模式](#anchor3)  

 
<a name="anchor1"></a>
# 1. 享元模式简介

享元模式(Flyweight)是对象的结构模式，它以共享的方式高效地支持大量的细粒度对象。

它的好处是，通过共享来避免大量拥有相同内容对象的开销。

享元模式中的对象称为享元对象，享元对象分为内蕴状态和外蕴状态。  
内蕴对象和外蕴对象是相互独立的：内蕴状态是存储在享元对象内部，并且不会随环境改变而有所不同的；内蕴状态是可以共享。外蕴状态是随环境改变而改变，不可以共享的状态；享云对象的外蕴状态必须由客户端保存，并在享元对象被创建之后，在需要使用的时候再传入到享云对象内部。


<br/>
一个比较典型的享云模式示例就是："Java中的String对象"。

例如，定义一个String常量a和b，它们的值为"hello"，定义如下：

    String a="hello";
    String b="hello";

实际上，a和b都是存在常量池中。由于a和b的值都是"hello"，它们实际上是常量池的同一个对象！

常量池(constant pool)指的是在编译期就被确定，并被保存在已编译的.class文件中的一些数据。它包括了关于类、方法、接口等中的常量，也包括字符串常量。

 

## String示例1

    public class String1 {
        public static void main(String[] args) {
            String a = "hello";
            String b = "hello";
            String c = "he"+"llo";
            System.out.println(a==b);
            System.out.println(a==c);
        }
    }

运行结果：

    true
    true

结果说明：  
(01) a和b中的"hello"都是字符串常量，它们存储在常量池中，在编译期就被确定了，所以a==b为true。  
(02) "he"和"llo"也都是字符串常量，当一个字符串由多个字符串常量连接而成时，它自己肯定也是字符串常量，所以c也同样在编译期就被解析为一个字符串常量，所以c也是常量池中"hello"的一个引用。

 

## String示例2

    public class String2 {
        public static void main(String[] args) {
            String a = "hello";
            String b = new String("hello");
            String c = "he"+new String("llo");
            System.out.println(a==b);
            System.out.println(a==c);
            System.out.println(b==c);
        }
    }

运行结果：

    false
    false
    false

结果说明：a还是常量池中"hello"字符串常量；而b则无法在编译期确定，它是运行时创建"hello"对象的引用；而c因为有后半部分new String("llo")，所以c也无法在编译期确定，因此，c也是一个新创建的"hello"对象的应用。


继续回到享元模式。它分为"**单纯享元模式**" 和 "**复合享元模式**"。

 
<a name="anchor2"></a>
# 2. 单纯享元模式

在单纯的享元模式中，所有的享元对象都是可以共享的。它的UML类图如下：

![img](/media/pic/design_patterns/pattern12_01.jpg)

单纯享元模式涉及到的角色：**抽象享元(Flyweight)，具体享元(ConcreteFlyweight)，享元工厂(FlyweightFactory)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象享元 | 该角色是所有的具体享元类的超类，它给出了具体享元类需要实现的公共接口。 |
| 具体享元 | 实现抽象享元角色所规定出的接口。 |
| 享元工厂 | 本角色负责创建和管理享元角色。本角色必须保证享元对象可以被系统适当地共享。当一个客户端对象调用一个享元对象的时候，享元工厂角色会检查系统中是否已经有一个符合要求的享元对象。如果已经有了，享元工厂角色就应当提供这个已有的享元对象；如果系统中没有一个适当的享元对象的话，享元工厂角色就应当创建一个合适的享元对象。 |


## 2.1 抽象享元

    abstract public class Flyweight {
        // 示意性方法，参数state是外蕴状态。
        public abstract void operation(String state);
    }

 

## 2.2 具体享元

    public class ConcreteFlyweight extends Flyweight {
        private Character intrinsicState = null;

        // 内蕴状态作为参数传入
        public ConcreteFlyweight(Character state) {
            this.intrinsicState = state;
        }
        // 外蕴状态作为参量传入方法中，改变方法的行为。
        public void operation(String state) {
            System.out.println("\tIntrisic State="
                + intrinsicState 
                + ", Extriinsic State="+state);
        }
    }

 

## 2.3 享元工厂

    import java.util.Map;
    import java.util.HashMap;
    import java.util.Iterator;

    public class FlyweightFactory {
        private HashMap flies = new HashMap();

        public Flyweight factory(Character state) {
            if (flies.containsKey(state)) {
                System.out.println(state+" exists.");
                return (Flyweight)flies.get(state);
            } else {
                System.out.println(state+" not exists!");
                Flyweight fly = new ConcreteFlyweight(state);
                flies.put(state, fly);
                return fly;
            }
        }
    }

 

## 2.4 客户端测试程序

    public class Client {

        public static void main(String[] args) {
            FlyweightFactory factory = new FlyweightFactory();
            Flyweight fly = factory.factory(new Character('a'));
            fly.operation("First Call");
            
            fly = factory.factory(new Character('b'));
            fly.operation("Second Call");
            
            fly = factory.factory(new Character('a'));
            fly.operation("Third Call");
        }
    }

 

运行结果：

    a not exists!
    Intrisic State=a, Extriinsic State=First Call
    b not exists!
    Intrisic State=b, Extriinsic State=Second Call
    a exists.
    Intrisic State=a, Extriinsic State=Third Call

结果说明：

抽象享元类Flyweight提供了operation()公共接口。  
具体享元类ConcreteFlyweight实现了抽象享元所给定的接口。  
享元工厂类FlyweightFactory负责创建和管理享元角色。  
客户端不能直接将具体享元类实例化，而必须通过调用工厂对象的factory()来获取享元对象。  
在示例中，虽然客户端申请了三个享元对象，但是实际创建的享元对象只有两个，这就是共享的含义。

 


<a name="anchor3"></a>
# 3. 复合享元模式

在单纯享元模式中，所有的享元对象都是单纯享元对象，也就是说都是可以直接共享的。而还有一种较为复杂的情况，将一些单纯享元使用合成模式加以复合，形成复合享元对象。这样的复合享元对象本身不能共享，但是它们可以分解成单纯享元对象，而后者则可以共享。

复合享元模式的UML类图如下：

![img](/media/pic/design_patterns/pattern12_02.jpg)

复合享元角色所涉及到的角色如下：**抽象享元(Flyweight)，具体享元(ConcreteFlyweight)，复合享元(ConcreteCompositeFlyweight)，享元工厂(FlyweightFactory)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象享元 | 该角色是所有的具体享元类的超类，它给出了具体享元类需要实现的公共接口。  |
| 具体享元 | 实现抽象享元角色所规定出的接口。  |
| 复合享元 | 复合享元角色所代表的对象是不可以共享的，但是一个复合享元对象可以分解成为多个本身是单纯享元对象的组合。复合享元角色又称作不可共享的享元对象。  |
| 享元工厂 | 本角色负责创建和管理享元角色。  |

 

## 3.1 抽象享元

抽象享元类是Flyweight，它定义了operation()公共接口，operation()方法接收一个外蕴状态作为参量。

    abstract public class Flyweight {
        // 示意性方法，参数state是外蕴状态
        public abstract void operation(String state);
    }

 

## 3.2 具体享元

具体享元类是ConcreteFlyweight。它主要责任有两个：  
(01) 实现抽象享元所定义的接口operation()。  
(02) 为内涵状态提供存储空间，也就是ConcreteFlyweight中的intrinsicState树形。

    public class ConcreteFlyweight extends Flyweight {
        private Character intrinsicState = null;

        // 构造函数，内蕴状态作为参数传入
        public ConcreteFlyweight(Character state){
            this.intrinsicState = state;
        }
        
        // 外蕴状态作为参数传入方法中，改变方法的行为，
        @Override
        public void operation(String state) {
            System.out.println("\tIntrisic State="
                + intrinsicState 
                + ", Extriinsic State="+state);
        }
    }

 

## 3.3 复合享元

复合享元类是ConcreteCompositeFlyweight。

复合享元是由"单纯享元"对象通过复合而成，因此，它提供了add()这样的聚集管理方法。由于一个复合享元对象具有不同的聚集元素，这些聚集元素在复合享元对象被创建之后加入，这本身就意味着复合享元对象的状态是会改变的，因此复合享元对象是不能共享的。

此外，复合享元角色实现了抽象享元角色所规定的接口，也就是operation()方法，这个方法有一个参数，代表复合享元对象的外蕴状态。一个复合享元对象的所有单纯享元对象元素的外蕴状态都是与复合享元对象的外蕴状态相等的；而一个复合享元对象所含有的单纯享元对象的内蕴状态一般是不相等的，不然就没有使用价值了。

    import java.util.Map;
    import java.util.HashMap;
    import java.util.Iterator;

    // 复合享元：它由若干个"单纯享元对象"组成。
    public class ConcreteCompositeFlyweight extends Flyweight {
        
        private HashMap flies = new HashMap();
        private Flyweight flyweight;

        // 增加一个新的单纯享元对象到复合享元中
        public void add(Character key , Flyweight fly) {
            flies.put(key,fly);
        }

        // 外蕴状态作为参数传入到方法中
        @Override
        public void operation(String state) {
            Flyweight fly = null;
            for (Iterator it = flies.entrySet().iterator();
                 it.hasNext(); ) {
                Map.Entry e = (Map.Entry) it.next();
                fly = (Flyweight) e.getValue();
                fly.operation(state);
            }
        }
    }

 

## 3.4 享元工厂

享元工厂是FlyweightFactory类。

享元工厂提供两种不同的"返回抽象享元"的方法，一种用于提供单纯享元对象，另一种用于提供复合享元对象。

当需要单纯享元的时候，就调用factory()，并传入Character类型的参数。  
当需要复合享元的时候，就调用factory()，并传入String类型的参数。

    import java.util.Map;
    import java.util.HashMap;
    import java.util.Iterator;

    public class FlyweightFactory {
        private HashMap flies = new HashMap();

        // 创建"复合享元"的工厂方法
        public Flyweight factory(String compositeState){
            ConcreteCompositeFlyweight compositeFly = new ConcreteCompositeFlyweight();
            int length = compositeState.length();
            Character state = null;
            for (int i=0; i<length; i++) {
                state = new Character(compositeState.charAt(i));
                System.out.println("factory("+state+")");
                // 通过 this.factory(state)创建"单纯享元"，
                // 然后 将"单纯享元"添加到"复合享元"中。
                compositeFly.add(state, this.factory(state));
            }
     
            return compositeFly;
        }

        // 创建"单纯享元"的工厂方法
        public Flyweight factory(Character state){
            if (flies.containsKey(state)) {
                return (Flyweight)flies.get(state);
            } else {
                Flyweight fly = new ConcreteFlyweight(state);
                flies.put(state, fly);
                return fly;
            }
        }
    }

 

## 3.5 客户端测试程序

客户端是Client。

    public class Client {

        public static void main(String[] args) {
            FlyweightFactory factory = new FlyweightFactory();

            String str = "abcda";
            Flyweight com1 = factory.factory(str);
            Flyweight com2 = factory.factory(str);
            com1.operation("Composite Call");
            com2.operation("Composite Call");
            System.out.println("com1==com2:"+(com1==com2));

            Character c = '1';
            Flyweight pure1 = factory.factory(c);
            Flyweight pure2 = factory.factory(c);
            pure1.operation("pure Call");
            pure2.operation("pure Call");
            System.out.println("pure1==pure2:"+(pure1==pure2));
        }
    }

 

运行结果：

    factory(a)
    factory(b)
    factory(c)
    factory(d)
    factory(a)
    factory(a)
    factory(b)
    factory(c)
    factory(d)
    factory(a)
        Intrisic State=d, Extriinsic State=Composite Call
        Intrisic State=b, Extriinsic State=Composite Call
        Intrisic State=c, Extriinsic State=Composite Call
        Intrisic State=a, Extriinsic State=Composite Call
        Intrisic State=d, Extriinsic State=Composite Call
        Intrisic State=b, Extriinsic State=Composite Call
        Intrisic State=c, Extriinsic State=Composite Call
        Intrisic State=a, Extriinsic State=Composite Call
    com1==com2:false
        Intrisic State=1, Extriinsic State=pure Call
        Intrisic State=1, Extriinsic State=pure Call

结果说明：  
Client中创建两个复合享元com1和com2，它们的内容都是"abcda"。但是com1==com2:false表明：复合享元是不能够共享的。  
Client中创建两个单纯享元pure1和pure2，它们的内容都是'1'。pure1==pure2:true表明：单纯享元是可以共享的。


 
