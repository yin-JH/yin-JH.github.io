---
layout: post
title: JVM系列(五)之JVM Runtime Data Area 大致分布和 JVM Stacks详解
categories: JVM
description: 本篇将会介绍JVM的运行时数据区的分布结构，并且详细讲解其中的JVM Stacks区域，顺带讲解一些常用的JVM指令
keywords: JVM
---

本篇将会介绍JVM的运行时数据区的分布结构，并且详细讲解其中的JVM Stacks区域，顺带讲解一些常用的JVM指令
======

### JVM Runtime Data Area

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-1.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-2.png)

#### Program Counter

- 英文描述
  1. Each Java Virtual Machine thread has its own pc register.
  2. At any point,each Java Vitural Machine thread is executing the code of a single method,namely the current method for that thread
  3. If that method is not native,the pc register contains the address of the Java Virtual Machine instruction currently being executed
- 谷歌机翻
  1. 每个Java虚拟机线程都有其自己的pc寄存器。
  2. 在任何时候，每个Java Vitural Machine线程都在执行单个方法的代码，即该线程的当前方法
  3. 如果该方法不是本机方法，则pc寄存器包含当前正在执行的Java虚拟机指令的地址 
- 程序计数器，简称PC，用处是存放程序执行指令的位置，每一个线程都会拥有一个属于自己的程序计数器，这是因为线程之间会切换如果线程发生切换的话，所有线程共享PC是无法准确记录每个线程各自执行到的指令位置的
- 虚拟机的简单运行逻辑如下

```java
while(not end){
	get PC; //得到当前计数器执行的指令位置
	run;    //执行程序计数器指向的指令
 	PC++;   //程序计数器指向下一条指令
}
```

#### Direct Memory

- 对外内存，通过这个区域可以直接访问计算机的内核空间，实现**零拷贝**
- 以前的JVM在尝试访问内核空间中的数据时，需要将内核空间中的数据拷贝到JVM的空间中，后来进行优化之后，就可以通过Direct Memory来直接访问内核空间中的数据，不再需要拷贝

#### Heap

- 英文描述
  1. The Java Virtual Machine has a *heap* that is shared among all Java Virtual Machine threads. 
  2. The heap is the run-time data area from which memory for all class instances and arrays is allocated.
- 谷歌翻译
  1. Java虚拟机具有一个在所有Java虚拟机线程之间共享的堆。 
  2. 堆是运行时数据区，从中分配了所有类实例和数组的内存。
- 堆空间，一般来说对象就是存储在堆空间的
- 堆空间是所有的线程共享一个堆空间

#### Method Area

- 英文描述
  1. The Java Virtual Machine has a *method area* that is shared among all Java Virtual Machine threads. 
  2.  It stores per-class structures such as the run-time constant pool, field and method data, and the code for methods and constructors, including the special methods used in class and instance initialization and interface initialization.
- 谷歌翻译
  1. Java虚拟机具有一个“方法区域”，该区域在所有Java虚拟机线程之间共享。
  2. 它存储每个类的结构，例如运行时常量池，字段和方法数据，以及方法和构造函数的代码，包括用于类和实例初始化以及接口初始化的特殊方法。
- Method Area，也就是我们平时说的方法区内存；方法区内存是一个称呼，它的实现有两种，一种叫perm-space（永久代）；另一种叫meta space（元数据区）；
- perm-space是jdk1.8之前的实现，meta space是jdk1.8及以后的实现
- perm-space的大小在虚拟机启动的时候通过参数来配置，无法扩容；meta space的大小可以扩容
- 当jdk<1.8时，字符串常量被存储在perm-space中；当jdk>=1.8时，字符串常量被存储在堆空间中
- perm-space不会启动 FGC **这是一个重大的bug**；meta space如果设置了容量上限，那么达到上限后就会启动 FGC

#### JVM Stacks

- 英文描述
  1. Each Java Virtual Machine thread has a private *Java Virtual Machine stack*, created at the same time as the thread.
  2.  A Java Virtual Machine stack stores frames ([§2.6](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6)). 
  3. A Java Virtual Machine stack is analogous to the stack of a conventional language such as C: it holds local variables and partial results, and plays a part in method invocation and return. Because the Java Virtual Machine stack is never manipulated directly except to push and pop frames, frames may be heap allocated. 
  4. The memory for a Java Virtual Machine stack does not need to be contiguous.
- 谷歌机翻
  1. 每个Java虚拟机线程都有一个专用的Java虚拟机堆栈，与该线程同时创建。
  2. Java虚拟机堆栈类似于C之类的常规语言的堆栈：它保存局部变量和部分结果，并在方法调用和返回中起作用。 因为除了推送和弹出帧外，从不直接操纵Java虚拟机堆栈，所以可以为堆分配帧。
  3. Java虚拟机堆栈的内存不必是连续的。
- JVM Stacks 里面存储的是栈帧(frame)，每一个栈帧都指向一个方法，也就是说，每一次线程执行一个方法的时候都会向JVM Stacks里面压入一个栈帧
- 栈帧结构
  1. local variable table：每一个栈帧都有一个局部变量表，局部变量表里面保存了当前栈帧指向的方法里面使用的变量
  2. operand stack：每一个栈帧都有一个操作数栈，这是归属于一个栈帧的小栈，通过这个栈可以真正指向JVM的汇编指令
  3. Dynamic linking：动态链接，顾名思义，这个动态链接其实指向的是这个方法当前的运行时常量池，当我们调用方法的时候，会通过dynamic linking去找到方法的具体实现，如果这个方法的实现在常量池里面还是符号引用的时候，那么我们会将之解析成为动态引用
  4. return address：返回地址，一个方法运行完毕后这个栈帧将会从JVM Stacks弹栈，但是这个方法返回后还有可能JVM Stacks里面还有没有结束的方法，通过return address就可以找到这个方法接下来该执行的指令

#### Native Method Stacks

- 英文描述
  1. An implementation of the Java Virtual Machine may use conventional stacks, colloquially called "C stacks," to support native methods (methods written in a language other than the Java programming language). 
- 谷歌翻译
  1. Java虚拟机的实现可以使用传统的堆栈（俗称“ C堆栈”）来支持本机方法（以不同于Java编程语言的语言编写的方法）。
- 这个栈大致和JVM Stacks一样，区别就在于JVM Stacks调用的是代码中的方法；Native Method Stacks中调用的是本地的C语言提供的方法

### JVM Stacks详细讲解

#### 从Demo了解JVM Stacks工作原理

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-3.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-4.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-5.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-6.png)

- 我们只是稍微调换了以下位置，但是却得到了截然不同的结果，理解了这个结果的原理就基本理解了JVM Stacks

- i = i++;

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-3.png)

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-7.png)

  我们可以看到，在main方法的本地变量表里面有两个本地变量，一个是args，index是0；另一个是i，index是1

  我们可以仔细查看一下JVM汇编指令

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-8.png)

  第一步是 bipush 9，这条指令的意思是将byte 9 转换为 int类型，然后将之push 入operand stack

  第二步是 istore_1，将栈顶的元素弹出，将这个元素存入 本地变量表编号为1的变量中，也就是将9赋值给i

  第三步是 iload_1，将本地变量表里面的编号为1的变量值压入栈，也就是把i的值压入栈，i的值是9

  第四步 iinc 1 by 1，意思是将本地变量表中的index为1的变量增加1，请注意，**这个操作不会压栈弹栈！！**此时操作数栈里面有一个int 9，本地变量表里面的i的值却是10

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-9.png)

  第五步iload_i，将操作数栈栈顶的值弹出，将之覆盖保存在index中

  通过这五个步骤，我们就知道了为什么i=i++;的结果是9

- i=++i;

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-5.png)

  ![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-10.png)

  第一步是bipush 9，和上面一样

  第二步将9弹栈并赋值给i，i的值变为9

  第三步就执行了i++，此时操作数栈中为空

  第四步将i读出来，放到操作数栈，值为10

  第五步将i重新覆盖回去，所以我们可以看到结果是10

#### 通过方法细微变化展现JVM Stacks

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-11.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-12.png)

- 当一个方法不是静态方法的时候，这个方法的本地变量表里面的index 0一定是对象this

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-14.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-13.png)

- 当i的赋予值变为700时就不再是 bipush，而是sipush了，bipush可以理解为byte-->int push; sipush可以理解为 short-->int push，因为700超过了byte能够表达的最大值127，所以就从bipush变成sipush，后面的变换规律一致

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-15.png)

![image](\images\posts\JVM\2021-03-27-JVM系列(五)之JVM Runtime Data Area大致分布和JVM Stacks详解-16.png)

- 我们以前在讲对象创建的时候，有一个指令没有讲解，就是dup指令，这里我们讲解一下：dup的意思是复制，当我们new完一个内存空间后，会往操作数栈中压入一个指向这个对象的引用，dup的意思就是将这个引用复制一下，然后再压栈，这样操作数栈中就有两个引用，此时我们再弹出一个引用，通过这个引用来调用invokespecial指令，调用完毕后操作数栈中就存在一个引用可以指向一个已经被初始化完毕的对象

### 指令集讲解

#### 常见指令集

- push：这个指令集非常的常见，我们使用的bipush x，sipush。push一般就是将一些非本地变量表中的变量（一般是一些没有引用的变量）推入栈中
- store_n：这个指令集的用处就是将栈顶的元素弹出，然后存到本地变量表中，后面的n就是变量在本地变量表中的index，前面有可能会出现一些a、i，例如：istore_n、astore_n，这里面i代表的是int，a代表的是对象
- load_n：这个方法的用处就是将本地变量表中的变量对应的值压入栈中，注意，如果已经压入栈了，那么直接修改本地变量表中变量的值，栈中元素的值是不会改变的，这个方法的前面也可以有i、a，意思和其他的一样
- pop：当一个创建的新对象或者调用方法的返回值没有被引用指向的话，那么这个栈顶变量会直接被弹出

#### invoke指令集

- invokestatic：这个方法光看名字就知道了，当方法调用了static方法的时候，在JVM汇编层面就会调用invokestatic指令
- invokevirtual：这个方法是将栈顶的对象弹出来，然后调用栈顶对象中的方法，这个有点像多态，直接调用的对象的方法，和引用无关，这个指令是在调用多态方法、调用final方法的时候使用的
- invokespecial：这个指令是在调用构造方法（无多态情况），调用对象未重写的方法时使用的指令
- invokeinterface：这个指令的作用就是在调用接口方法的时候使用的指令
- invokedynamic：这个指令非常的复杂，我们这里只做简单的介绍：当我们使用了lambda表达式的时候就会调用这个方法，因为invokedynamic方法可以动态地加载类，我们在使用lambda表达式的时候其实是会动态产生一个内部类的，这个内部类的名字就叫做lambda，在这个内部类的里面还会生成一个内部类，这个内部类就是我们在使用的lambda表达式的时候产生的内部类，invokedynamic就是在这个时候使用的类

