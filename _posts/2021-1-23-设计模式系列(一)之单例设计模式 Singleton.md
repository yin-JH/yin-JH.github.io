---
layout: post
title: 设计模式系列(一)之单例设计模式
categories: Design_Pattern
desctiption: 单例设计模式的讲解
keywords: Design_Pattern, Singleton
---

设计模式之单例设计模式，本篇探讨单例设计模式的特点、代码、优劣及适用场景
======

## 设计模式之单例设计模式

### 单例设计模式介绍

- 单例设计模式英文名为Singleton，一般整个程序的生命周期中，某一个对象在任何时刻最多只需要一个时，这个类就可以被设计成单例，例如某种工厂类等
- 单例设计模式一共有8种广为人知的写法

### 单例设计模式解析

#### 简单饿汉式

```java
public class Singleton01{
    public static final Singleton01 INSTANCE = new Singleton01();
    
    private Singleton01(){
        
    }
    
    public static Singleton01 getInstance(){
        return INSTANCE;
    }
    
    public void m(){
        System.out.println("Singleton01 is running!")
    }
}
```

- 饿汉式的特点
  1. 拥有一个静态、final的本身实例
  2. 构造方法时私有的，外部类无法再次构建
  3. 每次要调用实例对象的时候，直接使用Singleton01.getInstance();
- 饿汉式的优点
  1. 代码量少，非常简单
  2. 效率高
  3. 不会出现线程安全问题
- 饿汉式缺点
  1. 当类被加载的时候，这个对象就生成了，也许生成了就不会很快被使用，某种程度上来说占用了空间
- 总结
  1. 非常优秀的写法，非常推荐

#### 简单饿汉式的变种写法

```java
public class Singleton02{
    public static Singleton01 INSTANCE;
    
    static{
        INSTANCE = new Singleton02();
    }
    
    private Singleton02(){
        
    }
    
    public static Singleton02 getInstance(){
        return INSTANCE;
    }
    
    public void m(){
        System.out.println("Singleton02 is running!")
    }
}
```

- 这仅仅只是饿汉式的另外写法，本质上和简单饿汉式没有区别

#### 懒汉式

```java
public class Singleton03{
    private static Singleton03 INSTANCE;
    
    private Singleton03(){
        
    }
    
    public static Singleton03 getInstance(){
        if(INSTANCE == null){
            INSTANCE = new Singleton03();
        }
        
        return INSTANCE;
    }
    
    public void m(){
        System.out.println("Singleton03 is running!");
    }
}
```

- 懒汉式特点
  1. 懒汉式也需要设置私有静态的INSTANCE和私有的构造方法
  2. 懒汉式中的静态方法getInstance()方法在调用的时候首先检查INSTANCE是否为空，为空就创建

- 懒汉式优点：
  1. 可以实现不调用就不会实例化
- 懒汉式缺点：
  1. 带来线程不安全
- 总结：
  1. 不推荐使用

#### 解决懒汉式线程不安全的写法

```java
public class Singleton04 {
    private static Singleton04 INSTANCE;

    private Singleton04() {

    }

    public static Singleton04 getInstance() {
        synchronized (Singleton04.class) {
            /*
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            */
            if (INSTANCE == null) {
                INSTANCE = new Singleton04();
            }
            return INSTANCE;
        }
    }

    public void m() {
        System.out.println("running!");
    }
}
```

- 解决懒汉式线程不安全的特点
  1. 这里是将getInstance()方法添加锁，这样就可以保证线程安全，同时保留了懒汉式的优点
- 解决懒汉式线程不安全的优点：
  1. 可以到使用的时候再实例化；线程安全
- 解决懒汉式线程不安全的缺点：
  1. 每一个使用getInstance()的线程都需要申请锁，大大影响效率
- 总结：
  1. 不推荐使用

#### 解决懒汉式线程不安全的另外一种写法

```java
public class Singleton05 {
    private static volatile Singleton05 INSTANCE;

    private Singleton05() {

    }

    public static synchronized Singleton05 getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Singleton05();
        }
        return INSTANCE;
    }

    public void m() {
        System.out.println("running!");
    }
}
```

这里的写法和上面一种写法没有任何区别，只不过一个的锁是写在方法中，这个写在方法上

#### 双重检查（Double Check）解决加上锁后效率降低的问题

```java
public class Singleton06 {
    private static volatile Singleton06 INSTANCE;

    private Singleton06() {

    }

    public static Singleton06 getInstance() {
        if (INSTANCE == null) {
            /*
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            */
            synchronized (Singleton06.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton06();
                }
            }
        }
        return INSTANCE;
    }

    public void m() {
        System.out.println("running!");
    }
}
```

- 双重检查特点：
  1. 在上一个的基础上加上一层 INSTANCE == null 的检测，如果检测通过，就申请锁，然后再检测一次，通过才可以创建实例
- 双重检查优点：
  1. 加载的时候不会实例化，只有第一次调用的时候才会实例化；保证线程安全；效率比起上面加锁的方法高出不少
- 总结：
  1. 这种写法是一种完美的方法之一，在以前很长的一段时间内都是公认完美无缺的写法

#### 静态内部类的单例写法

```java
public class Singleton07 {

    private Singleton07() {

    }

    private static class InstanceHolder{
        private static final Singleton07 INSTANCE = new Singleton07();
    }

    public static Singleton07 getInstance() {
        return InstanceHolder.INSTANCE;
    }

    public void m() {
        System.out.println("running!");
    }
}
```

- 静态内部类的单例特点
  1. 这里不需要写私有静态的INSTANCE实例，这个INSTANCE实例交给匿名静态内部类 InstanceHolder ，作为这个内部类的静态实例变量，在静态内部类加载的时候才会被创建
  2. getInstance()方法返回的就是 InstanceHolder.INSTANCE 
- 静态内部类的单例优点：
  1. 保留了懒汉式中不调用就不会实例化的优点；线程安全；因为少了加锁，效率比双重检测还要高
- 总结：
  1. 完美的写法之一，很完美，推荐使用

#### 枚举单例

```java
public enum Singleton08 {
    INSTANCE;

    public void m() {
        System.out.println("running!");
    }
}
```

- 枚举单例特点：
  1. 将类设置为枚举类，里面放置一个枚举实例 INSTANCE 
  2. 想要调用单例对象的时候直接 枚举类名.INSTANCE 就可以得到实例
- 枚举单例优点：
  1. 写法简单；线程安全；效率高；可以防止反序列化；
- 总结：
  1. Effective Java作者提出的一种写法，完美中的完美，推荐使用

### 单例设计模式的探究

1. 单例设计模式是用于设计一些整个程序中只需要一个实例的类的
2. 单例设计模式是一种非常常见的设计模式
3. 一般来说，最基础的饿汉式单例模式就已经足够使用了，因为这种写法足够简单，并且还可以保证线程安全
