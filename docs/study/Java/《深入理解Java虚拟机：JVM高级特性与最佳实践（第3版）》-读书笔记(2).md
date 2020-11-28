# 《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》-读书笔记(2)

# 4. 第四章 虚拟机性能监控、故障处理工具

​	Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。

## 4.1 概述

​	经过前面两章对于虚拟机内存分配与回收技术各方面的介绍，相信读者已经建立了一个比较系统、完整的理论基础。理论总是作为指导实践的工具，把这些知识应用到实际工作中才是我们的最终目的。接下来的两章，我们将从实践的角度去认识虚拟机内存管理的世界。

​	给一个系统定位问题的时候，知识、经验是关键基础，数据是依据，工具是运用知识处理数据的手段。这里说的数据包括但不限于异常堆栈、虚拟机运行日志、垃圾收集器日志、线程快照（threaddump/javacore文件）、堆转储快照（heapdump/hprof文件）等。恰当地使用虚拟机故障处理、分析的工具可以提升我们分析数据、定位并解决问题的效率，但我们在学习工具前，也应当意识到工具永远都是知识技能的一层包装，没有什么工具是“秘密武器”，拥有了就能“包治百病”。

## 4.2 基础故障处理工具

> [jdk bin目录下工具介绍](https://blog.csdn.net/weixin_38658886/article/details/103530538)

​	Java开发人员肯定都知道JDK的bin目录中有java.exe、javac.exe这两个命令行工具，但并非所有程序员都了解过JDK的bin目录下其他各种小工具的作用。随着JDK版本的更迭，这些小工具的数量和功能也在不知不觉地增加与增强。除了编译和运行Java程序外，打包、部署、签名、调试、监控、运维等各种场景都可能会用到它们。

![jdk内置工具](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jb2Rpbmd3aHkuY29tL3VwbG9hZC81L2ltYWdlLzIwMTYwOTA4LzE0NzMzMTY0OTYxMTQwODUuanBn?x-oss-process=image/format,png)

​	在本章，笔者（书籍作者）将介绍这些工具中的一部分，主要是用于**监视虚拟机运行状态和进行故障处理**的工具。这些故障处理工具并不单纯是被Oracle公司作为“礼物”附赠给JDK的使用者，根据软件可用性和授权的不同，可以把它们划分成三类：

+ 商业授权工具：主要是JMC（Java Mission Control）及它要使用到的JFR（Java Flight Recorder），JMC这个原本来自于JRockit的运维监控套件从JDK 7 Update 40开始就被集成到OracleJDK中，JDK 11之前都无须独立下载，但是在商业环境中使用它则是要付费的[^1]。

+ 正式支持工具：这一类工具属于被长期支持的工具，不同平台、不同版本的JDK之间，这类工具可能会略有差异，但是不会出现某一个工具突然消失的情况[^2]。

+ 实验性工具：这一类工具在它们的使用说明中被声明为“没有技术支持，并且是实验性质的”（Unsupported and Experimental）产品，日后可能会转正，也可能会在某个JDK版本中无声无息地消失。但事实上它们通常都非常稳定而且功能强大，也能在处理应用程序性能问题、定位故障时发挥很大的作用。

​	*注意：　如果读者在工作中需要监控运行于JDK 5的虚拟机之上的程序，在程序启动时请添加参数“`-Dcom.sun.management.jmxremote`”开启JMX管理功能，否则由于大部分工具都是基于或者要用到JMX（包括下一节的可视化工具），它们都将无法使用，如果被监控程序运行于JDK 6或以上版本的虚拟机之上，那JMX管理默认是开启的，虚拟机启动时无须再添加任何参数。*

[^1]:无论是GPL、BCL还是OTN协议，JMC在个人开发环境中使用是免费的。
[^2]:这并不意味着永久存在，只是被移除前会有“deprecated”的过渡期，正式工具被移除的数量并不比实验性工具来得少。

### 4.2.1 jps：虚拟机进程状况工具

​	*<small>JDK的很多小工具的名字都参考了UNIX命令的命名方式，**jps（JVM Process Status Tool）**是其中的典型。除了名字像UNIX的ps命令之外，它的功能也和ps命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）。虽然功能比较单一，但它绝对是使用频率最高的JDK命令行工具，因为其他的JDK工具大多需要输入它查询到的LVMID来确定要监控的是哪一个虚拟机进程。**对于本地虚拟机进程来说，LVMID与操作系统的进程ID（PID，Process Identifier）是一致的**，使用Windows的任务管理器或者UNIX的ps命令也可以查询到虚拟机进程的LVMID，<u>但如果同时启动了多个虚拟机进程，无法根据进程名称定位时，那就必须依赖jps命令显示主类的功能才能区分了</u>。</small>*

jps命令格式：

```shell
jps [ options ] [ hostid ]
```

jps执行样例：

```shell
jps -l
2388 D:\Develop\glassfish\bin\..\modules\admin-cli.jar
2764 com.sun.enterprise.glassfish.bootstrap.ASMain
3788 sun.tools.jps.Jps
```

jps还可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，参数hostid为RMI注册表中

注册的主机名。jps的其他常用选项见下表。

| 选项 | 作用                                                 |
| ---- | ---------------------------------------------------- |
| -q   | 只输出LVMID，省略主类的名称                          |
| -m   | 输出虚拟机进程启动时传递给主类main()函数的参数       |
| -l   | 输出主类的全名，如果进程执行的是jar包，则输出JAR路径 |
| -v   | 输出虚拟机进程启动时的JVM参数                        |

> [分布式架构基础:Java RMI详解](https://www.jianshu.com/p/de85fad05dcb)

### 4.2.2 jstat：虚拟机统计信息监视工具

​	**jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具**。它可以显示本地或者远程[^3]虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据，在没有GUI图形界面、只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的常用工具。

​	jstat命令格式为：

```shell
jstat [ option vmid [interval[s|ms] [count]] ]
```

​	对于命令格式中的VMID与LVMID需要特别说明一下：如果是本地虚拟机进程，VMID与LVMID是一致的；如果是远程虚拟机进程，那VMID的格式应当是：

```shell
[protocol:][//]lvmid[@hostname[:port]/servername]
```

​	参数interval和count代表查询间隔和次数，如果省略这2个参数，说明只查询一次。假设需要每250毫秒查询一次进程2764垃圾收集状况，一共查询20次，那命令应当是：

```shell
jstat -gc 2764 250 20
```

​	选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况。详细请参考表4-2中的描述。

| 选项               | 作用                                                         |
| ------------------ | ------------------------------------------------------------ |
| -class             | 监视类加载、卸载数量、总空间以及类装载所耗费的时间           |
| -gc                | 监视Java堆状况，包括Eden区、2个Survivor区、老年代、永久代等的容量、已用空间，垃圾收集时间合集等信息 |
| -gccapacity        | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间 |
| -gcutil            | 监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比 |
| -gccause           | 与-gcutil功能一样，但是会额外输出导致上一次垃圾收集产生的原因 |
| -gcnew             | 监视新生代垃圾收集状况                                       |
| -gcnewcapacity     | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间 |
| -gcold             | 监视老年代垃圾收集状况                                       |
| -gcoldcapacity     | 监视内容与-gcold基本相同，输出主要关注使用到的最大、最小空间 |
| -gcpermcapacity    | 输出永久代使用到的最大、最小空间                             |
| -compiler          | 输出即使编译器编译过的方法、耗时等信息                       |
| -printcomplication | 输出已经被即使编译的方法                                     |

​	jstat监视选项众多，囿于版面原因无法逐一演示，这里仅举一个在命令行下监视一台刚刚启动的GlassFish v3服务器的内存状况的例子，用以演示如何查看监视结果。监视参数与输出结果如代码清单4-1所示。

​	代码清单4-1　jstat执行样例

```shell
jstat -gcutil 2764
S0 S1 E O P YGC YGCT FGC FGCT GCT
0.00 0.00 6.20 41.42 47.20 16 0.105 3 0.472 0.577

# 个人尝试
[root@iZuf6akazugpwb7oouu4grZ ~]# jstat -gcutil 1691
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00   8.05  29.02  62.87  87.51  75.78   8347   61.361    14    3.622   64.983

[root@iZuf6akazugpwb7oouu4grZ ~]# jstat -gcold 1691
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT   
118144.0 103388.5  15488.0  11736.9    172860.0    108684.8   8347    14    3.622   64.983

[root@iZuf6akazugpwb7oouu4grZ ~]# jstat -compiler 1691
Compiled Failed Invalid   Time   FailedType FailedMethod
   23697      2       0   414.66          1 sun/security/util/math/intpoly/IntegerPolynomialP521 carryReduce
```

​	查询结果表明：这台服务器的新生代Eden区（E，表示Eden）使用了6.2%的空间，2个Survivor区（S0、S1，表示Survivor0、Survivor1）里面都是空的，老年代（O，表示Old）和永久代（P，表示Permanent）则分别使用了41.42%和47.20%的空间。程序运行以来共发生Minor GC（YGC，表示YoungGC）16次，总耗时0.105秒；发生Full GC（FGC，表示Full GC）3次，总耗时（FGCT，表示Full GCTime）为0.472秒；所有GC总耗时（GCT，表示GC Time）为0.577秒。

​	使用jstat工具在纯文本状态下监视虚拟机状态的变化，在用户体验上也许不如后文将会提到的JMC、VisualVM等可视化的监视工具直接以图表展现那样直观，但在实际生产环境中不一定可以使用图形界面，而且多数服务器管理员也都已经习惯了在文本控制台工作，直接在控制台中使用jstat命令依然是一种常用的监控方式。

[^3]:需要远程主机提供RMI支持，JDK中提供了jstatd工具可以很方便地建立远程RMI服务器。

### 4.2.3 info: Java配置信息工具

​	**jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数**。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了（如果只限于JDK 6或以上版本的话，使用`java -XX：+PrintFlagsFinal`查看参数默认值也是一个很好的选择）。jinfo还可以使用-sysprops选项把虚拟机进程的`System.getProperties()`的内容打印出来。这个命令在JDK 5时期已经随着Linux版的JDK发布，当时只提供了信息查询的功能，JDK 6之后，jinfo在Windows和Linux平台都有提供，并且加入了在运行期修改部分参数值的能力（可以使用-flag[+|-]name或者-flag name=value在运行期修改一部分运行期可写的虚拟机参数值）。在JDK 6中，jinfo对于Windows平台功能仍然有较大限制，只提供了最基本的-flag选项。

​	jinfo命令格式：

```shell
jinfo [ option ] pid
```

​	执行样例：查询CMSInitiatingOccupancyFraction参数值

```shell
jinfo -flag CMSInitiatingOccupancyFraction 1444
-XX:CMSInitiatingOccupancyFraction=85

# 个人尝试1
[root@iZuf6akazugpwb7oouu4grZ ~]# jinfo 4384
Attaching to process ID 4384, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.271-b09
Java System Properties:

java.runtime.name = Java(TM) SE Runtime Environment
java.vm.version = 25.271-b09
sun.boot.library.path = /usr/etc/jdk1.8.0_271/jre/lib/amd64
java.protocol.handler.pkgs = org.springframework.boot.loader
java.vendor.url = http://java.oracle.com/
java.vm.vendor = Oracle Corporation
path.separator = :
file.encoding.pkg = sun.io
java.vm.name = Java HotSpot(TM) 64-Bit Server VM
sun.os.patch.level = unknown
sun.java.launcher = SUN_STANDARD
user.country = US
user.dir = /opt/jenkins_jars
java.vm.specification.name = Java Virtual Machine Specification
PID = 4384
java.runtime.version = 1.8.0_271-b09
java.awt.graphicsenv = sun.awt.X11GraphicsEnvironment
os.arch = amd64
java.endorsed.dirs = /usr/etc/jdk1.8.0_271/jre/lib/endorsed
line.separator = 

java.io.tmpdir = /tmp
java.vm.specification.vendor = Oracle Corporation
os.name = Linux
sun.jnu.encoding = UTF-8
java.library.path = /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
spring.beaninfo.ignore = true
java.specification.name = Java Platform API Specification
java.class.version = 52.0
sun.management.compiler = HotSpot 64-Bit Tiered Compilers
os.version = 3.10.0-957.27.2.el7.x86_64
LOG_FILE = logs/spring.log
user.home = /root
user.timezone = Asia/Shanghai
catalina.useNaming = false
java.awt.printerjob = sun.print.PSPrinterJob
file.encoding = UTF-8
java.specification.version = 1.8
catalina.home = /tmp/tomcat.1306879443728080690.8081
user.name = root
java.class.path = /opt/jenkins_jars/qidao_demo_001-0.0.1-SNAPSHOT.jar
java.vm.specification.version = 1.8
sun.arch.data.model = 64
sun.java.command = /opt/jenkins_jars/qidao_demo_001-0.0.1-SNAPSHOT.jar
java.home = /usr/etc/jdk1.8.0_271/jre
user.language = en
java.specification.vendor = Oracle Corporation
awt.toolkit = sun.awt.X11.XToolkit
java.vm.info = mixed mode
java.version = 1.8.0_271
java.ext.dirs = /usr/etc/jdk1.8.0_271/jre/lib/ext:/usr/java/packages/lib/ext
sun.boot.class.path = /usr/etc/jdk1.8.0_271/jre/lib/resources.jar:/usr/etc/jdk1.8.0_271/jre/lib/rt.jar:/usr/etc/jdk1.8.0_271/jre/lib/sunrsasign.jar:/usr/etc/jdk1.8.0_271/jre/lib/jsse.jar:/usr/etc/jdk1.8.0_271/jre/lib/jce.jar:/usr/etc/jdk1.8.0_271/jre/lib/charsets.jar:/usr/etc/jdk1.8.0_271/jre/lib/jfr.jar:/usr/etc/jdk1.8.0_271/jre/classes
java.awt.headless = true
java.vendor = Oracle Corporation
catalina.base = /tmp/tomcat.1306879443728080690.8081
file.separator = /
java.vendor.url.bug = http://bugreport.sun.com/bugreport/
sun.io.unicode.encoding = UnicodeLittle
sun.cpu.endian = little
LOG_PATH = logs
sun.cpu.isalist = 

VM Flags:
Non-default VM flags: -XX:CICompilerCount=2 -XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=2147483648 -XX:MaxNewSize=715784192 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=357892096 -XX:OldSize=715849728 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseGCOverheadLimit 
Command line:  -XX:-UseGCOverheadLimit -Xms1024m -Xmx2048m

# 个人尝试2
[root@iZuf6akazugpwb7oouu4grZ ~]# jinfo -flag CMSInitiatingOccupancyFraction 4384
-XX:CMSInitiatingOccupancyFraction=-1
```

### 4.2.4 jmap：Java内存映像工具

​	**jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文件）**。如果不使用jmap命令，要想获取Java堆转储快照也还有一些比较“暴力”的手段：譬如在第2章中用过的`-XX：+HeapDumpOnOutOfMemoryError`参数，可以让虚拟机在内存溢出异常出现之后自动生成堆转储快照文件，通过`-XX：+HeapDumpOnCtrlBreak`参数则可以使用[Ctrl]+[Break]键让虚拟机生成堆转储快照文件，又或者在Linux系统下通过Kill-3命令发送进程退出信号“恐吓”一下虚拟机，也能顺利拿到堆转储快照。

​	**jmap的作用并不仅仅是为了获取堆转储快照，它还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等。**

​	和jinfo命令一样，jmap有部分功能在Windows平台下是受限的，除了生成堆转储快照的-dump选项和用于查看每个类的实例、空间占用统计的-histo选项在所有操作系统中都可以使用之外，其余选项都只能在Linux/Solaris中使用。

​	jmap命令格式：

```shell
jmap [ option ] vmid
```

option选项的合法值与具体含义如下表所示。

| 选项           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| -dump          | 生成Java堆转储快照。格式为-dump:[live.]format=b.file=\<filename\>，其中live子参数说明是否只dump出存活的对象 |
| -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在Linux/Solaris平台下有效 |
| -heap          | 显示Java堆详细信息，如使用哪种回收器、参数配置、分代状况等。只在Linux/Solaris平台下有效 |
| -histo         | 显示堆中对象统计信息，包括类、实例数量、合计容量             |
| -permstat      | 以ClassLoader为统计口径显示永久代内存状态。只在Linux/Solaris平台下有效 |
| -F             | 当虚拟机进程对-dump选项没有响应时，可使用这个选项强制生成dump快照。只在Linux/Solaris平台下有效 |

​	代码清单4-2是使用jmap生成一个正在运行的Eclipse的堆转储快照文件的例子，例子中的3500是通过jps命令查询到的LVMID。

​	代码清单4-2　使用jmap生成dump文件

```shell
jmap -dump:format=b,file=eclipse.bin 3500
Dumping heap to C:\Users\IcyFenix\eclipse.bin ...
Heap dump file created

# 个人尝试1 
[root@iZuf6akazugpwb7oouu4grZ ~]# jmap -heap 4384
Attaching to process ID 4384, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.271-b09

using thread-local object allocation.
Mark Sweep Compact GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 357892096 (341.3125MB)
   MaxNewSize               = 715784192 (682.625MB)
   OldSize                  = 715849728 (682.6875MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 322174976 (307.25MB)
   used     = 139136328 (132.69074249267578MB)
   free     = 183038648 (174.55925750732422MB)
   43.186572007380235% used
Eden Space:
   capacity = 286392320 (273.125MB)
   used     = 111086816 (105.94064331054688MB)
   free     = 175305504 (167.18435668945312MB)
   38.78833622354119% used
From Space:
   capacity = 35782656 (34.125MB)
   used     = 28049512 (26.750099182128906MB)
   free     = 7733144 (7.374900817871094MB)
   78.38856903187958% used
To Space:
   capacity = 35782656 (34.125MB)
   used     = 0 (0.0MB)
   free     = 35782656 (34.125MB)
   0.0% used
tenured generation:
   capacity = 715849728 (682.6875MB)
   used     = 42424808 (40.459449768066406MB)
   free     = 673424920 (642.2280502319336MB)
   5.92649634980374% used

22967 interned Strings occupying 2399168 bytes.

# 个人尝试2
[root@iZuf6akazugpwb7oouu4grZ ~]# jmap -dump:format=b,file=/usr/etc/tmp.bin 4384
Dumping heap to /usr/etc/tmp.bin ...
File exists
```

### 4.2.5 jhat：虚拟机堆转储快照分析工具

> [虚拟机性能监控与故障处理 — 命令行工具 — JVM系列(十一)](https://blog.csdn.net/jiangxiulilinux/article/details/105507456)

​	JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/Web服务器，生成堆转储快照的分析结果后，可以在浏览器中查看。**不过实事求是地说，在实际工作中，除非手上真的没有别的工具可用，否则多数人是不会直接使用jhat命令来分析堆转储快照文件的**，主要原因有两个方面。一是一般不会在部署应用程序的服务器上直接分析堆转储快照，即使可以这样做，也会尽量将堆转储快照文件复制到其他机器[^4]上进行分析，因为分析工作是一个耗时而且极为耗费硬件资源的过程，既然都要在其他机器上进行，就没有必要再受命令行工具的限制了。另外一个原因是jhat的分析功能相对来说比较简陋，后文将会介绍到的VisualVM，以及专业用于分析堆转储快照文件的Eclipse Memory Analyzer、IBM HeapAnalyzer[^5]等工具，都能实现比jhat更强大专业的分析功能。代码清单4-3演示了使用jhat分析上一节采用jmap生成的Eclipse IDE的内存快照文件。

​	代码清单4-3　使用jhat分析dump文件

```shell
jhat eclipse.bin
Reading from eclipse.bin...
Dump file created Fri Nov 19 22:07:21 CST 2010
Snapshot read, resolving...
Resolving 1225951 objects...
Chasing references, expect 245 dots....
Eliminating duplicate references...
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

​	屏幕显示“Server is ready.”的提示后，用户在浏览器中输入http://localhost:7000/可以看到分析结果。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200414110441348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5neGl1bGlsaW51eA==,size_16,color_FFFFFF,t_70)

​	分析结果默认以包为单位进行分组显示，分析内存泄漏问题主要会使用到其中的“HeapHistogram”（与jmap-histo功能一样）与OQL页签的功能，前者可以找到内存中总容量最大的对象，后者是标准的对象查询语言，使用类似SQL的语法对内存中的对象进行查询统计。如果读者需要了解具体OQL的语法和使用方法，可参见本书附录D的内容。

[^4]:用于分析的机器一般也是服务器，由于加载dump快照文件需要比生成dump更大的内存，所以一般在64位JDK、大内存的服务器上进行。
[^5]:IBM HeapAnalyzer用于分析IBM J9虚拟机生成的映像文件，各个虚拟机产生的映像文件格式并不一致，所以分析工具也不能通用。

### 4.2.6 jstack：Java堆栈跟踪工具

​	**jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）**。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，**<u>生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因</u>**。线程出现停顿时通过jstack来查看各个线程的调用堆栈，就可以获知没有响应的线程到底在后台做些什么事情，或者等待着什么资源。

​	jstack命令格式：

```shell
jstack [ option ] vmid
```

​	option选项的合法值与具体含义如表4-4所示。

| 选项 | 作用                                         |
| ---- | -------------------------------------------- |
| -F   | 当正常输出的请求不被响应时，强制输出线程堆栈 |
| -l   | 除堆栈外，显示关于锁的附加信息               |
| -m   | 如果调用到本地方法的话，可以显示C/C++的堆栈  |

​	代码清单4-4是使用jstack查看Eclipse线程堆栈的例子，例子中的3500是通过jps命令查询到的LVMID。

​	代码清单4-4　使用jstack查看线程堆栈（部分结果）

```shell
jstack -l 3500
2010-11-19 23:11:26
Full thread dump Java HotSpot(TM) 64-Bit Server VM (17.1-b03 mixed mode):
"[ThreadPool Manager] - Idle Thread" daemon prio=6 tid=0x0000000039dd4000 nid= 0xf50 in Object.wait() [0x000000003c96f000]
java.lang.Thread.State: WAITING (on object monitor)
at java.lang.Object.wait(Native Method)
- waiting on <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
at java.lang.Object.wait(Object.java:485)
at org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor.run (Executor. java:106)
- locked <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
Locked ownable synchronizers:
- None


# 个人尝试1
jstack -l 4384
2020-11-28 16:25:00
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.271-b09 mixed mode):

"Attach Listener" #38 daemon prio=9 os_prio=0 tid=0x00007fad8400b000 nid=0x3c8e waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"logback-8" #37 daemon prio=5 os_prio=0 tid=0x00007fad7c099800 nid=0x5288 waiting on condition [0x00007fad788d5000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000ab20b648> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1081)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None

"logback-7" #36 daemon prio=5 os_prio=0 tid=0x00007fad7c9b5800 nid=0x5287 waiting on condition [0x00007fad789d6000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000ab20b648> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1081)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- None
// ...... 省略
"Service Thread" #7 daemon prio=9 os_prio=0 tid=0x00007fadac0db800 nid=0x1129 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C1 CompilerThread1" #6 daemon prio=9 os_prio=0 tid=0x00007fadac0d8800 nid=0x1128 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C2 CompilerThread0" #5 daemon prio=9 os_prio=0 tid=0x00007fadac0d6000 nid=0x1127 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Signal Dispatcher" #4 daemon prio=9 os_prio=0 tid=0x00007fadac0d4800 nid=0x1126 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Finalizer" #3 daemon prio=8 os_prio=0 tid=0x00007fadac0a3800 nid=0x1125 in Object.wait() [0x00007fadb0140000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
	- locked <0x00000000ab20c0e0> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:216)

   Locked ownable synchronizers:
	- None

"Reference Handler" #2 daemon prio=10 os_prio=0 tid=0x00007fadac09f000 nid=0x1124 in Object.wait() [0x00007fadb0241000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
	- locked <0x00000000ab20c298> (a java.lang.ref.Reference$Lock)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

   Locked ownable synchronizers:
	- None

"VM Thread" os_prio=0 tid=0x00007fadac095000 nid=0x1123 runnable 

"VM Periodic Task Thread" os_prio=0 tid=0x00007fadac0de000 nid=0x112a waiting on condition 

JNI global references: 2047
```

​	从JDK 5起，java.lang.Thread类新增了一个`getAllStackTraces()`方法用于获取虚拟机中所有线程的StackTraceElement对象。使用这个方法可以通过简单的几行代码完成jstack的大部分功能，在实际项目中不妨调用这个方法做个管理员页面，可以随时使用浏览器来查看线程堆栈，如代码清单4-5所示，这也算是笔者（书籍作者）的一个小经验。

​	代码清单4-5　查看线程状况的JSP页面

```jsp
<%@ page import="java.util.Map"%>
<html>
  <head>
    <title>服务器线程信息</title>
  </head>
  <body>
    <pre>
<%
for (Map.Entry<Thread, StackTraceElement[]> stackTrace : Thread.getAllStackTraces().entrySet()) {
  Thread thread = (Thread) stackTrace.getKey();
  StackTraceElement[] stack = (StackTraceElement[]) stackTrace.getValue();
  if (thread.equals(Thread.currentThread())) {
    continue;
  }
  out.print("\n线程：" + thread.getName() + "\n");
  for (StackTraceElement element : stack) {
    out.print("\t"+element+"\n");
  }
}
%>
</pre>
  </body>
</html>
```

> [What is lvmid in java?](https://stackoverflow.com/questions/4375028/what-is-lvmid-in-java)

### 4.2.7 基础工具总结

​	下面表4-5～表4-14中罗列了JDK附带的全部（包括曾经存在但已经在最新版本中被移除的）工具及其简要用途，限于篇幅，本节只讲解了6个常用的命令行工具。笔者选择这几个工具除了因为它们是最基础的命令外，还因为它们已经有很长的历史，能适用于大多数读者工作、学习中使用的JDK版本。在高版本的JDK中，这些工具大多已有了功能更为强大的替代品，譬如JCMD、JHSDB的命令行模式，但使用方法也是相似的，无论JDK发展到了什么版本，学习这些基础的工具命令并不会过时和浪费。

+ 基础工具：用于支持基本的程序创建和运行

  | 名称         | 主要作用                                                 |
  | ------------ | -------------------------------------------------------- |
  | appletviewer | 在不使用Web浏览器的情况下运行和调试Applet，JDK11中被移除 |
  | extcheck     | 检查JAR冲突的工具，从JDK9中被移除                        |
  | jar          | 创建和管理JAR文件                                        |
  | java         | Java运行Class文件或JAR文件                               |
  | javac        | 用于Java编程语言的编译器                                 |
  | javadoc      | Java的API文档生成器                                      |
  | javah        | C语言头文件和Stub函数生成器，用于编写JNI方法             |
  | javap        | Java字节码分析工具                                       |
  | jlink        | 将Module和它的依赖打包成一个运行时镜像文件               |
  | jdb          | 基于JPDA协议的调试器，以类似于GDB的方式进行调试Java代码  |
  | jdeps        | Java类依赖性分析器                                       |
  | jdbprscan    | 用于搜索JAR包中使用了"deprecated"的类，从JDK9开始提供    |

+ 安全：用于程序签名、设置安全测试等

  | 名称       | 主要作用                                                     |
  | ---------- | ------------------------------------------------------------ |
  | keytool    | 管理密钥库和证书。主要用于获取或缓存Kerberos协议的票据授权票据。允许用户查看本地凭据缓存和密钥表中的条目（用于Kerberos协议） |
  | jarsigner  | 生成并验证JAR签名                                            |
  | policytool | 管理策略文件的GUI工具，用于管理用户策略文件（.java.policy），在JDK10中被移除 |

+ 国际化：用于创建本地语言文件

  | 名称         | 主要作用                                                     |
  | ------------ | ------------------------------------------------------------ |
  | native2ascii | 本地编码到ASCII编码的转换器（Native-to-ASCII Converter），用于"任意受支持的字符编码"和与之对应的"ASCII编码和Unicode转义"之间的相互转换 |

+ **远程方法调用**：用于跨Web或网络的服务交互

  | 名称        | 主要作用                                                     |
  | ----------- | ------------------------------------------------------------ |
  | rmic        | Java **RMI**编辑器，为使用JRMP或IIOP协议的远程对象生成Stub、Skeleton和Tie类，也用于生成OMG IDL |
  | rmiregistry | 远程对象注册表服务，用于当前主机的指定端口上创建并启动一个远程对象注册表 |
  | rmid        | 启动激活系统守护进程，允许在虚拟机中注册或激活对象           |
  | serialver   | 生成并返回指定类的序列化版本ID                               |

+ Java IDL与RMI-IIOP：在JDK11中结束了十余年的CORBA支持，这些工具不再提供[^6]。

  | 名称       | 主要作用                                                     |
  | ---------- | ------------------------------------------------------------ |
  | tnameserv  | 提供对命名服务的访问                                         |
  | idlj       | IDL转Java编译器（IDL-to-Java Compiler），生成映射OMG IDL接口的Java源文件，并启用以Java编程语言编写的使用CORBA功能的应用程序的Java源文件。IDL意即接口定义语言（Interface Definition Language） |
  | orbd       | 对象请求代理守护进程（Object Request Broker Daemon），提供从客户端查找和调用CORBA环境服务端上的持久化对象的功能。使用ORBD代替瞬态命名服务tnameserv。ORBD包括瞬态命名服务和持久命名服务。ORBD工具集成了服务器管理器、互操作命名服务和引导名称服务器的功能。当客户端想进行服务器时定位、注册和激活功能时，可以与servertool一起使用 |
  | servertool | 为应用程序注册、注销、启动和关闭服务器提供易用的接口         |

+ 部署工具：用于程序打包、发布和部署

  | 名称         | 主要作用                                                     |
  | ------------ | ------------------------------------------------------------ |
  | javapackager | 打包、签名Java和JavaFX应用程序，在JDK11中被移除              |
  | pack200      | 使用Java GZIP压缩器将JAR文件转换为压缩的Pack200文件。压缩的压缩文件是高度压缩的JAR，可以直接部署，节省带宽并减少下载时间 |
  | unpack200    | 将Pack200生成的打包文件解压提取成JAR文件                     |

+ Java Web Srart

  | 名称   | 主要作用                                                |
  | ------ | ------------------------------------------------------- |
  | javaws | 启动Java Web Start并设置各种选项的工具。在JDK11中被移除 |

+ 性能监控和故障处理：用于监控分析Java虚拟机运行信息，排查问题

  | 名称      | 主要作用                                                     |
  | --------- | ------------------------------------------------------------ |
  | jps       | JVM Process Status Tool，显示指定系统内所有的HotSpot虚拟机进程 |
  | jstat     | JVM Statistics Monitoring Tool，用于收集Hotspot虚拟机各方面的运行数据 |
  | jstatd    | JVM Statistics Monitoring Tool Daemon，jstat的守护程序，启动一个RMI服务器应用程序，用于监视测试的HotSpot虚拟机的创建和终止，并提供一个界面，允许远程监控工具附加到在本地系统上运行的虚拟机。在JDK9中集成到了JHSDB中 |
  | jinfo     | Configuration Info for Java，生成虚拟机的内存转存快照（heapdump文件）。在JDK9中集成到了JHSDB中 |
  | jhat      | JVM Heap Analysis Tool，用于分析堆转存快照，它会建立一个HTTP/Web服务器，让用户可以在浏览器上查看分析结果。在JDK9中被JHSDB代替 |
  | jstack    | Stack Trace for Java，显示虚拟机的线程快照。在JDK9中集成了JHSDB中 |
  | jhsdb     | Java HotSpot Debugger，一个基于Serviceability Agent的HotSpot进程调试器，从JDK9开始提供 |
  | jsadebugd | Java Serviceability Agent Debug Daemon，适用于Java的可维护代理调试守护程序，主要用于附加到指定的Java进程、核心文件，或充当一个调试服务器 |
  | jcmd      | JVM Command，虚拟机诊断命令工具，将诊断命令请求发送到正在运行的Java虚拟机。从JDK7开始提供 |
  | jconsole  | Java Console，用于监控Java虚拟机的使用JMX规范的图形工具。它可以监控本地和远程Java虚拟机，还可以监控和管理应用程序 |
  | jmc       | Java Mission Control，包含用于监控和管理Java应用程序的工具，而不会引入与这些工具相关联的性能开销。开发者可以使用jmc命令来创建JMC工具，从JDK7 Update 40开始集成到OracleJDK中 |
  | jvisualvm | Java VisualVM，一种图形化工具，可在Java虚拟机中运行时提供有关基于Java技术的应用程序（Java应用程序）的详细信息。Java VisualVM提供内存和CPU分析、堆转存分析、内存泄漏检测、MBean访问和垃圾收集。从JDK6 Update7开始提供；从JDK9开始不再打包入JDK中，但仍保持更新发展，可以独立下载 |

+ WebService工具：与CORBA一起在JDK 11中被移除

  | 名称      | 主要作用                                                     |
  | --------- | ------------------------------------------------------------ |
  | schemagen | 用于XML绑定的Schema生成器，用于生成XML Schema文件            |
  | wsgen     | XML Web Service 2.0 的Java API，生成用于JAX-WS Web Service 的JAX-WS便携式产物 |
  | wsimport  | XML Web Service 2.0 的Java API，主要用于根据服务端发布的WSDL文件生成客户端 |
  | xjc       | 主要用于根据XML Schema文件生成对应的Java类                   |

+ REPL和脚本工具

  | 名称       | 主要作用                                                     |
  | ---------- | ------------------------------------------------------------ |
  | jshell     | 基于Java的Shell REPL（Read-Eval-Print Loop）交互工具         |
  | jjs        | 对Nashorn引擎的调用入口。Nashorn是基于Java实现的一个轻量级高性能JavaScript运行环境 |
  | jrunscript | Java命令行脚本外壳攻击（Command Line Script Shell），主要用于解释执行JavaScript、Groovy、Ruby等脚本语言 |

[^6]: 详细信息见http://openjdk.java.net/jeps/320。

# 4.3 可视化故障处理工具

​	JDK中除了附带大量的命令行工具外，还提供了几个功能集成度更高的可视化工具，用户可以使用这些可视化工具以更加便捷的方式进行进程故障诊断和调试工作。这类工具主要包括<u>JConsole、JHSDB、VisualVM和JMC</u>四个。其中，JConsole是最古老，早在JDK 5时期就已经存在的虚拟机监控工具，而JHSDB虽然名义上是JDK 9中才正式提供，但之前已经以sa-jdi.jar包里面的HSDB（可视化工具）和CLHSDB（命令行工具）的形式存在了很长一段时间[^7]。它们两个都是JDK的正式成员，随着JDK一同发布，无须独立下载，使用也是完全免费的。

​	VisualVM在JDK 6 Update 7中首次发布，直到JRockit Mission Control与OracleJDK的融合工作完成之前，它都曾是Oracle主力推动的多合一故障处理工具，现在它已经从OracleJDK中分离出来，成为一个独立发展的开源项目[^8]。VisualVM已不是JDK中的正式成员，但仍是可以免费下载、使用的。

​	Java Mission Control，曾经是大名鼎鼎的来自BEA公司的图形化诊断工具，随着BEA公司被Oracle收购，它便被融合进OracleJDK之中。在JDK 7 Update 40时开始随JDK一起发布，后来Java SEAdvanced产品线建立，Oracle明确区分了Oracle OpenJDK和OracleJDK的差别[^9]，JMC从JDK 11开始又被移除出JDK。虽然在2018年Oracle将JMC开源并交付给OpenJDK组织进行管理，但开源并不意味着免费使用，JMC需要与HotSpot内部的“飞行记录仪”（Java Flight Recorder，JFR）配合才能工作，而在JDK 11以前，JFR的开启必须解锁OracleJDK的商业特性支持（使用JCMD的VM.unlock_commercial_features或启动时加入`-XX：+UnlockCommercialFeatures`参数），所以这项功能在生产环境中仍然是需要付费才能使用的商业特性。

​	为避免本节讲解的内容变成对软件说明文档的简单翻译，笔者准备了一些代码样例，大多数是笔者特意编写的反面教材。稍后将会使用几款工具去监控、分析这些代码存在的问题，算是本节简单的实战演练。读者可以把在可视化工具观察到的数据、现象，与前面两章中讲解的理论知识进行互相验证。

[^7]:准确来说是Linux和Solaris在OracleJDK 6就可以使用HSDB和CLHSDB了，Windows上要到Oracle-JDK 7才可以用。
[^8]:VisualVM官方站点：https://visualvm.github.io。
[^9]:详见https://blogs.oracle.com/java-platform-group/oracle-jdk-releases-for-java-11-and-later。

### 4.3.1 JHSDB：基于服务性代理的调试工具

> [使用JHSDB分析Java对象的存储](https://blog.csdn.net/feixiang2039/article/details/109211568)	<=	网友的操作记录
>
> [《深入理解Java虚拟机》：JVM高级特性与最佳实践（第3 版）4.3.1 JHSDB：基于服务性代理的调试工具 (笔记随录)](https://www.cnblogs.com/Ashiamd/p/14054498.html)	<=	我自己的截图
>
> 这章节重点的内容即：
>
> + **从《Java虚拟机规范》所定义的概念模型来看，所有Class相关的信息都应该存放在方法区之中**，<u>但方法区该如何实现，《Java虚拟机规范》并未做出规定，这就成了一件允许不同虚拟机自己灵活把握的事情</u>。
>
> + **JDK 7及其以后版本的HotSpot虚拟机选择把静态变量与类型在Java语言一端的映射Class对象存放在一起，存储于Java堆之中**

​	JDK中提供了JCMD和JHSDB两个集成式的多功能工具箱，它们不仅整合了上一节介绍到的所有基础工具所能提供的专项功能，而且由于有着“后发优势”，能够做得往往比之前的老工具们更好、更强大，下表所示是JCMD、JHSDB与原基础工具实现相同功能的简要对比。

| 基础工具                | JCMD                              | JHSDB                   |
| ----------------------- | --------------------------------- | ----------------------- |
| jps -lm                 | jcmd                              | N/A                     |
| jmap -dump \<pid\>      | jcmd \<pid\> GC.heap_dump         | jhsdb jmap --binaryheap |
| jmap -histo \<pid\>     | jcmd \<pid> GC.class_histogram    | jhsdb jmap --histo      |
| jstack \<pid\>          | jcmd \<pid\> Thead.print          | jhsdb stack --locks     |
| jinfo -sysprops \<pid\> | jcmd \<pid\> VM.system_properties | jhsdb info --sysprops   |
| jinfo -flags \<pid\>    | jcmd \<pid\> VM.flags             | jhsdb jinfo --flags     |

​	本节的主题是可视化的故障处理，所以JCMD及JHSDB的命令行模式就不再作重点讲解了，读者可参考上一节的基础命令，再借助它们在JCMD和JHSDB中的help去使用，相信是很容易举一反三、触类旁通的。接下来笔者要通过一个实验来讲解JHSDB的图形模式下的功能。

​	<u>JHSDB是一款基于**服务性代理（Serviceability Agent，SA）**实现的进程外调试工具</u>。

​	**服务性代理是HotSpot虚拟机中一组用于映射Java虚拟机运行信息的、主要基于Java语言（含少量JNI代码）实现的API集合**。

​	服务性代理以HotSpot内部的数据结构为参照物进行设计，把这些C++的数据抽象出Java模型对象，相当于HotSpot的C++代码的一个镜像。通过服务性代理的API，可以在一个独立的Java虚拟机的进程里分析其他HotSpot虚拟机的内部数据，或者从HotSpot虚拟机进程内存中dump出来的转储快照里还原出它的运行状态细节。服务性代理的工作原理跟Linux上的GDB或者Windows上的Windbg是相似的。本次，我们要借助JHSDB来分析一下代码清单4-6中的代码[^10]，并通过实验来回答一个简单问题：staticObj、instanceObj、localObj这三个变量本身（而不是它们所指向的对象）存放在哪里？

​	代码清单4-6　JHSDB测试代码

```java
/**
* staticObj、instanceObj、localObj存放在哪里？
*/
public class JHSDB_TestCase {
  static class Test {
    static ObjectHolder staticObj = new ObjectHolder();
    ObjectHolder instanceObj = new ObjectHolder();
    void foo() {
      ObjectHolder localObj = new ObjectHolder();
      System.out.println("done"); // 这里设一个断点
    }
  }
  private static class ObjectHolder {}
  public static void main(String[] args) {
    Test test = new JHSDB_TestCase.Test();
    test.foo();
  }
}
```

​	答案读者当然都知道：<u>staticObj随着Test的类型信息存放在方法区，instanceObj随着Test的对象实例存放在Java堆，localObject则是存放在foo()方法栈帧的局部变量表</u>中。这个答案是通过前两章学习的理论知识得出的，现在要做的是通过JHSDB来实践验证这一点。

​	首先，我们要确保这三个变量已经在内存中分配好，然后将程序暂停下来，以便有空隙进行实验，这只要把断点设置在代码中加粗的打印语句上，然后在调试模式下运行程序即可。<u>由于JHSDB本身对压缩指针的支持存在很多缺陷，建议用64位系统的读者在实验时禁用压缩指针，另外为了后续操作时可以加快在内存中搜索对象的速度，也建议读者限制一下Java堆的大小</u>。本例中，笔者（书籍作者）采用的运行参数如下：

```java
-Xmx10m -XX:+UseSerialGC -XX:-UseCompressedOops
```

程序执行后通过jps查询到测试程序的进程ID，具体如下：

```shell
jps -l
8440 org.jetbrains.jps.cmdline.Launcher
11180 JHSDB_TestCase
15692 jdk.jcmd/sun.tools.jps.Jps
```

使用以下命令进入JHSDB的图形化模式，并使其附加进程11180：

```shell
jhsdb hsdb --pid 11180
```

命令打开的JHSDB的界面如下图所示。

![img](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128204939810-1892971627.png)

​	阅读上述代码可知，运行至断点位置一共会创建三个ObjectHolder对象的实例，只要是对象实例必然会在Java堆中分配，既然我们要查找引用这三个对象的指针存放在哪里，不妨从这三个对象开始着手，先把它们从Java堆中找出来。

​	首先点击菜单中的Tools->Heap Parameters[^11]，结果如下图所示，因为笔者（书籍作者）的运行参数中指定了使用的是Serial收集器，图中我们看到了典型的Serial的分代内存布局，Heap Parameters窗口中清楚列出了新生代的Eden、S1、S2和老年代的容量（单位为字节）以及它们的虚拟内存地址起止范围。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128204939810-1892971627.png)

​	实践时如果不指定收集器，即使用JDK默认的G1的话，得到的信息应该类似如下所示：

```shell
Heap Parameters:
garbage-first heap [0x00007f32c7800000, 0x00007f32c8200000] region size 1024K
```

注意一下图中各个区域的内存地址范围，后面还要用到它们。打开Windows->Console窗

口，使用scanoops命令在Java堆的新生代（从Eden起始地址到To Survivor结束地址）范围内查找

ObjectHolder的实例(然而我报错了。下述是课本例子)：

```shell
hsdb>scanoops 0x00007f32c7800000 0x00007f32c7b50000 JHSDB_TestCase$ObjectHolder
0x00007f32c7a7c458 JHSDB_TestCase$ObjectHolder
0x00007f32c7a7c480 JHSDB_TestCase$ObjectHolder
0x00007f32c7a7c490 JHSDB_TestCase$ObjectHolder
```

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128212602830-1429004213.png)

​	三个实例的地址都落到了Eden的范围之内，算是顺带验证了一般情况下新对象在Eden中创建的分配规则。再使用Tools->Inspector功能确认一下这三个地址中存放的对象。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128213311994-847339399.png)

​	<u>Inspector为我们展示了对象头和指向对象元数据的指针，里面包括了**Java类型的名字、继承关系、实现接口关系，字段信息、方法信息、运行时常量池的指针、内嵌的虚方法表（vtable）以及接口方法表（itable）**等</u>。由于我们的确没有在ObjectHolder上定义过任何字段，所以图中并没有看到任何实例字段数据，读者在做实验时不妨定义一些不同数据类型的字段，观察它们在HotSpot虚拟机里面是如何存储的。

​	接下来要根据堆中对象实例地址找出引用它们的指针，原本JHSDB的Tools菜单中有ComputeReverse Ptrs来完成这个功能，但在笔者的运行环境中一点击它就出现Swing的界面异常，看后台日志是报了个空指针，这个问题只是界面层的异常，跟虚拟机关系不大，所以笔者没有继续去深究（我也同样遇到报错），改为使用命令来做也很简单，先拿第一个对象来试试看：

```shell
revptrs 0x000000010dc532b0
null
Oop for java/lang/Class @ 0x000000010dc519a8
hsdb> 
```

​	果然找到了一个引用该对象的地方，是在一个java.lang.Class的实例里，并且给出了这个实例的地址，通过Inspector查看该对象实例，可以清楚看到这确实是一个java.lang.Class类型的对象实例，里面有一个名为staticObj的实例字段

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128213953482-1861188555.png)

​	**从《Java虚拟机规范》所定义的概念模型来看，所有Class相关的信息都应该存放在方法区之中**，<u>但方法区该如何实现，《Java虚拟机规范》并未做出规定，这就成了一件允许不同虚拟机自己灵活把握的事情</u>。

​	**JDK 7及其以后版本的HotSpot虚拟机选择把静态变量与类型在Java语言一端的映射Class对象存放在一起，存储于Java堆之中**，从我们的实验中也明确验证了这一点[^12]。

​	接下来继续查找第二个对象实例：

​	这次找到一个类型为JHSDB_TestCase$Test的对象实例。这个结果完全符合我们的预期，第二个ObjectHolder的指针是在Java堆中JHSDB_TestCase$Test对象的instanceObj字段上

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128220048431-1821284105.png)

​	但是我们采用相同方法查找第三个ObjectHolder实例时，JHSDB返回了一个null，表示未查找到任何结果：

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128220438045-1086699492.png)

​	看来<u>revptrs命令并不支持查找栈上的指针引用</u>，不过没有关系，得益于我们测试代码足够简洁，人工也可以来完成这件事情。在Java Thread窗口选中main线程后点击Stack Memory按钮查看该**线程的栈内存**。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201128220912067-1424097163.png)

​	JHSDB在旁边已经自动生成注释，说明这里确实是引用了一个来自新生代的JHSDB_TestCase$ObjectHolder对象。至此，本次实验中三个对象均已找到，并成功追溯到引用它们的地方，也就实践验证了开篇中提出的这些对象的引用是存储在什么地方的问题。

​	JHSDB提供了非常强大且灵活的命令和功能，本节的例子只是其中一个很小的应用，在实际开发、学习时，可以用它来调试虚拟机进程或者dump出来的内存转储快照，以积累更多的实际经验。

[^10]:本小节的原始案例来自RednaxelaFX的博客https://rednaxelafx.iteye.com/blog/1847971。
[^11]:效果与在Windows->Console中输入universe命令是等价的，JHSDB的图形界面中所有操作都可以通过命令行完成，读者感兴趣的话，可以在控制台中输入help命令查看更多信息。
[^12]:在JDK 7以前，即还没有开始“去永久代”行动时，这些静态变量是存放在永久代上的，JDK 7起把静态变量、字符常量这些从永久代移除出去。

### 4.3.2 







[^13]: