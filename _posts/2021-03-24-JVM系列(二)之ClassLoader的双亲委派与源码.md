---
post: layout
title: JVM系列(二)之ClassLoader的双亲委派与源码
categories: JVM
description: 本篇将介绍 ClassLoader load class的过程，详细解释ClassLoader的结构层次，讲解ClassLoader的双亲委派机制，阅读ClassLoader的load方法源码，还会介绍如何实现自定义的ClassLoader
keywords: JVM
---

本篇将介绍 ClassLoader load class的过程，详细解释ClassLoader的结构层次，讲解ClassLoader的双亲委派机制，阅读ClassLoader的load方法源码，还会介绍如何实现自定义的ClassLoader
======

### ClassLoader  load class

![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-1.png)

- 在ClassLoader load class的过程中，一共分为3步骤

  1. loading
  2. linking
  3. initialization

- 这三个大步骤中的第二步又被分为三小步，分别是

  1. verification
  2. preparation
  3. resolution

- 我们来简单介绍这些流程

  - loading：

    首先，我们经过编译后的class文件现在正躺在硬盘上，我们通过ClassLoader可以把它从硬盘上load到内存中

  - linking——verification：

    我们的class文件已经被load到内存上了，现在，我们需要进行第二步，linking。首先我们会进行verification，其实就是校验这个class文件的前4个字节是不是cafe babe，如果不是的话就会在verification环节被拦截

  - linking——preparation：

    通过了一些初识校验后，我们会进行静态变量赋默认值过程，也就是说，这个class里面如果有静态变量，我们会先赋予一个默认值，这个默认值是固定的，如果是int类型，那么赋予0；如果是long类型，赋予oL；如果是浮点类型，赋予0.0；如果是对象，赋予null；顺带一提，如果这个静态变量是final标识的，那么直接就赋予初始值，而不是默认值

  - linking——resolution

    resolution阶段，会将符号引用解析为直接引用；在常量池里面，有很多的常量，例如有个就是java/lang/Object，一开始的时候，这个引用指向的就是一串字符串 “java/lang/Object” 但是当我们真正要使用object的时候，靠这个字符引用是没有用的，resolution阶段就是将这个字符引用变成真正的指向内存地址的引用

  - initialization

    在初始化阶段，才会将用户赋予静态变量的初始值赋予给静态变量，用于替换一开始的默认值

- 番外

  当ClassLoader load class的时候，其实会在内存中开辟两块儿地址，第一块地址是放置这个class文件的二进制码，也就是这个class初始文件；第二块地址放置的是这个class文件形成的class对象，我们使用反射也是通过这个class对象来直接new实例，然后调用

### ClassLoader详解

#### ClassLoader结构

![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-2.png)

- ClassLoader是有父加载器的概念的，不同类型的加载器有加载不同内容的权限

#### 父加载器、加载器的父类、加载加载器的加载器

- 在加载器的学习过程中，这几个概念非常容易混淆，我们特地来解释一下
- 父加载器
  1. 用户自定义的加载器的父加载器是 AppClassLoader
  2. App ClassLoader的父加载器是 ExtensionClassLoader
  3. ExtensionClassLoader的父加载器是 BootStrapClassLoader
  4. 父加载器和加载器的关系是：加载器有一个成员变量叫做parent，这个parent就是指向的父加载器。也就是说，**父加载器不是加载器的父类！！**

![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-3.png)

- 这里我们可以对classloader进行验证

![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-4.png)

- 这里我们可以发现，AppClassLoader的类加载器是BootStrapClassLoader，AppClassLoader的父加载器是ExtClassLoader
- ExtClassLoader的加载器是BootStrapClassLoader，ExtClassLoader的父加载器是BootStrapClassLoader
- 总结：AppClassLoader和ExtClassLoader的加载器是根加载器，但是他们的父加载器却是不同的

#### ClassLoader的双亲委派机制

![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-5.png)

- 双亲委派机制的文字描述介绍

  1. 我们有一个自定义的classLoader，我们使用它来load .class文件

  1. 我们调用了 customClassLoader的load方法，然后customClassLoader将会去自己维护的容器中查询这个.class文件是否已经被加载了，如果加载了，那么直接返回，如果没有返回进入下一步
  2. 下一步就是customClassLoader调用parent（父类加载器）的load方法来尝试得到.class文件，这个父加载器就是appClassLoader
  3. appClassLoader首先也会去自己维护的容器中查询这个.class文件是否已经被加载了，如果加载了，那么就返回，如果没有，也会调用appClassLoader的父加载器的加载方法
  4. appClassLoader的父加载器是ExtClassLoader，这个ExtClassLoader加载的流程也是一样，首先去自己维护的容器中去查找是否已经有被加载完毕的class文件，如果有，那么我们就直接返回。如果没有就调用父加载器的加载方法
  5. ExtClassLoader的父加载器是BootStrapClassLoader，也就是根加载器，如果到了这一步，根加载器首先也会去自己维护的容器中去试图寻找这个想要加载的class文件，如果找不到，那么根加载器就会自己尝试加载这个class文件，如果加载不出来，就会“说”：‘欸，儿子，我加载不了，你来加载！’然后就会将这个加载任务交给它的子加载器，也就是ExtClassLoader来加载
  6. ExtClassLoader加载器再尝试加载，如果加载成功，那么就直接返回，加载不出来那么就会交给自己的子加载器来加载，ExtClassLoader的子加载器就是AppClassLoader
  7. AppClassLoader加载器再尝试加载，如果加载成功，那么就直接返回，加载不出来那么就会交给自己的子加载器来加载，AppClassLoader的子加载器就是CustomClassLoader
  8. CustomClassLoader尝试加载，如果加载成功，那么返回，如果加载失败，那么就直接抛出异常：“ClassNotFount”

- 双亲委派机制的源码解析

  ![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-6.png)

  ![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-7.png)

- 为什么要有双亲委派机制？

  1. 安全：如果一个class文件一来，我二话不说就将之加载到内存，那么就会出现非常严重的bug，假如我自定义了一个类，就叫做java.lang.String，这个类是我自己定义的，里面隐藏了一个方法，将被赋予String对象中的内容通过邮件发给我，那么我把这个类混杂在一堆的jar包里面，交给用户使用，这样用户在导入了我的jar包之后，密码就会被泄漏！
  2. 防止资源浪费：如果我已经load过一遍的class内容，再次load一遍就太浪费了，没有必要
  3. 总结：双亲委派机制最重要的就是为了安全，这是重中之重！！

#### 自定义ClassLoader

- 自定义的classLoader只需要继承classLoader，然后重写findClass(String name)方法即可

  ```java
  public class YINClassLoader extends ClassLoader {
      @Override
      protected Class<?> findClass(String name) throws ClassNotFoundException {
          File f = new File("d:/java code/", name.replace(".", "/").concat(".class"));
          try {
              FileInputStream fis = new FileInputStream(f);
              ByteArrayOutputStream baos = new ByteArrayOutputStream();
              int b = 0;
  
              while ((b=fis.read()) !=0) {
                  baos.write(b);
              }
  
              byte[] bytes = baos.toByteArray();
              baos.close();
              fis.close();//可以写的更加严谨
  
              return defineClass(name, bytes, 0, bytes.length);
          } catch (Exception e) {
              e.printStackTrace();
          }
          return super.findClass(name); //throws ClassNotFoundException
      }
  
      public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
          ClassLoader yinClassLoader = new YINClassLoader();
  
          Class clazz = yinClassLoader.loadClass("cn.yin.vm02.HelloWorld");
  
          HelloWorld h = (HelloWorld)clazz.newInstance();
          h.m();
      }
  }
  
  
  
  
  public class HelloWorld {
  
    static String test;
  
    static {
      test = "加载成功！自定义类加载器成功！";
    }
  
    public  void m(){
      System.out.println(test);
    }  
  }
  ```

  ![image](\images\posts\JVM\2021-03-24-JVM系列(二)之ClassLoader的双亲委派与源码-8.png)

### JVM的懒加载机制

#### 懒加载解释

- JVM在加载资源的时候不会一股脑将所有资源加载，而是用到什么资源就加载什么资源

#### 懒加载与初始化

- 虽然JVM没有规定加载资源的时间，但是JVM规定了什么时候对资源进行初始化！
  1. new 、getstatic 、putstatic 、invokestaticmethod（访问final变量除外）
  2. 使用java.lang.reflect对类进行反射调用
  3. 初始化子类，父类必须先初始化
  4. JVM启动时，被执行的主类必须初始化
  5. 使用动态语言支持，一个java.lang.invoke.MethodHandle实例最后解析的结果REF_getstatic 、REF_putstatic 、REF_invokestatic的方法句柄会触发初始化

### 解释执行 Vs 编译执行

#### 解释执行

- 解释执行就是将源语言编写的源程序作为输入，**解释一句就交给计算机执行一句**，并不形成目标程序
- 优点：方便快捷，适用于小型计算机，不依赖平台
- 缺点：执行速度慢，遇到循环可以慢到姥姥家

#### 编译执行

- 编译执行就是将源语言编写的源程序作为输入，然后将之完全编译完毕后形成计算机语言的目标程序（例如c语言编译完毕的.exe程序，静静的躺在磁盘中）
- 优点：编译完毕后执行速度极快，并且可以多次使用
- 缺点：编译执行的程序在启动阶段会比较慢

#### Java的执行模式

- 大家都知道，Java是混合模式的，Oracle提供的有一个JVM叫做HotSpot，直译过来就是 “热点” ，意思是JVM可以将执行次数多的热点代码段检测出来并且编译
- 一开始JVM会使用解释执行，但是Java存在一个热点检测机制，有一个计数器，当一段代码在短时间内被执行了一定次数后就会判定这段代码是热点代码，然后就会通过JIT（Just In-Time Compiler）将这段代码编译