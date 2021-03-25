---
layout: post
title: JVM系列(一)之初识JVM和ClassFIleFormat
categories: JVM
description: 本篇将简单介绍JVM以及介绍 Class 文件的文件格式
keywords: JVM
---

本篇开始，就要进入JVM的学习，我们将逐步了解JVM，理解JVM底层的一些实现，最终要做到能够对JVM进行调优
======

### 初识JVM

#### JVM和Java

- 有人说，JVM是专属Java的虚拟机，这个说法其实不太准确。我们大家都知道Java是跨平台的语言，那么JVM又是什么呢？其实JVM可以叫做跨语言的平台
- 据不完全统计，JVM上面可以运行的语言超过100种，Java只是其中最出名的那个语言
- JVM其实和.java文件没有什么关系，**JVM只和.class文件有关！**，也就是说，你将.java文件扔给JVM，JVM是无法识别的，只有你将.class文件交给JVM，JVM才可以运行，至于.java文件，那是交给javac的事情
- 只要你能将一个语言规范地编译成.class文件，那么这个文件就可以交给JVM来运行

#### JVM 、JRE 、JDK之间的关系

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-1.png)

- JVM的用处就是执行class文件

- JRE在包括JVM的基础上，还包括了核心类库，例如 Object 类、String 类等

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-2.png)

- JDK在包括JRE的基础上，还包括一些工具，如图中，除了jre文件中的其他所有内容都是 development kit

#### 市面上的JVM

- 市面上有非常多的JVM，例如 HotSpot 、TaoBaoVM 、OpenJDK 、Zing 等等

### ClassFileFormat

#### 用最简单的程序来阅读class文件

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-3.png)

- 这个程序极其简单，我们可以编译之后看看class文件

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-4.png)

- 犹记得第一次学Java的时候，老师就提到过，一个class，如果没有构造方法的话会默认提供一个空构造，我们通过class文件看到了，不过这个class文件是通过反编译看到的，我们想要直接看二进制文件

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-5.png)

#### Class File 大致结构

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-6.png)

##### 魔数（magic number）

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-7.png)

- 在文件的最开始有4个字节叫做“魔数”，魔数的作用是表示这个文件的类型，JVM可以通过这个开头的4个字节了解到这个文件类型，例如.class文件的魔数翻译为16进制就是 CAFE BABE

##### 版本号（major number 、minor number）

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-8.png)

- 在魔数的后面，紧跟着4个字节，前两个是 minor number 、后两个是 major number，这里我们可以看到，minor number是0，major number是52
- major number是大的版本号，52指的就是Java 1.8，53就是Java 1.9；minor number是0，指的就是1.8.0，没有小版本更新

##### 常量池中常量数量（constant_pool_count）

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-9.png)

- 这两个字节表示的就是常量池中常量的个数，虽然转化成为10进制这个值是16，但是常量池中的常量数量却是15，这是因为常量池中的第零位是一个保留位，代表没有任何引用指向这个类

##### 常量池（constant_pool）

- 这是一个长度为 constant_pool_size - 1 的表，这个留到后面详细说

##### 权限修饰符（access_flag）

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-10.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-11.png)

- 我们可以发现，这个 access flag 就是 上图中的0X0001和0X0020进行与操作得到的结果

##### 后面其他的内容（知道即可）

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-12.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-13.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-14.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-15.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-16.png)

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-17.png)

#### constant_pool 详细解释

- constant_pool非常重要，如果可以读懂constant_pool的话，那么class file的东西基本就读懂了一半以上了

- 首先，常量池中对象的第一个字节标识这个常量池对象的tag，也就是“名称”，我们来尝试阅读一下

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-18.png)

  0A，翻译成十进制就是10，我们去查询一下tag为10的常量是什么

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-19.png)

  这个10号对象的名称是 Methodref_info 大致翻译一下就是 方法引用信息 ，我们可以看到，这个常量对象一共有5个字节，第一个字节是这个常量池对象的tag；后面两个字节是这个方法的 类或者接口 描述的“指向”（有点像指针，指向了常量池中的其他对象）；最后两个字节也是一个指向，指向了这个方法的NameAndType常量对象

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-20.png)

  常量池中的中间和最后两个字节分别是0X03和0X0D，翻译成十进制就是3和13，我们可以根据这个索引来查看

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-21.png)

  我们可以发现class info的信息指向的是object对象

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-22.png)

  我们可以发现NameAndType的信息指向的是一个 init 方法，也就是构造方法，并且这个构造方法的描述是<()V>，前面的括号指的是传入方法的参数，这里面为空；后面的V代表返回类型，V代表void

- 根据这一系列的解读，我们可以发现，这个常量池对象指向了两个常量池对象，指向的两个常量池对象有：class_info,这个class_info指向了一个常量对象，是编号为15的常量池对象，这个编号为15的常量池对象指向的就是 object 类

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-23.png)

  除了class_info，还有一个NameAndType，这个常量池对象也指向了两个常量池对象，分别是第4号和第5号

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-24.png)

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-25.png)

- 读通了一个常量池对象，我们再来读一个

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-26.png)

  这个07指的是tag为7的常量对象，我们查一下

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-27.png)

  我们发现，这个常量对象也是class_info，并且指向了一个类的名称，我们来查看一下

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-28.png)

  我们发现，这个常量对象指向了常量池中的第14号元素，我们看一下

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-29.png)

  一看我们就明白了，这个14号常量元素指的就是当前我们这个类的全限定名，这样我们就又读懂了一个常量池对象

#### 关于NameAndType中Description的详细介绍

- 我们稍微添加一个方法，然后编译之后读它的NameAndType

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-31.png)

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-32.png)

  我们可以看到，在方法区内存中存在的方法m里面有两个指向，都是指向的常量池中的对象，其中的description是：（I[I[J）[J  ，我们来仔细解析一下，首先，括号内是  I[I[J ,意思是有三个参数，第一个是 int，第二个是int数组，第三个是long数组，括号外面有一个[J，这就是说这个方法的返回值是long数组

  我们看一下所有的description的类型

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-30.png)

### 简单方法的Code阅读

#### 简介code阅读

- 一个方法最重要的部分就是这个方法的code部分，这里面是这个方法的实现，这个方法的实现其实要通过Java的汇编指令来实现

#### init 方法阅读

![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-33.png)

- 我们首先来阅读一下init方法的汇编指令

  首先，指向aload_0指令，我们查看一下

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-34.png)

  aload_n方法的意思就是将本地的一个变量load到操作数栈中，这里我们可以查看一下本地的变量表，看看第0号local variable是什么

  ![image](\images\posts\JVM\2021-03-23-JVM系列(一)之初识JVM和ClassFileFormat-35.png)

  这里我们可以看到，本地变量0号指的就是this对象，这个对象的详细信息在常量池中，并且是第9号元素

  在aload指令结束之后，我们会调用invokespecial指令，这个指令就是构造方法，在构造完毕之后我们就会直接返回