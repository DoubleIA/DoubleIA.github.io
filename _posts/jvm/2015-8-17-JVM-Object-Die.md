---
layout: post
title: JVM学习笔记：对象死了吗？
permalink: /jvm/object_die
---

Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的“高墙”，墙外的人想进去，墙里的人想出来。

GC的历史比Java久远，1960年诞生于MIT的Lisp是第一门真正使用内存动态分配和垃圾收集技术的语言。Lisp还在胚胎时，人们就开始思考GC需要完成的3件事情：1、哪些内存需要回收？2、什么时候回收？3、如何回收？

由于程序计数器、虚拟机栈和本地方法栈随线程生而生，随线程灭而灭，每一个栈帧分配多少内存在类结构确定下来时就已知的，而且不需要过多考虑内存回收的问题，因为方法结束或线程结束时，内存自然就跟着回收了。而Java堆和方法区不一样，这部分的内存分配和回收都是动态的，只有在运行时才能确定，垃圾收集器关注的就是这部分内存。接下来要学习的内存分配和垃圾回收也是针对这一部分的。

我们需要考虑一件事，那就是**对象**已死了吗？当垃圾收集器在堆进行回收前，第一件事情就是确定堆中的对象哪些还“存活”着，哪些已经“死去”。这也是今天要学习的主题。

###引用计数器算法：

引用计数器算法是指：给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值加1；当引用失效时，计数器值减一；任何计数器为0的对象就是不可能再被使用的。**引用计数器算法**(Reference Counting)实现简单，判定效率也很高，也有一些著名的应用案例，但是，Java虚拟机里没有选它来管理内存，其中最主要的原因是它很难解决对象之间循环引用的问题。

举个简单的例子，请看下面这段代码：

{% highlight java %}
/**
 * VM Args: -verbose:gc -Xloggc:gc.log 
 */
public class ReferenceCountingGC {
	
	public Object instance = null;
	
	private static final int _1MB = 1024 * 1024;
	
	/**
	 * 该成员属性唯一意义就是占内存，以便可以在GC日志中看清楚是否被回收过
	 */
	private byte[] bigSize = new byte[2 * _1MB];
	
	public static void testGC() {
		ReferenceCountingGC objA = new ReferenceCountingGC();
		ReferenceCountingGC objB = new ReferenceCountingGC();
		objA.instance = objB;
		objB.instance = objA;
		
		objA = null;
		objB = null;
		
		//假设在这行发生GC，objA和objB是否能被回收？
		System.gc();
	}
	
	public static void main(String[] args) {
		testGC();
	}

}
{% endhighlight %}

注意，执行代码时配置虚拟机参数，使得gc日志输出到gc.log中。在testGC()方法中，对象objA和objB都有字段instance，设置`objA.instance = objB;`和`objB.instance = objA;`后，虽然这两个对象已经不可能再被访问了，但是由于objA和objB互相引用着对方，导致他们的计数都不为0，于是引用计数算法无法通知GC回收他们。接下来我们查看gc.log中的内容：

{% highlight java %}
0.210: [GC 6738K->352K(251392K), 0.0034570 secs]
0.214: [Full GC 352K->284K(251392K), 0.0149290 secs]
{% endhighlight %}

可以看到GC日志中包含“6738K->352K”，意味着虚拟机并没有因为这两个对象互相引用就不在回收他们了，即说明虚拟机不是通过**引用计数器**算法来判断对象是否存活的。

###可达性分析算法：

主流的程序语言(Java, C#, 甚至Lisp)的主流实现中，都是称通过**可达性分析**(Reachability Analysis)来判定对象是否存活的。这个算法的基本思想是通过一系列称为“GC Roots”的对象作为起始点，开始向下搜索，搜索所走过的路线称为**引用链**(Reference Chain)，当一个对象到GC Roots没有任何引用链相连接时，即从GC Roots到这个对象不可达时，则证明此对象是不可用的。

在Java语言中，可作为GC Roots对象包括下面几种：

* 虚拟机栈(栈帧中的本地变量表)中的引用对象。
* 方法区中静态属性引用的对象。
* 方法区中常量引用的对象。
* 本地方法栈中JNI(即一般说的Native方法)引用的对象。

###再谈引用：

不论使用何种算法，判断对象的存活都与“引用”有关。JDK1.2之前将引用定义为：如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，则这块内存代表着一个引用。这种定义很纯粹，太过狭隘。而很多情况下，我们希望可以描述这样一种对象：当内存空间足够时，则能保留在内存中；如果内存空间在进行垃圾收集后还是非常紧张，则可以抛弃这些对象。很多系统的缓存功能都符合这样的应用场景。

在JDK1.2后，Java对引用的概念进行了扩充，将引用分为**强引用**(String Reference)、**软引用**(Soft Reference)、**弱引用**(Weak Reference)、**虚引用**(Phantom Reference)四种，这四种引用强度依次逐渐减弱。

* **强引用**就是指代码之中普遍存在的，类似“Object obj = new Object()”这类的引用，只要强引用存在，垃圾收集器永远不会回收掉被引用的对象。
* **软引用**是用来描述一些还有用但并非必需的对象。对于软引用关联的对象，在系统要发生内存溢出异常之前，会把这些对象列进回收范围内进行二次回收。如果这次回收后还是没有足够的内存，才会抛出内存溢出异常。JDK1.2后，提供了SoftReference类来实现软引用。
* **弱引用**也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当下一次的垃圾收集器工作时，无论内存是否充足，都会回收掉只被弱引用关联的对象。JDK1.2后，提供了WeakReference类来实现弱引用。
* **虚引用**也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能再这个对象被收集器回收时得到一个系统通知。JDK1.2后，提供了PhantomReference类来实现虚引用。

###生存还是死亡：

即使在**可达性分析**算法中不可达对象，也并非非死不可，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那么它将会被第一次标记且进行一次筛选，筛选的条件时此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法时，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象有必要执行finalize()方法，那么这个对象将会放置在一个叫做F-Queue的队列之中，并在稍后由一个虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，这样做的原因是，如果一个对象的finalize()方法中执行缓慢，或者发生了死循环(更极端的情况)，将很有可能会导致F-Queue队列中其他对象永久处于等待，甚至导致整个内存回收系统奔溃。

finalize()方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己(this关键字)赋值给某个类变量或者对象的成员变量，那么在第二次标记时它将被移除“即将回收”的集合；如果对象这个时候还没有逃脱，那么基本上它就真的被回收了。参看下面这段代码，可以看到一个对象的finalize()方法被执行了，但是它仍然可以存活。

{% highlight java %}
/**
 * 此代码演示了两点内容：
 * 1. 对象可以在被GC时自我拯救
 * 2. 这种自救的机会只有一次，因为一个对象的finalize()方法最多只会被系统自动调用一次。
 */
public class FinalizeEscapeGC {
	
	public static FinalizeEscapeGC SAVE_HOOK = null;
	
	public void isAlive() {
		System.out.println("yes, i am still alive :)");
	}
	
	@Override
	protected void finalize() throws Throwable {
		super.finalize();
		System.out.println("finalize method executed!");
		FinalizeEscapeGC.SAVE_HOOK = this;
	}
	
	public static void main(String[] args) throws Throwable {
		SAVE_HOOK = new FinalizeEscapeGC();
		
		//对象第一次成功拯救自己
		SAVE_HOOK = null;
		System.gc();
		//因为finalize方法优先级很低，所以暂停0.5秒以等待它执行
		Thread.sleep(500);
		if (SAVE_HOOK != null) {
			SAVE_HOOK.isAlive();
		} else {
			System.out.println("no, i am dead :(");
		}
		
		//下面这段代码与上面的完全相同，但是这次自救失败了
		SAVE_HOOK = null;
		System.gc();
		//因为finalize方法优先级很低，所以暂停0.5秒以等待它执行
		Thread.sleep(500);
		if (SAVE_HOOK != null) {
			SAVE_HOOK.isAlive();
		} else {
			System.out.println("no, i am dead :(");
		}
		
	}

}
{% endhighlight %}

执行结果：

{% highlight java %}
finalize method executed!
yes, i am still alive :)
no, i am dead :(
{% endhighlight %}

从执行结果可以看出，对于这段完全相同的代码，执行结果却是一次逃脱成功，一次逃脱失败，这是因为任何一个对象的finalize方法都只会被系统调用一次，由于该方法在第一段代码中已经被调用了，因此，第二段代码的自救行动失败了。

值得注意的是，并不鼓励用这种方式拯救对象，应该尽量避免使用它，因为这不是C/C++的析构函数，而是Java诞生时为了使C/C++程序员更容易接受它所做的妥协。它运行代价高昂，不确定性大，无法保证各个对象调用顺序。finalize()能做的所有工作，使用try-finally或者其他方式都可以做的更好、更及时。

###回收方法区：

Java虚拟机规范确实说过可以不要求虚拟机在方法区实现垃圾收集，而且方法区中进行垃圾收集的“性价比”一般比较低：在堆中，常规应用进行一次垃圾收集一般可以回收70%~95%空间，而永久代的垃圾收集效率远低于此。

永久代主要回收两部分内容：**废弃常量**和**无用的类**。判定一个常量是否是“废弃常量”比较简单，回收机制也与回收Java堆中的对象类似。而判定一个类是否是“无用的类”的条件则相对苛刻许多。类要同时满足一下3个条件才能算是“无用的类”：

* 该类所有的实例都已经被回收，也就是Java堆中不存在类的任何实例。
* 加载类的ClassLoader已经被回收。
* 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

虚拟机**可以**对满足了以上三个条件的类进行回收，这里仅仅说的是“可以”，而不是和对象一样，不使用了就必然会回收。是否对类进行回收，HotSpot提供了-Xnoclassgc参数进行控制，还可以使用-verbose:class以及-XX:+TraceClassLoading(在Product版的虚拟机中使用)、-XX:+TraceClassUnLoading(在FastDebug版的虚拟机中使用)查看类加载和卸载信息。

在大量使用反射，动态代理，CGLib等ByteCode框架、动态生成JSP以及OSGi这类频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能，以保证永久代不会溢出。
