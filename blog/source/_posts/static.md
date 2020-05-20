---
title: static
copyright: true
date: 2020-03-27 18:22:26
categories:
- Java
- base
tags:
---

# 修饰用法

## 基本用法

一般用来修饰成员变量或函数，可以在没有创建对象的情况下来进行调用。

## 修饰类

普通类是不允许声明为静态的，只有内部类才可以。

<!-- more -->

## 修饰函数/方法

修饰方法的时候，其实跟类一样，可以直接通过类名来进行调用。

## 修饰变量

被static修饰的成员变量叫做静态变量，也叫做类变量，说明这个变量是属于这个类的，而不是属于是对象，没有被static修饰的成员变量叫做实例变量，说明这个变量是属于某个具体的对象的。

## 修饰代码块

静态代码块在类第一次被载入时执行，类初始化的顺序如下：

父类静态变量、父类静态代码块、子类静态变量、子类静态代码块、父类普通变量、父类普通代码块、父类构造函数、子类普通变量、子类普通代码块、子类构造函数。

## 注意事项

1、静态方法只能访问静态成员。（非静态既可以访问静态，又可以访问非静态）

2、静态方法中不可以使用this或者super关键字。

3、主函数是静态的

# 静态方法访问限制

静态方法和静态变量是属于某一个类，而不属于类的对象。

### 结论

静态方法是属于类的，动态方法属于实例对象。静态成员在类“加载”（实际上是准备阶段）的时候就会分配内存，可以通过类名直接去访问，非静态成员（变量和方法）属于类的对象，所以只有该对象初始化之后才存在，然后通过类的对象去访问。

也就是说如果我们在静态方法中调用非静态成员变量会超前，可能会调用了一个还未初始化的变量。因此编译器会报错。

# 内部类实例化与static

## 非静态内部类

```java
public class StaticTest {
        class InnerClass{
            private int x = 10;
        }

    public static void main(String ...args){
        InnerClass xx = new StaticTest().new InnerClass();
      //InnerClass xx = new StaticTest.InnerClass();
        System.out.println("Hello :: "+xx.x);
    }
}
```

非静态内部类需要在`StaticTest`实例化获得对象之后，才被加载到堆内存中。

## 静态内部类

```java
public class StaticTest {
        static class InnerClass{
            private int x = 10;
        }

    public static void main(String ...args){
//      InnerClass xx = new StaticTest().new InnerClass();
        InnerClass xx = new InnerClass();
        System.out.println("Hello :: "+xx.x);
    }
}
```

运行`main`方法前，第一步`StaticTest`的加载，在加载过程中，已经扫描到静态内部类（类变量）。所以在`main`（静态）方法中，可以直接用`new`实例化`InnerClass`静态内部类。

## 结论

静态成员属于类，普通成员属于类的实例对象。