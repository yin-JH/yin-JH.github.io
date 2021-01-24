---
layout: post
title: 设计模式系列之(一)单例设计模式
categories: Design_Pattern
desctiption: 单例设计模式的讲解
keywords: Design_Pattern, Singleton
---

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

