---
layout: post
permalink: /jvm/err_practice
title: JVM学习笔记：OutOfMemoryError异常
---

在Java虚拟机规范描述中，除了程序计数器外，虚拟机内存的其他几个运行时区域都有发生**OutOfMemoryError**(下文简称为OOM)异常的可能，本文将通过实践来演示几个异常发生的场景，并会介绍几个与内存相关的最基本的虚拟机参数，这些测试都是基于HotSpot虚拟机的。建议看本文前先学习一下[JAVA内存模型]({{site.baseurl}}/jvm/memory_model)相关的知识。

本文的目的有两个：一是验证Java虚拟机规范中描述的各个运行时区域存储的内容；二是掌握根据异常信息快速判断是哪个区域内存出现了溢出，以及出现这些异常后该如何处理。

下文中所有的代码片段都注释了执行时所需要设置的虚拟机启动参数(VM Args后跟着的参数)，这些参数对结果有直接影响，在调试代码的时候千万不要忽略。如果使用控制台命令来执行程序，可以在Java命令后直接书写虚拟机参数，而使用Eclipse IDE时，需要在Debug/Run页面中设置虚拟机参数。即Run Configuration中Arguments界面下VM arguments一栏，设置参数为示例：`-verbose:gc -Xmx20m -Xms20m`。

###Java堆溢出：

Java堆用于存储对象实例，只要不断创建对象，并且保证GC Roots到对象之间有可达的路径来避免垃圾回收机制清楚这些对象，那么在对象数量达到最大堆的容量限制后就会产生内存溢出异常。下面这段代码，限制Java堆的大小为20MB，设置堆的最小值-Xms和堆的最大值-Xmx参数同为20MB，可以避免自动扩展，通过参数-XX:+HeapDumpOnOutOfMemoryError可以让虚拟机出现内存溢出异常时Dump出当前的内存堆转储快照以便事后进行分析。
{% highlight java%}
/**
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

	static class OOMObject {
		
	}
	
	public static void main(String[] args) {
		List<OOMObject> list = new ArrayList<OOMObject>();
		
		while(true)
			list.add(new OOMObject());
	}
	
}
{% endhighlight %}

运行结果：

{% highlight java %}
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid43541.hprof ...
Heap dump file created [27586500 bytes in 0.185 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:2245)
	at java.util.Arrays.copyOf(Arrays.java:2219)
	at java.util.ArrayList.grow(ArrayList.java:242)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:216)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:208)
	at java.util.ArrayList.add(ArrayList.java:440)
	at com.doubleia.jvm.chp2.HeapOOM.main(HeapOOM.java:20)
{% endhighlight %}

可以看到，堆内存溢出时，异常堆栈信息“java.lang.OutOfMemoryError”会跟着进一步提示“java heap space”。要解决这个问题，一般的手段是先通过内存映像分析工具(如[Eclipse Memory Analyzer][ana])对Dump出的堆转储快照进行分析。重点在于我们需要确认内存中的对象是否是必要的，也就是分清楚到底是出现了**内存泄露**(Memory Leak)还是出现了**内存溢出**(Memory Overflow)。
[ana]: http://www.tuicool.com/articles/2uIRRrF

如果是内存泄露，可进一步通过工具查看泄露对象到GC Roots的引用链。于是可以找到泄露对象是通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收它们的。有了泄露对象的类型信息及GC Roots引用链的信息，就可以较容易的找到泄露代码的位置。

如果不存在泄露，即内存中的对象确实都还必须存活着，那就应当检查虚拟机堆参数(-Xmx与-Xmx)，与机器物理内存相比是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。这些仅仅是处理Java堆内存问题的简单思路，这些内容的细节可见CHP3。

###虚拟机栈和本地方法栈溢出：

HotSpot虚拟机没有区分虚拟机栈和本地方法栈，所以-Xoss参数(用于设置本地方法栈大小)是无效的，栈容量只能通过-Xss参数设定。对于这两个栈，Java虚拟机规范中描述了两种异常：

* 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出*StackOverflowError*异常。
* 如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出*OutOfMemoryError*异常。

虽然将异常分为两种，看似严谨，但却存在着一些重叠的地方：当栈空间无法继续分配时，到底是内存太小还是已使用的栈空间太大，其本质上只是对同一件事情的两种描述而已。

在单线程实验中，使用-Xss参数减小栈内存容量或定义大量的本地变量，增加此方法帧中本地变量的长度，都无法使虚拟机产生*OutOfMemoryError*异常，得到的结果都是抛出*StackOverflowError*异常，异常出现时输出的堆栈深度相应缩小。使用-Xss参数的测试代码如下：

{% highlight java %}
/**
 * VM Args: -Xss256k
 */
public class JavaVMStackSOF {
	
	private int stackLength = -1;
	
	public void stackLeak() {
		stackLength++;
		stackLeak();
	}
	
	public static void main(String[] args) throws Throwable {
		JavaVMStackSOF oom = new JavaVMStackSOF();
		try {
			oom.stackLeak();
		} catch (Throwable e) {
			System.out.println("stack length:" + oom.stackLength);
			throw e;
		}
	}
	
}
{% endhighlight %}

运行结果：

{% highlight java %}
stack length:1889
Exception in thread "main" java.lang.StackOverflowError
	at com.doubleia.jvm.chp2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:11)
	at com.doubleia.jvm.chp2.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
	......
{% endhighlight %}

实验结果表明：在单个线程下，无论是由于栈帧太大还是虚拟机容量太小，当内存无法分配时，虚拟机抛出的都是*StackOverflowError*异常。

在多线程的实验中，通过不断创建线程的方式可以产生内存溢出异常，但是这样产生的异常与栈空间是否足够大并没有任何联系，甚至在这种情况下，为每个线程分配的内存越大，反而越容易产生内存溢出异常。原因不难理解，操作系统分配给每个进程的内存是有限的，对于虚拟机栈和本地方法栈可用的内存来说，每个线程分配的栈容量越大，可以创建的线程数就越少，建立线程时就越容易把剩下的内存消耗尽。所以，在这种情况下，如果不能减少线程数或者更换64位的虚拟机，就只能通过减少最大堆和减少栈容量来换取更多地线程。这种通过“减少内存”的手段来解决内存溢出的方式会比较难以想到。创建线程导致内存溢出异常的测试代码如下，注意，运行该测试代码可能会导致系统假死和系统卡顿等现象。

{% highlight java %}
/**
 * VM Args: -Xss2M
 */
public class JavaVMStackOOM {
	
	private void dontStop() {
		while (true) {
			
		}
	}
	
	public void stackLeakByThread() {
		while (true) {
			Thread thread = new Thread(new Runnable() {
				
				public void run() {
					dontStop();
				}
			});
			thread.start();
		}
	}
	
	public static void main(String[] args) {
		JavaVMStackOOM oom = new JavaVMStackOOM();
		oom.stackLeakByThread();
	}

}
{% endhighlight %}

执行结果：

{% highlight java %}
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
{% endhighlight %}

###方法区和运行时常量池溢出：

由于运行时常量池是方法区的一部分，这两个区域测试将一起进行。此外，还会观察JDK 1.7逐步“去永久代”对程序的实际影响。

String.intern()是一个Native方法，作用是：如果字符串常量池中已包含一个等于此String对象的字符串，则返回代表池中这个字符串String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。JDK 1.6及以前的版本中，由于常量池分配在永久代内，我们可以通过-XX:PermSize和-XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量，运行时常量池导致的内存溢出测试代码如下：
{% highlight java %}
/**
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class RuntimeConstantPoolOOM {
	
	public static void main(String[] args) {
		//使用List保持常量池引用，避免Full GC回收常量池行为
		List<String> list = new ArrayList<String>();
		//10M的PerSize在integer范围内足够产生OOM了
		int i = 0;
		while (true) {
			list.add(String.valueOf(i).intern());
		}
	}

}

{% endhighlight %}

JDK 1.6环境下的执行结果：

{% highlight java %}
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
	at java.lang.String.intern(Native Method)
	at com.doubleia.jvm.chp2.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:20)
{% endhighlight %}

从运行结果可以看到，提示信息为“PermGen space”，说明运行时常量池属于方法区(虚拟机中的永久代)的一部分。而使用JDK1.7运行这段程序就不会得到相同的结果，while循环将一直运行下去，最后虽然也会抛出*OutOfMemoryError*异常，但提示信息为“Java heap space”。下面这段String.intern()返回引用的测试更加有意思些：

{% highlight java %}
public class RuntieConstantPoolFun {
	
	public static void main(String[] args) {
		String str1 = new StringBuilder("Computer").append(" Science").toString();
		System.out.println(str1.intern() == str1);
		
		String str2 = new StringBuilder("ja").append("va").toString();
		System.out.println(str2.intern() == str2);
	}

}
{% endhighlight %}

JDK 1.7环境下的执行结果：

{% highlight java %}
true
false
{% endhighlight %}

这段代码在JDK 1.6环境下测试，将返回两个false，而在JDK 1.7环境下测试时，会得到一个true和一个false。原因是：在JDK 1.6中，intern()方法会把首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个字符串实例的引用，而由StringBuilder创建的字符串实例在Java堆上，所以必然不是同一个引用，返回false。而JDK 1.7(及部分其他虚拟机)的intern()实现不会再复制实例，只是在常量池中记录首次出现的实例引用，因此intern()返回的引用和由StringBuffer创建的那个字符串实例是同一个。注：java字符串存在于字符串常量池中，不是首次出现，所以intern()返回的引用与str2不同，因此返回false;而Computer Science首次出现，intern返回的引用就是str1，所以返回true。

方法区存放Class相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。对于这本分的测试，基本思路是运行时产生大量的类去填满方法区，直到溢出。下面这个实验借助CGLib直接操作字节码运行时生成了大量的动态类。要注意的是，下面这个实验场景经常会出现在实际应用中，如Spring，Hibernate，在对类进行增强时，都会使用CGLib这类字节码技术，增强的类越多，就需要越大的方法区来保证动态生成的Class可以载入内存。

{% highlight java %}
/**
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMehodAreaOOm {
	
	public static void main(String[] args) {
		while (true) {
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(OOMObject.class);
			enhancer.setUseCache(false);
			enhancer.setCallback(new MethodInterceptor() {
				
				public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
					return proxy.invokeSuper(obj, args);
				}
			});
			enhancer.create();
		}
	}
	
	static class OOMObject {
		
	}

}
{% endhighlight %}

JDK1.7环境下的运行结果：

{% highlight java %}
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
{% endhighlight %}

方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾回收器回收掉，判定条件是比较苛刻的。在经常动态生成大量Class的应用中，需要特别注意类的收回情况。

###本机直接内存溢出：

DirectMemory容量可以通过-XX:MaxDirectMemorySize指定，不指定，则默认Java最大堆(由-Xmx指定)一样，这部分测试没有得到预期的结果，原因还没有搞清楚。不过，由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的内容了。

###总结：

目前已经明白了虚拟机中的内存是如何划分的，哪部分区域、什么样的代码和操作可能导致内存溢出异常。接下来将学习Java垃圾回收机制为了避免内存溢出异常的出现做了哪些努力。
