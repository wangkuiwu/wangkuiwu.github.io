---
layout: post
title: "设计模式09之 合成(Composite)模式(结构模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-19 09:01
---
 
> 本章介绍"合成模式"。

> **目录**  
[1. 合成模式简介](#anchor1)  
[2. 安全合成模式](#anchor2)  
[3. 透明合成模式](#anchor3)  
[4. 合成模式示例](#anchor4)  


 
<a name="anchor1"></a>
# 1. 合成模式简介

合成模式属于对象的结构型模式，有时又被称为"部分-整体模式"。合成模式将对象组织到树结构中，可以用来描述整体与部分的关系。

例如，文件系统就是一个典型的合成模式系统。合成模式的树形图如下所示：

![img](/media/pic/design_patterns/pattern09_01.jpg)

它的UML类图如下所示：

![img](/media/pic/design_patterns/pattern09_02.jpg)

它共包括3个角色：**抽象构件(Component)**，**树叶构件(Leaf)** 和 **树枝构件(Composite)**。

|     角色   |       说明      |
| ---------- | --------------- |
|   抽象构件 | 这是一个抽象角色，它给参加组合的对象规定一个接口。这个角色给出共有的接口及其默认行为。 |
|   树叶构件 | 代表参加组合的树叶对象。树叶对象没有下级子对象，它定义出参加组合的原始对象的行为。 |
|   树枝构件 | 代表参加组合的有下级子对象的对象，并给出树枝构件对象的行为。 |

根据"参加组合的对象的管理方式"，可以将合成模式分为两种不同的形式：**安全合成模式** 和 **透明合成模式**。  
• 安全合成模式：此方式只允许"树枝构件"有对象的管理方法。  
• 透明合成模式：此方式只允许"树枝构件'和"树叶构件"都有对象的管理方法，但"树叶构件"中的管理方法无实际意义。  

 
<a name="anchor2"></a>
# 2. 安全合成模式

安全合成模式要求管理聚集的方法只出现在树枝构件类中，而不出现在树叶构件类中。

安全合成模式的UML图如下：

![img](/media/pic/design_patterns/pattern09_03.jpg)

它共包括3个角色：**抽象构件(Component)，树叶构件(Leaf) 和 树枝构件(Composite)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象构件 | 这是一个抽象角色，它给参加组合的对象定义出公共的接口及其默认行为，可以用来管理所有的子对象。合成对象通常把它所包含的子对象当做类型为Component的对象。在安全式的合成模式里，构件角色并不定义出管理子对象的方法，这一定义由树枝构件对象给出。 |
| 树叶构件 | 树叶对象是没有下级子对象的对象，定义出参加组合的原始对象的行为。 |
| 树枝构件 | 代表参加组合的有下级子对象的对象。树枝构件类给出所有的管理子对象的方法，如add()、remove()以及getChild()。 |


示例代码

    public interface Component {
        // 返还自己的示例
        Composite getComposite();
        // 某个商业方法
        void sampleOperation();
    }

    public class Composite implements Component {
        private Vector componentVector = new Vector();
        public Composite getComposite() {
            return this;
        }
        public void sampleOperation() {
            Enumeration enum = composites();
            while (enum.hasMoreElements()) {
                ((Composite)enum.nextElement()).sampleOperation();
            }
        }
        // 添加一个子构件对象
        public void add(Component component) {
            componentVector.addElement(component);
        }
        // 删除一个子构件对象
        public void remove(Component component) {
            componentVector.removeElement(component);
        }
        // 聚集管理办法，返还聚集的Enumeration对象
        public Enumeration components() {
            return componentVector.elements();
        }
    }

    public class Leaf implements Component {
        public void sampleOperation() {
        }
        public Composite getComposite() {
            return null;
        }
    }

从中可以看出，树叶构建(Leaf)中没有对节点的管理方法，而只有树枝构建(Composite)中有节点的管理方法。

 
<a name="anchor3"></a>
# 3. 透明合成模式

与安全式的合成模式不同的是，透明式的合成模式要求所有的具体构件类，不论树枝构件还是树叶构件，均符合一个固定接口。

透明合成模式的UML图如下：

![img](/media/pic/design_patterns/pattern09_04.jpg)

它共包括3个角色：**抽象构件(Component)，树叶构件(Leaf) 和 树枝构件(Composite)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象构件 | 这是一个抽象角色，它给参加组合的对象规定一个接口，规范共有的接口及默认行为。这个接口可以用来管理所有的子对象，要提供一个接口以规范取得和管理下层组件的接口，包括add(), remove()以及getChild()之类的方法。 |
| 树叶构件 | 代表参加组合的树叶对象，定义出参加组合的原始对象的行为。树叶类会给出add(), remove()以及getChild()之类的用来管理子类对象的方法的平庸实现。 |
| 树枝构件 | 代表参加组合的有子对象的对象，定义出这样的对象的行为。 |

示例代码

    public interface Component {
        // 某个商业方法
        void sampleOperation();
        // 返还自己的示例
        Composite getComposite();
        // 聚集管理方法，增加子构件对象
        void add(Component component);
        // 聚集管理方法，删除子构件对象
        void remove(Component component);
        // 聚集管理办法，返还聚集的Enumeration对象
        Enumeration components();
    }

    public class Composite implements Component {
        private Vector componentVector = new Vector();
        public Composite getComposite() {
            return this;
        }
        public void sampleOperation() {
            Enumeration enum = composites();
            while (enum.hasMoreElements()) {
                ((Composite)enum.nextElement()).sampleOperation();
            }
        }
        // 添加一个子构件对象
        public void add(Component component) {
            componentVector.addElement(component);
        }
        // 删除一个子构件对象
        public void remove(Component component) {
            componentVector.removeElement(component);
        }
        // 聚集管理办法，返还聚集的Enumeration对象
        public Enumeration components() {
            return componentVector.elements();
        }
    }

    public class Leaf implements Component {
        public void sampleOperation() {
        }
        public Composite getComposite() {
            return null;
        }
        // 添加一个子构件对象
        public void add(Component component) {
        }
        // 删除一个子构件对象
        public void remove(Component component) {
        }
        // 聚集管理办法，返还聚集的Enumeration对象
        public Enumeration components() {
            return null;
        }

    }　

 
<a name="anchor4"></a>
# 4. 合成模式示例

下面以一个逻辑树为例子，演示合成模式。

## 4.1 抽象构件(IFile.java)

    public interface IFile { 
        //返回自己的实例 
        IFile getComposite(); 

        //某个商业方法 
        void sampleOperation(); 

        //获取深度 
        int getDeep(); 

        //设置深度 
        void setDeep(int x); 
    }

## 4.2 树枝构件(Folder.java)

    import java.util.Vector; 

    public class Folder implements IFile { 
        private String name;    //文件名字 
        private int deep;       //层级深度，根深度为0 
        private Vector<IFile> componentVector = new Vector<IFile>(); 

        public Folder(String name) { 
            this.name = name; 
        } 

        //返回自己的实例 
        public IFile getComposite() { 
            return this; 
        } 

        //某个商业方法 
        public void sampleOperation() { 
            System.out.println("执行了某个商业方法！"); 
        } 

        //增加一个文件或文件夹 
        public void add(IFile IFile) { 
            componentVector.addElement(IFile); 
            IFile.setDeep(this.deep + 1); 
        } 

        //删除一个文件或文件夹 
        public void remove(IFile IFile) { 
            componentVector.removeElement(IFile); 
        } 

        //返回直接子文件（夹）集合 
        public Vector getAllComponent() { 
            return componentVector; 
        } 

        public String getName() { 
            return name; 
        } 

        public void setName(String name) { 
            this.name = name; 
        } 

        public int getDeep() { 
            return deep; 
        } 

        public void setDeep(int deep) { 
            this.deep = deep; 
        } 
    }

## 4.3 树枝构件(File.java)

    public class File implements IFile { 
        private String name;    //文件名字 
        private int deep;       //层级深度 

        public File(String name) { 
            this.name = name; 
        } 

        //返回自己的实例 
        public IFile getComposite() { 
            return this; 
        } 

        //某个商业方法 
        public void sampleOperation() { 
            System.out.println("执行了某个商业方法！"); 
        } 

        public String getName() { 
            return name; 
        } 

        public void setName(String name) { 
            this.name = name; 
        } 

        public int getDeep() { 
            return deep; 
        } 

        public void setDeep(int deep) { 
            this.deep = deep; 
        } 
    }

## 4.4 客户端测试程序(Client.java)

    import java.util.Iterator; 
    import java.util.Vector; 
    import java.util.Collections; 

    public class Client { 
        private static final String INDENT_CHAR = "\t";       //文件层次缩进字符 

        public static void main(String args[]) { 
            new Client().test(); 
        } 

        /** 
         * 客户端测试方法 
         */ 
        public void test() { 
            //根下文件及文件夹 
            Folder root = new Folder("树根"); 

            Folder b1_1 = new Folder("1_枝1"); 
            Folder b1_2 = new Folder("1_枝2"); 
            Folder b1_3 = new Folder("1_枝3"); 
            File l1_1 = new File("1_叶1"); 
            File l1_2 = new File("1_叶2"); 
            File l1_3 = new File("1_叶3"); 

            //b1_2下的文件及文件夹 
            Folder b2_1 = new Folder("2_枝1"); 
            Folder b2_2 = new Folder("2_枝2"); 
            File l2_1 = new File("2_叶1"); 

            //缔造树的层次关系（简单测试，没有重复添加的控制） 
            root.add(b1_1); 
            root.add(b1_2); 
            root.add(l1_1); 
            root.add(l1_2); 

            b1_2.add(b2_1); 
            b1_2.add(b2_2); 
            b1_2.add(l2_1); 
            root.add(l1_3); 
            root.add(b1_3); 
            //控制台打印树的层次 
            outTree(root); 
        } 

        public void outTree(Folder folder) { 
            System.out.println(folder.getName()); 
            iterateTree(folder); 
        } 

        /** 
         * 遍历文件夹，输入文件树 
         * 
         * @param folder 
         */ 
        public void iterateTree(Folder folder) { 
            Vector<IFile> clist = folder.getAllComponent(); 
            for (Iterator<IFile> it = clist.iterator(); it.hasNext();) { 
                IFile em = it.next(); 
                if (em instanceof Folder) { 
                    Folder cm = (Folder) em; 
                    System.out.println(getIndents(em.getDeep()) + cm.getName()); 
                    iterateTree(cm); 
                } else { 
                    System.out.println(getIndents(em.getDeep()) + ((File) em).getName()); 
                } 
            } 
        } 

        /** 
         * 文件层次缩进字符串 
         * 
         * @param x 缩进字符个数 
         * @return 缩进字符串 
         */ 
        public static String getIndents(int x) { 
            StringBuilder sb = new StringBuilder(); 
            for (int i = 0; i < x; i++) { 
                sb.append(INDENT_CHAR); 
            } 
            return sb.toString(); 
        } 
    }

 

运行结果：

    树根
        1_枝1
        1_枝2
            2_枝1
            2_枝2
            2_叶1
        1_叶1
        1_叶2
        1_叶3
        1_枝3


