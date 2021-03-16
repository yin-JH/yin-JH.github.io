---
layout: post
title: 多线程与高并发编程系列(四)之concurrent同步锁补充及AQS源码阅读
categories: JUC
description: 本篇补充上篇未补充完毕的concurrent同步锁、多线程题目实战以及AQS源码阅读
keywords: JUC
---

本篇将介绍上篇concurrent同步锁中没有介绍的LockSupport，详解多线程与高并发的面试题目，阅读AQS源码分享源码阅读的原则======

### LockSupport介绍

- LockSupport，顾名思义就是专门提供对锁的支持的类，通过这个类我们可以直接不加锁，直接阻塞线程

```java
public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            System.out.println("Thread 1 启动");

            LockSupport.park();

            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("Thread 1 结束");
        });

        Thread t2 = new Thread(() -> {
            System.out.println("Thread 2 启动");

            LockSupport.unpark(t1);



            System.out.println("Thread 2 启动了 Thread 1 的unpark");

        });


        t1.start();
        t2.start();


        TimeUnit.SECONDS.sleep(2);
    }
```

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-1.png)

### 经典多线程面试题

#### 题目介绍

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-2.png)

#### 错误写法--第一题

```java
public class T01_WithoutVolatile {
    /*volatile*/ List list = new ArrayList<>();

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {
        T01_WithoutVolatile t01 = new T01_WithoutVolatile();

        Thread t1 = new Thread(()->{
            for (int i = 0; i < 10; i++) {
                /*try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                System.out.println(i);
                t01.add(new Object());
            }
        });

        Thread t2 = new Thread(()->{
            while (true) {
                if(t01.size() == 5){
                    System.out.println("线程二感知到list中有5个对象");
                    break;
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

- 我们这样写会因为线程之间的不可见性导致失败

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-3.png)

- 有人可能就会好奇，因为线程之间不可见，那么我直接加上volatile是不是就可以直接解决这个问题了呢？

```java
public class T01_WithoutVolatile {
    volatile List list = new ArrayList<>();

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {
        T01_WithoutVolatile t01 = new T01_WithoutVolatile();

        Thread t1 = new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(i);
                t01.add(new Object());
            }
        });

        Thread t2 = new Thread(()->{
            while (true) {
                if(t01.size() == 5){
                    System.out.println("线程二感知到list中有5个对象");
                    break;
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

- 请注意，我们解封了volatile标注，同时我们还让线程一每add一次就 sleep 一秒，这样得到了我们想要的结果

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-4.png)

- 但是！！如果我们将sleep给注释掉，会得到一个完全不一样的结果

```java
public class T01_WithoutVolatile {
    volatile List list = new ArrayList<>();

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {
        T01_WithoutVolatile t01 = new T01_WithoutVolatile();

        Thread t1 = new Thread(()->{
            for (int i = 0; i < 10; i++) {
                /*try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                System.out.println(i);
                t01.add(new Object());
            }
        });

        Thread t2 = new Thread(()->{
            while (true) {
                if(t01.size() == 5){
                    System.out.println("线程二感知到list中有5个对象");
                    break;
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-3.png)

- 对此，我的认知是volatile保证线程可见性是要将线程中修改的内容重新写回到主内存中的共享变量里面，但是如果volatile修饰的是一个对象的话，那么对象里面的成员变量发生变化，例如这个list的size如果++的速度过快，volatile还没来得及将5写入的主内存中的共享变量里面的时候就已经从5加到了6，但是主内存中的共享变量里面size还是4，线程二在进行while的验证的时候发现还是4，于是无法触发if条件，但是下一次线程一将size写入主内存的时候就不是写5了，而是直接写6，于是就会出现这种情况；当线程一每add一下就会sleep1秒的时候，这一秒内volatile起作用，将共享变量写回到了主内存中去，于是就不会出现这种情况
- 总结
  1. 没有绝对的把握，不要使用volatile
  2. 不要使用volatile修饰对象，要使用volatile，最好修饰基础数据类型

#### wait、notify解决--第一题

```java
public class T01_WaitNotify {
    static final Object lock = new Object();

    List list = new ArrayList<>();

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {
        T01_WaitNotify t01 = new T01_WaitNotify();

        Thread t1 = null;
        Thread t2 = null;


        t1 = new Thread(()->{
            synchronized (lock){
                for (int i = 0; i < 10; i++) {
                /*try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                    System.out.println(i);
                    t01.add(new Object());

                    if(t01.size() == 5){
                        lock.notify();
                    }

                }
            }
        });

        t2 = new Thread(()->{
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("线程二感知到list中有5个对象");
            }
        });

        t2.start();
        t1.start();
    }
}
```

- 这种做法的思想是首先让线程二，也就是监视list中size的线程先启动，然后wait住，接着启动线程一往list中添加对象，当list中的size达到5的时候就激活方法notify叫醒正在wait的线程二
- 要注意的点是，必须让线程二先启动，先wait住，要不然会报错
- 这样的思路是没有问题的，但是上面的代码却有细节上的问题，我们可以看运行结果

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-5.png)

- 结果并没有达到预期的效果，即，在4的时候就打印出“线程二感知到list中有5个对象”，这样的结果不尽如人意，这是为什么呢？

- 原因很简单，那是因为notify不会释放锁资源：

  1. 我们的线程一在运行到list里面有5个对象的时候成功的调用了notify方法，但是，此时仍然处于同步代码块儿中，线程一仍然持有对象的锁，此时哪怕线程二已经被叫醒了，但是仍然无法执行，因为被阻塞住了，没有获得锁
  2. 解决方法很简单，我们只需要让线程一让出锁资源即可

- 改进写法

  ```java
  public class T01_WaitNotify {
      static final Object lock = new Object();
  
      static Thread t1 = null;
      static Thread t2 = null;
  
      List list = new ArrayList<>();
  
      public void add(Object o){
          list.add(o);
      }
  
      public int size() {
          return list.size();
      }
  
      public static void main(String[] args) {
          T01_WaitNotify t01 = new T01_WaitNotify();
  
  
  
  
          t1 = new Thread(()->{
              synchronized (lock){
                  for (int i = 0; i < 10; i++) {
                  /*try {
                      TimeUnit.SECONDS.sleep(1);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }*/
                      System.out.println(i);
                      t01.add(new Object());
  
                      if(t01.size() == 5){
                          lock.notify();
                          try {
                              lock.wait();
                          } catch (InterruptedException e) {
                              e.printStackTrace();
                          }
                      }
  
                  }
              }
          });
  
          t2 = new Thread(()->{
              synchronized (lock) {
                  try {
                      lock.wait();
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  System.out.println("线程二感知到list中有5个对象");
                  lock.notify();
              }
          });
  
          t2.start();
          try {
              Thread.sleep(1);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
          t1.start();
      }
  }
  ```

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-4.png)

- 请注意如果线程二在结束的时候还没有调用notify方法的话，线程一就会一直沉睡

#### CountDownLatch解决--第一题

```java
public class T01_CountDownLatch {
    static CountDownLatch firstLatch = new CountDownLatch(1);
    static CountDownLatch secondLatch = new CountDownLatch(1);

    volatile List list = new ArrayList<>();

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {

        T01_CountDownLatch t01 = new T01_CountDownLatch();

        Thread t1 = new Thread(()->{
            for (int i = 0; i < 10; i++) {
                /*try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                System.out.println(i);
                t01.add(new Object());

                if(t01.size() == 5){
                    firstLatch.countDown();
                    try {
                        secondLatch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });

        Thread t2 = new Thread(()->{
            try {
                firstLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程二感知到list中有5个对象");
            secondLatch.countDown();
        });

        t1.start();
        t2.start();
    }
}
```

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-4.png)

- 这里CountDownLatch模拟的就是wait notity的思路，用到了两个门闩

#### LockSupport解决--第一题

```java
public class T01_LockSupport {
    volatile List list = new ArrayList<>();

    static Thread t1 = null;
    static Thread t2 = null;

    public void add(Object o){
        list.add(o);
    }

    public int size() {
        return list.size();
    }

    public static void main(String[] args) {
        T01_LockSupport t01 = new T01_LockSupport();

        t1 = new Thread(()->{
            for (int i = 0; i < 10; i++) {
                /*try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                System.out.println(i);
                t01.add(new Object());
                if(t01.size() == 5){
                    LockSupport.unpark(t2);
                    LockSupport.park();
                }
            }
        });

        t2 = new Thread(()->{
            LockSupport.park();
            System.out.println("线程二感知到list中有5个对象");
            LockSupport.unpark(t1);
        });

        t1.start();
        t2.start();
    }
}
```

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-4.png)

#### wait、notify解决--第二题

```java
public class T02_WaitNotify {
    LinkedList list = new LinkedList();
    private static final int MAX = 10;
    private static int count = 0;

    synchronized void put(Object o) throws InterruptedException {
        while (list.size() == MAX){
            System.out.println("生产者：" + Thread.currentThread().getName() + "   发现容器已满，停止生产！");
            this.wait();
        }

        System.out.println("生产者：" + Thread.currentThread().getName() + "  生产后，容器中有 " + list.size());
        list.add(o);
        ++count;
        this.notifyAll();
    }

    synchronized void get() throws InterruptedException {
        while (count == 0){
            System.out.println("消费者：" + Thread.currentThread().getName() + " 发现容器中没有对象，停止拿取");
            this.wait();
        }

        list.removeFirst();
        --count;
        System.out.println("消费者：" + Thread.currentThread().getName() + " 消费一个，还有 " + count);
        this.notifyAll();
    }

    public static void main(String[] args) {
        T02_WaitNotify t02 = new T02_WaitNotify();

        //消费者启动
        for(int i = 0; i < 10; i++){
            new Thread(()-> {
                try {
                    t02.get();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "p" + i).start();
        }

        for(int i = 0; i < 2; i++){
            new Thread(()-> {
                try {
                    for(int j = 0; j < 5; j++){
                        t02.put(new Object());
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "c" + i).start();
        }

    }
}
```

![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-6.png)

- 这种写法的缺点是当一个生产者在容器满的时候会先叫醒其他所有的线程然后再沉睡，这样有可能会使得其他的生产者线程被叫醒，但是却又无法生产，占用资源，可以优化

#### ReentrantLock解决--第二题

```java
public class T02_ReentrantLock {
    private ReentrantLock lock = new ReentrantLock();
    private Condition consumer = lock.newCondition();
    private Condition producer = lock.newCondition();

    LinkedList list = new LinkedList();
    private static final int MAX = 10;
    private static int count = 0;

    synchronized void put(Object o) throws InterruptedException {
        try {
            lock.lock();
            while (list.size() == MAX){
                System.out.println("生产者：" + Thread.currentThread().getName() + "   发现容器已满，停止生产！");
                producer.await();
            }

            System.out.println("生产者：" + Thread.currentThread().getName() + "  生产后，容器中有 " + list.size());
            list.add(o);
            ++count;
            consumer.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    synchronized void get() {
        try {
            lock.lock();

            while (count == 0){
                System.out.println("消费者：" + Thread.currentThread().getName() + " 发现容器中没有对象，停止拿取");
                consumer.await();
            }

            list.removeFirst();
            --count;
            System.out.println("消费者：" + Thread.currentThread().getName() + " 消费一个，还有 " + count);
            producer.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        T02_ReentrantLock t02 = new T02_ReentrantLock();

        for(int i = 0; i < 2; i++){
            new Thread(()-> {
                try {
                    for(int j = 0; j < 5; j++){
                        t02.put(new Object());
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "c" + i).start();
        }

        //消费者启动
        for(int i = 0; i < 10; i++){
            new Thread(()-> {
                t02.get();
            }, "p" + i).start();
        }

    }
}
```

- 通过这种方式，我们可以在生产者沉睡的时候专门叫醒消费者去消费；消费者沉睡的时候专门叫醒生产者去生产

### AQS源码与源码阅读技巧入门

#### 源码阅读的原则

1. 跑不起来不读：读源码时，要将代码跑起来，跟着代码的逻辑来阅读
2. 解决问题即可：阅读源码是为了解决问题（有时是应付面试）
3. 一条线索到底：跟着运行轨迹一路读下去
4. 无关细节掠过：不要死磕某些细节，阅读源码最重要的是理解作者的思路

#### ReentrantLock的lock()阅读

- 首先，我们打上断点，一步步跟进

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-8.png)

- 我们发现，lock方法调用了sync.lock()，这个sync是什么呢？我们可以稍微看一眼

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-9.png)

- 原来，这个sync就是大名鼎鼎的AQS

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-10.png)

- 当然，这个Sync还只是一个模板，真正的实现我们需要查看一下

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-11.png)

- 也就是说，当我们在new ReentrantLock的时候，new 出来的是 非公平锁，这个NonfairSync也是继承自AQS的，sync的lock的方法我们可以大致看一眼

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-12.png)

- 我们可以看到，我们在lock的时候，调用的其实是cas，如果这个ReentrantLock没有被上锁的话，那么直接上锁，将state从0改为1，然后将当前的线程设置为这个lock的独占线程，最后直接返回，上锁成功；

- 我们研究的重点在于cas失败，这个锁已经被其他线程占用的情况

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-13.png)

- 首先直接进行acquire同步，我们根据源码的结构可以看到，首先会进行tryAcquire，如果成功，直接结束方法；如果失败，进行后续操作，我们先不介绍后续操作。

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-14.png)

- 我们可以看到，首先我们会判断 state 的值是否为0，这个state非常有意思，它来自于AQS，对于不同的实现了AQS的Concurrent锁，state都有不同的含义，在ReentrantLock中，state的意思是上锁次数：如果state为0，那么这个锁还没有被线程独占；如果state为1，那么这个锁已经被一个线程独占了；如果state大于1，那么这个锁已经被独占线程重入了

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-15.png)

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-16.png)

- 首先，我们会判断 state 是否为0，如果为0的话，那么我们直接进行cas操作，如果成功，那么就同步成功；如果同步失败，我们会先检测当前的锁是否已经被当前线程独占，如果独占了，那么直接重入一次，如果没有被独占，那么就返回tryAcquire失败的返回值

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-17.png)

- 如果失败了，那么我们就会回到一开始的地方进行后续操作。首先，我们观察一下这个方法的名字，acquireQueued，意思是将当前线程送入一个队列中，这里面还有一个方法，叫做 addWaiter，顾名思义，就是添加等待者，这个传入的参数是Node.EXCLUSIVE，这个Node.EXCLUSIVE的类型是Node。

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-18.png)

- 这个Node的类型一共有两种，一种是独占型，一种是共享型

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-19.png)

- 接下来就是AQS的核心，这里会调用cas方法，尝试将这个Node添加到AQS维护的一个双向链表的末尾，如果成功，返回这个Node，如果失败，会调用enq(Node)方法，这个方法里面是一个死循环，唯一的目标就是在双向链表的末尾添加上这个Node

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-20.png)

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-21.png)

- 添加成功后，这个新的线程节点就已经是在队列中了，这个队列就是大名鼎鼎的同步队列，接下来就是核心了，在接下来的代码里面，我们首先会获取新添加节点的前一个节点，如果前一个节点是head的话，就证明这个当前节点是第二个尝试获取锁资源的线程，前面只有一个线程，head获取了锁资源，此时就会开始尝试获取锁，也就是tryacquire方法，如果获取成功，那么当前线程就获取了锁资源，如果没有成功就去执行方法 shouldParkAfterFailedAcquired，这个方法顾名思义就是判断是否需要被park住；如果这个当前线程是队列中的第二个的话，那么就不会被park住，会一直尝试获取锁资源；如果这个当前线程不是队列中的第二个，而是第三个乃至更加后面的话，那么就会被park住

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-22.png)

- 总结

  1. 我们发现，AQS的效率之所以在一些特定条件下高于Sync的地方是因为如果使用Sync的话需要将整个同步队列上锁来保证线程安全，但是AQS运用了cas操作，这样就可以只关心队列的末尾是否是我期望的那个Node，如果是我就往尾部添加Node，如果不是就重复cas操作，直到添加完毕为止

  2. 在AQS实现的同步锁ReentrantLock内部，只有在同步队列的第二个节点才会重复的tryAcquire，后面的节点全部都被park住了

  3. 为什么说AQS的底层是由volatile和cas实现的，通过上面的源码阅读，我们已经发现了cas的运用非常广泛，至于volatile的用处，我只需要提一句就知道了，那就是 AQS 里面的state就是用 volatile 来修饰的

     ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-23.png)

#### ReentrantLock的unlock()阅读

- 通过了lock()方法较为详细的解读后，我们就简单地解读一下unlock()方法

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-24.png)

- unlock方法里面会调用release方法

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-25.png)

- 首先会进行一次tryRelease，如果tryRelease成功的话就会成功地释放锁资源

- 在tryrelease方法里面，会进行线程判断，如果当前线程没有获得锁资源的话会直接抛出异常，如果通过这一层检测，接下来会对state检测，如果释放了一层锁资源，state的值减了1之后为0的话，那么就成功解锁；如果减了1还不等于1的话就说明这个锁至少被重入了1次，因此无法释放锁资源

  ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-26.png)

- 总结

  1. 可重入锁ReentrantLock中的state是标识重入次数的变量

#### ReentrantLock的公平锁的lock()方法和非公平锁的lock()方法的区别

1. 非公平锁一进入lock方法就会尝试一次加锁，失败了会调用acquire(1)方法；公平锁一进入lock()方法不会忙着尝试加锁，而是直接调用acquire(1)方法

   ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-27.png)

   ![image](\images\posts\JUC\2021-3-14-多线程与高并发编程系列(四)之LockSupport、多线程面试题、AQS源码简单阅读-28.png)

2. 非公平锁可以减少cpu唤醒线程的代价，提高整体效率；公平锁会挨个唤醒，效率会略低

3. 非公平锁可能导致有线程‘饿死’，公平锁因为是挨着排队，可以保证每个线程都有公平执行的机会

