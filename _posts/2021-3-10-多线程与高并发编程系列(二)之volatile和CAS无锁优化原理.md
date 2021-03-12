---
layout: post
title: 多线程与高并发编程系列(二)之volatile和CAS无锁优化原理
categories: JUC
description: 介绍volatile和CAS无锁优化原理，对乐观锁和悲观锁进行解释
keywords: JUC
---

b本篇简介volatile的作用，介绍不使用 volatile 有可能出现的问题，简单介绍 cpu 的指令重排序，介绍Java的乐观锁和悲观锁======

### 基础知识普及

#### 简介线程之间的可见性

- 一个线程对共享变量的修改能够及时被其他线程发觉

  i.代码验证线程之间不可见

  ```java
  public class TestVolatile {
  
      public static boolean flag = true;
  
      public void m(){
          System.out.println("循环开始");
  
          while(flag){
  
          }
  
          System.out.println("循环结束");
      }
  
      public static void main(String[] args) throws InterruptedException {
          new Thread(()->{new TestVolatile().m();}).start();
  
          Thread.sleep(1000);
  
          flag = false;
      }
  }
  ```

  1. 这是一个非常简单的测试，用于验证线程之间不可见
  2. 我们运行这段代码，我们会发现，原本应该在程序运行了1000ms后程序中断，但是却一直没有停止，这和线程之间的不可见有关

#### 简介共享变量

- 当一个变量在很多线程里面都存在副本时，这个变量就被称为共享变量

#### 简介JMM内存模型（Java Memory Model）

- 所有变量都存储在主内存中

- 所有的线程都有自己的一块独立内存空间，这些内存空间里面保存有这个线程使用到的变量的副本

- 线程对共享变量的操作都是只能对自己线程中共享变量的副本进行操作，不能直接读写主内存中的共享变量

- 线程之间无法相互访问共享变量的副本

  ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-2.png)

#### 线程之间实现可见性的原理

- 一个线程对共享变量进行修改的时候，必须在修改完毕后将修改结果刷入主内存中
- 另一个线程在使用共享变量的时候，必须从主内存中进行一次读取操作

### volatile实现共享变量可见性

#### volatile简介

- volatile的英文意思是可变的，易变的，从字面理解就是说被volatile标记的变量它很容易发生改变

#### volatile的功能

- volatile可以实现共享变量的可见性

  1. 在一开始展示线程之间不可见的代码中，如果我们给变量 flag 标记 volatile 的话，就会出现不一样的结果

- volatile可以禁止指令重排序

  i.在著名的 double check 单例中，我们会给 INSTANCE 添加上 volatile 的标记，这并不是为了实现线程之间的可见性，而是阻止指令重排序

  ii.指令重排序介绍

  1. 假设我有两个线程正在运行，其中有一个线程执行 a=4；b=8；另一个线程执行 b=5；在cpu进行指令重排序的时候，有可能是先执行b=8；b=5再执行a=4；这就是cpu的指令重排序。cpu的指令重排序大大提高了效率

  2. Java中的一行操作不一定等于一条cpu指令，例如 new Object();

     ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-3.png)

     在java中，new一个对象大致分为三步：

     - 在内存空间中创建一个初始的空间，初始空间中对象的成员变量都是默认值，例如int就等于0，对象就等于null，这里对应的就是cpu指令的 new 指令
     - 将初始对象中所有的成员变量赋予初始值，例如我有一个int，我赋予的初始值是5，那么就是在这一步进行的，这里对应的就是cpu指令的 invokespecial 指令
     - 让栈空间中存在的指针指向这个对象，这里对应的就是cpu指令的 putstatic 指令

  iii.为什么双重检查单例需要防止指令重排序

  1. 在淘宝的秒杀环节中，某一个冰箱的订单是由单例管理的，原本这个冰箱已经被下单了1000件，但是后台在调用的时候却变成了0，这到底是为什么呢？

     ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-4.png)

  2. 原来，在极高并发量的情况下出现了一个非常罕见的情况，在创建对象的时候发生了指令重排序，将原本应该先初始化在赋值的指令顺序调换了一下，结果更巧合的事情也发生了，还没有将这三条指令全部执行完毕，这个对象被拿来进行==null判断了，结果自然是这个对象!=null，于是这个对象就被直接返回，拿到后台进行操作了，结果自然是不言而喻的

- volatile不能保证原子性

  ```java
  public class TestVolatile {
  
      public static volatile int count;
  
      public static void m(){
          for(int i = 0; i < 100000; i++){
              count++;
          }
      }
  
      public static void main(String[] args) throws InterruptedException {
  
          List<Thread> threads = new ArrayList<> ();
  
          for(int i = 0; i < 100; i++){
              Thread thread = new Thread(() -> m());
              thread.start();
              threads.add(thread);
          }
  
          for (Thread thread : threads) {
              Thread thread1 = Thread.currentThread();
              thread1.join(thread.currentThread().getId());
          }
  
          System.out.println(count);
      }
  }
  ```

  如果volatile可以保证原子性的话，那么最后打印这个count的结果应该是10000000，但是结果却不一样，这是因为count++本身就分为3条指令，很容易被打断

  ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-5.png)

  这足以证明volatile无法保证原子性

#### volatile和sync的简单比较

- volatile可以实现线程间共享变量可见，synchronized也可以实现
- volatile可以禁止指令重排序，synchronized不可以
- volatile不可以保证原子性，synchronized可以

### CAS无锁优化原理

#### 无锁优化的作用

- 当我有一个变量，这个变量被许多的线程访问，我如果想要保证数据一致性，我要将所有关于这个变量的操作全部加上sync锁，这非常影响性能，有很多人想要优化这一情况，于是就提出了无锁优化 CAS 的方法

#### 无锁优化实例

```java
public class TestAtomicInteger {
    public static AtomicInteger count;

    static {
        count = new AtomicInteger(0);
    }

    public static void m(){
        for(int i = 0; i < 100000; i++){
            count.incrementAndGet();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        List<Thread> threads = new ArrayList<>();

        for(int i = 0; i < 100; i++){
            Thread thread = new Thread(() -> m());
            thread.start();
            threads.add(thread);
        }

        for (Thread thread : threads) {
            Thread thread1 = Thread.currentThread();
            thread1.join(thread.currentThread().getId());
        }

        System.out.println(count);
    }
}
```

![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-6.png)

- 我们可以看到，我们原本的count只是使用了int类型，如果想要保证原子性就必须在m()方法中添加sync锁，但是我们使用了 AtomicInteger 对象之后，我们没有加 sync 锁就解决了原子性问题

#### 无锁优化的原理

- 我们从源码部分出发来简单查看

  ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-7.png)

  这是 AtomicInteger 的方法 incrementAndGet ，我们发现调用了unsafe对象的 getAndAddInt 方法

  ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-8.png)

  这个unsafe对象在调用方法的时候，本质上是使用了 compareAndSwap 方法，这就是我们平时经常提到的 CAS 

- CAS 的运作逻辑

  ![image](\images\posts\JUC\2021-3-10-多线程与高并发编程系列(二)之Volatile简介-9.png)

  1. 这是cas的简单逻辑，我们传入三个值，第一个一个是当前要修改对象的当前值，也就是我们的count的当前值；第二个是我们期望这个count的值；第三个是这个count要被修改后的值；
  2. 假如这个count正在进行++操作，上一次修改完毕后，这个count的值是5，于是我们就期望这个count的值就是5，如果在进入到cas方法里进行比较的时候我们发现这个count目前的值就是5，和我们期望的值一样，也就证明在我们运行的这个期间，没有其他的线程来将这个count++，我们就用修改后的值，也就是6来覆盖这个count；但是还有另外一种情况，那就是当前这个count值不是5，而是6、7乃至于8，反正不等于期望值5，那么就会放弃这次修改，而是重新尝试一次cas
  3. 也就是说，每当cas失败后都会重新调用一次cas，这就非常像自旋锁，cas的本质就类似于自旋

#### CAS 的原子性保证

- 有人可能就会有疑问，如果我们在cas进行到if(value==expect)比较之后，线程发生了切换，另一个线程也进行 if(value==expect) 比较怎么办？
- 其实这种情况根本不会发生，cas方法里面的指令原子性是从cpu层面进行了保障的，也就是说cas方法里面的指令执行不会被分割

#### CAS的ABA问题

- ABA问题的介绍
  1. 一开始cas在进行判定的时候count的期望值是5，当前值也是5，将要进行cas操作的时候，另一个线程将这个count的值从5增加到6，然后再将6减回到5，此时第一个线程再次进行cas操作，并不会发现这个count被修改过然后重新尝试cas
- ABA可能引起的问题
  1. 如果我们设定的Atomic对象是基础数据类型，那么完全没有影响，哪怕没有发现值已经经过了修改也没有问题
  2. 如果我们设定的Atomic对象是一个引用数据类型，那么就有可能出现问题，例如：我有一个老师对象A，老师对象里面有一个成员变量叫宠物，这个宠物对象有一个属性叫名字，这个名字叫做旺财；现在有一个cas操作，要将老师对象的宠物的名字取出来进行比较，如果这个时候有一个线程进来将老师的引用改变了，改成了另外一个老师B，然后将老师A的宠物名字修改，然后再次将老师的引用从B指向回A。如果接下来在进行cas操作的话，cas是无法判断这个对象是否被修改过的，这样有可能会出现问题
- ABA问题的解决方法
  1. 我们每一次进行cas操作的时候就记录一个版本号，以后进行cas操作时，不但要比较内容是否发生了修改，还需要比较cas的版本号是否符合期望

### 乐观锁和悲观锁

#### 乐观锁和悲观锁定义

- 乐观锁，如其名字，乐观锁默认这个锁住的对象不会被其他线程访问修改，于是并不会真正意义上去加锁，而是在使用这个对象的时候去判断一下是否有线程对此对象进行了修改，Java中的CAS就是乐观锁
- 悲观锁，默认其他线程都会去访问这个对象并且对这个对象进行修改，于是就会加上锁，让其他想要访问这个资源的线程阻塞住，Java中的悲观锁就是Synchronized

#### 乐观锁和悲观锁的适用情况

- 乐观锁适用与锁争用情况较少的场景，这样就省去了影响效率的加锁过程
- 悲观锁适用与锁争用情况较多的场景，这样可以避免非常多的线程不断进行“自旋”操作，从而跑高cpu的情况

### Sync锁的细节补充

#### sync锁有可能出现的问题

- 我们对一个对象进行sync锁的时候，实质上是在这个对象的头信息上的某两位进行修改，假如说我们sync一个对象，但是在后面的执行过程中，我们将这个对象的引用指向了另外一个对象，这样就会导致这个新对象的头信息和旧对象的头信息不同，使得sync锁出现问题

#### sync锁的优化

- 当我们进行sync操作的时候要尽量地减少锁住的代码量

  ```java
  void synchronized m(){
      
      //业务逻辑
      
      //想要进行原子性控制的变量 count
      count++;
      
      //业务逻辑
      
  }
  ```

  在以上的情况中，如果我们将m方法整个锁住，的确可以保证原子性，但是这个sync锁却锁住了整个方法，这个方法里面有很多的业务逻辑，这个时候需要对锁进行优化

  ```java
  void m(){
      
      //业务逻辑
      
      //想要进行原子性控制的变量 count
      synchronized(this){
  	    count++;
      }    
      //业务逻辑
      
  }
  ```

  这种优化方式叫做锁的细化

- 但是在某种情况下，我们不需要进行锁细化，反而需要扩大锁的范围

  ```java
  void m(){
      
      
      synchronized(this){
  	    //业务逻辑
      }
      
      //想要进行原子性控制的变量 count
      synchronized(this){
  	    count++;
      }    
      
      synchronized(this){
  	    //业务逻辑
      }
      
      
  }
  ```

  在上面的情况中有非常多的锁，这种情况下就可以考虑扩大锁的范围，这样就可以减少锁的争用情况，反而可以提高效率



