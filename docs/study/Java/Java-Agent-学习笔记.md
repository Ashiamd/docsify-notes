# Java-Agent-学习笔记01

> 视频：[新课程《Java Agent基础篇》-lsieun-lsieun-哔哩哔哩视频 (bilibili.com)](https://www.bilibili.com/list/1321054247)
> 视频对应的文章：[Java Agent系列一：基础篇 | lsieun](https://lsieun.github.io/java-agent/java-agent-01.html) <= 笔记内容大部分都摘自该文章，下面不再重复声明
>
> ASM官网：[ASM (ow2.io)](https://asm.ow2.io/)
> ASM中文手册：[ASM4 使用手册(中文版) (yuque.com)](https://www.yuque.com/mikaelzero/asm)
>
> Java Virtual Machine Tool Interface (JVM TI)-官方文档：
>
> + [JVM(TM) Tool Interface 17.0.0 (oracle.com)](https://docs.oracle.com/en/java/javase/17/docs/specs/jvmti.html#bci)
> + [java.lang.instrument (Java SE 17 & JDK 17) (oracle.com)](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html) <= 主要和java agent相关的介绍，建议阅读
> + [Java Virtual Machine Tool Interface (JVM TI) (oracle.com)](https://docs.oracle.com/javase/8/docs/technotes/guides/jvmti/index.html)
> + [Java(TM) java.lang.instrument (oracle.com)](https://docs.oracle.com/javase/8/docs/technotes/guides/instrumentation/index.html)
>
> Java语言规范和JVM规范-官方文档：[Java SE Specifications (oracle.com)](https://docs.oracle.com/javase/specs/index.html)

---

> 其他不错的网文/视频汇总
>
> [【java】JavaAgent介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1fN411h7eD/?spm_id_from=444.41.list.card_archive.click&vd_source=ba4d176271299cb334816d3c4cbc885f)

## 0. Java Agent通灵之术

> [Java Agent通灵之术-lsieun-lsieun-哔哩哔哩视频 (bilibili.com)](https://www.bilibili.com/list/1321054247?bvid=BV1R34y1b7U7&oid=809582144)
>
> [Java Agent：通灵之术 | lsieun](https://lsieun.github.io/article/java-agent-summoning-jutsu.html) <= 下方的笔记来源

### 0.1 概述

![img](https://lsieun.github.io/assets/images/java/agent/java-agent-dump-class.png)

“通灵之术”，在Java领域，代表什么意思呢？就是将正在运行的JVM当中的class进行导出。

本文的主要目的：**借助于Java Agent将class文件从JVM当中导出**。

场景应用：

- 第一个场景，两个不同版本的类。在开发环境（development），在类里面添加一个功能，测试之后，能够正常运行。到了生产环境（production），这个功能就是不正常。 **可能的一种情况，在线上的服务器上有两个版本的类文件，每次都会加载旧版本的类文件。这个时候，把JVM当中的类导出来看一看，到底是不是最新的版本**。（这个加载旧版本类文件的场景，我个人还真在企业项目中遇到过，主要是底层依赖的几个项目jar包内使用的某个类版本不同，导致后续有些逻辑执行失败）
- 第二个场景，破解软件。将某个类从JVM当中导出，然后修改，再提交给JVM进行redefine。

### 0.2 准备工作

开发环境：

- JDK版本：Java 8（我这边本地是jdk17，下面一样能正常运行）
- 开发工具：记事本或`vi`

创建文件目录结构：准备一个`prepare.sh`文件

```shell
#!/bin/bash

mkdir -p application/{src,out}/sample/
touch application/src/sample/{HelloWorld.java,Program.java}

mkdir -p java-agent/{src,out}/
touch java-agent/src/{ClassDumpAgent.java,ClassDumpTransformer.java,ClassDumpUtils.java,manifest.txt}

mkdir -p tools-attach/{src,out}/
touch tools-attach/src/Attach.java
```

目录结构：（编译之前）

```shell
java-agent-summoning-jutsu
├─── application
│    └─── src
│         └─── sample
│              ├─── HelloWorld.java
│              └─── Program.java
├─── java-agent
│    └─── src
│         ├─── ClassDumpAgent.java
│         ├─── ClassDumpTransformer.java
│         ├─── ClassDumpUtils.java
│         └─── manifest.txt
└─── tools-attach
     └─── src
          └─── Attach.java
```

目录结构：（编译之后）

```shell
java-agent-summoning-jutsu
├─── application
│    ├─── out
│    │    └─── sample
│    │         ├─── HelloWorld.class
│    │         └─── Program.class
│    └─── src
│         └─── sample
│              ├─── HelloWorld.java
│              └─── Program.java
├─── java-agent
│    ├─── out
│    │    ├─── ClassDumpAgent.class
│    │    ├─── classdumper.jar
│    │    ├─── ClassDumpTransformer.class
│    │    ├─── ClassDumpUtils.class
│    │    └─── manifest.txt
│    └─── src
│         ├─── ClassDumpAgent.java
│         ├─── ClassDumpTransformer.java
│         ├─── ClassDumpUtils.java
│         └─── manifest.txt
└─── tools-attach
     ├─── out
     │    └─── Attach.class
     └─── src
          └─── Attach.java
```

### 0.3 Application

#### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }
}
```

#### Program.java

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // (1) print process id (打印进程号)
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // (2) count down (整体逻辑即600秒内随机打印 a + b 或 a - b)
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### 编译和运行

进行编译：

```shell
# 进行编译
$ cd application/
$ javac src/sample/*.java -d out/
```

运行结果：

```shell
$ cd out/
$ java sample.Program
5556@LenovoWin7
|000| 5556@LenovoWin7 remains 600 seconds
a - b = 6
|001| 5556@LenovoWin7 remains 599 seconds
a - b = -4
...
```

### 0.4 Java Agent

曾经有一篇文章《Retrieving .class files from a running app》，最初是发表在Sun公司的网站，后来转移到了Oracle的网站，再后来就从Oracle网站消失了。

Sometimes it is better to dump `.class` files of generated/modified classes for off-line debugging - for example, we may want to view such classes using tools like [jclasslib](https://github.com/ingokegel/jclasslib).

#### ClassDumpAgent.java

> [java.lang.instrument (Java SE 17 & JDK 17) (oracle.com)](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html) <= java agent的使用可以参考官方文章

介绍：

+ java agent类，`premain`作为入口，在被使用方的main执行前执行（JVM启动时先调用premain方法）
+ 后续会将该agent类以及manifest.txt配置说明打包成jar包
+ 通过`-javaagent:<jarpath>[=<options>]`指令在启动指定类之前将agent所在jar包的路径作为参数传入（该文章制定类为`Program.java`），使得agent能够作用到指定类所在的JVM中。
+ `agentmain`在JVM启动后被执行（JVM先调用premain，再调用main，最后调用agentmain）
+ **这里既定义了`premain`，又定义了`agentmain`方法，相当于不管是JVM启动时通过`javaagent`指令执行agent逻辑，还是JVM启动后再临时执行指定的agent逻辑，都是可行的**

```java
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;
import java.util.ArrayList;
import java.util.List;

/**
 * This is a java.lang.instrument agent to dump .class files
 * from a running Java application.
 */
public class ClassDumpAgent {
  // JVM启动时，执行main前会先执行premain方法
  // agentArgs即-javaagent:<jarpath>[=<options>] 指令的<options>参数(String)
  // Instrumentation inst 由JVM自动传入
  public static void premain(String agentArgs, Instrumentation inst) {
    // 用户自定义的逻辑
    agentmain(agentArgs, inst);
  }

  // JVM启动后，执行agentmain方法
  public static void agentmain(String agentArgs, Instrumentation inst) {
    System.out.println("agentArgs: " + agentArgs);
    // 解析 参数(dump导出文件的路径、需要dump的class名)
    ClassDumpUtils.parseArgs(agentArgs);
    // 注册自定义的类转换器，自定义的逻辑即找到 指定的class类，并导出到指定dump路径
    inst.addTransformer(new ClassDumpTransformer(), true);
    // by the time we are attached, the classes to be
    // dumped may have been loaded already.
    // So, check for candidates in the loaded classes.
    Class[] classes = inst.getAllLoadedClasses();
    List<Class> candidates = new ArrayList<>();
    for (Class c : classes) {
      String className = c.getName();

      // 第一步，排除法：不考虑JDK自带的类
      if (className.startsWith("java")) continue;
      if (className.startsWith("javax")) continue;
      if (className.startsWith("jdk")) continue;
      if (className.startsWith("sun")) continue;
      if (className.startsWith("com.sun")) continue;

      // 第二步，筛选法：只留下感兴趣的类（正则表达式匹配）
      boolean isModifiable = inst.isModifiableClass(c);
      boolean isCandidate = ClassDumpUtils.isCandidate(className);
      if (isModifiable && isCandidate) {
        candidates.add(c);
      }

      // 不重要：打印调试信息
      String message = String.format("[DEBUG] Loaded Class: %s ---> Modifiable: %s, Candidate: %s", className, isModifiable, isCandidate);
      System.out.println(message);
    }
    try {
      // 第三步，将具体的class进行dump操作
      // if we have matching candidates, then retransform those classes
      // so that we will get callback to transform.
      if (!candidates.isEmpty()) {
        // 触发ClassDumpTransformer的transform逻辑，导出指定的一个类文件
        inst.retransformClasses(candidates.toArray(new Class[0]));

        // 不重要：打印调试信息
        String message = String.format("[DEBUG] candidates size: %d", candidates.size());
        System.out.println(message);
      }
    }
    catch (UnmodifiableClassException ignored) {
    }
  }
}
```

#### ClassDumpTransformer.java

```java
import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class ClassDumpTransformer implements ClassFileTransformer {

  public byte[] transform(ClassLoader loader,
                          String className,
                          Class redefinedClass,
                          ProtectionDomain protDomain,
                          byte[] classBytes) {
    // check and dump .class file
    if (ClassDumpUtils.isCandidate(className)) {
      ClassDumpUtils.dumpClass(className, classBytes);
    }

    // we don't mess with .class file, just return null
    return null;
  }

}
```

#### ClassDumpUtils.java

```java
import java.io.File;
import java.io.FileOutputStream;
import java.util.regex.Pattern;

public class ClassDumpUtils {
  // directory where we would write .class files
  private static String dumpDir;
  // classes with name matching this pattern will be dumped
  private static Pattern classes;

  // parse agent args of the form arg1=value1,arg2=value2
  public static void parseArgs(String agentArgs) {
    if (agentArgs != null) {
      String[] args = agentArgs.split(",");
      for (String arg : args) {
        String[] tmp = arg.split("=");
        if (tmp.length == 2) {
          String name = tmp[0];
          String value = tmp[1];
          if (name.equals("dumpDir")) {
            dumpDir = value;
          }
          else if (name.equals("classes")) {
            classes = Pattern.compile(value);
          }
        }
      }
    }
    if (dumpDir == null) {
      dumpDir = ".";
    }
    if (classes == null) {
      classes = Pattern.compile(".*");
    }
    System.out.println("[DEBUG] dumpDir: " + dumpDir);
    System.out.println("[DEBUG] classes: " + classes);
  }

  public static boolean isCandidate(String className) {
    // ignore array classes
    if (className.charAt(0) == '[') {
      return false;
    }
    // convert the class name to external name
    className = className.replace('/', '.');
    // check for name pattern match
    return classes.matcher(className).matches();
  }

  public static void dumpClass(String className, byte[] classBuf) {
    try {
      // create package directories if needed
      className = className.replace("/", File.separator);
      StringBuilder buf = new StringBuilder();
      buf.append(dumpDir);
      buf.append(File.separatorChar);
      int index = className.lastIndexOf(File.separatorChar);
      if (index != -1) {
        String pkgPath = className.substring(0, index);
        buf.append(pkgPath);
      }
      String dir = buf.toString();
      new File(dir).mkdirs();
      // write .class file
      String fileName = dumpDir + File.separator + className + ".class";
      FileOutputStream fos = new FileOutputStream(fileName);
      fos.write(classBuf);
      fos.close();
      System.out.println("[DEBUG] FileName: " + fileName);
    }
    catch (Exception ex) {
      ex.printStackTrace();
    }
  }

}
```

#### manifest.txt

```txt
Premain-Class: ClassDumpAgent
Agent-Class: ClassDumpAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true

```

注意：在结尾处添加一个空行。

#### 编译和打包

第一步，进行编译：

```shell
$ javac src/ClassDump*.java -d ./out
```

在Windows操作系统，如果遇到如下错误：

```pseudocode
错误: 编码GBK的不可映射字符
```

可以添加`-encoding`选项：

```shell
javac -encoding UTF-8 src/ClassDump*.java -d ./out
```

第二步，生成Jar文件：

```shell
$ cp src/manifest.txt out/
$ cd out/
$ jar -cvfm classdumper.jar manifest.txt ClassDump*.class
```

### 0.5 Tools Attach

将一个Agent Jar与一个正在运行的Application建立联系，需要用到Attach机制：

```pseudocode
Agent Jar ---> Tools Attach ---> Application(JVM)
```

与Attach机制相关的类，定义在`tools.jar`文件：

```shell
JDK_HOME/lib/tools.jar
```

#### Attach

```java
import com.sun.tools.attach.VirtualMachine;

/**
   * Simple attach-on-demand client tool
   * that loads the given agent into the given Java process.
   */
public class Attach {
  public static void main(String[] args) throws Exception {
    if (args.length < 2) {
      System.out.println("usage: java Attach <pid> <agent-jar-full-path> [<agent-args>]");
      System.exit(1);
    }
    // JVM is identified by process id (pid).
    VirtualMachine vm = VirtualMachine.attach(args[0]);
    String agentArgs = (args.length > 2) ? args[2] : null;
    // load a specified agent onto the JVM
    vm.loadAgent(args[1], agentArgs);
    vm.detach();
  }
}
```

#### 编译

```shell
# 编译（Linux)
$ javac -cp "${JAVA_HOME}/lib/tools.jar":. src/Attach.java -d out/

# 编译（MINGW64)
$ javac -cp "${JAVA_HOME}/lib/tools.jar"\;. src/Attach.java -d out/

# 编译（Windows)
$ javac -cp "%JAVA_HOME%/lib/tools.jar";. src/Attach.java -d out/
```

#### 运行

```shell
# 运行（Linux)
java -cp "${JAVA_HOME}/lib/tools.jar":. Attach <pid> <full-path-of-classdumper.jar> dumpDir=<dir>,classes=<name-pattern>

# 运行（MINGW64)
java -cp "${JAVA_HOME}/lib/tools.jar"\;. Attach <pid> <full-path-of-classdumper.jar> dumpDir=<dir>,classes=<name-pattern>

# 运行（Windows)
java -cp "%JAVA_HOME%/lib/tools.jar";. Attach <pid> <full-path-of-classdumper.jar> dumpDir=<dir>,classes=<name-pattern>
```

示例：

```shell
java -cp "${JAVA_HOME}/lib/tools.jar"\;. Attach <pid> \
D:/tmp/java-agent-summoning-jutsu/java-agent/out/classdumper.jar \
dumpDir=D:/tmp/java-agent-summoning-jutsu/dump,classes=sample\.HelloWorld
```

### 0.6 总结

本文内容总结如下：

- 第一点，主要功能。从功能的角度来讲，是如何从一个正在运行的JVM当中将某一个class文件导出的磁盘上。
- 第二点，实现方式。从实现方式上来说，是借助于Java Agent和正则表达式（区配类名）来实现功能。
- 第三点，注意事项。在Java 8的环境下，想要将Agent Jar加载到一个正在运行的JVM当中，需要用到`tools.jar`。

当然，将class文件从运行的JVM当中导出，只是Java Agent功能当中的一个小部分，想要更多的了解Java Agent的内容，可以学习《[Java Agent基础篇](https://ke.qq.com/course/4335150)》。

# 第一章 三个组成部分

## 1. Java Agent概览

### 1.1 Java Agent 是什么

Java agent is a powerful tool introduced with Java 5.

> 出现时间：Java 5（2004.09）；到了Java 6的时候（2006.12），有了一些改进；到了Java 9的时候（2017.09），又有一些改进。

在操作系统当中，Java Agent 的具体表现形式是一个 `.jar` 文件。 An agent is deployed as a JAR file.

> 存在形式：jar文件

在 Java Agent 当中，核心的作用是进行 bytecode instrumentation。 The true power of java agents lie on their ability to do the bytecode instrumentation.

> 主要功能：bytecode instrumentation

**The mechanism for instrumentation is modification of the byte-codes of methods.**

```pseudocode
Java Agent = bytecode instrumentation
```

#### Instrumentation分类

> [JEP 451: Prepare to Disallow the Dynamic Loading of Agents (openjdk.org)](https://openjdk.org/jeps/451) 未来的JDK版本，会默认禁用动态加载agent（对应Dynamic Instrumentation），如果需要使用，则需要在运行java程序时显式指定运行参数`-XX:+EnableDynamicAgentLoading`

Instrumentation can be inserted in one of three ways: [JVM(TM) Tool Interface 17.0.0 (oracle.com)](https://docs.oracle.com/en/java/javase/17/docs/specs/jvmti.html#bci)

- **Static Instrumentation**: The class file is instrumented before it is loaded into the VM - for example, by creating a duplicate directory of `*.class` files which have been modified to add the instrumentation. This method is extremely awkward and, in general, an agent cannot know the origin of the class files which will be loaded.
- **Load-Time Instrumentation**: When a class file is loaded by the VM, the raw bytes of the class file are sent for instrumentation to the agent. This mechanism provides efficient and complete access to one-time instrumentation.
- **Dynamic Instrumentation**: A class which is already loaded (and possibly even running) is modified. Classes can be modified multiple times and can be returned to their original state. The mechanism allows instrumentation which changes during the course of execution.

其实，这里就是讲了对 `.class` 文件进行修改（Instrumentation）的三个不同的时机（时间和机会）：没有被加载、正在被加载、已经被加载。

![img](https://lsieun.github.io/assets/images/java/agent/three-ways-of-instrumentation.png)

对于 Java Agent 这部分内容来说，我们只关注 **Load-Time Instrumentation** 和 **Dynamic Instrumentation** 两种情况。

#### 如何编写代码

上面的Instrumentation分类，只是一个抽象的划分，终究是要落实到具体的代码上：

```pseudocode
写代码 --> 编译成.class文件 --> 生成jar包
```

那么，编写Java Agent的代码，需要哪些知识呢？需要两方面的知识：

- 一方面，熟悉 `java.lang.instrument` 相关的API。这些API是我们编写 Java Agent 的主要依据，是我们关注的重点。
- 另一方面，需要掌握操作字节码的类库，并不是我们关注的重点。比较常用的字节码的类库有：[ASM](https://lsieun.github.io/java/asm/index.html)、ByteBuddy和Javassist。

知识体系：

```pseudocode
    Java Agent       一个机会：可以修改字节码的机会
------------------
    Java ASM         操作字节码的类库
------------------
    ClassFile        理论基础
```

#### 如何启动Java Agent

启动 Java Agent 有两种方式：命令行和 Attach 机制。

第一种方式，是从命令行（Command Line）启动 Java Agent，它对应着 Load-Time Instrumentation。

在使用 `java` 命令时，使用 `-javagent` 选项：

```shell
-javaagent:jarpath[=options]
```

具体示例：

```shell
java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
```

第二种方式，是通过虚拟机提供的 Attach 机制来启动 Java Agent，它对应着 Dynamic Instrumentation。

```java
import com.sun.tools.attach.VirtualMachine;

public class VMAttach {
  public static void main(String[] args) throws Exception {
    // 注意：需要修改pid的值
    String pid = "1234";
    String agentPath = "D:\\git-repo\\learn-java-agent\\target\\TheAgent.jar";
    VirtualMachine vm = VirtualMachine.attach(pid);
    vm.loadAgent(agentPath);
    vm.detach();
  }
}
```

### 1.2 如何学习Java Agent

在我们学习 Java Agent 的过程中，可以从四个层面来着手：

- 第一个层面，

  用户层面。也就是说，Java Agent 是一件事物（野马、怪物），那从用户（人）的角度出发，如何使用（驾驭、打败）它呢？有两种方式，

  - 第一种方式，在启动 JVM 之前，执行 `java` 命令时，使用 `-javaagent` 选项加载 Agent Jar，这就是 Load-Time Instrumentation
  - 第二种方式，在启动 JVM 之后，使用 Attach 机制加载 Agent Jar，这就是 Dynamic Instrumentation

- 第二个层面，**磁盘层面**。也就是说，从磁盘（操作系统）的角度来说，Java Agent 就是一个 `.jar` 文件，它包含哪些主要组成部分。

- 第三个层面，**Java层面**。也就是说，我们在使用 Java 语言开发 Java Agent，主要是利用 `java.lang.instrument` 包的API来实现一些功能。

- 第四个层面，**JVM层面**。也就是说，Java Agent 也是要运行在 JVM之 上的，它要与 JVM 进行“沟通”，理解其中一些细节之处，能够帮助我们更好的掌握 Java Agent。

![Java Agent的四个层次](https://lsieun.github.io/assets/images/java/agent/java-agent-mindmap.png)

### 1.3 总结

本文内容总结如下：

- 第一点，了解 Java Agent 是什么。Java Agent 的核心作用就是进行 bytecode Instrumentation。
- 第二点，如何学习 Java Agent。在学习 Java Agent 的过程当中，我们把相关的内容放到四个不同的层面来理解，这样便于形成一个整体的、有逻辑的知识体系。

## 2. Agent Jar的三个主要组成部分

### 2.1 三个主要组成部分

在 Java Agent 对应的 `.jar` 文件里，有三个主要组成部分：

- Manifest
- Agent Class
- ClassFileTransformer

![Agent Jar中的三个组成部分](https://lsieun.github.io/assets/images/java/agent/agent-jar-three-components.png)

```pseudocode
                ┌─── Manifest ───────────────┼─── META-INF/MANIFEST.MF
                │
                │                            ┌─── LoadTimeAgent.class: premain
TheAgent.jar ───┼─── Agent Class ────────────┤
                │                            └─── DynamicAgent.class: agentmain
                │
                └─── ClassFileTransformer ───┼─── ASMTransformer.class
```

### 2.2 Manifest Attributes

首先，在 Manifest 文件当中，可以定义的属性非常多，但是与 Java Agent 相关的属性有6、7个。

- 在[Java 8](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html) 版本当中，定义的属性有6个；
- 在[Java 9](https://docs.oracle.com/javase/9/docs/api/java/lang/instrument/package-summary.html) 至[Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html) 版本当中，定义的属性有7个。其中，`Launcher-Agent-Class`属性，是Java 9引入的。

其次，我们将 Manifest 定义的属性分成了三组：基础、能力和特殊情况。

```pseudocode
                                       ┌─── Premain-Class
                       ┌─── Basic ─────┤
                       │               └─── Agent-Class
                       │
                       │               ┌─── Can-Redefine-Classes
                       │               │
Manifest Attributes ───┼─── Ability ───┼─── Can-Retransform-Classes
                       │               │
                       │               └─── Can-Set-Native-Method-Prefix
                       │
                       │               ┌─── Boot-Class-Path
                       └─── Special ───┤
                                       └─── Launcher-Agent-Class
```

分组的目的，是为了便于理解：一下子记住7个属性，不太容易；分成三组，每次记忆两、三个属性，就相对容易一些。

这些分组，也对应着先、后的学习顺序，它也是由简单到复杂的过程。

**注意**：这个分组是我个人的理解，并不一定是对的。如果你有更好的认知方式，可以按自己的思路来。

#### 基础

An attribute in the JAR file manifest specifies the **agent class** which will be loaded to start the agent.

- `Premain-Class`: When an agent is specified at JVM launch time this attribute specifies the agent class. That is, the class containing the `premain` method. When an agent is specified at JVM launch time this attribute is required. If the attribute is not present the JVM will abort. Note: this is a class name, not a file name or path.
- `Agent-Class`: If an implementation supports a mechanism to start agents sometime after the VM has started then this attribute specifies the agent class. That is, the class containing the `agentmain` method. This attribute is required, if it is not present the agent will not be started. Note: this is a class name, not a file name or path.

```pseudocode
Premain-Class: lsieun.agent.LoadTimeAgent
Agent-Class: lsieun.agent.DynamicAgent

```

An agent JAR file may have both the `Premain-Class` and `Agent-Class` attributes present in the manifest.

- When the agent is started on the command-line using the `-javaagent` option then the `Premain-Class` attribute specifies the name of the agent class and the `Agent-Class` attribute is ignored.
- Similarly, if the agent is started sometime after the VM has started, then the `Agent-Class` attribute specifies the name of the agent class (the value of `Premain-Class` attribute is ignored).

#### 能力

能力，体现在两个层面上：JVM 和 Java Agent。

```pseudocode
                              ┌─── redefine
                              │
           ┌─── Java Agent ───┼─── retransform
           │                  │
           │                  └─── native method prefix
Ability ───┤
           │                  ┌─── redefine
           │                  │
           └─── JVM ──────────┼─── retransform
                              │
                              └─── native method prefix
```

下面三个属性，就是确定Java Agent的能力：

- `Can-Redefine-Classes`: Boolean (`true` or `false`, case irrelevant). Is the ability to redefine classes needed by this agent. Values other than `true` are considered `false`. This attribute is optional, the default is `false`.
- `Can-Retransform-Classes`: Boolean (`true` or `false`, case irrelevant). Is the ability to retransform classes needed by this agent. Values other than `true` are considered `false`. This attribute is optional, the default is `false`.
- `Can-Set-Native-Method-Prefix`: Boolean (`true` or `false`, case irrelevant). Is the ability to set native method prefix needed by this agent. Values other `than` true are considered `false`. This attribute is optional, the default is `false`.

#### 特殊情况

- `Boot-Class-Path`: A list of paths to be searched by the bootstrap class loader. Paths represent directories or libraries (commonly referred to as JAR or zip libraries on many platforms). These paths are searched by the bootstrap class loader after the platform specific mechanisms of locating a class have failed. Paths are searched in the order listed. Paths in the list are separated by one or more spaces. A path takes the syntax of the path component of a hierarchical URI. The path is absolute if it begins with a slash character (`/`), otherwise it is relative. A relative path is resolved against the absolute path of the agent JAR file. Malformed and non-existent paths are ignored. When an agent is started sometime after the VM has started then paths that do not represent a JAR file are ignored. This attribute is optional.
- `Launcher-Agent-Class`: If an implementation supports a mechanism to start an application as an executable JAR then the main manifest may include this attribute to specify the class name of an agent to start before the application `main` method is invoked.

### 2.3 Agent Class

#### LoadTimeAgent

如果我们想使用 Load-Time Instrumentation，那么就必须有一个 `premain` 方法，它有两种写法。

The JVM first attempts to invoke the following method on the agent class:（推荐使用）

```java
public static void premain(String agentArgs, Instrumentation inst);
```

If the agent class does not implement this method then the JVM will attempt to invoke:

```java
public static void premain(String agentArgs);
```

#### DynamicAgent

如果我们想使用 Dynamic Instrumentation，那么就必须有一个 `agentmain` 方法，它有两种写法。

The JVM first attempts to invoke the following method on the agent class:（推荐使用）

```java
public static void agentmain(String agentArgs, Instrumentation inst);
```

If the agent class does not implement this method then the JVM will attempt to invoke:

```java
public static void agentmain(String agentArgs);
```

### 2.4 ClassFileTransformer

在 `java.lang.instrument` 下包含了 `Instrumentation` 和 `ClassFileTransformer` 接口：

- `java.lang.instrument.Instrumentation`
- `java.lang.instrument.ClassFileTransformer`

在 `Instrumentation` 接口中，定义了添加和移除 `ClassFileTransformer` 的方法：

```java
/**
 * This class provides services needed to instrument Java
 * programming language code.
 * Instrumentation is the addition of byte-codes to methods for the
 * purpose of gathering data to be utilized by tools.
 * Since the changes are purely additive, these tools do not modify
 * application state or behavior.
 * Examples of such benign tools include monitoring agents, profilers,
 * coverage analyzers, and event loggers.
 *
 * <P>
 * There are two ways to obtain an instance of the
 * <code>Instrumentation</code> interface:
 *
 * <ol>
 *   <li><p> When a JVM is launched in a way that indicates an agent
 *     class. In that case an <code>Instrumentation</code> instance
 *     is passed to the <code>premain</code> method of the agent class.
 *     </p></li>
 *   <li><p> When a JVM provides a mechanism to start agents sometime
 *     after the JVM is launched. In that case an <code>Instrumentation</code>
 *     instance is passed to the <code>agentmain</code> method of the
 *     agent code. </p> </li>
 * </ol>
 * <p>
 * These mechanisms are described in the
 * {@linkplain java.lang.instrument package specification}.
 * <p>
 * Once an agent acquires an <code>Instrumentation</code> instance,
 * the agent may call methods on the instance at any time.
 *
 * @apiNote This interface is not intended to be implemented outside of
 * the java.instrument module.
 *
 * @since   1.5
 */
public interface Instrumentation {
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

  boolean removeTransformer(ClassFileTransformer transformer);
}
```

在 `ClassFileTransformer` 接口中，定义了 `transform` 抽象方法：

```java
/**
 * A transformer of class files. An agent registers an implementation of this
 * interface using the {@link Instrumentation#addTransformer addTransformer}
 * method so that the transformer's {@link
 * ClassFileTransformer#transform(Module,ClassLoader,String,Class,ProtectionDomain,byte[])
 * transform} method is invoked when classes are loaded,
 * {@link Instrumentation#redefineClasses redefined}, or
 * {@link Instrumentation#retransformClasses retransformed}. The implementation
 * should override one of the {@code transform} methods defined here.
 * Transformers are invoked before the class is defined by the Java virtual
 * machine.
 *
 * <P>
 * There are two kinds of transformers, determined by the <code>canRetransform</code>
 * parameter of
 * {@link java.lang.instrument.Instrumentation#addTransformer(ClassFileTransformer,boolean)}:
 *  <ul>
 *    <li><i>retransformation capable</i> transformers that were added with
 *        <code>canRetransform</code> as true
 *    </li>
 *    <li><i>retransformation incapable</i> transformers that were added with
 *        <code>canRetransform</code> as false or where added with
 *        {@link java.lang.instrument.Instrumentation#addTransformer(ClassFileTransformer)}
 *    </li>
 *  </ul>
 *
 * <P>
 * Once a transformer has been registered with
 * {@link java.lang.instrument.Instrumentation#addTransformer(ClassFileTransformer,boolean)
 * addTransformer},
 * the transformer will be called for every new class definition and every class redefinition.
 * Retransformation capable transformers will also be called on every class retransformation.
 * The request for a new class definition is made with
 * {@link java.lang.ClassLoader#defineClass ClassLoader.defineClass}
 * or its native equivalents.
 * The request for a class redefinition is made with
 * {@link java.lang.instrument.Instrumentation#redefineClasses Instrumentation.redefineClasses}
 * or its native equivalents.
 * The request for a class retransformation is made with
 * {@link java.lang.instrument.Instrumentation#retransformClasses Instrumentation.retransformClasses}
 * or its native equivalents.
 * The transformer is called during the processing of the request, before the class file bytes
 * have been verified or applied.
 * When there are multiple transformers, transformations are composed by chaining the
 * <code>transform</code> calls.
 * That is, the byte array returned by one call to <code>transform</code> becomes the input
 * (via the <code>classfileBuffer</code> parameter) to the next call.
 *
 * <P>
 * Transformations are applied in the following order:
 *  <ul>
 *    <li>Retransformation incapable transformers
 *    </li>
 *    <li>Retransformation incapable native transformers
 *    </li>
 *    <li>Retransformation capable transformers
 *    </li>
 *    <li>Retransformation capable native transformers
 *    </li>
 *  </ul>
 *
 * <P>
 * For retransformations, the retransformation incapable transformers are not
 * called, instead the result of the previous transformation is reused.
 * In all other cases, this method is called.
 * Within each of these groupings, transformers are called in the order registered.
 * Native transformers are provided by the <code>ClassFileLoadHook</code> event
 * in the Java Virtual Machine Tool Interface).
 *
 * <P>
 * The input (via the <code>classfileBuffer</code> parameter) to the first
 * transformer is:
 *  <ul>
 *    <li>for new class definition,
 *        the bytes passed to <code>ClassLoader.defineClass</code>
 *    </li>
 *    <li>for class redefinition,
 *        <code>definitions.getDefinitionClassFile()</code> where
 *        <code>definitions</code> is the parameter to
 *        {@link java.lang.instrument.Instrumentation#redefineClasses
 *         Instrumentation.redefineClasses}
 *    </li>
 *    <li>for class retransformation,
 *         the bytes passed to the new class definition or, if redefined,
 *         the last redefinition, with all transformations made by retransformation
 *         incapable transformers reapplied automatically and unaltered;
 *         for details see
 *         {@link java.lang.instrument.Instrumentation#retransformClasses
 *          Instrumentation.retransformClasses}
 *    </li>
 *  </ul>
 *
 * <P>
 * If the implementing method determines that no transformations are needed,
 * it should return <code>null</code>.
 * Otherwise, it should create a new <code>byte[]</code> array,
 * copy the input <code>classfileBuffer</code> into it,
 * along with all desired transformations, and return the new array.
 * The input <code>classfileBuffer</code> must not be modified.
 *
 * <P>
 * In the retransform and redefine cases,
 * the transformer must support the redefinition semantics:
 * if a class that the transformer changed during initial definition is later
 * retransformed or redefined, the
 * transformer must insure that the second class output class file is a legal
 * redefinition of the first output class file.
 *
 * <P>
 * If the transformer throws an exception (which it doesn't catch),
 * subsequent transformers will still be called and the load, redefine
 * or retransform will still be attempted.
 * Thus, throwing an exception has the same effect as returning <code>null</code>.
 * To prevent unexpected behavior when unchecked exceptions are generated
 * in transformer code, a transformer can catch <code>Throwable</code>.
 * If the transformer believes the <code>classFileBuffer</code> does not
 * represent a validly formatted class file, it should throw
 * an <code>IllegalClassFormatException</code>;
 * while this has the same effect as returning null. it facilitates the
 * logging or debugging of format corruptions.
 *
 * <P>
 * Note the term <i>class file</i> is used as defined in section 3.1 of
 * <cite>The Java Virtual Machine Specification</cite>, to mean a
 * sequence of bytes in class file format, whether or not they reside in a
 * file.
 *
 * @see     java.lang.instrument.Instrumentation
 * @since   1.5
 */
public interface ClassFileTransformer {
  byte[] transform(ClassLoader         loader,
                   String              className,
                   Class<?>            classBeingRedefined,
                   ProtectionDomain    protectionDomain,
                   byte[]              classfileBuffer) throws IllegalClassFormatException;

}
```

当我们想对 Class 进行 bytecode instrumentation 时，就要实现 `ClassFileTransformer` 接口，并重写它的 `transform` 方法。

### 2.5 总结

本文内容总结如下：

- 第一点，了解 Agent Jar 的三个主要组成部分：Manifest、Agent Class 和 ClassFileTransformer。
- 第二点，在 Agent Jar 当中，这些不同的组成部分之间是如何联系在一起的。

```pseudocode
Manifest --> Agent Class --> Instrumentation --> ClassFileTransformer
```

## 3. 手工打包（一）：Load-Time Agent打印加载的类

### 3.1 预期目标

我们的预期目标：打印正在加载的类。

开发环境：

- JDK 版本：Java 8
- 编辑器：记事本（Windows）或 `vi`（Linux）

我们尽量使用简单的工具，来理解 Agent Jar 的生成过程。

代码目录结构：[Code](https://lsieun.github.io/assets/zip/java-agent-manual-01.zip)

```pseudocode
java-agent-manual-01
└─── src
     ├─── lsieun
     │    ├─── agent
     │    │    └─── LoadTimeAgent.java
     │    └─── instrument
     │         └─── InfoTransformer.java
     ├─── manifest.txt
     └─── sample
          ├─── HelloWorld.java
          └─── Program.java
```

做一些准备工作（`prepare01.sh`）：

```shell
DIR=java-agent-manual-01
mkdir ${DIR} && cd ${DIR}

mkdir -p src/sample
touch src/sample/{HelloWorld.java,Program.java}

mkdir -p src/lsieun/{agent,instrument}
touch src/lsieun/agent/LoadTimeAgent.java
touch src/lsieun/instrument/InfoTransformer.java
touch src/manifest.txt
```

### 3.2 Application

#### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }
}
```

#### Program.java

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // (1) print process id
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // (2) count down
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### 编译和运行

进行编译：

```shell
# 进行编译
$ mkdir out
$ javac src/sample/*.java -d out/

# 查看编译结果
$ find ./out/ -type f
./out/sample/HelloWorld.class
./out/sample/Program.class
```

运行结果：

```shell
$ cd out/
$ java sample.Program
5556@LenovoWin7
|000| 5556@LenovoWin7 remains 600 seconds
a - b = 6
|001| 5556@LenovoWin7 remains 599 seconds
a - b = -4
...
```

### 3.3 Agent Jar

#### manifest.txt

修改 `manifest.txt` 文件内容：

```
Premain-Class: lsieun.agent.LoadTimeAgent

```

注意：在 `manifest.txt` 文件的结尾处有**一个空行**。(make sure the last line in the file is **a blank line**)

#### LoadTimeAgent.java

```java
package lsieun.agent;

import lsieun.instrument.InfoTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    ClassFileTransformer transformer = new InfoTransformer();
    inst.addTransformer(transformer);
  }
}
```

#### InfoTransformer.java

```java
package lsieun.instrument;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Formatter;

public class InfoTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    StringBuilder sb = new StringBuilder();
    Formatter fm = new Formatter(sb);
    fm.format("ClassName: %s%n", className);
    fm.format("    ClassLoader: %s%n", loader);
    fm.format("    ClassBeingRedefined: %s%n", classBeingRedefined);
    fm.format("    ProtectionDomain: %s%n", protectionDomain);
    System.out.println(sb.toString());

    return null;
  }
}
```

#### 生成 Jar 包

编译：

```shell
# 切换目录
$ cd java-agent-manual-01/

# 找到所有 .java 文件
$ find ./src/lsieun/ -name "*.java" > sources.txt
$ cat sources.txt
./src/lsieun/agent/LoadTimeAgent.java
./src/lsieun/transformer/InfoTransformer.java

# 进行编译
$ javac -d out/ @sources.txt
```

生成 Jar 包：

```shell
# 复制 manifest.txt 文件
$ cp src/manifest.txt out/

# 切换目录
$ cd out/
$ ls
lsieun/  manifest.txt  sample/

# 进行打包（第一种方式）
            ┌─── f: TheAgent.jar
         ┌──┴──┐
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
          └─────────┬────────┘
                    └─── m: manifest.txt
# 进行打包（第二种方式）
                   ┌─── f: TheAgent.jar
          ┌────────┴────────┐
$ jar -cvmf manifest.txt TheAgent.jar lsieun/
         └───┬──┘
             └─── m: manifest.txt
```

输出信息：

```shell
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
已添加清单
正在添加: lsieun/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/agent/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/agent/LoadTimeAgent.class(输入 = 503) (输出 = 309)(压缩了 38%)
正在添加: lsieun/instrument/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/instrument/InfoTransformer.class(输入 = 1214) (输出 = 639)(压缩了 47%)
```

### 3.4 运行

在使用 `java` 命令时，我们可以通过使用 `-javaagent` 选项来使用 Agent Jar：

```shell
$ java -javaagent:TheAgent.jar sample.Program
```

部分输出结果：

```
...
ClassName: sample/Program
    ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
    ClassBeingRedefined: null
    ProtectionDomain: ProtectionDomain  (file:/D:/tmp/myAgent/java-agent-manual-01/out/ <no signer certificates>)
 sun.misc.Launcher$AppClassLoader@18b4aac2
 <no principals>
 java.security.Permissions@4aa298b7 (
 ("java.io.FilePermission" "\D:\tmp\myAgent\java-agent-manual-01\out\-" "read")
 ("java.lang.RuntimePermission" "exitVM")
)
...
```

另外，在使用 `java` 命令时，可以添加 `-verbose:class` 选项，它可以显示每个已加载类的信息。

```
$ java -verbose:class sample.Program
```

### 3.5 总结

本文内容总结如下：

- 第一点，本文的主要目的是对 Java Agent 有一个整体的印象，因此不需要理解技术细节。
- 第二点，Agent Jar 当中有三个重要组成部分：manifest、Agent Class 和 ClassFileTransformer。
- 第三点，使用 `java` 命令加载 Agent Jar 时，需要使用 `-javaagent` 选项。

## 4. 手工打包（二）：Load-Time Agent打印方法接收的参数

### 4.1 预期目标

我们的预期目标：借助于 JDK 内置的 ASM 打印出方法接收的参数，使用 Load-Time Instrumentation 的方式实现。

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-load-time-instrumentation.png)

开发环境：

- JDK版本：Java 8
- 编辑器：记事本（Windows）或 `vi` （Linux）

代码目录结构：[Code](https://lsieun.github.io/assets/zip/java-agent-manual-02.zip)

```pseudocode
java-agent-manual-02
└─── src
     ├─── lsieun
     │    ├─── agent
     │    │    └─── LoadTimeAgent.java
     │    ├─── asm
     │    │    ├─── adapter
     │    │    │    └─── MethodInfoAdapter.java
     │    │    ├─── cst
     │    │    │    └─── Const.java
     │    │    └─── visitor
     │    │         └─── MethodInfoVisitor.java
     │    ├─── instrument
     │    │    └─── ASMTransformer.java
     │    └─── utils
     │         └─── ParameterUtils.java
     ├─── manifest.txt
     └─── sample
          ├─── HelloWorld.java
          └─── Program.java
```

代码逻辑梳理：

```pseudocode
Manifest --> Agent Class --> Instrumentation --> ClassFileTransformer --> ASM
```

做一些准备工作（`prepare02.sh`）：

```shell
DIR=java-agent-manual-02
mkdir ${DIR} && cd ${DIR}

mkdir -p src/sample
touch src/sample/{HelloWorld.java,Program.java}

mkdir -p src/lsieun/{agent,asm,instrument,utils}
mkdir -p src/lsieun/asm/{adapter,cst,visitor}
touch src/lsieun/agent/LoadTimeAgent.java
touch src/lsieun/instrument/ASMTransformer.java
touch src/lsieun/asm/adapter/MethodInfoAdapter.java
touch src/lsieun/asm/cst/Const.java
touch src/lsieun/asm/visitor/MethodInfoVisitor.java
touch src/lsieun/utils/ParameterUtils.java
touch src/manifest.txt
```

### 4.2 Application

#### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }
}
```

#### Program.java

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // (1) print process id
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // (2) count down
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

### 4.3 ASM 相关

在这个部分，我们要借助于 JDK 内置的 ASM 类库（`jdk.internal.org.objectweb.asm`），来实现打印方法参数的功能。

#### ParameterUtils.java

在 `ParameterUtils.java` 文件当中，主要是定义了各种类型的 `print` 方法：

```java
package lsieun.utils;

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

public class ParameterUtils {
  private static final DateFormat fm = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

  public static void printValueOnStack(boolean value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(byte value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(char value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(short value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(int value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(float value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(long value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(double value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(Object value) {
    if (value == null) {
      System.out.println("    " + value);
    }
    else if (value instanceof String) {
      System.out.println("    " + value);
    }
    else if (value instanceof Date) {
      System.out.println("    " + fm.format(value));
    }
    else if (value instanceof char[]) {
      System.out.println("    " + Arrays.toString((char[]) value));
    }
    else if (value instanceof Object[]) {
      System.out.println("    " + Arrays.toString((Object[]) value));
    }
    else {
      System.out.println("    " + value.getClass() + ": " + value.toString());
    }
  }

  public static void printText(String str) {
    System.out.println(str);
  }

  public static void printStackTrace() {
    Exception ex = new Exception();
    ex.printStackTrace(System.out);
  }
}
```

#### Const.java

在 `Const.java` 文件当中，主要是定义了 `ASM_VERSION` 常量，它标识了使用的 ASM 的版本：

```java
package lsieun.asm.cst;

import jdk.internal.org.objectweb.asm.Opcodes;

public class Const {
  public static final int ASM_VERSION = Opcodes.ASM5;
}
```

#### MethodInfoAdapter.java

```java
package lsieun.asm.adapter;

import jdk.internal.org.objectweb.asm.MethodVisitor;
import jdk.internal.org.objectweb.asm.Opcodes;
import jdk.internal.org.objectweb.asm.Type;
import lsieun.asm.cst.Const;

public class MethodInfoAdapter extends MethodVisitor {
  private final String owner;
  private final int methodAccess;
  private final String methodName;
  private final String methodDesc;

  public MethodInfoAdapter(MethodVisitor methodVisitor, String owner,
                           int methodAccess, String methodName, String methodDesc) {
    super(Const.ASM_VERSION, methodVisitor);
    this.owner = owner;
    this.methodAccess = methodAccess;
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public void visitCode() {
    if (mv != null) {
      String line = String.format("Method Enter: %s.%s:%s", owner, methodName, methodDesc);
      printMessage(line);

      int slotIndex = (methodAccess & Opcodes.ACC_STATIC) != 0 ? 0 : 1;
      Type methodType = Type.getMethodType(methodDesc);
      Type[] argumentTypes = methodType.getArgumentTypes();
      for (Type t : argumentTypes) {
        int sort = t.getSort();
        int size = t.getSize();
        int opcode = t.getOpcode(Opcodes.ILOAD);
        super.visitVarInsn(opcode, slotIndex);

        if (sort >= Type.BOOLEAN && sort <= Type.DOUBLE) {
          String desc = t.getDescriptor();
          printValueOnStack("(" + desc + ")V");
        }
        else {
          printValueOnStack("(Ljava/lang/Object;)V");
        }
        slotIndex += size;
      }
    }

    super.visitCode();
  }

  private void printMessage(String str) {
    super.visitLdcInsn(str);
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);
  }

  private void printValueOnStack(String descriptor) {
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printValueOnStack", descriptor, false);
  }

  private void printStackTrace() {
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printStackTrace", "()V", false);
  }
}
```

#### MethodInfoVisitor.java

```java
package lsieun.asm.visitor;

import jdk.internal.org.objectweb.asm.ClassVisitor;
import jdk.internal.org.objectweb.asm.MethodVisitor;
import lsieun.asm.adapter.MethodInfoAdapter;
import lsieun.asm.cst.Const;

public class MethodInfoVisitor extends ClassVisitor {
  private String owner;

  public MethodInfoVisitor(ClassVisitor classVisitor) {
    super(Const.ASM_VERSION, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !name.equals("<init>") && !name.equals("<clinit>")) {
      mv = new MethodInfoAdapter(mv, owner, access, name, descriptor);
    }
    return mv;
  }
}
```

### 4.4 Agent Jar

#### manifest.txt

在 `manifest.txt` 文件中，记录 Agent Class 的信息：

```pseudocode
Premain-Class: lsieun.agent.LoadTimeAgent

```

注意：在 `manifest.txt` 文件的结尾处有**一个空行**。(make sure the last line in the file is **a blank line**)

那么，如果不添加一个空行，会有什么结果呢？虽然可以成功生成 `.jar` 文件，但是不会将 `manifest.txt` 里的信息（`Premain-Class: lsieun.agent.LoadTimeAgent`）转换到 `META-INF/MANIFEST.MF` 里。

```shell
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
...
# 在没有添加空行的情况下，会出现如下错误
$ java -javaagent:TheAgent.jar sample.Program
Failed to find Premain-Class manifest attribute in TheAgent.jar
Error occurred during initialization of VM
agent library failed to init: instrument
```

#### LoadTimeAgent.java

```java
package lsieun.agent;

import lsieun.instrument.ASMTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    ClassFileTransformer transformer = new ASMTransformer();
    inst.addTransformer(transformer);
  }
}
```

#### ASMTransformer.java

```java
package lsieun.instrument;

import jdk.internal.org.objectweb.asm.ClassReader;
import jdk.internal.org.objectweb.asm.ClassVisitor;
import jdk.internal.org.objectweb.asm.ClassWriter;
import lsieun.asm.visitor.MethodInfoVisitor;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class ASMTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className == null) return null;
    if (className.startsWith("java")) return null;
    if (className.startsWith("javax")) return null;
    if (className.startsWith("jdk")) return null;
    if (className.startsWith("sun")) return null;
    if (className.startsWith("org")) return null;
    if (className.startsWith("com")) return null;
    if (className.startsWith("lsieun")) return null;

    System.out.println("candidate className: " + className);

    if (className.equals("sample/HelloWorld")) {
      ClassReader cr = new ClassReader(classfileBuffer);
      ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
      ClassVisitor cv = new MethodInfoVisitor(cw);

      int parsingOptions = 0;
      cr.accept(cv, parsingOptions);

      return cw.toByteArray();
    }

    return null;
  }
}
```

#### 生成 Jar 包

> [手把手教你从Java8升级到Java11_androidstarjack的博客-CSDN博客](https://blog.csdn.net/androidstarjack/article/details/122762398)

编译：

```shell
# 切换目录
$ cd java-agent-manual-02/

# 添加输出目录
$ mkdir out

# 找到所有 .java 文件
$ find ./src/ -name "*.java" > sources.txt
$ cat sources.txt
./src/lsieun/agent/LoadTimeAgent.java
./src/lsieun/asm/adapter/MethodInfoAdapter.java
./src/lsieun/asm/cst/Const.java
./src/lsieun/asm/visitor/MethodInfoVisitor.java
./src/lsieun/instrument/ASMTransformer.java
./src/lsieun/utils/ParameterUtils.java
./src/sample/HelloWorld.java
./src/sample/Program.java
```

以下列出错误编译和正确编译两种示例：

```shell
# 错误的编译
$ javac -d out/ @sources.txt

# 正确的编译
$ javac -XDignore.symbol.file -d out/ @sources.txt

# jdk17需要如下编译 [ 否则报错error: package jdk.internal.org.objectweb.asm is not visible \n import jdk.internal.org.objectweb.asm.Opcodes; \n  (package jdk.internal.org.objectweb.asm is declared in module java.base, which does not export it to the unnamed module) ]
$ javac --add-exports java.base/jdk.internal.org.objectweb.asm=ALL-UNNAMED -XDignore.symbol.file=true -d out/ @sources.txt
```

注意：在编译的时候，要添加 `-XDignore.symbol.file` 选项；否则，会编译出错。

那么，如果不使用这个选项，为什么会出错呢？是因为在上面的代码当中用到了 `jdk.internal.org.objectweb.asm` 里的类，如果不使用这个选项，就会提示找不到相应的类。

[StackOverflow](https://stackoverflow.com/questions/4065401/using-internal-sun-classes-with-javac): When `javac` is compiling code it doesn’t link against `rt.jar` by default. Instead it uses special symbol file `lib/ct.sym` with class stubs. Surprisingly this file contains many but not all of internal `sun` classes. And the answer is: `javac -XDignore.symbol.file`. That’s what `javac` uses for compiling `rt.jar`.

[Oracle: Why Developers Should Not Write Programs That Call 'sun' Packages](https://www.oracle.com/java/technologies/faq-sun-packages.html)

- **The `java.\*`, `javax.\*` and `org.\*` packages documented in the Java Platform Standard Edition API Specification make up the official, supported, public interface.** If a Java program directly calls only API in these packages, it will operate on all Java-compatible platforms, regardless of the underlying OS platform.
- **The `sun.\*` packages are not part of the supported, public interface.** A Java program that directly calls into `sun.*` packages is not guaranteed to work on all Java-compatible platforms. In fact, such a program is not guaranteed to work even in future versions on the same platform.
- In general, writing java programs that rely on `sun.*` is risky: those classes are not portable, and are not supported.

编译完成之后，我们需要将分散的内容整合成一个 Jar 包文件：

```shell
# 复制 manifest.txt 文件
$ cp src/manifest.txt out/

# 切换目录
$ cd out/
$ ls
lsieun/  manifest.txt  sample/

# 进行打包（第一种方式）
            ┌─── f: TheAgent.jar
         ┌──┴──┐
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
          └─────────┬────────┘
                    └─── m: manifest.txt
# 进行打包（第二种方式）
                   ┌─── f: TheAgent.jar
          ┌────────┴────────┐
$ jar -cvmf manifest.txt TheAgent.jar lsieun/
         └───┬──┘
             └─── m: manifest.txt
```

打包过程中的输出信息：

```shell
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
已添加清单
正在添加: lsieun/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/agent/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/agent/LoadTimeAgent.class(输入 = 502) (输出 = 310)(压缩了 38%)
正在添加: lsieun/asm/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/asm/adapter/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/asm/adapter/MethodInfoAdapter.class(输入 = 2363) (输出 = 1229)(压缩了 47%)
正在添加: lsieun/asm/cst/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/asm/cst/Const.class(输入 = 298) (输出 = 242)(压缩了 18%)
正在添加: lsieun/asm/visitor/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/asm/visitor/MethodInfoVisitor.class(输入 = 1177) (输出 = 552)(压缩了 53%)
正在添加: lsieun/instrument/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/instrument/ASMTransformer.class(输入 = 1728) (输出 = 921)(压缩了 46%)
正在添加: lsieun/utils/(输入 = 0) (输出 = 0)(存储了 0%)
正在添加: lsieun/utils/ParameterUtils.class(输入 = 2510) (输出 = 1046)(压缩了 58%)
```

### 4.5 运行

> [JDK17环境项目报错 Package ‘java.lang‘ is declared in module ‘java.base‘, which is not in the module graph - 简书 (jianshu.com)](https://www.jianshu.com/p/f73da9a1d045)

在使用 `java` 命令时，我们可以通过使用 `-javaagent` 选项来使用 Java Agent Jar：

```shell
$ java -javaagent:TheAgent.jar sample.Program

# jdk17 使用以下指令，否则报错 (Caused by: java.lang.IllegalAccessError: superclass access check failed: class lsieun.asm.visitor.MethodInfoVisitor (in unnamed module @0x7440e464) cannot access class jdk.internal.org.objectweb.asm.ClassVisitor (in module java.base) because module java.base does not export jdk.internal.org.objectweb.asm to unnamed module @0x7440e464)
java --add-opens java.base/jdk.internal.org.objectweb.asm=ALL-UNNAMED -javaagent:TheAgent.jar sample.Program
```

输出结果：

```shell
$ java -javaagent:TheAgent.jar sample.Program
candidate className: sample/Program
5096@LenovoWin7
|000| 5096@LenovoWin7 remains 600 seconds
candidate className: sample/HelloWorld
Method Enter: sample/HelloWorld.add:(II)I
    4
    3
a + b = 7
...
```

那么，`TheAgent.jar` 到底做了一件什么事情呢？

在一般情况下，我们先编写 `HelloWorld.java` 文件，然后编译生成 `HelloWorld.class` 文件，最后加载到JVM当中运行。

当 Instrumentation 发生的时候，它是将原有的 `HelloWorld.class` 的内容进行修改（bytecode transformation），生成一个新的 `HelloWorld.class`，最后将这个新的 `HelloWorld.class` 加载到JVM当中运行。

```pseudocode
┌────────────────────┐   compile   ┌────────────────────┐ load original bytecode     ┌────────────────────┐
│  HelloWorld.java   ├─────────────┤  HelloWorld.class  ├────────────────────────────┤        JVM         │
└────────────────────┘             └─────────┬──────────┘                            │                    │
                                             │                                       │                    │
                                             │bytecode transformation                │                    │
                                             │                                       │                    │
                                   ┌─────────┴──────────┐ load transformed bytecode  │                    │
                                   │  HelloWorld.class  ├────────────────────────────┤                    │
                                   └────────────────────┘                            └────────────────────┘
                                   Instrumentation/Java Agent
```

### 4.6 总结

本文内容总结如下：

- 第一点，本文的主要目的是希望大家对 Java Agent 有一个整体的印象，因此不需要理解技术细节（特别是 [ASM](https://lsieun.github.io/java/asm/index.html)相关内容）。
- 第二点，**Agent Jar 当中有三个重要组成部分：manifest、Agent Class 和 ClassFileTransformer**。
- 第三点，当使用 `javac` 命令编译时，如果在程序当中使用到了 `jdk.*` 或 `sun.*` 当中的类，要添加 `-XDignore.symbol.file` 选项。
- 第四点，**当使用 `java` 命令加载 Agent Jar 时（Load-Time Instrumentation），需要添加 `-javaagent` 选项**。

## 5. 手工打包（三）：Dynamic Agent打印方法接收的参数

### 5.1 预期目标

我们的预期目标：借助于 JDK 内置的 ASM 打印出方法接收的参数，使用 **Dynamic Instrumentation** 的方式实现。

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-dynamic-instrumentation.png)

开发环境：

- JDK 版本：Java 8
- 编辑器：记事本（Windows）或 `vi` （Linux）

代码目录结构：[Code](https://lsieun.github.io/assets/zip/java-agent-manual-03.zip)

```pseudocode
java-agent-manual-03
└─── src
     ├─── attach
     │    └─── VMAttach.java
     ├─── lsieun
     │    ├─── agent
     │    │    └─── DynamicAgent.java
     │    ├─── asm
     │    │    ├─── adapter
     │    │    │    └─── MethodInfoAdapter.java
     │    │    ├─── cst
     │    │    │    └─── Const.java
     │    │    └─── visitor
     │    │         └─── MethodInfoVisitor.java
     │    ├─── instrument
     │    │    └─── ASMTransformer.java
     │    └─── utils
     │         └─── ParameterUtils.java
     ├─── manifest.txt
     └─── sample
          ├─── HelloWorld.java
          └─── Program.java
```

做一些准备工作（`prepare03.sh`）：

```shell
DIR=java-agent-manual-03
mkdir ${DIR} && cd ${DIR}

mkdir -p src/sample
touch src/sample/{HelloWorld.java,Program.java}

mkdir -p src/lsieun/{agent,asm,instrument,utils}
mkdir -p src/lsieun/asm/{adapter,cst,visitor}
touch src/lsieun/agent/DynamicAgent.java
touch src/lsieun/instrument/ASMTransformer.java
touch src/lsieun/asm/adapter/MethodInfoAdapter.java
touch src/lsieun/asm/cst/Const.java
touch src/lsieun/asm/visitor/MethodInfoVisitor.java
touch src/lsieun/utils/ParameterUtils.java
touch src/manifest.txt

mkdir -p src/attach
touch src/attach/VMAttach.java
```

### 5.2 Application

#### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }
}
```

#### Program.java

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // (1) print process id
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // (2) count down
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

### 5.3 ASM相关

在这个部分，我们要借助于 JDK 内置的 ASM 类库（`jdk.internal.org.objectweb.asm`），来实现打印方法参数的功能。

#### ParameterUtils.java

```java
package lsieun.utils;

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

public class ParameterUtils {
  private static final DateFormat fm = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

  public static void printValueOnStack(boolean value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(byte value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(char value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(short value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(int value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(float value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(long value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(double value) {
    System.out.println("    " + value);
  }

  public static void printValueOnStack(Object value) {
    if (value == null) {
      System.out.println("    " + value);
    }
    else if (value instanceof String) {
      System.out.println("    " + value);
    }
    else if (value instanceof Date) {
      System.out.println("    " + fm.format(value));
    }
    else if (value instanceof char[]) {
      System.out.println("    " + Arrays.toString((char[]) value));
    }
    else if (value instanceof Object[]) {
      System.out.println("    " + Arrays.toString((Object[]) value));
    }
    else {
      System.out.println("    " + value.getClass() + ": " + value.toString());
    }
  }

  public static void printText(String str) {
    System.out.println(str);
  }

  public static void printStackTrace() {
    Exception ex = new Exception();
    ex.printStackTrace(System.out);
  }
}
```

#### Const.java

```java
package lsieun.asm.cst;

import jdk.internal.org.objectweb.asm.Opcodes;

public class Const {
  public static final int ASM_VERSION = Opcodes.ASM5;
}
```

#### MethodInfoAdapter.java

```java
package lsieun.asm.adapter;

import jdk.internal.org.objectweb.asm.MethodVisitor;
import jdk.internal.org.objectweb.asm.Opcodes;
import jdk.internal.org.objectweb.asm.Type;
import lsieun.asm.cst.Const;

public class MethodInfoAdapter extends MethodVisitor {
  private final String owner;
  private final int methodAccess;
  private final String methodName;
  private final String methodDesc;

  public MethodInfoAdapter(MethodVisitor methodVisitor, String owner,
                           int methodAccess, String methodName, String methodDesc) {
    super(Const.ASM_VERSION, methodVisitor);
    this.owner = owner;
    this.methodAccess = methodAccess;
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public void visitCode() {
    if (mv != null) {
      String line = String.format("Method Enter: %s.%s:%s", owner, methodName, methodDesc);
      printMessage(line);

      int slotIndex = (methodAccess & Opcodes.ACC_STATIC) != 0 ? 0 : 1;
      Type methodType = Type.getMethodType(methodDesc);
      Type[] argumentTypes = methodType.getArgumentTypes();
      for (Type t : argumentTypes) {
        int sort = t.getSort();
        int size = t.getSize();
        int opcode = t.getOpcode(Opcodes.ILOAD);
        super.visitVarInsn(opcode, slotIndex);

        if (sort >= Type.BOOLEAN && sort <= Type.DOUBLE) {
          String desc = t.getDescriptor();
          printValueOnStack("(" + desc + ")V");
        }
        else {
          printValueOnStack("(Ljava/lang/Object;)V");
        }
        slotIndex += size;
      }
    }

    super.visitCode();
  }

  private void printMessage(String str) {
    super.visitLdcInsn(str);
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);
  }

  private void printValueOnStack(String descriptor) {
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printValueOnStack", descriptor, false);
  }

  private void printStackTrace() {
    super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printStackTrace", "()V", false);
  }
}
```

#### MethodInfoVisitor.java

```java
package lsieun.asm.visitor;

import jdk.internal.org.objectweb.asm.ClassVisitor;
import jdk.internal.org.objectweb.asm.MethodVisitor;
import lsieun.asm.adapter.MethodInfoAdapter;
import lsieun.asm.cst.Const;

public class MethodInfoVisitor extends ClassVisitor {
  private String owner;

  public MethodInfoVisitor(ClassVisitor classVisitor) {
    super(Const.ASM_VERSION, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !name.equals("<init>") && !name.equals("<clinit>")) {
      mv = new MethodInfoAdapter(mv, owner, access, name, descriptor);
    }
    return mv;
  }
}
```

### 5.4 Agent Jar

#### manifest.txt

```txt
Agent-Class: lsieun.agent.DynamicAgent
Can-Retransform-Classes: true

```

注意：在 `manifest.txt` 文件的结尾处有**一个空行**。(make sure the last line in the file is **a blank line**)

#### DynamicAgent.java

```java
package lsieun.agent;

import lsieun.instrument.ASMTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    ClassFileTransformer transformer = new ASMTransformer();
    try {
      inst.addTransformer(transformer, true);
      Class<?> targetClass = Class.forName("sample.HelloWorld");
      inst.retransformClasses(targetClass);
    } catch (Exception ex) {
      ex.printStackTrace();
    }
    finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

#### ASMTransformer.java

```java
package lsieun.instrument;

import jdk.internal.org.objectweb.asm.ClassReader;
import jdk.internal.org.objectweb.asm.ClassVisitor;
import jdk.internal.org.objectweb.asm.ClassWriter;
import lsieun.asm.visitor.MethodInfoVisitor;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class ASMTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className == null) return null;
    if (className.startsWith("java")) return null;
    if (className.startsWith("javax")) return null;
    if (className.startsWith("jdk")) return null;
    if (className.startsWith("sun")) return null;
    if (className.startsWith("org")) return null;
    if (className.startsWith("com")) return null;
    if (className.startsWith("lsieun")) return null;

    System.out.println("candidate className: " + className);

    if (className.equals("sample/HelloWorld")) {
      ClassReader cr = new ClassReader(classfileBuffer);
      ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
      ClassVisitor cv = new MethodInfoVisitor(cw);

      int parsingOptions = 0;
      cr.accept(cv, parsingOptions);

      return cw.toByteArray();
    }

    return null;
  }
}
```

### 5.5 JVM Attach

#### VMAttach.java

在下面的代码中，我们要用到 `com.sun.tools.attach` 里定义的类，因此编译的时候需要用到 `tools.jar` 文件。

```java
package attach;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.util.List;

public class VMAttach {
  public static void main(String[] args) throws Exception {
    String agent = "TheAgent.jar";
    System.out.println("Agent Path: " + agent);
    List<VirtualMachineDescriptor> vmds = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : vmds) {
      if (vmd.displayName().equals("sample.Program")) {
        VirtualMachine vm = VirtualMachine.attach(vmd.id());
        System.out.println("Load Agent");
        vm.loadAgent(agent);
        System.out.println("Detach");
        vm.detach();
      }
    }
  }
}
```

### 5.6 编译和运行

#### 编译

编译Application：

```shell
# 切换目录
$ cd java-agent-manual-03/

# 输出目录
$ mkdir out

# 编译
$ javac src/sample/*.java -d out/
```

编译和生成 Agent Jar：

```shell
# 切换目录
$ cd java-agent-manual-03/

# 找到所有 .java 文件
$ find ./src/lsieun/ -name "*.java" > sources.txt
$ cat sources.txt
./src/lsieun/agent/DynamicAgent.java
./src/lsieun/asm/adapter/MethodInfoAdapter.java
./src/lsieun/asm/cst/Const.java
./src/lsieun/asm/visitor/MethodInfoVisitor.java
./src/lsieun/instrument/ASMTransformer.java
./src/lsieun/utils/ParameterUtils.java

# 错误的编译
$ javac -d out/ @sources.txt

# 正确的编译
$ javac -XDignore.symbol.file -d out/ @sources.txt

# 复制 manifest.txt 文件
$ cp src/manifest.txt out/

# 切换目录
$ cd out/
$ ls
lsieun/  manifest.txt  sample/

# 进行打包
$ jar -cvfm TheAgent.jar manifest.txt lsieun/
```

编译 VM Attach：

```shell
# 切换目录
$ cd java-agent-manual-03/

# 编译 VMAttach.java（Windows)
$ javac -cp "%JAVA_HOME%/lib/tools.jar";. -d out/ src/attach/VMAttach.java

# 编译 VMAttach.java（Linux)
$ javac -cp "${JAVA_HOME}/lib/tools.jar":. -d out/ src/attach/VMAttach.java

# 编译 VMAttach.java（MINGW64)
$ javac -cp "${JAVA_HOME}/lib/tools.jar"\;. -d out/ src/attach/VMAttach.java
```

#### 运行

运行：

```shell
# 切换目录
$ cd java-agent-manual-03/
$ cd out/

# 1. 先运行需要被agent处理的JVM
$ java sample.Program

# 2. 后运行 VMAttach.java（Windows)
$ java -cp "%JAVA_HOME%/lib/tools.jar";. attach.VMAttach

# 2. 后运行 VMAttach.java（Linux)
$ java -cp "${JAVA_HOME}/lib/tools.jar":. attach.VMAttach

# 2. 后运行 VMAttach.java（MINGW64)
$ java -cp "${JAVA_HOME}/lib/tools.jar"\;. attach.VMAttach
```

前后输出对比：

```shell
9094@ashiamddeMacBook-Pro.local
|000| 9094@ashiamddeMacBook-Pro.local remains 600 seconds
a - b = -3
|001| 9094@ashiamddeMacBook-Pro.local remains 599 seconds
a + b = 5
|002| 9094@ashiamddeMacBook-Pro.local remains 598 seconds
a + b = 9
candidate className: sample/HelloWorld
|003| 9094@ashiamddeMacBook-Pro.local remains 597 seconds
Method Enter: sample/HelloWorld.add:(II)I
    1
    1
a + b = 2
|004| 9094@ashiamddeMacBook-Pro.local remains 596 seconds
Method Enter: sample/HelloWorld.add:(II)I
    2
    8
a + b = 10
|005| 9094@ashiamddeMacBook-Pro.local remains 595 seconds
Method Enter: sample/HelloWorld.sub:(II)I
    2
    3
a - b = -1
# ...
```

### 5.7 总结

本文内容总结如下：

- 第一点，本文的主要目的是对 Java Agent 有一个整体的印象，因此不需要理解技术细节。
- 第二点，Java Agent 的 Jar 包当中有三个重要组成部分：manifest、Agent Class 和 ClassFileTransformer。
- 第三点，当使用 `javac` 命令编译时，如果在程序当中使用到了 `jdk.*` 或 `sun.*` 当中的类，要添加 `-XDignore.symbol.file` 选项。
- 第四点，**当运行 Dynamic Instrumentation 时，需要在 `CLASSPATH` 当中引用 `JAVA_HOME/lib/tools.jar`**。

## 6. Maven：Load-Time Agent和Dynamic Agent

### 6.1 预期目标

我们的预期目标：打印方法接收的参数值和返回值，借助于 Maven 管理依赖和进行编译，避免手工打Jar包的麻烦。

本文内容虽然很多，但是我们静下心来想一想，它有一个简单的目标：生成一个 Agent Jar。因此，在过程当中的内容细节，都是为 `TheAgent.jar` 做一定的铺垫。

新建一个 Maven 项目，取名为 `java-agent-maven`，代码目录结构：[Code](https://lsieun.github.io/assets/zip/java-agent-maven.zip)

```
java-agent-maven
├─── pom.xml
└─── src
     └─── main
          └─── java
               ├─── lsieun
               │    ├─── agent
               │    │    ├─── DynamicAgent.java
               │    │    └─── LoadTimeAgent.java
               │    ├─── asm
               │    │    ├─── adapter
               │    │    │    └─── PrintMethodInfoStdAdapter.java
               │    │    ├─── cst
               │    │    │    └─── Const.java
               │    │    └─── visitor
               │    │         ├─── MethodInfo.java
               │    │         └─── PrintMethodInfoVisitor.java
               │    ├─── instrument
               │    │    └─── ASMTransformer.java
               │    └─── Main.java
               ├─── run
               │    ├─── DynamicInstrumentation.java
               │    ├─── LoadTimeInstrumentation.java
               │    └─── PathManager.java
               └─── sample
                    ├─── HelloWorld.java
                    └─── Program.java
```

**问题：为什么没有 `manifest.txt` 文件呢？**

**回答：因为 `META-INF/MANIFEST.MF` 的信息由 `pom.xml` 文件中 `maven-jar-plugin` 提供。**

生成Jar文件，我们有三种选择：

- 第一种，`maven-jar-plugin` + `maven-dependency-plugin`
- 第二种，`maven-assembly-plugin`
- 第三种，`maven-shade-plugin`

### 6.2 pom.xml

在 Maven 项目当中，一个非常重要的配置就是 `pom.xml` 文件。

#### properties

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <java.version>1.8</java.version>
  <maven.compiler.source>${java.version}</maven.compiler.source>
  <maven.compiler.target>${java.version}</maven.compiler.target>
  <asm.version>9.0</asm.version>
</properties>
```

#### dependencies

```xml
<dependencies>
</dependencies>
```

##### ASM

在这里不再使用 JDK 内置的 ASM 类库，因为内置的版本比较低。

我们想使用更高的 ASM 版本，也就能够支持更高版本 `.class` 文件操作。

```xml
<dependency>
  <groupId>org.ow2.asm</groupId>
  <artifactId>asm</artifactId>
  <version>${asm.version}</version>
</dependency>
<dependency>
  <groupId>org.ow2.asm</groupId>
  <artifactId>asm-util</artifactId>
  <version>${asm.version}</version>
</dependency>
<dependency>
  <groupId>org.ow2.asm</groupId>
  <artifactId>asm-commons</artifactId>
  <version>${asm.version}</version>
</dependency>
<dependency>
  <groupId>org.ow2.asm</groupId>
  <artifactId>asm-tree</artifactId>
  <version>${asm.version}</version>
</dependency>
<dependency>
  <groupId>org.ow2.asm</groupId>
  <artifactId>asm-analysis</artifactId>
  <version>${asm.version}</version>
</dependency>
```

##### tools.jar

**在 `tools.jar` 文件当中，包含了 `com.sun.tools.attach.VirtualMachine` 类，会在 `DynamicInstrumentation` 类当中用到**。

在 Java 9 之后的版本，引入了模块化系统，`com.sun.tools.attach` 包位于 `jdk.attach` 模块。

```xml
<dependency>
  <groupId>com.sun</groupId>
  <artifactId>tools</artifactId>
  <version>8</version>
  <scope>system</scope>
  <systemPath>${env.JAVA_HOME}/lib/tools.jar</systemPath>
</dependency>
```

#### plugins

```xml
<build>
  <finalName>TheAgent</finalName>
  <plugins>
  </plugins>
</build>
```

##### compiler-plugin

下面的 `maven-compiler-plugin` 插件主要关注 `compilerArgs` 下的三个参数：

- `-g`: 生成所有调试信息
- `-parameters`: 生成 属性
- `-XDignore.symbol.file`: 在编译过程中，进行link时，不使用 `lib/ct.sym`，而是直接使用 `rt.jar` 文件。

```xml
<!-- Java Compiler -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.8.1</version>
  <configuration>
    <source>${java.version}</source>
    <target>${java.version}</target>
    <fork>true</fork>
    <compilerArgs>
      <arg>-g</arg>
      <arg>-parameters</arg>
      <arg>-XDignore.symbol.file</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

##### jar-plugin

下面的[`maven-jar-plugin`](https://maven.apache.org/shared/maven-archiver/index.html)插件主要做以下两件事情：

- 第一，设置`META-INF/MANIFEST.MF`中的信息。
- 第二，确定在jar包当中包含哪些文件。

关于 `<archive>` 的配置，可以参考 [Apache Maven Archiver](https://maven.apache.org/shared/maven-archiver/index.html)。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.2.0</version>
  <configuration>
    <archive>
      <manifest>
        <mainClass>lsieun.Main</mainClass>
        <addClasspath>true</addClasspath>
        <classpathPrefix>lib/</classpathPrefix>
        <addDefaultImplementationEntries>false</addDefaultImplementationEntries>
        <addDefaultSpecificationEntries>false</addDefaultSpecificationEntries>
      </manifest>
      <manifestEntries>
        <Premain-Class>lsieun.agent.LoadTimeAgent</Premain-Class>
        <Agent-Class>lsieun.agent.DynamicAgent</Agent-Class>
        <Launcher-Agent-Class>lsieun.agent.LauncherAgent</Launcher-Agent-Class>
        <Can-Redefine-Classes>true</Can-Redefine-Classes>
        <Can-Retransform-Classes>true</Can-Retransform-Classes>
        <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
      </manifestEntries>
      <addMavenDescriptor>false</addMavenDescriptor>
    </archive>
    <includes>
      <include>lsieun/**</include>
    </includes>
  </configuration>
</plugin>
```

如果我们想使用配置文件，可以使用 `manifestFile` ：

```xml
<configuration>
  <archive>
    <manifestFile>src/main/resources/manifest.mf</manifestFile>
  </archive>
</configuration>
```

##### dependency-plugin

下面的 `maven-dependency-plugin` 插件主要目的：将依赖的 jar 包复制到 `lib` 目录下。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>3.2.0</version>
  <executions>
    <execution>
      <id>lib-copy-dependencies</id>
      <phase>package</phase>
      <goals>
        <goal>copy-dependencies</goal>
      </goals>
      <configuration>
        <excludeArtifactIds>tools</excludeArtifactIds>
        <outputDirectory>${project.build.directory}/lib</outputDirectory>
        <overWriteReleases>false</overWriteReleases>
        <overWriteSnapshots>false</overWriteSnapshots>
        <overWriteIfNewer>true</overWriteIfNewer>
      </configuration>
    </execution>
  </executions>
</plugin>
```

##### assembly-plugin

下面的 [`maven-assembly-plugin`](https://maven.apache.org/plugins/maven-assembly-plugin/index.html) 插件主要目的：生成一个 jar 文件，它包含了依赖的 jar 包。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <version>3.3.0</version>
  <configuration>
    <archive>
      <manifest>
        <mainClass>lsieun.Main</mainClass>
        <addDefaultEntries>false</addDefaultEntries>
      </manifest>
      <manifestEntries>
        <Premain-Class>lsieun.agent.LoadTimeAgent</Premain-Class>
        <Agent-Class>lsieun.agent.DynamicAgent</Agent-Class>
        <Launcher-Agent-Class>lsieun.agent.LauncherAgent</Launcher-Agent-Class>
        <Can-Redefine-Classes>true</Can-Redefine-Classes>
        <Can-Retransform-Classes>true</Can-Retransform-Classes>
        <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
      </manifestEntries>
    </archive>
    <descriptorRefs>
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
  </configuration>
  <executions>
    <execution>
      <id>make-assembly</id>
      <phase>package</phase>
      <goals>
        <goal>single</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

##### shade-plugin

下面的 [`maven-shade-plugin`](https://maven.apache.org/plugins/maven-shade-plugin/index.html) 插件主要目的：生成一个 jar 文件，它包含了依赖的 jar 包，可以进行精简。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.2.4</version>
  <configuration>
    <minimizeJar>true</minimizeJar>
    <filters>
      <filter>
        <artifact>*:*</artifact>
        <excludes>
          <exclude>run/*</exclude>
          <exclude>sample/*</exclude>
        </excludes>
      </filter>
    </filters>
  </configuration>
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>shade</goal>
      </goals>
      <configuration>
        <transformers>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <manifestEntries>
              <Main-Class>lsieun.Main</Main-Class>
              <Premain-Class>lsieun.agent.LoadTimeAgent</Premain-Class>
              <Agent-Class>lsieun.agent.DynamicAgent</Agent-Class>
              <Launcher-Agent-Class>lsieun.agent.LauncherAgent</Launcher-Agent-Class>
              <Can-Redefine-Classes>true</Can-Redefine-Classes>
              <Can-Retransform-Classes>true</Can-Retransform-Classes>
              <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
            </manifestEntries>
          </transformer>
        </transformers>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### 6.3 Application

#### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }
}
```

#### Program.java

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // (1) print process id
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // (2) count down
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

### 6.4 ASM相关

#### Const.java

```java
package lsieun.asm.cst;

import org.objectweb.asm.Opcodes;

public class Const {
  public static final int ASM_VERSION = Opcodes.ASM9;
}
```

#### PrintMethodInfoStdAdapter.java

```java
package lsieun.asm.adapter;

import lsieun.asm.cst.Const;
import lsieun.asm.visitor.MethodInfo;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;

import java.util.Objects;
import java.util.Set;

public class PrintMethodInfoStdAdapter extends MethodVisitor implements Opcodes {
  private final String owner;
  private final int methodAccess;
  private final String methodName;
  private final String methodDesc;
  private final Set<MethodInfo> flags;

  public PrintMethodInfoStdAdapter(MethodVisitor methodVisitor,
                                   String owner, int methodAccess, String methodName, String methodDesc,
                                   Set<MethodInfo> flags) {
    super(Const.ASM_VERSION, methodVisitor);

    Objects.requireNonNull(flags);

    this.owner = owner;
    this.methodAccess = methodAccess;
    this.methodName = methodName;
    this.methodDesc = methodDesc;
    this.flags = flags;

  }

  @Override
  public void visitCode() {
    if (mv != null) {
      if (flags.contains(MethodInfo.NAME_AND_DESC)) {
        String line = String.format("Method Enter: %s.%s:%s", owner, methodName, methodDesc);
        printMessage(line);
      }

      if (flags.contains(MethodInfo.PARAMETER_VALUES)) {
        int slotIndex = (methodAccess & Opcodes.ACC_STATIC) != 0 ? 0 : 1;
        Type methodType = Type.getMethodType(methodDesc);
        Type[] argumentTypes = methodType.getArgumentTypes();
        for (Type t : argumentTypes) {
          printParameter(slotIndex, t);

          int size = t.getSize();
          slotIndex += size;
        }
      }

      if (flags.contains(MethodInfo.CLASSLOADER)) {
        printClassLoader();
      }

      if (flags.contains(MethodInfo.THREAD_INFO)) {
        printThreadInfo();
      }

      if (flags.contains(MethodInfo.STACK_TRACE)) {
        printStackTrace();
      }
    }

    super.visitCode();
  }

  @Override
  public void visitInsn(int opcode) {
    if (flags.contains(MethodInfo.RETURN_VALUE)) {
      Type t = Type.getMethodType(methodDesc);
      Type returnType = t.getReturnType();

      if (opcode == Opcodes.ATHROW) {
        String line = String.format("Method throws Exception: %s.%s:%s", owner, methodName, methodDesc);
        printMessage(line);
        String message = "    abnormal return";
        printMessage(message);
        printMessage("=================================================================================");
      }
      else if (opcode == Opcodes.RETURN) {
        String line = String.format("Method Return: %s.%s:%s", owner, methodName, methodDesc);
        printMessage(line);
        String message = "    return void";
        printMessage(message);
        printMessage("=================================================================================");
      }
      else if (opcode >= Opcodes.IRETURN && opcode <= Opcodes.ARETURN) {
        String line = String.format("Method Return: %s.%s:%s", owner, methodName, methodDesc);
        printMessage(line);

        printReturnValue(returnType);
        printMessage("=================================================================================");
      }
      else {
        assert false : "should not be here";
      }
    }


    super.visitInsn(opcode);
  }

  private void printMessage(String message) {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitLdcInsn(message);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
  }

  private void printParameter(int slotIndex, Type t) {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitTypeInsn(NEW, "java/lang/StringBuilder");
    super.visitInsn(DUP);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
    super.visitLdcInsn("    ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    if (slotIndex >= 0 && slotIndex <= 5) {
      super.visitInsn(ICONST_0 + slotIndex);
    }
    else {
      super.visitIntInsn(BIPUSH, slotIndex);
    }

    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(I)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn(": ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

    int opcode = t.getOpcode(Opcodes.ILOAD);
    super.visitVarInsn(opcode, slotIndex);

    int sort = t.getSort();
    String descriptor;
    if (sort == Type.SHORT) {
      descriptor = "(I)Ljava/lang/StringBuilder;";
    }
    else if (sort >= Type.BOOLEAN && sort <= Type.DOUBLE) {
      descriptor = "(" + t.getDescriptor() + ")Ljava/lang/StringBuilder;";
    }
    else {
      descriptor = "(Ljava/lang/Object;)Ljava/lang/StringBuilder;";
    }

    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", descriptor, false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
  }

  private void printThreadInfo() {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitTypeInsn(NEW, "java/lang/StringBuilder");
    super.visitInsn(DUP);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
    super.visitLdcInsn("Thread Id: ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKESTATIC, "java/lang/Thread", "currentThread", "()Ljava/lang/Thread;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Thread", "getName", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn("@");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKESTATIC, "java/lang/Thread", "currentThread", "()Ljava/lang/Thread;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Thread", "getId", "()J", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn("(");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKESTATIC, "java/lang/Thread", "currentThread", "()Ljava/lang/Thread;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Thread", "isDaemon", "()Z", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Z)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn(")");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
  }

  private void printClassLoader() {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitTypeInsn(NEW, "java/lang/StringBuilder");
    super.visitInsn(DUP);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
    super.visitLdcInsn("ClassLoader: ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn(Type.getObjectType(owner));
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Class", "getClassLoader", "()Ljava/lang/ClassLoader;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/Object;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
  }

  private void printStackTrace() {
    super.visitTypeInsn(NEW, "java/lang/Exception");
    super.visitInsn(DUP);
    super.visitTypeInsn(NEW, "java/lang/StringBuilder");
    super.visitInsn(DUP);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
    super.visitLdcInsn("Exception from ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitLdcInsn(Type.getObjectType(owner));
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Class", "getName", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/Exception", "<init>", "(Ljava/lang/String;)V", false);
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "(Ljava/io/PrintStream;)V", false);
  }

  private void printReturnValue(Type returnType) {
    int size = returnType.getSize();
    if (size == 1) {
      super.visitInsn(DUP);
    }
    else if (size == 2) {
      super.visitInsn(DUP2);
    }
    else {
      assert false : "should not be here";
    }

    printValueOnStack(returnType);
  }

  private void printValueOnStack(Type t) {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    int size = t.getSize();
    if (size == 1) {
      super.visitInsn(SWAP);
    }
    else if (size == 2) {
      super.visitInsn(DUP_X2);
      super.visitInsn(POP);
    }
    else {
      assert false : "should not be here";
    }

    super.visitTypeInsn(NEW, "java/lang/StringBuilder");
    super.visitInsn(DUP);
    super.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
    super.visitLdcInsn("    ");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

    if (size == 1) {
      super.visitInsn(SWAP);
    }
    else {
      super.visitInsn(DUP_X2);
      super.visitInsn(POP);
    }

    int sort = t.getSort();
    String descriptor;
    if (sort == Type.SHORT) {
      descriptor = "(I)Ljava/lang/StringBuilder;";
    }
    else if (sort >= Type.BOOLEAN && sort <= Type.DOUBLE) {
      descriptor = "(" + t.getDescriptor() + ")Ljava/lang/StringBuilder;";
    }
    else {
      descriptor = "(Ljava/lang/Object;)Ljava/lang/StringBuilder;";
    }

    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", descriptor, false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
  }
}
```

#### MethodInfo.java

```java
package lsieun.asm.visitor;

import java.util.EnumSet;

public enum MethodInfo {
  NAME_AND_DESC,
  PARAMETER_VALUES,
  RETURN_VALUE,
  CLASSLOADER,
  STACK_TRACE,
  THREAD_INFO;

  public static final EnumSet<MethodInfo> ALL = EnumSet.allOf(MethodInfo.class);
}
```

#### PrintMethodInfoVisitor.java

```java
package lsieun.asm.visitor;

import lsieun.asm.adapter.PrintMethodInfoStdAdapter;
import lsieun.asm.cst.Const;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

import java.util.Set;

public class PrintMethodInfoVisitor extends ClassVisitor {
  private static final String ALL = "*";

  private String owner;
  private final String methodName;
  private final String methodDesc;
  private final Set<MethodInfo> flags;

  public PrintMethodInfoVisitor(ClassVisitor classVisitor, Set<MethodInfo> flags) {
    this(classVisitor, ALL, ALL, flags);
  }

  public PrintMethodInfoVisitor(ClassVisitor classVisitor, String methodName, String methodDesc, Set<MethodInfo> flags) {
    super(Const.ASM_VERSION, classVisitor);
    this.methodName = methodName;
    this.methodDesc = methodDesc;
    this.flags = flags;
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);

    if (mv == null) return mv;

    boolean isAbstract = (access & Opcodes.ACC_ABSTRACT) != 0;
    boolean isNative = (access & Opcodes.ACC_NATIVE) != 0;
    if (isAbstract || isNative) return mv;

    if (name.equals("<init>") || name.equals("<clinit>")) return mv;

    boolean process = false;
    if (ALL.equals(methodName) && ALL.equals(methodDesc)) {
      process = true;
    }
    else if (name.equals(methodName) && ALL.equals(methodDesc)) {
      process = true;
    }
    else if (name.equals(methodName) && descriptor.equals(methodDesc)) {
      process = true;
    }

    if (process) {
      String line = String.format("---> %s.%s:%s", owner, name, descriptor);
      System.out.println(line);
      mv = new PrintMethodInfoStdAdapter(mv, owner, access, name, descriptor, flags);
    }

    return mv;
  }
}
```

### 6.5 Agent Jar

#### LoadTimeAgent.java

```java
package lsieun.agent;

import lsieun.instrument.ASMTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");

    ClassFileTransformer transformer = new ASMTransformer("sample/HelloWorld");
    inst.addTransformer(transformer, false);
  }
}
```

#### DynamicAgent.java

```java
package lsieun.agent;

import lsieun.instrument.ASMTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    System.out.println("Agent-Class: " + DynamicAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");

    ClassFileTransformer transformer = new ASMTransformer("sample/HelloWorld");
    inst.addTransformer(transformer, true);

    try {
      Class<?> targetClass = Class.forName("sample.HelloWorld");
      inst.retransformClasses(targetClass);
    } catch (Exception ex) {
      ex.printStackTrace();
    }
    finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

#### LauncherAgent.java

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LauncherAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    System.out.println("Launcher-Agent-Class: " + LauncherAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");
  }
}
```

#### ASMTransformer.java

```java
package lsieun.instrument;

import lsieun.asm.visitor.*;
import org.objectweb.asm.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.*;

public class ASMTransformer implements ClassFileTransformer {
  public static final List<String> ignoredPackages = Arrays.asList("com/", "com/sun/", "java/", "javax/", "jdk/", "lsieun/", "org/", "sun/");

  private final String internalName;

  public ASMTransformer(String internalName) {
    Objects.requireNonNull(internalName);
    this.internalName = internalName.replace(".", "/");
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className == null) return null;

    for (String name : ignoredPackages) {
      if (className.startsWith(name)) {
        return null;
      }
    }
    System.out.println("candidate class: " + className);

    if (className.equals(internalName)) {
      System.out.println("transform class: " + className);
      ClassReader cr = new ClassReader(classfileBuffer);
      ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
      Set<MethodInfo> flags = EnumSet.of(
        MethodInfo.NAME_AND_DESC,
        MethodInfo.PARAMETER_VALUES,
        MethodInfo.RETURN_VALUE);
      ClassVisitor cv = new PrintMethodInfoVisitor(cw, flags);

      int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
      cr.accept(cv, parsingOptions);

      return cw.toByteArray();
    }

    return null;
  }
}
```

#### Main.java

```java
package lsieun;

public class Main {
  public static void main(String[] args) {
    System.out.println("This is a Java Agent Jar");
  }
}
```

### 6.6 Run

#### LoadTimeInstrumentation.java

```java
package run;

import java.util.Formatter;

public class LoadTimeInstrumentation {
  public static void main(String[] args) {
    usage();
  }

  public static void usage() {
    String jarPath = PathManager.getJarPath();
    StringBuilder sb = new StringBuilder();
    Formatter fm = new Formatter(sb);
    fm.format("Usage:%n");
    fm.format("    java -javaagent:/path/to/TheAgent.jar sample.Program%n");
    fm.format("Example:%n");
    fm.format("    java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program%n");
    fm.format("    java -cp ./target/classes/ -javaagent:%s sample.Program", jarPath);
    String result = sb.toString();
    System.out.println(result);
  }
}
```

#### DynamicInstrumentation.java

```java
package run;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.util.List;

public class DynamicInstrumentation {
  public static void main(String[] args) throws Exception {
    String agent = PathManager.getJarPath();
    System.out.println("Agent Path: " + agent);
    List<VirtualMachineDescriptor> vmds = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : vmds) {
      if (vmd.displayName().equals("sample.Program")) {
        VirtualMachine vm = VirtualMachine.attach(vmd.id());
        vm.getSystemProperties();
        System.out.println("Load Agent");
        vm.loadAgent(agent);
        System.out.println("Detach");
        vm.detach();
      }
    }
  }
}
```

#### PathManager.java

```java
package run;

import java.io.File;
import java.net.URISyntaxException;

public class PathManager {
  public static String getJarPath() {
    String filepath = null;

    try {
      filepath = new File(PathManager.class.getProtectionDomain().getCodeSource().getLocation().toURI()).getPath();
    } catch (URISyntaxException ex) {
      ex.printStackTrace();
    }

    if (filepath == null || !filepath.endsWith(".jar")) {
      filepath = System.getProperty("user.dir") + File.separator + "target/TheAgent.jar";
    }

    return filepath.replace(File.separator, "/");
  }
}
```

#### 生成Jar包和运行

生成 Jar 包：

```shell
mvn clean package
```

上述命令执行完成之后，会在 `target` 文件夹下生成 `TheAgent.jar`，其内容如下：

```pseudocode
TheAgent.jar
├─── META-INF/MANIFEST.MF
├─── lsieun/agent/DynamicAgent.class
├─── lsieun/agent/LoadTimeAgent.class
├─── lsieun/asm/adapter/PrintMethodInfoStdAdapter.class
├─── lsieun/asm/cst/Const.class
├─── lsieun/asm/visitor/MethodInfo.class
├─── lsieun/asm/visitor/PrintMethodInfoVisitor.class
├─── lsieun/instrument/ASMTransformer.class
└─── lsieun/Main.class
```

运行 Load-Time Instrumentation：

```shell
# 相对路径
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

# 绝对路径（\\）
$ java -cp ./target/classes/ -javaagent:D:\\git-repo\\java-agent-maven\\target\\TheAgent.jar sample.Program

# 绝对路径（/）
$ java -cp ./target/classes/ -javaagent:D:/git-repo/java-agent-maven/target/TheAgent.jar sample.Program
```

运行 Dynamic Instrumentation：

```shell
# Windows
$ java -cp "%JAVA_HOME%/lib/tools.jar";./target/classes/ run.DynamicInstrumentation

# Linux
$ java -cp "${JAVA_HOME}/lib/tools.jar":./target/classes/ run.DynamicInstrumentation

# MINGW64
$ java -cp "${JAVA_HOME}/lib/tools.jar"\;./target/classes/ run.DynamicInstrumentation
```

如果是 Java 9 及以上的版本，不需要引用 `tools.jar` 文件，可以直接运行：

```shell
$ java -cp ./target/classes/ run.DynamicInstrumentation
```

### 6.7 总结

本文内容总结如下：

- 第一点，使用 Maven 会提供很大的方便，但是 Agent Jar 的核心三要素没有发生变化，包括 manifest、Agent Class 和 ClassFileTransformer，三者缺一不可。
- 第二点，使用 ASM 修改字节码（bytecode）的内容是属于 Java Agent 的“辅助部分”。如果我们熟悉其它的字节码操作类库（例如，Javassist、ByteBuddy），可以将 ASM 替换掉。
- 第三点，细节之处的把握。
  - **在 `pom.xml` 文件中，对 `${env.JAVA_HOME}/lib/tools.jar` 进行了依赖，是因为我们用到 `com.sun.tools.attach.VirtualMachine` 类**。
  - **在 `pom.xml` 文件中，`maven-jar-plugin` 部分提供的与 manifest 相关的信息，会转换到 `META-INF/MANIFEST.MF` 文件中去**。

## 7. Java Agent + Java Module System

### 7.1 从Java 8到Java 9

在Java 8和Java 9版本之间有一个比较大的跨越：模块化系统（Module System）。

如果使用Java 8以后的版本，那么推荐使用Java 11或Java 17，因为它们是LTS（long-term support，长期提供技术支持的）版本。

### 7.2 tools.jar

在Java 8版本中，`com.sun.tools.attach`包位于`tools.jar`文件，来进行Dynamic Attach。在`pom.xml`文件中，有相应的依赖：

```xml
<dependency>
  <groupId>com.sun</groupId>
  <artifactId>tools</artifactId>
  <version>8</version>
  <scope>system</scope>
  <systemPath>${env.JAVA_HOME}/lib/tools.jar</systemPath>
</dependency>
```

相应的，Java 9之后版本，引入了模块化系统（Module System），这样`tools.jar`文件也不存在了。 那么，`com.sun.tools.attach`包位于`jdk.attach`模块当中， 此时需要我们在`module-info.java`文件添加对`jdk.attach`的依赖：

```java
module lsieun.java.agent {
  requires java.instrument;
  requires java.management;
  requires jdk.attach;
  requires org.objectweb.asm;
}
```

## 8. 总结

### 8.1 三个组成部分

在Java Agent对应的`.jar`文件里，有三个主要组成部分：

- Manifest
- Agent Class
- ClassFileTransformer

![Agent Jar中的三个组成部分](https://lsieun.github.io/assets/images/java/agent/agent-jar-three-components.png)

三个组成部分：

```pseudocode
                ┌─── Manifest ───────────────┼─── META-INF/MANIFEST.MF
                │
                │                            ┌─── LoadTimeAgent.class: premain
TheAgent.jar ───┼─── Agent Class ────────────┤
                │                            └─── DynamicAgent.class: agentmain
                │
                └─── ClassFileTransformer ───┼─── ASMTransformer.class
```

彼此之间的关系：

```pseudocode
Manifest --> Agent Class --> Instrumentation --> ClassFileTransformer
```

### 8.2 Load-Time VS. Dynamic

#### Load-Time

在Load-Time Instrumentation当中，只涉及到一个JVM：

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-load-time-instrumentation.png)

在Manifest部分，需要定义`Premain-Class`属性。

在Agent Class部分，需要定义 `premain` 方法。下面是 `premain` 的两种写法：

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

在运行的时候，需要配置 `-javaagent` 选项加载 Agent Jar：

```shell
java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
```

**在运行的过程当中，先执行 Agent Class 的 `premain` 方法，再执行 Application 的 `main` 方法。**

#### Dynamic

在 Dynamic Instrumentation 当中，涉及到两个 JVM：

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-dynamic-instrumentation.png)

在 Manifest 部分，需要定义 `Agent-Class` 属性。

在 Agent Class 部分，需要定义 `agentmain` 方法。下面是 `agentmain` 的两种写法：

```java
public static void agentmain(String agentArgs, Instrumentation inst);
public static void agentmain(String agentArgs);
```

**在运行的时候，需要使用 Attach 机制加载 Agent Jar。**

**在运行的过程当中，一般 Application 的 `main` 方法已经开始执行，而 Agent Class 的 `agentmain` 方法后执行。**

### 8.3 总结

本文内容总结如下：

- 第一点，Agent Jar 的三个组成部分：Manifest、Agent Class 和 ClassFileTransformer。
- 第二点，对 Load-Time Instrumentation 和 Dynamic Instrumentation 有一个初步的理解。
  - Load-Time Instrumentation: `Premain-Class` —> `premain()` —> `-javaagent`
  - Dynamic Instrumentation: `Agent-Class` —> `agentmain()` —> Attach

# 第二章 两种启动方式

## 1. Load-Time: agentArgs参数

首先，我们需要注意：并不是所有的虚拟机，都支持从 command line 启动 Java Agent。

An implementation is not required to provide a way to start agents from the command-line interface. [Link](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

> 问题：当前的 JVM implementation 是否支持从 command line 启动 Java Agent？回答：一般都支持

其次，如何在 command-line 为 Agent Jar 添加参数信息呢？

### 1.1 命令行启动

#### Command-Line

从命令行启动 Java Agent 需要使用 `-javagent` 选项：

```shell
-javaagent:jarpath[=options]
```

- `jarpath` is the path to the agent JAR file.
- `options` is the agent options.

```pseudocode
                                                  ┌─── -javaagent:jarpath
                             ┌─── Command-Line ───┤
                             │                    └─── -javaagent:jarpath=options
Load-Time Instrumentation ───┤
                             │                    ┌─── MANIFEST.MF - Premain-Class: lsieun.agent.LoadTimeAgent
                             └─── Agent Jar ──────┤
                                                  └─── Agent Class - premain(String agentArgs, Instrumentation inst)
```

示例：

```shell
java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=this-is-a-long-message sample.Program
```

#### Agent Jar

在 `TheAgent.jar` 中，依据 `META-INF/MANIFEST.MF` 里定义 `Premain-Class` 属性找到 Agent Class:

```txt
Premain-Class: lsieun.agent.LoadTimeAgent

```

The agent is passed its agent `options` via the `agentArgs` parameter.

```java
public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // ...
  }
}
```

The agent `options` are passed as a single string, any additional parsing should be performed by the agent itself.

![img](https://lsieun.github.io/assets/images/java/agent/java-agent-command-line-options.png)

### 1.2 示例一：读取agentArgs

#### LoadTimeAgent.java

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("agentArgs: " + agentArgs);
    System.out.println("Instrumentation Class: " + inst.getClass().getName());
  }
}
```

#### 运行

每次修改代码之后，都需要重新生成 `.jar` 文件：

```shell
mvn clean package
```

获取示例命令：

```shell
$ cd learn-java-agent

$ java -jar ./target/TheAgent.jar
Load-Time Usage:
    java -javaagent:/path/to/TheAgent.jar sample.Program
Example:
    java -cp ./target/classes/ sample.Program
    java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
```

第一次运行，在使用 `-javagent` 选项时不添加 `options` 信息：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
agentArgs: null
Instrumentation Class: sun.instrument.InstrumentationImpl
```

从上面的输出结果中，我们可以看到：

- 第一点，`agentArgs` 的值为 `null`。
- 第二点，`Instrumentation` 是一个接口，它的具体实现是 `sun.instrument.InstrumentationImpl` 类。

第二次运行，在使用 `-javagent` 选项时添加 `options` 信息：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=this-is-a-long-message sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
agentArgs: this-is-a-long-message
Instrumentation Class: sun.instrument.InstrumentationImpl
```

### 1.3 示例二：解析 agentArgs

我们传入的信息，一般情况下是 `key-value` 的形式，有人喜欢用 `:` 分隔，有人喜欢用 `=` 分隔：

```shell
username:tomcat,password:123456
username=tomcat,password=123456
```

#### LoadTimeAgent.java

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("agentArgs: " + agentArgs);
    System.out.println("Instrumentation Class: " + inst.getClass().getName());

    if (agentArgs != null) {
      String[] array = agentArgs.split(",");
      int length = array.length;
      for (int i = 0; i < length; i++) {
        String item = array[i];
        String[] key_value_pair = getKeyValuePair(item);

        String key = key_value_pair[0];
        String value = key_value_pair[1];

        String line = String.format("|%03d| %s: %s", i, key, value);
        System.out.println(line);
      }
    }
  }

  private static String[] getKeyValuePair(String str) {
    {
      int index = str.indexOf("=");
      if (index != -1) {
        return str.split("=", 2);
      }
    }

    {
      int index = str.indexOf(":");
      if (index != -1) {
        return str.split(":", 2);
      }
    }
    return new String[]{str, ""};
  }
}
```

#### 运行

第一次运行，使用 `:` 分隔：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=username:tomcat,password:123456 sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
agentArgs: username:tomcat,password:123456
Instrumentation Class: sun.instrument.InstrumentationImpl
|000| username: tomcat
|001| password: 123456
```

第二次运行，使用 `=` 分隔：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=username=jerry,password=12345 sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
agentArgs: username=jerry,password=12345
Instrumentation Class: sun.instrument.InstrumentationImpl
|000| username: jerry
|001| password: 12345
```

### 1.4 总结

本文内容总结如下：

- 第一点，在命令行启动 Java Agent，需要使用 `-javaagent:jarpath[=options]` 选项，其中的 `options` 信息会转换成为 `premain` 方法的 `agentArgs` 参数。
- 第二点，对于 `agentArgs` 参数的进一步解析，需要由我们自己来完成。

## 2. Load-Time: inst参数

在 `LoadTimeAgent` 类当中，有一个 `premain` 方法，我们关注两个问题：

- 第一个问题，`Instrumentation` 是一个接口，它的具体实现是哪个类？
- 第二个问题，是“谁”调用了 `LoadTimeAgent.premain()` 方法的呢？

```java
public static void premain(String agentArgs, Instrumentation inst)
```

### 2.1 查看 StackTrace

#### LoadTimeAgent

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("agentArgs: " + agentArgs);
    System.out.println("Instrumentation Class: " + inst.getClass().getName());

    Exception ex = new Exception("Exception from LoadTimeAgent");
    ex.printStackTrace(System.out);
  }
}
```

#### 运行

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
agentArgs: null
Instrumentation Class: sun.instrument.InstrumentationImpl
java.lang.Exception: Exception from LoadTimeAgent
        at lsieun.agent.LoadTimeAgent.premain(LoadTimeAgent.java:11)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
```

从上面的输出结果中，可以看到 `InstrumentationImpl.loadClassAndCallPremain` 方法。

### 2.2 InstrumentationImpl

`sun.instrument.InstrumentationImpl` 实现了 `java.lang.instrument.Instrumentation` 接口：

```java
/**
 * The Java side of the JPLIS implementation. Works in concert with a native JVMTI agent
 * to implement the JPLIS API set. Provides both the Java API implementation of
 * the Instrumentation interface and utility Java routines to support the native code.
 * Keeps a pointer to the native data structure in a scalar field to allow native
 * processing behind native methods.
 */
public class InstrumentationImpl implements Instrumentation {
}
```

#### loadClassAndCallPremain

在 `sun.instrument.InstrumentationImpl` 类当中，`loadClassAndCallPremain` 方法的实现非常简单，它直接调用了 `loadClassAndStartAgent` 方法：

```java
public class InstrumentationImpl implements Instrumentation {
  // WARNING: the native code knows the name & signature of this method
  private void loadClassAndCallPremain(String classname, String optionsString) throws Throwable {
    loadClassAndStartAgent(classname, "premain", optionsString);
  }
}
```

#### loadClassAndCallAgentmain

```java
public class InstrumentationImpl implements Instrumentation {
  // WARNING: the native code knows the name & signature of this method
  private void loadClassAndCallAgentmain(String classname, String optionsString) throws Throwable {
    loadClassAndStartAgent(classname, "agentmain", optionsString);
  }
}
```

#### loadClassAndStartAgent

在 `sun.instrument.InstrumentationImpl` 类当中，`loadClassAndStartAgent` 方法的作用就是通过Java反射的机制来对 `premain` 或 `agentmain` 方法进行调用。

在 `loadClassAndStartAgent` 源码中，我们能够看到更多的细节信息：

- 第一步，从自身的方法定义中，去寻找目标方法：先找带有两个参数的方法；如果没有找到，则找带有一个参数的方法。如果第一步没有找到，则进行第二步。
- 第二步，从父类的方法定义中，去寻找目标方法：先找带有两个参数的方法；如果没有找到，则找带有一个参数的方法。

```java
public class InstrumentationImpl implements Instrumentation {
  // Attempt to load and start an agent
  private void loadClassAndStartAgent(String classname, String methodname, String optionsString) throws Throwable {

    ClassLoader mainAppLoader = ClassLoader.getSystemClassLoader();
    Class<?> javaAgentClass = mainAppLoader.loadClass(classname);

    Method m = null;
    NoSuchMethodException firstExc = null;
    boolean twoArgAgent = false;

    // The agent class must have a premain or agentmain method that
    // has 1 or 2 arguments. We check in the following order:
    //
    // 1) declared with a signature of (String, Instrumentation)
    // 2) declared with a signature of (String)
    // 3) inherited with a signature of (String, Instrumentation)
    // 4) inherited with a signature of (String)
    //
    // So the declared version of either 1-arg or 2-arg always takes
    // primary precedence over an inherited version. After that, the
    // 2-arg version takes precedence over the 1-arg version.
    //
    // If no method is found then we throw the NoSuchMethodException
    // from the first attempt so that the exception text indicates
    // the lookup failed for the 2-arg method (same as JDK5.0).

    try {
      m = javaAgentClass.getDeclaredMethod(methodname,
                                           new Class<?>[]{
                                             String.class,
                                             java.lang.instrument.Instrumentation.class
                                               }
                                          );
      twoArgAgent = true;
    } catch (NoSuchMethodException x) {
      // remember the NoSuchMethodException
      firstExc = x;
    }

    if (m == null) {
      // now try the declared 1-arg method
      try {
        m = javaAgentClass.getDeclaredMethod(methodname, new Class<?>[]{String.class});
      } catch (NoSuchMethodException x) {
        // ignore this exception because we'll try
        // two arg inheritance next
      }
    }

    if (m == null) {
      // now try the inherited 2-arg method
      try {
        m = javaAgentClass.getMethod(methodname,
                                     new Class<?>[]{
                                       String.class,
                                       java.lang.instrument.Instrumentation.class
                                         }
                                    );
        twoArgAgent = true;
      } catch (NoSuchMethodException x) {
        // ignore this exception because we'll try
        // one arg inheritance next
      }
    }

    if (m == null) {
      // finally try the inherited 1-arg method
      try {
        m = javaAgentClass.getMethod(methodname, new Class<?>[]{String.class});
      } catch (NoSuchMethodException x) {
        // none of the methods exists so we throw the
        // first NoSuchMethodException as per 5.0
        throw firstExc;
      }
    }

    // the premain method should not be required to be public,
    // make it accessible so we can call it
    // Note: The spec says the following:
    //     The agent class must implement a public static premain method...
    setAccessible(m, true);

    // invoke the 1 or 2-arg method
    if (twoArgAgent) {
      m.invoke(null, new Object[]{optionsString, this});
    }
    else {
      m.invoke(null, new Object[]{optionsString});
    }

    // don't let others access a non-public premain method
    setAccessible(m, false);
  }
}
```

### 2.3 总结

本文内容总结如下：

- 第一点，在 `premain` 方法中，`Instrumentation` 接口的具体实现是 `sun.instrument.InstrumentationImpl` 类。
- 第二点，查看 Stack Trace，可以看到 `sun.instrument.InstrumentationImpl.loadClassAndCallPremain` 方法对 `LoadTimeAgent.premain` 方法进行了调用。

## 3. Dynamic: Attach API

### 3.1 Attach API

> [Java Attach API使用笔记_墨、鱼的博客-CSDN博客](https://blog.csdn.net/qq_18515155/article/details/114831664)
>
> [com.sun.tools.attach - Java 11中文版 - API参考文档 (apiref.com)](https://www.apiref.com/java11-zh/jdk.attach/com/sun/tools/attach/package-summary.html)

在进行Dynamic Instrumentation的时候，我们需要用到**Attach API，它允许一个JVM连接到另一个JVM**。

> 作用：允许一个JVM连接到另一个JVM

Attach API是在Java 1.6引入的。

> 时间：Java 1.6之后

在Attach API当中，主要的类位于`com.sun.tools.attach`包，它有版本的变化：

- 在Java 8版本，`com.sun.tools.attach`包位于`JDK_HOME/lib/tools.jar`文件。
- 在Java 9版本之后，`com.sun.tools.attach`包位于`jdk.attach`模块（`JDK_HOME/jmods/jdk.attach.jmod`文件）。

> 位置：tools.jar文件或jdk.attach模块

我们主要使用Java 8版本。

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-dynamic-instrumentation.png)

在`com.sun.tools.attach`包当中，包含的类内容如下：

> com.sun.tools.attach有哪些主要的类

```pseudocode
                        ┌─── spi ──────────────────────────────┼─── AttachProvider
                        │
                        ├─── AgentInitializationException
                        │
                        ├─── AgentLoadException
                        │
                        ├─── AttachNotSupportedException
com.sun.tools.attach ───┤
                        ├─── AttachOperationFailedException
                        │
                        ├─── AttachPermission
                        │
                        ├─── VirtualMachine
                        │
                        └─── VirtualMachineDescriptor
```

在上面这些类当中，我们忽略掉其中的Exception和Permission类，简化之后如下：

```pseudocode
                        ┌─── spi ────────────────────────┼─── AttachProvider
                        │
com.sun.tools.attach ───┼─── VirtualMachine
                        │
                        └─── VirtualMachineDescriptor
```

在这三个类当中，核心的类是 `VirtualMachine` 类，代码围绕着它来展开； `VirtualMachineDescriptor` 类比较简单，就是对几个字段（id、provider和display name）的包装； 而 `AttachProvider` 类提供了底层实现。

### 3.2 VirtualMachine

A `com.sun.tools.attach.VirtualMachine` represents a Java virtual machine to which this Java virtual machine has attached. The Java virtual machine to which it is attached is sometimes called the **target virtual machine**, or **target VM**.

![img](https://lsieun.github.io/assets/images/java/agent/vm-attach-load-agent-detach.png)

使用 `VirtualMachine` 类，我们分成三步：

- 第一步，与 target VM 建立连接，获得一个 `VirtualMachine` 对象。
- 第二步，使用 `VirtualMachine` 对象，可以将Agent Jar加载到target VM上，也可以从target VM 读取一些属性信息。
- 第三步，与 target VM 断开连接。

```pseudocode
                                       ┌─── VirtualMachine.attach(String id)
                  ┌─── 1. Get VM ──────┤
                  │                    └─── VirtualMachine.attach(VirtualMachineDescriptor vmd)
                  │
                  │                                            ┌─── VirtualMachine.loadAgent(String agent)
                  │                    ┌─── Load Agent ────────┤
VirtualMachine ───┤                    │                       └─── VirtualMachine.loadAgent(String agent, String options)
                  ├─── 2. Use VM ──────┤
                  │                    │                       ┌─── VirtualMachine.getAgentProperties()
                  │                    └─── read properties ───┤
                  │                                            └─── VirtualMachine.getSystemProperties()
                  │
                  └─── 3. detach VM ───┼─── VirtualMachine.detach()
```

#### Get VM

##### attach 1

A `VirtualMachine` is obtained by invoking the `attach` method with an identifier that identifies the target virtual machine. The identifier is implementation-dependent but is typically the process identifier (or pid) in environments where each Java virtual machine runs in its own operating system process.

```java
public abstract class VirtualMachine {
  public static VirtualMachine attach(String id) throws AttachNotSupportedException, IOException {
    // ...
  }
}
```

##### attach 2

Alternatively, a `VirtualMachine` instance is obtained by invoking the `attach` method with a `VirtualMachineDescriptor` obtained from the list of virtual machine descriptors returned by the `list` method.

```java
public abstract class VirtualMachine {
  public static VirtualMachine attach(VirtualMachineDescriptor vmd) throws AttachNotSupportedException, IOException {
    // ...
  }
}
```

#### Use VM

##### Load Agent

Once a reference to a virtual machine is obtained, the `loadAgent`, `loadAgentLibrary`, and `loadAgentPath` methods are used to load agents into target virtual machine.

- **The `loadAgent` method is used to load agents that are written in the Java Language and deployed in a JAR file.**
- **The `loadAgentLibrary` and `loadAgentPath` methods are used to load agents that are deployed either in a dynamic library or statically linked into the VM and make use of the JVM Tools Interface.**

```java
public abstract class VirtualMachine {
  public void loadAgent(String agent) throws AgentLoadException, AgentInitializationException, IOException {
    loadAgent(agent, null);
  }

  public abstract void loadAgent(String agent, String options)
    throws AgentLoadException, AgentInitializationException, IOException;    
}
```

##### read properties

In addition to loading agents a `VirtualMachine` provides read access to the **system properties** in the target VM. This can be useful in some environments where properties such as `java.home`, `os.name`, or `os.arch` are used to construct the path to agent that will be loaded into the target VM.

```java
public abstract class VirtualMachine {
  public abstract Properties getSystemProperties() throws IOException;
  public abstract Properties getAgentProperties() throws IOException;
}
```

这两个方法的区别：**`getAgentProperties()`是vm为agent专门维护的属性**。

- `getSystemProperties()`: This method returns the system properties in the target virtual machine. Properties whose key or value is not a `String` are omitted. The method is approximately equivalent to the invocation of the method `System.getProperties()` in the target virtual machine except that properties with a key or value that is not a `String` are not included.
- `getAgentProperties()`: The target virtual machine can maintain a list of properties on behalf of agents. The manner in which this is done, the names of the properties, and the types of values that are allowed, is implementation specific. Agent properties are typically used to store communication end-points and other agent configuration details.

#### Detach VM

Detach from the virtual machine.

```java
public abstract class VirtualMachine {
  public abstract void detach() throws IOException;
}
```

#### 其他方法

<u>第一个是 `id()` 方法，它返回 target VM 的进程 ID 值</u>。

```java
public abstract class VirtualMachine {
  private final String id;

  public final String id() {
    return id;
  }
}
```

第二个是 `list()` 方法，它返回一组 `VirtualMachineDescriptor` 对象，描述所有潜在的target VM 对象。

```java
public abstract class VirtualMachine {
  public static List<VirtualMachineDescriptor> list() {
    // ...
  }
}
```

第三个是 `provider()` 方法，它返回一个 `AttachProvider` 对象。

```java
public abstract class VirtualMachine {
  private final AttachProvider provider;

  public final AttachProvider provider() {
    return provider;
  }
}
```

### 3.3 VirtualMachineDescriptor

**A `com.sun.tools.attach.VirtualMachineDescriptor` is a container class used to describe a Java virtual machine.**

```java
public class VirtualMachineDescriptor {
  private String id;
  private String displayName;
  private AttachProvider provider;

  public String id() {
    return id;
  }

  public String displayName() {
    return displayName;
  }

  public AttachProvider provider() {
    return provider;
  }    
}
```

- **an identifier** that identifies a target virtual machine.
- **a reference** to the `AttachProvider` that should be used when attempting to attach to the virtual machine.
- The **display name** is typically a human readable string that a tool might display to a user.

`VirtualMachineDescriptor` instances are typically created by invoking the `VirtualMachine.list()` method. This returns the complete list of descriptors to describe the Java virtual machines known to all installed attach providers.

### 3.4 AttachProvider

`com.sun.tools.attach.spi.AttachProvider` 是一个抽象类，它需要一个具体的实现：

```java
public abstract class AttachProvider {
  //...
}
```

不同平台上的JVM，它对应的具体 `AttachProvider` 实现是不一样的：

- Linux: `sun.tools.attach.LinuxAttachProvider`
- Windows: `sun.tools.attach.WindowsAttachProvider`

An attach provider implementation is typically tied to a Java virtual machine implementation, version, or even mode of operation. That is, **a specific provider implementation** will typically only be capable of attaching to **a specific Java virtual machine implementation or version**. For example, Sun’s JDK implementation ships with provider implementations that can only attach to Sun’s HotSpot virtual machine.

An attach provider is identified by its `name` and `type`:

- The `name` is typically, but not required to be, a name that corresponds to the VM vendor. The Sun JDK implementation, for example, ships with attach providers that use the name “sun”.
- The `type` typically corresponds to the attach mechanism. For example, an implementation that uses the Doors inter-process communication mechanism might use the type “doors”.

The purpose of the `name` and `type` is to identify providers in environments where there are multiple providers installed.

### 3.5 总结

本文内容总结如下：

- 第一点，Attach API位于`com.sun.tools.attach`包：
  - 在Java 8版本，`com.sun.tools.attach` 包位于 `JDK_HOME/lib/tools.jar` 文件。
  - 在Java 9版本之后，`com.sun.tools.attach` 包位于`jdk.attach` 模块（`JDK_HOME/jmods/jdk.attach.jmod`文件）。
- 第二点，在 `com.sun.tools.attach` 包当中，重要的类有三个： `VirtualMachine`（核心功能）、 `VirtualMachineDescriptor`（三个属性）和 `AttachProvider`（底层实现）。
- **第三点，使用`VirtualMachine`类分成三步：**
  - **第一步，与 target VM 建立连接，获得一个 `VirtualMachine` 对象。**
  - **第二步，使用 `VirtualMachine` 对象，可以将 Agent Jar 加载到 target VM 上，也可以从 target VM 读取一些属性信息。**
  - **第三步，与 target VM 断开连接**。

## 4. Dynamic: Attach API示例

### 4.1 VirtualMachine

#### attach and detach

```java
package sample;

import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException {
    // 注意：需要修改pid的值
    String pid = "1234";
    VirtualMachine vm = VirtualMachine.attach(pid);
    vm.detach();
  }
}
```

#### loadAgent

```java
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    Class<?> agentClass = DynamicAgent.class;
    System.out.println("Agent-Class: " + agentClass.getName());
    System.out.println("agentArgs: " + agentArgs);
    System.out.println("Instrumentation: " + inst.getClass().getName());
    System.out.println("ClassLoader: " + agentClass.getClassLoader());
    System.out.println("Thread Id: " + Thread.currentThread().getName() + "@" +
                       Thread.currentThread().getId() + "(" + Thread.currentThread().isDaemon() + ")"
                      );
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");
  }
}
```

```java
package sample;

import com.sun.tools.attach.AgentInitializationException;
import com.sun.tools.attach.AgentLoadException;
import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException,
  AgentLoadException, AgentInitializationException {
    // 注意：需要修改pid的值
    String pid = "1234";
    String agentPath = "D:\\git-repo\\learn-java-agent\\target\\TheAgent.jar";
    VirtualMachine vm = VirtualMachine.attach(pid);
    vm.loadAgent(agentPath, "Hello JVM Attach");
    vm.detach();
  }
}
```

#### getSystemProperties

```java
package sample;

import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;
import java.util.Properties;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException {
    // 注意：需要修改pid的值
    String pid = "1234";
    VirtualMachine vm = VirtualMachine.attach(pid);
    Properties properties = vm.getSystemProperties();
    properties.list(System.out);
    vm.detach();
  }
}
```

Output:

```shell
-- listing properties --
java.runtime.name=Java(TM) SE Runtime Environment
sun.boot.library.path=C:\Program Files\Java\jdk1.8.0_301\jr...
java.vm.version=25.301-b09
java.vm.vendor=Oracle Corporation
java.vendor.url=http://java.oracle.com/
path.separator=;
java.vm.name=Java HotSpot(TM) 64-Bit Server VM
...
```

#### getAgentProperties

```java
package sample;

import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;
import java.util.Properties;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException {
    // 注意：需要修改pid的值
    String pid = "1234";
    VirtualMachine vm = VirtualMachine.attach(pid);
    Properties properties = vm.getAgentProperties();
    properties.list(System.out);
    vm.detach();
  }
}
```

Output:

```shell
-- listing properties --
sun.jvm.args=-javaagent:C:\Program Files\JetBrains...
sun.jvm.flags=
sun.java.command=sample.Program
```

#### list

```java
import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import com.sun.tools.attach.spi.AttachProvider;

import java.io.IOException;
import java.util.List;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();

    for (VirtualMachineDescriptor vmd : list) {
      String id = vmd.id();
      String displayName = vmd.displayName();
      AttachProvider provider = vmd.provider();
      System.out.println("Id: " + id);
      System.out.println("Name: " + displayName);
      System.out.println("Provider: " + provider);
      System.out.println("=====================");
    }
  }
}
```

```java
package sample;

import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.io.IOException;
import java.util.List;
import java.util.Properties;

public class VMAttach {
  public static void main(String[] args) throws IOException, AttachNotSupportedException {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();

    String className = "sample.Program";
    VirtualMachine vm = null;
    for (VirtualMachineDescriptor item : list) {
      String displayName = item.displayName();
      if (displayName != null && displayName.equals(className)) {
        vm = VirtualMachine.attach(item);
        break;
      }
    }

    if (vm != null) {
      Properties properties = vm.getSystemProperties();
      properties.list(System.out);
      vm.detach();
    }
  }
}
```

### 4.2 AttachProvider

```java
import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.spi.AttachProvider;

public class VMAttach {
  public static void main(String[] args) throws Exception {
    // 注意：需要修改pid的值
    String pid = "1234";
    VirtualMachine vm = VirtualMachine.attach(pid);
    AttachProvider provider = vm.provider();
    String name = provider.name();
    String type = provider.type();
    System.out.println("Provider Name: " + name);
    System.out.println("Provider Type: " + type);
    System.out.println("Provider Impl: " + provider.getClass());
  }
}
```

Windows 7环境下输出：

```shell
Provider Name: sun
Provider Type: windows
Provider Impl: class sun.tools.attach.WindowsAttachProvider
```

Ubuntu 20环境下输出：

```shell
Provider Name: sun
Provider Type: socket
Provider Impl: class sun.tools.attach.LinuxAttachProvider
```

### 4.3 总结

本文主要是对Attach API的内容进行举例。

## 5. 总结

### 5.1 Command Line

进行 Load-Time Instrumentation，需要从命令行启动 Agent Jar，需要使用 `-javagent` 选项：

```shell
-javaagent:jarpath[=options]
```

![img](https://lsieun.github.io/assets/images/java/agent/java-agent-command-line-options.png)

在 Load-Time Instrumentation 过程中，会用到 `premain` 方法，我们关注两个问题：

- 第一个问题，`Instrumentation` 是一个接口，它的具体实现是哪个类？回答：`sun.instrument.InstrumentationImpl` 类。
- 第二个问题，是“谁”调用了 `premain()` 方法的呢？回答：`InstrumentationImpl.loadClassAndStartAgent()` 方法

```java
public static void premain(String agentArgs, Instrumentation inst)
```

### 5.2 Attach

进行 Dynmaic Instrumentation，需要用到 Attach API。

Attach API 体现在 `com.sun.tools.attach` 包。

在 `com.sun.tools.attach` 包，最核心的是 `VirtualMachine` 类。

```pseudocode
                                       ┌─── VirtualMachine.attach(String id)
                  ┌─── 1. Get VM ──────┤
                  │                    └─── VirtualMachine.attach(VirtualMachineDescriptor vmd)
                  │
                  │                                            ┌─── VirtualMachine.loadAgent(String agent)
                  │                    ┌─── Load Agent ────────┤
VirtualMachine ───┤                    │                       └─── VirtualMachine.loadAgent(String agent, String options)
                  ├─── 2. Use VM ──────┤
                  │                    │                       ┌─── VirtualMachine.getAgentProperties()
                  │                    └─── read properties ───┤
                  │                                            └─── VirtualMachine.getSystemProperties()
                  │
                  └─── 3. detach VM ───┼─── VirtualMachine.detach()
```

# 第三章 Instrumentation API

## 1. Instrumentation API

### 1.1 java.lang.instrument

首先，`java.lang.instrument`包的作用是什么？

- 它定义了一些“规范”，比如 Manifest 当中的 `Premain-Class` 和 `Agent-Class` 属性，再比如 `premain` 和 `agentmain` 方法，这些“规范”是 Agent Jar 必须遵守的。
- 它定义了一些类和接口，例如 `Instrumentation` 和 `ClassFileTransformer`，这些类和接口允许我们在 Agent Jar 当中实现修改某些类的字节码。

举个形象化的例子。这些“规范”让一个普通的 `.jar` 文件（普通士兵）成为 Agent Jar（禁卫军）； 接着，Agent Jar（禁卫军）就可以在 target JVM（紫禁城）当中对 Application 当中加载的类（平民、官员）进行 instrumentation（巡查守备）任务了。

> 作用

**The mechanism for instrumentation is modification of the byte-codes of methods.** [Link](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

```pseudocode
instrumentation = modification of the byte-codes of methods
```

其次，`java.lang.instrument` 是Java 1.5引入的。

> 时间

再者，`java.lang.instrument` 包在哪里？

- 在Java 8版本中，位于`rt.jar`文件
- 在Java 9版本之后，位于 `java.instrument` 模块（`JDK_HOME/jmods/java.instrument.jmod`）

> 位置

最后，`java.lang.instrument` 包有哪些类呢？

> 有哪些类

```pseudocode
                        ┌─── ClassDefinition (类)
                        │
                        ├─── ClassFileTransformer (接口)
                        │
java.lang.instrument ───┼─── Instrumentation (接口)
                        │
                        ├─── IllegalClassFormatException (异常)
                        │
                        └─── UnmodifiableClassException (异常)
```

其中，`IllegalClassFormatException` 类和 `UnmodifiableClassException` 类，都是 `Exception` 类的子类，它们表示了某些情况下无法实现的操作，不需要投入很多时间研究。

我们的关注点就是理解：

- `ClassDefinition` 类
- `ClassFileTransformer` 接口
- `Instrumentation` 接口

换句话说，理解了这三者，也就是理解了 `java.lang.instrument` 的精髓。

在 Agent Jar 当中，Agent Class 是“名义”上的“老大”，但真正的做事情是借助于 `Instrumentation` 对象去完成的。

打个比方，英国的女王是国家虚位元首，象征性的最高领导者，无实权，但真正上管理国家的是首相。

### 1.2 ClassDefinition

其中，`ClassDefinition` 类是一个非常简单的类，本质上就是对 `Class` 和 `byte[]` 两个字段的封装，很容易就能够掌握。

```java
package java.lang.instrument;

/**
 * This class serves as a parameter block to the <code>Instrumentation.redefineClasses</code> method.
 * Serves to bind the <code>Class</code> that needs redefining together with the new class file bytes.
 *
 * @see     java.lang.instrument.Instrumentation#redefineClasses
 * @since   1.5
 */
public final class ClassDefinition {
  /**
   *  The class to redefine
   */
  private final Class<?> mClass;
  /**
   *  The replacement class file bytes
   */
  private final byte[] mClassFile;
}
```

### 1.3 ClassFileTransformer

An agent provides an implementation of this interface in order to transform class files. The **transformation** occurs before the class is **defined** by the JVM.

#### transform

```java
public interface ClassFileTransformer {
  byte[] transform(ClassLoader         loader,
                   String              className,
                   Class<?>            classBeingRedefined,
                   ProtectionDomain    protectionDomain,
                   byte[]              classfileBuffer)
    throws IllegalClassFormatException;
}
```

Parameters:

- `loader` - the defining loader of the class to be transformed, may be `null` if the **bootstrap loader**
- `className` - the name of the class in the internal form of fully qualified class and interface names as defined in The Java Virtual Machine Specification. For example, “java/util/List”.
- `classBeingRedefined` - if this is triggered by a `redefine` or `retransform`, the class being redefined or retransformed; if this is a class load, `null`
- `protectionDomain` - the protection domain of the class being defined or redefined
- `classfileBuffer` - the input byte buffer in class file format - **must not be modified**

Returns:

- a well-formed class file buffer (the result of the transform), or **`null` if no transform is performed**.

在 `transform` 方法中，我们重点关注 `className` 和 `classfileBuffer` 两个接收参数，以及返回值，

小总结：

- `loader`: 如果值为 `null`，则表示 bootstrap loader。
- `className`: 表示 internal class name，例如 `java/util/List`。
- `classfileBuffer`: 一定不要修改它的原有内容，可以复制一份，在复制的基础上就进行修改。
- 返回值: 如果返回值为 `null`，则表示没有进行修改。

### 1.4 Instrumentation

> [JAVA进阶之Agent_setnativemethodprefix_purple.taro的博客-CSDN博客](https://blog.csdn.net/zxlyx/article/details/124120795)

在 `java.lang.instrument` 当中，`Instrumentation` 是一个接口：

```java
/**
 * This class provides services needed to instrument Java
 * programming language code.
 * Instrumentation is the addition of byte-codes to methods for the
 * purpose of gathering data to be utilized by tools.
 * Since the changes are purely additive, these tools do not modify
 * application state or behavior.
 * Examples of such benign tools include monitoring agents, profilers,
 * coverage analyzers, and event loggers.
 *
 * <P>
 * There are two ways to obtain an instance of the
 * <code>Instrumentation</code> interface:
 *
 * <ol>
 *   <li><p> When a JVM is launched in a way that indicates an agent
 *     class. In that case an <code>Instrumentation</code> instance
 *     is passed to the <code>premain</code> method of the agent class.
 *     </p></li>
 *   <li><p> When a JVM provides a mechanism to start agents sometime
 *     after the JVM is launched. In that case an <code>Instrumentation</code>
 *     instance is passed to the <code>agentmain</code> method of the
 *     agent code. </p> </li>
 * </ol>
 * <p>
 * These mechanisms are described in the
 * {@linkplain java.lang.instrument package specification}.
 * <p>
 * Once an agent acquires an <code>Instrumentation</code> instance,
 * the agent may call methods on the instance at any time.
 *
 * @apiNote This interface is not intended to be implemented outside of
 * the java.instrument module.
 *
 * @since   1.5
 */
public interface Instrumentation {
}
```

```
                                                         ┌─── isRedefineClassesSupported()
                                                         │
                                     ┌─── ability ───────┼─── isRetransformClassesSupported()
                                     │                   │
                   ┌─── Agent Jar ───┤                   └─── isNativeMethodPrefixSupported()
                   │                 │
                   │                 │                   ┌─── addTransformer()
                   │                 └─── transformer ───┤
                   │                                     └─── removeTransformer()
                   │
                   │                                     ┌─── appendToBootstrapClassLoaderSearch()
                   │                 ┌─── classloader ───┤
                   │                 │                   └─── appendToSystemClassLoaderSearch()
Instrumentation ───┤                 │
                   │                 │                                         ┌─── loading ───┼─── transform
                   │                 │                                         │
                   │                 │                   ┌─── status ──────────┤                                  ┌─── getAllLoadedClasses()
                   │                 │                   │                     │               ┌─── get ──────────┤
                   │                 │                   │                     │               │                  └─── getInitiatedClasses()
                   │                 │                   │                     └─── loaded ────┤
                   │                 │                   │                                     │                  ┌─── isModifiableClass()
                   │                 ├─── class ─────────┤                                     │                  │
                   └─── target VM ───┤                   │                                     └─── modifiable ───┼─── redefineClasses()
                                     │                   │                                                        │
                                     │                   │                                                        └─── retransformClasses()
                                     │                   │
                                     │                   │                     ┌─── isNativeMethodPrefixSupported()
                                     │                   └─── native method ───┤
                                     │                                         └─── setNativeMethodPrefix()
                                     │
                                     ├─── object ────────┼─── getObjectSize()
                                     │
                                     │                   ┌─── isModifiableModule()
                                     └─── module ────────┤
                                                         └─── redefineModule()
```

#### isXxxSupported

读取`META-INF/MANIFEST.MF`文件中的属性信息：

```java
public interface Instrumentation {
  boolean isRedefineClassesSupported();
  boolean isRetransformClassesSupported();
  boolean isNativeMethodPrefixSupported();
}
```

#### transform

##### xxxTransformer

添加和删除 `ClassFileTransformer`：

```java
public interface Instrumentation {
  void addTransformer(ClassFileTransformer transformer);
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
  boolean removeTransformer(ClassFileTransformer transformer);
}
```

同时，我们也说明一下三个类之间的关系：

```pseudocode
Agent Class --> Instrumentation --> ClassFileTransformer
```

三个类之间更详细的关系如下：

```pseudocode
               ┌─── premain(String agentArgs, Instrumentation inst)
Agent Class ───┤
               └─── agentmain(String agentArgs, Instrumentation inst)

                   ┌─── void addTransformer(ClassFileTransformer transformer, boolean canRetransform)
Instrumentation ───┤
                   └─── boolean removeTransformer(ClassFileTransformer transformer)
```

##### redefineClasses

```java
public interface Instrumentation {
  void redefineClasses(ClassDefinition... definitions)
    throws ClassNotFoundException, UnmodifiableClassException;
}
```

##### retransformClasses

```java
public interface Instrumentation {
  void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
}
```

#### loader + class + object

##### ClassLoaderSearch

```java
public interface Instrumentation {
  // 1.6
  void appendToSystemClassLoaderSearch(JarFile jarfile);
  // 1.6
  void appendToBootstrapClassLoaderSearch(JarFile jarfile);
}
```

##### xxxClasses

下面三个方法都与已经加载的 `Class` 相关：

```java
public interface Instrumentation {
  Class[] getAllLoadedClasses();
  Class[] getInitiatedClasses(ClassLoader loader);
  /**
   * Primitive classes (for example, java.lang.Integer.TYPE) and array classes are never modifiable.
   */
  boolean isModifiableClass(Class<?> theClass);
}
```

##### Object

```java
public interface Instrumentation {
  long getObjectSize(Object objectToSize);
}
```

#### native

```java
public interface Instrumentation {
  boolean isNativeMethodPrefixSupported();
  void setNativeMethodPrefix(ClassFileTransformer transformer, String prefix);
}
```

#### module

Java 9引入

```java
public interface Instrumentation {
  boolean isModifiableModule(Module module);
  void redefineModule (Module module,
                       Set<Module> extraReads,
                       Map<String, Set<Module>> extraExports,
                       Map<String, Set<Module>> extraOpens,
                       Set<Class<?>> extraUses,
                       Map<Class<?>, List<Class<?>>> extraProvides);
}
```

### 1.5 总结

本文内容总结如下：

- 第一点，理解 `java.lang.instrument` 包的主要作用：它让一个普通的 Jar 文件成为一个 Agent Jar。
- 第二点，在 `java.lang.instrument` 包当中，有三个重要的类型：`ClassDefinition`、`ClassFileTransformer` 和 `Instrumentation`。

## 2. Instrumentation.isXxxSupported()

### 2.1 isXxxSupported()

```java
public interface Instrumentation {
    boolean isRedefineClassesSupported();
    boolean isRetransformClassesSupported();
    boolean isNativeMethodPrefixSupported();
}
```

- `boolean isRedefineClassesSupported()`: Returns whether or not the current JVM configuration supports **redefinition of classes**.
  - The ability to redefine an already loaded class is an optional capability of a JVM.
  - Redefinition will only be supported if the `Can-Redefine-Classes` manifest attribute is set to `true` in the agent JAR file and the JVM supports this capability.
  - During a single instantiation of a single JVM, multiple calls to this method will always return the same answer.
- `boolean isRetransformClassesSupported()`: Returns whether or not the current JVM configuration supports **retransformation of classes**.
  - The ability to retransform an already loaded class is an optional capability of a JVM.
  - Retransformation will only be supported if the `Can-Retransform-Classes` manifest attribute is set to `true` in the agent JAR file and the JVM supports this capability.
  - During a single instantiation of a single JVM, multiple calls to this method will always return the same answer.
- `boolean isNativeMethodPrefixSupported()`: Returns whether the current JVM configuration supports **setting a native method prefix**.
  - The ability to set a native method prefix is an optional capability of a JVM.
  - Setting a native method prefix will only be supported if the `Can-Set-Native-Method-Prefix` manifest attribute is set to `true` in the agent JAR file and the JVM supports this capability.
  - During a single instantiation of a single JVM, multiple calls to this method will always return the same answer.

小总结：

- 第一，判断 JVM 是否支持该功能。
- 第二，判断 Java Agent Jar 内的 `MANIFEST.MF` 文件里的属性是否为 `true`。
- 第三，在一个 JVM 实例当中，多次调用某个 `isXxxSupported()` 方法，该方法的返回值是不会改变的。

### 2.2 示例

#### LoadTimeAgent.java

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
  }
}
```

#### 运行

在 `pom.xml` 文件中，`maven-jar-plugin` 处可以设置 `Can-Redefine-Classes`、`Can-Retransform-Classes` 和 `Can-Set-Native-Method-Prefix` 属性。

第一次测试时，将三个属性设置为 `true`：

```xml
<manifestEntries>
  <Premain-Class>lsieun.agent.LoadTimeAgent</Premain-Class>
  <Agent-Class>lsieun.agent.DynamicAgent</Agent-Class>
  <Can-Redefine-Classes>true</Can-Redefine-Classes>
  <Can-Retransform-Classes>true</Can-Retransform-Classes>
  <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
</manifestEntries>
```

第二次测试时，将三个属性设置为 `false`：

```xml
<manifestEntries>
  <Premain-Class>lsieun.agent.LoadTimeAgent</Premain-Class>
  <Agent-Class>lsieun.agent.DynamicAgent</Agent-Class>
  <Can-Redefine-Classes>false</Can-Redefine-Classes>
  <Can-Retransform-Classes>false</Can-Retransform-Classes>
  <Can-Set-Native-Method-Prefix>false</Can-Set-Native-Method-Prefix>
</manifestEntries>
```

每次测试之前，都需要重新生成 `.jar` 文件：

```shell
mvn clean package
```

第一次运行，将三个属性设置成 `true`，示例输出：

```
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
```

第二次运行，将三个属性设置成 `false`，示例输出：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
Can-Redefine-Classes: false
Can-Retransform-Classes: false
Can-Set-Native-Method-Prefix: false
```

### 2.3 总结

本文内容总结如下：

- 第一点，判断某一个`isXxxSupported()`方法是否为`true`，需要考虑两个因素：
  - 判断 JVM 是否支持该功能。
  - 判断 Agent.jar 内的 `MANIFEST.MF` 文件里的属性是否为 `true`。
- 第二点，在一个 JVM 实例当中，多次调用某个 `isXxxSupported()` 方法，该方法的返回值是不会改变的。

## 3. Instrumentation.xxxTransformer()

在本文当中，我们关注两个问题：

- `Instrumentation`和`ClassFileTransformer`两者是如何建立联系？
- `ClassFileTransformer.transform()`方法什么时候调用呢？

### 3.1 添加和删除Transformer

```java
public interface Instrumentation {
  void addTransformer(ClassFileTransformer transformer);
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
  boolean removeTransformer(ClassFileTransformer transformer);
}
```

#### addTransformer

在`Instrumentation`当中，定义了两个`addTransformer`方法，但两者本质上一样的：

- 调用`addTransformer(ClassFileTransformer transformer)`方法，相当于调用`addTransformer(transformer, false)`

```java
public interface Instrumentation {
  // Since 1.5
  void addTransformer(ClassFileTransformer transformer);
  // Since 1.6
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
}
```

那么，`addTransformer`方法的`canRetransform`参数起到一个什么样的作用呢？

- 第一点，它影响`transformer`对象**存储的位置**
- 第二点，它影响`transformer`对象**功能的发挥**

关于`transformer`对象存储的位置，我们可以参考`sun.instrument.InstrumentationImpl`源码当中的实现：

```java
public class InstrumentationImpl implements Instrumentation {
  private final TransformerManager mTransformerManager;
  private TransformerManager mRetransfomableTransformerManager;

  // 以下是经过简化之后的代码
  public synchronized void addTransformer(ClassFileTransformer transformer, boolean canRetransform) {
    if (canRetransform) {
      mRetransfomableTransformerManager.addTransformer(transformer);
    }
    else {
      mTransformerManager.addTransformer(transformer);
    }
  }    
}
```

- 如果`canRetransform`的值为`true`，我们就将`transformer`对象称为retransformation capable transformer
- 如果`canRetransform`的值为`false`，我们就将`transformer`对象称为retransformation incapable transformer

小总结：

- 第一点，两个`addTransformer`方法两者本质上是一样的。
- 第二点，第二个参数`canRetransform`影响第一个参数`transformer`的存储位置。

#### removeTransformer

不管是retransformation capable transformer，还是retransformation incapable transformer，都使用同一个`removeTransformer`方法：

```java
public interface Instrumentation {
  boolean removeTransformer(ClassFileTransformer transformer);
}
```

同样，我们可以参考`InstrumentationImpl`类当中的实现：

```java
public class InstrumentationImpl implements Instrumentation {
  private final TransformerManager mTransformerManager;
  private TransformerManager mRetransfomableTransformerManager;

  // 以下是经过简化之后的代码
  public synchronized boolean removeTransformer(ClassFileTransformer transformer) {
    TransformerManager mgr = findTransformerManager(transformer);
    if (mgr != null) {
      mgr.removeTransformer(transformer);
      return true;
    }
    return false;
  }  
}
```

在什么时候对`removeTransformer`方法进行调用呢？有两种情况。

第一种情况，**想处理的`Class`很明确，那就尽量早的调用`removeTransformer`方法，让`ClassFileTransformer`影响的范围最小化**。

```java
public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    System.out.println("Agent-Class: " + DynamicAgent.class.getName());
    ClassFileTransformer transformer = new ASMTransformer();
    try {
      inst.addTransformer(transformer, true);
      Class<?> targetClass = Class.forName("sample.HelloWorld");
      inst.retransformClasses(targetClass);
    } catch (Exception ex) {
      ex.printStackTrace();
    }
    finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

第二种情况，想处理的`Class`不明确，可以不调用`removeTransformer`方法。

```java
public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    ClassFileTransformer transformer = new InfoTransformer();
    inst.addTransformer(transformer);
  }
}
```

### 3.2 调用的时机

当我们将`ClassFileTransformer`添加到`Instrumentation`之后，`ClassFileTransformer`类当中的`transform`方法什么时候执行呢？

```java
public interface ClassFileTransformer {
  byte[] transform(ClassLoader         loader,
                   String              className,
                   Class<?>            classBeingRedefined,
                   ProtectionDomain    protectionDomain,
                   byte[]              classfileBuffer)
    throws IllegalClassFormatException;
}
```

**首先，对`ClassFileTransformer.transform()`方法调用的时机有三个：**

- **类加载的时候**
- **调用`Instrumentation.redefineClasses`方法的时候**
- **调用`Instrumentation.retransformClasses`方法的时候**

在OpenJDK的源码中，`hotspot/src/share/vm/prims/jvmtiThreadState.hpp`文件定义了一个`JvmtiClassLoadKind`结构：

```c++
enum JvmtiClassLoadKind {
  jvmti_class_load_kind_load = 100,
  jvmti_class_load_kind_retransform,
  jvmti_class_load_kind_redefine
};
```

**接着，来介绍一下redefine和retransform两个概念，它们与类的加载状态有关系：**

- **对于正在加载的类进行修改，它属于define和transform的范围。**
- **对于已经加载的类进行修改，它属于redefine和retransform的范围。**

对于已经加载的类（loaded class），<u>redefine侧重于以“新”换“旧”，而retransform侧重于对“旧”的事物进行“修补”</u>。

```
                               ┌─── define: ClassLoader.defineClass
               ┌─── loading ───┤
               │               └─── transform
class state ───┤
               │               ┌─── redefine: Instrumentation.redefineClasses
               └─── loaded ────┤
                               └─── retransform: Instrumentation.retransformClasses
```

再者，触发的方式不同：

- **load，是类在加载的过程当中，JVM内部机制来自动触发。**
- **redefine和retransform，是我们自己写代码触发**。

最后，就是不同的时机（load、redefine、retransform）能够接触到的transformer也不相同：

![img](https://lsieun.github.io/assets/images/java/agent/define-redefine-retransform.png)

### 3.3 总结

本文内容总结如下：

- 第一点，介绍了`Instrumentation`添加和移除`ClassFileTransformer`的两个方法。
- 第二点，介绍了`ClassFileTransformer`被调用的三个时机：load、redefine和retransform。

> [java - Difference between redefine and retransform in javaagent - Stack Overflow](https://stackoverflow.com/questions/19009583/difference-between-redefine-and-retransform-in-javaagent)

## 4. Instrumentation.redefineClasses()

### 4.1 redefine

#### redefineClasses

Redefine the supplied set of classes using the supplied class files.

```java
public interface Instrumentation {
  void redefineClasses(ClassDefinition... definitions)
    throws ClassNotFoundException, UnmodifiableClassException;
}
```

This method operates on a set in order to allow interdependent changes to **more than one class** at the same time (a redefinition of class A can require a redefinition of class B).

#### ClassDefinition

##### class info

```java
public final class ClassDefinition {
}
```

##### fields

```java
public final class ClassDefinition {
  private final Class<?> mClass;
  private final byte[] mClassFile;
}
```

##### constructor

```java
public final class ClassDefinition {
  public ClassDefinition(Class<?> theClass, byte[] theClassFile) {
    if (theClass == null || theClassFile == null) {
      throw new NullPointerException();
    }
    mClass      = theClass;
    mClassFile  = theClassFile;
  }
}
```

##### methods

```java
public final class ClassDefinition {
  public Class<?> getDefinitionClass() {
    return mClass;
  }

  public byte[] getDefinitionClassFile() {
    return mClassFile;
  }
}
```

### 4.2 示例一：替换Object类

#### StaticInstrumentation

在 `StaticInstrumentation` 类当中，主要是对 `java.lang.Object` 类的 `byte[]` 内容进行修改：让 `toString()` 方法返回 `This is an object.` 字符串。

修改前：

```java
public class Object {
  public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
  }
}
```

修改后：

```java
public class Object {
  public String toString() {
    return "This is an object.";
  }  
}
```

将修改后 `byte[]` 内容保存到工作目录下的 `target/classes/data/java/lang/Object.class` 文件中。

```java
import lsieun.asm.visitor.*;
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

import java.io.File;

public class StaticInstrumentation {
  public static void main(String[] args) {
    Class<?> clazz = Object.class;
    String user_dir = System.getProperty("user.dir");
    String filepath = user_dir + File.separator +
      "target" + File.separator +
      "classes" + File.separator +
      "data" + File.separator +
      clazz.getName().replace(".", "/") + ".class";
    filepath = filepath.replace(File.separator, "/");

    byte[] bytes = dump(clazz);
    FileUtils.writeBytes(filepath, bytes);
    System.out.println("file:///" + filepath);
  }

  public static byte[] dump(Class<?> clazz) {
    String className = clazz.getName();
    byte[] bytes = FileUtils.readClassBytes(className);

    ClassReader cr = new ClassReader(bytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    ClassVisitor cv = new ToStringVisitor(cw, "This is an object.");

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);
    return cw.toByteArray();
  }
}
```

#### Application

```java
package sample;

public class Program {
  public static void main(String[] args) {
    Object obj = new Object();
    System.out.println(obj);
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

import lsieun.utils.*;

import java.io.InputStream;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，redefine
    try {
      Class<?> clazz = Object.class;
      if (inst.isModifiableClass(clazz)) {
        InputStream in = LoadTimeAgent.class.getResourceAsStream("/data/java/lang/Object.class");
        int available = in.available();
        byte[] bytes = new byte[available];
        in.read(bytes);
        ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
        inst.redefineClasses(classDefinition);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

#### 运行

第一次运行，直接运行 `sample.Program` 类：

```shell
$ java -cp ./target/classes/ sample.Program
java.lang.Object@15db9742
```

第二次运行，加载 `TheAgent.jar` 运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1704856573
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

This is an object.
```

第三次运行，在 `pom.xml` 文件中，将 `Can-Redefine-Classes` 设置成 `false`：

```shell
<Can-Redefine-Classes>false</Can-Redefine-Classes>
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.country=US
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1704856573
    (4) Can-Redefine-Classes: false
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

java.lang.UnsupportedOperationException: redefineClasses is not supported in this environment
        at sun.instrument.InstrumentationImpl.redefineClasses(InstrumentationImpl.java:156)
        at lsieun.agent.LoadTimeAgent.premain(LoadTimeAgent.java:23)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
java.lang.Object@70dea4e
```

### 4.3 示例二：Hot Swap

#### Application

##### Program.java

```java
package sample;

public class Program {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    for (int i = 1; i < 20; i++) {
      instance.test(12, 3);
      System.out.println("intValue: " + HelloWorld.intValue);
    }
  }
}
```

##### HelloWorld.java

```java
package sample;

public class HelloWorld {
  public static int intValue = 20;

  public void test(int a, int b) {
    System.out.println("a = " + a);
    System.out.println("b = " + b);
    try {
      Thread.sleep(5000);
    } catch (Exception ignored) {
    }
    int c = a * b;
    System.out.println("a * b = " + c);
    System.out.println("============");
    System.out.println();
  }
}
```

#### Agent Jar

##### Agent Class

```java
import lsieun.thread.HotSwapThread;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    Class<?> agentClass = LoadTimeAgent.class;
    System.out.println("===>Premain-Class: " + agentClass.getName());
    System.out.println("ClassLoader: " + agentClass.getClassLoader());
    System.out.println("Thread Id: " + Thread.currentThread().getName() + "@" + Thread.currentThread().getId());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");

    Thread t = new HotSwapThread("hot-swap-thread", inst);
    t.setDaemon(true);
    t.start();
  }
}
```

##### HotSwapThread

```java
import java.io.InputStream;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;
import java.nio.file.*;
import java.util.List;

import static java.nio.file.StandardWatchEventKinds.ENTRY_MODIFY;

public class HotSwapThread extends Thread {
  private final Instrumentation inst;

  public HotSwapThread(String name, Instrumentation inst) {
    super(name);
    this.inst = inst;
  }

  @Override
  public void run() {
    try {
      FileSystem fs = FileSystems.getDefault();
      WatchService watchService = fs.newWatchService();
      // 注意：修改这里的路径信息
      Path watchPath = fs.getPath("D:\\git-repo\\learn-java-agent\\target\\classes\\sample\\");
      watchPath.register(watchService, ENTRY_MODIFY);
      WatchKey changeKey;
      while ((changeKey = watchService.take()) != null) {
        // Prevent receiving two separate ENTRY_MODIFY events: file modified and timestamp updated.
        // Instead, receive one ENTRY_MODIFY event with two counts.
        Thread.sleep( 50 );

        System.out.println("Thread Id: ===>" + Thread.currentThread().getName() + "@" + Thread.currentThread().getId());
        List<WatchEvent<?>> watchEvents = changeKey.pollEvents();
        for (WatchEvent<?> watchEvent : watchEvents) {
          // Ours are all Path type events:
          WatchEvent<Path> pathEvent = (WatchEvent<Path>) watchEvent;

          Path path = pathEvent.context();
          WatchEvent.Kind<Path> eventKind = pathEvent.kind();
          System.out.println(eventKind + "(" + pathEvent.count() +")" + " for path: " + path);
          String filepath = path.toFile().getCanonicalPath();
          if (!filepath.endsWith("HelloWorld.class")) continue;

          Class<?> clazz = Class.forName("sample.HelloWorld");
          if (inst.isModifiableClass(clazz)) {
            System.out.println("Before Redefine");
            InputStream in = clazz.getResourceAsStream("HelloWorld.class");
            int available = in.available();
            byte[] bytes = new byte[available];
            in.read(bytes);
            ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
            inst.redefineClasses(classDefinition);
            System.out.println("After Redefine");
          }

        }
        changeKey.reset(); // Important!
        System.out.println("Thread Id: <===" + Thread.currentThread().getName() + "@" + Thread.currentThread().getId());
      }
    } catch (Exception ex) {
      ex.printStackTrace();
    }
  }
}
```

#### Run

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

===>Premain-Class: lsieun.agent.LoadTimeAgent
ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
Thread Id: main@1
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
a = 12
b = 3
a * b = 36
============

a = 12
b = 3
Thread Id: ===>hot-swap-thread@6
ENTRY_MODIFY(1) for path: HelloWorld.class
Before Redefine
After Redefine
Thread Id: <===hot-swap-thread@6
a * b = 36                                               // 注意：这里仍然执行乘法操作
============

a = 12
b = 3
a + b = 15
============

a = 12
b = 3
a + b = 15
============
```

在 `Instrumentation.redefineClasses` 方法的API中描述到： <u>If a redefined method has **active stack frames**, those active frames continue to run the bytecodes of the original method. The redefined method will be used on new invokes.</u>

### 4.4 细节之处

- 第一点，`redefineClasses()` 方法是对已经加载的类进行以“新”换“旧”操作。
- 第二点，**如果某个方法正在执行（active stack frames），修改之后的方法会在下一次执行**。
- 第三点，**静态初始化（class initialization）不会再次执行，不受 `redefineClasses()` 方法的影响**。
- 第四点，`redefineClasses()` 方法的功能是有限的，主要集中在对方法体（method body）的修改。
- 第五点，当`redefineClasses()` 方法出现异常的时候，就相当于“什么都没有发生过”，不会对类产生影响。

#### fix-and-continue

This method is used to **replace** the definition of a class without reference to **the existing class file bytes**, as one might do when recompiling from source for **fix-and-continue** debugging. Where **the existing class file bytes** are to be transformed (for example in bytecode instrumentation) `retransformClasses` should be used.

#### active stack frames

If a redefined method has **active stack frames**, those active frames continue to run the bytecodes of the original method. The redefined method will be used on new invokes.

#### initialization

This method does not cause any initialization except that which would occur under the customary JVM semantics. In other words, redefining a class does not cause its initializers to be run. The values of static variables will remain as they were prior to the call.

#### restrictions

<u>The redefinition may change method bodies, the constant pool and attributes.</u>

<u>The redefinition must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance. These restrictions maybe be lifted in future versions.</u>

#### exception

The class file bytes are not checked, verified and installed until after the transformations have been applied, if the resultant bytes are in error this method will throw an exception.

If this method throws an exception, no classes have been redefined.

### 4.5 总结

本文内容总结如下：

- 第一点，`redefineClasses()` 方法可以对Class进行重新定义。
- 第二点，`redefineClasses()` 方法的一个使用场景就是fix-and-continue。
- 第三点，使用`redefineClasses()` 方法需要注意一些细节。

## 5. Instrumentation.retransformClasses()

### 5.1 retransformClasses

Retransform the supplied set of classes.

```java
public interface Instrumentation {
  void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
}
```

This method operates on a set in order to allow interdependent changes to **more than one class** at the same time (a retransformation of class A can require a retransformation of class B).

### 5.2 示例一：修改toString方法

#### Application

```java
package sample;

public class Program {
  public static void main(String[] args) {
    Object obj = new Object();
    System.out.println(obj);
  }
}
```

#### Agent Jar

##### LoadTimeAgent

调用顺序：

- create a transformer
- add the transformer
- call retransform
- remove the transformer

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "java.lang.Object";

    // 第三步，使用inst：添加transformer --> retransform --> 移除transformer
    ClassFileTransformer transformer = new ASMTransformer(className);
    inst.addTransformer(transformer, true);
    try {
      Class<?> clazz = Class.forName(className);
      boolean isModifiable = inst.isModifiableClass(clazz);
      if (isModifiable) {
        inst.retransformClasses(clazz);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
    finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

##### ASMTransformer

```java
package lsieun.instrument;

import lsieun.asm.visitor.*;
import org.objectweb.asm.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Objects;

public class ASMTransformer implements ClassFileTransformer {
  private final String internalName;

  public ASMTransformer(String internalName) {
    Objects.requireNonNull(internalName);
    this.internalName = internalName.replace(".", "/");
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className.equals(internalName)) {
      System.out.println("transform class: " + className);
      ClassReader cr = new ClassReader(classfileBuffer);
      ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
      ClassVisitor cv = new ToStringVisitor(cw, "This is an object.");

      int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
      cr.accept(cv, parsingOptions);

      return cw.toByteArray();
    }

    return null;
  }
}
```

#### Run

```shell
mvn clean package
```

##### None

```shell
$ java -cp ./target/classes/ sample.Program
java.lang.Object@15db9742
```

##### Load-Time

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

This is an object.
```

##### addTransformer: false

将 `Instrumentation.addTransformer(ClassFileTransformer, boolean)` 的第二个参数设置为 `false`：

```java
inst.addTransformer(transformer, false);
```

那么，再次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

java.lang.Object@7d4991ad
```

##### Can-Retransform: false

在 `pom.xml` 文件中，将 `Can-Retransform-Classes` 设置成 `false`：

```xml
<Can-Retransform-Classes>false</Can-Retransform-Classes>
```

再次运行，会出现 `UnsupportedOperationException` 异常：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

Caused by: java.lang.UnsupportedOperationException: adding retransformable transformers is not supported in this environment
        at sun.instrument.InstrumentationImpl.addTransformer(InstrumentationImpl.java:88)
        at lsieun.agent.LoadTimeAgent.premain(LoadTimeAgent.java:20)
        ... 6 more
FATAL ERROR in native method: processing of -javaagent failed
```

### 5.3 示例二：Dump

本示例的目的是将 JVM 当中已经加载的类导出。

#### Agent Jar

##### LoadTimeAgent

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要处理的类
    String className = "java.lang.Object";

    // 第三步，使用inst：添加transformer --> retransform --> 移除transformer
    ClassFileTransformer transformer = new DumpTransformer(className);
    inst.addTransformer(transformer, true);
    try {
      Class<?> clazz = Class.forName(className);
      boolean isModifiable = inst.isModifiableClass(clazz);
      if (isModifiable) {
        inst.retransformClasses(clazz);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
    finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

##### DumpTransformer

```java
package lsieun.instrument;

import lsieun.utils.DateUtils;
import lsieun.utils.DumpUtils;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Objects;

public class DumpTransformer implements ClassFileTransformer {
  private final String internalName;

  public DumpTransformer(String internalName) {
    Objects.requireNonNull(internalName);
    this.internalName = internalName.replace(".", "/");
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className.equals(internalName)) {
      String timeStamp = DateUtils.getTimeStamp();
      String filename = className.replace("/", ".") + "." + timeStamp + ".class";
      DumpUtils.dump(filename, classfileBuffer);
    }
    return null;
  }
}
```

#### Run

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

file:///D:\git-repo\learn-java-agent\dump\java.lang.Object.2022.01.28.10.00.01.768.class
```

### 5.4 示例三：Dump（Regex）

本示例的目的是使用正则表达式（Regular Expression）将 JVM 当中已经加载的一些类导出。

#### Agent Jar

##### DynamicAgent

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.util.ArrayList;
import java.util.List;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，设置正则表达式：agentArgs
    RegexUtils.setPattern(agentArgs);

    // 第三步，使用inst：进行re-transform操作
    ClassFileTransformer transformer = new DumpTransformer();
    inst.addTransformer(transformer, true);
    try {
      Class<?>[] classes = inst.getAllLoadedClasses();
      List<Class<?>> candidates = new ArrayList<>();
      for (Class<?> c : classes) {
        String className = c.getName();

        // 这些if判断的目的是：不考虑JDK自带的类
        if (className.startsWith("java")) continue;
        if (className.startsWith("javax")) continue;
        if (className.startsWith("jdk")) continue;
        if (className.startsWith("sun")) continue;
        if (className.startsWith("com.sun")) continue;
        if (className.startsWith("[")) continue;

        boolean isModifiable = inst.isModifiableClass(c);
        boolean isCandidate = RegexUtils.isCandidate(className);

        System.out.println("Loaded Class: " + className + " - " + isModifiable + ", " + isCandidate);
        if (isModifiable && isCandidate) {
          candidates.add(c);
        }
      }

      System.out.println("candidates size: " + candidates.size());
      if (!candidates.isEmpty()) {
        inst.retransformClasses(candidates.toArray(new Class[0]));
      }
    } catch (Exception ex) {
      ex.printStackTrace();
    } finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

##### DumpTransformer

```java
package lsieun.instrument;

import lsieun.utils.DateUtils;
import lsieun.utils.DumpUtils;
import lsieun.utils.RegexUtils;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class DumpTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (RegexUtils.isCandidate(className)) {
      String timeStamp = DateUtils.getTimeStamp();
      String filename = className.replace("/", ".") + "." + timeStamp + ".class";
      DumpUtils.dump(filename, classfileBuffer);
    }
    return null;
  }
}
```

#### Run

在 `run.instrument.DynamicInstrumentation` 类中，传入参数：

```java
vm.loadAgent(agent, "sample\\..*");
file:///D:\git-repo\learn-java-agent\dump\sample.HelloWorld.2022.01.28.09.45.51.218.class
file:///D:\git-repo\learn-java-agent\dump\sample.Program.2022.01.28.09.45.51.223.class
```

### 5.5 注意事项

- 第一点，**`retransformClasses()` 方法是针对已经加载的类（already loaded classes）**。
- 第二点，**如果某个方法正在执行（active stack frames），修改之后的方法会在下一次执行**。
- 第三点，**静态初始化（class initialization）不会再次执行，不受`retransformClasses()`方法的影响**。
- 第四点，`retransformClasses()` 方法的功能是有限的，主要集中在对方法体（method body）的修改。
- 第五点，当`retransformClasses()` 方法出现异常的时候，就相当于“什么都没有发生过”，不会对类产生影响。

This function facilitates the instrumentation of **already loaded classes**.

#### active stack frames

If a retransformed method has active stack frames, those active frames continue to run the bytecodes of the original method. The retransformed method will be used on new invokes.

#### initialization

This method does not cause any initialization except that which would occur under the customary JVM semantics. In other words, redefining a class does not cause its initializers to be run. The values of static variables will remain as they were prior to the call.

Instances of the retransformed class are not affected.

#### restrictions

The retransformation may change method bodies, the constant pool and attributes.

The retransformation must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance. These restrictions maybe be lifted in future versions.

#### exception

The class file bytes are not checked, verified and installed until after the transformations have been applied, if the resultant bytes are in error this method will throw an exception.

If this method throws an exception, no classes have been retransformed.

### 5.6 总结

本文内容总结如下：

- 第一点，**`retransformClasses()` 方法的主要作用是针对已经加载的类（already loaded classes）进行转换**。
- 第二点，`retransformClasses()` 方法的一个特殊用途是将加载类的字节码进行导出。
- 第三点，在使用`retransformClasses()` 方法的过程中，需要注意一些细节内容。

## 6. redefine VS. retransform

> - [StackOverflow: Difference between redefine and retransform in javaagent](https://stackoverflow.com/questions/19009583/difference-between-redefine-and-retransform-in-javaagent)

**redefine 和 retransform 的共同点之处：两者都是对已经加载的类（already loaded classes）进行修改。**

### 6.1 时间不同

在 `Instrumentation` 类版本演进的过程中，先有 redefine，后有 retransform。

在 Java 1.5 的时候（2004.09），retransform 的概念还没有出现：

```java
public interface Instrumentation {
  // Since 1.5
  boolean isRedefineClassesSupported();
  // Since 1.5
  void addTransformer(ClassFileTransformer transformer);
  // Since 1.5
  void redefineClasses(ClassDefinition... definitions) throws ClassNotFoundException, UnmodifiableClassException;
}
```

到 Java 1.6 的时候（2006.12），才有了 retransform 相关的内容：

```java
public interface Instrumentation {
  // Since 1.6
  boolean isRetransformClassesSupported();
  // Since 1.6
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
  // Since 1.6
  void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;
}
```

### 6.2 处理方式不同：替换和修改

- redefine 是进行“替换”
- retransform 是进行“修改”

The main difference seems to be that when we `redefine` a class, we supply a `byte[]` with the new definition out of the blue, whereas when we `retransform`, we get a `byte[]` containing the current definition via the same API, and we return a modified `byte[]`.

Therefore, to `redefine`, we need to know more about the class. With `retransform` you can do that more directly: just look at the bytecode given, modify it, and return it.

### 6.3 影响范围不同：transformer

redefine 操作会触发：

- retransformation incapable transformer
- retransformation capable transformer

retransform 操作会触发：

- retransformation capable transformer

![img](https://lsieun.github.io/assets/images/java/agent/define-redefine-retransform.png)

### 6.4 总结

本文内容总结如下：

- 第一点，redefine 和 retransform 的共同之处：两者都是处理已经加载的类（already loaded classes）。
- 第二点，redefine 和 retransform 的不同之处：出现时间不同、处理方式不同、影响范围不同。

## 7. Instrumentation.xxxClasses()

### 7.1 xxxClasses()

```java
public interface Instrumentation {
    Class[] getAllLoadedClasses();
    Class[] getInitiatedClasses(ClassLoader loader);
    boolean isModifiableClass(Class<?> theClass);
}
```

- `Class[] getAllLoadedClasses()`: Returns an array containing all the classes loaded by the JVM, zero-length if there are none.
- `Class[] getInitiatedClasses(ClassLoader loader)`: Returns an array of all classes for which `loader` is **an initiating loader**. If the supplied loader is `null`, classes initiated by the **bootstrap class loader** are returned.
- `boolean isModifiableClass(Class<?> theClass)`: Determines whether a class is modifiable by **retransformation or redefinition**. If a class is modifiable then this method returns `true`. If a class is not modifiable then this method returns `false`.
  - For a class to be **retransformed**, `isRetransformClassesSupported` must also be `true`. But the value of `isRetransformClassesSupported()` does not influence the value returned by this function.
  - For a class to be **redefined**, `isRedefineClassesSupported` must also be `true`. But the value of `isRedefineClassesSupported()` does not influence the value returned by this function.
  - **Primitive classes** (for example, `java.lang.Integer.TYPE`) and **array classes** are never modifiable.

这三个方法都与`java.lang.Class`相关，要么是方法的返回值，要么是方法的参数。

<u>第一个方法，`Class[] getAllLoadedClasses()`的作用就是获取**所有已经加载的类**；功能虽然强大，但是要慎重使用，因为它花费的时间也比较多。</u>

有的时候，我们要找到某个类，就想调用`getAllLoadedClasses()`方法，然后遍历查找，这样的执行效率会比较低。**如果我们明确的知道要找某个类，可以直接使用`Class.forName()`方法**。

第二个方法，`Class[] getInitiatedClasses(ClassLoader loader)`的作用是获取由某一个initiating class loader已经加载的类。

什么是initiating class loader和defining class loader呢？

In Java terminology, a class loader that is asked to load a type, but returns a type loaded by some other class loader, is called an **initiating class loader** of that type. The class loader that actually defines the type is called the **defining class loader** for the type.

第三个方法，`boolean isModifiableClass(Class<?> theClass)`的作用是判断某一个Class是否可以被修改（modifiable）。

**要对一个已经加载的类进行修改，需要考虑四个因素：**

- **第一，JVM是否支持？**
- **第二，Agent Jar是否支持？在`MANIFEST.MF`文件中，是否将`Can-Redefine-Classes`和`Can-Retransform-Classes`设置为`true`？**
- **第三，`Instrumentation`和`ClassFileTransformer`是否支持？是否将`addTransformer(ClassFileTransformer transformer, boolean canRetransform)`的`canRetransform`参数设置为`true`？**
- **第四，当前的Class是否为可修改的？`boolean isModifiableClass(Class<?> theClass)`是否返回`true`？**

<u>需要注意的是，**Primitive classes**和**array classes**是不能被修改的</u>。

### 7.2 示例一：All Loaded Class和Modifiable

#### LoadTimeAgent.java

```java
package lsieun.agent;

import lsieun.utils.PrintUtils;

import java.lang.instrument.Instrumentation;
import java.util.ArrayList;
import java.util.List;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，获取所有加载的类
    Class<?>[] allLoadedClasses = inst.getAllLoadedClasses();

    // 第三步，分成两组
    List<String> modifiableClassList = new ArrayList<>();
    List<String> notModifiableClassList = new ArrayList<>();
    for (Class<?> clazz : allLoadedClasses) {
      boolean isModifiable = inst.isModifiableClass(clazz);
      String className = clazz.getName();
      if (isModifiable) {
        modifiableClassList.add(className);
      }
      else {
        notModifiableClassList.add(className);
      }
    }

    // 第四步，输出
    System.out.println("Modifiable Classes:");
    for (String item : modifiableClassList) {
      System.out.println("    " + item);
    }
    System.out.println("Not Modifiable Classes:");
    for (String item : notModifiableClassList) {
      System.out.println("    " + item);
    }
  }
}
```

#### 运行

在使用`Instrumentation.isModifiableClass(Class<?> theClass)`判断时， 除了**原始类型**和**数组类型**（例如，`[Ljava.lang.Object;`和`[B`等）返回`false`值；其它的类型（例如，`java.lang.Object`）都会返回`true`值。 这也就意味着我们可以对大部分的Class进行retransform和redefine操作。

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1704856573
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

Modifiable Classes:
    lsieun.utils.PrintUtils    // 自己写的类，是可以被修改的
    java.lang.Object           // JDK内部的类，也是可以被修改的
    ...
Not Modifiable Classes:
    ...
    [[C
    [Ljava.lang.Object;
    [Ljava.lang.Long;          // 数组类型，是不可以被修改的
    [J
    [I
    [S
    [B
    [D
    [F
    [C
    [Z
```

### 7.3 示例二：initiating class loader

这个示例主要是为了验证一下：System ClassLoader是不是`java.lang.StrictMath`类的initiating class loader？

```java
package lsieun.agent;

import lsieun.utils.PrintUtils;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，加载一个java.lang.StrictMath类
    {
      Class<StrictMath> clazz = StrictMath.class;
      System.out.println("Load Class: " + clazz.getName());
      System.out.println("==============================");
    }

    // 第三步，查看加载的类
    ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
    Class<?>[] initiatedClasses = inst.getInitiatedClasses(systemClassLoader);
    if (initiatedClasses != null) {
      for (Class<?> clazz : initiatedClasses) {
        String message = String.format("%s: %s", clazz.getName(), clazz.getClassLoader());
        System.out.println(message);
      }
    }
  }
}
```

### 7.4 示例三：尝试修改 int.class

本示例目的：当我们尝试修改不能修改的类时，会出现`UnmodifiableClassException`异常。

```java
package lsieun.agent;

import lsieun.utils.*;

import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，使用inst
    Class<?> clazz = int.class; // 或 String[].class;
    boolean isModifiable = inst.isModifiableClass(clazz);
    System.out.println(isModifiable);
    inst.retransformClasses(clazz);
  }
}
```

输出结果：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

false
Exception in thread "main" java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
Caused by: java.lang.instrument.UnmodifiableClassException
        at sun.instrument.InstrumentationImpl.retransformClasses0(Native Method)
        at sun.instrument.InstrumentationImpl.retransformClasses(InstrumentationImpl.java:144)
        at lsieun.agent.LoadTimeAgent.premain(LoadTimeAgent.java:17)
        ... 6 more
FATAL ERROR in native method: processing of -javaagent failed
```

### 7.5 总结

本文内容总结如下：

- 第一点，`getAllLoadedClasses()`方法是获取所有已经加载的类；功能虽然强大，但是要慎重使用。如果知道类的名字，使用`Class.forName()`是更好的选择。
- 第二点，`getInitiatedClasses(ClassLoader loader)`方法是加载由某个class loader加载的类。注意区分initiating class loader和defining class loader两个概念。
- 第三点，`isModifiableClass(Class<?> theClass)`是判断一个Class是否可以被修改，这是进行redefine和retransform的前提。同时，要注意**Primitive classes**和**array classes**是不能被修改的。

## 8. Instrumentation.getObjectSize()

### 8.1 getObjectSize

```java
public interface Instrumentation {
  long getObjectSize(Object objectToSize);
}
```

Returns an implementation-specific approximation of the amount of storage consumed by the specified object. The result may include some or all of the object’s **overhead**, and thus is useful for comparison within an implementation but not between implementations. The estimate may change during a single invocation of the JVM.

<u>Note that the `getObjectSize()` method does not include the memory used by other objects referenced by the object passed in</u>. For example, if Object A has a reference to Object B, then Object A’s reported memory usage will include only the bytes needed for the reference to Object B (usually 4 bytes), not the actual object.

小总结

- 第一，调用`getObjectSize(Object objectToSize)`方法后，得到的是一个粗略的对象大小，不同的虚拟机实现可能是不同的。
- 第二，调用`getObjectSize(Object objectToSize)`方法只是返回传入对象的大小，并不包含它关联的对象大小（例如，字段指向另一个对象）。

### 8.2 示例一：获取对象大小

#### Application

```java
package sample;

import lsieun.agent.LoadTimeAgent;

import java.lang.instrument.Instrumentation;

public class Program {
  public static void printInstrumentationSize(final Object obj) {
    Class<?> clazz = obj.getClass();
    Instrumentation inst = LoadTimeAgent.getInstrumentation();
    long size = inst.getObjectSize(obj);
    String message = String.format("Object of type %s has size of %s bytes.", clazz, size);
    System.out.println(message);
  }

  public static void main(String[] args) throws Exception {
    // 第一组
    Object obj = new Object();
    final StringBuilder sb = new StringBuilder();

    // 第二组
    String emptyString = "";
    String noneEmptyString = "ToBeOrNotToBeThatIsTheQuestion";

    // 第三组
    String[] strArray10 = new String[10];
    String[] strArray20 = new String[20];

    printInstrumentationSize(obj);
    printInstrumentationSize(sb);
    printInstrumentationSize(emptyString);
    printInstrumentationSize(noneEmptyString);
    printInstrumentationSize(strArray10);
    printInstrumentationSize(strArray20);
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  private static volatile Instrumentation globalInstrumentation;

  public static void premain(String agentArgs, Instrumentation inst) {
    globalInstrumentation = inst;
  }

  public static void agentmain(String agentArgs, Instrumentation inst) {
    globalInstrumentation = inst;
  }

  public static Instrumentation getInstrumentation() {
    if (globalInstrumentation == null) {
      throw new IllegalStateException("Agent not initialized.");
    }
    return globalInstrumentation;
  }
}
```

#### Run

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

Object of type class java.lang.Object has size of 16 bytes.
Object of type class java.lang.StringBuilder has size of 24 bytes.
Object of type class java.lang.String has size of 24 bytes.
Object of type class java.lang.String has size of 24 bytes.
Object of type class [Ljava.lang.String; has size of 56 bytes.
Object of type class [Ljava.lang.String; has size of 96 bytes.
```

### 8.3 示例二：获取对象大小（深度）

Calculating “deep” memory usage of an object

Usually it is more interesting to query the **“deep” memory usage** of an object, which includes “subobjects” (objects referred to by a given object).

#### Application

```java
package sample;

import lsieun.agent.LoadTimeAgent;
import lsieun.utils.MemoryUtils;

import java.lang.instrument.Instrumentation;

public class Program {
  public static void printInstrumentationSize(final Object obj) {
    Class<?> clazz = obj.getClass();
    Instrumentation inst = LoadTimeAgent.getInstrumentation();

    MemoryUtils.setInstrumentation(inst);
    long size = MemoryUtils.deepMemoryUsageOf(obj);

    String message = String.format("Object of type %s has size of %s bytes.", clazz, size);
    System.out.println(message);
  }

  public static void main(String[] args) throws Exception {
    // 第一组
    Object obj = new Object();
    final StringBuilder sb = new StringBuilder();

    // 第二组
    String emptyString = "";
    String noneEmptyString = "ToBeOrNotToBeThatIsTheQuestion";

    // 第三组
    String[] strArray10 = new String[10];
    String[] strArray20 = new String[20];

    printInstrumentationSize(obj);
    printInstrumentationSize(sb);
    printInstrumentationSize(emptyString);
    printInstrumentationSize(noneEmptyString);
    printInstrumentationSize(strArray10);
    printInstrumentationSize(strArray20);
  }
}
```

#### Agent Jar

这部分代码来参考自[Classmexer agent](https://www.javamex.com/classmexer/)。

```java
package lsieun.utils;

public enum VisibilityFilter {
  ALL, PRIVATE_ONLY, NON_PUBLIC, NONE;
}
```

```pseudocode
             ┌─── Primitive Type
             │
             │                                        ┌─── static field
             │                                        │
Java Type ───┤                      ┌─── Class ───────┤                        ┌─── public
             │                      │                 │                        │
             │                      │                 │                        ├─── protected
             │                      │                 └─── non-static field ───┤
             │                      │                                          ├─── package
             └─── Reference Type ───┤                                          │
                                    │                                          └─── private
                                    │
                                    ├─── Interface
                                    │
                                    └─── Array
```

```java
package lsieun.utils;

import java.lang.instrument.Instrumentation;
import java.lang.reflect.Field;
import java.util.Collection;
import java.util.HashSet;
import java.util.Set;
import java.util.Stack;

public class MemoryUtils {
  private static final int ACC_PUBLIC    = 0x0001; // class, field, method
  private static final int ACC_PRIVATE   = 0x0002; // class, field, method
  private static final int ACC_PROTECTED = 0x0004; // class, field, method
  private static final int ACC_STATIC    = 0x0008; // field, method

  private static Instrumentation inst;

  public static void setInstrumentation(Instrumentation inst) {
    MemoryUtils.inst = inst;
  }

  public static Instrumentation getInstrumentation() {
    return inst;
  }

  public static long memoryUsageOf(Object obj) {
    return getInstrumentation().getObjectSize(obj);
  }

  public static long deepMemoryUsageOf(Object obj) {
    return deepMemoryUsageOf(obj, VisibilityFilter.NON_PUBLIC);
  }

  public static long deepMemoryUsageOf(Object obj, VisibilityFilter referenceFilter) {
    return deepMemoryUsageOf0(getInstrumentation(), new HashSet<>(), obj, referenceFilter);
  }

  public static long deepMemoryUsageOfAll(Collection<?> coll) {
    return deepMemoryUsageOfAll(coll, VisibilityFilter.NON_PUBLIC);
  }

  public static long deepMemoryUsageOfAll(Collection<?> coll, VisibilityFilter referenceFilter) {
    Instrumentation inst = getInstrumentation();
    long total = 0L;
    Set<Integer> counted = new HashSet<>(coll.size() * 4);
    for (Object obj : coll) {
      total += deepMemoryUsageOf0(inst, counted, obj, referenceFilter);
    }
    return total;
  }

  private static long deepMemoryUsageOf0(Instrumentation instrumentation, Set<Integer> counted, Object obj, VisibilityFilter filter) {
    Stack<Object> stack = new Stack<>();
    stack.push(obj);
    long total = 0L;
    while (!stack.isEmpty()) {
      Object item = stack.pop();
      // 计算对象的hash值是为了避免同一个对象重复计算多次
      if (counted.add(System.identityHashCode(item))) {
        long size = instrumentation.getObjectSize(item);
        total += size;
        Class<?> clazz = item.getClass();

        // 如果是数组类型，则要计算每一个元素的大小
        Class<?> compType = clazz.getComponentType();
        if (compType != null && !compType.isPrimitive()) {
          Object[] array = (Object[]) item;
          for (Object element : array) {
            if (element != null) {
              stack.push(element);
            }
          }
        }

        // 递归查找类里面定义的具体字段值大小
        while (clazz != null) {
          Field[] fields = clazz.getDeclaredFields();
          for (Field f : fields) {
            int modifiers = f.getModifiers();
            if ((modifiers & ACC_STATIC) == 0 && isOf(filter, modifiers)) {
              Class<?> fieldClass = f.getType();
              if (!fieldClass.isPrimitive()) {
                if (!f.isAccessible()) {
                  f.setAccessible(true);
                }
                try {
                  Object subObj = f.get(item);
                  if (subObj != null) {
                    stack.push(subObj);
                  }
                } catch (IllegalAccessException illAcc) {
                  throw new InternalError("Couldn't read " + f);
                }
              }
            }
          }
          clazz = clazz.getSuperclass();
        }
      }
    }
    return total;
  }

  private static boolean isOf(VisibilityFilter f, int modifiers) {
    switch (f) {
      case NONE:
        return false;
      case PRIVATE_ONLY:
        return ((modifiers & ACC_PRIVATE) != 0);
      case NON_PUBLIC:
        return ((modifiers & ACC_PUBLIC) == 0);
      case ALL:
        return true;
    }
    throw new IllegalArgumentException("Illegal filter " + modifiers);
  }
}
```

#### Run

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Object of type class java.lang.Object has size of 16 bytes.
Object of type class java.lang.StringBuilder has size of 72 bytes.
Object of type class java.lang.String has size of 40 bytes.
Object of type class java.lang.String has size of 104 bytes.
Object of type class [Ljava.lang.String; has size of 56 bytes.
Object of type class [Ljava.lang.String; has size of 96 bytes.
```

### 8.4 总结

本文内容总结如下：

- 第一点，调用`getObjectSize(Object objectToSize)`方法返回的是一个对象占用的空间大小，是一个粗略值，不一定十分准确。
- 第二点，如何计算一个对象的”deep” memory usage。

## 9. Instrumentation.appendToXxxClassLoaderSearch()

### 9.1 ClassLoaderSearch

```java
public interface Instrumentation {
  // 1.6
  void appendToBootstrapClassLoaderSearch(JarFile jarfile);
  // 1.6
  void appendToSystemClassLoaderSearch(JarFile jarfile);
}
```

- `void appendToBootstrapClassLoaderSearch(JarFile jarfile)`: Specifies a JAR file with instrumentation classes to be defined by the **bootstrap class loader**.
  - When the virtual machine’s built-in class loader, known as the “bootstrap class loader”, unsuccessfully searches for a class, the entries in the JAR file will be searched as well.
  - This method may be used **multiple times** to add multiple JAR files to be searched in the order that this method was invoked.
  - The agent should take care to ensure that the JAR does not contain any classes or resources other than those to be defined by the bootstrap class loader for the purpose of instrumentation. Failure to observe this warning could result in unexpected behavior that is difficult to diagnose.
- `void appendToSystemClassLoaderSearch(JarFile jarfile)`: Specifies a JAR file with instrumentation classes to be defined by the **system class loader**.
  - When the system class loader for delegation unsuccessfully searches for a class, the entries in the JarFile will be searched as well.
  - This method may be used **multiple times** to add multiple JAR files to be searched in the order that this method was invoked.
  - The agent should take care to ensure that the JAR does not contain any classes or resources other than those to be defined by the system class loader for the purpose of instrumentation. Failure to observe this warning could result in unexpected behavior that is difficult to diagnose (see appendToBootstrapClassLoaderSearch).
  - <u>This method does not change the value of `java.class.path` system property</u>.

这两个方法很相似，都是将`JarFile`添加到class path当中，不同的地方在于：一个是添加到bootstrap classloader，另一个是添加到system classloader。

这两个方法的共同点，还体现在：

- 方法可以调用多次，来添加多个`JarFile`。
- 使用时候要注意，只添加必要的`JarFile`；否则，可能会造成无法预料的问题。

### 9.2 示例一：Class Path

本示例的目的：看看这两个方法对class path有什么影响。

#### Application

##### 版本一

第一个版本，通过`URLClassLoader.getURLs()`来获取class path

```java
package sample;

import lsieun.utils.PrintUtils;

public class Program {
  public static void main(String[] args) {
    PrintUtils.printBootstrapClassPath();
    PrintUtils.printExtensionClassPath();
    PrintUtils.printApplicationClassPath();
  }
}
```

##### 版本二

第二个版本，通过读取属性（例如，`java.class.path`）来获取class path

```java
package sample;

import lsieun.utils.ClassPathType;
import lsieun.utils.PrintUtils;

public class Program {
  public static void main(String[] args) {
    PrintUtils.printClassPath(ClassPathType.SUN_BOOT_CLASS_PATH);
    PrintUtils.printClassPath(ClassPathType.JAVA_EXT_DIRS);
    PrintUtils.printClassPath(ClassPathType.JAVA_CLASS_PATH);
  }
}
```

#### Agent Jar

```java
import lsieun.utils.ArgUtils;
import lsieun.utils.JarUtils;
import lsieun.utils.PrintUtils;

import java.io.IOException;
import java.lang.instrument.Instrumentation;
import java.util.jar.JarFile;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) throws IOException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，添加Class Search Path
    String jarPath = JarUtils.getToolsJarPath();
    JarFile jarFile = new JarFile(jarPath);
    String append = ArgUtils.parseAgentArgs(agentArgs, "append");
    if ("app".equals(append)) {
      inst.appendToSystemClassLoaderSearch(jarFile);
      System.out.println("Append to Application Class Path: " + jarPath);
    }
    else if ("boot".equals(append)) {
      inst.appendToBootstrapClassLoaderSearch(jarFile);
      System.out.println("Append to Bootstrap Class Path: " + jarPath);
    }
    else {
      System.out.println("No Append: " + jarPath);
    }


    // 第三步，加载Class
    String className = ArgUtils.parseAgentArgs(agentArgs, "class");
    if (className != null) {
      System.out.println("try to load class: " + className);
      try {
        Class<?> clazz = Class.forName(className);
        ClassLoader loader = clazz.getClassLoader();
        String message = String.format("load class %s from %s", clazz.getName(), loader);
        System.out.println(message);
      } catch (ClassNotFoundException e) {
        System.out.println("load class failed: " + className);
        e.printStackTrace();
      }
    }
  }
}
```

#### Run

##### None

```shell
$ java -cp ./target/classes/ sample.Program
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.country=US
=========Bootstrap ClassPath=========
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/resources.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/rt.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/sunrsasign.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jsse.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jce.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/charsets.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jfr.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/classes

=========Extension ClassPath=========
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/access-bridge-64.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/cldrdata.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/dnsns.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/jaccess.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/jfxrt.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/localedata.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/nashorn.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/sunec.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/sunjce_provider.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/sunmscapi.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/sunpkcs11.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/ext/zipfs.jar

=========Application ClassPath=========
--->file:/D:/git-repo/learn-java-agent/target/classes/
```

##### Load-Time

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

No Append: C:\Program Files\Java\jdk1.8.0_301\lib\tools.jar
=========Bootstrap ClassPath=========
......

=========Extension ClassPath=========
......

=========Application ClassPath=========
--->file:/D:/git-repo/learn-java-agent/target/classes/
--->file:/D:/git-repo/learn-java-agent/target/TheAgent.jar         // 注意，这里是新添加的内容
```

##### Class-Path

在`TheAgent.jar`的`META-INF/MANIFEST.MF`文件中，有`Class-Path`属性：

```shell
Class-Path: lib/asm-9.2.jar lib/asm-util-9.2.jar lib/asm-commons-9.2.jar
 lib/asm-tree-9.2.jar lib/asm-analysis-9.2.jar
```

```shell
class=org.objectweb.asm.Opcodes
```

再次运行时，添加`class:org.objectweb.asm.Opcodes`选项：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=class=org.objectweb.asm.Opcodes sample.Program
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.country=US
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: class=org.objectweb.asm.Opcodes
    (3) Instrumentation: sun.instrument.InstrumentationImpl
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

No Append: C:\Program Files\Java\jdk1.8.0_301\lib\tools.jar
try to load class: org.objectweb.asm.Opcodes
load class org.objectweb.asm.Opcodes from sun.misc.Launcher$AppClassLoader@18b4aac2    // 注意，Opcodes类是从Class-Path中加载到的
=========Bootstrap ClassPath=========
......

=========Extension ClassPath=========
......

=========Application ClassPath=========
--->file:/D:/git-repo/learn-java-agent/target/classes/
--->file:/D:/git-repo/learn-java-agent/target/TheAgent.jar
```

##### SystemCLSearch

测试目标：

- 将`tools.jar`文件添加到System ClassLoader的搜索范围
- 尝试加载`com.sun.tools.attach.VirtualMachine`类，该类位于`tools.jar`文件内

```shell
append=app,class=com.sun.tools.attach.VirtualMachine
```

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=append=app,class=com.sun.tools.attach.VirtualMachine sample.Program
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.country=US
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: append=app,class=com.sun.tools.attach.VirtualMachine
    (3) Instrumentation: sun.instrument.InstrumentationImpl
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

Append to Application Class Path: C:\Program Files\Java\jdk1.8.0_301\lib\tools.jar
try to load class: com.sun.tools.attach.VirtualMachine
load class com.sun.tools.attach.VirtualMachine from sun.misc.Launcher$AppClassLoader@18b4aac2    // 注意，这里是由AppClassLoader加载
=========Bootstrap ClassPath=========
......

=========Extension ClassPath=========
......

=========Application ClassPath=========
--->file:/D:/git-repo/learn-java-agent/target/classes/
--->file:/D:/git-repo/learn-java-agent/target/TheAgent.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/lib/tools.jar    // 注意，这里是tools.jar文件
```

##### BootstrapCLSearch

测试目标：

- 将`tools.jar`文件添加到Bootstrap ClassLoader的搜索范围
- 尝试加载`com.sun.tools.attach.VirtualMachine`类，该类位于`tools.jar`文件内

```shell
append=boot,class=com.sun.tools.attach.VirtualMachine
```

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar=append=boot,class=com.sun.tools.attach.VirtualMachine sample.Program
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en -Duser.country=US
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: append=boot,class=com.sun.tools.attach.VirtualMachine
    (3) Instrumentation: sun.instrument.InstrumentationImpl
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

Append to Bootstrap Class Path: C:\Program Files\Java\jdk1.8.0_301\lib\tools.jar
try to load class: com.sun.tools.attach.VirtualMachine
load class com.sun.tools.attach.VirtualMachine from null    // 注意，这里是由Bootstrap ClassLoader加载
=========Bootstrap ClassPath=========
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/resources.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/rt.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/sunrsasign.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jsse.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jce.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/charsets.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/lib/jfr.jar
--->file:/C:/Program%20Files/Java/jdk1.8.0_301/jre/classes    // 注意，并没有出现tools.jar

=========Extension ClassPath=========
......

=========Application ClassPath=========
--->file:/D:/git-repo/learn-java-agent/target/classes/
--->file:/D:/git-repo/learn-java-agent/target/TheAgent.jar
```

### 9.3 示例二：Bootstrap Search

本示例是介绍一种应用场景：JDK的内部类如何调用我们自己写的类。

`StrictMath`修改之前的代码：

```java
public final class StrictMath {
  public static int addExact(int x, int y) {
    return Math.addExact(x, y);
  }
}
```

`StrictMath`第一次修改：（正常）

```java
public final class StrictMath {
  public static int addExact(int var0, int var1) {
    System.out.println("Method Enter: java/lang/StrictMath.addExact(II)I");
    return Math.addExact(var0, var1);
  }
}
```

`StrictMath`第二次修改：（出错）

```java
public final class StrictMath {
  public static int addExact(int var0, int var1) {
    ParameterUtils.printText("Method Enter: java/lang/StrictMath.addExact(II)I");
    return Math.addExact(var0, var1);
  }
}
```

#### Application

```java
package sample;

public class Program {
  public static void main(String[] args) {
    int sum = StrictMath.addExact(10, 20);
    System.out.println(sum);
  }
}
```

#### Agent Jar

##### LoadTimeAgent

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "java.lang.StrictMath";

    // 第三步，使用inst：添加transformer
    ClassFileTransformer transformer = new ASMTransformer(className);
    inst.addTransformer(transformer, false);
  }
}
```

##### ASMTransformer

```java
package lsieun.instrument;

import lsieun.asm.visitor.*;
import org.objectweb.asm.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Objects;

public class ASMTransformer implements ClassFileTransformer {
  private final String internalName;

  public ASMTransformer(String internalName) {
    Objects.requireNonNull(internalName);
    this.internalName = internalName.replace(".", "/");
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className.equals(internalName)) {
      System.out.println("transform class: " + className);
      ClassReader cr = new ClassReader(classfileBuffer);
      ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
      ClassVisitor cv = new MethodEnterVisitor(cw);

      int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
      cr.accept(cv, parsingOptions);

      return cw.toByteArray();
    }

    return null;
  }
}
```

##### MethodEnterVisitor

```java
import lsieun.cst.Const;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class MethodEnterVisitor extends ClassVisitor {
  private String owner;

  public MethodEnterVisitor(ClassVisitor classVisitor) {
    super(Const.ASM_VERSION, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT;
      boolean isNativeMethod = (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodEnterAdapter(mv, owner, name, descriptor);
      }
    }
    return mv;
  }

  private static class MethodEnterAdapter extends MethodVisitor {
    private final String owner;
    private final String methodName;
    private final String methodDesc;

    public MethodEnterAdapter(MethodVisitor methodVisitor, String owner, String methodName, String methodDesc) {
      super(Const.ASM_VERSION, methodVisitor);
      this.owner = owner;
      this.methodName = methodName;
      this.methodDesc = methodDesc;
    }

    @Override
    public void visitCode() {
      // 首先，处理自己的代码逻辑
      String message = String.format("Method Enter: %s.%s%s", owner, methodName, methodDesc);
      // (1) 引用自定义的类
      //            super.visitLdcInsn(message);
      //            super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);

      // (2) 引用JDK的内部类
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitLdcInsn(message);
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

      // 其次，调用父类的方法实现
      super.visitCode();
    }
  }
}
```

#### 出现问题

当我们使用`MethodEnterAdapter`类当中第(2)种方式时，不会出现错误；但是，当我们使用第(1)种方式时，就会出现`NoClassDefFoundError`错误：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1704856573
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

transform class: java/lang/StrictMath
Exception in thread "main" java.lang.NoClassDefFoundError: lsieun/utils/ParameterUtils
        at java.lang.StrictMath.addExact(Unknown Source)
        at sample.Program.main(Program.java:5)
```

#### 解决问题

那么，如何解决这个问题呢？

首先，我们可以将`lsieun.utils.ParameterUtils`放到一个`.jar`文件当中，取名为`lsieun-utils.jar`：

```shell
jar -cvf lsieun-utils.jar lsieun/utils/ParameterUtils.class
```

第一种解决方式，我们在代码当中调用`Instrumentation.appendToBootstrapClassLoaderSearch()`方法来加载`lsieun-utils.jar`：

```java
import lsieun.instrument.*;
import lsieun.utils.*;

import java.io.IOException;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.util.jar.JarFile;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) throws IOException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "java.lang.StrictMath";

    // 第三步，使用inst：添加transformer
    ClassFileTransformer transformer = new ASMTransformer(className);
    inst.addTransformer(transformer, false);

    // 第四步，添加jar包
    String jarPath = "D:\\git-repo\\learn-java-agent\\target\\lsieun-utils.jar";
    JarFile jarFile = new JarFile(jarPath);
    inst.appendToBootstrapClassLoaderSearch(jarFile);
  }
}
```

第二种解决方式，在`MANIFEST.MF`文件中添加`Boot-Class-Path`属性：

```xml
<Boot-Class-Path>lsieun-utils.jar</Boot-Class-Path>
```

### 9.4 总结

本文内容总结如下：

- 第一点，这两个方法的本质就是“请求支援”。当前的Agent Jar没有办法实现某种功能，因此请求外来的`JarFile`来协助。
- 第二点，示例一是演示两个方法对于class path的影响。
- 第三点，示例二是介绍了一种使用`void appendToBootstrapClassLoaderSearch(JarFile jarfile)`的场景：JDK的内部类如何调用我们自己写的类。

## 10. Instrumentation.redefineModule()

### 10.1 redefineModule

```java
public interface Instrumentation {
  boolean isModifiableModule(Module module);
  void redefineModule (Module module,
                       Set<Module> extraReads,
                       Map<String, Set<Module>> extraExports,
                       Map<String, Set<Module>> extraOpens,
                       Set<Class<?>> extraUses,
                       Map<Class<?>, List<Class<?>>> extraProvides);
}
```

- `isModifiableModule`: Tests whether a module can be modified with `redefineModule`.
- `redefineModule`: Redefine a module to expand the set of modules that it reads, the set of packages that it exports or opens, or the services that it uses or provides.

### 10.2 示例

#### Application

```java
package sample;

import java.lang.instrument.Instrumentation;

public class Program {
  public static void main(String[] args) {
    Module baseModule = Object.class.getModule();
    Module instrumentModule = Instrumentation.class.getModule();

    boolean canRead = baseModule.canRead(instrumentModule);
    String message = String.format("%s can read %s: %s", baseModule.getName(), instrumentModule.getName(), canRead);
    System.out.println(message);
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;
import java.util.Map;
import java.util.Set;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");

    // 第二步，判断一个module是否可以读取另一个module
    Module baseModule = Object.class.getModule();
    Module instrumentModule = Instrumentation.class.getModule();
    boolean canRead = baseModule.canRead(instrumentModule);

    // 第三步，使用inst：修改module权限
    if (!canRead && inst.isModifiableModule(baseModule)) {
      Set<Module> extraReads = Set.of(instrumentModule);
      inst.redefineModule(baseModule, extraReads, Map.of(), Map.of(), Set.of(), Map.of());
    }
  }
}
```

#### Run

##### None

```shell
$ java -cp ./target/classes/ sample.Program
java.base can read java.instrument: false
```

##### Load-Time

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
java.base can read java.instrument: true
```

### 10.3 总结

本文内容总结如下：

- 第一点， `redefineModule` 方法的作用是对 module 的访问权限进行修改，该方法是在 Java 9 引入的。

## 11. Instrumentation.redefineModule()

### 11.1 redefineModule

```java
public interface Instrumentation {
  boolean isModifiableModule(Module module);
  void redefineModule (Module module,
                       Set<Module> extraReads,
                       Map<String, Set<Module>> extraExports,
                       Map<String, Set<Module>> extraOpens,
                       Set<Class<?>> extraUses,
                       Map<Class<?>, List<Class<?>>> extraProvides);
}
```

- `isModifiableModule`: Tests whether a module can be modified with `redefineModule`.
- `redefineModule`: Redefine a module to expand the set of modules that it reads, the set of packages that it exports or opens, or the services that it uses or provides.

### 11.2 示例

#### Application

```java
package sample;

import java.lang.instrument.Instrumentation;

public class Program {
  public static void main(String[] args) {
    Module baseModule = Object.class.getModule();
    Module instrumentModule = Instrumentation.class.getModule();

    boolean canRead = baseModule.canRead(instrumentModule);
    String message = String.format("%s can read %s: %s", baseModule.getName(), instrumentModule.getName(), canRead);
    System.out.println(message);
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

import java.lang.instrument.Instrumentation;
import java.util.Map;
import java.util.Set;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息
    System.out.println("Premain-Class: " + LoadTimeAgent.class.getName());
    System.out.println("Can-Redefine-Classes: " + inst.isRedefineClassesSupported());
    System.out.println("Can-Retransform-Classes: " + inst.isRetransformClassesSupported());
    System.out.println("Can-Set-Native-Method-Prefix: " + inst.isNativeMethodPrefixSupported());
    System.out.println("========= ========= =========");

    // 第二步，判断一个module是否可以读取另一个module
    Module baseModule = Object.class.getModule();
    Module instrumentModule = Instrumentation.class.getModule();
    boolean canRead = baseModule.canRead(instrumentModule);

    // 第三步，使用inst：修改module权限
    if (!canRead && inst.isModifiableModule(baseModule)) {
      Set<Module> extraReads = Set.of(instrumentModule);
      inst.redefineModule(baseModule, extraReads, Map.of(), Map.of(), Set.of(), Map.of());
    }
  }
}
```

#### Run

##### None

```shell
$ java -cp ./target/classes/ sample.Program
java.base can read java.instrument: false
```

##### Load-Time

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Premain-Class: lsieun.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
java.base can read java.instrument: true
```

### 11.3 总结

本文内容总结如下：

- 第一点， `redefineModule` 方法的作用是对 module 的访问权限进行修改，该方法是在 Java 9 引入的。

## 12. InstrumentationImpl

在本文当中，我们的关注点在于`InstrumentationImpl`、transformer和`TransformerManager`三者的关系。

### 12.1 InstrumentationImpl

#### class info

`sun.instrument.InstrumentationImpl`实现了`java.lang.instrument.Instrumentation`接口：

```java
public class InstrumentationImpl implements Instrumentation {
}
```

#### fields

```java
public class InstrumentationImpl implements Instrumentation {
  private final TransformerManager mTransformerManager;
  private TransformerManager mRetransfomableTransformerManager;

  // ...
}
```

#### constructor

```java
public class InstrumentationImpl implements Instrumentation {
  private InstrumentationImpl(long nativeAgent,
                              boolean environmentSupportsRedefineClasses,
                              boolean environmentSupportsNativeMethodPrefix) {
    mTransformerManager = new TransformerManager(false);
    mRetransfomableTransformerManager = null;

    // ...
  }
}
```

#### xxxTransformer

##### addTransformer

```java
public class InstrumentationImpl implements Instrumentation {
  public void addTransformer(ClassFileTransformer transformer) {
    addTransformer(transformer, false);
  }

  public synchronized void addTransformer(ClassFileTransformer transformer, boolean canRetransform) {
    if (transformer == null) {
      throw new NullPointerException("null passed as 'transformer' in addTransformer");
    }
    if (canRetransform) {
      if (!isRetransformClassesSupported()) {
        throw new UnsupportedOperationException("adding retransformable transformers is not supported in this environment");
      }
      if (mRetransfomableTransformerManager == null) {
        mRetransfomableTransformerManager = new TransformerManager(true);
      }
      mRetransfomableTransformerManager.addTransformer(transformer);
      if (mRetransfomableTransformerManager.getTransformerCount() == 1) {
        setHasRetransformableTransformers(mNativeAgent, true);
      }
    }
    else {
      mTransformerManager.addTransformer(transformer);
    }
  }
}
```

##### removeTransformer

```java
public class InstrumentationImpl implements Instrumentation {
  public synchronized boolean removeTransformer(ClassFileTransformer transformer) {
    if (transformer == null) {
      throw new NullPointerException("null passed as 'transformer' in removeTransformer");
    }
    TransformerManager mgr = findTransformerManager(transformer);
    if (mgr != null) {
      mgr.removeTransformer(transformer);
      if (mgr.isRetransformable() && mgr.getTransformerCount() == 0) {
        setHasRetransformableTransformers(mNativeAgent, false);
      }
      return true;
    }
    return false;
  }
}
```

##### findTransformerManager

```java
public class InstrumentationImpl implements Instrumentation {
  private TransformerManager findTransformerManager(ClassFileTransformer transformer) {
    if (mTransformerManager.includesTransformer(transformer)) {
      return mTransformerManager;
    }
    if (mRetransfomableTransformerManager != null &&
        mRetransfomableTransformerManager.includesTransformer(transformer)) {
      return mRetransfomableTransformerManager;
    }
    return null;
  }
}
```

#### transform

```java
public class InstrumentationImpl implements Instrumentation {
  private byte[] transform(ClassLoader loader,
                           String classname,
                           Class<?> classBeingRedefined,
                           ProtectionDomain protectionDomain,
                           byte[] classfileBuffer,
                           boolean isRetransformer) {
    TransformerManager mgr = isRetransformer ? mRetransfomableTransformerManager : mTransformerManager;
    if (mgr == null) {
      return null; // no manager, no transform
    }
    else {
      return mgr.transform(loader, classname, classBeingRedefined, protectionDomain, classfileBuffer);
    }
  }
}
```

### 12.2 TransformerInfo

```java
private class TransformerInfo {
  final ClassFileTransformer mTransformer;
  String mPrefix;

  TransformerInfo(ClassFileTransformer transformer) {
    mTransformer = transformer;
    mPrefix = null;
  }

  ClassFileTransformer transformer() {
    return mTransformer;
  }

  String getPrefix() {
    return mPrefix;
  }

  void setPrefix(String prefix) {
    mPrefix = prefix;
  }
}
```

### 12.3 TransformerManager

#### class info

```java
/**
 * Support class for the InstrumentationImpl. Manages the list of registered transformers.
 * Keeps everything in the right order, deals with sync of the list,
 * and actually does the calling of the transformers.
 */
public class TransformerManager {
}
```

#### fields

```java
public class TransformerManager {
  private TransformerInfo[] mTransformerList;
  private boolean mIsRetransformable;
}
```

#### constructor

```java
public class TransformerManager {
  TransformerManager(boolean isRetransformable) {
    mTransformerList = new TransformerInfo[0];
    mIsRetransformable = isRetransformable;
  }
}
```

#### xxxTransformer

##### addTransformer

```java
public class TransformerManager {
  public synchronized void addTransformer(ClassFileTransformer transformer) {
    TransformerInfo[] oldList = mTransformerList;
    TransformerInfo[] newList = new TransformerInfo[oldList.length + 1];
    System.arraycopy(oldList, 0, newList, 0, oldList.length);
    newList[oldList.length] = new TransformerInfo(transformer);
    mTransformerList = newList;
  }
}
```

##### removeTransformer

```java
public class TransformerManager {
  public synchronized boolean removeTransformer(ClassFileTransformer transformer) {
    boolean found = false;
    TransformerInfo[] oldList = mTransformerList;
    int oldLength = oldList.length;
    int newLength = oldLength - 1;

    // look for it in the list, starting at the last added, and remember
    // where it was if we found it
    int matchingIndex = 0;
    for (int x = oldLength - 1; x >= 0; x--) {
      if (oldList[x].transformer() == transformer) {
        found = true;
        matchingIndex = x;
        break;
      }
    }

    // make a copy of the array without the matching element
    if (found) {
      TransformerInfo[] newList = new TransformerInfo[newLength];

      // copy up to but not including the match
      if (matchingIndex > 0) {
        System.arraycopy(oldList, 0, newList, 0, matchingIndex);
      }

      // if there is anything after the match, copy it as well
      if (matchingIndex < (newLength)) {
        System.arraycopy(oldList, matchingIndex + 1, newList, matchingIndex, (newLength) - matchingIndex);
      }
      mTransformerList = newList;
    }
    return found;
  }
}
```

##### includesTransformer

```java
public class TransformerManager {
  synchronized boolean includesTransformer(ClassFileTransformer transformer) {
    for (TransformerInfo info : mTransformerList) {
      if (info.transformer() == transformer) {
        return true;
      }
    }
    return false;
  }
}
```

#### transform

```java
public class TransformerManager {
  public byte[] transform(ClassLoader loader,
                          String classname,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) {
    boolean someoneTouchedTheBytecode = false;

    TransformerInfo[] transformerList = getSnapshotTransformerList();

    byte[] bufferToUse = classfileBuffer;

    // order matters, gotta run 'em in the order they were added
    for (int x = 0; x < transformerList.length; x++) {
      TransformerInfo transformerInfo = transformerList[x];
      ClassFileTransformer transformer = transformerInfo.transformer();
      byte[] transformedBytes = null;

      try {
        transformedBytes = transformer.transform(loader, classname, classBeingRedefined, protectionDomain, bufferToUse);
      } catch (Throwable t) {
        // don't let any one transformer mess it up for the others.
        // This is where we need to put some logging. What should go here? FIXME
      }

      if (transformedBytes != null) {
        someoneTouchedTheBytecode = true;
        bufferToUse = transformedBytes;
      }
    }

    // if someone modified it, return the modified buffer.
    // otherwise return null to mean "no transforms occurred"
    byte[] result;
    if (someoneTouchedTheBytecode) {
      result = bufferToUse;
    }
    else {
      result = null;
    }

    return result;
  }
}
```

### 12.4 总结

本文内容总结如下：

- 第一点，在`InstrumentationImpl`当中，transformer会根据是否具体retransform能力而分开存储。
- 第二点，在`TransformerManager`当中，重点关注`transform`方法的处理逻辑。

## 13. ClassFileTransformer

### 13.1 如何实现 Transformer

An agent provides an implementation of `ClassFileTransformer` interface in order to transform class files.

```java
public interface ClassFileTransformer {
  byte[] transform(ClassLoader         loader,
                   String              className,
                   Class<?>            classBeingRedefined,
                   ProtectionDomain    protectionDomain,
                   byte[]              classfileBuffer)
    throws IllegalClassFormatException;
}
```

如果我们实现了 `ClassFileTransformer` 接口，就可以对某一个 class file（`classfileBuffer`）进行处理，一般要考虑两个问题：

- 首先，有哪些类可以不处理？
- 其次，如何来对 `classfileBuffer` 进行处理

#### 有哪些类可以不处理

在实现 `ClassFileTransformer.transform()` 方法时，我们要考虑一下哪些类不需要处理，让“影响范围”最小化：

- 第一， primitive type（原始类型，例如 `int`）和 array（数组）不处理。（因为本来原始类型和数组类型就不允许redefine和retransform）
- 第二，JDK 的内置类或第三方类库当中的 `.class` 文件，一般情况下不修改，特殊情况下才进行修改。
- 第三，自己写的 Agent Jar 当中的类

注意：对于 primitive type（例如， `int` ）和 array 的判断是多余的。 当使用 `Instrumentation.isModifiableClass()` 对 `int.class` 和 `String[].class` 进行判断时，都会返回 `false` 值。 如果对 `int.class` 和 `String[].class` 进行 `Instrumentation.retransformClasses()` 操作，会出现 `UnmodifiableClassException` 异常。

写法一：（逻辑清晰）

```java
package lsieun.instrument;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class FilterTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {

    // 第一，数组，不处理
    if (className.startsWith("[")) return null;

    // 第二，JDK的内置类，不处理
    if (className.startsWith("java/")) return null;
    if (className.startsWith("javax/")) return null;
    if (className.startsWith("jdk/")) return null;
    if (className.startsWith("com/sun/")) return null;
    if (className.startsWith("sun/")) return null;
    if (className.startsWith("org/")) return null;

    // 第三，自己写的类，不处理
    if (className.startsWith("lsieun")) return null;

    // TODO: 使用字节码类库对classfileBuffer进行转换

    // 如果不修改，则返回null值
    return null;
  }
}
```

写法二：（代码简洁）

```java
package lsieun.instrument;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Arrays;
import java.util.List;

public class FilterTransformer implements ClassFileTransformer {
  public static final List<String> ignoredPackages = Arrays.asList("[", "com/", "com/sun/", "java/", "javax/", "jdk/", "lsieun/", "org/", "sun/");

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    // 有些类，不处理
    if (className == null) return null;
    for (String name : ignoredPackages) {
      if (className.startsWith(name)) {
        return null;
      }
    }

    // TODO: 使用字节码类库对classfileBuffer进行转换

    // 如果不修改，则返回null值
    return null;
  }
}
```

另外，如果我们不想对 bootstrap classloader 加载的类进行修改，也可以判断 `loader` 是否为 `null`。

再有一点，Lambda表达式生成的类要慎重处理。 在[Alibaba Arthas](https://github.com/alibaba/arthas)当中对 Lambda 表达式生成的类进行了“忽略”处理：

```java
public class ClassUtils {
  public static boolean isLambdaClass(Class<?> clazz) {
    return clazz.getName().contains("$$Lambda$");
  }
}
```

下面是对`ClassUtils.isLambdaClass()`方法的使用示例：

```java
public class InstrumentationUtils {
  public static void retransformClasses(Instrumentation inst, ClassFileTransformer transformer, Set<Class<?>> classes) {
    try {
      inst.addTransformer(transformer, true);

      for (Class<?> clazz : classes) {
        if (ClassUtils.isLambdaClass(clazz)) {
          logger.info("ignore lambda class: {}, because jdk do not support retransform lambda class: https://github.com/alibaba/arthas/issues/1512.",
                      clazz.getName());
          continue;
        }
        try {
          inst.retransformClasses(clazz);
        } catch (Throwable e) {
          String errorMsg = "retransformClasses class error, name: " + clazz.getName();
          logger.error(errorMsg, e);
        }
      }
    } finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

在 `InnerClassLambdaMetafactory` 类的构造方法中，有如下代码：

```java
lambdaClassName = targetClass.getName().replace('.', '/') + "$$Lambda$" + counter.incrementAndGet();
```

还有一种情况，我们明确知道要处理的是哪一个类：

```java
package lsieun.instrument;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Objects;

public class FilterTransformer implements ClassFileTransformer {
  private final String internalName;

  public FilterTransformer(String internalName) {
    Objects.requireNonNull(internalName);
    this.internalName = internalName.replace(".", "/");
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (className.equals(internalName)) {
      // TODO: 使用字节码类库对classfileBuffer进行转换
    }

    // 如果不修改，则返回null值
    return null;
  }
}
```

#### 如何来处理classfileBuffer

处理`ClassFileTransformer.transform()`方法中的`byte[] classfileBuffer`参数，一般要借助于第三方的操作字节码的类库，例如[ASM](https://asm.ow2.io/)、[ByteBuddy](https://bytebuddy.net/)和[Javassist](https://www.javassist.org/)。

#### 返回值

If the implementing method determines that no transformations are needed, it should return `null`.

Otherwise, it should create a new `byte[]` array, copy the input `classfileBuffer` into it, along with all desired transformations, and return the new array.

<u>The input `classfileBuffer` must not be modified.</u>

小总结：

- 第一，无论是否要进行transform操作，一定不要修改 `classfileBuffer` 的内容。
- 第二，如果不进行transform操作，则直接返回 `null` 就可以了。
- 第三，如果进行transform操作，则可以复制 `classfileBuffer` 内容后进行修改，再返回。

#### Lambda

在Java 8版本当中，我们可以将 `ClassFileTransformer` 接口用 Lambda 表达式提供实现，因为它有一个抽象的 `transform` 方法； 但是，到了Java 9之后，`ClassFileTransformer` 接口就不能再用 Lambda 表达式了，因为它有两个`default` 实现的 `transform` 方法。

我的个人理解：

- 对于一个简单的功能，将 `ClassFileTransformer` 接口写成 Lambda 表达式的形式，会方便一些；
- 对于一个复杂的功能，我更愿意把 `ClassFileTransformer` 写成一个具体的实现类，作为一个单独的文件存在。

```java
package lsieun.agent;

import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，使用inst：添加transformer
    ClassFileTransformer transformer = (loader, className, classBeingRedefined, protectionDomain, classfileBuffer) -> {
      return null;
    };
    inst.addTransformer(transformer, false);
  }
}
```

### 13.2 示例一：不排除自己写的类

#### Agent Jar

##### LoadTimeAgent

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，使用inst：添加transformer
    ClassFileTransformer transformer = new FilterTransformer();
    inst.addTransformer(transformer, false);
  }
}
```

##### FilterTransformer

```java
package lsieun.instrument;

import lsieun.asm.visitor.*;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class FilterTransformer implements ClassFileTransformer {
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    // 第一，数组，不处理
    if (className.startsWith("[")) return null;

    // 第二，JDK的内置类，不处理
    if (className.startsWith("java")) return null;
    if (className.startsWith("javax")) return null;
    if (className.startsWith("jdk")) return null;
    if (className.startsWith("com/sun")) return null;
    if (className.startsWith("sun")) return null;
    if (className.startsWith("org")) return null;

    // 使用ASM进行转换
    System.out.println("transform class: " + className);
    ClassReader cr = new ClassReader(classfileBuffer);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    ClassVisitor cv = new MethodEnterVisitor(cw);

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    return cw.toByteArray();
  }
}
```

##### MethodEnterVisitor

```java
package lsieun.asm.visitor;

import lsieun.cst.Const;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class MethodEnterVisitor extends ClassVisitor {
  private String owner;

  public MethodEnterVisitor(ClassVisitor classVisitor) {
    super(Const.ASM_VERSION, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT;
      boolean isNativeMethod = (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodEnterAdapter(mv, owner, name, descriptor);
      }
    }
    return mv;
  }

  private static class MethodEnterAdapter extends MethodVisitor {
    private final String owner;
    private final String methodName;
    private final String methodDesc;

    public MethodEnterAdapter(MethodVisitor methodVisitor, String owner, String methodName, String methodDesc) {
      super(Const.ASM_VERSION, methodVisitor);
      this.owner = owner;
      this.methodName = methodName;
      this.methodDesc = methodDesc;
    }

    @Override
    public void visitCode() {
      // 首先，处理自己的代码逻辑
      String message = String.format("Method Enter: %s.%s%s", owner, methodName, methodDesc);
      // (1) 引用自定义的类
      super.visitLdcInsn(message);
      super.visitMethodInsn(Opcodes.INVOKESTATIC, "lsieun/utils/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);

      // (2) 引用JDK的内部类
      //            super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      //            super.visitLdcInsn(message);
      //            super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

      // 其次，调用父类的方法实现
      super.visitCode();
    }
  }
}
```

#### Run

当我们运行的时候，会出现 `StackOverflowError` 错误：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

transform class: sample/Program
transform class: lsieun/utils/ParameterUtils
Exception in thread "main" java.lang.StackOverflowError
        at lsieun.utils.ParameterUtils.printText(Unknown Source)
        at lsieun.utils.ParameterUtils.printText(Unknown Source)
        at lsieun.utils.ParameterUtils.printText(Unknown Source)
        ...
```

分析原因：`ParameterUtils` 类当中的 `printText` 对自身进行了调用，进入无尽循环的状态。

```java
public class ParameterUtils {
  public static void printText(String var0) {
    printText("Method Enter: lsieun/utils/ParameterUtils.printText(Ljava/lang/String;)V");
    System.out.println(var0);
  }
}
```

### 13.3 一个Transformer

#### Transformer的分类

There are two kinds of transformers, determined by the `canRetransform` parameter of `Instrumentation.addTransformer(ClassFileTransformer,boolean)`:

- **retransformation capable transformers** that were added with `canRetransform` as `true`
- **retransformation incapable transformers** that were added with `canRetransform` as `false` or where added with `Instrumentation.addTransformer(ClassFileTransformer)`

```java
public interface Instrumentation {
  void addTransformer(ClassFileTransformer transformer);
  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
}
```

#### 调用时机

- <u>Once a transformer has been registered with `addTransformer`, the transformer will be called for **every new class definition** and **every class redefinition**.</u>
- **Retransformation capable transformers** will also be called on **every class retransformation**.

![img](https://lsieun.github.io/assets/images/java/agent/define-redefine-retransform.png)

- The request for a **new class definition** is made with `ClassLoader.defineClass` or its native equivalents.
- The request for a **class redefinition** is made with `Instrumentation.redefineClasses` or its native equivalents.
- The request for a **class retransformation** is made with `Instrumentation.retransformClasses` or its native equivalents.

```pseudocode
                               ┌─── define: ClassLoader.defineClass
               ┌─── loading ───┤
               │               └─── transform
class state ───┤
               │               ┌─── redefine: Instrumentation.redefineClasses
               └─── loaded ────┤
                               └─── retransform: Instrumentation.retransformClasses
```

在OpenJDK的源码中，`hotspot/src/share/vm/prims/jvmtiThreadState.hpp`文件定义了一个`JvmtiClassLoadKind`结构：

```c++
enum JvmtiClassLoadKind {
  jvmti_class_load_kind_load = 100,
  jvmti_class_load_kind_retransform,
  jvmti_class_load_kind_redefine
};
```

### 13.4 多个Transformer

#### 串联执行

When there are **multiple transformers**, transformations are composed by chaining the `transform` calls. That is, the byte array returned by one call to `transform` becomes the input (via the `classfileBuffer` parameter) to the next call.

**Transformations are applied in the following order:**

- **Retransformation incapable transformers**
- **Retransformation incapable native transformers**
- **Retransformation capable transformers**
- **Retransformation capable native transformers**

<u>For **retransformations**, the **retransformation incapable transformers** are not called, instead the result of the previous transformation is reused. In all other cases, this method is called. Within each of these groupings, transformers are called in the order registered. Native transformers are provided by the `ClassFileLoadHook` event in the Java Virtual Machine Tool Interface.</u>

JVM会去调用`InstrumentationImpl.transform()`方法，会再进一步调用`TransformerManager.transform()`方法：

```java
public class InstrumentationImpl implements Instrumentation {
  // WARNING: the native code knows the name & signature of this method
  private byte[] transform(ClassLoader loader,
                           String classname,
                           Class<?> classBeingRedefined,
                           ProtectionDomain protectionDomain,
                           byte[] classfileBuffer,
                           boolean isRetransformer) {
    TransformerManager mgr = isRetransformer ? mRetransfomableTransformerManager : mTransformerManager;
    if (mgr == null) {
      return null; // no manager, no transform
    }
    else {
      return mgr.transform(loader, classname, classBeingRedefined, protectionDomain, classfileBuffer);
    }
  }
}
```

在`TransformerManager.transform()`方法中，我们重点关注`someoneTouchedTheBytecode`和`bufferToUse`两个局部变量：

```java
public class TransformerManager {
  public byte[] transform(ClassLoader loader,
                          String classname,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) {
    boolean someoneTouchedTheBytecode = false;

    TransformerInfo[] transformerList = getSnapshotTransformerList();

    byte[] bufferToUse = classfileBuffer;

    // order matters, gotta run 'em in the order they were added
    for (int x = 0; x < transformerList.length; x++) {
      TransformerInfo transformerInfo = transformerList[x];
      ClassFileTransformer transformer = transformerInfo.transformer();
      byte[] transformedBytes = null;

      try {
        transformedBytes = transformer.transform(loader, classname, classBeingRedefined, protectionDomain, bufferToUse);
      } catch (Throwable t) {
        // don't let any one transformer mess it up for the others.
        // This is where we need to put some logging. What should go here? FIXME
      }

      if (transformedBytes != null) {
        someoneTouchedTheBytecode = true;
        bufferToUse = transformedBytes;
      }
    }

    // if someone modified it, return the modified buffer.
    // otherwise return null to mean "no transforms occurred"
    byte[] result;
    if (someoneTouchedTheBytecode) {
      result = bufferToUse;
    }
    else {
      result = null;
    }

    return result;
  }    
}
```

If the transformer throws an exception (which it doesn’t catch), subsequent transformers will still be called and the load, redefine or retransform will still be attempted. **Thus, throwing an exception has the same effect as returning `null`.**

小总结：

- 第一，在多个transformer当中，任何一个transform抛出任何异常，则相当于该transformer返回了`null`值。
- 第二，在多个transformer当中，某一个transform抛出任何异常，并不会影响后续transformer的执行。

#### First Input

The input (via the `classfileBuffer` parameter) to the first transformer is:

- for **new class definition**, the bytes passed to `ClassLoader.defineClass`
- for **class redefinition**, `definitions.getDefinitionClassFile()` where `definitions` is the parameter to `Instrumentation.redefineClasses`
- for **class retransformation**, the bytes passed to the **new class definition** or, **if redefined, the last redefinition**, with all transformations made by **retransformation incapable transformers** reapplied automatically and unaltered

![img](https://lsieun.github.io/assets/images/java/agent/define-redefine-retransform.png)

在OpenJDK的源码中，`hotspot/src/share/vm/prims/jvmtiExport.cpp`文件有如下代码：

```c++
void post_all_envs() {
  if (_load_kind != jvmti_class_load_kind_retransform) {
    // for class load and redefine,
    // call the non-retransformable agents
    JvmtiEnvIterator it;
    for (JvmtiEnv* env = it.first(); env != NULL; env = it.next(env)) {
      if (!env->is_retransformable() && env->is_enabled(JVMTI_EVENT_CLASS_FILE_LOAD_HOOK)) {
        // non-retransformable agents cannot retransform back,
        // so no need to cache the original class file bytes
        post_to_env(env, false);
      }
    }
  }
  JvmtiEnvIterator it;
  for (JvmtiEnv* env = it.first(); env != NULL; env = it.next(env)) {
    // retransformable agents get all events
    if (env->is_retransformable() && env->is_enabled(JVMTI_EVENT_CLASS_FILE_LOAD_HOOK)) {
      // retransformable agents need to cache the original class file bytes
      // if changes are made via the ClassFileLoadHook
      post_to_env(env, true);
    }
  }
}

void post_to_env(JvmtiEnv* env, bool caching_needed) {
  unsigned char *new_data = NULL;
  jint new_len = 0;

  JvmtiClassFileLoadEventMark jem(_thread, _h_name, _class_loader,
                                  _h_protection_domain,
                                  _h_class_being_redefined);
  JvmtiJavaThreadEventTransition jet(_thread);
  JNIEnv* jni_env =  (JvmtiEnv::get_phase() == JVMTI_PHASE_PRIMORDIAL) ? NULL : jem.jni_env();
  jvmtiEventClassFileLoadHook callback = env->callbacks()->ClassFileLoadHook;
  if (callback != NULL) {
    (*callback)(env->jvmti_external(), jni_env,
                jem.class_being_redefined(),
                jem.jloader(), jem.class_name(),
                jem.protection_domain(),
                _curr_len, _curr_data,
                &new_len, &new_data);
  }
  if (new_data != NULL) {
    // this agent has modified class data.
    if (caching_needed && *_cached_class_file_ptr == NULL) {
      // data has been changed by the new retransformable agent
      // and it hasn't already been cached, cache it
      JvmtiCachedClassFileData *p;
      p = (JvmtiCachedClassFileData *)os::malloc(
        offset_of(JvmtiCachedClassFileData, data) + _curr_len, mtInternal);

      p->length = _curr_len;
      memcpy(p->data, _curr_data, _curr_len);
      *_cached_class_file_ptr = p;
    }

    if (_curr_data != *_data_ptr) {
      // curr_data is previous agent modified class data.
      // And this has been changed by the new agent so
      // we can delete it now.
      _curr_env->Deallocate(_curr_data);
    }

    // Class file data has changed by the current agent.
    _curr_data = new_data;
    _curr_len = new_len;
    // Save the current agent env we need this to deallocate the
    // memory allocated by this agent.
    _curr_env = env;
  }
}
```

### 13.5 示例二：First Input

#### Agent Jar

##### LoadTimeAgent

```java
package lsieun.agent;

import lsieun.asm.visitor.MethodInfo;
import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.util.EnumSet;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要处理的类
    String className = "sample.HelloWorld";

    // 第三步，使用inst：添加incapable transformer
    ClassFileTransformer transformer1 = new VersatileTransformer(className, EnumSet.of(MethodInfo.PARAMETER_VALUES));
    ClassFileTransformer transformer2 = new VersatileTransformer(className, EnumSet.of(MethodInfo.NAME_AND_DESC));
    inst.addTransformer(transformer1, false);
    inst.addTransformer(transformer2, false);

    // 第四步，使用inst：添加capable transformer
    ClassFileTransformer transformer3 = new VersatileTransformer(className, EnumSet.of(MethodInfo.THREAD_INFO));
    inst.addTransformer(transformer3, true);
    ClassFileTransformer transformer4 = new VersatileTransformer(className, EnumSet.of(MethodInfo.CLASSLOADER));
    inst.addTransformer(transformer4, true);
    ClassFileTransformer transformer5 = new DumpTransformer(className);
    inst.addTransformer(transformer5, true);

    // 第五步，加载目标类 define
    try {
      System.out.println("load class: " + className);
      Class<?> clazz = Class.forName(className);
      System.out.println("load success");
    } catch (ClassNotFoundException e) {
      System.out.println("load failed");
      e.printStackTrace();
    }

    // 第六步，使用inst：移除transformer
    inst.removeTransformer(transformer3);
  }
}
```

##### DynamicAgent

```java
package lsieun.agent;

import lsieun.utils.*;

import java.io.InputStream;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，指定要处理的类
    String className = "sample.HelloWorld";

    // 第三步，使用inst：进行redefine操作
    try {
      Class<?> clazz = Class.forName(className);
      if (inst.isModifiableClass(clazz)) {
        InputStream in = LoadTimeAgent.class.getResourceAsStream("/sample/HelloWorld.class");
        int available = in.available();
        byte[] bytes = new byte[available];
        in.read(bytes);
        ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
        inst.redefineClasses(classDefinition);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }

    // 第三步，使用inst：进行re-transform操作
    try {
      Class<?> clazz = Class.forName(className);
      if (inst.isModifiableClass(clazz)) {
        inst.retransformClasses(clazz);
      }
    } catch (Exception ex) {
      ex.printStackTrace();
    }
  }
}
```

#### Run

##### define

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

load class: sample.HelloWorld
transform class: sample/HelloWorld with [PARAMETER_VALUES]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
transform class: sample/HelloWorld with [NAME_AND_DESC]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
load success
transform class: sample/HelloWorld with [THREAD_INFO]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
transform class: sample/HelloWorld with [CLASSLOADER]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
```

##### redefine

```shell
transform class: sample/HelloWorld with [PARAMETER_VALUES]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
transform class: sample/HelloWorld with [NAME_AND_DESC]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
transform class: sample/HelloWorld with [CLASSLOADER]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
```

##### retransform

```shell
transform class: sample/HelloWorld with [CLASSLOADER]
---> sample/HelloWorld.add:(II)I
---> sample/HelloWorld.sub:(II)I
```

### 13.6 总结

本文内容总结如下：

- 第一点，如何实现`ClassFileTransformer`接口，应考虑哪些事情。
- 第二点，一个transformer关注的内容有两个：
  - Transformer的分类：retransform capable transformer和retransform incapable transformer
  - **Transformer被JVM调用的三个时机：define、redefine和retransform**
- 第三点，多个transformer关注的内容也有两个：
  - 在多个transformer的情况下，它们的前后调用关系：串联执行，前面的transformer输出，成为后面transformer的输入；遇到transformer异常，相当于返回`null`值，不影响后续transformer执行。
  - 在多个transformer的情况下，第一个transformer接收到`classfileBuffer`到底是什么呢？在三种不同的时机下，它的值是不同的。

## 14. All In One Examples

### 14.1 Application

#### Program

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // 第一步，打印进程ID
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // 第二步，倒计时退出
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      Random rand = new Random(System.currentTimeMillis());
      int a = rand.nextInt(10);
      int b = rand.nextInt(10);
      boolean flag = rand.nextBoolean();
      String message;
      if (flag) {
        message = String.format("a + b = %d", HelloWorld.add(a, b));
      }
      else {
        message = String.format("a - b = %d", HelloWorld.sub(a, b));
      }
      System.out.println(message);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### HelloWorld

```java
package sample;

public class HelloWorld extends Object implements Cloneable {
  public int intValue;
  public String strValue;

  public static int add(int a, int b) {
    return a + b;
  }

  public static int sub(int a, int b) {
    return a - b;
  }

  @Override
  public Object clone() throws CloneNotSupportedException {
    return super.clone();
  }
}
```

### 14.2 Agent Jar

#### define

加载某个类时（define时），即对该类做修改

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "sample.HelloWorld";

    // 第三步，使用inst：添加transformer
    ClassFileTransformer transformer = new ASMTransformer(className);
    inst.addTransformer(transformer, false);
  }
}
```

#### redefine

某个类已经加载后（预先调用了`Class.forName(className)`），再告知JVM用新的字节码数据`byte[]`替换该类（redefine）

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.io.InputStream;
import java.lang.instrument.ClassDefinition;
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "sample.HelloWorld";

    // 第三步，使用inst：进行redefine操作
    // ClassFileTransformer transformer = new StackTraceTransformer(className);
    // inst.addTransformer(transformer, true);
    try {
      Class<?> clazz = Class.forName(className);
      if (inst.isModifiableClass(clazz)) {
        String item = String.format("/%s.class", className.replace(".", "/"));
        System.out.println(item);
        InputStream in = LoadTimeAgent.class.getResourceAsStream(item);
        int available = in.available();
        byte[] bytes = new byte[available];
        in.read(bytes);
        ClassDefinition classDefinition = new ClassDefinition(clazz, bytes);
        inst.redefineClasses(classDefinition);
      }
    } catch (Exception ex) {
      ex.printStackTrace();
    }
  }
}
```

#### retransform

某个类已经加载后（预先调用了`Class.forName(className)`），在该类原先字节码`byte[]`基础上，对该类继续做修改

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，指定要修改的类
    String className = "sample.HelloWorld";

    // 第三步，使用inst：进行re-transform操作
    ClassFileTransformer transformer = new ASMTransformer(className);
    inst.addTransformer(transformer, true);
    try {
      Class<?> clazz = Class.forName(className);
      if (inst.isModifiableClass(clazz)) {
        inst.retransformClasses(clazz);
      }
    } catch (Exception ex) {
      ex.printStackTrace();
    } finally {
      inst.removeTransformer(transformer);
    }
  }
}
```

### 14.3 Run

可以看出来redefine和retransform还是比较有局限性的

|                    | define | redefine | retransform |
| ------------------ | ------ | -------- | ----------- |
| Interface Add      | OK     | NO       | NO          |
| Field Add          | OK     | NO       | NO          |
| Method Add         | OK     | NO       | NO          |
| Method Remove      | OK     | NO       | NO          |
| Method Body Modify | OK     | OK       | OK          |

#### Interface Add

在 `ASMTransformer` 当中，修改代码：

```java
ClassVisitor cv = new AddInterfaceVisitor(cw, "java/io/Serializable");
```

在 define 的情况下，正常运行；在 redefine 和 retransform 的情况下，则出现 `UnsupportedOperationException` 异常：

```shell
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change superclass or interfaces
```

#### Field Add

在 `ASMTransformer` 当中，修改代码：

```java
ClassVisitor cv = new AddFiledVisitor(cw, Opcodes.ACC_PUBLIC, "objValue", "Ljava/lang/Object;");
```

在define的情况下，正常运行；在redefine和retransform的情况下，则出现 `UnsupportedOperationException` 异常：

```shell
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the schema (add/remove fields)
```

#### Method Add

在 `ASMTransformer` 当中，修改代码：

```java
ClassVisitor cv = new AddMethodVisitor(cw, Opcodes.ACC_PUBLIC, "mul", "(II)I", null, null) {
  @Override
  protected void generateMethodBody(MethodVisitor mv) {
    mv.visitCode();
    mv.visitVarInsn(Opcodes.ILOAD, 1);
    mv.visitVarInsn(Opcodes.ILOAD, 2);
    mv.visitInsn(Opcodes.IMUL);
    mv.visitInsn(Opcodes.IRETURN);
    mv.visitMaxs(2, 3);
    mv.visitEnd();
  }
};
```

在define的情况下，正常运行；在redefine和retransform的情况下，则出现 `UnsupportedOperationException` 异常：

```shell
java.lang.UnsupportedOperationException: class redefinition failed: attempted to add a method
```

#### Method Remove

在 `ASMTransformer` 当中，修改代码：

```java
ClassVisitor cv = new RemoveMethodVisitor(cw, "sub", "(II)I");
```

在define的情况下，正常运行；在redefine和retransform的情况下，则出现 `UnsupportedOperationException` 异常：

```shell
java.lang.UnsupportedOperationException: class redefinition failed: attempted to delete a method
```

#### Method Body Modify

在 `ASMTransformer` 当中，修改代码：

```java
ClassVisitor cv = new PrintMethodParameterVisitor(cw);
```

在define的情况下，正常运行；在redefine和retransform的情况下，也正常运行。

#### Stack Trace

> [利用ClassLoader#defineClass动态加载字节码_classloader defineclass_Thunderclap_的博客-CSDN博客](https://blog.csdn.net/Thunderclap_/article/details/128914911)
>
> [Java中的ClasLoader之自定义ClassLoader (baidu.com)](https://baijiahao.baidu.com/s?id=1698071570748924792&wfr=spider&for=pc)
>
> [假笨说-谨防JDK8重复类定义造成的内存泄漏 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/440073760)
>
> [类加载时JVM在搞什么？JVM源码分析+OOP-KLASS模型分析_躺平程序猿的博客-CSDN博客](https://blog.csdn.net/yangxiaofei_java/article/details/118469738) <= 推荐阅读
>
> [类加载器一篇足以 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/520521579) <= 推荐阅读
>
> 类加载机制的基本特征
>
> - 双亲委派模型。但是不是所有类加载都遵守这个模型，有时候，启动类加载器所加载的类型，是可能需要加载用户代码的比如SIP机制，具体如JDBC的驱动发现等，在这种情况下就不会用双亲委派模型去加载了，而是利用线程上下文类加载器去打破它（默认的线程上下文类加载器就是系统类加载器）。
> - 可见性，子类加载器可以访问父类加载器加载的类型，但是反过来是不被允许的
> - 单一性，由于父加载器加载的类对于子类加载器是可见的，所以父加载器中加载过的类型，就不会再子加载器中重复加载。但是类加载器"邻居"间（MyClassloader的两个实例），同一类型可以被多次加载，因为互相并不可见。

将 `ASMTransformer` 类替换成 `StackTraceTransformer` 类。

在 define 情况，从下面的输出结果可以看到是 `ClassLoader.defineClass()` 方法触发的：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

java.lang.Exception: Exception From lsieun.instrument.StackTraceTransformer
        at lsieun.instrument.StackTraceTransformer.transform(StackTraceTransformer.java:23)
        at sun.instrument.TransformerManager.transform(TransformerManager.java:188)
        at sun.instrument.InstrumentationImpl.transform(InstrumentationImpl.java:428)
        at java.lang.ClassLoader.defineClass1(Native Method)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:756)
        at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
        at java.net.URLClassLoader.defineClass(URLClassLoader.java:468)
        at java.net.URLClassLoader.access$100(URLClassLoader.java:74)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:369)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:363)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:362)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
        at sample.Program.main(Program.java:25)
```

在 redefine 情况，从下面的输出结果可以看到是 `InstrumentationImpl.redefineClasses()` 方法触发的：

```shell
java.lang.Exception: Exception From lsieun.instrument.StackTraceTransformer
        at lsieun.instrument.StackTraceTransformer.transform(StackTraceTransformer.java:23)
        at sun.instrument.TransformerManager.transform(TransformerManager.java:188)
        at sun.instrument.InstrumentationImpl.transform(InstrumentationImpl.java:428)
        at sun.instrument.InstrumentationImpl.redefineClasses0(Native Method)
        at sun.instrument.InstrumentationImpl.redefineClasses(InstrumentationImpl.java:170)
        at lsieun.agent.DynamicAgent.agentmain(DynamicAgent.java:32)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
```

在 retransform 情况，从下面的输出结果可以看到是 `InstrumentationImpl.retransformClasses()` 方法触发的：

```shell
java.lang.Exception: Exception From lsieun.instrument.StackTraceTransformer
        at lsieun.instrument.StackTraceTransformer.transform(StackTraceTransformer.java:23)
        at sun.instrument.TransformerManager.transform(TransformerManager.java:188)
        at sun.instrument.InstrumentationImpl.transform(InstrumentationImpl.java:428)
        at sun.instrument.InstrumentationImpl.retransformClasses0(Native Method)
        at sun.instrument.InstrumentationImpl.retransformClasses(InstrumentationImpl.java:144)
        at lsieun.agent.DynamicAgent.agentmain(DynamicAgent.java:42)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
```

## 15. 总结

### 15.1 两种启动方式

进行 Load-Time Instrumentation，需要从命令行启动 Java Agent 需要使用 `-javagent` 选项：

```shell
-javaagent:jarpath[=options]
```

进行 Dynamic Instrumentation ，需要使用到 JVM 的 Attach 机制。

### 15.2 Agent Jar的三个组成部分

在 Agent Jar 当中有三个主要组成部分：

![Agent Jar中的三个组成部分](https://lsieun.github.io/assets/images/java/agent/agent-jar-three-components.png)

在 Manifest 文件中，与 Java Agent 相关的属性有7个：

```pseudocode
                                       ┌─── Premain-Class
                       ┌─── Basic ─────┤
                       │               └─── Agent-Class
                       │
                       │               ┌─── Can-Redefine-Classes
                       │               │
Manifest Attributes ───┼─── Ability ───┼─── Can-Retransform-Classes
                       │               │
                       │               └─── Can-Set-Native-Method-Prefix
                       │
                       │               ┌─── Boot-Class-Path
                       └─── Special ───┤
                                       └─── Launcher-Agent-Class
```

在 Agent Class 当中，可以定义 `premain` 和 `agentmain` 方法：

```java
public static void premain(String agentArgs, Instrumentation inst);

public static void agentmain(String agentArgs, Instrumentation inst);
```

### 15.3 Instrumentation API

在 `java.lang.instrument` 最重要的三个类型：

```pseudocode
                        ┌─── Instrumentation (接口)
                        │
java.lang.instrument ───┼─── ClassFileTransformer (接口)
                        │
                        └─── ClassDefinition (类)
```

其中，`Instrumentation` 接口的方法可以分成不同的类别：

```pseudocode
                                                         ┌─── isRedefineClassesSupported()
                                                         │
                                     ┌─── ability ───────┼─── isRetransformClassesSupported()
                                     │                   │
                   ┌─── Agent Jar ───┤                   └─── isNativeMethodPrefixSupported()
                   │                 │
                   │                 │                   ┌─── addTransformer()
                   │                 └─── transformer ───┤
                   │                                     └─── removeTransformer()
                   │
                   │                                     ┌─── appendToBootstrapClassLoaderSearch()
                   │                 ┌─── classloader ───┤
                   │                 │                   └─── appendToSystemClassLoaderSearch()
Instrumentation ───┤                 │
                   │                 │                                         ┌─── loading ───┼─── transform
                   │                 │                                         │
                   │                 │                   ┌─── status ──────────┤                                  ┌─── getAllLoadedClasses()
                   │                 │                   │                     │               ┌─── get ──────────┤
                   │                 │                   │                     │               │                  └─── getInitiatedClasses()
                   │                 │                   │                     └─── loaded ────┤
                   │                 │                   │                                     │                  ┌─── isModifiableClass()
                   │                 ├─── class ─────────┤                                     │                  │
                   └─── target VM ───┤                   │                                     └─── modifiable ───┼─── redefineClasses()
                                     │                   │                                                        │
                                     │                   │                                                        └─── retransformClasses()
                                     │                   │
                                     │                   │                     ┌─── isNativeMethodPrefixSupported()
                                     │                   └─── native method ───┤
                                     │                                         └─── setNativeMethodPrefix()
                                     │
                                     ├─── object ────────┼─── getObjectSize()
                                     │
                                     │                   ┌─── isModifiableModule()
                                     └─── module ────────┤
                                                         └─── redefineModule()
```

我们可以实现 `ClassFileTransformer` 接口中的 `transform()` 方法可以对具体的 ClassFile 进行转换：

```java
public interface ClassFileTransformer {
  byte[] transform(ClassLoader         loader,
                   String              className,
                   Class<?>            classBeingRedefined,
                   ProtectionDomain    protectionDomain,
                   byte[]              classfileBuffer)
    throws IllegalClassFormatException;
}
```

那么，`ClassFileTransformer.transform` 方法会在什么时候被调用呢？

```pseudocode
                               ┌─── define: ClassLoader.defineClass
               ┌─── loading ───┤
               │               └─── transform
class state ───┤
               │               ┌─── redefine: Instrumentation.redefineClasses
               └─── loaded ────┤
                               └─── retransform: Instrumentation.retransformClasses
```

在 define、redefine 和 retransform 的情况下，会触发哪些 transformer：

![img](https://lsieun.github.io/assets/images/java/agent/define-redefine-retransform.png)

# 第四章 应用与技巧

## 1. Load-Time VS. Dynamic Agent

### 1.1 虚拟机数量

Load-Time Instrumentation只涉及到一个虚拟机：

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-load-time-instrumentation.png)

Dynamic Instrumentation涉及到两个虚拟机：

![img](https://lsieun.github.io/assets/images/java/agent/virtual-machine-of-dynamic-instrumentation.png)

### 1.2 时机不同

在进行Load-Time Instrumentation时，会执行Agent Jar当中的`premain()`方法；`premain()`方法是先于`main()`方法执行，此时Application当中使用的**大多数类还没有被加载**。

在进行Dynamic Instrumentation时，会执行Agent Jar当中的`agentmain()`方法；而`agentmain()`方法是往往是在`main()`方法之后执行，此时Application当中使用的**大多数类已经被加载**。

### 1.3 能力不同

**Load-Time Instrumentation可以做很多事情：添加和删除字段、添加和删除方法等**。

Dynamic Instrumentation做的事情比较有限，大多集中在对于方法体的修改。

> 个人理解，Load-Time Instrumentation基本都是在类第一次被加载前就做修改，即属于define的步骤，而不是第N次类加载(N>1)的redefine和retransform，所以JVM允许更大幅度的修改。这时候修改，某种意义上就和临时改java源代码重新生成class文件一样，毕竟不管怎么修改，这里最后修改的class文件(byte[])都是头一次实际被JVM加载成java类(元空间Klass类对象，堆空间Class类对象)
>
> [类加载时JVM在搞什么？JVM源码分析+OOP-KLASS模型分析_躺平程序猿的博客-CSDN博客](https://blog.csdn.net/yangxiaofei_java/article/details/118469738)

### 1.4 线程不同

Load-Time Instrumentation是运行在`main`线程里：

```shell
Thread Id: main@1(false)
```

```java
package lsieun.agent;

import lsieun.utils.*;

import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);
  }
}
```

**Dynamic Instrumentation是运行在`Attach Listener`线程里**：

```shell
Thread Id: Attach Listener@5(true)
```

```java
package lsieun.agent;

import lsieun.utils.*;

import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);
  }
}
```

### 1.5 Exception处理

在处理Exception的时候，Load-Time Instrumentation和Dynamic Instrumentation有差异： [Link](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

- 当Load-Time Instrumentation时，出现异常，会报告错误信息，并且停止执行，退出虚拟机。（毕竟premain在main之前执行）
- 当Dynamic Instrumentation时，出现异常，会报告错误信息，但是不会停止虚拟机，而是继续执行。（毕竟agentmain在main之后执行）

在Load-Time Instrumentation过程中，遇到异常：

- If the agent cannot be resolved (for example, because the agent class cannot be loaded, or because the agent class does not have an appropriate `premain` method), **the JVM will abort**.
- If a `premain` method throws an uncaught exception, **the JVM will abort**.

在Dynamic Instrumentation过程中，遇到异常：

- If the agent cannot be started (for example, because the agent class cannot be loaded, or because the agent class does not have a conformant `agentmain` method), **the JVM will not abort**.
- If the `agentmain` method throws an uncaught exception, **it will be ignored**.

#### Agent Class不存在

##### Load-Time Agent

在`pom.xml`中，修改`Premain-Class`属性，指向一个不存在的Agent Class：

```xml
<Premain-Class>lsieun.agent.NonExistentAgent</Premain-Class>
```

会出现`ClassNotFoundException`异常：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Exception in thread "main" java.lang.ClassNotFoundException: lsieun.agent.NonExistentAgent
        at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:304)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
FATAL ERROR in native method: processing of -javaagent failed
```

##### Dynamic Agent

在`pom.xml`中，修改`Agent-Class`属性，指向一个不存在的Agent Class：

```xml
<Agent-Class>lsieun.agent.NonExistentAgent</Agent-Class>
```

会出现`ClassNotFoundException`异常：

```shell
Exception in thread "Attach Listener" java.lang.ClassNotFoundException: lsieun.agent.NonExistentAgent
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:304)
	at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
Agent failed to start!
```

#### incompatible xxx-main

##### Load-Time Agent

如果`premain`方法不符合规范：

```java
public class LoadTimeAgent {
  public static void premain() {
    // do nothing
  }
}
```

会出现`NoSuchMethodException`异常：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Exception in thread "main" java.lang.NoSuchMethodException: lsieun.agent.LoadTimeAgent.premain(java.lang.String, java.lang.instrument.Instrumentation)
        at java.lang.Class.getDeclaredMethod(Class.java:2130)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:327)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
FATAL ERROR in native method: processing of -javaagent failed
```

##### Dynamic Agent

如果`agentmain`方法不符合规范：

```java
public class DynamicAgent {
  public static void agentmain() {
    // do nothing
  }
}
```

会出现`NoSuchMethodException`异常：

```shell
Exception in thread "Attach Listener" java.lang.NoSuchMethodException: lsieun.agent.DynamicAgent.agentmain(java.lang.String, java.lang.instrument.Instrumentation)
	at java.lang.Class.getDeclaredMethod(Class.java:2130)
	at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:327)
	at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
Agent failed to start!
```

#### 抛出异常

##### Load-Time Agent

```java
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    throw new RuntimeException("exception from LoadTimeAgent");
  }
}
```

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
Exception in thread "main" java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
        at sun.instrument.InstrumentationImpl.loadClassAndCallPremain(InstrumentationImpl.java:401)
Caused by: java.lang.RuntimeException: exception from LoadTimeAgent
        at lsieun.agent.LoadTimeAgent.premain(LoadTimeAgent.java:7)
        ... 6 more
FATAL ERROR in native method: processing of -javaagent failed
```

##### Dynamic Agent

```java
import java.lang.instrument.Instrumentation;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) {
    throw new RuntimeException("exception from DynamicAgent");
  }
}
```

```shell
Exception in thread "Attach Listener" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:386)
	at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
Caused by: java.lang.RuntimeException: exception from DynamicAgent
	at lsieun.agent.DynamicAgent.agentmain(DynamicAgent.java:7)
	... 6 more
Agent failed to start!
```

### 1.6 总结

本文内容总结如下：

- 第一点，虚拟机的数量不同。
- 第二点，时机不同和能力不同。
- 第三点，线程不同。
- 第四点，处理异常的方式不同。

## 2. None Instrumentation

### 2.1 设置Property

有的时候，需要在命令行设置一些属性信息；当属性信息比较多的时候，命令行的内容就特别长。我们可以使用一个Agent Jar来设置这些属性信息。

#### Application

```java
package sample;

public class Program {
  public static void main(String[] args) {
    String username = System.getProperty("lsieun.agent.username");
    String password = System.getProperty("lsieun.agent.password");
    System.out.println(username);
    System.out.println(password);
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

public class LoadTimeAgent {
  public static void premain(String agentArgs) {
    System.setProperty("lsieun.agent.username", "tomcat");
    System.setProperty("lsieun.agent.password", "123456");
  }
}
```

#### Run

第一次运行：

```shell
$ java -cp ./target/classes/ sample.Program

null
null
```

第二次运行：

```shell
$ java -cp ./target/classes/ -Dlsieun.agent.username=jerry -Dlsieun.agent.password=12345 sample.Program

jerry
12345
```

第三次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

tomcat
123456
```

### 2.2 不打印信息

有的时候，程序当中有许多打印语句，但是我们并不想让它们输出。

#### Application

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // 第一步，打印进程ID
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // 第二步，倒计时退出
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### Agent Jar

```java
package lsieun.agent;

import java.io.PrintStream;

public class LoadTimeAgent {
  public static void premain(String agentArgs) {
    System.setOut(new PrintStream(System.out) {
      @Override
      public void println(String x) {
        // super.println("What are you doing: " + x);
      }
    });
  }
}
```

#### Run

第一次运行：

```shell
$ java -cp ./target/classes/ sample.Program

8472@LenovoWin7
|000| 8472@LenovoWin7 remains 600 seconds
|001| 8472@LenovoWin7 remains 599 seconds
```

第二次运行：（没有输出内容）

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program
```

第三次运行：（取消注释）

```shell
$ java -cp ./target/classes/ -javaagent:./target/TheAgent.jar sample.Program

What are you doing: 9072@LenovoWin7
What are you doing: |000| 9072@LenovoWin7 remains 600 seconds
What are you doing: |001| 9072@LenovoWin7 remains 599 seconds
```

## 3. Multiple Agents

### 3.1 Multiple Agents

#### Load-Time

The `-javaagent` switch may be used multiple times on the same command-line, thus creating **multiple agents**. More than one agent may use the same jarpath. [Link](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

```shell
-javaagent:jarpath[=options]
```

#### Dynamic

在[java.lang.instrument API](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)文档中， 并没有明确提到使用多个Dynamic Agent，但是我们可以进行测试。

#### 顺序执行

当有多个Agent Jar时，它们是按先后顺序执行，还是以某种随机、不固定的方式执行呢？

不管是Load-Time Instrumentation，还是Dynamic Instrumentation，多个Agent Jar是按照加载的先后顺序执行。

After the Java Virtual Machine (JVM) has initialized, each `premain` method will be called in the order the agents were specified, then the real application `main` method will be called.

Each `premain` method must return in order for the startup sequence to proceed.

### 3.2 不同视角

#### ClassLoader视角

##### Load-Time

The agent class will be loaded by the **system class loader**. This is the class loader which typically loads the class containing the application `main` method. The `premain` methods will be run under the same security and classloader rules as the application `main` method.

There are no modeling restrictions on what the agent `premain` method may do. Anything application `main` can do, including **creating threads**, is legal from `premain`.

##### Dynamic

The agent JAR is appended to the **system class path**. This is the class loader that typically loads the class containing the application `main` method. The agent class is loaded and the JVM attempts to invoke the `agentmain` method.

#### Thread视角

在Load-Time Instrumentation情况下，多个Agent Class运行在`main`线程当中。

在Dynamic Instrumentation情况下，多个Agent Class运行在`Attach Listener`线程当中。

**如果某一个Agent Jar的`premain()`或`agentmain()`方法不退出，后续的Agent Jar执行不了**。

#### Instrumentation实例

**JVM加载每一个的Agent Jar都有一个属于自己的`Instrumentation`实例。每一个`Instrumentation`实例管理自己的`ClassFileTransformer`**。

![img](https://lsieun.github.io/assets/images/java/agent/multi-agent-jar.png)

### 3.3 示例：多个Agent Jar

#### Application

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // 第一步，打印进程ID
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // 第二步，倒计时退出
    int count = 600;
    for (int i = 0; i < count; i++) {
      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### Agent Jar

如果手工的生成一个一个的Agent Jar文件，会比较麻烦一些。 那么，我们可以写一个类的“模板”，然后借助于ASM修改类的名字，然后生成多个Agent Jar文件。

```java
package lsieun.agent;

import lsieun.utils.PrintUtils;

import java.lang.instrument.Instrumentation;
import java.util.concurrent.TimeUnit;

public class TemplateAgent {
  public static void premain(String agentArgs, Instrumentation inst) throws InterruptedException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(TemplateAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，睡10秒钟
    for (int i = 0; i < 10; i++) {
      String message = String.format("%s: %03d", TemplateAgent.class.getSimpleName(), i);
      System.out.println(message);
      TimeUnit.SECONDS.sleep(1);
    }
  }

  public static void agentmain(String agentArgs, Instrumentation inst) throws InterruptedException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(TemplateAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，睡30秒钟
    for (int i = 0; i < 30; i++) {
      String message = String.format("%s: %03d", TemplateAgent.class.getSimpleName(), i);
      System.out.println(message);
      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### Run

##### Load-Time

```
$ java -cp ./target/classes -javaagent:./target/TemplateAgent001.jar -javaagent:./target/TemplateAgent002.jar sample.Program

========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.TemplateAgent001
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1550089733
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

TemplateAgent001: 000
TemplateAgent001: 001
...
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.TemplateAgent002
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@2101973421
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

TemplateAgent002: 000
TemplateAgent002: 001
...
```

##### Dynamic

```
java -cp "${JAVA_HOME}/lib/tools.jar"\;./target/classes run.instrument.DynamicInstrumentation ./target/TemplateAgent001.jar
java -cp "${JAVA_HOME}/lib/tools.jar"\;./target/classes run.instrument.DynamicInstrumentation ./target/TemplateAgent002.jar
java -cp "${JAVA_HOME}/lib/tools.jar"\;./target/classes run.instrument.DynamicInstrumentation ./target/TemplateAgent003.jar
```

```
$ java -cp ./target/classes/ sample.Program
...
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Agent-Class: lsieun.agent.TemplateAgent001
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1582797393
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: Attach Listener@5(true)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@73d16e93
========= ========= ========= SEPARATOR ========= ========= =========

TemplateAgent001: 000
TemplateAgent001: 001
...
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Agent-Class: lsieun.agent.TemplateAgent002
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1760363924
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: Attach Listener@5(true)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@73d16e93
========= ========= ========= SEPARATOR ========= ========= =========

TemplateAgent002: 000
TemplateAgent002: 001
...
```

### 3.4 总结

本文内容总结如下：

- 第一点，**不管是Load-Time Instrumentation情况，还是Dynamic Instrumentation的情况，多个Agent Jar可以一起使用的，它们按照加载的先后顺序执行**。
- 第二点，从不同的视角来理解多个Agent Jar的运行
  - 从ClassLoader的视角来说，它们都是使用system classloader加载。
  - 从Thread的视角来说，Load-Time Agent Class运行在`main`线程里，Dynamic Agent Class运行在`Attach Listener`线程里。
  - **从Instrumentation实例的视角来说，JVM对于每一个加载的Agent Jar都有一个属于自己的`Instrumentation`实例**。

## 4. Multiple Agents: Sandwich

### 4.1 三明治

在数学的概念当中，有一个迫敛定理或三明治定理（英文：Squeeze Theorem、Sandwich Theorem），可以帮助我们确定某一点的函数值到底是多少：

![img](https://lsieun.github.io/assets/images/java/agent/sandwich-theorem.png)

这种“三明治”思路可以应用到Multiple Agents当中，这样我们就可以检测某一个Agent Jar修改了什么内容。

但是，需要注意的一点是：属于同一组的transformer才能进行这种“三明治”操作

- 同属于retransformation incapable transformer（define 或 redefine使用），
- 同属于retransformation capable transformer（define或redefine时经历到，retransform使用）。

### 4.2 示例

#### Agent Jar 1

##### LoadTimeAgent

```java
package lsieun.agent;

import lsieun.instrument.*;
import lsieun.utils.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class LoadTimeAgent {
  public static void premain(String agentArgs, Instrumentation inst) {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(LoadTimeAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，解析参数：agentArgs
    boolean flag = Boolean.parseBoolean(agentArgs);

    // 第三步，使用inst：添加transformer
    ClassFileTransformer transformer = new SandwichTransformer(flag);
    inst.addTransformer(transformer, false);
  }
}
```

##### SandwichTransformer

```java
package lsieun.instrument;

import lsieun.utils.DateUtils;
import lsieun.utils.DumpUtils;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class SandwichTransformer implements ClassFileTransformer {
  private final boolean compare;

  public SandwichTransformer(boolean compare) {
    this.compare = compare;
  }

  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    if (!compare) {
      addClass(className, classfileBuffer);
    }
    else {
      compareClass(className, classfileBuffer);
    }
    return null;
  }


  private static final Map<String, byte[]> map = new HashMap<>();

  private static void addClass(String className, byte[] bytes) {
    map.put(className, bytes);
  }

  private static void compareClass(String className, byte[] bytes) {
    byte[] origin_bytes = map.get(className);
    if (origin_bytes == null) return;
    boolean isEqual = Arrays.equals(origin_bytes, bytes);
    if (isEqual) {
      map.remove(className);
      return;
    }

    String newName = className.replace('/', '.');
    String dateStr = DateUtils.getTimeStamp();
    String filenameA = String.format("%s.%s.%s.class", newName, dateStr, "A");
    String filenameB = String.format("%s.%s.%s.class", newName, dateStr, "B");
    DumpUtils.dump(filenameA, origin_bytes);
    DumpUtils.dump(filenameB, bytes);
    System.out.println("Diff: " + filenameA);
    System.out.println("Diff: " + filenameB);
  }
}
```

#### Agent Jar 2

##### TemplateAgent

```java
package lsieun.agent;

import lsieun.utils.PrintUtils;
import lsieun.utils.TransformerUtils;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class TemplateAgent {
  public static void premain(String agentArgs, Instrumentation inst) throws InterruptedException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(TemplateAgent.class, "Premain-Class", agentArgs, inst);

    // 第二步，指明要处理的类
    TransformerUtils.internalName = "sample/HelloWorld";

    // 第三步，使用inst：添加transformer
    ClassFileTransformer transformer = TransformerUtils::enterMethod;
    inst.addTransformer(transformer, false);
  }

  public static void agentmain(String agentArgs, Instrumentation inst) throws InterruptedException {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(TemplateAgent.class, "Agent-Class", agentArgs, inst);
  }
}
```

#### Run

```shell
$ java -cp ./target/classes/ \
  -javaagent:./target/TheAgent.jar=false \
  -javaagent:./target/TemplateAgent001.jar \
  -javaagent:./target/TheAgent.jar=true \
  sample.Program
  
========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: false
    (3) Instrumentation: sun.instrument.InstrumentationImpl@1704856573
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.TemplateAgent001
    (2) agentArgs: null
    (3) Instrumentation: sun.instrument.InstrumentationImpl@21685669
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

========= ========= ========= SEPARATOR ========= ========= =========
Agent Class Info:
    (1) Premain-Class: lsieun.agent.LoadTimeAgent
    (2) agentArgs: true
    (3) Instrumentation: sun.instrument.InstrumentationImpl@764977973
    (4) Can-Redefine-Classes: true
    (5) Can-Retransform-Classes: true
    (6) Can-Set-Native-Method-Prefix: true
    (7) Thread Id: main@1(false)
    (8) ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
========= ========= ========= SEPARATOR ========= ========= =========

10012@LenovoWin7
|000| 10012@LenovoWin7 remains 600 seconds
transform class: sample/HelloWorld
file:///D:\git-repo\learn-java-agent\dump\sample.HelloWorld.2022.02.03.04.14.09.089.A.class
file:///D:\git-repo\learn-java-agent\dump\sample.HelloWorld.2022.02.03.04.14.09.089.B.class
```

### 4.3 总结

本文内容总结如下：

- 第一点，通过“三明治”的方式，我们可以检测某一个Agent Jar做了哪些修改。
- 第二点，使用“三明治”的方式，要注意transformer属于同一组当中。

## 5. Self Attach

有些情况下，我们想在当前的JVM当中获得一个`Instrumentation`实例。

### 5.1 获取VM PID

#### Pre Java 9

代码片段：

```java
String jvmName = ManagementFactory.getRuntimeMXBean().getName();
String jvmPid = jvmName.substring(0, jvmName.indexOf('@'));
```

完整代码：

```java
import java.lang.management.ManagementFactory;

public class HelloWorld {
  public static void main(String[] args) {
    String jvmName = ManagementFactory.getRuntimeMXBean().getName();
    String jvmPid = jvmName.substring(0, jvmName.indexOf('@'));
    System.out.println(jvmName);
    System.out.println(jvmPid);
  }
}
```

运行结果：

```shell
5452@LenovoWin7
5452
```

#### Java 9

In Java 9 the `java.lang.ProcessHandle` can be used:

```java
long pid = ProcessHandle.current().pid();
import java.lang.management.ManagementFactory;

public class HelloWorld {
  public static void main(String[] args) {
    String jvmName = ManagementFactory.getRuntimeMXBean().getName();
    String jvmPid = jvmName.substring(0, jvmName.indexOf('@'));
    System.out.println(jvmName);
    System.out.println(jvmPid);

    long pid = ProcessHandle.current().pid();
    System.out.println(pid);
  }
}
```

Output:

```shell
9836@LenovoWin7
9836
9836
```

### 5.2 示例

#### Application

```java
package sample;

import lsieun.agent.SelfAttachAgent;

import java.lang.instrument.Instrumentation;

public class Program {
  public static void main(String[] args) throws Exception {
    Instrumentation inst = SelfAttachAgent.getInstrumentation();
    System.out.println(inst);
  }
}
```

#### SelfAttachAgent

```java
package lsieun.agent;

import com.sun.tools.attach.VirtualMachine;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.instrument.Instrumentation;
import java.lang.management.ManagementFactory;
import java.util.jar.Attributes;
import java.util.jar.JarOutputStream;
import java.util.jar.Manifest;

public class SelfAttachAgent {
  private static volatile Instrumentation globalInstrumentation;

  public static void premain(String agentArgs, Instrumentation inst) {
    globalInstrumentation = inst;
  }

  public static void agentmain(String agentArgs, Instrumentation inst) {
    globalInstrumentation = inst;
  }

  public static Instrumentation getInstrumentation() {
    if (globalInstrumentation == null) {
      loadAgent();
    }
    return globalInstrumentation;
  }

  public static void loadAgent() {
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    int index = nameOfRunningVM.indexOf('@');
    String pid = nameOfRunningVM.substring(0, index);

    VirtualMachine vm = null;
    try {
      String jarPath = createTempJarFile().getPath();
      System.out.println(jarPath);

      vm = VirtualMachine.attach(pid);
      vm.loadAgent(jarPath, "");

    } catch (Exception e) {
      throw new RuntimeException(e);
    } finally {
      if (vm != null) {
        try {
          vm.detach();
        } catch (IOException ignored) {
        }
      }
    }
  }

  public static File createTempJarFile() throws IOException {
    File jar = File.createTempFile("agent", ".jar");
    jar.deleteOnExit();
    createJarFile(jar);
    return jar;
  }

  private static void createJarFile(File jar) throws IOException {
    String className = SelfAttachAgent.class.getName();

    Manifest manifest = new Manifest();
    Attributes attrs = manifest.getMainAttributes();
    attrs.put(Attributes.Name.MANIFEST_VERSION, "1.0");
    attrs.put(new Attributes.Name("Premain-Class"), className);
    attrs.put(new Attributes.Name("Agent-Class"), className);
    attrs.put(new Attributes.Name("Can-Retransform-Classes"), "true");
    attrs.put(new Attributes.Name("Can-Redefine-Classes"), "true");

    JarOutputStream jos = new JarOutputStream(new FileOutputStream(jar), manifest);
    jos.flush();
    jos.close();
  }
}
```

#### Run

##### Java 8

```shell
$ java -cp "${JAVA_HOME}/lib/tools.jar"\;./target/classes/ sample.Program

7704
C:\Users\liusen\AppData\Local\Temp\agent3349937074235412866.jar
sun.instrument.InstrumentationImpl@65b54208
```

##### Java 9

```
$ java -cp ./target/classes/ sample.Program

Caused by: java.io.IOException: Can not attach to current VM
```

**Attach API cannot be used to attach to the current VM by default** [Link](https://www.oracle.com/java/technologies/javase/9-notes.html)

The implementation of Attach API has changed in JDK 9 to disallow attaching to the current VM by default. This change should have no impact on tools that use the Attach API to attach to a running VM. It may impact libraries that misuse this API as a way to get at the `java.lang.instrument` API. The system property `jdk.attach.allowAttachSelf` may be set on the command line to mitigate any compatibility with this change.

Note: since JDK 9 attaching to current process requires setting the system property:

```
-Djdk.attach.allowAttachSelf=true
$ java -Djdk.attach.allowAttachSelf=true -cp ./target/classes/ sample.Program
```

Java 9 adds the `Launcher-Agent-Class` attribute which can be used on executable JAR files to start an agent before the main class is loaded.

### 5.3 总结

本文内容总结如下：

- 第一点，如何获取当前VM的PID值。
- 第二点，在Java 9的情况下，默认不允许attach到当前VM，解决方式是将`jdk.attach.allowAttachSelf`属性设置成`true`。

## 6. JMX: JMXConnectorServer和JMXConnector

JMX是Java Management Extension的缩写。在本文当中，我们对JMX进行一个简单的介绍，目的是为后续内容做一个铺垫。

### 6.1 JMX介绍

> [JMX_百度百科 (baidu.com)](https://baike.baidu.com/item/JMX/2829357?fr=aladdin)
>
> JMX（[Java](https://baike.baidu.com/item/Java/85979?fromModule=lemma_inlink) Management Extensions，即[Java管理扩展](https://baike.baidu.com/item/Java管理扩展/18428506?fromModule=lemma_inlink)）是一个为[应用程序](https://baike.baidu.com/item/应用程序/5985445?fromModule=lemma_inlink)、设备、系统等植入[管理功能](https://baike.baidu.com/item/管理功能/3277338?fromModule=lemma_inlink)的框架。JMX可以跨越一系列异构操作系统平台、[系统体系结构](https://baike.baidu.com/item/系统体系结构/6842760?fromModule=lemma_inlink)和[网络传输协议](https://baike.baidu.com/item/网络传输协议/332131?fromModule=lemma_inlink)，灵活的开发[无缝集成](https://baike.baidu.com/item/无缝集成/5135178?fromModule=lemma_inlink)的系统、网络和[服务管理](https://baike.baidu.com/item/服务管理/7543685?fromModule=lemma_inlink)应用。

如果两个JVM，当它们使用JMX进行沟通时，可以简单的描述成下图：

![img](https://lsieun.github.io/assets/images/java/jmx/jmx-mbean-server-connector.png)

#### MBean

要理解MBean，我们需要把握住两个核心概念：resouce和MBean。两者的关系是，先有resouce，后有MBean。

##### Resource和MBean

首先，我们说第一个概念：resource。其实，一个事物是不是resouce（资源），是一个非常主观的判断。比如说，地上有一块石头，我们会觉得它是一个无用的事物；换一个场景，有一天需要盖房子，石头就成了盖房子的有用资源。

在编程的语言环境当中，resource也是一个模糊的概念，可以体现在不同的层面上：

- 可能是Class层面，例如，类里的一个字段，它记录了某个方法被调用的次数
- 可能是Application层面，例如，在线用户的数量
- 可能是JVM层面，例如：线程信息、垃圾回收信息
- 可能是硬件层面，例如：CPU的能力、磁盘空间的大小

**JMX的一个主要目标就是对resource（资源）进行management（管理）和monitor（监控），它要把一个我们关心的事物（resouce）给转换成manageable resource。**

A **resource** is any entity in the system that needs to be monitored and/or controlled by a management application; resources that can be monitored and controlled are called **manageable**.

接着，我们来说第二个概念：MBean。MBean是managed bean的缩写，它就代表manageable resource本身，或者是对manageable resource的进一步封装， 它就是manageable resource在JMX架构当中所对应的一个“术语”或标准化之后的“概念”。

<u>An MBean is an application or system resource that has been instrumented to be manageable through JMX.</u>

- Application components designed with their *management interface* in mind can typically be written as *MBeans*.
- *MBeans* can be used as *wrappers* for legacy code without a management interface or as *proxies* for code with a legacy management interface.

##### Standard MBean

在JMX当中，MBean有不同的类型：

- Standard MBean
- Dynamic MBean
- Open MBean
- Model MBean

在这里，我们只关注Standard MBean。在JMX当中，Standard MBean有一些要求，需要我们在编写代码的过程当中遵守：

- 类名层面。比如说，有一个`SmartChild`类，它就是我们关心的resource，它必须实现一个接口（management interface），这个接口的名字必须是`SmartChildMBean`。 也就是，在原来`SmartChild`类名的基础上，再加上`MBean`后缀。
- 构造方法层面。比如说，`SmartChild`类必须有一个**public constructor**。
- Attributes层面，或者Getter和Setter层面。
  - 比如，getter方法不能接收参数，`int getAge()`是合理的，而`int getAge(String name)`是不合理的。
  - 比如说，setter方法只能接收一个参数，`void setAge(int age)`是合理的，而`void setAge(int age, String name)`是不合理的。
- Operations。在MBean当中，排除Attributes之外的方法，就属于Operations操作。

```pseudocode
                                                           ┌─── getter
                                         ┌─── attribute ───┤
                  ┌─── Java interface ───┤                 └─── setter
                  │                      └─── operation
Standard MBean ───┤
                  │
                  └─── Java class
```

------

For any resource class `XYZ` that is to be instrumented as a standard MBean, a Java interface called `XYZMBean` must be defined, and it must be implemented by `XYZ`. Note that the `MBean` suffix is case-sensitive: `Mbean` is incorrect, as is `mBean` or `mbean`.

A standard MBean is defined by writing a Java interface called `SomethingMBean` and a Java class called `Something` that implements that interface. Every method in the interface defines either an **attribute** or an **operation** in the MBean. By default, every method defines an operation. Attributes and operations are methods that follow certain design patterns. A **standard MBean** is composed of **an MBean interface** and **a class**. [Link](https://docs.oracle.com/javase/tutorial/jmx/mbeans/standard.html)

- The MBean interface lists the methods for all exposed attributes and operations.
- The class implements this interface and provides the functionality of the instrumented resource.

Management **attributes** are named characteristics of an MBean. With Standard MBeans, attributes are defined in the MBean interface via the use of **naming conventions** in the interface methods. There are three kinds of attributes, read-only, write-only, and read-write attributes. [Link](https://www.informit.com/articles/article.aspx?p=27842&seqNum=3)

Management **operations** for Standard MBeans include all the methods declared in the MBean interface that are not recognized as being either a read or write method to an attribute. The operations don’t have to follow any specific naming rules as long as they do not intervene with the management attribute naming conventions.

#### MBeanServer

在JMX当中，`MBeanServer`表示managed bean server。

在某一个`MBean`对象创建好之后，需要将`MBean`对象注册到`MBeanServer`当中，分成两个步骤：

- 第一步，创建`MBeanServer`对象
- 第二步，将`MBean`对象注册到`MBeanServer`上

##### 创建MBeanServer

在这里，我们介绍两种创建MBeanServer对象的方式。

```pseudocode
               ┌─── MBeanServerFactory.createMBeanServer()
MBeanServer ───┤
               └─── ManagementFactory.getPlatformMBeanServer()
```

第一种方式：借助于`javax.management.MBeanServerFactory`类的`createMBeanServer()`方法：

```java
MBeanServer beanServer = MBeanServerFactory.createMBeanServer();
```

第二种方式：借助于`java.lang.management.ManagementFactory`类的`getPlatformMBeanServer()`方法：

```java
MBeanServer platformMBeanServer = ManagementFactory.getPlatformMBeanServer();
```

**两种方式相比较，推荐使用`ManagementFactory.getPlatformMBeanServer()`**。

- The **platform MBean server** was introduced in Java SE 5.0, and is an MBean server that is built into the Java Virtual Machine (Java VM).
- The **platform MBean server** can be shared by all managed components that are running in the Java VM.
- However, there is generally no need for more than one MBean server, so **using the platform MBean server is recommended.**

##### 注册MBean

注册`MBean`对象，需要使用`MBeanServer`类的`registerMBean(Object object, ObjectName name)`方法：

```java
SmartChild bean = new SmartChild("Tom", 10);
ObjectName objectName = new ObjectName(Const.SMART_CHILD_BEAN);
beanServer.registerMBean(bean, objectName);
```

其中，`Const.SMART_CHILD_BEAN`的值为`lsieun.management.bean:type=child,name=SmartChild`。

在注册MBean的时候，需要指定唯一的**object name**：

```pseudocode
                               ┌─── MBean
MBeanServer.registerMBean() ───┤
                               └─── ObjectName
```

Each `ObjectName` contains a string made up of two components: the **domain name** and the **key property list**.
The combination of **domain name** and **key property list** must be unique for any given MBean and has the format:

```pseudocode
domain-name:key1=value1[,key2=value2,...,keyN=valueN]
```

在`java.lang.management.ManagementFactory`类的文档注释中，有如下示例：

| Management Interface    | ObjectName                       |
| ----------------------- | -------------------------------- |
| `ClassLoadingMXBean`    | `java.lang:type=ClassLoading`    |
| `MemoryMXBean`          | `java.lang:type=Memory`          |
| `ThreadMXBean`          | `java.lang:type=Threading`       |
| `RuntimeMXBean`         | `java.lang:type=Runtime`         |
| `OperatingSystemMXBean` | `java.lang:type=OperatingSystem` |
| `PlatformLoggingMXBean` | `java.util.logging:type=Logging` |

#### Connector

A connector consists of a **connector client** and a **connector server**.

- A **connector server** is attached to an **MBean server** and listens for connection requests from clients. **A given connector server may establish many concurrent connections with different clients.**
- A **connector client** is responsible for establishing a connection with the **connector server**.

<u>A **connector client** will usually be in a different Java Virtual Machine (Java VM) from the connector server, and will often be running on a different machine.</u>

A **connector server** usually has **an address**, used to establish connections between connector clients and the connector server.

##### Connector Server

创建`JMXConnectorServer`对象，我们可以可以借助于`javax.management.remote.JMXConnectorServerFactory`类的`newJMXConnectorServer()`方法：

```java
JMXServiceURL serviceURL = new JMXServiceURL("rmi", "127.0.0.1", 9876);
JMXConnectorServer connectorServer = JMXConnectorServerFactory.newJMXConnectorServer(serviceURL, null, beanServer);
```

在创建`JMXConnectorServer`对象时，我们用到了`JMXServiceURL`类，如果打印一下`serviceURL`变量，会输出以下结果：

```shell
service:jmx:rmi://127.0.0.1:9876
```

一个更通用的表达形式如下：

```shell
service:jmx:protocol://host[:port][url-path]
```

**在创建`JMXConnectorServer`对象完成之后，它处于“未激活”状态**：

```java
boolean status = connectorServer.isActive();
System.out.println(status); // false
```

**当我们调用`start()`方法后，它才开始监听connector client的连接请求，并进入“激活”状态**：

```java
connectorServer.start();
```

当监听开始之后，我们可以调用`getAddress()`方法来获取connector client可以连接到connector server的服务地址：

```java
JMXServiceURL connectorServerAddress = connectorServer.getAddress();
```

示例如下：

```shell
service:jmx:rmi://127.0.0.1:9876/stub/rO0ABX...
```

**当我们调用`stop()`方法后，它停止监听connector client的连接请求，并进入“未激活”状态**：

```java
connectorServer.stop();
```

##### Connector Client

现在connector server端已经准备好了，接下来就是connector client端要做的事情了。 从API的角度来说，`JMXConnector`类就是connector client。

要创建`JMXConnector`类的实例，我们可以借助于`javax.management.remote.JMXConnectorFactory`类的`connect()`方法：

```java
JMXServiceURL address = new JMXServiceURL(connectorAddress);
JMXConnector connector = JMXConnectorFactory.connect(address);
```

然后，我们再利用`JMXConnector`类的`getMBeanServerConnection()`方法来获取一个`MBeanServerConnection`对象：

```java
MBeanServerConnection beanServerConnection = connector.getMBeanServerConnection();
```

有了`MBeanServerConnection`对象之后，就可以与`MBeanServer`对象进行交互了：

```java
ObjectName objectName = new ObjectName(Const.SMART_CHILD_BEAN);
MBeanInfo beanInfo = beanServerConnection.getMBeanInfo(objectName);
```

值得一提的是，`MBeanServer`本身是一个接口，它继承自`MBeanServerConnection`接口。

```pseudocode
    MBeanServer         MBeanServerConnection
------------------------------------------------
  connector server        connector client
```

### 6.2 JMX示例

#### MBean

```java
package lsieun.management.bean;

public interface SmartChildMBean {
  String getName();
  void setName(String name);

  int getAge();
  void setAge(int age);

  void study(String subject);
}
```

```java
package lsieun.management.bean;

public class SmartChild implements SmartChildMBean {
  private String name;
  private int age;

  public SmartChild(String name, int age) {
    this.name = name;
    this.age = age;
  }

  @Override
  public String getName() {
    return name;
  }

  @Override
  public void setName(String name) {
    this.name = name;
  }

  @Override
  public int getAge() {
    return age;
  }

  @Override
  public void setAge(int age) {
    this.age = age;
  }

  @Override
  public void study(String subject) {
    String message = String.format("%s (%d) is studying %s.", name, age, subject);
    System.out.println(message);
  }
}
```

#### Server

```java
package run.jmx;

import lsieun.cst.Const;
import lsieun.management.bean.SmartChild;

import javax.management.MBeanServer;
import javax.management.MBeanServerFactory;
import javax.management.ObjectName;
import javax.management.remote.JMXConnectorServer;
import javax.management.remote.JMXConnectorServerFactory;
import javax.management.remote.JMXServiceURL;
import java.util.concurrent.TimeUnit;

public class JMXServer {
  public static void main(String[] args) throws Exception {
    // 第一步，创建MBeanServer
    MBeanServer beanServer = MBeanServerFactory.createMBeanServer();

    // 第二步，注册MBean
    SmartChild bean = new SmartChild("Tom", 10);
    ObjectName objectName = new ObjectName(Const.SMART_CHILD_BEAN);
    beanServer.registerMBean(bean, objectName);

    // 第三步，创建Connector Server
    JMXServiceURL serviceURL = new JMXServiceURL("rmi", "127.0.0.1", 9876);
    JMXConnectorServer connectorServer = JMXConnectorServerFactory.newJMXConnectorServer(serviceURL, null, beanServer);

    // 第四步，开启Connector Server监听
    connectorServer.start();
    JMXServiceURL connectorServerAddress = connectorServer.getAddress();
    System.out.println(connectorServerAddress);

    // 休息5分钟
    TimeUnit.MINUTES.sleep(5);

    // 第五步，关闭Connector Server监听
    connectorServer.stop();
  }
}
```

####  Client

```java
package run.jmx;

import lsieun.cst.Const;

import javax.management.MBeanServerConnection;
import javax.management.ObjectName;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;

public class JMXClient {
  public static void main(String[] args) throws Exception {
    // 第一步，创建Connector Client
    String connectorAddress = "service:jmx:rmi://127.0.0.1:9876/stub/rO0AB...";
    JMXServiceURL address = new JMXServiceURL(connectorAddress);
    JMXConnector connector = JMXConnectorFactory.connect(address);

    // 第二步，获取MBeanServerConnection对象
    MBeanServerConnection beanServerConnection = connector.getMBeanServerConnection();

    // 第三步，向MBean Server发送请求
    ObjectName objectName = new ObjectName(Const.SMART_CHILD_BEAN);
    String[] array = new String[]{"Chinese", "Math", "English"};
    for (String item : array) {
      beanServerConnection.invoke(objectName, "study", new Object[]{item}, new String[]{String.class.getName()});
    }

    // 第四步，关闭Connector Client
    connector.close();
  }
}
```

### 6.3 总结

本文内容总结如下：

- 第一点，理解两个JVM使用JMX进行沟通的整体思路和相关概念。
- 第二点，介绍JMX的目的是为了结合Java Agent和JMX一起使用。

## 7. JMX: management-agent.jar

### 7.1 Pre Java 9

#### management-agent.jar

<u>A JMX client uses the Attach API to dynamically attach to a target virtual machine and load the JMX agent</u> (if it is not already loaded) from the `management-agent.jar` file, which is located in the `lib` subdirectory of the target virtual machine’s JRE home directory.

```shell
JRE_HOME/lib/management-agent.jar
```

其中，只有一个`META-INF/MANIFEST.MF`文件，内容如下：

```txt
Manifest-Version: 1.0
Created-By: 1.7.0_291 (Oracle Corporation)
Agent-Class: sun.management.Agent
Premain-Class: sun.management.Agent

```

在`sun.management.Agent`类当中，定义了`LOCAL_CONNECTOR_ADDRESS_PROP`静态字段：

```java
public class Agent {
  private static final String LOCAL_CONNECTOR_ADDRESS_PROP = "com.sun.management.jmxremote.localConnectorAddress";

  private static synchronized void startLocalManagementAgent() {
    Properties agentProps = VMSupport.getAgentProperties();

    // start local connector if not started
    if (agentProps.get(LOCAL_CONNECTOR_ADDRESS_PROP) == null) {
      JMXConnectorServer cs = ConnectorBootstrap.startLocalConnectorServer();
      String address = cs.getAddress().toString();
      // Add the local connector address to the agent properties
      agentProps.put(LOCAL_CONNECTOR_ADDRESS_PROP, address);

      try {
        // export the address to the instrumentation buffer
        ConnectorAddressLink.export(address);
      } catch (Exception x) {
        // Connector server started but unable to export address
        // to instrumentation buffer - non-fatal error.
        warning(EXPORT_ADDRESS_FAILED, x.getMessage());
      }
    }
  }
}
```

#### Application

```java
package sample;

import java.lang.management.ManagementFactory;
import java.util.concurrent.TimeUnit;

public class Program {
  public static void main(String[] args) throws Exception {
    // 第一步，打印进程ID
    String nameOfRunningVM = ManagementFactory.getRuntimeMXBean().getName();
    System.out.println(nameOfRunningVM);

    // 第二步，倒计时退出
    int count = 600;
    for (int i = 0; i < count; i++) {
      String info = String.format("|%03d| %s remains %03d seconds", i, nameOfRunningVM, (count - i));
      System.out.println(info);

      TimeUnit.SECONDS.sleep(1);
    }
  }
}
```

#### Attach

```java
package run;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import lsieun.utils.JarUtils;

import javax.management.MBeanServerConnection;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;
import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.util.List;

public class VMAttach {
  static final String LOCAL_CONNECTOR_ADDRESS_PROP = "com.sun.management.jmxremote.localConnectorAddress";

  public static void main(String[] args) throws Exception {
    // 第一步，准备参数
    String displayName = "sample.Program";

    // 第二步，使用Attach机制，加载management-agent.jar文件，来启动JMX
    String pid = findPID(displayName);
    VirtualMachine vm = VirtualMachine.attach(pid);
    String connectorAddress = vm.getAgentProperties().getProperty(LOCAL_CONNECTOR_ADDRESS_PROP);
    if (connectorAddress == null) {
      String javaHome = vm.getSystemProperties().getProperty("java.home");
      String agent = JarUtils.getJDKJarPath(javaHome, "management-agent.jar");
      vm.loadAgent(agent);
      connectorAddress = vm.getAgentProperties().getProperty(LOCAL_CONNECTOR_ADDRESS_PROP);
      if (connectorAddress == null) {
        throw new NullPointerException("connectorAddress is null");
      }
    }
    vm.detach();
    System.out.println(connectorAddress);

    // 第三步，借助于JMX进行沟通
    JMXServiceURL servURL = new JMXServiceURL(connectorAddress);
    JMXConnector con = JMXConnectorFactory.connect(servURL);
    MBeanServerConnection mbsc = con.getMBeanServerConnection();
    RuntimeMXBean proxy = ManagementFactory.getPlatformMXBean(mbsc, RuntimeMXBean.class);
    long uptime = proxy.getUptime();
    System.out.println(uptime);
  }

  public static String findPID(String name) {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : list) {
      String displayName = vmd.displayName();
      if (displayName != null && displayName.equals(name)) {
        return vmd.id();
      }
    }
    throw new RuntimeException("Not Exist: " + name);
  }
}
```

### 7.2 Since Java 9

在[JDK 9 Release Notes](https://www.oracle.com/java/technologies/javase/9-removed-features.html#JDK-8043939)中提到：**management-agent.jar is removed**。

`management-agent.jar` has been removed. Tools that have been using the Attach API to load this agent into a running VM should be aware that the Attach API has been updated in JDK 9 to define two new methods for starting a management agent:

- `com.sun.tools.attach.VirtualMachine.startManagementAgent(Properties agentProperties)`
- `com.sun.tools.attach.VirtualMachine.startLocalManagementAgent()`

#### startLocalManagementAgent

```java
package run;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import javax.management.MBeanServerConnection;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;
import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.util.List;

public class VMAttach {
  static final String LOCAL_CONNECTOR_ADDRESS_PROP = "com.sun.management.jmxremote.localConnectorAddress";

  public static void main(String[] args) throws Exception {
    // 第一步，准备参数
    String displayName = "sample.Program";

    // 第二步，启动JMX
    String pid = findPID(displayName);
    VirtualMachine vm = VirtualMachine.attach(pid);
    String connectorAddress = vm.getAgentProperties().getProperty(LOCAL_CONNECTOR_ADDRESS_PROP);
    if (connectorAddress == null) {
      vm.startLocalManagementAgent();
      connectorAddress = vm.getAgentProperties().getProperty(LOCAL_CONNECTOR_ADDRESS_PROP);
      if (connectorAddress == null) {
        throw new NullPointerException("connectorAddress is null");
      }
    }
    vm.detach();
    System.out.println(connectorAddress);

    // 第三步，借助于JMX进行沟通
    JMXServiceURL servURL = new JMXServiceURL(connectorAddress);
    JMXConnector con = JMXConnectorFactory.connect(servURL);
    MBeanServerConnection mbsc = con.getMBeanServerConnection();
    RuntimeMXBean proxy = ManagementFactory.getPlatformMXBean(mbsc, RuntimeMXBean.class);
    long uptime = proxy.getUptime();
    System.out.println(uptime);
  }

  public static String findPID(String name) {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : list) {
      String displayName = vmd.displayName();
      if (displayName != null && displayName.equals(name)) {
        return vmd.id();
      }
    }
    throw new RuntimeException("Not Exist: " + name);
  }
}
```

#### startManagementAgent

```java
package run;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import javax.management.MBeanServerConnection;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;
import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.util.List;
import java.util.Properties;

public class VMAttach {
  public static void main(String[] args) throws Exception {
    // 第一步，准备参数
    int port = 5000;
    String displayName = "sample.Program";

    // 第二步，启动JMX
    String pid = findPID(displayName);
    VirtualMachine vm = VirtualMachine.attach(pid);
    Properties props = new Properties();
    props.put("com.sun.management.jmxremote.port", String.valueOf(port));
    props.put("com.sun.management.jmxremote.authenticate", "false");
    props.put("com.sun.management.jmxremote.ssl", "false");
    vm.startManagementAgent(props);
    vm.getAgentProperties().list(System.out);
    vm.detach();

    // 第三步，借助于JMX进行沟通
    String jmxUrlStr = String.format("service:jmx:rmi:///jndi/rmi://localhost:%d/jmxrmi", port);
    JMXServiceURL servURL = new JMXServiceURL(jmxUrlStr);
    JMXConnector con = JMXConnectorFactory.connect(servURL);
    MBeanServerConnection mbsc = con.getMBeanServerConnection();
    RuntimeMXBean proxy = ManagementFactory.getPlatformMXBean(mbsc, RuntimeMXBean.class);
    long uptime = proxy.getUptime();
    System.out.println(uptime);
  }

  public static String findPID(String name) {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : list) {
      String displayName = vmd.displayName();
      if (displayName != null && displayName.equals(name)) {
        return vmd.id();
      }
    }
    throw new RuntimeException("Not Exist: " + name);
  }
}
```

### 7.3 总结

本文内容总结如下：

- 第一点，在Java 8版本当中，我们需要借助于`VirtualMachine`类的`loadAgent()`方法和`management-agent.jar`来开启JMX服务。
- 第二点，在Java 9之后的版本中，我们需要借助于`VirtualMachine`类的`startLocalManagementAgent()`和`startManagementAgent()`方法来开启JMX服务。

## 8. JMX: Instrumentation

### 8.1 MBean

#### GoodChildMBean

```java
package lsieun.management.bean;

public interface GoodChildMBean {
  void study(String className, String methodName, String methodDesc, String options);
}
```

#### GoodChild

```java
package lsieun.management.bean;

import lsieun.asm.visitor.MethodInfo;
import lsieun.cst.Const;
import lsieun.instrument.InabilityTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.util.Formatter;
import java.util.HashSet;
import java.util.Set;

public class GoodChild implements GoodChildMBean {
  protected final Instrumentation instrumentation;

  public GoodChild(Instrumentation instrumentation) {
    this.instrumentation = instrumentation;
  }

  @Override
  public void study(String className, String methodName, String methodDesc, String option) {
    StringBuilder sb = new StringBuilder();
    Formatter fm = new Formatter(sb);
    fm.format("%s%n", Const.SEPARATOR);
    fm.format("GoodChild.study%n");
    fm.format("    class  : %s%n", className);
    fm.format("    method : %s:%s%n", methodName, methodDesc);
    fm.format("    option : %s%n", option);
    fm.format("    thread : %s@%s(%s)%n",
              Thread.currentThread().getName(),
              Thread.currentThread().getId(),
              Thread.currentThread().isDaemon()
             );
    fm.format("%s%n", Const.SEPARATOR);
    System.out.println(sb);

    Set<MethodInfo> flags = new HashSet<>();
    if (option != null) {
      String[] array = option.split(",");
      for (String element : array) {
        if ("".equals(element)) continue;
        MethodInfo methodInfo = Enum.valueOf(MethodInfo.class, element);
        flags.add(methodInfo);
      }
    }

    // 第一种方式，用Class.forName()方法，速度较快
    try {
      Class<?> clazz = Class.forName(className);
      transform(clazz, methodName, methodDesc, flags);
      return;
    } catch (Exception ex) { /* Nope */ }

    // 第二种方式，用Instrumentation.getAllLoadedClasses()方法，速度较慢
    Class<?>[] allLoadedClasses = instrumentation.getAllLoadedClasses();
    for (Class<?> clazz : allLoadedClasses) {
      if (clazz.getName().equals(className)) {
        transform(clazz, methodName, methodDesc, flags);
        return;
      }
    }
    throw new RuntimeException("Failed to locate class [" + className + "]");
  }

  /**
     * Registers a transformer and executes the transform
     *
     * @param clazz      The class to transform
     * @param methodName The method name to instrument
     * @param methodDesc The method signature to match
     */
  protected void transform(Class<?> clazz, String methodName, String methodDesc, Set<MethodInfo> flags) {
    ClassLoader classLoader = clazz.getClassLoader();
    ClassFileTransformer transformer = new InabilityTransformer(classLoader, clazz.getName(), methodName, methodDesc, flags);
    instrumentation.addTransformer(transformer, true);
    try {
      instrumentation.retransformClasses(clazz);
    } catch (Exception ex) {
      throw new RuntimeException("Failed to transform [" + clazz.getName() + "]", ex);
    } finally {
      instrumentation.removeTransformer(transformer);
    }
  }
}
```

### 8.2 Agent Jar

#### DynamicAgent

```java
package lsieun.agent;

import lsieun.cst.Const;
import lsieun.management.bean.GoodChild;
import lsieun.utils.*;

import javax.management.MBeanServer;
import javax.management.ObjectName;
import java.lang.instrument.Instrumentation;
import java.lang.management.ManagementFactory;

public class DynamicAgent {
  public static void agentmain(String agentArgs, Instrumentation inst) throws Exception {
    // 第一步，打印信息：agentArgs, inst, classloader, thread
    PrintUtils.printAgentInfo(DynamicAgent.class, "Agent-Class", agentArgs, inst);

    // 第二步，创建MBean
    System.out.println("Installing JMX Agent...");
    GoodChild child = new GoodChild(inst);
    ObjectName objectName = new ObjectName(Const.GOOD_CHILD_BEAN);

    // 第三步，注册MBean
    MBeanServer beanServer = ManagementFactory.getPlatformMBeanServer();
    beanServer.registerMBean(child, objectName);

    // 第四步，设置属性
    System.setProperty(Const.AGENT_MANAGEMENT_PROP, "true");
    System.out.println("JMX Agent Installed");
  }
}
```

### 8.3 JMX Client

#### AgentInstaller

```java
package run.jmx;

import com.sun.tools.attach.VirtualMachine;
import lsieun.cst.Const;
import lsieun.utils.JarUtils;
import lsieun.utils.VMAttachUtils;

import javax.management.MBeanServerConnection;
import javax.management.ObjectName;
import javax.management.remote.JMXConnector;
import javax.management.remote.JMXConnectorFactory;
import javax.management.remote.JMXServiceURL;
import java.util.Properties;

public class AgentInstaller {
  public static void main(String[] args) throws Exception {
    // 第一步，获取PID
    String displayName = "sample.Program";
    String pid = VMAttachUtils.findPID(displayName);
    System.out.println("pid: " + pid);

    // 第二步，利用Attach 机制，加载两个Agent Jar
    VirtualMachine vm = VirtualMachine.attach(pid);
    Properties properties = vm.getSystemProperties();
    String value = properties.getProperty(Const.AGENT_MANAGEMENT_PROP);
    if (value == null) {
      // 加载第一个Agent Jar
      String jarPath = JarUtils.getJarPath();
      vm.loadAgent(jarPath);
    }

    String connectorAddress = vm.getAgentProperties().getProperty(Const.LOCAL_CONNECTOR_ADDRESS_PROP, null);
    vm.getAgentProperties().list(System.out);
    if (connectorAddress == null) {
      // 加载第二个Agent Jar
      String home = vm.getSystemProperties().getProperty("java.home");
      String managementAgentJarPath = JarUtils.getManagementAgentJarPath(home);
      vm.loadAgent(managementAgentJarPath);
      connectorAddress = vm.getAgentProperties().getProperty(Const.LOCAL_CONNECTOR_ADDRESS_PROP, null);
      vm.getAgentProperties().list(System.out);
    }
    System.out.println(connectorAddress);
    vm.detach();

    // 第三步，准备参数
    String beanName = Const.GOOD_CHILD_BEAN;
    String beanMethodName = "study";
    String[] beanMethodArgArray = new String[]{
      //                "sample.HelloWorld", "add", "(II)I", "",
      "sample.HelloWorld", "add", "(II)I", "NAME_AND_DESC,PARAMETER_VALUES",
      //                "sample.HelloWorld", "add", "(II)I", "NAME_AND_DESC,PARAMETER_VALUES,RETURN_VALUE",
    };

    // 第四步，借助JMXConnector，调用MBean的方法
    ObjectName objectName = new ObjectName(beanName);
    JMXServiceURL serviceURL = new JMXServiceURL(connectorAddress);
    try (JMXConnector connector = JMXConnectorFactory.connect(serviceURL)) {
      MBeanServerConnection server = connector.getMBeanServerConnection();
      server.invoke(objectName, beanMethodName, beanMethodArgArray,
                    new String[]{
                      String.class.getName(),
                      String.class.getName(),
                      String.class.getName(),
                      String.class.getName(),
                    });
    }
  }
}
```

从下面的输出结果当中，我们可以看到`GoodChild.study()`方法运行在不同的线程（thread）：

```shell
GoodChild.study
    class  : sample.HelloWorld
    method : add:(II)I
    option : NAME_AND_DESC,PARAMETER_VALUES
    thread : RMI TCP Connection(6)-192.168.200.1@20(true)
GoodChild.study
    class  : sample.HelloWorld
    method : sub:(II)I
    option : RETURN_VALUE
    thread : RMI TCP Connection(4)-192.168.200.1@18(true)
GoodChild.study
    class  : sample.HelloWorld
    method : sub:(II)I
    option : NAME_AND_DESC
    thread : RMI TCP Connection(3)-192.168.200.1@17(true)
```

#### JConsole

在下面的`jconsole`当中，`study`方法的参数值：

- `p1`: `sample.HelloWorld`
- `p2`: `add`
- `p3`: `(II)I`
- `p4`: `NAME_AND_DESC`

![img](https://lsieun.github.io/assets/images/java/agent/jmx-instrumentation-good-child-study.png)

## 9. ja-netfilter分析

### 9.1 ja-netfilter分析

#### Project

[mini-jn](https://gitee.com/lsieun/mini-jn)

```pseudocode
mini-jn
├─── pom.xml
├─── src
│    └─── main
│         └─── java
│              ├─── boot
│              │    └─── filter
│              │         ├─── BigIntegerFilter.java
│              │         ├─── HttpClientFilter.java
│              │         ├─── InetAddressFilter.java
│              │         ├─── LinkedTreeMapFilter.java
│              │         └─── VMManagementImplFilter.java
│              ├─── jn
│              │    ├─── agent
│              │    │    └─── LoadTimeAgent.java
│              │    ├─── asm
│              │    │    ├─── MyClassNode.java
│              │    │    └─── tree
│              │    │         ├─── BigIntegerNode.java
│              │    │         ├─── HttpClientNode.java
│              │    │         ├─── InetAddressNode.java
│              │    │         ├─── LinkedTreeMapNode.java
│              │    │         └─── VMManagementImplNode.java
│              │    ├─── cst
│              │    │    └─── Const.java
│              │    ├─── Main.java
│              │    └─── utils
│              │         ├─── ClassUtils.java
│              │         ├─── FileUtils.java
│              │         └─── TransformerUtils.java
│              ├─── run
│              │    └─── instrument
│              │         └─── StaticInstrumentation.java
│              └─── sample
│                   └─── Program.java
└─── target
     ├─── boot-support.jar
     └─── mini-jn.jar
```

```shell
$ mvn clean package
$ cd ./target/classes/
$ jar -cvf boot-support.jar boot/
$ mv boot-support.jar ../
```

### 9.2 测试

#### VMManagementImpl

```java
package sample;

import sun.management.VMManagement;

import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.lang.reflect.Field;
import java.util.List;

public class Program {
  public static void main(String[] args) throws Exception {
    RuntimeMXBean runtime = ManagementFactory.getRuntimeMXBean();
    Field jvm = runtime.getClass().getDeclaredField("jvm");
    jvm.setAccessible(true);
    VMManagement mgmt = (sun.management.VMManagement) jvm.get(runtime);
    System.out.println(mgmt.getClass());

    List<String> vmArguments = mgmt.getVmArguments();
    for (String item : vmArguments) {
      System.out.println(item);
    }
  }
}
```

第一次运行：

```shell
$ java -cp ./target/classes/ -Duser.language=en -Duser.country=US -Djanf.debug=true sample.Program

class sun.management.VMManagementImpl
-Duser.language=en
-Duser.country=US
-Djanf.debug=true
```

第二次运行：

```shell
$ java -cp ./target/classes/ -Duser.language=en -Duser.country=US -Djanf.debug=true -javaagent:./target/mini-jn.jar sample.Program

Premain-Class: jn.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
class sun.management.VMManagementImpl
-Duser.language=en
-Duser.country=US
```

#### InetAddress

```java
package sample;

import java.net.InetAddress;

public class Program {
  public static void main(String[] args) throws Exception {
    getAllByName();
    isReachable();
  }

  private static void getAllByName() {
    try {
      String host = "jetbrains.com";
      InetAddress[] addresses = InetAddress.getAllByName(host);
      System.out.println("host: " + host);
      for (InetAddress item : addresses) {
        System.out.println("    " + item);
      }
    } catch (Exception ignored) {
    }
  }

  private static void isReachable() {
    try {
      String host = "jetbrains.com";
      String ip = "13.33.141.66";
      String[] array = ip.split("\\.");
      byte[] ip_bytes = new byte[4];
      for (int i = 0; i < 4; i++) {
        ip_bytes[i] = (byte) (Integer.parseInt(array[i]) & 0xff);
      }
      InetAddress address = InetAddress.getByAddress(host, ip_bytes);
      boolean reachable = address.isReachable(2000);
      System.out.println(reachable);
    } catch (Exception ignored) {
    }
  }
}
```

第一次运行：

```shell
$ java -cp ./target/classes/ sample.Program
host: jetbrains.com
    jetbrains.com/13.33.141.66
    jetbrains.com/13.33.141.72
    jetbrains.com/13.33.141.29
    jetbrains.com/13.33.141.64
true
```

第二次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/mini-jn.jar sample.Program
Premain-Class: jn.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
Reject dns query: jetbrains.com
Reject dns reachable test: jetbrains.com
false
```

#### HttpClient

```java
package sample;

import java.net.URL;
import java.net.URLConnection;

public class Program {
  public static void main(String[] args) throws Exception {
    URL url = new URL("https://account.jetbrains.com/lservice/rpc/validateKey.action");
    URLConnection urlConnection = url.openConnection();
    urlConnection.connect();
  }
}
```

第一次运行：

```shell
$ java -cp ./target/classes/ sample.Program
```

第二次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/mini-jn.jar sample.Program
Premain-Class: jn.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
Exception in thread "main" java.net.SocketTimeoutException: connect timed out
        at boot.filter.HttpClientFilter.testURL(HttpClientFilter.java:15)
        at sun.net.www.http.HttpClient.openServer(Unknown Source)
```

#### LinkedTreeMap

```java
package sample;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import jn.cst.Const;

public class Program {
  public static void main(String[] args) throws Exception {
    Gson gson = new GsonBuilder().setPrettyPrinting().create();
    Object obj = gson.fromJson(Const.LICENSE_JSON, Object.class);
    System.out.println(gson.toJson(obj));
  }
}
```

第一次运行：

```shell
$ java -cp ./target/classes/\;./target/lib/gson-2.8.9.jar sample.Program
{
  "licenseId": "HELLOWORLD",
  "licenseeName": "Jerry",
  "products": [
    {
      "code": "II",
      "fallbackDate": "2020-01-10",
      "paidUpTo": "2021-01-09"
    }
  ],
  "gracePeriodDays": 7.0,
  "autoProlongated": false,
  "isAutoProlongated": false
}
```

第二次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/mini-jn.jar sample.Program
Premain-Class: jn.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
{
  "licenseId": "HELLOWORLD",
  "licenseeName": "Tom",
  "products": [
    {
      "code": "II",
      "fallbackDate": "2020-01-10",
      "paidUpTo": "2022-12-31"
    }
  ],
  "gracePeriodDays": "30",
  "autoProlongated": false,
  "isAutoProlongated": false
}
```

#### BigInteger

```java
package sample;

import java.math.BigInteger;

public class Program {
  public static void main(String[] args) {
    // a^b mod c
    BigInteger a = new BigInteger("5");
    BigInteger b = new BigInteger("3");
    BigInteger c = new BigInteger("101");

    BigInteger actualValue = a.modPow(b, c);
    System.out.println(actualValue);

    BigInteger expectedValue = new BigInteger("21");
    System.out.println(expectedValue);
  }
}
```

第一次运行：

```shell
$ java -cp ./target/classes/ sample.Program
24
21
```

第二次运行：

```shell
$ java -cp ./target/classes/ -javaagent:./target/mini-jn.jar sample.Program
Premain-Class: jn.agent.LoadTimeAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
========= ========= =========
21
21
```

# 其他相关资料

> [JEP 451: Prepare to Disallow the Dynamic Loading of Agents (openjdk.org)](https://openjdk.org/jeps/451)
>
> [StackOverflow: Difference between redefine and retransform in javaagent](https://stackoverflow.com/questions/19009583/difference-between-redefine-and-retransform-in-javaagent) <= 推荐阅读
>
> [类加载器一篇足以 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/520521579) <= 推荐阅读
>
> [类加载时JVM在搞什么？JVM源码分析+OOP-KLASS模型分析_躺平程序猿的博客-CSDN博客](https://blog.csdn.net/yangxiaofei_java/article/details/118469738) <= 推荐阅读
>
> [jmx 是什么？应用场景是什么？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/376796753)
>
> [什么是JMX？ - 简书 (jianshu.com)](https://www.jianshu.com/p/8c5133cab858)

