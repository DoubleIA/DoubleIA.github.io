---
layout: post
title: JVM学习笔记：Java内存模型区域
permalink: /jvm/memory_model
---

Java内存模型的运行时数据区域主要包括**程序计数器**、**虚拟机栈**、**本地方法栈**、**Java堆**、**方法区**。下面将依次介绍这些内存区域。此外还将简单介绍一下**运行时常量池**和**直接内存**。

### 程序计数器(Program Counter Register)：

1. 程序计数器是一块较小的内存空间，他的作用可以看成是当前线程执行的字节码得行号指示器。
   通过改变这个计数器的值来选取下一条需要执行的字节码指令。

2. 为了适应Java虚拟机多线程机制，使得线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，
   各条线程之间的计数器相互不影响，独立存储，于是又称这类内存区域为“线程私有”的内存。

3. 当执行的是一个Java方法，该计数器纪录虚拟机字节码指令的地址，如果执行的是一个Native方法，则计数器值为空。

4. 注意，此内存区域是唯一一个在Java虚拟机规范中没有规定任何*OutOfMemoryError*情况的区域。

### 虚拟机栈(Java Virtual Machine Stacks)：

1. 虚拟机栈也是线程私有的，他的生命周期与线程相同。

2. 虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个**栈帧**(Stack Frame)用于存储
   局部变量表、操作栈、动态链接、方法出口等信息。每个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟
   机栈从入到出的过程。

3. 我们通常说的栈内存，其实只对应着虚拟机栈中**局部变量表**部分。局部变量表存放了编译期间可知的各种基本数据类型(
   boolean, byte, char, short, int, float, long, double)、对象引用(reference)和returnAddress类型
   (指向了一条字节码指令地址)。其中long和double会占两个局部变量空间(slot)。

4. 局部变量表所需要的内存空间在编译期间完成分配，方法运行期间不会改变局部变量表的大小。

5. Java虚拟机规范中对这个区域规定了两种异常状况：
   * 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出*StackOverflowError*异常。
   * 如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出*OutOfMemoryError*异常。

### 本地方法栈(Native Method Stacks)：

*  本地方法栈与虚拟机栈所发挥的作用非常相似，区别是本地方法栈为虚拟机使用到的Native方法服务(虚拟机栈为虚拟机
   执行的Java方法服务)。虚拟机可以自由的实现本地方法栈，虚拟机规范并没有对其强制规定。和虚拟机栈一样，本地方
   法栈也会抛出*StackOverflowError*和*OutOfMemoryError*异常。

### Java堆(Java Heap)：

1. Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的
   对象实例都在这里分配内存。然而，随着JIT编译器([Just-In-Time Complier][jit])的发展与[逃逸分析技术][esc]的逐
   渐成熟，栈上分配，标量替换优化技术将会到时一些微妙变化，使得所有对象都分配在堆内存上也渐渐变得不是那么“绝对”
   了。
   [jit]: http://baike.baidu.com/link?url=L6RmWrYImwktiD53JXJaBR48Wdp07rIsP6vW64iY9ETArKcrBrmSen7x1tbgdnUdyHIcs03v6m4OjZ2vjL_6mq
   [esc]: http://www.douban.com/note/217416722/

2. Java堆是垃圾回收器管理的主要区域，因此很多时候也被称作“GC堆”(Garbage Collected Heap)。从内存回收角度看，
   Java堆可以细分为**新生代**和**老生带**；再细致点可以分为Eden空间，From Survivor空间、To Survivor空间等。
   从内存分配的角度看，线程共享的Java堆中可能划分出多个线程私有的**分配缓冲区**(Thread Local Allocation Buffer,TLAB)。划分的目的仅仅是为了更好地回收内存或分配内存，无论哪个存储区域存储的都是实例对象，各区域的分配与回收请看TODO。

3. Java堆可以处于物理上不连续的内存空间，只要是逻辑上连续的即可。既可以实现成固定大小的，也可以是可扩展的，
   主流虚拟机都是按照可扩展来实现的。当然，如果堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出*OutOfMemory*异常。

### 方法区(Method Area)：

1. 方法区也是各个线程共享的内存区域。用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。   虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做**非堆**(Non-Heap)，目的应该是与Java堆区   分开来。同样，当方法区无法满足内存分配的需求是，将抛出*OutOfMemoryError*异常。

2. Java虚拟机规范对这个区域的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选   择不实现垃圾回收。但并非数据进入了方法区就如永久代的名字一样“永久”存在了，这个区域的内存回收目标主要是针对常量池   的回收的类型的卸载。虽然这个区域的回收并不令人满意，对于类型的卸载条件尤其苛刻，但是回收确实是有必要的，许多严重   的BUG就是由于低版本的虚拟机对此区域未完全回收而导致内存泄露。

###运行时常量池(Runtime Constant Pool)：

1. 运行时常量池是方法区的一部分。Class文件除了类的版本、字段、方法、接口等描述信息外，还有一项信息就是**常量池**
   (Constant Pool Table)，用于存在编译期间生成的各种**字面量**和**符号引用**，这部分内容将在类加载后放到方法区的运    行时常量池中。

2. Java虚拟机对Class文件每一部分的格式都有严格规定，每一个字节用于存储哪种数据都必须符合规范上的要求，这样才会被虚 
   机认可、装载、执行。但对于运行时常量池，Java虚拟机规范没有做任何细节的要求，可以按需来实现这个区域。一般来说，还
   会把翻译出来的直接引用也存储在运行时常量池中。

###直接内存(Direct Memory)：

1. 直接内存不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用，而且
   也可能到时*OutOfMemoryError*异常。

2. 像NIO，可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引
   用进行操作，这样在一些场景中可以提高性能，因为避免了在Java堆和Native堆中来回复制数据。

3. 虽然直接内存的分配不会受到Java堆大小的限制，但是，也会受到本机总内存(RAM及SWAP区域或者分页文件)的大小及处理及器
   寻址空间的限制。常犯的错误是，服务器管理员分配虚拟机内存参数时，会根据实际内存设计-Xmx等参数，但经常忽略直接内存
   使得各个区域的总和大于物理内存限制，从而导致动态扩展时出现OutOfMemoryError异常。

###注记：

以上内容均来自《深入理解Java虚拟机》，为个人学习摘录，内容仅做参考，不得用于其他用途。
