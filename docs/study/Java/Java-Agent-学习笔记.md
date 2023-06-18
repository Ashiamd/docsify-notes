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























































# 其他相关资料

> [JEP 451: Prepare to Disallow the Dynamic Loading of Agents (openjdk.org)](https://openjdk.org/jeps/451)

