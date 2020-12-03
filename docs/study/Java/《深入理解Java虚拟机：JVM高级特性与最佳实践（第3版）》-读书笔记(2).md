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

## 4.3 可视化故障处理工具

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

> [JDK1.8之前和之后的方法区](https://blog.csdn.net/qq_41872909/article/details/87903370)
>
> jdk1.7之前：方法区位于永久代(PermGen)，永久代和堆相互隔离，永久代的大小在启动JVM时可以设置一个固定值，不可变；
> jdk.7：存储在永久代的部分数据就已经转移到Java Heap或者Native memory。但永久代仍存在于JDK 1.7中，并没有完全移除，譬如符号引用(Symbols)转移到了native memory；字符串常量池(interned strings)转移到了Java heap；类的静态变量(class statics variables )转移到了Java heap；
> jdk1.8：仍然保留方法区的概念，只不过实现方式不同。取消永久代，方法存放于元空间(Metaspace)，元空间仍然与堆不相连，但与堆共享物理内存，逻辑上可认为在堆中。
>
> 1）移除了永久代（PermGen），替换为元空间（Metaspace）；
> 2）永久代中的 class metadata 转移到了 native memory（本地内存，而不是虚拟机）；
> 3）永久代中的 interned Strings 和 class static variables 转移到了 Java heap；
> 4）永久代参数 （PermSize MaxPermSize） -> 元空间参数（MetaspaceSize MaxMetaspaceSize）。

### 4.3.2 JConsole：Java监视与管理控制台

​	JConsole（Java Monitoring and Management Console）是一款基于JMX（Java Management Extensions）的可视化监视、管理工具。它的主要功能是通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整。<u>JMX是一种开放性的技术，不仅可以用在虚拟机本身的管理上，还可以运行于虚拟机之上的软件中，典型的如中间件大多也基于JMX来实现管理与监控</u>。虚拟机对JMXMBean的访问也是完全开放的，可以使用代码调用API、支持JMX协议的管理控制台，或者其他符合JMX规范的软件进行访问。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201129221748002-1178032282.png)

1.  启动JConsole

   主要菜单选项有：OverView、Memory、Threads、Classes、VM Summary、MBeans

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201129230532524-42259536.png)

2. 内存监控

   ​	“内存”页签的作用相当于可视化的jstat命令，用于监视被收集器管理的虚拟机内存（被收集器直接管理的Java堆和被间接管理的方法区）的变化趋势。我们通过运行代码清单4-7中的代码来体验一下它的监视功能。运行时设置的虚拟机参数为：`-Xms100m -Xmx100m -XX:+UseSerialGC`

   ​	代码清单4-7　JConsole监视代码

   ```java
   import java.util.ArrayList;
   import java.util.List;
   
   public class JConsoleTestCase {
   
     /**
        * 内存占位符对象，一个OOMObject大约占64K
        */
     static class OOMObject {
       public byte[] placeholder = new byte[64 * 1024];
     }
   
     public static void fillHeap(int num) throws InterruptedException {
       List<OOMObject> list = new ArrayList<OOMObject>();
       for (int i = 0; i < num; i++) {
         // 稍作延时，令监视曲线的变化更加明显
         Thread.sleep(50);
         list.add(new OOMObject());
       }
       System.gc();
     }
   
     public static void main(String[] args) throws Exception {
       fillHeap(1000);
     }
   
   }
   ```

   ​	这段代码的作用是以64KB/50ms的速度向Java堆中填充数据，一共填充1000次，使用JConsole的“内存”页签进行监视，观察曲线和柱状指示图的变化。

   ​	程序运行后，在“内存”页签中可以看到内存池Eden区的运行趋势呈现折线状，如下图所示。监视范围扩大至整个堆后，会发现曲线是一直平滑向上增长的。从柱状图可以看到，在1000次循环执行结束，运行了`System.gc()`后，虽然整个新生代Eden和Survivor区都基本被清空了，但是代表老年代的柱状图仍然保持峰值状态，说明被填充进堆中的数据在`System.gc()`方法执行之后仍然存活。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201129234320251-1596587974.png)

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201129231139276-474635835.png)

   1. 虚拟机启动参数只限制了Java堆为100MB，但没有明确使用-Xmn参数指定新生代大小，能否从监控图中估算出新生代的容量？

      答：上图显示Eden空间为27328KB，因为没有设置`-XX：SurvivorRadio`参数，所以Eden与Survivor空间比例的默认值为8∶1，因此整个新生代空间大约为27328KB×125%=34160KB。

   2. 为何执行了`System.gc()`之后，图中代表老年代的柱状图仍然显示峰值状态，代码需要如何调整才能让`System.gc()`回收掉填充到堆中的对象？

      答：执行`System.gc()`之后，空间未能回收是因为List\<OOMObject\>list对象仍然存活，`fillHeap()`方法仍然没有退出，因此list对象在`System.gc()`执行时仍然处于作用域之内[^13]。**如果把`System.gc()`移动到`fillHeap()`方法外调用就可以回收掉全部内存。**

3. 线程监控

   ​	**如果说JConsole的“内存”页签相当于可视化的jstat命令的话，那“线程”页签的功能就相当于可视化的jstack命令了，遇到线程停顿的时候可以使用这个页签的功能进行分析**。前面讲解jstack命令时提到<u>线程长时间停顿的主要原因有等待外部资源（数据库连接、网络资源、设备资源等）、死循环、锁等待等</u>，代码清单4-8将分别演示这几种情况。

   ​	代码清单4-8　线程等待演示代码

   ```java
   import java.io.BufferedReader;
   import java.io.InputStreamReader;
   
   /**
    * @author zzm
    */
   public class ThreadDeadLockTestCase_1 {
       /**
        * 线程死循环演示
        */
       public static void createBusyThread() {
           Thread thread = new Thread(new Runnable() {
               @Override
               public void run() {
                   while (true)   // 第41行
                       ;
               }
           }, "testBusyThread");
           thread.start();
       }
   
       /**
        * 线程锁等待演示
        */
       public static void createLockThread(final Object lock) {
           Thread thread = new Thread(new Runnable() {
               @Override
               public void run() {
                   synchronized (lock) {
                       try {
                           lock.wait();
                       } catch (InterruptedException e) {
                           e.printStackTrace();
                       }
                   }
               }
           }, "testLockThread");
           thread.start();
       }
   
       public static void main(String[] args) throws Exception {
           BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
           br.readLine();
           createBusyThread();
           br.readLine();
           Object obj = new Object();
           createLockThread(obj);
       }
   
   }
   ```

   ​	程序运行后，首先在“线程”页签中选择main线程，如图4-13所示。堆栈追踪显示BufferedReader的`readBytes()`方法正在等待System.in的键盘输入，这时候线程为Runnable状态，Runnable状态的线程仍会被分配运行时间，但readBytes()方法检查到流没有更新就会立刻归还执行令牌给操作系统，这种等待只消耗很小的处理器资源。

   ​	接着监控testBusyThread线程，如图4-14所示。testBusyThread线程一直在执行空循环，从堆栈追踪中看到一直在MonitoringTest.java代码的41行停留，41行的代码为`while(true)`。这时候线程为Runnable状态，而且没有归还线程执行令牌的动作，所以会在空循环耗尽操作系统分配给它的执行时间，直到线程切换为止，这种等待会消耗大量的处理器资源。

   ​	<small>ps：(这个我根据上面代码，本地跑jconsole，不显示"testBusyThread"和"testLockThread"这两线程，所以直接用书籍的插图了)</small>

   图4-13　main线程

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130001724858-878962935.png)

   图4-14　testBusyThread线程

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130001854427-824654847.png)

   图4-15显示testLockThread线程在等待lock对象的notify()或notifyAll()方法的出现，线程这时候处于WAITING状态，在重新唤醒前不会被分配执行时间。

   图4-15　testLockThread线程

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130001937741-922713234.png)

   testLockThread线程正处于正常的活锁等待中，只要lock对象的notify()或notifyAll()方法被调用，这个线程便能激活继续执行。

   ---

   代码清单4-9　死锁代码样例

   ```java
   /**
    * @author zzm
    */
   public class ThreadDeadLockTestCase_2 {
   
     /**
        * 线程死锁等待演示
        */
     static class SynAddRunnalbe implements Runnable {
       int a, b;
       public SynAddRunnalbe(int a, int b) {
         this.a = a;
         this.b = b;
       }
   
       @Override
       public void run() {
         synchronized (Integer.valueOf(a)) {
           synchronized (Integer.valueOf(b)) {
             System.out.println(a + b);
           }
         }
       }
     }
   
     public static void main(String[] args) {
       for (int i = 0; i < 100; i++) {
         new Thread(new SynAddRunnalbe(1, 2)).start();
         new Thread(new SynAddRunnalbe(2, 1)).start();
       }
     }
   
   
   }
   ```

   ​	这段代码开了200个线程去分别计算1+2以及2+1的值，理论上for循环都是可省略的，两个线程也可能会导致死锁，不过那样概率太小，需要尝试运行很多次才能看到死锁的效果。如果运气不是特别差的话，上面带for循环的版本最多运行两三次就会遇到线程死锁，程序无法结束。

   ​	造成死锁的根本原因是`Integer.valueOf()`方法出于减少对象创建次数和节省内存的考虑，会对数值为-128～127之间的Integer对象进行缓存[^14]，如果`valueOf()`方法传入的参数在这个范围之内，就直接返回缓存中的对象。也就是说代码中尽管调用了200次`Integer.valueOf()`方法，但一共只返回了两个不同的Integer对象。**假如某个线程的两个synchronized块之间发生了一次线程切换，那就会出现线程A在等待被线程B持有的`Integer.valueOf(1)`，线程B又在等待被线程A持有的`Integer.valueOf(2)`，结果大家都跑不下去的情况。**

   ​	出现线程死锁之后，点击JConsole线程面板的“检测到死锁”按钮，将出现一个新的“死锁”页签，如图4-16所示。

   图4-16　线程死锁

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130002454011-627698124.png)

   ​	图4-16中很清晰地显示，线程Thread-43在等待一个被线程Thread-12持有的Integer对象，而点击线程Thread-12则显示它也在等待一个被线程Thread-43持有的Integer对象，这样两个线程就互相卡住，除非牺牲其中一个，否则死锁无法释放。

[^13]:准确地说，只有虚拟机使用解释器执行的时候，“在作用域之内”才能保证它不会被回收，因为**这里的回收还涉及局部变量表变量槽的复用、即时编译器介入时机等问题**，具体可参考第8章的代码清单8-1。
[^14]:这是《Java虚拟机规范》中明确要求缓存的默认值，实际值可以调整，具体取决于java.lang.Integer.Integer-Cache.high参数的设置。

### 4.3.3 VisualVM：多合-故障处理工具

​	<u>VisualVM（All-in-One Java Troubleshooting Tool）是功能最强大的运行监视和故障处理程序之一，曾经在很长一段时间内是Oracle官方主力发展的虚拟机故障处理工具</u>。Oracle曾在VisualVM的软件说明中写上了“All-in-One”的字样，预示着它除了常规的运行监视、故障处理外，还将提供其他方面的能力，譬如性能分析（Profiling）。VisualVM的性能分析功能比起JProfiler、YourKit等专业且收费的Profiling工具都不遑多让。而且相比这些第三方工具，<u>**VisualVM还有一个很大的优点：不需要被监视的程序基于特殊Agent去运行，因此它的通用性很强，对应用程序实际性能的影响也较小，使得它可以直接应用在生产环境中**</u>。这个优点是JProfiler、YourKit等工具无法与之媲美的。

1. VisualVM兼容范围与插件安装

   ​	VisualVM基于NetBeans平台开发工具，所以一开始它就具备了通过插件扩展功能的能力，有了插件扩展支持，VisualVM可以做到：

   + 显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。

   + 监视应用程序的处理器、垃圾收集、堆、方法区以及线程的信息（jstat、jstack）。

   + dump以及分析堆转储快照（jmap、jhat）。

   + 方法级的程序运行性能分析，找出被调用最多、运行时间最长的方法。

   + 离线程序快照：收集程序的运行时配置、线程dump、内存dump等信息建立一个快照，可以将快照发送开发者处进行Bug反馈。

   + 其他插件带来的无限可能性。

   ​	VisualVM在JDK 6 Update 7中首次发布，但并不意味着它只能监控运行于JDK 6上的程序，它具备很优秀的向下兼容性，甚至能向下兼容至2003年发布的JDK 1.4.2版本[^15]，这对无数处于已经完成实施、正在维护的遗留项目很有意义。当然，也并非所有功能都能完美地向下兼容，主要功能的兼容性见下表所示。

   | 特性         | JDK 1.4.2 | JDK 5 | JDK 6 local | JDK 6 remote |
   | ------------ | --------- | ----- | ----------- | ------------ |
   | 运行环境     | yes       | yes   | yes         | yes          |
   | 系统属性     |           |       | yes         |              |
   | 监视面板     | yes       | yes   | yes         | yes          |
   | 线程面板     |           | yes   | yes         | yes          |
   | 性能监控     |           |       | yes         |              |
   | 堆、线程Dump |           |       | yes         |              |
   | MBean管理    |           | yes   | yes         | yes          |
   | JConsole插件 |           | yes   | yes         | yes          |

   ​	首次启动VisualVM后，读者先不必着急找应用程序进行监测，初始状态下的VisualVM并没有加载任何插件，虽然基本的监视、线程面板的功能主程序都以默认插件的形式提供，但是如果不在VisualVM上装任何扩展插件，就相当于放弃它最精华的功能，和没有安装任何应用软件的操作系统差不多。

   ​	VisualVM的插件可以手工进行安装，在网站[^16]上下载nbm包后，点击“工具->插件->已下载”菜单，然后在弹出对话框中指定nbm包路径便可完成安装。独立安装的插件存储在VisualVM的根目录，譬如JDK 9之前自带的VisulalVM，插件安装后是放在JDK_HOME/lib/visualvm中的。手工安装插件并不常用，VisualVM的自动安装功能已可找到大多数所需的插件，在有网络连接的环境下，点击“工具->插件菜单”，弹出如下图所示的插件页签，在页签的“可用插件”及“已安装”中列举了当前版本VisualVM可以使用的全部插件，选中插件后在右边窗口会显示这个插件的基本信息，如开发者、版本、功能描述等。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130204322532-201842452.png)

   ​	选择一个需要监视的程序就可以进入程序的主界面了，如下图所示。VisualVM的版本以及选择安装插件数量的不同，显示的界面可能不同。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130213257397-790980954.png)

   ​	VisualVM中“概述”、“监视”、“线程”、“MBeans”的功能与前面介绍的JConsole差别不大，可根据上一节内容类比使用，这里笔者（书籍作者）挑选几个有特色的功能和插件进行简要介绍。

2. 生成、浏览堆转储快照

   ​	在VisualVM中生成堆转储快照文件有两种方式，可以执行下列任一操作：

   + 在“应用程序”窗口中右键单击应用程序节点，然后选择“堆Dump”。
   + 在“应用程序”窗口中双击应用程序节点以打开应用程序标签，然后在“监视”标签中单击“堆Dump”。

   ​	生成堆转储快照文件之后，应用程序页签会在该堆的应用程序下增加一个以[heap-dump]开头的子节点，并且在主页签中打开该转储快照，如下图所示。如果需要把堆转储快照保存或发送出去，就应在heapdump节点上右键选择“另存为”菜单，否则当VisualVM关闭时，生成的堆转储快照文件会被当作临时文件自动清理掉。要打开一个由已经存在的堆转储快照文件，通过文件菜单中的“装入”功能，选择硬盘上的文件即可。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130213617806-448068708.png)

   ​	堆页签中的“摘要”面板可以看到应用程序dump时的运行时参数、System.getProperties()的内容、线程堆栈等信息；“类”面板则是以类为统计口径统计类的实例数量、容量信息；“实例”面板不能直接使用，因为VisualVM在此时还无法确定用户想查看哪个类的实例，所以需要通过“类”面板进入，在“类”中选择一个需要查看的类，然后双击即可在“实例”里面看到此类的其中500个实例的具体属性信息；“OQL控制台”面板则是运行OQL查询语句的，同jhat中介绍的OQL功能一样。如果读者想要了解具体OQL的语法和使用方法，可参见本书附录D的内容。

3. 分析程序性能

   ​	在Profiler页签中，VisualVM提供了程序运行期间方法级的处理器执行时间分析以及内存分析。做Profiling分析肯定会对程序运行性能有比较大的影响，所以一般不在生产环境使用这项功能，或者改用JMC来完成，JMC的Profiling能力更强，对应用的影响非常轻微。

   ​	要开始性能分析，先选择“CPU”和“内存”按钮中的一个，然后切换到应用程序中对程序进行操作，VisualVM会记录这段时间中应用程序执行过的所有方法。如果是进行处理器执行时间分析，将会统计每个方法的执行次数、执行耗时；如果是内存分析，则会统计每个方法关联的对象数以及这些对象所占的空间。等要分析的操作执行结束后，点击“停止”按钮结束监控过程，如下图所示（自己本地执行莫名没画面，就干脆用书上的图片了）。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130214611769-995339643.png)

   *注意：　在JDK 5之后，在客户端模式下的虚拟机加入并且自动开启了类共享——这是一个在多虚拟机进程共享rt.jar中类数据以提高加载速度和节省内存的优化，而根据相关Bug报告的反映，VisualVM的Profiler功能会因为类共享而导致被监视的应用程序崩溃，所以读者进行Profiling前，最好在被监视程序中使用-Xshare：off参数来关闭类共享优化。*

   分析自己的应用程序时，可根据实际业务复杂程度与方法的时间、调用次数做比较，找到最优化价值方法。

4. BTrace动态日志跟踪

   ​	BTrace[^18]是一个很神奇的VisualVM插件，它本身也是一个可运行的独立程序。BTrace的作用是在不中断目标程序运行的前提下，通过HotSpot虚拟机的Instrument功能[^19]动态加入原本并不存在的调试代码。这项功能对实际生产中的程序很有意义：如当程序出现问题时，排查错误的一些必要信息时（譬如方法参数、返回值等），在开发时并没有打印到日志之中以至于不得不停掉服务时，都可以通过调试增量来加入日志代码以解决问题。

   ​	在VisualVM中安装了BTrace插件后，在应用程序面板中右击要调试的程序，会出现“Trace Application…”菜单，点击将进入BTrace面板。这个面板看起来就像一个简单的Java程序开发环境，里面甚至已经有了一小段Java代码，如下图所示。

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130220322620-645730181.png)

   ​	笔者（书籍作者）准备了一段简单的Java代码来演示BTrace的功能：产生两个1000以内的随机整数，输出这两个数字相加的结果，如代码清单4-10所示。

   ​	代码清单4-10　BTrace跟踪演示

   ```java
   import java.io.BufferedReader;
   import java.io.IOException;
   import java.io.InputStreamReader;
   
   /**
    * @author zzm
    */
   public class BTraceTest {
   
     public int add(int a, int b) {
       return a + b;
     }
   
     public static void main(String[] args) throws IOException {
       BTraceTest test = new BTraceTest();
       BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
       for (int i = 0; i < 10; i++) {
         reader.readLine();
         int a = (int) Math.round(Math.random() * 1000);
         int b = (int) Math.round(Math.random() * 1000);
         System.out.println(test.add(a, b));
       }
     }
   }
   ```

   ​	假设这段程序已经上线运行，而我们现在又有了新的需求，想要知道程序中生成的两个随机数是什么，但程序并没有在执行过程中输出这一点。此时，在VisualVM中打开该程序的监视，在BTrace页签填充TracingScript的内容，输入调试代码，如代码清单4-11所示，即可在不中断程序运行的情况下做到这一点。

   ​	代码清单4-11　BTrace调试代码

   ```java
   /* BTrace Script Template */
   import com.sun.btrace.annotations.*;
   import static com.sun.btrace.BTraceUtils.*;
   
   @BTrace
   public class TracingScript {
     @OnMethod(
       clazz="org.fenixsoft.monitoring.BTraceTest",
       method="add",
       location=@Location(Kind.RETURN)
     )
   
     public static void func(@Self org.fenixsoft.monitoring.BTraceTest instance,int a,int b,@Return int result) {
       println("调用堆栈:");
       jstack();
       println(strcat("方法参数A:",str(a)));
       println(strcat("方法参数B:",str(b)));
       println(strcat("方法结果:",str(result)));
     }
   }
   ```

   ​	点击Start按钮后稍等片刻，编译完成后，Output面板中会出现“BTrace code successfuly deployed”的字样。当程序运行时将会在Output面板输出如图4-23所示的调试信息。

   <small>(本地尝试了下，但是报错了，试改了一些代码，无果。就先用书籍的插图了)</small>

   ![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130224455515-263986918.png)

   ​	BTrace的用途很广泛，打印调用堆栈、参数、返回值只是它最基础的使用形式，在它的网站上有使用BTrace进行性能监视、定位连接泄漏、内存泄漏、解决多线程竞争问题等的使用案例，有兴趣的读者可以去网上了解相关信息。

   ​	<u>BTrace能够实现动态修改程序行为，是因为它是基于Java虚拟机的Instrument开发的。Instrument是Java虚拟机工具接口（Java Virtual Machine Tool Interface，JVMTI）的重要组件，提供了一套代理（Agent）机制，使得第三方工具程序可以以代理的方式访问和修改Java虚拟机内部的数据</u>。阿里巴巴开源的诊断工具Arthas也通过Instrument实现了与BTrace类似的功能。

[^15]:早于JDK 6的平台，需要打开-Dcom.sun.management.jmxremote参数才能被VisualVM管理。
[^16]:插件中心地址：https://visualvm.github.io/pluginscenters.html。
[^17]:官方主页：https://github.com/btraceio/btrace。
[^18]:是JVMTI中的主要组成部分，**HotSpot虚拟机允许在不停止运行的情况下，更新已经加载的类的代码**。

> [java debug 体系-JVMTI](https://www.jianshu.com/p/e59c4eed44a2)

### 4.3.4 Java Mission Control：可持续在线的监控工具

​	除了大家熟知的面向通用计算（General Purpose Computing）可免费使用的Java SE外，Oracle公司还开辟过带商业技术支持的Oracle Java SE Support和面向独立软件供应商（ISV）的Oracle Java SEAdvanced & Suite产品线。

​	除去带有7×24小时的技术支持以及可以为企业专门定制安装包这些非技术类的增强服务外，Oracle Java SE Advanced & Suite[^19]与普通Oracle Java SE在功能上的主要差别是前者包含了一系列的监控、管理工具，譬如用于企业JRE定制管理的AMC（Java Advanced Management Console）控制台、JUT（Java Usage Tracker）跟踪系统，用于持续收集数据的JFR（Java Flight Recorder）飞行记录仪和用于监控Java虚拟机的JMC（Java Mission Control）。这些功能全部都是需要商业授权才能在生产环境中使用，但根据Oracle Binary Code协议，在个人开发环境中，允许免费使用JMC和JFR，本节笔者将简要介绍它们的原理和使用。

​	<u>JFR是一套**内建在HotSpot虚拟机**里面的监控和基于事件的信息搜集框架，与其他的监控工具（如JProfiling）相比，Oracle特别强调它“可持续在线”（Always-On）的特性。JFR在生产环境中对吞吐量的影响一般不会高于1%（甚至号称是Zero Performance Overhead），而且JFR监控过程的开始、停止都是完全可动态的，即不需要重启应用。JFR的监控对应用也是完全透明的，即不需要对应用程序的源码做任何修改，或者基于特定的代理来运行。</u>

​	JMC最初是BEA公司的产品，因此并没有像VisualVM那样一开始就基于自家的Net-Beans平台来开发，而是选择了由IBM捐赠的Eclipse RCP作为基础框架，现在的JMC不仅可以下载到独立程序，更常见的是作为Eclipse的插件来使用。JMC与虚拟机之间同样采取JMX协议进行通信，JMC一方面作为JMX控制台，显示来自虚拟机MBean提供的数据；另一方面作为JFR的分析工具，展示来自JFR的数据。启动后JMC的主界面如下图所示<small>（mac打开JMC-7会出问题，暂时没找到解决方法，就直接用书上插图了）</small>。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130230920692-935637162.png)

​	在左侧的“JVM浏览器”面板中自动显示了通过JDP协议（Java Discovery Protocol）找到的本机正在运行的HotSpot虚拟机进程，如果需要监控其他服务器上的虚拟机，可在“文件->连接”菜单中创建远程连接，如下图所示。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130231151989-1957328248.png)

​	这里要填写的信息应该在被监控虚拟机进程启动的时候以虚拟机参数的形式指定，以下是一份被监控端的启动参数样例：

```java
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
-Djava.rmi.server.hostname=192.168.31.4
-XX:+UnlockCommercialFeatures -XX:+FlightRecorder
```

​	本地虚拟机与远程虚拟机进程的差别只限于创建连接这个步骤，连接成功创建以后的操作就是完全一样的了。把“JVM浏览器”面板中的进程展开后，可以看到每个进程的数据都有MBean和JFR两个数据来源。关于MBean这部分数据，与JConsole和VisualVM上取到的内容是一样的，只是展示形式上有些差别，笔者（书籍作者）就不再重复了，后面着重介绍JFR的数据记录。

​	双击“飞行记录器”，将会出现“启动飞行记录”窗口（如果第一次使用，还会收到解锁商业功能的警告窗），如下图所示。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130231330756-927167290.png)

​	在启动飞行记录时，可以进行记录时间、垃圾收集器、编译器、方法采样、线程记录、异常记录、网络和文件I/O、事件记录等选项和频率设定，这部分比较琐碎，不一一截图讲解了。点击“完成”按钮后马上就会开始记录，记录时间结束以后会生成飞行记录报告，如下图所示。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130231449782-1581141260.png)

飞行记录报告里包含以下几类信息：

+ 一般信息：关于虚拟机、操作系统和记录的一般信息。

+ 内存：关于内存管理和垃圾收集的信息。

+ 代码：关于方法、异常错误、编译和类加载的信息。

+ 线程：关于应用程序中线程和锁的信息。

+ I/O：关于文件和套接字输入、输出的信息。

+ 系统：关于正在运行Java虚拟机的系统、进程和环境变量的信息。

+ 事件：关于记录中的事件类型的信息，可以根据线程或堆栈跟踪，按照日志或图形的格式查看。

​	JFR的基本工作逻辑是开启一系列事件的录制动作，当某个事件发生时，这个事件的所有上下文数据将会以循环日志的形式被保存至内存或者指定的某个文件当中，循环日志相当于数据流被保留在一个环形缓存中，所以只有最近发生的事件的数据才是可用的。JMC从虚拟机内存或者文件中读取并展示这些事件数据，并通过这些数据进行性能分析。

​	即使不考虑对被测试程序性能影响方面的优势，JFR提供的数据质量通常也要比其他工具通过代理形式采样获得或者从MBean中取得的数据高得多。<u>以垃圾搜集为例，HotSpot的MBean中一般有各个分代大小、收集次数、时间、占用率等数据（根据收集器不同有所差别），这些都属于“结果”类的信息，而JFR中还可以看到内存中这段时间分配了哪些对象、哪些在TLAB中（或外部）分配、分配速率和压力大小如何、分配归属的线程、收集时对象分代晋升的情况等，这些就是属于“过程”类的信息，对排查问题的价值是难以估量的</u>。

[^19]:Advanced是“Advanced Monitoring & Management of Java in the Enterprise”的缩写。

## 4.4 HotSpot虚拟机插件及工具

​	HotSpot虚拟机发展了二十余年，现在已经是一套很复杂的软件系统，如果深入挖掘HotSpot的源码，可以发现在HotSpot的研发过程中，开发团队曾经编写（或者收集）过不少虚拟机的插件和辅助工具，它们存放在HotSpot源码hotspot/src/share/tools目录下，包括（含曾经有过但新版本中已被移除的）：

+ Ideal Graph Visualizer：用于可视化展示C2即时编译器是如何将字节码转化为理想图，然后转化为机器码的。

+ Client Compiler Visualizer[^20]：用于查看C1即时编译器生成高级中间表示（HIR），转换成低级中间表示（LIR）和做物理寄存器分配的过程。

+ MakeDeps：帮助处理HotSpot的编译依赖的工具。

+ Project Creator：帮忙生成Visual Studio的.project文件的工具。

+ LogCompilation：将`-XX：+LogCompilation`输出的日志整理成更容易阅读的格式的工具。

+ HSDIS：即时编译器的反汇编插件。

​	关于Client Compiler Visualizer和Ideal Graph Visualizer，在本书第11章会有专门的使用介绍，而Project Creator、LogCompilation、MakeDeps这三个工具对本书的讲解和实验帮助有限，最后一个HSDIS是学习、实践本书第四部分“程序编译与代码优化”的有力辅助工具，借本章讲解虚拟机工具的机会，简要介绍其使用方法。

---

**HSDIS：JIT生成代码反汇编**

​	在《Java虚拟机规范》里详细定义了虚拟机指令集中每条指令的语义，尤其是执行过程前后对操作数栈、局部变量表的影响。这些细节描述与早期Java虚拟机（Sun Classic虚拟机）高度吻合，但随着技术的发展，高性能虚拟机真正的细节实现方式已经渐渐与《Java虚拟机规范》所描述的内容产生越来越大的偏差，《Java虚拟机规范》中的规定逐渐成为Java虚拟机实现的“概念模型”，即实现只保证与规范描述等效，而不一定是按照规范描述去执行。由于这个原因，我们在讨论程序的执行语义问题（虚拟机做了什么）时，在字节码层面上分析完全可行，但讨论程序的执行行为问题（虚拟机是怎样做的、性能如何）时，在字节码层面上分析就没有什么意义了，必须通过其他途径解决。

​	至于分析程序如何执行，使用软件调试工具（GDB、Windbg等）来进行断点调试是一种常见的方式，但是这样的调试方式在Java虚拟机中也遇到了很大麻烦，因为大量执行代码是通过即时编译器动态生成到代码缓存中的，并没有特别简单的手段来处理这种混合模式的调试，不得不通过一些曲线的间接方法来解决问题。在这样的背景下，本节的主角——HSDIS插件就正式登场了。

​	**HSDIS是一个被官方推荐的HotSpot虚拟机即时编译代码的反汇编插件**，它包含在HotSpot虚拟机的源码当中[^21]，在OpenJDK的网站[^22]也可以找到单独的源码下载，但并没有提供编译后的程序。

​	HSDIS插件的作用是让HotSpot的-XX：+PrintAssembly指令调用它来把即时编译器动态生成的本地代码还原为汇编代码输出，同时还会自动产生大量非常有价值的注释，这样我们就可以通过输出的汇编代码来从最本质的角度分析问题。读者可以根据自己的操作系统和处理器型号，从网上直接搜索、下载编译好的插件，直接放到JDK_HOME/jre/bin/server目录（JDK 9以下）或JDK_HOME/lib/amd64/server（JDK 9或以上）中即可使用。如果读者确实没有找到所采用操作系统的对应编译成品[^23]，那就自己用源码编译一遍（网上能找到各种操作系统下的编译教程）。

​	另外还有一点需要注意，如果读者使用的是SlowDebug或者FastDebug版的HotSpot，那可以直接通过-XX：+PrintAssembly指令使用的插件；如果读者使用的是Product版的HotSpot，则还要额外加入一个-XX：+UnlockDiagnosticVMOptions参数才可以工作。笔者以代码清单4-12中的测试代码为例简单演示一下如何使用这个插件。

​	代码清单4-12　测试代码

```java
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
```

​	编译这段代码，并使用以下命令执行：

```java
java -XX:+PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*Bar.sum -XX:Compile-Command=compileonly,*Bar.sum test.
```

​	其中，参数-Xcomp是让虚拟机以编译模式执行代码，这样不需要执行足够次数来预热就能触发即时编译。两个-XX：CompileCommand的意思是让编译器不要内联sum()并且只编译sum()，-XX：+PrintAssembly就是输出反汇编内容。如果一切顺利的话，屏幕上会出现类似代码清单4-13所示的内容。

​	代码清单4-13　测试代码

```assembly
[Disassembling for mach='i386']
[Entry Point]
[Constants]
# {method} 'sum' '(I)I' in 'test/Bar'
# this: ecx = 'test/Bar'
# parm0: edx = int
# [sp+0x20] (sp of caller)
……
0x01cac407: cmp 0x4(%ecx),%eax
0x01cac40a: jne 0x01c6b050 ; {runtime_call}
[Verified Entry Point]
0x01cac410: mov %eax,-0x8000(%esp)
0x01cac417: push %ebp
0x01cac418: sub $0x18,%esp ; *aload_0
; - test.Bar::sum@0 (line 8)
;; block B0 [0, 10]
0x01cac41b: mov 0x8(%ecx),%eax ; *getfield a
; - test.Bar::sum@1 (line 8)
0x01cac41e: mov $0x3d2fad8,%esi ; {oop(a
'java/lang/Class' = 'test/Bar')}
0x01cac423: mov 0x68(%esi),%esi ; *getstatic b
; - test.Bar::sum@4 (line 8)
0x01cac426: add %esi,%eax
0x01cac428: add %edx,%eax
0x01cac42a: add $0x18,%esp
0x01cac42d: pop %ebp
0x01cac42e: test %eax,0x2b0100 ; {poll_return}
0x01cac434: ret
```

虽然是汇编，但代码并不多，我们一句一句来阅读：

1）mov%eax，-0x8000(%esp)：检查栈溢。

2）push%ebp：保存上一栈帧基址。

3）sub$0x18，%esp：给新帧分配空间。

4）mov 0x8(%ecx)，%eax：取实例变量a，这里0x8(%ecx)就是ecx+0x8的意思，前面代码片段“[Constants]”中提示了“this：ecx='test/Bar'”，即ecx寄存器中放的就是this对象的地址。偏移0x8是越过this对象的对象头，之后就是实例变量a的内存位置。这次是访问Java堆中的数据。

5）mov$0x3d2fad8，%esi：取test.Bar在方法区的指针。

6）mov 0x68(%esi)，%esi：取类变量b，这次是访问方法区中的数据。

7）add%esi，%eax、add%edx，%eax：做2次加法，求a+b+c的值，前面的代码把a放在eax中，把b放在esi中，而c在[Constants]中提示了，“parm0：edx=int”，说明c在edx中。

8）add$0x18，%esp：撤销栈帧。

9）pop%ebp：恢复上一栈帧。

10）test%eax，0x2b0100：轮询方法返回处的SafePoint。

11）ret：方法返回。

​	在这个例子中测试代码比较简单，肉眼直接看日志中的汇编输出是可行的，但在正式环境中-XX：+PrintAssembly的日志输出量巨大，且难以和代码对应起来，这就必须使用工具来辅助了。

​	JITWatch[^24]是HSDIS经常搭配使用的可视化的编译日志分析工具，为便于在JITWatch中读取，读者可使用以下参数把日志输出到logfile文件：

```java
-XX:+UnlockDiagnosticVMOptions
-XX:+TraceClassLoading
-XX:+LogCompilation
-XX:LogFile=/tmp/logfile.log
-XX:+PrintAssembly
-XX:+TraceClassLoading
```

​	在JITWatch中加载日志后，就可以看到执行期间使用过的各种对象类型和对应调用过的方法了，界面如下图所示。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130234103186-1963839743.png)

​	选择想要查看的类和方法，即可查看对应的Java源代码、字节码和即时编译器生成的汇编代码，如下图所示。

![](https://img2020.cnblogs.com/blog/1244059/202011/1244059-20201130234557986-46124353.png)

[^20]:不同于Ideal Graph Visualizer，Client Compiler Visualizer的源码其实从未进入过HotSpot的代码仓库，不过为了C1、C2配对，还是把它列在这里。
[^21]:OpenJDK中的源码位置：hotspot/src/share/tools/hsdis/。
[^22]:地址：http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/tip/src/share/tools/hsdis/。也可以在GitHub上搜索HSDIS得到。
[^23]:HLLVM圈子中有已编译好的，地址：http://hllvm.group.iteye.com/。
[^24]:下载地址：https://github.com/AdoptOpenJDK/jitwatch。

## 4.5 本章小结

​	本章介绍了随JDK发布的6个命令行工具与4个可视化的故障处理工具，灵活使用这些工具，可以为处理问题带来很大的便利。除了本章涉及的OpenJDK中自带的工具之外，还有很多其他监控和故障处理工具，如何进行监控和故障诊断，这并不是《Java虚拟机规范》中定义的内容，而是取决于虚拟机实现自身的设计，因此每种处理工具都有针对的目标范围，如果读者使用的是非HotSpot系的虚拟机，就更需要使用对应的工具进行分析，如：

+ IBM的Support Assistant[^25]、Heap Analyzer[^26]、Javacore Analyzer[^27]、Garbage CollectorAnalyzer[^28]适用于IBM J9/OpenJ9 VM。

+ HP的HPjmeter、HPjtune适用于HP-UX、SAP、HotSpot VM。

+ Eclipse的Memory Analyzer Tool[^29]（MAT）适用于HP-UX、SAP、HotSpot VM，安装IBMDTFJ[^30]插件后可支持IBM J9虚拟机。

[^25]:http://www-01.ibm.com/software/support/isa/。
[^26]:http://www.alphaworks.ibm.com/tech/heapanalyzer/download。
[^27]:http://www.alphaworks.ibm.com/tech/jca/download。
[^28]:http://www.alphaworks.ibm.com/tech/pmat/download。
[^29]:http://www.eclipse.org/mat/。
[^30]:http://www.ibm.com/developerworks/java/jdk/tools/dtfj.html。

# 5. 第5章　调优案例分析与实战

​	Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。

## 5.1 概述

​	在前面3章笔者系统性地介绍了处理Java虚拟机内存问题的知识与工具，在处理应用中的实际问题时，除了知识与工具外，经验同样是一个很重要的因素。在本章，将会与读者分享若干较有代表性的实际案例。

​	考虑到虚拟机的故障处理与调优主要面向各类服务端应用，而大多数Java程序员较少有机会直接接触生产环境的服务器，因此本章还准备了一个所有开发人员都能够进行“亲身实战”的练习，希望大家通过实践能获得故障处理、调优的经验。

*（吐槽：服务器倒是经常接触，倒是这些JVM工具什么的，确实用得不熟）*

## 5.2 案例分析

### 5.2.1　大内存硬件上的程序部署策略

**个人小结**：

+ 一个Java进程分配超大内存	=>	GC虽然初期不频繁，但后期FullGC时，由于内存大，GC时间长
+ 多个Java进程负载均衡（每个内存较少）=>	内存利用率高，GC相对频繁，但是GC时间相对短

降低FullGC次数、时长、影响：

+ 单个JVM分配的内存适当
+ 多Java进程负载均衡时，减少HashMap等Key-Value的分散使用，考虑集中式缓存（Redis等）
+ 考虑集中管理（数据库）连接池资源
+ 考虑在夜深人静时主动触发FullGC

----

> JVM老年代和新生代的比例](https://www.cnblogs.com/shoshana-kong/p/11314677.html)
>
> [常用基础参数NewRatio讲解](https://www.jianshu.com/p/c4f00b61b423)
>
> 使用指令`java -XX:+PrintFlagsFinal`	查看所有`-XX:`非标参数
>
> `uintx NewRatio                 = 2`	<=	默认老年代Old : 新生代Young = 2
>
> `uintx SurvivorRatio              = 8`	<= 默认 Eden : Survivor(from/to) = 8

​	一个15万PV/日左右的在线文档类型网站最近更换了硬件系统，服务器的硬件为四路志强处理器、16GB物理内存，操作系统为64位CentOS 5.4，Resin作为Web服务器。整个服务器暂时没有部署别的应用，所有硬件资源都可以提供给这访问量并不算太大的文档网站使用。软件版本选用的是64位的JDK 5，管理员启用了一个虚拟机实例，使用-Xmx和-Xms参数将Java堆大小固定在12GB。使用一段时间后发现服务器的运行效果十分不理想，网站经常不定期出现长时间失去响应。

​	监控服务器运行状况后发现网站失去响应是由垃圾收集停顿所导致的，在该系统软硬件条件下，**HotSpot虚拟机是以服务端模式运行，默认使用的是吞吐量优先收集器**，回收12GB的Java堆，一次FullGC的停顿时间就高达14秒。由于程序设计的原因，访问文档时会把文档从磁盘提取到内存中，导致内存中出现很多由文档序列化产生的大对象，这些大对象大多在分配时就直接进入了老年代，没有在Minor GC中被清理掉。这种情况下即使有12GB的堆，内存也很快会被消耗殆尽，由此导致每隔几分钟出现十几秒的停顿，令网站开发、管理员都对使用Java技术开发网站感到很失望。

​	分析此案例的情况，程序代码问题这里不延伸讨论，程序部署上的主要问题显然是过大的堆内存进行回收时带来的长时间的停顿。<u>经调查，更早之前的硬件使用的是32位操作系统，给HotSpot虚拟机只分配了1.5GB的堆内存，当时用户确实感觉到使用网站比较缓慢，但还不至于发生长达十几秒的明显停顿，后来将硬件升级到64位系统、16GB内存希望能提升程序效能，却反而出现了停顿问题，尝试过将Java堆分配的内存重新缩小到1.5GB或者2GB，这样的确可以避免长时间停顿，但是在硬件上的投资就显得非常浪费</u>。

​	每一款Java虚拟机中的每一款垃圾收集器都有自己的应用目标与最适合的应用场景，如果在特定场景中选择了不恰当的配置和部署方式，自然会事倍功半。

​	**目前单体应用在较大内存的硬件上主要的部署方式有两种：**

1. **通过一个单独的Java虚拟机实例来管理大量的Java堆内存。**
2. **同时使用若干个Java虚拟机，建立逻辑集群来利用硬件资源。**

​	此案例中的管理员采用了第一种部署方式。对于用户交互性强、对停顿时间敏感、内存又较大的系统，并不是一定要使用Shenandoah、ZGC这些明确以控制延迟为目标的垃圾收集器才能解决问题（当然不可否认，如果情况允许的话，这是最值得考虑的方案），使用Parallel Scavenge/Old收集器，并且给Java虚拟机分配较大的堆内存也是有很多运行得很成功的案例的，但前提是**必须把应用的Full GC频率控制得足够低，至少要低到不会在用户使用过程中发生**，譬如十几个小时乃至一整天都不出现一次Full GC，这样<u>可以通过在深夜执行定时任务的方式触发Full GC甚至是自动重启应用服务器来保持内存可用空间在一个稳定的水平</u>。

​	控制Full GC频率的关键是老年代的相对稳定，这主要取决于应用中绝大多数对象能否符合“朝生夕灭”的原则，即**大多数对象的生存时间不应当太长，尤其是不能有成批量的、长生存时间的大对象产生，这样才能保障老年代空间的稳定**。

​	**<u>在许多网站和B/S形式的应用里，多数对象的生存周期都应该是请求级或者页面级的，会话级和全局级的长生命对象相对较少</u>**。只要代码写得合理，实现在超大堆中正常使用没有Full GC应当并不困难，这样的话，使用超大堆内存时，应用响应速度才可能会有所保证。除此之外，如果读者计划使用单个Java虚拟机实例来管理大内存，还需要考虑下面可能面临的问题：

+ 回收大块堆内存而导致的长时间停顿，自从G1收集器的出现，增量回收得到比较好的应用[^31]，这个问题有所缓解，但要到ZGC和Shenandoah收集器成熟之后才得到相对彻底地解决。
+ **<u>大内存必须有64位Java虚拟机的支持，但由于压缩指针、处理器缓存行容量（Cache Line）等因素，64位虚拟机的性能测试结果普遍略低于相同版本的32位虚拟机</u>**。
+ 必须保证应用程序足够稳定，因为这种大型单体应用要是发生了堆内存溢出，几乎无法产生堆转储快照（要产生十几GB乃至更大的快照文件），哪怕成功生成了快照也难以进行分析；如果确实出了问题要进行诊断，可能就必须应用JMC这种能够在生产环境中进行的运维工具。
+ **相同的程序在64位虚拟机中消耗的内存一般比32位虚拟机要大，这是由于指针膨胀，以及数据类型对齐补白等因素导致的，可以开启（默认即开启）压缩指针功能来缓解**。

​	鉴于上述这些问题，现阶段仍然有一些系统管理员选择第二种方式来部署应用：同时使用若干个虚拟机建立逻辑集群来利用硬件资源。做法是在一台物理机器上启动多个应用服务器进程，为每个服务器进程分配不同端口，然后在前端搭建一个负载均衡器，以反向代理的方式来分配访问请求。<u>这里无须太在意均衡器转发所消耗的性能，即使是使用第一个部署方案，多数应用也不止有一台服务器，因此应用中前端的负载均衡器总是免不了的</u>。

​	**<u>考虑到我们在一台物理机器上建立逻辑集群的目的仅仅是尽可能利用硬件资源，并不是要按职责、按领域做应用拆分，也不需要考虑状态保留、热转移之类的高可用性需求，不需要保证每个虚拟机进程有绝对准确的均衡负载，因此使用无Session复制的亲合式集群是一个相当合适的选择</u>**。仅仅需要保障集群具备亲合性，也就是均衡器按一定的规则算法（譬如根据Session ID分配）将一个固定的用户请求永远分配到一个固定的集群节点进行处理即可，这样程序开发阶段就几乎不必为集群环境做任何特别的考虑。

​	当然，第二种部署方案也不是没有缺点的，如果读者计划使用逻辑集群的方式来部署程序，可能会遇到下面这些问题：

+ **节点竞争全局的资源，最典型的就是磁盘竞争，各个节点如果同时访问某个磁盘文件的话（尤其是并发写操作容易出现问题），很容易导致I/O异常**。
+ 很难最高效率地利用某些资源池，譬如连接池，一般都是在各个节点建立自己独立的连接池，这样有可能导致一些节点的连接池已经满了，而另外一些节点仍有较多空余。尽管可以使用集中式的[JNDI](https://baike.baidu.com/item/JDNI/6370333?fr=aladdin)来解决，但这个方案有一定复杂性并且可能带来额外的性能代价。
+ 如果使用32位Java虚拟机作为集群节点的话，各个节点仍然不可避免地受到32位的内存限制，在32位Windows平台中每个进程只能使用2GB的内存，考虑到堆以外的内存开销，堆最多一般只能开到1.5GB。在某些Linux或UNIX系统（如Solaris）中，可以提升到3GB乃至接近4GB的内存，但32位中仍然受最高4GB（2的32次幂）内存的限制。

+ **大量使用本地缓存（如大量使用HashMap作为K/V缓存）的应用，在逻辑集群中会造成较大的内存浪费，因为每个逻辑节点上都有一份缓存，这时候可以考虑把本地缓存改为集中式缓存。**

​	介绍完这两种部署方式，重新回到这个案例之中，最后的部署方案并没有选择升级JDK版本，而是调整为建立5个32位JDK的逻辑集群，每个进程按2GB内存计算（其中堆固定为1.5GB），占用了10GB内存。另外建立一个Apache服务作为前端均衡代理作为访问门户。<u>考虑到用户对响应速度比较关心，并且文档服务的主要压力集中在磁盘和内存访问，处理器资源敏感度较低，因此改为CMS收集器进行垃圾回收。部署方式调整后，服务再没有出现长时间停顿，速度比起硬件升级前有较大提升</u>。

[^31]:以前CMS也有i-CMS的增量回收模式，但与G1的增量回收并不相同，而且并不好用，已被废弃。

### 5.2.2 集群间同步导致的内存溢出

**个人小结**：

​	非集中式的集群缓存，通常允许频繁的读操作。但是在网络资源有限的情况下，也不能无限制的读操作(即使读操作消耗资源少)

*Ps： 一般读操作在并发时涉及的内存屏障开销比写操作少许多，底层锁对读操作来说更是基本用不上了。*

---

> [《深入理解Java虚拟机》-----第5章 jvm调优案例分析与实战](https://www.cnblogs.com/java-chen-hao/p/10579570.html)

​	一个基于B/S的MIS系统，硬件为两台双路处理器、8GB内存的HP小型机，应用中间件是WebLogic9.2，每台机器启动了3个WebLogic实例，构成一个6个节点的亲合式集群。由于是亲合式集群，节点之间没有进行Session同步，但是有一些需求要实现部分数据在各个节点间共享。最开始这些数据是存放在数据库中的，但由于读写频繁、竞争很激烈，性能影响较大，后面使用JBossCache构建了一个全局缓存。全局缓存启用后，服务正常使用了一段较长的时间。但在最近不定期出现多次的内存溢出问题。

​	在内存溢出异常不出现的时候，服务内存回收状况一直正常，每次内存回收后都能恢复到一个稳定的可用空间。开始怀疑是程序某些不常用的代码路径中存在内存泄漏，但管理员反映最近程序并未更新、升级过，也没有进行什么特别操作。只好让服务带着`-XX：+HeapDumpOnOutOfMemoryError`参数运行了一段时间。在最近一次溢出之后，管理员发回了堆转储快照，发现里面存在着大量的org.jgroups.protocols.pbcast.NAKACK对象。

​	JBossCache是基于自家的JGroups进行集群间的数据通信，JGroups使用协议栈的方式来实现收发数据包的各种所需特性自由组合，数据包接收和发送时要经过每层协议栈的up()和down()方法，其中的NAKACK栈用于保障各个包的有效顺序以及重发。

![img](https://img-blog.csdn.net/20151007112624814)

​	**<u>由于信息有传输失败需要重发的可能性，在确认所有注册在GMS（Group Membership Service）的节点都收到正确的信息前，发送的信息必须在内存中保留。而此MIS的服务端中有一个负责安全校验的全局过滤器，每当接收到请求时，均会更新一次最后操作时间，并且将这个时间同步到所有的节点中去，使得一个用户在一段时间内不能在多台机器上重复登录。在服务使用过程中，往往一个页面会产生数次乃至数十次的请求，因此这个过滤器导致集群各个节点之间网络交互非常频繁。当网络情况不能满足传输要求时，重发数据在内存中不断堆积，很快就产生了内存溢出。</u>**

​	这个案例中的问题，既有JBossCache的缺陷，也有MIS系统实现方式上的缺陷。JBoss-Cache官方的邮件讨论组中讨论过很多次类似的内存溢出异常问题，据说后续版本也有了改进。而**<u>更重要的缺陷是，这一类被集群共享的数据要使用类似JBossCache这种非集中式的集群缓存来同步的话，可以允许读操作频繁，因为数据在本地内存有一份副本，读取的动作不会耗费多少资源，但不应当有过于频繁的写操作，会带来很大的网络同步的开销</u>**。

### 5.2.3 堆外内存导致的溢出错误

**个人小结**：

+ <big>**Direct Memory不属于堆，只有在Full GC发生时，才会进行Direct Memory的垃圾回收**</big>。
+ 而堆内存的Young、Old在内存不足时就会自动触发垃圾回收

所以**<u>可能出现明明堆内存稳定，进程却不断抛出OOM异常的情况=>堆外内存不足（Direct Memory等）</u>**

---

> [JVM参数-XX:+HeapDumpOnOutOfMemoryError使用方法](https://blog.csdn.net/lusa1314/article/details/84134458)
>
> `-XX:+HeapDumpOnOutOfMemoryError`参数表示当JVM发生OOM时，自动生成DUMP文件

​	这是一个学校的小型项目：基于B/S的电子考试系统，为了实现客户端能实时地从服务器端接收考试数据，系统使用了逆向AJAX技术（也称为Comet或者Server Side Push），选用CometD 1.1.1作为服务端推送框架，服务器是Jetty 7.1.4，硬件为一台很普通PC机，Core i5 CPU，4GB内存，运行32位Windows操作系统。

​	测试期间发现服务端不定时抛出内存溢出异常，服务不一定每次都出现异常，但假如正式考试时崩溃一次，那估计整场电子考试都会乱套。网站管理员尝试过把堆内存调到最大，32位系统最多到1.6GB基本无法再加大了，而且<u>开大了基本没效果，抛出内存溢出异常好像还更加频繁</u>。加入`-XX：+HeapDumpOnOutOfMemoryError`参数，居然也没有任何反应，抛出内存溢出异常时什么文件都没有产生。无奈之下只好挂着jstat紧盯屏幕，发现垃圾收集并不频繁，Eden区、Survivor区、老年代以及方法区的内存全部都很稳定，压力并不大，但就是照样不停抛出内存溢出异常。最后，在内存溢出后从系统日志中找到异常堆栈如代码清单5-1所示。

​	代码清单5-1　异常堆栈

```java
[org.eclipse.jetty.util.log] handle failed java.lang.OutOfMemoryError: null
at sun.misc.Unsafe.allocateMemory(Native Method)
at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:99)
at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:288)
at org.eclipse.jetty.io.nio.DirectNIOBuffer.<init>
……
```

​	如果认真阅读过本书第2章，看到异常堆栈应该就清楚这个抛出内存溢出异常是怎么回事了。我们知道操作系统对每个进程能管理的内存是有限制的，这台服务器使用的32位Windows平台的限制是2GB，其中划了1.6GB给Java堆，而Direct Memory耗用的内存并不算入这1.6GB的堆之内，因此它最大也只能在剩余的0.4GB空间中再分出一部分而已。在此应用中导致溢出的关键是**<u>垃圾收集进行时，虚拟机虽然会对直接内存进行回收，但是直接内存却不能像新生代、老年代那样，发现空间不足了就主动通知收集器进行垃圾回收，它只能等待老年代满后Full GC出现后，“顺便”帮它清理掉内存的废弃对象</u>**。否则就不得不一直等到抛出内存溢出异常时，先捕获到异常，再在Catch块里面通过`System.gc()`命令来触发垃圾收集。但<u>如果Java虚拟机再打开了`-XX：+DisableExplicitGC`开关，禁止了人工触发垃圾收集的话，那就只能眼睁睁看着堆中还有许多空闲内存，自己却不得不抛出内存溢出异常了</u>。而本案例中使用的CometD 1.1.1框架，正好有大量的NIO操作需要使用到直接内存。

​	从实践经验的角度出发，在处理小内存或者32位的应用问题时，除了Java堆和方法区之外，我们注意到下面这些区域还会占用较多的内存，这里所有的内存总和受到操作系统进程最大内存的限制：

+ **直接内存**：可通过`-XX：MaxDirectMemorySize`调整大小，内存不足时抛出OutOfMemoryError或者OutOfMemoryError：Direct buffer memory。

  <small>*ps我本地jdk8和11都有`uintx MaxDirectMemorySize            = 0`*</small>

+ <u>**线程堆栈**：可通过`-Xss`调整大小，内存不足时抛出StackOverflowError（如果线程请求的栈深度大于虚拟机所允许的深度）或者OutOfMemoryError（如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存）</u>。

+ **<u>Socket缓存区</u>**：**<u>每个Socket连接都Receive和Send两个缓存区，分别占大约37KB和25KB内存，连接多的话这块内存占用也比较可观。如果无法分配，可能会抛出IOException：Too many open files异常。</u>**

+ **JNI代码**：<u>如果代码中使用了JNI调用本地库，那本地库使用的内存也不在堆中，而是占用Java虚拟机的**本地方法栈**和**本地内存**的</u>。
+ 虚拟机和垃圾收集器：虚拟机、垃圾收集器的工作也是要消耗一定数量的内存的。

### 5.2.4 外部命令导致系统缓慢

个人小结：

+ exec()，这个在操作系统笔记也提到过了。复制当前进程环境=>执行其他指定程序（触发COW，写时复制，这时候才与源进程环境分开）=> 即新建一个进程执行指定的程序。
+ 尽量避免新建进程。（毕竟进程资源独立，线程资源来自于当前进程进程的一部分）

---

​	一个数字校园应用系统，运行在一台四路处理器的Solaris 10操作系统上，中间件为GlassFish服务器。系统在做大并发压力测试的时候，发现请求响应时间比较慢，通过操作系统的mpstat工具发现处理器使用率很高，但是系统中占用绝大多数处理器资源的程序并不是该应用本身。这是个不正常的现象，**<u>通常情况下用户应用的处理器占用率应该占主要地位，才能说明系统是在正常工作</u>**。

​	通过Solaris 10的dtrace脚本可以查看当前情况下哪些系统调用花费了最多的处理器资源，dtrace运行后发现最消耗处理器资源的竟然是**“fork”系统调用**。众所周知，**“fork”系统调用是Linux用来产生新进程的**，<u>在Java虚拟机中，用户编写的Java代码通常最多只会创建新的线程，不应当有进程的产生，这又是个相当不正常的现象</u>。

​	通过联系该系统的开发人员，最终找到了答案：每个用户请求的处理都需要执行一个外部Shell脚本来获得系统的一些信息。执行<u>这个Shell脚本是通过Java的Runtime.getRuntime().exec()方法来调用的。这种调用方式可以达到执行Shell脚本的目的，但是它在Java虚拟机中是非常消耗资源的操作，即使外部命令本身能很快执行完毕，频繁调用时创建进程的开销也会非常可观</u>。**Java虚拟机执行这个命令的过程是首先复制一个和当前虚拟机拥有一样环境变量的进程，再用这个新的进程去执行外部命令，最后再退出这个进程**。如果频繁执行这个操作，系统的消耗必然会很大，而且不仅是处理器消耗，内存负担也很重。

​	用户根据建议去掉这个Shell脚本执行的语句，改为使用Java的API去获取这些信息后，系统很快恢复了正常。

### 5.2.5 服务器虚拟机进程崩溃

**个人小结**：

+ 注意通信节点之间的资源不对等问题（网络资源、IO资源）
+ **资源不对等的场景，考虑使用生产者/消费者模式代替异步调用**

---

​	一个基于B/S的MIS系统，硬件为两台双路处理器、8GB内存的HP系统，服务器是WebLogic9.2（与第二个案例中那套是同一个系统）。正常运行一段时间后，最近发现在运行期间频繁出现集群节点的虚拟机进程自动关闭的现象，留下了一个hs_err_pid###.log文件后，虚拟机进程就消失了，两台物理机器里的每个节点都出现过进程崩溃的现象。从系统日志中注意到，每个节点的虚拟机进程在崩溃之前，都发生过大量相同的异常，见代码清单5-2。

​	代码清单5-2　异常堆栈2

```java
java.net.SocketException: Connection reset
at java.net.SocketInputStream.read(SocketInputStream.java:168)
at java.io.BufferedInputStream.fill(BufferedInputStream.java:218)
at java.io.BufferedInputStream.read(BufferedInputStream.java:235)
at org.apache.axis.transport.http.HTTPSender.readHeadersFromSocket(HTTPSender.java:583)
at org.apache.axis.transport.http.HTTPSender.invoke(HTTPSender.java:143)
... 99 more
```

​	这是一个远端断开连接的异常，通过系统管理员了解到系统最近与一个OA门户做了集成，在MIS系统工作流的待办事项变化时，要通过Web服务通知OA门户系统，把待办事项的变化同步到OA门户之中。通过SoapUI测试了一下同步待办事项的几个Web服务，发现调用后竟然需要长达3分钟才能返回，并且返回结果都是超时导致的连接中断。

​	由于MIS系统的用户多，待办事项变化很快，为了不被OA系统速度拖累，<u>使用了异步的方式调用Web服务，但由于两边服务速度的完全不对等，时间越长就累积了越多Web服务没有调用完成，导致在等待的线程和Socket连接越来越多，最终超过虚拟机的承受能力后导致虚拟机进程崩溃</u>。通知OA门户方修复无法使用的集成接口，并将**异步调用改为生产者/消费者模式的消息队列**实现后，系统恢复正常。

### 5.2.6 不恰当数据结构导致内存占用过大

**个人小结**：

+ 治标：JVM存在许多需长期存活的对象时，仅从GC调优角度，可以考虑禁用Survivor，Mirror GC后直接将存活对象存入Old区
+ 治本：修改程序代码，不恰当的数据结构使用，会导致资源利用率低（可通过"对象有效字节/对象实际占用字节"计算利用率）

---

​	一个后台RPC服务器，使用64位Java虚拟机，内存配置为-Xms4g-Xmx8g-Xmn1g，使用ParNew加CMS的收集器组合。平时对外服务的Minor GC时间约在30毫秒以内，完全可以接受。但业务上需要每10分钟加载一个约80MB的数据文件到内存进行数据分析，这些数据会在内存中形成超过100万个HashMap\<Long，Long\>Entry，在这段时间里面Minor GC就会造成超过500毫秒的停顿，对于这种长度的停顿时间就接受不了了，具体情况如下面的收集器日志所示。

```java
{Heap before GC invocations=95 (full 4):
	par new generation total 903168K, used 803142K [0x00002aaaae770000, 0x00002aaaebb70000, 0x00002aaaebb70000)
		eden space 802816K, 100% used [0x00002aaaae770000, 0x00002aaadf770000, 0x00002aaadf770000)
		from space 100352K, 0% used [0x00002aaae5970000, 0x00002aaae59c1910, 0x00002aaaebb70000)
		to space 100352K, 0% used [0x00002aaadf770000, 0x00002aaadf770000, 0x00002aaae5970000)
concurrent mark-sweep generation total 5845540K, used 3898978K [0x00002aaaebb70000, 0x00002aac507f9000, 0x00002aacae770000)
concurrent-mark-sweep perm gen total 65536K, used 40333K [0x00002aacae770000, 0x00002aacb2770000, 0x00002aacb2770000)
2011-10-28T11:40:45.162+0800: 226.504: [GC 226.504: [ParNew: 803142K-> 100352K(903168K), 0.5995670 secs] 4702120K->Heap after GC invocations=96 (full 4):
	par new generation total 903168K, used 100352K [0x00002aaaae770000, 0x00002-aaaebb70000, 0x00002aaaebb70000)
		eden space 802816K, 0% used [0x00002aaaae770000, 0x00002aaaae770000, 0x00002aaadf770000)
		from space 100352K, 100% used [0x00002aaadf770000, 0x00002aaae5970000, 0x00002aaae5970000)
		to space 100352K, 0% used [0x00002aaae5970000, 0x00002aaae5970000, 0x00002aaaebb70000)
concurrent mark-sweep generation total 5845540K, used 3955980K [0x00002aaaebb70000, 0x00002aac507f9000, 0x00002aacae770000)
concurrent-mark-sweep perm gen total 65536K, used 40333K [0x00002aacae770000, 0x00002aacb2770000, 0x00002aacb2770000)
}
Total time for which application threads were stopped: 0.6070570 seconds
```

​	观察这个案例的日志，平时Minor GC时间很短，原因是新生代的绝大部分对象都是可清除的，在Minor GC之后Eden和Survivor基本上处于完全空闲的状态。但是在分析数据文件期间，800MB的Eden空间很快被填满引发垃圾收集，但Minor GC之后，新生代中绝大部分对象依然是存活的。我们知道**<u>ParNew收集器使用的是复制算法，这个算法的高效是建立在大部分对象都“朝生夕灭”的特性上的，如果存活对象过多，把这些对象复制到Survivor并维持这些对象引用的正确性就成为一个沉重的负担，因此导致垃圾收集的暂停时间明显变长</u>**。

​	如果不修改程序，仅从GC调优的角度去解决这个问题，可以考虑直接将Survivor空间去掉（加入参数`-XX：SurvivorRatio=65536`、`-XX：MaxTenuringThreshold=0`或者`-XX：+AlwaysTenure`），**<u>让新生代中存活的对象在第一次Minor GC后立即进入老年代，等到Major GC的时候再去清理它们</u>**。这种措施可以治标，但也有很大副作用；<u>治本的方案必须要修改程序，因为这里产生问题的根本原因是用HashMap\<Long，Long\>结构来存储数据文件空间效率太低了</u>。

​	<u>**我们具体分析一下HashMap空间效率，在HashMap<Long，Long>结构中，只有Key和Value所存放的两个长整型数据是有效数据，共16字节（2×8字节）。这两个长整型数据包装成java.lang.Long对象之后，就分别具有8字节的Mark Word、8字节的Klass指针，再加8字节存储数据的long值。然后这2个Long对象组成Map.Entry之后，又多了16字节的对象头，然后一个8字节的next字段和4字节的int型的hash字段，为了对齐，还必须添加4字节的空白填充，最后还有HashMap中对这个Entry的8字节的引用，这样增加两个长整型数字，实际耗费的内存为(Long(24byte)×2)+Entry(32byte)+HashMapRef(8byte)=88byte，空间效率为有效数据除以全部内存空间，即16字节/88字节=18%，这确实太低了**</u>。

### 5.2.7 由Windows虚拟内存导致的长时间停顿[^31]

> [《深入理解Java虚拟机》-----第5章 jvm调优案例分析与实战](https://www.cnblogs.com/java-chen-hao/p/10579570.html)

​	有一个带心跳检测功能的GUI桌面程序，每15秒会发送一次心跳检测信号，如果对方30秒以内都没有信号返回，那就认为和对方程序的连接已经断开。程序上线后发现心跳检测有误报的可能，查询日志发现误报的原因是程序会偶尔出现间隔约一分钟的时间完全无日志输出，处于停顿状态。

​	因为是桌面程序，所需的内存并不大（`-Xmx256m`），所以开始并没有想到是垃圾收集导致的程序停顿，但是加入参数`-XX：+PrintGCApplicationStoppedTime`，`-XX：+PrintGCDateStamps`，`-Xloggc：gclog.log`后，从收集器日志文件中确认了停顿确实是由垃圾收集导致的，大部分收集时间都控制在100毫秒以内，但偶尔就出现一次接近1分钟的长时间收集过程。

```java
Total time for which application threads were stopped: 0.0112389 seconds
Total time for which application threads were stopped: 0.0001335 seconds
Total time for which application threads were stopped: 0.0003246 seconds
Total time for which application threads were stopped: 41.4731411 seconds
Total time for which application threads were stopped: 0.0489481 seconds
Total time for which application threads were stopped: 0.1110761 seconds
Total time for which application threads were stopped: 0.0007286 seconds
Total time for which application threads were stopped: 0.0001268 seconds
```

​	从收集器日志中找到长时间停顿的具体日志信息（再添加了`-XX：+PrintReferenceGC`参数），找到的日志片段如下所示。从日志中看到，真正执行垃圾收集动作的时间不是很长，但从准备开始收集，到真正开始收集之间所消耗的时间却占了绝大部分。

![img](https://img-blog.csdn.net/20151008223224850)

​	**<u>除收集器日志之外，还观察到这个GUI程序内存变化的一个特点，当它最小化的时候，资源管理中显示的占用内存大幅度减小，但是虚拟内存则没有变化，因此怀疑程序在最小化时它的工作内存被自动交换到磁盘的页面文件之中了，这样发生垃圾收集时就有可能因为恢复页面文件的操作导致不正常的垃圾收集停顿</u>**。

​	在MSDN上查证[^32]确认了这种猜想，在Java的GUI程序中要避免这种现象，可以加入参数“`-Dsun.awt.keepWorkingSetOnMinimize=true`”来解决。这个参数在许多AWT的程序上都有应用，例如JDK（曾经）自带的VisualVM，启动配置文件中就有这个参数，保证程序在恢复最小化时能够立即响应。在这个案例中加入该参数，问题马上得到解决。

[^31]:本案例来源于ITEye HLLVM群组的讨论：http://hllvm.group.iteye.com/group/topic/28745。
[^32]:http://support.microsoft.com/default.aspx?scid=kb；en-us；293215。

### 5.2.8 由安全点导致长时间停顿[^33]

个人小结：

+ **处理器时间代表的是线程占用处理器一个核心的耗时计数，而时钟时间就是现实世界中的时间计数**

  <small>（<u>单核单线程的场景下，这两者可以认为是等价的，但如果是多核环境下，同一个时钟时间内有多少处理器核心正在工作，就会有多少倍的处理器时间被消耗和记录下来</u>）</small>

+ **安全点是以“是否具有让程序长时间执行的特征”为原则进行选定的**，所以方法调用、循环跳转、异常跳转这些位置都可能会设置有安全点，但是HotSpot虚拟机为了避免安全点过多带来过重的负担，对循环还有一项优化措施，认为循环次数较少的话，执行时间应该也不会太长，**<u>所以使用int类型或范围更小的数据类型作为索引值的循环默认是不会被放置安全点的</u>**。<u>这种循环被称为**可数循环（CountedLoop）**，相对应地，使用long或者范围更大的数据类型作为索引值的循环就被称为**不可数循环（Uncounted Loop）**，将会被放置安全点</u>。

  + 可数循环（CountedLoop）：使用int类型或范围更小的数据类型作为索引值的循环
  + 不可数循环（UncountedLoop）：使用long或者范围更大的数据类型作为索引值的循环

---

> [调优案例分析与实战-1- （深入理解java虚拟机）](https://www.cnblogs.com/yanliang12138/p/12732191.html)

​	有一个比较大的承担公共计算任务的离线HBase集群，运行在JDK 8上，使用G1收集器。每天都有大量的MapReduce或Spark离线分析任务对其进行访问，同时有很多其他在线集群Replication过来的数据写入，因为集群读写压力较大，而离线分析任务对延迟又不会特别敏感，所以将`-XX：MaxGCPauseMillis`参数设置到了500毫秒。不过运行一段时间后发现垃圾收集的停顿经常达到3秒以上，而且实际垃圾收集器进行回收的动作就只占其中的几百毫秒，现象如以下日志所示。

![img](https://img2020.cnblogs.com/blog/712711/202004/712711-20200419163451362-414429794.png)

​	考虑到不是所有读者都了解计算机体系和操作系统原理，笔者先解释一下user、sys、real这三个时间的概念：

+ **user：进程执行用户态代码所耗费的处理器时间。**

+ **sys：进程执行核心态代码所耗费的处理器时间。**

+ **real：执行动作从开始到结束耗费的时钟时间。**

​	请注意，前面两个是处理器时间，而最后一个是时钟时间，它们的区别是**<u>处理器时间代表的是线程占用处理器一个核心的耗时计数，而时钟时间就是现实世界中的时间计数</u>**。如果是<u>单核单线程的场景下，这两者可以认为是等价的，但如果是多核环境下，同一个时钟时间内有多少处理器核心正在工作，就会有多少倍的处理器时间被消耗和记录下来</u>。

​	**<u>在垃圾收集调优时，我们主要依据real时间为目标来优化程序，因为最终用户只关心发出请求到得到响应所花费的时间，也就是响应速度，而不太关心程序到底使用了多少个线程或者处理器来完成任务</u>**。日志显示这次垃圾收集一共花费了0.14秒，但其中用户线程却足足停顿了有2.26秒，两者差距已经远远超出了正常的TTSP（Time To Safepoint）耗时的范畴。所以先加入参数`-XX：+PrintSafepointStatistics`和`-XX:PrintSafepointStatisticsCount=1`去查看安全点日志，具体如下所示：

```java
vmop [threads: total initially_running wait_to_block]
65968.203: ForceAsyncSafepoint [931 1 2]
[time: spin block sync cleanup vmop] page_trap_count
[2255 0 2255 11 0] 1
```

​	日志显示当前虚拟机的操作（VM Operation，VMOP）是等待所有用户线程进入到安全点，但是有两个线程特别慢，导致发生了很长时间的自旋等待。<u>日志中的2255毫秒自旋（Spin）时间就是指由于部分线程已经走到了安全点，但还有一些特别慢的线程并没有到，所以垃圾收集线程无法开始工作，只能空转（自旋）等待</u>。

​	解决问题的第一步是把这两个特别慢的线程给找出来，这个倒不困难，添加`-XX：+SafepointTimeout`和`-XX：SafepointTimeoutDelay=2000`两个参数，让虚拟机在等到线程进入安全点的时间超过2000毫秒时就认定为超时，这样就会输出导致问题的线程名称，得到的日志如下所示：

```java
# SafepointSynchronize::begin: Timeout detected:
# SafepointSynchronize::begin: Timed out while spinning to reach a safepoint.
# SafepointSynchronize::begin: Threads which did not reach the safepoint:
# "RpcServer.listener,port=24600" #32 daemon prio=5 os_prio=0 tid=0x00007f4c14b22840
nid=0xa621 runnable [0x0000000000000000]
java.lang.Thread.State: RUNNABLE
# SafepointSynchronize::begin: (End of list)
```

​	从错误日志中顺利得到了导致问题的线程名称为“RpcServer.listener，port=24600”。但是为什么它们会出问题呢？有什么因素可以阻止线程进入安全点？在第3章关于安全点的介绍中，我们已经知道**<u>安全点是以“是否具有让程序长时间执行的特征”为原则进行选定的，所以方法调用、循环跳转、异常跳转这些位置都可能会设置有安全点，但是HotSpot虚拟机为了避免安全点过多带来过重的负担，对循环还有一项优化措施，认为循环次数较少的话，执行时间应该也不会太长，所以使用int类型或范围更小的数据类型作为索引值的循环默认是不会被放置安全点的</u>**。<u>这种循环被称为**可数循环（CountedLoop）**，相对应地，使用long或者范围更大的数据类型作为索引值的循环就被称为**不可数循环（Uncounted Loop）**，将会被放置安全点</u>。**通常情况下这个优化措施是可行的，但循环执行的时间不单单是由其次数决定，如果循环体单次执行就特别慢，那即使是可数循环也可能会耗费很多的时间**。

​	**HotSpot原本提供了`-XX：+UseCountedLoopSafepoints`参数去强制在可数循环中也放置安全点，不过这个参数在JDK 8下有Bug[^34]，有导致虚拟机崩溃的风险**，所以就不得不找到RpcServer线程里面的缓慢代码来进行修改。最终查明导致这个问题是HBase中一个连接超时清理的函数，由于集群会有多个MapReduce或Spark任务进行访问，而每个任务又会同时起多个Mapper/Reducer/Executer，其每一个都会作为一个HBase的客户端，这就导致了同时连接的数量会非常多。**<u>更为关键的是，清理连接的索引值就是int类型，所以这是一个可数循环，HotSpot不会在循环中插入安全点</u>**。当垃圾收集发生时，如果RpcServer的Listener线程刚好执行到该函数里的可数循环时，则必须等待循环全部跑完才能进入安全点，此时其他线程也必须一起等着，所以从现象上看就是长时间的停顿。找到了问题，解决起来就非常简单了，<u>把循环索引的数据类型从int改为long即可，但如果不具备安全点和垃圾收集的知识，这种问题是很难处理的</u>。

[^33]:原始案例来自“小米云技术”公众号，原文地址为https://juejin.im/post/5d1b1fc46fb9a07ef7108d82，笔者做了一些改动。
[^34]:https://bugs.openjdk.java.net/browse/JDK-8161147。

## 5.3 实战：Eclipse运行速度调优

​	很多Java开发人员都有一种错觉，认为系统调优的工作都是针对服务端应用的，规模越大的系统，就需要越专业的调优运维团队参与。这个观点不能说不对，只是有点狭隘了。上一节中笔者所列举的案例确实大多是服务端运维、调优的例子，但不只服务端需要调优，其他应用类型也是需要的，作为一个普通的Java开发人员，学习到的各种虚拟机的原理和最佳实践方法距离我们并不遥远，开发者身边就有很多场景可以使用上这些知识。下面就通过一个普通程序员日常工作中可以随时接触到的开发工具开始这次实战。[^35]

[^35]:此实战是本书第2版时编写的内容，今天看来里面的Eclipse和HotSpot版本已经较旧，不过软件版本的落后并未影响笔者要表达的意图，本案例目前也仍然有相同的实战价值，所以在第3版里笔者并未刻意将Eclipse和HotSpot升级后重写一次。

### 5.3.1 调优前的程序运行状态

### 5.3.2 升级JDK版本的性能变化及兼容问题

+ JDK5、JDK6都有`-XX：MaxPermSize=64MB`，即永久代默认最大容量64MB

+ **版本升级也有不少性能倒退的案例，受程序、第三方包兼容性以及中间件限制，在企业应用中升级JDK版本是一件需要慎重考虑的事情。**

### 5.3.3 编译时间和类加载时间的优化

个人小结：

1. 除了类加载时间以外，在VisualGC中监视曲线中显示了两项很大的**非用户程序耗时**：

   + **编译时间（Compile Time）**

   + **垃圾收集时间（GC Time）**

2. **<u>编译时间是指虚拟机的即时编译器（Just In Time Compiler）编译热点代码（Hot SpotCode）的耗时</u>**

3. Java语言为了实现跨平台的特性，Java代码编译出来后形成Class文件中储存的是字节码（Byte Code），虚拟机通过解释方式执行字节码命令，比起C/C++编译成本地二进制代码来说，速度要慢不少。

4. **为了解决程序解释执行的速度问题，JDK 1.2以后，HotSpot虚拟机内置了两个即时编译器**[^39]

5. 如果一段Java方法被调用次数到达一定程度，就会被判定为**热代码**交给<u>即时编译器</u>**即时编译为本地代码**，提高运行速度（这就是HotSpot虚拟机名字的来由）

   *<small><u>**完全有可能在运行期动态编译比C/C++的编译期静态编译出来的结果要更加优秀**，因为运行期的编译器可以收集很多静态编译器无法得知的信息，也可以采用一些激进的优化手段，针对“大多数情况”而忽略“极端情况”进行假设优化，当优化条件不成立的时候再逆优化退回到解释状态或者重新编译执行</u></small>*

6. **随着运行时间增长，java代码被编译得越来越彻底，运行速度应当是越运行越快的**。不过，**<u>Java的运行期编译的一大缺点就是它进行编译需要消耗机器的计算资源，影响程序正常的运行时间，这也就是上面所说的“编译时间”</u>**。

7. HotSpot虚拟机提供了一个参数`-Xint`来禁止编译器运作，强制虚拟机对字节码采用纯解释方式执行

8. 与解释执行相对应的另一方面，HotSpot虚拟机还有另一个力度更强的即时编译器：<u>当虚拟机运行在客户端模式的时候，使用的是一个代号为C1的轻量级编译器，另外还有一个代号为C2的相对重量级的服务端编译器能提供更多的优化措施</u>。

---

​	从Eclipse启动时间来看，升级到JDK 6所带来的性能提升是……嗯？基本上没有提升。多次测试的平均值与JDK 5的差距完全在实验误差范围之内。

​	各位读者不必失望，Sun公司给的JDK 6性能白皮书[^36]描述的众多相对于JDK 5的提升并不至于全部是广告词，尽管总启动时间并没有减少，但在查看运行细节的时候，却发现了一件很令人玩味的事情：在JDK 6中启动完Eclipse所消耗的类加载时间比JDK 5长了接近一倍，读者注意不要看反了，这里写的是JDK 6的类加载比JDK 5慢一倍，测试结果见代码清单5-7，反复测试多次仍然是相似的结果。

​	代码清单5-7　JDK 5、JDK 6中的类加载时间对比

```java
使用JDK 6的类加载时间：
C:\Users\IcyFenix>jps
3552
6372 org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
6900 Jps
C:\Users\IcyFenix>jstat -class 6372
Loaded Bytes Unloaded Bytes Time
	7917 10190.3 0 0.0 8.18
使用JDK 5类加载时间：
C:\Users\IcyFenix>jps
3552
7272 Jps
7216 org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
C:\Users\IcyFenix>jstat -class 7216
Loaded Bytes Unloaded Bytes Time
	7902 9691.2 3 2.6 4.34
```

​	在本例中类加载时间上的差距并不能作为一个具有普适性的测试结论去说明JDK 6的类加载必然比JDK 5慢，笔者测试了自己机器上的Tomcat和GlassFish启动过程，并没有出现类似的差距。在国内最大的Java社区中，笔者发起过关于此问题的讨论[^37]。从参与者反馈的测试结果来看，此问题只在一部分机器上存在，而且在JDK 6的各个更新包之间，测试结果也存在很大差异。

​	经多轮试验后，发现在笔者机器上两个JDK进行类加载时，<u>字节码验证部分</u>耗时差距尤其严重，暂且认为是JDK 6中新加入类型检查验证器时，可能在某些机器上会影响到以前类型检查验证器的工作[^38]。考虑到实际情况，Eclipse使用者甚多，它的编译代码我们可以认为是安全可靠的，可以不需要在加载的时候再进行字节码验证，因此通过参数`-Xverify：none`禁止掉字节码验证过程也可作为一项优化措施。加入这个参数后，两个版本的JDK类加载速度都有所提高，此时JDK 6的类加载速度仍然比JDK 5要慢，但是两者的耗时已经接近了很多，测试结果如代码清单5-8所示。

```java
使用JDK 1.6的类加载时间：
C:\Users\IcyFenix>jps
5512 org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
5596 Jps
C:\Users\IcyFenix>jstat -class 5512
Loaded Bytes Unloaded Bytes Time
	6749 8837.0 0 0.0 3.94
使用JDK 1.5的类加载时间：
C:\Users\IcyFenix>jps
4724 org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
5412 Jps
C:\Users\IcyFenix>jstat -class 4724
Loaded Bytes Unloaded Bytes Time
	6885 9109.7 3 2.6 3.10
```

​	关于类与类加载的话题，譬如刚刚提到的字节码验证是怎么回事，本书专门规划了两个章节进行详细讲解，在此暂不再展开了。

​	在取消字节码验证之后，JDK 5的平均启动下降到了13秒，而在JDK 6的测试数据平均比JDK 5快了1秒左右，下降到平均12秒。在类加载时间仍然落后的情况下，依然可以看到JDK 6在性能上确实比JDK 5略有优势，说明至少在Eclipse启动这个测试用例上，升级JDK版本确实能带来一些“免费的”性能提升。

​	<u>前面提到过，除了类加载时间以外，在VisualGC中监视曲线中显示了两项很大的非用户程序耗时：**编译时间（Compile Time）和垃圾收集时间（GC Time）**</u>。垃圾收集时间读者应该非常清楚了，而编译时间是什么东西？程序在运行之前不是已经编译了吗？

​	虚拟机的即时编译与垃圾收集一样，是本书的一个重点部分，后面有专门章节讲解，这里先简要介绍一下：**<u>编译时间是指虚拟机的即时编译器（Just In Time Compiler）编译热点代码（Hot SpotCode）的耗时</u>**。

​	<u>我们知道Java语言为了实现跨平台的特性，Java代码编译出来后形成Class文件中储存的是字节码（Byte Code），虚拟机通过解释方式执行字节码命令，比起C/C++编译成本地二进制代码来说，速度要慢不少</u>。**为了解决程序解释执行的速度问题，JDK 1.2以后，HotSpot虚拟机内置了两个即时编译器**[^39]，<u>如果一段Java方法被调用次数到达一定程度，就会被判定为热代码交给即时编译器即时编译为本地代码，提高运行速度（这就是HotSpot虚拟机名字的来由）</u>。而且<u>**完全有可能在运行期动态编译比C/C++的编译期静态编译出来的结果要更加优秀**，因为运行期的编译器可以收集很多静态编译器无法得知的信息，也可以采用一些激进的优化手段，针对“大多数情况”而忽略“极端情况”进行假设优化，当优化条件不成立的时候再逆优化退回到解释状态或者重新编译执行</u>。所以Java程序只要代码编写没有问题（典型的是各种泄漏问题，如内存泄漏、连接泄漏），**随着运行时间增长，代码被编译得越来越彻底，运行速度应当是越运行越快的**。不过，**<u>Java的运行期编译的一大缺点就是它进行编译需要消耗机器的计算资源，影响程序正常的运行时间，这也就是上面所说的“编译时间”</u>**。

​	<u>HotSpot虚拟机提供了一个参数`-Xint`来禁止编译器运作，强制虚拟机对字节码采用纯解释方式执行</u>。如果读者想使用这个参数省下Eclipse启动中那2秒的编译时间获得一个哪怕只是“更好看”的启动成绩的话，那恐怕要大失所望了，加上这个参数之后虽然编译时间确实下降到零，但Eclipse启动的总时间却剧增到27秒，就是因为没有即时编译的支持，执行速度大幅下降了。现在这个参数最大的作用，除了某些场景调试上的需求外，似乎就剩下让用户缅怀一下JDK 1.2之前Java语言那令人心酸心碎的运行速度了。

​	与解释执行相对应的另一方面，HotSpot虚拟机还有另一个力度更强的即时编译器：<u>当虚拟机运行在客户端模式的时候，使用的是一个代号为C1的轻量级编译器，另外还有一个代号为C2的相对重量级的服务端编译器能提供更多的优化措施</u>。由于本次实战所采用的HotSpot版本还不支持多层编译，所以虚拟机只会单独使用其中一种即时编译器，如果使用客户端模式的虚拟机启动Eclipse将会使用到C2编译器，这时从VisualGC可以看到启动过程中虚拟机使用了超过15秒的时间去进行代码编译。如果读者的工作习惯是长时间不会关闭Eclipse的话，服务端编译器所消耗的额外编译时间最终是会在运行速度的提升上“赚”回来的，这样使用服务端模式是一个相当不错的选择。不过至少在本次实战中，我们还是继续选用客户端虚拟机来运行Eclipse。

[^36]:白皮书：http://java.sun.com/performance/reference/whitepapers/6_performance.html。
[^37]:笔者发起的关于JDK 6与JDK 5在Eclipse启动时类加载速度差异的讨论：http://www.javaeye.com/topic/826542。
[^38]:这部分内容可常见第7章关于类加载过程的介绍。
[^39]:JDK 1.2之前也可以使用外挂JIT编译器进行本地编译，但只能与解释器二选其一，不能同时工作。

### 5.3.4 调整内存设置控制垃圾收集频率

个人小结：

+ **在绝大多数的应用中，都不可能出现持续不断的类被加载和卸载。在程序运行一段时间后，随着热点方法被不断编译，新的热点方法数量也总会下降，这都会让类加载和即时编译的影响随运行时间增长而下降，但是<u>垃圾收集则是随着程序运行而持续运作的，所以它对性能的影响才显得最为重要</u>**。

+ 垃圾回收而导致的停顿，严格来说，不包括正在执行native代码的用户线程，因为native代码一般不会改变Java对象的引用关系，所以没有必要挂起它们来等待垃圾回收。

+ 新生代垃圾收集频繁发生时，需考虑是否由虚拟机分配给新生代的空间太小导致

  *<small>使用`-Xmn`参数可手工调整新生代的大小</small>*

+ Full GC频繁触发时，需考虑是否老年代空间耗尽（频繁Full GC、频繁扩容）

+ 我们可以把`-Xms`和`-XX：PermSize`参数值设置为`-Xmx`和`-XX：MaxPermSize`参数值一样，这样就强制虚拟机在启动的时候就把老年代和永久代的容量固定下来，避免运行时自动扩展

+ **需要说明一点，虚拟机启动的时候就会把参数中所设定的内存全部划为私有，即使扩容前有一部分内存不会被用户代码用到，这部分内存也不会交给其他进程使用。这部分内存在虚拟机中被标识为“Virtual”内存**。

+ 在老年代空间充足时，却出现Full GC，可观察是否程序中主动执行了`System.gc()`

---

> [[深入理解Java虚拟机]第五章 调优案例分析与实战](https://blog.csdn.net/coslay/article/details/48950677?utm_source=blogxgwz6)

​	<u>三大块非用户程序时间中，还剩下“GC时间”没有调整，而“GC时间”却又是其中最重要的一块，并不单单因为它是耗时最长的一块，更因为它是一个稳定持续的消耗</u>。由于我们做的测试是在测程序的启动时间，类加载和编译时间的影响力在这项测试里被大幅放大了。**在绝大多数的应用中，都不可能出现持续不断的类被加载和卸载**。**在程序运行一段时间后，随着热点方法被不断编译，新的热点方法数量也总会下降，这都会让类加载和即时编译的影响随运行时间增长而下降，但是垃圾收集则是随着程序运行而持续运作的，所以它对性能的影响才显得最为重要**。

​	在Eclipse启动的原始数据样本中，短短15秒，类共发生了19次Full GC和378次Minor GC，一共397次GC共造成了超过4秒的停顿，也就是超过1/4的时间都是在做垃圾收集，这样的运行数据看起来实在太糟糕了。

​	首先来解决新生代中的Minor GC，尽管垃圾收集的总时间只有不到1秒，但却发生了378次之多。从VisualGC的线程监视中看到Eclipse启动期间一共发起了超过70条线程，同时在运行的线程数超过25条，<u>每当发生一次垃圾收集，所有用户线程[^40]都必须跑到最近的一个**安全点**然后挂起线程来等待垃圾回收。这样过于频繁的垃圾收集就会导致很多没有必要的线程挂起及恢复动作。</u>

​	<u>新生代垃圾收集频繁发生，很明显是由于虚拟机分配给新生代的空间太小导致</u>，Eden区加上一个Survivor区的总大小还不到35MB。所以完全有必要使用`-Xmn`参数手工调整新生代的大小。

​	再来看一看那19次Full GC，看起来19次相对于378次Minor GC来说并“不多”，但总耗时有3.166秒，占了绝大部分的垃圾收集时间，降低垃圾收集停顿时间的主要目标就是要降低Full GC这部分时间。从VisualGC的曲线图上看得不够精确，这次直接从收集器日志[^41]中分析一下这些Full GC是如何产生的，代码清单5-9中是启动最开始的2.5秒内发生的10次Full GC记录。

​	代码清单5-9　Full GC记录

```java
0.278: [GC 0.278: [DefNew: 574K->33K(576K), 0.0012562 secs]
0.279: [Tenured: 1467K->997K(1536K), 0.0181775 secs] 
0.312: [GC 0.312: [DefNew: 575K->64K(576K), 0.0004974 secs]
0.312: [Tenured: 1544K->1608K(1664K), 0.0191592 secs] 
0.590: [GC 0.590: [DefNew: 576K->64K(576K), 0.0006360 secs]
0.590: [Tenured: 2675K->2219K(2684K), 0.0256020 secs] 
0.958: [GC 0.958: [DefNew: 551K->64K(576K), 0.0011433 secs]
0.959: [Tenured: 3979K->3470K(4084K), 0.0419335 secs] 
1.575: [Full GC ....
1.575: [Tenured: 4800K->5046K(5784K), 0.0543136 secs] 5189K->5046K(6360K), [Perm : 12287K->12287K(1.703: [GC 1.703: [DefNew: 703K->63K(704K), 0.0012609 secs]
1.705: [Tenured: 8441K->8505K(8540K), 0.0607638 secs] 
1.837: [GC 1.837: [DefNew: 1151K->64K(1152K), 0.0020698 secs]
1.839: [Tenured: 14616K->14680K(14688K), 0.0708748 ....
2.144: [GC 2.144: [DefNew: 1856K->191K(1856K), 0.0026810 secs]
2.147: [Tenured: 25092K->24656K(25108K), 0.1112429 ....
2.337: [GC 2.337: [DefNew: 1914K->0K(3136K), 0.0009697 secs]
2.338: [Tenured: 41779K->27347K(42056K), 0.0954341 secs] 
2.465: [GC 2.465: [DefNew: 2490K->0K(3456K), 0.0011044 secs]
2.466: [Tenured: 46379K->27635K(46828K), 0.0956937 secs]
```

​	括号中加粗的数字代表着老年代的容量，这组GC日志显示，<u>10次Full GC发生的原因全部都是老年代空间耗尽</u>，每发生一次Full GC都伴随着一次老年代空间扩容：1536KB→1664KB→2684KB→…→42056KB→46828KB。10次GC以后老年代容量从起始的1536KB扩大到46828KB，当15秒后Eclipse启动完成时，老年代容量扩大到了103428KB，代码编译开始后，老年代容量到达顶峰473MB，整个Java堆到达最大容量512MB。

​	日志还显示有些时候内存回收状况很不理想，空间扩容成为获取可用内存的最主要手段，譬如这一句：

```java
Tenured: 25092K->24656K(25108K) , 0.1112429 secs
```

​	代表老年代当前容量为25108KB，内存使用到25092KB的时候发生了Full GC，花费0.11秒把内存使用降低到24656KB，只回收了不到500KB的内存，这次垃圾收集基本没有什么回收效果，仅仅做了扩容，**扩容过程相比起回收过程可以看作是基本不需要花费时间的**，所以说这0.11秒几乎是平白浪费了。

​	由上述分析可以得出结论：<u>Eclipse启动时Full GC大多数是由于老年代容量扩展而导致的，由永久代空间扩展而导致的也有一部分。为了避免这些扩展所带来的性能浪费，我们可以把`-Xms`和`-XX：PermSize`参数值设置为`-Xmx`和`-XX：MaxPermSize`参数值一样，这样就强制虚拟机在启动的时候就把老年代和永久代的容量固定下来，避免运行时自动扩展[^42]。</u>

​	根据以上分析，优化计划确定为：

+ 把新生代容量提升到128MB，避免新生代频繁发生Minor GC；
+ 把Java堆、永久代的容量分别固定为512MB和96MB[^43]，避免内存扩展。

​	这几个数值都是根据机器硬件和Eclipse插件、工程数量决定，读者实战的时候应依据VisualGC和日志里收集到的实际数据进行设置。改动后的eclipse.ini配置如代码清单5-10所示。

​	代码清单5-10　内存调整后的Eclipse配置文件

```java
-vm
D:/_DevSpace/jdk1.6.0_21/bin/javaw.exe
-startup
plugins/org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
--launcher.library
plugins/org.eclipse.equinox.launcher.win32.win32.x86_1.0.200.v20090519
-product
org.eclipse.epp.package.jee.product
-showsplash
org.eclipse.platform
-vmargs
-Dosgi.requiredJavaVersion=1.5
-Xverify:none
-Xmx512m
-Xms512m
-Xmn128m
-XX:PermSize=96m
-XX:MaxPermSize=96m
```

​	现在这个配置之下，垃圾收集的次数已经大幅度降低，图5-9是Eclipse启动后一分钟的监视曲线，只发生了8次Minor GC和4次Full GC，总耗时为1.928秒。

![img](https://img-blog.csdn.net/20151008225154149)

​	这个结果已经算是基本正常，但是还存在一点瑕疵：从Old Gen的曲线上看，老年代直接固定在384MB，而内存使用量只有66MB，并且一直很平滑，完全不应该发生Full GC才对，那4次Full GC是怎么来的？使用`jstat-gccause`查询一下最近一次GC的原因，见代码清单5-11。

​	代码清单5-11　查询GC原因

```java
C:\Users\IcyFenix>jps
9772 Jps
4068 org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
C:\Users\IcyFenix>jstat -gccause 4068
S0 S1 E O P YGC YGCT FGC FGCT GCT
LGCC GCC
0.00 0.00 1.00 14.81 39.29 6 0.422 20 5.992 6.414
System.gc() No GC
```

​	<u>从LGCC（Last GC Cause）中看到原来是代码调用`System.gc()`显式触发的垃圾收集，在内存设置调整后，这种显式垃圾收集不符合我们的期望，因此在eclipse.ini中加入参数`-XX：+DisableExplicitGC`屏蔽掉`System.gc()`</u>。再次测试发现启动期间的Full GC已经完全没有了，只发生了6次Minor GC，总共耗时417毫秒，与调优前4.149秒的测试结果相比，正好是十分之一。进行GC调优后Eclipse的启动时间下降非常明显，比整个垃圾收集时间降低的绝对值还大，现在启动只需要7秒多。

[^40]:严格来说，不包括正在执行native代码的用户线程，因为native代码一般不会改变Java对象的引用关系，所以没有必要挂起它们来等待垃圾回收。
[^41]:可以通过以下几个参数要求虚拟机生成GC日志：-XX：+PrintGCTimeStamps（打印GC停顿时间）、-XX：+PrintGCDetails（打印GC详细信息）、-verbose：gc（打印GC信息，输出内容已被前一个参数包括，可以不写）、-Xloggc：gc.log。
[^42]:需要说明一点，虚拟机启动的时候就会把参数中所设定的内存全部划为私有，即使扩容前有一部分内存不会被用户代码用到，这部分内存也不会交给其他进程使用。这部分内存在虚拟机中被标识为“Virtual”内存。
[^43]:512MB和96MB两个数值对于笔者的应用情况来说依然偏少，但由于笔者需要同时开VMware虚拟机工作，所以需要预留较多内存，读者在实际调优时不妨再设置大一些。

### 5.3.5 选择收集器降低延迟

个人小结：

+ ParNew是使用CMS收集器后的默认新生代收集器
+ 

> [[深入理解Java虚拟机]第五章 调优案例分析与实战](https://blog.csdn.net/coslay/article/details/48950677?utm_source=blogxgwz6)

​	现在Eclipse启动已经比较迅速了，但我们的调优实战还没有结束，毕竟Eclipse是拿来写程序用的，不是拿来测试启动速度的。我们不妨再在Eclipse中进行一个非常常用但又比较耗时的操作：代码编译。图5-11是当前配置下，Eclipse进行代码编译时的运行数据，从图中可以看到，新生代每次回收耗时约65毫秒，老年代每次回收耗时约725毫秒。对于用户来说，新生代垃圾收集的耗时也还好，65毫秒的停顿在使用中基本无法察觉到，而老年代每次垃圾收集要停顿接近1秒钟，虽然较长时间才会出现一次，但这样的停顿已经是可以被人感知了，会影响到体验。

![img](https://img-blog.csdn.net/20151009003705592)

​	再注意看一下编译期间的处理器资源使用状况，整个编译过程中平均只使用了不到30%的处理器资源，垃圾收集的处理器使用率曲线更是几乎与坐标横轴紧贴在一起，这说明处理器资源还有很多可利用的余地。

​	列举垃圾收集的停顿时间、处理器资源富余的目的，都是为了给接下来替换掉客户端模式的虚拟机中默认的新生代、老年代串行收集器做个铺垫。

​	Eclipse应当算是与使用者交互非常频繁的应用程序，由于代码太多，笔者习惯在做全量编译或者清理动作的时候，使用“Run in Background”功能一边编译一边继续工作。回顾一下在第3章提到的几种收集器，很容易想到在JDK 6版本下提供的收集器里，CMS是最符合这类场景的选择。我们在eclipse.ini中再加入这两个参数，`-XX：+UseConc-MarkSweepGC`和`-XX：+UseParNewGC`（**ParNew是使用CMS收集器后的默认新生代收集器**，写上仅是为了配置更加清晰），要求虚拟机在新生代和老年代分别使用ParNew和CMS收集器进行垃圾回收。指定收集器之后，再次测试的结果如图5-13所示，与原来使用串行收集器对比，新生代停顿从每次65毫秒下降到了每次53毫秒，而<u>老年代的**停顿时间**更是从725毫秒大幅下降到了36毫秒</u>。

​	当然，由于CMS的停顿时间只是整个收集过程中的一小部分，大部分收集行为是与用户程序并发进行的，所以并不是真的把垃圾收集时间从725毫秒直接缩短到36毫秒了。在收集器日志中可以看到CMS与程序并发的时间约为400毫秒，这样收集器的运行结果就比较令人满意了。

​	到这里为止，对于虚拟机内存的调优基本就结束了，这次实战可以看作一次简化的服务端调优过程，**<u>服务端调优有可能还会在更多方面，如数据库、资源池、磁盘I/O等</u>**，但对于虚拟机内存部分的优化，与这次实战中的思路没有什么太大差别。即使读者实际工作中不接触到服务器，根据自己工作环境做一些试验，总结几个参数让自己日常工作环境速度有较大幅度提升也是很能提升工作幸福感的。最终eclipse.ini的配置如代码清单5-12所示。

​	代码清单5-12　修改收集器配置后的Eclipse配置

```java
-vm
D:/_DevSpace/jdk1.6.0_21/bin/javaw.exe
-startup
plugins/org.eclipse.equinox.launcher_1.0.201.R35x_v20090715.jar
--launcher.library
plugins/org.eclipse.equinox.launcher.win32.win32.x86_1.0.200.v20090519
-product
org.eclipse.epp.package.jee.product
-showsplash
org.eclipse.platform
-vmargs
-Dcom.sun.management.jmxremote
-Dosgi.requiredJavaVersion=1.5
-Xverify:none
-Xmx512m
-Xms512m
-Xmn128m
-XX:PermSize=96m
-XX:MaxPermSize=96m
-XX:+DisableExplicitGC
-Xnoclassgc
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=85
```

## 5.4 本章小结

​	Java虚拟机的内存管理与垃圾收集是虚拟机结构体系中最重要的组成部分，对程序的性能和稳定有着非常大的影响。在本书的第2～5章里，笔者（书籍作者）从理论知识、异常现象、代码、工具、案例、实战等几个方面对其进行讲解，希望读者能有所收获。

​	本书关于虚拟机内存管理部分到此就结束了，下一章我们将开始学习Class文件与虚拟机执行子系统方面的知识。







[^44]: