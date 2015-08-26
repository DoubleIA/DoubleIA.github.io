---
layout: post
permalink: /jvm/com_tools
title: JDK命令行工具
---

###`jps`：虚拟机进程状况工具

`jps`：JVM Process Status Tool，它的功能和Unix的ps命令类似，可以列出正在运行的虚拟机进程，并显示虚拟机执行主类(Main Class)名称及这些进程的本地虚拟机唯一ID(Local Virtual Machine Identifier, LVMID)。

通过`jps -help`命令可以查看`jps`命令用法：

{% highlight java %}
wangyingbo$ jps -help
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]
{% endhighlight %}

`jps`执行样例：

{% highlight java %}
wangyingbo$ jps -l
6370 sun.tools.jps.Jps
{% endhighlight %}

`jps`可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，hostid为RMI注册表中注册的主机名。

`jps`的其他常用选项如下：

* -q：只输出LVMID，省略主类名称。
* -m：输出虚拟机进程启动时传递给主类main()函数的参数。
* -l：输出主类全名，如果进程执行的是Jar包，输出Jar路径。
* -v：输出虚拟机进程启动时JVM参数。

###`jstat`：虚拟机统计信息监视工具

`jstat`：JVM Statistics Monitoring Tool，它用于监视虚拟机各种运行状态信息的命令行工具。他可以显示本地或者远程虚拟机进程中的**类装载**、**内存**、**垃圾收集**、**JIT编译**等运行数据，在没有GUI图形界面，只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的首选工具。

通过`jstat -help`命令可以查看`jstat`命令用法：

{% highlight java %}
wangyingbo$ jstat -help
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as 
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
{% endhighlight %}

其中的`<option>`可以通过`jstat -options`命令查看：

{% highlight java %}
wangyingbo$ jstat -options
-class
-compiler
-gc
-gccapacity
-gccause
-gcnew
-gcnewcapacity
-gcold
-gcoldcapacity
-gcpermcapacity
-gcutil
-printcompilation
{% endhighlight %}

选项`option`代表用户希望查询虚拟机信息，主要分为3类：**类加载**、**垃圾收集**、**运行期编译状况**，这些参数的意义可以很容易从名字中猜出，不做过多解释了。

对于命令中的VMID需要说明一下，如果是本地虚拟机进程，VMID与LVMID是一致的，如果是远程虚拟机进程，那么VMID的格式应该是：

`[protocol:][//]lvmid[@hostname[:port]/servername]`

参数`interval`和`count`代表查询间隔和次数，如果省略，只查询一次，假设需要每250毫秒查询一次进程2764垃圾收集情况，一共查询20次，那命令为：

`jstat -gc 2764 250 20`

###`jinfo`：Java配置信息工具

`jinfo`：Configuration Info for Java，它的作用是实时查看和调整虚拟机各项参数。如果想知道未被显示指定的虚拟机参数的系统默认值，就可以使用`jinfo`进行查询了，当然JDK1.6或以上版本，也可以使用-XX:+PrintFlagsFinal查看参数默认值。

通过`jinfo -help`命令可以查看`jinfo`命令用法：

{% highlight java %}
wangyingbo$ jinfo -help
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
{% endhighlight %}

可以看出，`jinfo`可以修改一部分运行期可写的虚拟机参数(`name`对应为虚拟机参数名)，也可以把虚拟机进程的`System.getProperties()`内容打印出来(通过`-sysprops`选项)。

###`jmap`：Java内存映像工具

`jmap`：Memory Map for Java，它的作用是生成堆转储快照(一般称为`heapdump`或`dump`文件)。如果不使用`jmap`命令，要想获得Java堆转储快照，还有一些比较暴力的手段，譬如使用-XX:+HeapDumpOnOutOfMemoryError参数，可以让虚拟机发生OOM异常后自动生成`dump`文件，或通过-XX:+HeapDumpOnCtrlBreak参数在使用[Ctrl]+[Break]键让虚拟机生成`dump`文件，又或在Linux下通过`Kill -3`命令发送进程退出信号“吓唬”一下虚拟机，也可以拿到`dump`文件。

通过`jmap -help`命令可以查看`jmap`命令用法：

{% highlight java %}
wangyingbo$ jmap -help
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -permstat            to print permanent generation statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
{% endhighlight %}

可以看出，`jmap`的作用并不仅仅是获得`dump`文件，它还可以查询**`finalize`执行队列**、**Java堆**和**永久代**的详细信息，如**空间使用率**、当前使用的**收集器**等等。

使用`jmap`生成一个正在运行的Eclipse的`dump`快照文件，如果其LVMID是3500，则可以这样运行：

`jmap -dump:format=b,file=eclipse.bin 3500`

###`jhat`：虚拟机堆转储快照分析工具

`jhat`：JVM Heap Analysis Tool，它可以与`jmap`搭配使用，来分析`jmap`生成的堆转储快照。`jhat`内置了一个微型的HTTP/HTML服务器，生成`dump`文件的分析结果后，可以在浏览器中查看(http://localhost:7000)。不过实际当中，除非实在没有别的工具可用，一般不会直接使用`jhat`命令直接分析`dump`文件，主要原因有：

* 其一，一般不会在部署应用程序的服务器上直接分析dump文件，即使可以这样做，也会尽量将`dump`文件复制到其他机器上分析，因为分析工作是一个耗时而且消耗硬件资源的过程，既然都要在其他机器进行，就没必要受到命令工具的限制了。
* 其二，`jhat`的分析功能相对来说比较简陋，一般都会用VisualVM以及专业用户分析`dump`文件的Eclipse Memory Analyzer、IBM HeapAnalyzer等工具进行分析。这些工具比`jhat`更强大，更专业。

使用`jhat -help`命令可以查看`jhat`命令用法：

{% highlight java %}
jhat -help
Usage:  jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

	-J<flag>          Pass <flag> directly to the runtime system. For
			  example, -J-mx512m to use a maximum heap size of 512MB
	-stack false:     Turn off tracking object allocation call stack.
	-refs false:      Turn off tracking of references to objects
	-port <port>:     Set the port for the HTTP server.  Defaults to 7000
	-exclude <file>:  Specify a file that lists data members that should
			  be excluded from the reachableFrom query.
	-baseline <file>: Specify a baseline object dump.  Objects in
			  both heap dumps with the same ID and same class will
			  be marked as not being "new".
	-debug <int>:     Set debug level.
			    0:  No debug output
			    1:  Debug hprof file parsing
			    2:  Debug hprof file parsing, no server
	-version          Report version number
	-h|-help          Print this help and exit
	<file>            The file to read

For a dump file that contains multiple heap dumps,
you may specify which dump in the file
by appending "#<number>" to the file name, i.e. "foo.hprof#3".

All boolean options default to "true"
{% endhighlight %}

###`jstack`：Java堆栈跟踪工具

`jstack`：Stack	Trace for Java，它的作用是生成虚拟机当前时刻的**线程快照**(一般称为`threaddump`或`javacore`文件)。**线程快照**就是当前虚拟机内每一条线程正在执行的方法堆栈集合，生成**线程快照**的主要目的是定位线程出现长时间停顿的原因，如**线程间死锁**、**死循环**、请求外部资源导致的**长时间等待**都是导致线程**长时间停顿**的常见原因。线程出现停顿的时候通过`jstack`来查看各个线程的**调用堆栈**，就可以知道没有响应的线程到底在后台做些什么事情，或者等着什么资源。

可以使用`jstack -help`命令查看`jstack`命令用法：

{% highlight java %}
wangyingbo$ jstack -help
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
{% endhighlight %}

###`HSDIS`：JIT生成代码反汇编

`HSDIS`是Sun官方推荐的HotSpot虚拟机`JIT`编译代码的反汇编插件，包含在HotSpot虚拟机源码之中，但没有提供编译后程序。它的作用是让HotSpot的-XX:+PrintAssembly指令调用它来把动态生成的本地代码还原为汇编代码输出，同时还生成大量十分有价值的注释，这样我们可以通过输出地代码来分析问题。对于Debug版的HotSpot，可以直接通过-XX:+PrintAssembly指令使用插件；如果是Product版的HotSpot，那么还要额外加入一个-XX:+UnlockDiagnosticVMOptions参数。下面看一个例子以演示一下这个插件的使用。

{% highlight java %}
public class Bar {
	int a = 1;
	static int b = 2;
	
	public int sum(int c) {
		return a + b + c;
	}
	
	public static void main(String[] args) {
		new Bar().sum(3);
	}
}
{% endhighlight %}

可以使用如下命令对其进行编译：

`java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*Bar.sum -XX:CompileCommand=compileonly,*Bar.sum com.doubleia.jvm.chp4.Bar`

其中，参数-Xcomp是让虚拟机以编译模式执行代码(在较新的HotSpot中被移除了，需要加个循环预热代码触发JIT编译)，这样可以不需要执行足够次数来预热就可以触发JIT编译。两个-XX:CompileCommand参数意思让编译器不要内联`sum()`并且只编译`sum()`。com.doubleia.jvm.chp4为Bar的package名，编译时需要在project的根目录下进行(即com文件所在目录)。此外，对于mac系统，还需要下载hsdis-am64.dylib，并在~/.bash\_profile中添加export LD\_LIBRARY\_PATH=dir，其中dir为hsdis-am64.dylib文件所在目录，之后注销重新登陆，执行上述命令就可以得到如下输出信息了：

{% highlight java %}
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
CompilerOracle: dontinline *Bar.sum
CompilerOracle: compileonly *Bar.sum
Loaded disassembler from hsdis-amd64.dylib
Decoding compiled method 0x00000001062c7310:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Constants]
  # {method} 'sum' '(I)I' in 'com/doubleia/jvm/chp4/Bar'
  # this:     rsi:rsi   = 'com/doubleia/jvm/chp4/Bar'
  # parm0:    rdx       = int
  #           [sp+0x20]  (sp of caller)
  0x00000001062c7440: mov    0x8(%rsi),%r10d
  0x00000001062c7444: shl    $0x3,%r10
  0x00000001062c7448: cmp    %r10,%rax
  0x00000001062c744b: jne    0x000000010629c960  ;   {runtime_call}
  0x00000001062c7451: data32 xchg %ax,%ax
  0x00000001062c7454: nopl   0x0(%rax,%rax,1)
  0x00000001062c745c: data32 data32 xchg %ax,%ax
[Verified Entry Point]
  0x00000001062c7460: sub    $0x18,%rsp
  0x00000001062c7467: mov    %rbp,0x10(%rsp)    ;*synchronization entry
                                                ; - com.doubleia.jvm.chp4.Bar::sum@-1 (line 14)
  0x00000001062c746c: movabs $0x7aab27ad8,%r10  ;   {oop(a 'java/lang/Class' = 'com/doubleia/jvm/chp4/Bar')}
  0x00000001062c7476: mov    0x58(%r10),%r10d
  0x00000001062c747a: add    0xc(%rsi),%r10d
  0x00000001062c747e: mov    %edx,%eax
  0x00000001062c7480: add    %r10d,%eax         ;*iadd
                                                ; - com.doubleia.jvm.chp4.Bar::sum@9 (line 14)
  0x00000001062c7483: add    $0x10,%rsp
  0x00000001062c7487: pop    %rbp
  0x00000001062c7488: test   %eax,-0x134348e(%rip)        # 0x0000000104f84000
                                                ;   {poll_return}
  0x00000001062c748e: retq
{% endhighlight %}

由于本笔记目的不在汇编，这里不讲述汇编代码的意义了。
