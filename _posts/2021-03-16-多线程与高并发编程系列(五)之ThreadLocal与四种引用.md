---
layout: post
title: 2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用
categories: JUC
description: 本篇介绍多线程中的ThreadLocal以及Java中的四种引用
keywords: JUC
---

本篇介绍多线程中的ThreadLocal以及Java中的四种引用------强软弱虚======

### ThreadLocal

#### ThreadLocal简单介绍

- ThreadLocal的用用处是为了实现线程之间隔离，ThreadLocal中存储的数据专属于当前线程，在多线程场景下可以防止其他线程篡改变量

#### ThreadLocal使用场景

- Sping中采用了ThreadLocal的方法，用于保证单个线程的数据库操作都是同一个connection对象

#### ThreadLocal使用

```java
public class ThreadLocal_Test {
    static ThreadLocal<Person> tl = new ThreadLocal<>();

    public static void main(String[] args) {

        new Thread(()->{
            Person person = new Person();
            person.name = "yin";

            tl.set(person);

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("t1 " + tl.get().name);
        }).start();

        new Thread(()->{
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            tl.set(new Person());
            System.out.println("t2 " + tl.get().name);
        }).start();
    }

    static class Person {
        String name = "zhangsan";
    }
}
```

- 我们可以发现，虽然是同一个ThreadLocal对象，也是调用的同一个方法 get ，但是得到的结果却不一样

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-1.png)

- 但是，值得注意的是，当我们使用完毕ThreadLocal的时候，我们最好就将它remove掉

#### ThreadLocal的set()源码阅读

- 我们跟着set的运行进入阅读

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-2.png)

- 这个方法非常简单，我们可以看到，首先是获取当前线程，然后得到当前线程里面的一个对象，这个对象叫做ThreadLocalMap，如果这个map为空，那么就创建一个map，然后将this对象，也就是 ThreadLocal 对象作为key，想要set的value，这里就是Person对象作为value

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-3.png)

- 总结

  1. ThreadLocal方法set的val全部都存储在了当前线程的内存的内部，而不是存储在主内存中

#### ThreadLocalMap源码阅读

- 我们首先对ThreadLocalMap的大致结构进行阅读，ThreadLocal内有一个自定义的 Entry ，这个Entry的用处就是用来放置ThreadLocal set 的对象

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-4.png)

- 我们来看ThreadLocalMap的构造方法，首先，我们会创建一个table，这个table的初始长度为16，然后我们会将作为key的ThreadLocal对象拿来取一个hash值，之后将这个hash值和15进行一个与操作，最后得到的一个值作为table的index

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-5.png)

- 了解了ThreadLocalMap的构造方法，我们再来了解一下ThreadLocalMap的set方法，这个ThreadLocalMap的核心;首先会初始化一些基本数据，并且利用key的hashcode来得到一个初始的 index  i

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-6.png)

- 接下来会进入一个循环，这个循环是set的核心内容之一，这个循环的初始i是我们一开始通过 key 得到的 index 然后我们会将这个index位置上的 Entry 拿出来，如果这个Entry 不存在，也就是说 table[i] 位置上是一个空位置，那么直接退出for循环；或者这个位置上有 Entry ，但是这个 Entry 的key 和我们将要写入的key一致，那么我们就会将要写入的key的value将原本的value给直接覆盖一遍；或者这个 Entry 存在，但是这个 Entry 的key值为空（这种情况一般是一开始还存在一个ThreadLocal对象，但是这个ThreadLocal对象被GC回收了，于是就出现了key为空的情况）那么就会将这个位置给覆盖一遍。我们退出这个for循环的时候，第二第三种情况直接就return了，第一种情况会添加一个 Entry 到 table 中，这个时候会进行后续操作

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-7.png)

- 后续操作非常简单，会将table的size++，第一件事情是先将当前 tab[i] 位置的后面的 几个位置进行检测，检测的数量取决于table len 的长度，如果len为16，那么就会检测5个插槽，如果为32就会检测6个插槽，以此类推。检测插槽的目的是检查这些插槽是否已经被使用了（位置上存在 Entry），但是这些插槽的key是null，将会将这些插槽给清空，然后将size给减少；如果插槽检查发现检查的插槽都是健康的，那么就会进行判断sz >= threshold，顾名思义，就是对size进行检测，如果达到了临界点的话就会进行扩容，这个threshold（临界点）的值是最大length的2/3。

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-8.png)

- 扩容策略的第一步是进行插槽检查，将所有的插槽都检查一遍，在所有插槽检查一遍后如果检测后的size仍然大于3/4临界点，也就是大于最大size容量的一半就会启动扩容方法，也就是resize()

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-9.png)

- 扩容的思路很简单，就是将最大容量扩大为原来的2倍，然后再将原本index中的所有Entry的key值全部重新拿出来进行一次 hash & (len - 1) 操作得到一个新的index然后将之放进去

  ![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-10.png)

- 总结

  1. ThreadLocalMap和HashMap有一定的相似度，都有扩容的操作
  2. ThreadLocalMap中非常容易出现不健康的插槽，也就是 Entry 存在，但是 Eentry 的key为空，这种情况是因为ThreadLocal被GC给回收了，徒留下一个value，如果不处理的话就会导致内存泄漏，所以ThreadLocalMap中到处存在检测不健康插槽并且将之清理的操作
  3. 每一个线程的ThreadLocalMap的table初始容量为16，在达到容量上限的2/3时会开启扩容操作，在真正扩容之间会对所有的插槽进行检测，如果所有不健康插槽都被清理完毕之后当前size仍然大于最大容量的一半的时候，就会真正的扩容，将最大容量扩大为原来的一倍

### Java中的四种引用之强引用

#### 强引用简介

- 强引用是我们最最常使用的引用，我们平时new出来的对象的栈空间指向堆内存的引用就是强引用

#### 强引用特点

1. 堆中的对象如果被一个强引用指向，那么GC将不会回收这个对象
2. 当堆中的空间满的时候，GC也不会回收强引用指向的对象，哪怕抛出OOM异常
3. 被强引用指向的对象，只有当引用消失的时候才会被GC回收

#### 强引用实验

```java
public class M {
    @Override
    protected void finalize() throws Throwable {
        System.out.println("对象被回收");
    }
}

public class NormalReference {
    public static void main(String[] args) throws InterruptedException {

        M m = new M();

        for (int i = 0; i < 10; i++) {
            TimeUnit.SECONDS.sleep(1);
            System.gc();
            System.out.println(i);
            /*if(i == 4){
                m = null;
            }*/
        }
    }
}
```

![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-11.png)

- 我们可以发现，哪怕我们每隔1秒调用一次gc，但是因为强引用仍然指向了对象，所以这个对象没有被回收

```java
public class NormalReference {
    public static void main(String[] args) throws InterruptedException {

        M m = new M();

        for (int i = 0; i < 10; i++) {
            TimeUnit.SECONDS.sleep(1);
            System.gc();
            System.out.println(i);
            if(i == 4){
                m = null;
            }
        }
    }
}
```

![image](\images\posts\JUC\2021-03-16-多线程与高并发编程系列(五)之ThreadLocal与四种引用-12.png)

- 我们可以发现，当这个对象没有被强引用指向的时候，它在调用gc后被回收了

### Java中的四种引用之软引用

#### 软引用简介

- 软引用一般是用来指向一些有用处但是并不是必需的对象，使用的是SoftReference
- 一般使用SoftReference的地方可以是一些缓存，缓存就是典型的用处有，但是重要性不明显的数据，可有可无

#### 软引用特点

1. 被软引用指向的对象 有可能 被JVM直接回收
2. 当JVM被分配的内存使用完毕的时候，gc会直接回收所有的软引用对象

### Java中的四种引用之弱引用

#### 弱引用简介

- 弱引用指向的对象，如果没有被其他强引用指向的话，一旦被GC发现就会被回收，弱引用使用的是WeakReference
- 在ThreadLocalMap中的Entry也是使用的弱引用，如果不是使用的弱引用，而是使用普通的引用，那么当某一个线程被回收的时候，因为Entry是使用的强引用，那么这个Entry就不会被回收，导致内存泄漏

#### 弱引用特点

1. 一旦被GC发现就会被直接回收

### Java中的四种引用之虚引用

#### 虚引用简介

- 虚引用的存在完全不会影响一个对象的生存周期，虚引用的用处在于管理堆外内存。当一个虚引用指向的对象被回收的时候，jvm会将这个信息放到ReferenceQueue里面，当然，我们自己是无法get到的，但是写虚拟机的人是有方法手段get到的，然后这些人就可以用free(C)或者delete(C++)的方法将这个内存给释放
- 也就是说，虚引用是给写虚拟机的人使用的，我们普通程序员一般用不到

#### 虚引用特点

1. 虚引用指向的对象可以是堆内对象，也可以是堆外对象
2. 虚引用的用处是作为一个提示，提示虚拟机作者堆外有个对象要被回收了，这个时候虚拟机作者就可以调用写虚拟机的方法，可能是free，可能是delete来回收

