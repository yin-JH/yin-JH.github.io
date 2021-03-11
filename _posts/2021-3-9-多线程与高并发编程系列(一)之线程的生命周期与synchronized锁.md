---
layout: post
title: 多线程与高并发编程系列(一)之线程的生命周期与synchronized锁
categories: JUC
description: 学习多线程与高并发底层知识
keywords: JUC
---

本篇开始，学习多线程与高并发知识，理解底层并发逻辑======

### 关于线程的基础知识

#### 程序、进程、线程之间的区别

- 程序是位于磁盘上面的静态文件，例如qq.exe
- 当位于磁盘上的程序被执行后，会在内存空间中形成一块儿独立的内存体，这个内存体里有自己的地址空间和自己的堆，操作系统会以进程为单位分配空间
- 线程是轻量级的线程，是cpu调度的最小单位，是程序中的可执行路径

#### 创建多线程的方法

- 通过继承 java.lang.Thread 类来创建

  ```java
  package cn.yin;
  
  
  class T extends Thread {
      @Override
      public void run() {
          for (int i = 0; i < 10; i++){
              try {
                  Thread.sleep(10);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              System.out.println("i am t   " + i);
          }
      }
  }
  
  public class Test {
      public static void main(String[] args) throws InterruptedException {
          Thread t = new T();
          t.start();
  
          for(int i = 0; i < 10; i++){
              Thread.sleep(10);
              System.out.println("i am main   " + i);
          }
      }
  }
  ```

- 通过实现 java.lang.Runnable 接口创建

  ```java
  package cn.yin.startThread;
  
  class T2 implements Runnable {
      @Override
      public void run() {
          for (int i = 0; i < 10; i++){
              try {
                  Thread.sleep(10);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              System.out.println("i am t2   ");
          }
      }
  }
  
  public class Test02 {
      public static void main(String[] args) throws InterruptedException {
          Thread t2 = new Thread(new T2());
          t2.start();
  
          for(int i = 0; i < 10; i++){
              Thread.sleep(10);
              System.out.println("i am main   " + i);
          }
      }
  }
  ```

- 关于线程启动的注意事项

  1. 严格来说，线程启动的方法只有这两种，但是还可以通过线程池来启动，但是通过线程池来启动本质上还是使用这两种方法之一
  2. 关于线程的 start() 方法和 run() 方法，start() 是启动一个线程，run() 只是调用 run() 方法
  3. 一个线程如果结束了就无法再次启动它

#### 线程的6种状态

![image](\images\posts\JUC\2021-3-9-多线程与高并发编程系列(一)之线程的生命周期与synchronized锁-1.png)

- 一个线程首先被创建出来，这就是 new() 状态

- 当一个线程被调用 start() 方法的时候就进入了 runnable 状态

  i.Runnable状态分为两个状态，一个是 running 状态，就是线程被cpu调用执行的状态；一个是 ready 状态，就是线程就绪等待被调用

  ii.running状态随时有可能会加入ready状态

- 当线程种调用了sleep(time)、wait(time)、join(time)、LockSupport.parkNanos()、LockSupport.parkUntil()方法时，进入TimeWaiting状态

  i.TimeWaiting状态只需要等待一定的时间就可以解锁，回到runnable状态

- 当线程被调用wait()、join()、LockSupport.park()方法时进入Waiting状态

  i.在Waiting状态时，单纯的等待无法解锁这个状态，只有当调用了o.notify()、o.notifyAll()、LockSupport.park()方法时才会解除状态，进入 runnable 状态

- 当线程等待同步代码块的锁的时候，进入 blocking 状态

  i.当没有获得锁的时候就一直处于 blocking 状态，直到获得锁为止

- 当线程结束后线程就会进入terminated状态

### Synchronized锁的应用和升级

#### Synchronized锁的应用

- 锁是保证数据一致性的重要工具
- synchronized锁只能锁住对象，虽然我们平时经常说锁住一段代码，但是其实有可能是锁住的XXX.class对象
- 假如要对某一数据加以控制的话那么一定要加上锁，否则有可能出现脏读现象
- 千万不要锁定 String、基础数据类型 Integer 等
- 如果给非静态方法加上 synchronized 那么默认就会锁定这个对象；如果给静态方法加上 synchronized 方法，那么就会默认锁定 XXX.class 对象
- synchronized锁具有可重入属性，如果没有可重入属性的话，那么就会有可能出现继承了 Thread 的子类调用 start() 方法反而出现死锁现象

#### Synchronized锁升级

- Synchronized锁的历史

  i.在一开始的时候，sync锁是一个重量级锁，只要锁定一行代码，那么就会调用OS来加锁，OS锁的代价非常高，这让很多人不满，于是他们就自己开发了锁，这些锁的性能远超sync锁

  ii.后来官方优化了sync锁，引入了锁升级机制，这使得sync锁的效率变得极高

- Sync的锁升级

  i.当我们第一次给一个对象加锁的时候，JVM会默认只有这一个线程会访问这个对象，于是JVM会简单的记录一下线程的id，下一次只要还是这个线程来访问，在对比了id之后直接就会放行，这就是——偏向锁

  ii.当我们有两个线程在通过sync访问一个对象的时候，就会使得sync锁升级，从偏向锁升级为自旋锁，先来的线程获得访问资格，剩下的线程就在原地自旋，自旋完毕后再次尝试访问，这就是——自旋锁

  iii.当自旋完毕后线程再次尝试访问资源，但是发现资源还是没有被释放，这个时候锁就会再次升级变为重量级锁，通过 OS 来加上锁

- Sync锁的细节

  i.自旋锁的自旋次数默认是自旋十次

  ii.自旋锁和重量级锁的适用情况不同

  1. 当尝试访问同一资源的线程少的时候，自旋锁的效率高于重量级锁
  2. 当尝试访问同一资源的线程多的时候，如果还是自旋锁，那么有非常多的线程在自旋，无缘无故就会跑高cpu的资源，这时就不如重量级锁了

  iii.Sync锁在升级了之后就无法降级，但是如果你的技术高超，那么还是可以自己改动jdk，提供锁降级能力
