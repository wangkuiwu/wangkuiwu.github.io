---
layout: post
title: "Java异常(三) 《Java Puzzles》中关于异常的几个谜题"
description: "java exception"
category: java
tags: [java]
date: 2012-04-16 09:01
---
 
> 本章介绍《Java Puzzles》中关于异常的几个谜题。这一章都是以代码为例，相比上一章看起来更有意思。

> **目录**  
[谜题1: 优柔寡断](#anchor1)  
[谜题2: 极端不可思议](#anchor2)  
[谜题3: 不受欢迎的宾客](#anchor3)  
[谜题4: 您好,再见!](#anchor4)  
[谜题5: 不情愿的构造器](#anchor5)  
[谜题6: 域和流](#anchor6)  
[谜题7: 异常为循环而抛](#anchor7)  


 
<a name="anchor1"></a>
# 谜题1: 优柔寡断

看看下面的程序，它到底打印什么？

    public class Indecisive {

        public static void main(String[] args) {
            System.out.println(decision());
        }

        private static boolean decision() {
            try {
                return true;
            } finally {
                return false;
            }
        }
    }

运行结果：

    false

结果说明：

在一个 try-finally 语句中,finally 语句块总是在控制权离开 try 语句块时执行的。无论 try 语句块是正常结束的,还是意外结束的, 情况都是如此。

一条语句或一个语句块在它抛出了一个异常,或者对某个封闭型语句执行了一个 break 或 continue,或是象这个程序一样在方法中执行了一个return 时,将发生意外结束。它们之所以被称为意外结束,是因为它们阻止程序去按顺序执行下面的语句。当 try 语句块和 finally 语句块都意外结束时, try 语句块中引发意外结束的原因将被丢弃, 而整个 try-finally 语句意外结束的原因将于 finally 语句块意外结束的原因相同。在这个程序中,在 try 语句块中的 return 语句所引发的意外结束将被丢弃, try-finally 语句意外结束是由 finally 语句块中的 return 而造成的。

简单地讲, 程序尝试着 (try) (return) 返回 true, 但是它最终 (finally) 返回(return)的是 false。丢弃意外结束的原因几乎永远都不是你想要的行为, 因为意外结束的最初原因可能对程序的行为来说会显得更重要。对于那些在 try 语句块中执行 break、continue 或 return 语句,只是为了使其行为被 finally 语句块所否决掉的程序,要理解其行为是特别困难的。总之,每一个 finally 语句块都应该正常结束,除非抛出的是不受检查的异常。 千万不要用一个 return、break、continue 或 throw 来退出一个 finally 语句块,并且千万不要允许将一个受检查的异常传播到一个 finally 语句块之外去。对于语言设计者, 也许应该要求 finally 语句块在未出现不受检查的异常时必须正常结束。朝着这个目标,try-finally 结构将要求 finally 语句块可以正常结束。return、break 或 continue 语句把控制权传递到 finally 语句块之外应该是被禁止的, 任何可以引发将被检查异常传播到 finally 语句块之外的语句也同样应该是被禁止的。

 
<a name="anchor2"></a>
# 谜题2: 极端不可思议

下面的三个程序每一个都会打印些什么? 不要假设它们都可以通过编译。

**第一个程序**

    import java.io.IOException;

    public class Arcane1 {

        public static void main(String[] args) {
            try {
                System.out.println("Hello world");
            } catch(IOException e) {
                System.out.println("I've never seen println fail!");
            }
        }
    }

**第二个程序**

    public class Arcane2 {
        public static void main(String[] args) {
            try {
                // If you have nothing nice to say, say nothing
            } catch(Exception e) {
                System.out.println("This can't happen");
            }
        }
    }

**第三个程序**

    interface Type1 {
        void f() throws CloneNotSupportedException;
    }

    interface Type2 {
        void f() throws InterruptedException;
    }

    interface Type3 extends Type1, Type2 {
    }

    public class Arcane3 implements Type3 {
        public void f() {
            System.out.println("Hello world");
        }
        public static void main(String[] args) {
            Type3 t3 = new Arcane3();
            t3.f();
        }
    }

运行结果：

(01) 第一个程序编译出错！

    Arcane1.java:9: exception java.io.IOException is never thrown in body of corresponding try statement
            } catch(IOException e) {
          ^
    1 error

(02) 第二个程序能正常编译和运行。

(03) 第三个程序能正常编译和运行。输出结果是: Hello world

结果说明：

(01) Arcane1展示了被检查异常的一个基本原则。它看起来应该是可以编译的:try 子句执行 I/O,并且 catch 子句捕获 IOException 异常。但是这个程序不能编译,因为 println 方法没有声明会抛出任何被检查异常,而IOException 却正是一个被检查异常。语言规范中描述道:如果一个 catch 子句要捕获一个类型为 E 的被检查异常, 而其相对应的 try 子句不能抛出 E 的某种子类型的异常,那么这就是一个编译期错误。

(02) 基于同样的理由,第二个程序,Arcane2,看起来应该是不可以编译的,但是它却可以。它之所以可以编译,是因为它唯一的 catch 子句检查了 Exception。尽管在这一点上十分含混不清,但是捕获 Exception 或 Throwble 的 catch 子句是合法的,不管与其相对应的 try 子句的内容为何。尽管 Arcane2 是一个合法的程序,但是 catch 子句的内容永远的不会被执行,这个程序什么都不会打印。

(03) 第三个程序,Arcane3,看起来它也不能编译。方法 f 在 Type1 接口中声明要抛出被检查异常 CloneNotSupportedException,并且在 Type2 接口中声明要抛出被检查异常 InterruptedException。Type3 接口继承了 Type1 和 Type2,因此, 看起来在静态类型为 Type3 的对象上调用方法 f 时, 有潜在可能会抛出这些异常。一个方法必须要么捕获其方法体可以抛出的所有被检查异常, 要么声明它将抛出这些异常。Arcane3 的 main 方法在静态类型为 Type3 的对象上调用了方法 f,但它对 CloneNotSupportedException 和 InterruptedExceptioin 并没有作这些处理。那么,为什么这个程序可以编译呢?

上述分析的缺陷在于对“Type3.f 可以抛出在 Type1.f 上声明的异常和在 Type2.f 上声明的异常”所做的假设。这并不正确,因为每一个接口都限制了方法 f 可以抛出的被检查异常集合。一个方法可以抛出的被检查异常集合是它所适用的所有类型声明要抛出的被检查异常集合的交集,而不是合集。因此,静态类型为 Type3 的对象上的 f 方法根本就不能抛出任何被检查异常。因此,Arcane3可以毫无错误地通过编译,并且打印 Hello world。

 
<a name="anchor3"></a>
# 谜题3: 不受欢迎的宾客

下面的程序会打印出什么呢?

    public class UnwelcomeGuest {
        public static final long GUEST_USER_ID = -1;
        private static final long USER_ID;

        static {
            try {
                USER_ID = getUserIdFromEnvironment();
            } catch (IdUnavailableException e) {
                USER_ID = GUEST_USER_ID;
                System.out.println("Logging in as guest");
            }
        }

        private static long getUserIdFromEnvironment() 
            throws IdUnavailableException {
            throw new IdUnavailableException();
        }

        public static void main(String[] args) {
            System.out.println("User ID: " + USER_ID);
        }
    }

    class IdUnavailableException extends Exception {
    }

运行结果：

    UnwelcomeGuest.java:10: variable USER_ID might already have been assigned
                USER_ID = GUEST_USER_ID;
                ^
    1 error

结果说明：

该程序看起来很直观。对 getUserIdFromEnvironment 的调用将抛出一个异常, 从而使程序将 GUEST_USER_ID(-1L)赋值给 USER_ID, 并打印 Loggin in as guest。 然后 main 方法执行,使程序打印 User ID: -1。表象再次欺骗了我们,该程序并不能编译。如果你尝试着去编译它, 你将看到和一条错误信息。

问题出在哪里了?USER_ID 域是一个空 final(blank final),它是一个在声明中没有进行初始化操作的 final 域。很明显,只有在对 USER_ID 赋值失败时,才会在 try 语句块中抛出异常,因此,在 catch 语句块中赋值是相 当安全的。不管怎样执行静态初始化操作语句块,只会对 USER_ID 赋值一次,这正是空 final 所要求的。为什么编译器不知道这些呢? 要确定一个程序是否可以不止一次地对一个空 final 进行赋值是一个很困难的问题。事实上,这是不可能的。这等价于经典的停机问题,它通常被认为是不可能解决的。为了能够编写出一个编译器,语言规范在这一点上采用了保守的方式。在程序中,一个空 final 域只有在它是明确未赋过值的地方才可以被赋值。规范长篇大论,对此术语提供了一个准确的但保守的定义。 因为它是保守的,所以编译器必须拒绝某些可以证明是安全的程序。这个谜题就展示了这样的一个程序。幸运的是, 你不必为了编写 Java 程序而去学习那些骇人的用于明确赋值的细节。通常明确赋值规则不会有任何妨碍。如果碰巧你编写了一个真的可能会对一个空final 赋值超过一次的程序,编译器会帮你指出的。只有在极少的情况下,就像本谜题一样, 你才会编写出一个安全的程序, 但是它并不满足规范的形式化要求。编译器的抱怨就好像是你编写了一个不安全的程序一样,而且你必须修改你的程序以满足它。

解决这类问题的最好方式就是将这个烦人的域从空 final 类型改变为普通的final 类型,用一个静态域的初始化操作替换掉静态的初始化语句块。实现这一点的最佳方式是重构静态语句块中的代码为一个助手方法:

    public class UnwelcomeGuest {
        public static final long GUEST_USER_ID = -1;
        private static final long USER_ID = getUserIdOrGuest();
        private static long getUserIdOrGuest() {
            try {
                return getUserIdFromEnvironment();
            } catch (IdUnavailableException e) {
                System.out.println("Logging in as guest");
                return GUEST_USER_ID;
            }
        }

        private static long getUserIdFromEnvironment() 
            throws IdUnavailableException {
            throw new IdUnavailableException();
        }

        public static void main(String[] args) {
            System.out.println("User ID: " + USER_ID);
        }
    }

    class IdUnavailableException extends Exception {
    }

程序的这个版本很显然是正确的,而且比最初的版本根据可读性,因为它为了域值的计算而增加了一个描述性的名字, 而最初的版本只有一个匿名的静态初始化操作语句块。将这样的修改作用于程序,它就可以如我们的期望来运行了。总之,大多数程序员都不需要学习明确赋值规则的细节。该规则的作为通常都是正确的。如果你必须重构一个程序,以消除由明确赋值规则所引发的错误,那么你应该考虑添加一个新方法。这样做除了可以解决明确赋值问题,还可以使程序的可读性提高。

 
<a name="anchor4"></a>
# 谜题4: 您好,再见!

下面的程序将会打印出什么呢?

    public class HelloGoodbye {
        public static void main(String[] args) {
            try {
                System.out.println("Hello world");
                System.exit(0);
            } finally {
                System.out.println("Goodbye world");
            }
        }
    }

运行结果:

    Hello world

结果说明：

这个程序包含两个 println 语句: 一个在 try 语句块中, 另一个在相应的 finally语句块中。try 语句块执行它的 println 语句,并且通过调用 System.exit 来提前结束执行。在此时,你可能希望控制权会转交给 finally 语句块。然而,如果你运行该程序,就会发现它永远不会说再见:它只打印了 Hello world。这是否违背了"Indecisive示例" 中所解释的原则呢? 不论 try 语句块的执行是正常地还是意外地结束, finally 语句块确实都会执行。然而在这个程序中,try 语句块根本就没有结束其执行过程。System.exit 方法将停止当前线程和所有其他当场死亡的线程。finally 子句的出现并不能给予线程继续去执行的特殊权限。

当 System.exit 被调用时,虚拟机在关闭前要执行两项清理工作。首先,它执行所有的关闭挂钩操作,这些挂钩已经注册到了 Runtime.addShutdownHook 上。这对于释放 VM 之外的资源将很有帮助。务必要为那些必须在 VM 退出之前发生的行为关闭挂钩。下面的程序版本示范了这种技术,它可以如我们所期望地打印出 Hello world 和 Goodbye world:

    public class HelloGoodbye1 {
        public static void main(String[] args) {
            System.out.println("Hello world");
            Runtime.getRuntime().addShutdownHook(
            new Thread() {
                public void run() {
                System.out.println("Goodbye world");
                }
            });
            System.exit(0);
        }
    }

VM 执行在 System.exit 被调用时执行的第二个清理任务与终结器有关。如果System.runFinalizerOnExit 或它的魔鬼双胞胎 Runtime.runFinalizersOnExit被调用了,那么 VM 将在所有还未终结的对象上面调用终结器。这些方法很久以前就已经过时了,而且其原因也很合理。无论什么原因,永远不要调用System.runFinalizersOnExit 和 Runtime.runFinalizersOnExit: 它们属于 Java类库中最危险的方法之一[ThreadStop]。调用这些方法导致的结果是,终结器会在那些其他线程正在并发操作的对象上面运行, 从而导致不确定的行为或导致死锁。

总之,System.exit 将立即停止所有的程序线程,它并不会使 finally 语句块得到调用,但是它在停止 VM 之前会执行关闭挂钩操作。当 VM 被关闭时,请使用关闭挂钩来终止外部资源。通过调用 System.halt 可以在不执行关闭挂钩的情况下停止 VM,但是这个方法很少使用。

 
<a name="anchor5"></a>
# 谜题5: 不情愿的构造器

下面的程序将打印出什么呢?

    public class Reluctant {
        private Reluctant internalInstance = new Reluctant();
        public Reluctant() throws Exception {
            throw new Exception("I'm not coming out");
        }
        public static void main(String[] args) {
            try {
                Reluctant b = new Reluctant();
                System.out.println("Surprise!");
            } catch (Exception ex) {
                System.out.println("I told you so");
            }
        }
    }

运行结果：

    Exception in thread "main" java.lang.StackOverflowError
        at Reluctant.<init>(Reluctant.java:3)
        ...

结果说明：

main 方法调用了 Reluctant 构造器,它将抛出一个异常。你可能期望 catch 子句能够捕获这个异常,并且打印 I told you so。凑近仔细看看这个程序就会发现,Reluctant 实例还包含第二个内部实例,它的构造器也会抛出一个异常。无论抛出哪一个异常,看起来 main 中的 catch 子句都应该捕获它,因此预测该程序将打印 I told you 应该是一个安全的赌注。但是当你尝试着去运行它时,就会发现它压根没有去做这类的事情:它抛出了 StackOverflowError 异常,为什么呢?

与大多数抛出 StackOverflowError 异常的程序一样,本程序也包含了一个无限递归。当你调用一个构造器时,实例变量的初始化操作将先于构造器的程序体而运行[JLS 12.5]。在本谜题中, internalInstance 变量的初始化操作递归调用了构造器,而该构造器通过再次调用 Reluctant 构造器而初始化该变量自己的 internalInstance 域,如此无限递归下去。这些递归调用在构造器程序体获得执行机会之前就会抛出 StackOverflowError 异常,因为 StackOverflowError 是 Error 的子类型而不是 Exception 的子类型,所以 catch 子句无法捕获它。对于一个对象包含与它自己类型相同的实例的情况,并不少见。例如,链接列表节点、树节点和图节点都属于这种情况。你必须非常小心地初始化这样的包含实例,以避免 StackOverflowError 异常。

至于本谜题名义上的题目:声明将抛出异常的构造器,你需要注意,构造器必须声明其实例初始化操作会抛出的所有被检查异常。

 
<a name="anchor6"></a>
# 谜题6: 域和流

下面的方法将一个文件拷贝到另一个文件,并且被设计为要关闭它所创建的每一个流,即使它碰到 I/O 错误也要如此。遗憾的是,它并非总是能够做到这一点。为什么不能呢,你如何才能订正它呢?

    static void copy(String src, String dest) throws IOException {
        InputStream in = null;
        OutputStream out = null;
        try {
            in = new FileInputStream(src);
            out = new FileOutputStream(dest);
            byte[] buf = new byte[1024];
            int n;
            while ((n = in.read(buf)) > 0)
                out.write(buf, 0, n);
        } finally {
            if (in != null) in.close();
            if (out != null) out.close();
        }
    }

谜题分析：

这个程序看起来已经面面俱到了。其流域(in 和 out)被初始化为 null,并且新的流一旦被创建,它们马上就被设置为这些流域的新值。对于这些域所引用的流,如果不为空,则 finally 语句块会将其关闭。即便在拷贝操作引发了一个 IOException 的情况下,finally 语句块也会在方法返回之前执行。出什么错了呢?

问题在 finally 语句块自身中。close 方法也可能会抛出 IOException 异常。如果这正好发生在 in.close 被调用之时,那么这个异常就会阻止 out.close 被调用,从而使输出流仍保持在开放状态。请注意,该程序违反了"优柔寡断" 的建议:对 close 的调用可能会导致 finally 语句块意外结束。遗憾的是,编译器并不能帮助你发现此问题,因为 close 方法抛出的异常与 read 和 write 抛出的异常类型相同,而其外围方法(copy)声明将传播该异常。解决方式是将每一个 close 都包装在一个嵌套的 try 语句块中。

下面的 finally 语句块的版本可以保证在两个流上都会调用 close:

    try {
        // 和之前一样
    } finally {
        if (in != null) {
            try {
                in.close();
            } catch (IOException ex) {
                // There is nothing we can do if close fails
            }
        }

        if (out != null) {
            try {
                out.close();
            } catch (IOException ex) {
                // There is nothing we can do if close fails
            }
        }
    }

总之,当你在 finally 语句块中调用 close 方法时,要用一个嵌套的 try-catch 语句来保护它,以防止 IOException 的传播。更一般地讲,对于任何在 finally 语句块中可能会抛出的被检查异常都要进行处理,而不是任其传播。

 
<a name="anchor7"></a>
# 谜题7: 异常为循环而抛

下面的程序会打印出什么呢?

    public class Loop {
        public static void main(String[] args) {
            int[][] tests = { { 6, 5, 4, 3, 2, 1 }, { 1, 2 },
                { 1, 2, 3 }, { 1, 2, 3, 4 }, { 1 } };
            int successCount = 0;
            try {
                int i = 0;
                while (true) {
                    if (thirdElementIsThree(tests[i++]))
                        successCount ++;
                }
            } catch(ArrayIndexOutOfBoundsException e) {
                // No more tests to process
            }
            System.out.println(successCount);
        }
        private static boolean thirdElementIsThree(int[] a) {
            return a.length >= 3 & a[2] == 3;
        }
    }

运行结果：

    0

结果说明：

该程序主要说明了两个问题。

**第1个问题：不应该使用异常作为终止循环的手段！**

该程序用 thirdElementIsThree 方法测试了 tests 数组中的每一个元素。遍历这个数组的循环显然是非传统的循环:它不是在循环变量等于数组长度的时候终止,而是在它试图访问一个并不在数组中的元素时终止。尽管它是非传统的,但是这个循环应该可以工作。

如果传递给 thirdElementIsThree 的参数具有 3 个或更多的元素,并且其第三个元素等于 3,那么该方法将返回 true。对于 tests中的 5 个元素来说,有 2 个将返回 true,因此看起来该程序应该打印 2。如果你运行它,就会发现它打印的时 0。肯定是哪里出了问题,你能确定吗? 事实上,这个程序犯了两个错误。第一个错误是该程序使用了一种可怕的循环惯用法,该惯用法依赖的是对数组的访问会抛出异常。这种惯用法不仅难以阅读, 而且运行速度还非常地慢。不要使用异常来进行循环控制;应该只为异常条件而使用异常。为了纠正这个错误,可以将整个 try-finally 语句块替换为循环遍历数组的标准惯用法:

    for (int i = 0; i < test.length; i++)
        if (thirdElementIsThree(tests[i]))
            successCount++;

如果你使用的是 5.0 或者是更新的版本,那么你可以用 for 循环结构来代替:

    for (int[] test : tests)
        if(thirdElementIsThree(test))
            successCount++;

 

**第2个问题: 主要比较"&操作符" 和 "&&操作符"的区别。**注意示例中的操作符是&，这是按位进行"与"操作。

 
