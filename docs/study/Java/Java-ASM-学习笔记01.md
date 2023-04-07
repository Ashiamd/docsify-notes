

# Java-ASM-学习笔记01

> 视频：[Java ASM系列：（001）课程介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Py4y1g7dN/?spm_id_from=333.788&vd_source=ba4d176271299cb334816d3c4cbc885f)
> 视频对应的gitee：[learn-java-asm: 学习Java ASM (gitee.com)](https://gitee.com/lsieun/learn-java-asm)
> 视频对应的文章：[Java ASM系列一：Core API | lsieun](https://lsieun.github.io/java/asm/java-asm-season-01.html) <= 笔记有不少摘自该文章，下面不再重复声明
>
> ASM官网：[ASM (ow2.io)](https://asm.ow2.io/)
> ASM中文手册：[ASM4 使用手册(中文版) (yuque.com)](https://www.yuque.com/mikaelzero/asm)

# 1. ASM基础

## 1.1 ASM介绍

### 1.1.1 ASM简述

​	简述，ASM是一个操作字节码的jar包，能够生成`.class`文件，也能够在原有的`.class`文件基础上做改造生成新的`.class`文件。

![Java ASM系列：（001）ASM介绍_Core API](https://s2.51cto.com/images/20210618/1624004678130912.jpeg?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

+ ASM操作对象：字节码

+ ASM如何操作字节码：**拆分－修改－合并**

> 在[Wikipedia](https://en.wikipedia.org/wiki/ObjectWeb_ASM)上，对ASM进行了如下描述：
>
> ASM provides a simple API for decomposing(将一个整体拆分成多个部分), modifying(修改某一部分的信息), and recomposing(将多个部分重新组织成一个整体) binary Java classes (i.e. ByteCode).

ASM能够处理的事项

+ 通俗理解

  - 父类：修改成一个新的父类

  - 接口：添加一个新的接口、删除已有的接口

  - 字段：添加一个新的字段、删除已有的字段

  - 方法：添加一个新的方法、删除已有的方法、修改已有的方法

  - ……（省略）

+ 专业理解

  ASM is an all-purpose(多用途的；通用的) Java ByteCode **manipulation**(这里的manipulation应该是指generate和transform操作) and **analysis** framework. It can be used to modify existing classes or to dynamically generate classes, directly in binary form.

  The goal of the ASM library is to **generate**, **transform** and **analyze** compiled Java classes, represented as byte arrays (as they are stored on disk and loaded in the Java Virtual Machine).

  ![Java ASM系列：（001）ASM介绍_Java_03](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

+ 小总结

  + **generation：是从0到1的操作，是最简单的操作，在第二章进行介绍。**也就是说，原来什么都没有，经过generation操作，会生成一个新的`.class`文件。
  + **transformation：是从1到1的操作，是中度复杂的操作，在第三章进行介绍。**也就是说，原来有一个`.class`文件，经过transformation操作，会生成一个新的`.class`文件。
  + **analysis：是从1到0的操作，是最复杂的操作。**也就是说，原来有一个`.class`文件，经过analysis操作，虽然有分析的结果，但是不会生成新的`.class`文件。

### 1.1.2 ASM行业用例

+ Spring AOP

  在很多Java项目中，都会使用到Spring框架，而Spring框架当中的AOP（Aspect Oriented Programming）是依赖于ASM的。具体来说，Spring的AOP，可以通过JDK的动态代理来实现，也可以通过CGLIB实现。其中，CGLib(Code  Generation Library)是在ASM的基础上构建起来的，所以，Spring AOP是间接的使用了ASM。（参考自[Spring Framework Reference Documentation](https://docs.spring.io/spring-framework/docs/3.0.0.M3/reference/html/index.html)的[8.6 Proxying mechanisms](https://docs.spring.io/spring-framework/docs/3.0.0.M3/reference/html/ch08s06.html)）

+ JDK Lambda

  在Java 8中引入了一个非常重要的特性，就是支持Lambda表达式。Lambda表达式，允许把方法作为参数进行传递，它能够使代码变的更加简洁紧凑。但是，我们可能没有注意到，其实，**在现阶段（Java 8版本），Lambda表达式的调用是通过ASM来实现的**。

  在`rt.jar`文件的`jdk.internal.org.objectweb.asm`包当中，就包含了JDK内置的ASM代码。在JDK 8版本当中，它所使用的ASM 5.0版本。

  如果我们跟踪Lambda表达式的编码实现，就会找到`InnerClassLambdaMetafactory.spinInnerClass()`方法。在这个方法当中，我们就会看到：JDK会使用`jdk.internal.org.objectweb.asm.ClassWriter`来生成一个类，将lambda表达式的代码包装起来。

  - LambdaMetafactory.metafactory() 第一步，找到这个方法
    - InnerClassLambdaMetafactory.buildCallSite() 第二步，找到这个方法
      - InnerClassLambdaMetafactory.spinInnerClass() 第三步，找到这个方法

  > 注意：在《[Java ASM系列二：OPCODE](https://lsieun.github.io/java/asm/java-asm-season-02.html)》的第三章中的[Java 8 Lambda](https://lsieun.github.io/java-asm-02/java8-lambda.html)对Lambda实现进行了较为详细的介绍。

## 1.2 ASM组成部分

### 1.2.1 Core API和Tree API

​	整体上，ASM可以细分为Core API和Tree API两部分。

![Java ASM系列：（002）ASM的组成部分_Java](https://s2.51cto.com/images/20210618/1624028194528190.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

​	最初ASM只有asm.jar，后来发展到Core API，再后来衍生出Tree API。

### 1.2.2 Core API概览

#### asm.jar

+ **生成类(`.class`)**，主要涉及`ClassVisitor`、`ClassWriter`、`FieldVisitor`、`FieldWriter`、`MethodVisitor`、`MethodWriter`、`Label`和`Opcodes`类

+ **修改类(`.class`)**，主要涉及`ClassReader`和`Type`类

其中最为重要的是`ClassReader`、`ClassVisitor`和`ClassWriter`类

- `ClassReader`类，负责读取`.class`文件里的内容，然后**拆分**成各个不同的部分。
- `ClassVisitor`类，负责对`.class`文件中某一部分里的信息进行**修改**。
- `ClassWriter`类，负责将各个不同的部分重新**合并**成一个完整的`.class`文件。

![Java ASM系列：（002）ASM的组成部分_ASM_02](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

​	上图中，ClassReader读取类文件，ClassVisitor拆分内容，内部通过`visitField()`可修改成员变量信息，`visitMethod()`可修改方法签名，而MethodVisitor又有具体的方法可以修改方法体内执行的逻辑内容。

#### asm-util.jar

​	主要是一些<u>工具类</u>，按照类名前缀可以大致分成两类：

- 以`Check`开头的类，主要负责检查（Check），也就是检查生成的`.class`文件内容是否正确。
- 以`Trace`开头的类，主要负责追踪（Trace），也就是将`.class`文件的内容打印成文字输出，根据输出的文字信息，可以探索或追踪（Trace）`.class`文件的内部信息。

![Java ASM系列：（002）ASM的组成部分_ASM_03](https://s2.51cto.com/images/20210618/1624028362391858.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

> 在`asm-util.jar`当中，主要介绍`CheckClassAdapter`类和`TraceClassVisitor`类，也会简略的说明一下`Printer`、`ASMifier`和`Textifier`类。
>
> 在“第四章”当中，会介绍`asm-util.jar`里的内容。

#### asm-commons.jar

​	主要是一些**常用的功能类**。

![Java ASM系列：（002）ASM的组成部分_ASM_04](https://s2.51cto.com/images/20210618/1624028392439879.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

> 我们会介绍到其中的`AdviceAdapter`、`AnalyzerAdapter`、`Cla***emapper`、`GeneratorAdapter`、`InstructionAdapter`、`LocalVariableSorter`、`SerialVersionUIDAdapter`和`StaticInitMerger`类。
>
> 在“第四章”当中，介绍`asm-commons.jar`里的内容。

+ `asm-util.jar`里，它提供的是通用性的功能，没有特别明确的应用场景
+ `asm-commons.jar`里，它提供的功能，都是为解决某一种特定场景中出现的问题而提出的解决思路

### 1.2.3 搭建ASM开发环境

​	Java maven依赖可参考

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <java.version>1.8</java.version>
  <maven.compiler.source>${java.version}</maven.compiler.source>
  <maven.compiler.target>${java.version}</maven.compiler.target>
  <asm.version>9.0</asm.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>${asm.version}</version>
  </dependency>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-commons</artifactId>
    <version>${asm.version}</version>
  </dependency>
  <dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-util</artifactId>
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
</dependencies>

<build>
  <plugins>
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
        </compilerArgs>
      </configuration>
    </plugin>
  </plugins>
</build>
```

### 1.2.4 ASM生成类示例

#### 目标java类

目标是生成一个和下面`.java`功效一直的`.class`类

```java
package sample;

public class HelloWorld {
    @Override
    public String toString() {
        return "This is a HelloWorld object.";
    }
}
```

#### ASM生成类代码

```java
package com.example;

import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 9:37 PM
 */
public class HelloWorldDump implements Opcodes {

  public static byte[] dump() {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    // https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.25
    // The ACC_SUPER flag indicates which of two alternative semantics is to be expressed by the invokespecial
    // instruction (§invokespecial) if it appears in this class or interface. Compilers to the instruction set
    // of the Java Virtual Machine should set the ACC_SUPER flag. In Java SE 8 and above, the Java Virtual Machine
    // considers the ACC_SUPER flag to be set in every class file, regardless of the actual value of the flag in the
    // class file and the version of the class file.
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null,"java/lang/Object",null);

    // 构造函数
    {
      // HelloWorld类的构造函数 ()表示无参，V无返回值
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC,"<init>", "()V",null,null);
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      // Object的构造函数
      mv1.visitMethodInsn(INVOKESPECIAL,"java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      mv1.visitMaxs(1, 1);
      mv1.visitEnd();
    }

    // toString 方法
    {
      //L表示返回值, ()表示没有参数
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC,"toString","()Ljava/lang/String;",null,null);
      mv2.visitCode();
      mv2.visitLdcInsn("This is a HelloWorld object.");
      mv2.visitInsn(ARETURN);
      mv2.visitMaxs(1, 1);
      mv2.visitEnd();
    }

    cw.visitEnd();
    // 构造 .class 字节码返回
    return cw.toByteArray();
  }
}
```

#### 类加载验证

```java
package com.example;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:30 PM
 */
public class MyClassLoader extends ClassLoader {
  @Override
  protected Class<?> findClass(String name) throws ClassNotFoundException {
    if ("sample.HelloWorld".equals(name)) {
      byte[] bytes = HelloWorldDump.dump();
      Class<?> clazz = defineClass(name, bytes, 0, bytes.length);
      return clazz;
    }
    throw new ClassNotFoundException("Class Not Found" + name);
  }
}

```

```java
package com.example;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    MyClassLoader classLoader = new MyClassLoader();
    Class<?> clazz = classLoader.loadClass("sample.HelloWorld");
    Object instance = clazz.newInstance();
    System.out.println(instance);
  }
}

```

运行HelloWorldRun类的main方法后输出

```shell
This is a HelloWorld object.
```

## 1.3 ASM与ClassFile

