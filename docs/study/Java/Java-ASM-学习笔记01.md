

# Java-ASM-学习笔记01

> 视频：[Java ASM系列：（001）课程介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Py4y1g7dN/?spm_id_from=333.788&vd_source=ba4d176271299cb334816d3c4cbc885f)
> 视频对应的gitee：[learn-java-asm: 学习Java ASM (gitee.com)](https://gitee.com/lsieun/learn-java-asm)
> 视频对应的文章：[Java ASM系列一：Core API | lsieun](https://lsieun.github.io/java/asm/java-asm-season-01.html) <= 笔记内容大部分都摘自该文章，下面不再重复声明
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
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

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
      //Ljava/lang/String表示返回值为String类型, ()表示没有参数
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

### 1.3.1 ClassFile

​	我们都知道，在`.class`文件中，存储的是ByteCode数据。但是，这些ByteCode数据并不是杂乱无章的，而是遵循一定的数据结构。

​	这个`.class`文件遵循的数据结构就是由[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)中定义的 [The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)，如下所示。

```java
ClassFile {
  u4             magic;
  u2             minor_version;
  u2             major_version;
  u2             constant_pool_count;
  cp_info        constant_pool[constant_pool_count-1];
  u2             access_flags;
  u2             this_class;
  u2             super_class;
  u2             interfaces_count;
  u2             interfaces[interfaces_count];
  u2             fields_count;
  field_info     fields[fields_count];
  u2             methods_count;
  method_info    methods[methods_count];
  u2             attributes_count;
  attribute_info attributes[attributes_count];
}
```

### 1.3.2 字节码类库

在下面列举了几个比较常见的字节码类库：

- [ Apache Commons BCEL](https://commons.apache.org/proper/commons-bcel/)：其中BCEL为Byte Code Engineering Library首字母的缩写。
- [ Javassist](http://www.javassist.org/)：Javassist表示**Java** programming **assist**ant
- [ ObjectWeb ASM](https://asm.ow2.io/)：本课程的研究对象。
- [ Byte Buddy](https://bytebuddy.net/)：在ASM基础上实现的一个类库。

​	那么，字节码的类库和ClassFile之间是什么样的关系呢？我们可以用下图来表示

![Java ASM系列：（003）ASM与ClassFile_Core API_02](https://s2.51cto.com/images/20210619/1624105555567528.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

​	不考虑中间层，我们可以说，不同的字节码类库是在同一个ClassFile结构上发展起来的（`.class`文件都必须遵循JVM定义的ClassFile的规范）。

​	对比其他字节码类库，**ASM更快更小**。

> The ASM was designed to be as **fast** and as **small** as possible.
>
> - Being as **fast** as possible is important in order not to slow down too much the applications that use ASM at runtime, for dynamic class generation or transformation.
> - And being as **small** as possible is important in order to be used in memory constrained environments, and to avoid bloating the size of small applications or libraries using ASM.

### 1.3.3 ASM与ClassFile

​	Java ClassFile相当于“树根”部分，ObjectWeb ASM相当于“树干”部分，而ASM的各种应用场景属于“树枝”或“树叶”部分。

![Java ASM系列：（003）ASM与ClassFile_ASM_03](https://s2.51cto.com/images/20210619/1624105610265567.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

​	学习ASM有三个不同的层次：

- 第一个层次，ASM的应用层面。也就是说，我们可以使用ASM来做什么呢？对于一个`.class`文件来说，我们可以使用ASM进行analysis、generation和transformation操作。
- 第二个层次，ASM的源码层面。也就是，ASM的代码组织形式，它为分Core API和Tree API的内容。
- 第三个层次，**Java ClassFile层面。从JVM规范的角度，来理解`.class`文件的结构，来理解ASM中方法和参数的含义**。

![Java ASM系列：（003）ASM与ClassFile_ASM_04](https://s2.51cto.com/images/20210619/1624105643863231.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

> 在本次课程当中，会对Class Generation和Class Transformation进行介绍。
>
> - 第二章 生成新的类，是结合Class Generation和`asm.jar`的部分内容来讲。
> - 第三章 修改已有的类，是结合Class Transformation和`asm.jar`的部分内容来讲。
> - 第四章 工具类和常用类，是结合Class Transformation、`asm-util.jar`和`asm-commons.jar`的内容来讲。

## 1.4 ClassFile快速参考

> 这小节主要需要自己实操熟悉，建议跟着视频操作一遍

### 1.4.1 Java ClassFile

对于一个具体的`.class`而言，它是遵循ClassFile结构的。这个数据结构位于[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的 [The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)部分。

```java
ClassFile {
  u4             magic; // CAFEBABE
  u2             minor_version; // 编译版本
  u2             major_version; // 编译版本，和minor_versin共同组成 compiler_version
  u2             constant_pool_count; // 常量池内成员数量
  cp_info        constant_pool[constant_pool_count-1]; // 常量池以数组形式表示
  u2             access_flags; // 访问修饰符 ACC_PUBLIC, ACC_STATIC 等
  u2             this_class; // 全限制类名，用"/"分割，而不是"."
  u2             super_class; // 父类的全限制类名，同样"/"分割
  u2             interfaces_count; // 实现的接口数量
  u2             interfaces[interfaces_count]; // 实现的接口数组
  u2             fields_count; // 当前类的成员变量数量(不会记录继承父类的字段数量)
  field_info     fields[fields_count]; // 当前类的成员变量数组
  u2             methods_count; // 当前类的方法数量，比自己定义的多1，是因为JVM默认会生成无参构造函数<init>
  method_info    methods[methods_count]; // 当前类的方法数组
  u2             attributes_count; // 属性数量
  attribute_info attributes[attributes_count]; // 属性数组，比如Java类有一个是"SourceFile"
}
```

- `u1`: 表示占用1个字节
- `u2`: 表示占用2个字节
- `u4`: 表示占用4个字节
- `u8`: 表示占用8个字节

而`cp_info`、`field_info`、`method_info`和`attribute_info`表示较为复杂的结构，但它们也是由`u1`、`u2`、`u4`和`u8`组成的。

相应的，在`.class`文件当中，定义的字段，要遵循`field_info`的结构。

```java
field_info {
    u2             access_flags; // 字段的访问修饰符，比如 [ACC_PRIVATE,ACC_STATIC,ACC_FINAL]
    u2             name_index; // 字段名(在常量池中的索引，常量池中对应索引位置能找到字符串二进制数据)
    u2             descriptor_index; // 字段描述信息，即字段类型(在常量池中的索引，常量池中对应索引位置能找到类型对应的二进制数据，比如01000149 对应I，表示int)
    u2             attributes_count; // 属性的数量
    attribute_info attributes[attributes_count]; // 属性数组，比如 private static final int intValue = 10; 的 `10`就是这个 intValue字段的一个属性
}
```

同样的，在`.class`文件当中，定义的方法，要遵循`method_info`的结构。

```java
method_info {
    u2             access_flags; // 方法的访问修饰符，比如 [ACC_PUBLIC]
    u2             name_index; // 方法名(在常量池中的索引，比如JVM生成的无参构造函数"<init>")
    u2             descriptor_index; // 方法描述信息(在常量池中的索引，包括方法参数、方法返回值，比如 "([Ljava/lang/String;)V", Ljava/lang/String; 表示方法接收1个String参数，而V表示返回值void)
    u2             attributes_count; // 属性的数量
    attribute_info attributes[attributes_count]; // 属性数组，比如2个，一个是Code(表示方法体内的代码)，另一个是MethodParameters
}
```

在`method_info`结构中，方法当中方法体的代码，是存在于`Code`属性结构中，其结构如下：

```java
Code_attribute {
    u2 attribute_name_index; // 属性名(常量池中的索引，这里即"Code")
    u4 attribute_length; // attribute_length 下面几个剩余的 定义 所占用的字节总长度
    u2 max_stack; // the maximum stack size
    u2 max_locals; // the maximum number of local variables
    u4 code_length; // 方法体内的代码字节长度(这不是指我们写的Java源代码长度，而是转译成字节码指令后占用的字节总长度)
    u1 code[code_length]; // 方法体内的 代码数组(字节码指令，而不是实际的java源代码)
    u2 exception_table_length; // 方法内异常处理的异常处理表长度(不包括方法throws声明的，方法声明的会用number_of_exceptions记录数量，exception_index_table内的exception_index记录对应的常量池索引，比如声明抛出RuntimeException，则为"java/lang/RuntimeException")
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type; // 方法内抛出的异常类型(这里实际是常量池的索引，常量池会有对应的CONSTANT_Class信息，比如"java/lang/Exception")
    } exception_table[exception_table_length]; // 方法内异常处理表数组 
    u2 attributes_count; // 属性数量
    attribute_info attributes[attributes_count]; // 属性数组，比如示例代码中包括 LineNumberTable 和 LocalVariableTable 这两个属性
}
```

### 1.4.2 示例演示

在下面内容中，我们会使用到《[Java 8 ClassFile](https://edu.51cto.com/course/25908.html)》课程的源码[java8-classfile-tutorial](https://gitee.com/lsieun/java8-classfile-tutorial)。

假如，我们有一个`sample.HelloWorld`类，它的内容如下：

```java
package sample;

public class HelloWorld implements Cloneable {
    private static final int intValue = 10;

    public void test() {
        int a = 1;
        int b = 2;
        int c = a + b;
    }
}
```

```java
// 对 sample.HelloWorld 的 Code 查看 instruction
=== === ===  === === ===  === === ===
Method test:()V
=== === ===  === === ===  === === ===
max_stack = 2
max_locals = 4
code_length = 9
code = 043C053D1B1C603EB1 // 指令对应的字节
=== === ===  === === ===  === === ===
// 最左边表示在code这个字节信息中的下标(从0开始)，右边生成的注释则是具体的字节
// iconst_X 表示 int类型的数值，比如 iconst_1表示int类型的数值1
// istore_1 表示提取栈顶部内容，加载到 LocalVariableTable 的 index 为1的槽中，即将当前stack顶部的数值1加载到变量a:I中，a为变量名，I表示类型为int
// iload_1和i_load_2分别将数据从LocalVariableTable index 1和 LocalVariableTable index 2的位置(对应变量a和b)加载到栈顶(现在栈只有2个数据，从顶到底分别是2, 1)
// iadd 将 栈顶两个数据相加，然后放回栈顶，现在栈顶只剩1个数据，即3 (1+2的结果)
0000: iconst_1             // 04
0001: istore_1             // 3C
0002: iconst_2             // 05
0003: istore_2             // 3D
0004: iload_1              // 1B
0005: iload_2              // 1C
0006: iadd                 // 60
0007: istore_3             // 3E
0008: return               // B1
=== === ===  === === ===  === === ===
LocalVariableTable:
index  start_pc  length  name_and_type
    0         0       9  this:Lsample/HelloWorld;
    1         2       7  a:I
    2         4       5  b:I
    3         8       1  c:I
```

针对`sample.HelloWorld`类，我们可以

- 第一，运行`run.A_File_Hex`类，查看`sample.HelloWorld`类当中包含的数据，以十六进制进行呈现。
- 第二，运行`run.B_ClassFile_Raw`类，能够对`sample.HelloWorld`类当中包含的数据进行拆分。这样做的目的，是为了与ClassFile的结构进行对照，进行参考。
- 第三，运行`run.I_Attributes_Method`类，能够对`sample.HelloWorld`类当中方法的Code属性结构进行查看。
- 第四，运行`run.K_Code_Locals`类，能够对`sample.HelloWorld`类当中`Code`属性包含的instruction进行查看。

### 1.4.3 小结

​	本文主要是对Java ClassFile进行了介绍，内容总结如下：

- 第一点，一个具体的`.class`文件，它是要遵循ClassFile结构的；而ClassFile的结构是定义在[The Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)文档中。
- 第二点，示例演示，主要是希望大家能够把`.class`文件里的内容与ClassFile结构之间对应起来。

​	在后续内容当中，我们会讲到从“无”到“有”的生成新的Class文件，以及对已有的Class文件进行转换；此时，我们对ClassFile进行介绍，目的就是为了对生成Class或转换Class的过程有一个较深刻的理解。

## 1.5 如何编写ASM代码

> 本小节也主要需要自己动手实操，最好跟视频练习一下

​	在刚开始学习ASM的时候，编写ASM代码是不太容易的。或者，有些人原来对ASM很熟悉，但由于长时间不使用ASM，编写ASM代码也会有一些困难。在本文当中，我们介绍一个`ASMPrint`类，它能帮助我们**将`.class`文件转换为ASM代码**，这个功能非常实用。

### 1.5.1 ASMPrint类

​	下面是`ASMPrint`类的代码，它是利用`org.objectweb.asm.util.TraceClassVisitor`类来实现的。在使用的时候，我们注意修改一下`className`、`parsingOptions`和`asmCode`参数就可以了。

```java
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.util.ASMifier;
import org.objectweb.asm.util.Printer;
import org.objectweb.asm.util.Textifier;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.IOException;
import java.io.PrintWriter;

/**
 * 这里跟着视频练习一下 <br/>
 * {@link org.objectweb.asm.util.Printer} 的 main方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/9 7:30 PM
 */
public class ASMPrint {
  public static void main(String[] args) throws IOException {
    // (1) 设置参数
    String className = "sample.HelloWorld";
    int parsingOptions = ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG;
    boolean asmCode = true;

    // (2) 打印结果
    Printer printer = asmCode ? new ASMifier() : new Textifier();
    PrintWriter printWriter = new PrintWriter(System.out, true);
    TraceClassVisitor traceClassVisitor = new TraceClassVisitor(null, printer, printWriter);
    new ClassReader(className).accept(traceClassVisitor, parsingOptions);
  }
}

```

​	在现在阶段，我们可能并不了解这段代码的含义，没有关系的。现在，我们主要是使用这个类，来帮助我们生成ASM代码；等后续内容中，我们会介绍到`TraceClassVisitor`类，也会讲到`ASMPrint`类的代码，到时候就明白这段代码的含义了。

### 1.5.2 ASMPrint类使用示例

​	假如，有如下一个`HelloWorld`类：

```java
package sample;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/9 7:35 PM
 */
public class HelloWorld {
  public void test() {
    System.out.println("Test Method");
  }
}
```

​	对于`ASMPrint`类来说，其中

- `className`值设置为类的全限定名，可以是我们自己写的类，例如`sample.HelloWorld`，也可以是JDK自带的类，例如`java.lang.Comparable`。
- `asmCode`值设置为`true`或`false`。如果是`true`，可以打印出对应的ASM代码；如果是`false`，可以打印出方法对应的Instruction。
- `parsingOptions`值设置为`ClassReader.SKIP_CODE`、`ClassReader.SKIP_DEBUG`、`ClassReader.SKIP_FRAMES`、`ClassReader.EXPAND_FRAMES`的组合值，也可以设置为`0`，可以打印出详细程度不同的信息。

# 2. 生成新的类

## 2.1 ClassVisitor介绍

​	在ASM Core API中，最重要的三个类就是`ClassReader`、`ClassVisitor`和`ClassWriter`类。在进行Class Generation操作的时候，`ClassVisitor`和`ClassWriter`这两个类起着重要作用，而并不需要`ClassReader`类的参与。在本文当中，我们将对`ClassVisitor`类进行介绍。

> 回顾一下，ClassReader负责读取`.class`类，ClassVisitor负责将读取后的类信息进行拆分/修改，ClassWriter负责将类信息重新组合后生成一个`.class`类

![Java ASM系列：（006）ClassVisitor介绍_ASM](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 2.1.1 ClassVisitor类

#### class info

第一个部分，`ClassVisitor`是一个抽象类。 由于`ClassVisitor类`是一个`abstract`类，所以不能直接使用`new`关键字创建`ClassVisitor`对象。

```
public abstract class ClassVisitor {
}
```

同时，由于`ClassVisitor`类是一个`abstract`类，要想使用它，就必须有具体的子类来继承它。

第一个比较常见的`ClassVisitor`子类是`ClassWriter`类，属于Core API：

```
public class ClassWriter extends ClassVisitor {
}
```

第二个比较常见的`ClassVisitor`子类是`ClassNode`类，属于Tree API：

```
public class ClassNode extends ClassVisitor {
}
```

三个类的关系如下：

- `org.objectweb.asm.ClassVisitor`
  - `org.objectweb.asm.ClassWriter`
  - `org.objectweb.asm.tree.ClassNode`

#### fields

​	第二个部分，`ClassVisitor`类定义的字段有哪些。

```java
public abstract class ClassVisitor {

  /**
   * The ASM API version implemented by this visitor. The value of this field must be one of the
   * {@code ASM}<i>x</i> values in {@link Opcodes}.
   */
  protected final int api;

  /** The class visitor to which this visitor must delegate method calls. May be {@literal null}. */
  protected ClassVisitor cv;
  
  // ...
}
```

- `api`字段：它是一个`int`类型的数据，指出了当前使用的ASM版本，其可取值为`Opcodes.ASM4`~`Opcodes.ASM9`。我们使用的ASM版本是9.0，因此我们在给`api`字段赋值的时候，选择`Opcodes.ASM9`就可以了。
- `cv`字段：它是一个`ClassVisitor`类型的数据，它的作用是将多个`ClassVisitor`串连起来。

![Java ASM系列：（006）ClassVisitor介绍_Java_02](https://s2.51cto.com/images/20210620/1624196959837674.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

#### constructors

第三个部分，`ClassVisitor`类定义的构造方法有哪些。

```java
public abstract class ClassVisitor {
  public ClassVisitor(final int api) {
    this(api, null);
  }

  public ClassVisitor(final int api, final ClassVisitor classVisitor) {
    this.api = api;
    this.cv = classVisitor;
  }
}
```

#### methods

> [访问者模式 | 菜鸟教程 (runoob.com)](https://www.runoob.com/design-pattern/visitor-pattern.html)

​	第四个部分，`ClassVisitor`类定义的方法有哪些。在ASM当中，使用到了**Visitor Pattern（访问者模式）**，所以`ClassVisitor`当中许多的`visitXxx()`方法。

​	虽然，在`ClassVisitor`类当中，有许多`visitXxx()`方法，但是，我们只需要关注这4个方法：**`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`**。

​	为什么只关注这4个方法呢？因为这4个方法是`ClassVisitor`类的精髓或骨架，在这个“骨架”的基础上，其它的`visitXxx()`都容易扩展；同时，将`visitXxx()`方法缩减至4个，也能减少学习过程中的认知负担，学起来更容易。

```java
public abstract class ClassVisitor {
  public void visit(
    final int version,
    final int access,
    final String name,
    final String signature,
    final String superName,
    final String[] interfaces);
  public FieldVisitor visitField( // 访问字段
    final int access,
    final String name,
    final String descriptor,
    final String signature,
    final Object value);
  public MethodVisitor visitMethod( // 访问方法
    final int access,
    final String name,
    final String descriptor,
    final String signature,
    final String[] exceptions);
  public void visitEnd();
  // ......
}
```

​	在`ClassVisitor`的`visit()`方法、`visitField()`方法和`visitMethod()`方法中都带有`signature`参数。这个<u>`signature`参数与“泛型”密切相关</u>；换句话说，如果处理的是一个带有泛型信息的类、字段或方法，那么就需要给`signature`参数提供一定的值；如果处理的类、字段或方法不带有“泛型”信息，那么将`signature`参数设置为`null`就可以了。在本次课程当中，我们不去考虑“泛型”相关的内容，所以我们都将`signature`参数设置成`null`值。

> 如果大家对`signature`参数感兴趣，我们可以使用之前介绍的`PrintASMCodeCore`类去打印一下某个泛型类的ASM代码。例如，`java.lang.Comparable`是一个泛型接口，我们就可以使用`PrintASMCodeCore`类来打印一下它的ASM代码，从来查看`signature`参数的值是什么。

### 2.1.2 方法的调用顺序

在`ClassVisitor`类当中，定义了多个`visitXxx()`方法。这些`visitXxx()`方法，遵循一定的调用顺序（可参考API文档-ClassVisitor抽象类注释上有说明）：

```
visit
[visitSource][visitModule][visitNestHost][visitPermittedSubclass][visitOuterClass]
(
 visitAnnotation |
 visitTypeAnnotation |
 visitAttribute
)*
(
 visitNestMember |
 visitInnerClass |
 visitRecordComponent |
 visitField |
 visitMethod
)* 
visitEnd
```

其中，涉及到一些符号，它们的含义如下：

- `[]`: 表示最多调用一次，可以不调用，但最多调用一次。
- `()`和`|`: 表示在多个方法之间，可以选择任意一个，并且多个方法之间不分前后顺序。
- `*`: 表示方法可以调用0次或多次。

在本次课程当中，我们只关注`ClassVisitor`类当中的`visit()`方法、`visitField()`方法、`visitMethod()`方法和`visitEnd()`方法这4个方法，所以上面的方法调用顺序可以简化如下：

```
visit
(
 visitField |
 visitMethod
)* 
visitEnd
```

也就是说，先调用`visit()`方法，接着调用`visitField()`方法或`visitMethod()`方法，最后调用`visitEnd()`方法。

### 2.1.3 visitXxx()方法与ClassFile

`ClassVisitor`的`visitXxx()`方法与`ClassFile`之间存在对应关系：

```
ClassVisitor.visitXxx() --- .class --- ClassFile
```

在`ClassVisitor`中定义的`visitXxx()`方法，并不是凭空产生的，这些方法存在的目的就是为了生成一个合法的`.class`文件，而这个`.class`文件要符合ClassFile的结构，所以这些`visitXxx()`方法与ClassFile的结构密切相关。

> 回顾一下，前面说过ASM等jar用于操作`.class`，而`.class`都必须遵循JVM定义的ClassFile结构/规则，所以ASM的ClassVisitor定义visitXxx()方法时，需要的参数也可以在ClassFIle中找到相关字段

#### visit()方法

```java
/**
   * Visits the header of the class.
   *
   * @param version the class version. The minor version is stored in the 16 most significant bits,
   *     and the major version in the 16 least significant bits.
   * @param access the class's access flags (see {@link Opcodes}). This parameter also indicates if
   *     the class is deprecated {@link Opcodes#ACC_DEPRECATED} or a record {@link
   *     Opcodes#ACC_RECORD}.
   * @param name the internal name of the class (see {@link Type#getInternalName()}).
   * @param signature the signature of this class. May be {@literal null} if the class is not a
   *     generic one, and does not extend or implement generic classes or interfaces.
   * @param superName the internal of name of the super class (see {@link Type#getInternalName()}).
   *     For interfaces, the super class is {@link Object}. May be {@literal null}, but only for the
   *     {@link Object} class.
   * @param interfaces the internal names of the class's interfaces (see {@link
   *     Type#getInternalName()}). May be {@literal null}.
   */
public void visit(
  final int version,
  final int access,
  final String name,
  final String signature,
  final String superName,
  final String[] interfaces);
```

```java
ClassFile {
  u4             magic;
  u2             minor_version;
  u2             major_version;
  u2             constant_pool_count;
  cp_info        constant_pool[constant_pool_count-1];
  u2             access_flags;
  u2             this_class;
  u2             super_class;
  u2             interfaces_count;
  u2             interfaces[interfaces_count];
  u2             fields_count;
  field_info     fields[fields_count];
  u2             methods_count;
  method_info    methods[methods_count];
  u2             attributes_count;
  attribute_info attributes[attributes_count];
}
```

<table>
<thead>
<tr>
    <th>ClassVisitor方法</th>
    <th>参数</th>
    <th>ClassFile</th>
</tr>
</thead>
<tbody>
<tr>
    <td rowspan="6"><code>ClassVisitor.visit()</code></td>
    <td><code>version</code></td>
    <td><code>minor_version</code>和<code>major_version</code></td>
</tr>
<tr>
    <td><code>access</code></td>
    <td><code>access_flags</code></td>
</tr>
<tr>
    <td><code>name</code></td>
    <td><code>this_class</code></td>
</tr>
<tr>
    <td><code>signature</code></td>
    <td><code>attributes</code>的一部分信息</td>
</tr>
<tr>
    <td><code>superName</code></td>
    <td><code>super_class</code></td>
</tr>
<tr>
    <td><code>interfaces</code></td>
    <td><code>interfaces_count</code>和<code>interfaces</code></td>
</tr>
<tr>
    <td><code>ClassVisitor.visitField()</code></td>
    <td></td>
    <td><code>field_info</code></td>
</tr>
<tr>
    <td><code>ClassVisitor.visitMethod()</code></td>
    <td></td>
    <td><code>method_info</code></td>
</tr>
</tbody>
</table>

#### visitField()方法

```java
/**
   * Visits a field of the class.
   *
   * @param access the field's access flags (see {@link Opcodes}). This parameter also indicates if
   *     the field is synthetic and/or deprecated.
   * @param name the field's name.
   * @param descriptor the field's descriptor (see {@link Type}).
   * @param signature the field's signature. May be {@literal null} if the field's type does not use
   *     generic types.
   * @param value the field's initial value. This parameter, which may be {@literal null} if the
   *     field does not have an initial value, must be an {@link Integer}, a {@link Float}, a {@link
   *     Long}, a {@link Double} or a {@link String} (for {@code int}, {@code float}, {@code long}
   *     or {@code String} fields respectively). <i>This parameter is only used for static
   *     fields</i>. Its value is ignored for non static fields, which must be initialized through
   *     bytecode instructions in constructors or methods.
   * @return a visitor to visit field annotations and attributes, or {@literal null} if this class
   *     visitor is not interested in visiting these annotations and attributes.
   */
public FieldVisitor visitField( // 访问字段
  final int access,
  final String name,
  final String descriptor,
  final String signature,
  final Object value);
```

```java
field_info {
  u2             access_flags;
  u2             name_index;
  u2             descriptor_index;
  u2             attributes_count;
  attribute_info attributes[attributes_count];
}
```

<table>
<thead>
<tr>
    <th>ClassVisitor方法</th>
    <th>参数</th>
    <th>field_info</th>
</tr>
</thead>
<tbody>
<tr>
    <td rowspan="5"><code>ClassVisitor.visitField()</code></td>
    <td><code>access</code></td>
    <td><code>access_flags</code></td>
</tr>
<tr>
    <td><code>name</code></td>
    <td><code>name_index</code></td>
</tr>
<tr>
    <td><code>descriptor</code></td>
    <td><code>descriptor_index</code></td>
</tr>
<tr>
    <td><code>signature</code></td>
    <td><code>attributes</code>的一部分信息</td>
</tr>
<tr>
    <td><code>value</code></td>
    <td><code>attributes</code>的一部分信息</td>
</tr>
</tbody>
</table>

#### visitMethod()方法

```java
/**
   * Visits a method of the class. This method <i>must</i> return a new {@link MethodVisitor}
   * instance (or {@literal null}) each time it is called, i.e., it should not return a previously
   * returned visitor.
   *
   * @param access the method's access flags (see {@link Opcodes}). This parameter also indicates if
   *     the method is synthetic and/or deprecated.
   * @param name the method's name.
   * @param descriptor the method's descriptor (see {@link Type}).
   * @param signature the method's signature. May be {@literal null} if the method parameters,
   *     return type and exceptions do not use generic types.
   * @param exceptions the internal names of the method's exception classes (see {@link
   *     Type#getInternalName()}). May be {@literal null}.
   * @return an object to visit the byte code of the method, or {@literal null} if this class
   *     visitor is not interested in visiting the code of this method.
   */
public MethodVisitor visitMethod( // 访问方法
  final int access,
  final String name,
  final String descriptor,
  final String signature,
  final String[] exceptions);
```

```java
method_info {
  u2             access_flags;
  u2             name_index;
  u2             descriptor_index;
  u2             attributes_count;
  attribute_info attributes[attributes_count];
}
```

<table>
<thead>
<tr>
    <th>ClassVisitor方法</th>
    <th>参数</th>
    <th>method_info</th>
</tr>
</thead>
<tbody>
<tr>
    <td rowspan="5"><code>ClassVisitor.visitMethod()</code></td>
    <td><code>access</code></td>
    <td><code>access_flags</code></td>
</tr>
<tr>
    <td><code>name</code></td>
    <td><code>name_index</code></td>
</tr>
<tr>
    <td><code>descriptor</code></td>
    <td><code>descriptor_index</code></td>
</tr>
<tr>
    <td><code>signature</code></td>
    <td><code>attributes</code>的一部分信息</td>
</tr>
<tr>
    <td><code>exceptions</code></td>
    <td><code>attributes</code>的一部分信息</td>
</tr>
</tbody>
</table>

#### visitEnd()方法

`visitEnd()`方法，它是这些`visitXxx()`方法当中最后一个调用的方法。

为什么`visitEnd()`方法是“最后一个调用的方法”呢？是因为在`ClassVisitor`当中，定义了多个`visitXxx()`方法，这些个`visitXxx()`方法之间要遵循一个先后调用的顺序，而`visitEnd()`方法是最后才去调用的。

等到`visitEnd()`方法调用之后，就表示说再也不去调用其它的`visitXxx()`方法了，所有的“工作”已经做完了，到了要结束的时候了。

```java
/*
 * Visits the end of the class.
 * This method, which is the last one to be called,
 * is used to inform the visitor that all the fields and methods of the class have been visited.
 */
public void visitEnd() {
    if (cv != null) {
        cv.visitEnd();
    }
}
```

### 2.1.4 小结

​	本文主要对`ClassVisitor`类进行介绍，内容总结如下：

- 第一点，介绍了`ClassVisitor`类的不同部分。我们去了解这个类不同的部分，是为了能够熟悉`ClassVisitor`这个类。
- 第二点，在`ClassVisitor`类当中，定义了许多`visitXxx()`方法，这些方法的调用要遵循一定的顺序。
- 第三点，在`ClassVisitor`类当中，定义的`visitXxx()`方法中的参数与ClassFile结构密切相关。

## 2.2 ClassWriter介绍

### 2.2.1 ClassWriter类

#### class info

​	第一个部分，就是`ClassWriter`的父类是`ClassVisitor`，因此`ClassWriter`类继承了`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`等方法。

```
/**
 * A {@link ClassVisitor} that generates a corresponding ClassFile structure, as defined in the Java
 * Virtual Machine Specification (JVMS). It can be used alone, to generate a Java class "from
 * scratch", or with one or more {@link ClassReader} and adapter {@link ClassVisitor} to generate a
 * modified class from one or more existing Java classes.
 *
 * @see <a href="https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html">JVMS 4</a>
 * @author Eric Bruneton
 */
public class ClassWriter extends ClassVisitor {
}
```

#### fields

​	第二个部分，就是`ClassWriter`定义的字段有哪些。(这里只列举和ClassFile对应的字段，省略其他字段)

```java
public class ClassWriter extends ClassVisitor {
  private int version;
  private final SymbolTable symbolTable;

  private int accessFlags;
  private int thisClass;
  private int superClass;
  private int interfaceCount;
  private int[] interfaces;

  private FieldWriter firstField;
  private FieldWriter lastField;

  private MethodWriter firstMethod;
  private MethodWriter lastMethod;

  private Attribute firstAttribute;

  //......
}
```

这些字段与ClassFile结构密切相关：

```java
ClassFile {
    u4             magic; // 固定 CAFEBABE
    u2             minor_version; // version
    u2             major_version; // version
    u2             constant_pool_count; // symbolTable, 常量池
    cp_info        constant_pool[constant_pool_count-1]; // symbolTable, 
    u2             access_flags; // accessFlags, 访问修饰符
    u2             this_class; // thisClass, 类名在常量池中的索引
    u2             super_class; // superClass, 父类名在常量池中的索引
    u2             interfaces_count; // interfaceCount, 实现的接口数量
    u2             interfaces[interfaces_count]; // interfaces, 实现的接口数组
    u2             fields_count; // firstField, lastField
    field_info     fields[fields_count]; // firstField, lastField
    u2             methods_count; // firstMethod, lastMethod
    method_info    methods[methods_count]; // firstMethod, lastMethod
    u2             attributes_count; // firstAttribute
    attribute_info attributes[attributes_count]; // firstAttribute
}
```

#### constructors

第三个部分，就是`ClassWriter`定义的构造方法。

`ClassWriter`定义的构造方法有两个，这里只关注其中一个，也就是只接收一个`int`类型参数的构造方法。在使用`new`关键字创建`ClassWriter`对象时，**推荐使用`COMPUTE_FRAMES`参数**。

> 另一个构造方法，虽然这个构造方法声称做了一些"优化", 但这些优化有时候也是负面的。
>
> 1. 将constant pool and bootstrap methods直接从origin class复制到new class，就算有些常量池中的字面值无用，也还是会被复制一份过去，相当于浪费内存（主要也是为了节省计算时间）。
> 2. 未转换的方法也会从origin class直接复制到new class（主要是为了节省计算时间），同样会存在内存浪费的问题。
>
> ```java
> // ClassWriter(final int flags) 相当于调用 ClassWriter(null, flags)
> public ClassWriter(final ClassReader classReader, final int flags);
> ```

```java
public class ClassWriter extends ClassVisitor {
    /* A flag to automatically compute the maximum stack size and the maximum number of local variables of methods. */
    public static final int COMPUTE_MAXS = 1;
    /* A flag to automatically compute the stack map frames of methods from scratch. */
    public static final int COMPUTE_FRAMES = 2;

    // flags option can be used to modify the default behavior of this class.
    // Must be zero or more of COMPUTE_MAXS and COMPUTE_FRAMES.
    public ClassWriter(final int flags) {
        this(null, flags);
    }
}
```

- `COMPUTE_MAXS`: A flag to automatically compute **the maximum stack size** and **the maximum number of local variables** of methods. If this flag is set, then the arguments of the `MethodVisitor.visitMaxs` method of the `MethodVisitor` returned by the `visitMethod` method will be ignored, and computed automatically from the signature and the bytecode of each method.
- `COMPUTE_FRAMES`: A flag to automatically compute **the stack map frames** of methods from scratch. If this flag is set, then the calls to the `MethodVisitor.visitFrame` method are ignored, and the stack map frames are recomputed from the methods bytecode. The arguments of the `MethodVisitor.visitMaxs` method are also ignored and recomputed from the bytecode. In other words, `COMPUTE_FRAMES` implies `COMPUTE_MAXS`.

小总结：

- `COMPUTE_MAXS`: 计算max stack和max local信息。
- **`COMPUTE_FRAMES`: 既计算stack map frame信息，又计算max stack和max local信息。**

换句话说，`COMPUTE_FRAMES`是功能最强大的：

```
COMPUTE_FRAMES = COMPUTE_MAXS + stack map frame
```

#### methods

​	第四个部分，就是`ClassWriter`提供了哪些方法。

##### visitXxx()方法

在`ClassWriter`这个类当中，我们仍然是只关注其中的`visit()`方法、`visitField()`方法、`visitMethod()`方法和`visitEnd()`方法。

这些`visitXxx()`方法的调用，就是在为构建ClassFile提供“原材料”的过程。

```java
public class ClassWriter extends ClassVisitor {
  public void visit(
    final int version,
    final int access,
    final String name,
    final String signature,
    final String superName,
    final String[] interfaces);
  public FieldVisitor visitField( // 访问字段
    final int access,
    final String name,
    final String descriptor,
    final String signature,
    final Object value);
  public MethodVisitor visitMethod( // 访问方法
    final int access,
    final String name,
    final String descriptor,
    final String signature,
    final String[] exceptions);
  public void visitEnd();
  // ......
}
```

##### toByteArray()方法

在`ClassWriter`类当中，提供了一个`toByteArray()`方法。这个方法的作用是将“所有的努力”（对`visitXxx()`的调用）转换成`byte[]`，而这些`byte[]`的内容就遵循ClassFile结构。

在`toByteArray()`方法的代码当中，通过三个步骤来得到`byte[]`：

- 第一步，计算`size`大小。这个`size`就是表示`byte[]`的最终的长度是多少。
- 第二步，将数据填充到`byte[]`当中。
- 第三步，将`byte[]`数据返回。

```java
public class ClassWriter extends ClassVisitor {
  public byte[] toByteArray() {

    // First step: compute the size in bytes of the ClassFile structure.
    // The magic field uses 4 bytes, 10 mandatory fields (minor_version, major_version,
    // constant_pool_count, access_flags, this_class, super_class, interfaces_count, fields_count,
    // methods_count and attributes_count) use 2 bytes each, and each interface uses 2 bytes too.
    int size = 24 + 2 * interfaceCount;
    int fieldsCount = 0;
    FieldWriter fieldWriter = firstField;
    while (fieldWriter != null) {
      ++fieldsCount;
      size += fieldWriter.computeFieldInfoSize();
      fieldWriter = (FieldWriter) fieldWriter.fv;
    }
    int methodsCount = 0;
    MethodWriter methodWriter = firstMethod;
    while (methodWriter != null) {
      ++methodsCount;
      size += methodWriter.computeMethodInfoSize();
      methodWriter = (MethodWriter) methodWriter.mv;
    }

    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    int attributesCount = 0;

    // ......

    if (firstAttribute != null) {
      attributesCount += firstAttribute.getAttributeCount();
      size += firstAttribute.computeAttributesSize(symbolTable);
    }
    // IMPORTANT: this must be the last part of the ClassFile size computation, because the previous
    // statements can add attribute names to the constant pool, thereby changing its size!
    size += symbolTable.getConstantPoolLength();


    // Second step: allocate a ByteVector of the correct size (in order to avoid any array copy in
    // dynamic resizes) and fill it with the ClassFile content.
    ByteVector result = new ByteVector(size);
    result.putInt(0xCAFEBABE).putInt(version);
    symbolTable.putConstantPool(result);
    int mask = (version & 0xFFFF) < Opcodes.V1_5 ? Opcodes.ACC_SYNTHETIC : 0;
    result.putShort(accessFlags & ~mask).putShort(thisClass).putShort(superClass);
    result.putShort(interfaceCount);
    for (int i = 0; i < interfaceCount; ++i) {
      result.putShort(interfaces[i]);
    }
    result.putShort(fieldsCount);
    fieldWriter = firstField;
    while (fieldWriter != null) {
      fieldWriter.putFieldInfo(result);
      fieldWriter = (FieldWriter) fieldWriter.fv;
    }
    result.putShort(methodsCount);
    boolean hasFrames = false;
    boolean hasAsmInstructions = false;
    methodWriter = firstMethod;
    while (methodWriter != null) {
      hasFrames |= methodWriter.hasFrames();
      hasAsmInstructions |= methodWriter.hasAsmInstructions();
      methodWriter.putMethodInfo(result);
      methodWriter = (MethodWriter) methodWriter.mv;
    }
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    result.putShort(attributesCount);

    // ......

    if (firstAttribute != null) {
      firstAttribute.putAttributes(symbolTable, result);
    }

    // 这里的if，是因为ASM自己也定义了类似JVM要求的Opcode的一套ASM instructions，ASM中间构造时使用自己的instructions, 最后再统一转换回JVM要求的Opcode
    // Third step: replace the ASM specific instructions, if any.
    if (hasAsmInstructions) {
      return replaceAsmInstructions(result.data, hasFrames);
    } else {
      return result.data;
    }
  }
}
```

### 2.2.2 创建ClassWriter对象

#### 推荐使用COMPUTE_FRAMES

在创建`ClassWriter`对象的时候，要指定一个`flags`参数，它可以选择的值有三个：

- 第一个，可以选取的值是`0`。ASM不会自动计算max stacks和max locals，也不会自动计算stack map frames。
- 第二个，可以选取的值是`ClassWriter.COMPUTE_MAXS`。ASM会自动计算max stacks和max locals，但不会自动计算stack map frames。
- 第三个，可以选取的值是`ClassWriter.COMPUTE_FRAMES`（推荐使用）。ASM会自动计算max stacks和max locals，也会自动计算stack map frames。

| flags          | max stacks and max locals | stack map frames |
| -------------- | ------------------------- | ---------------- |
| 0              | NO                        | NO               |
| COMPUTE_MAXS   | YES                       | NO               |
| COMPUTE_FRAMES | YES                       | YES              |

#### 为什么推荐使用COMPUTE_FRAMES

​	在创建`ClassWriter`对象的时候，使用`ClassWriter.COMPUTE_FRAMES`，ASM会自动计算max stacks和max locals，也会自动计算stack map frames。

​	首先，来看一下max stacks和max locals。在ClassFile结构中，每一个方法都用`method_info`来表示，而方法里定义的代码则使用`Code`属性来表示，其结构如下：

```java
Code_attribute {
  u2 attribute_name_index;
  u4 attribute_length;
  u2 max_stack;     // 这里是max stacks, the maximum stack size
  u2 max_locals;    // 这里是max locals, the maximum number of local variables
  u4 code_length;
  u1 code[code_length];
  u2 exception_table_length;
  {   u2 start_pc;
   u2 end_pc;
   u2 handler_pc;
   u2 catch_type;
  } exception_table[exception_table_length];
  u2 attributes_count;
  attribute_info attributes[attributes_count];
}
```

​	如果我们在创建`ClassWriter(flags)`对象的时候，将`flags`参数设置为`ClassWriter.COMPUTE_MAXS`或`ClassWriter.COMPUTE_FRAMES`，那么ASM会自动帮助我们计算`Code`结构中`max_stack`和`max_locals`的值。

​	接着，来看一下stack map frames。在`Code`结构里，可能有多个`attributes`，其中一个可能就是`StackMapTable_attribute`。**`StackMapTable_attribute`结构，就是stack map frame具体存储格式，它的主要作用是对ByteCode进行类型检查**。

```java
StackMapTable_attribute {
  u2              attribute_name_index;
  u4              attribute_length;
  u2              number_of_entries;
  stack_map_frame entries[number_of_entries];
}
```

​	如果我们在创建`ClassWriter(flags)`对象的时候，将`flags`参数设置为`ClassWriter.COMPUTE_FRAMES`，那么ASM会自动帮助我们计算`StackMapTable_attribute`的内容。

![Java ASM系列：（007）ClassWriter介绍_Java](https://s2.51cto.com/images/20210621/1624254673117990.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

- 如果将`flags`参数的取值为`0`，那么我们就必须要提供正确的max stacks、max locals和stack map frame的值；
- 如果将`flags`参数的取值为`ClassWriter.COMPUTE_MAXS`，那么ASM会自动帮助我们计算max stacks和max locals，而我们则需要提供正确的stack map frame的值。

> ​	那么，ASM为什么会提供`0`和`ClassWriter.COMPUTE_MAXS`这两个选项呢？因为ASM在计算这些值的时候，要考虑各种各样不同的情况，所以它的算法相对来说就比较复杂，因而执行速度也会相对较慢。同时，ASM也鼓励开发者去研究更好的算法；如果开发者有更好的算法，就可以不去使用`ClassWriter.COMPUTE_FRAMES`，这样就能让程序的执行效率更高效。

​	要想计算max stacks、max locals和stack map frames，也不是一件容易的事情。出于方便的目的，就推荐大家使用`ClassWriter.COMPUTE_FRAMES`。在大多数情况下，`ClassWriter.COMPUTE_FRAMES`都能帮我们计算出正确的值。**在少数情况下，`ClassWriter.COMPUTE_FRAMES`也可能会出错，比如说，有些代码经过混淆（obfuscate）处理，它里面的stack map frame会变更非常复杂，使用`ClassWriter.COMPUTE_FRAMES`就会出现错误的情况**。针对这种少数的情况，我们可以在不改变原有stack map frame的情况下，使用`ClassWriter.COMPUTE_MAXS`，让ASM只帮助我们计算max stacks和max locals。

### 2.2.3 如何使用ClassWriter类

使用`ClassWriter`生成一个Class文件，可以大致分成三个步骤：

- 第一步，创建`ClassWriter`对象。
- 第二步，调用`ClassWriter`对象的`visitXxx()`方法。
- 第三步，调用`ClassWriter`对象的`toByteArray()`方法。

示例代码如下：

```java
import org.objectweb.asm.ClassWriter;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 4:45 PM
 */
public class HelloWorldGenerateCore {
    public static byte[] dump() throws Exception{
        // (1) 创建ClassWriter对象
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
      
        // (2) 调用 visitXxx()方法, 实际除了visitEnd，其他visitXxx需要传递参数，这里主要指事为了说明调用方法的流程
        // 可以调用 "1.5.1 ASMPrint类" 的 ASMPrint 来输出 ASM代码, 照猫画虎
        cw.visit();
        cw.visitField();
        cw.visitMethod();
        cw.visitEnd();

        // (3) 调用 toByteArray()方法
        return cw.toByteArray();
    }
}

```

## 2.3 ClassWriter代码示例

![Java ASM系列：（008）ClassWriter代码示例_ASM](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

​	在当前阶段，我们只能进行Class Generation的操作。

### 2.3.1 示例一：生成接口

#### 预期目标

​	我们的预期目标：生成`HelloWorld`接口。

```java
public interface HelloWorld {
}
```

#### 代码实现

注意点：

+ 访问修饰符，需要声明`ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE`

+ **interface接口，superName(父类名)需要指定为`Object`**

  (试了下，如果指定superName为null，下面使用`Class.forName`会报错说`Exception in thread "main" java.lang.ClassFormatError: Invalid superclass index 0 in class file sample/HelloWorld`。这里指定null后光看`.class`文件反编译后是没有区别的，很容易踩坑，可以用`javap -v -p {sample/HelloWorld.class}` 查看区别，会发现指定superName为Object时，`.class`里面会多出`super_class`值为`java/lang/Object`)

```java
import org.objectweb.asm.ClassWriter;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(
      // version
      V17, 
      // access
      ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE, 
      // name
      "sample/HelloWorld", 
      // signature
      null, 
      // superName (必须指定Object，否则后续Class.forName抛出异常)
      "java/lang/Object", 
      // interfaces
      null);
    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

​	在上述代码中，我们调用了`visit()`方法、`visitEnd()`方法和`toByteArray()`方法。

​	由于`sample.HelloWorld`这个接口中，并没有定义任何的字段和方法，因此，在上述代码中没有调用`visitField()`方法和`visitMethod()`方法。

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws ClassNotFoundException {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);
  }
}
```

#### 小结

##### visit()方法

​	在这里，我们重点介绍一下`visit(version, access, name, signature, superName, interfaces)`方法的各个参数：

- `version`: 表示当前类的版本信息。在上述示例代码中，其取值为`Opcodes.V17`，表示使用Java 17版本。
- `access`: 表示当前类的访问标识（access flag）信息。在上面的示例中，`access`的取值是`ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE`，也可以写成`ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE`。如果想进一步了解这些标识的含义，可以参考[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的"Chapter 4. The class File Format"部分。
- `name`: 表示当前类的名字，它采用的格式是**Internal Name**的形式。
- `signature`: 表示当前类的泛型信息。因为在这个接口当中不包含任何的泛型信息，因此它的值为`null`。
- `superName`: 表示当前类的父类信息，它采用的格式是**Internal Name**的形式。
- `interfaces`: 表示当前类实现了哪些接口信息。

##### Internal Name

​	同时，我们也要介绍一下Internal Name的概念：在`.java`文件中，我们使用Java语言来编写代码，使用类名的形式是**Fully Qualified Class Name**，例如`java.lang.String`；将`.java`文件编译之后，就会生成`.class`文件；在`.class`文件中，类名的形式会发生变化，称之为**Internal Name**，例如`java/lang/String`。因此，将**Fully Qualified Class Name**转换成**Internal Name**的方式就是，将`.`字符转换成`/`字符。

|          | Java Language              | Java ClassFile     |
| -------- | -------------------------- | ------------------ |
| 文件格式 | `.java`                    | `.class`           |
| 类名     | Fully Qualified Class Name | Internal Name      |
| 类名示例 | `java.lang.String`         | `java/lang/String` |

### 2.3.2 示例二：生成接口+字段+方法

#### 预期目标

我们的预期目标：生成`HelloWorld`接口。

```java
public interface HelloWorld extends Cloneable {
    int LESS = -1;
    int EQUAL = 0;
    int GREATER = 1;
    int compareTo(Object o);
}
```

#### 编码实现

注意点：

+ 虽然是接口，但是需要指定父类为Object
+ `visitField()`会返回FieldVisitor，记得再调用`visitEnd()`，（虽然我试了下，不调用也没有任何影响）
+ `visitMethod()`会返回MethodVisitor，记得再调用`visitEnd()`，（虽然我试了下，不调用也没有任何影响）

```java
import org.objectweb.asm.ClassWriter;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE, "sample/HelloWorld", null, "java/lang/Object", new String[]{"java/lang/Cloneable"});

    // access, name, descriptor, signature, value, 返回一个 FieldVisitor 抽象类
    cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "LESS", "I", null, -1)
      .visitEnd();
    cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "EQUAL", "I", null, 0)
      .visitEnd();
    cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "GREATER", "I", null, 1)
      .visitEnd();

    // access, name, descriptor, signature, exceptions, 返回一个 MethodVisitor 抽象类
    cw.visitMethod(ACC_PUBLIC | ACC_ABSTRACT, "compareTo", "(Ljava/lang/Object;)I", null, null)
      .visitEnd();

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

​	在上述代码中，我们调用了`visit()`方法、`visitField()`方法、`visitMethod()`方法、`visitEnd()`方法和`toByteArray()`方法。

#### 验证结果

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 6:17 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) throws ClassNotFoundException {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);
    Field[] declaredFields = clazz.getDeclaredFields();
    if(declaredFields.length > 0) {
      System.out.println("fields: ");
      for(Field field: declaredFields ) {
        System.out.println(" " + field);
      }
    }
    Method[] declaredMethods = clazz.getDeclaredMethods();
    if(declaredMethods.length > 0) {
      System.out.println("methods: ");
      for(Method method: declaredMethods) {
        System.out.println(" " + method);
      }
    }
  }
}
```

输出结果

```shell
interface sample.HelloWorld
fields: 
 public static final int sample.HelloWorld.LESS
 public static final int sample.HelloWorld.EQUAL
 public static final int sample.HelloWorld.GREATER
methods: 
 public abstract int sample.HelloWorld.compareTo(java.lang.Object)
```

#### 小结

##### visitField()和visitMethod()方法

在这里，我们重点说一下`visitField()`方法和`visitMethod()`方法的各个参数：

- `visitField (access, name, descriptor, signature, value)`
- `visitMethod(access, name, descriptor, signature, exceptions)`

这两个方法的前4个参数是相同的，不同的地方只在于第5个参数。

- `access`参数：表示当前字段或方法带有的访问标识（access flag）信息，例如`ACC_PUBLIC`、`ACC_STATIC`和`ACC_FINAL`等。
- `name`参数：表示当前字段或方法的名字。
- **`descriptor`参数：表示当前字段或方法的描述符。这些描述符，与我们平时使用的Java类型是有区别的（字段的描述符，描述字段的类型；方法的描述符，描述方法的请求参数和返回值）**。
- `signature`参数：表示当前字段或方法是否带有泛型信息。换句话说，如果不带有泛型信息，提供一个`null`就可以了；如果带有泛型信息，就需要给它提供某一个具体的值。
- `value`参数：是`visitField()`方法的第5个参数。这个参数的取值，与当前字段是否为常量有关系。如果当前字段是一个常量，就需要给`value`参数提供某一个具体的值；如果当前字段不是常量，那么使用`null`就可以了。
- **`exceptions`参数：是`visitMethod()`方法的第5个参数。这个参数的取值，与当前方法头（Method Header）中是否具有`throws XxxException`相关。**

我们可以使用`PrintASMCodeCore`类来查看下面的`sample.HelloWorld`类的ASM代码，从而观察`value`参数和`exceptions`参数的取值：

```java
import java.io.FileNotFoundException;
import java.io.IOException;

public class HelloWorld {
  // 这是一个常量字段，使用static、final关键字修饰
  public static final int constant_field = 10;
  // 这是一个非常量字段
  public int non_constant_field;

  public void test() throws FileNotFoundException, IOException {
    // do nothing
  }
}
```

对于上面的代码，

- `constant_field`字段：对应于`visitField(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "constant_field", "I", null, new Integer(10))`
- `non_constant_field`字段：对应于`visitField(ACC_PUBLIC, "non_constant_field", "I", null, null)`
- `test()`方法：对应于`visitMethod(ACC_PUBLIC, "test", "()V", null, new String[] { "java/io/FileNotFoundException", "java/io/IOException" })`

##### 描述符（descriptor）

在ClassFile当中，描述符（descriptor）是对“类型”的简单化描述。

- 对于字段（field）来说，描述符（descriptor）就是对**字段本身的类型**进行简单化描述。
- 对于方法（method）来说，描述符（descriptor）就是对方法的**接收参数的类型**和**返回值的类型**进行简单化描述。

| Java类型              | ClassFile描述符                                 |
| --------------------- | ----------------------------------------------- |
| `boolean`             | `Z`（Z表示Zero，零表示`false`，非零表示`true`） |
| `byte`                | `B`                                             |
| `char`                | `C`                                             |
| `double`              | `D`                                             |
| `float`               | `F`                                             |
| `int`                 | `I`                                             |
| `long`                | `J`                                             |
| `short`               | `S`                                             |
| `void`                | `V`                                             |
| `non-array reference` | `L<InternalName>;`                              |
| `array reference`     | `[`                                             |

对字段描述符的举例：

- `boolean flag`: `Z`
- `byte byteValue`: `B`
- `int intValue`: `I`
- `float floatValue`: `F`
- `double doubleValue`: `D`
- `String strValue`: `Ljava/lang/String;`
- `Object objValue`: `Ljava/lang/Object;`
- `byte[] bytes`: `[B`
- `String[] array`: `[Ljava/lang/String;`
- `Object[][] twoDimArray`: `[[Ljava/lang/Object;`

对方法描述符的举例：

- `int add(int a, int b)`: `(II)I`
- `void test(int a, int b)`: `(II)V`
- `boolean compare(Object obj)`: `(Ljava/lang/Object;)Z`
- `void main(String[] args)`: `([Ljava/lang/String;)V`

### 2.3.3 示例三：生成类

#### 预期目标

​	我们的预期目标：生成`HelloWorld`类。

```java
public class HelloWorld {
}
```

#### 编码实现

注意点：

+ **使用ASM操作字节码时，需要显式添加无参构造方法**（平时写java文件，则是JVM会自动生成无参构造方法）

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // access, name, descriptor, signature, exceptions, 返回一个 MethodVisitor 抽象类
    MethodVisitor methodVisitor = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
    methodVisitor.visitCode();
    // 下面2行调用父类Object的构造方法，相当于java代码的 super()
    methodVisitor.visitVarInsn(ALOAD,0);
    methodVisitor.visitMethodInsn(INVOKESPECIAL,"java/lang/Object", "<init>", "()V", false);
    // 在当前类的 构造方法执行return
    methodVisitor.visitInsn(RETURN);
    methodVisitor.visitMaxs(1,1);
    methodVisitor.visitEnd();

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws ClassNotFoundException {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);
  }
}
```

### 2.3.4 总结

本文主要对`ClassWriter`类的代码示例进行介绍，主要目的是希望大家能够对`ClassWriter`类熟悉起来。

本文内容总结如下：

- 第一点，我们需要注意`ClassWriter`/`ClassVisitor`中`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`方法的调用顺序。
- 第二点，我们对于`visit()`方法、`visitField()`方法和`visitMethod()`方法接收的参数进行了介绍。虽然我们并没有特别介绍`visitEnd()`方法和`toByteArray()`方法，并不表示这两个方法不重要，只是因为这两个方法不接收任何参数。
- <u>第三点，我们介绍了Internal Name和Descriptor（描述符）这两个概念，在使用时候需要加以注意，因为它们与我们在使用Java语言编写代码时是不一样的</u>。
- **第四点，在`.class`文件中，构造方法的名字是`<init>()`，表示instance initialization method；静态代码块的名字是`<clinit>()`，表示class initialization method**。

另外，`visitField()`方法会返回一个`FieldVisitor`对象，而`visitMethod()`方法会返回一个`MethodVisitor`对象；在后续的内容当中，我们会分别介绍`FieldVisitor`类和`MethodVisitor`类。

## 2.4 FieldVisitor介绍

在调用`ClassVistor`的`visitField()`方法时返回`FieldVisitor`实例；调用`ClassVistor`的`visitMethod()`方法时返回`MethodVisitor`实例。

`ClassWriter`是抽象类`ClassVistor`的实现类，`FieldWriter`是`FieldVisitor`的实现类，`MethodWriter`是`MethodVisitor`的实现类。

查看`ClassWriter`的`visitField()`方法实现时，也会发现其内部构造`FieldWriter`实例：

```java
@Override
public final FieldVisitor visitField(
  final int access,
  final String name,
  final String descriptor,
  final String signature,
  final Object value) {
  FieldWriter fieldWriter =
    new FieldWriter(symbolTable, access, name, descriptor, signature, value);
  if (firstField == null) {
    firstField = fieldWriter;
  } else {
    lastField.fv = fieldWriter;
  }
  return lastField = fieldWriter;
}
```

![Java ASM系列：（003）ASM与ClassFile_ASM_04](https://s2.51cto.com/images/20210619/1624105643863231.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 2.4.1 FieldVisitor类

​	**在学习`FieldVisitor`类的时候，可以与`ClassVisitor`类进行对比，这两个类在结构上有很大的相似性：两者都是抽象类，都定义了两个字段，都定义了两个构造方法，都定义了`visitXxx()`方法**。

#### class info

第一个部分，`FieldVisitor`类是一个`abstract`类。

```java
/**
 * A visitor to visit a Java field. The methods of this class must be called in the following order:
 * ( {@code visitAnnotation} | {@code visitTypeAnnotation} | {@code visitAttribute} )* {@code
 * visitEnd}.
 *
 * @author Eric Bruneton
 */
public abstract class FieldVisitor {
}
```

#### fields

第二个部分，`FieldVisitor`类定义的字段有哪些。

```java
public abstract class FieldVisitor {
  /**
   * The ASM API version implemented by this visitor. The value of this field must be one of the
   * {@code ASM}<i>x</i> values in {@link Opcodes}.
   */
  protected final int api;
  /** The field visitor to which this visitor must delegate method calls. May be {@literal null}. */
  protected FieldVisitor fv;
}
```

#### constructors

第三个部分，`FieldVisitor`类定义的构造方法有哪些。

```java
public abstract class FieldVisitor {
    public FieldVisitor(final int api) {
        this(api, null);
    }

    public FieldVisitor(final int api, final FieldVisitor fieldVisitor) {
        this.api = api;
        this.fv = fieldVisitor;
    }
}
```

#### methods

第四个部分，`FieldVisitor`类定义的方法有哪些。

在`FieldVisitor`类当中，一共定义了4个`visitXxx()`方法，但是，我们只需要关注其中的`visitEnd()`方法就可以了。

我们为什么只关注`visitEnd()`方法呢？因为我们刚开始学习ASM，有许多东西不太熟悉，为了减少我们的学习和认知“负担”，那么对于一些非必要的方法，我们就暂时忽略它；将`visitXxx()`方法精简到一个最小的认知集合，那么就只剩下`visitEnd()`方法了。

```java
public abstract class FieldVisitor {
  // ......

  /**
   * Visits the end of the field. This method, which is the last one to be called, is used to inform
   * the visitor that all the annotations and attributes of the field have been visited.
   */
  public void visitEnd() {
    if (fv != null) {
      fv.visitEnd();
    }
  }
}
```

另外，在`FieldVisitor`类内定义的多个`visitXxx()`方法，也需要遵循一定的调用顺序，如下所示：

```pseudocode
(
 visitAnnotation |
 visitTypeAnnotation |
 visitAttribute
)*
visitEnd
```

由于我们只关注`visitEnd()`方法，那么，这个调用顺序就变成如下这样：

```
visitEnd
```

### 2.4.2 示例一：字段常量

#### 预期目标

```java
public interface HelloWorld {
    int intValue = 100;
    String strValue = "ABC";
}
```

#### 编码实现

注意点：

+ `ClassWriter`对象调用`visitField()`会返回一个FieldVisitor对象，记得调用其`visitEnd()`方法

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE, "sample/HelloWorld", null, "java/lang/Object", null);

    // access, name, descriptor, signature, value (记得最后调用 visitEnd())
    FieldVisitor fv1 = cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "intValue","I",null,100);
    fv1.visitEnd();
    FieldVisitor fv2 = cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "strValue","Ljava/lang/String;",null,"ABC");
    fv2.visitEnd();

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

验证结果

```java
import java.lang.reflect.Field;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    MyClassLoader classLoader = new MyClassLoader();
    Class<?> clazz = classLoader.loadClass("sample.HelloWorld");
    System.out.println(clazz);
    Field[] declaredFields = clazz.getDeclaredFields();
    if(declaredFields.length > 0) {
      System.out.println("fields:");
      for(Field field : declaredFields) {
        Object value = field.get(null);
        System.out.println(field + ": " + value);
      }
    }
  }
}

```

输出结果

```shell
interface sample.HelloWorld
fields:
public static final int sample.HelloWorld.intValue: 100
public static final java.lang.String sample.HelloWorld.strValue: ABC
```

### 2.4.3 示例二：visitAnnotation

​	无论是`ClassVisitor`类，还是`FieldVisitor`类，又或者是`MethodVisitor`类，总会有一些`visitXxx()`方法是在课程当中不会涉及到的。但是，在日后的工作和学习当中，很可能，在某一天你突然就对一个`visitXxx()`方法产生了兴趣，那该如何学习这个`visitXxx()`方法呢？我们可以借助于`ASMPrint`类（先借住ASMPrint生成ASM代码，然后再对照学习`vistXxx()`）。

#### 预期目标

假如我们想生成如下`HelloWorld`类：

```java
public interface HelloWorld {
    @MyTag(name = "tomcat", age = 10)
    int intValue = 100;
}
```

其中，`MyTag`定义如下：

```java
public @interface MyTag {
  String name();
  int age();
}
```

#### 编码实现

注意点：

+ 调用`FieldVistor`的`visitAnnotation()`方法返回`AnnotationVisitor`实例，最后也需记得调用其`visitEnd()`方法
+ 可以用`javap -v -p sample/HelloWorld.class`对比一下`visitAnnotation("Lsample/MyTag", false)`和`visitAnnotation("Lsample/MyTag", true)`的区别

```java
import org.objectweb.asm.AnnotationVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE, "sample/HelloWorld", null, "java/lang/Object", null);

    // access, name, descriptor, signature, value (记得最后调用 visitEnd())
    FieldVisitor fv = cw.visitField(ACC_PUBLIC | ACC_STATIC | ACC_FINAL, "intValue","I",null,100);
    {
      // descriptor, visible
      AnnotationVisitor av = fv.visitAnnotation("Lsample/MyTag", false);
      // name, value
      av.visit("name", "tomcat");
      av.visit("age", 10);
      av.visitEnd();
    }
    fv.visitEnd();

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}
```

`visitAnnotation("Lsample/MyTag", false)`和`visitAnnotation("Lsample/MyTag", true)`区别

```java
Classfile sample/HelloWorld.class
  Last modified Apr 13, 2023; size 216 bytes
  SHA-256 checksum 7f272d63ae89f50e5ab3275c9416846b7965417289fccb27f3aec8255f85e6f1
public interface sample.HelloWorld
  minor version: 0
  major version: 61
  flags: (0x0601) ACC_PUBLIC, ACC_INTERFACE, ACC_ABSTRACT
  this_class: #2                          // sample/HelloWorld
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 0, attributes: 0
Constant pool:
   #1 = Utf8               sample/HelloWorld
   #2 = Class              #1             // sample/HelloWorld
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               intValue
   #6 = Utf8               I
   #7 = Integer            100
   #8 = Utf8               Lsample/MyTag
   #9 = Utf8               name
  #10 = Utf8               tomcat
  #11 = Utf8               age
  #12 = Integer            10
  #13 = Utf8               ConstantValue
  #14 = Utf8               RuntimeInvisibleAnnotations //如果是 true则这里是 RuntimeVisibleAnnotations
{
  public static final int intValue;
    descriptor: I
    flags: (0x0019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 100
    RuntimeInvisibleAnnotations: //如果是 true 则这里是 RuntimeVisibleAnnotations:
      0: #8(#9=s#10,#11=I#12)
        #8(
          name="tomcat"
          age=10
        )

}
```

### 2.4.4 总结

本文主要对`FieldVisitor`类进行了介绍，内容总结如下：

- 第一点，`FieldVisitor`类，从结构上来说，与`ClassVisitor`很相似；对于`FieldVisitor`类的各个不同部分进行介绍，以便从整体上来理解`FieldVisitor`类。
- 第二点，对于`FieldVisitor`类定义的方法，我们只需要关心`FieldVisitor.visitEnd()`方法就可以了。
- 第三点，我们可以借助于`ASMPrint`类来帮助我们学习新的`visitXxx()`方法。

## 2.5 FieldWriter介绍

`FieldWriter`类继承自`FieldVisitor`类。在`ClassWriter`类里，`visitField()`方法的实现就是通过`FieldWriter`类来实现的。

### 2.5.1 FieldWriter类

#### class info

​	第一个部分，`FieldWriter`类的父类是`FieldVisitor`类。<u>需要注意的是，`FieldWriter`类并不带有`public`修饰，因此它的有效访问范围只局限于它所处的package当中，不能像其它的`public`类一样被外部所使用</u>。

```java
/**
 * A {@link FieldVisitor} that generates a corresponding 'field_info' structure, as defined in the
 * Java Virtual Machine Specification (JVMS).
 *
 * @see <a href="https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.5">JVMS
 *     4.5</a>
 * @author Eric Bruneton
 */
final class FieldWriter extends FieldVisitor {
}
```

#### fields

第二个部分，`FieldWriter`类定义的字段有哪些。在`FieldWriter`类当中，一些字段如下：

```java
final class FieldWriter extends FieldVisitor {
    private final int accessFlags; // 访问修饰符 access_flags
    private final int nameIndex; // 字段名在常量池中的索引下标 name_index
    private final int descriptorIndex; // 描述信息，字段的描述信息就是字段类型 descriptor_index
    private Attribute firstAttribute; // 属性(字段值等) attributes_count attributes[attributes_count]
}
```

这些字段与ClassFile当中的`field_info`是对应的：

```java
field_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### constructors

第三个部分，`FieldWriter`类定义的构造方法有哪些。在`FieldWriter`类当中，只定义了一个构造方法；同时，它也不带有`public`标识，只能在package内使用。

```java
/**
   * Constructs a new {@link FieldWriter}.
   *
   * @param symbolTable where the constants used in this FieldWriter must be stored.
   * @param access the field's access flags (see {@link Opcodes}).
   * @param name the field's name.
   * @param descriptor the field's descriptor (see {@link Type}).
   * @param signature the field's signature. May be {@literal null}.
   * @param constantValue the field's constant value. May be {@literal null}.
   */
final class FieldWriter extends FieldVisitor {
  FieldWriter(SymbolTable symbolTable, int access, String name, String descriptor, String signature, Object constantValue) {
    super(Opcodes.ASM9);
    this.symbolTable = symbolTable;
    this.accessFlags = access;
    this.nameIndex = symbolTable.addConstantUtf8(name);
    this.descriptorIndex = symbolTable.addConstantUtf8(descriptor);
    if (signature != null) {
      this.signatureIndex = symbolTable.addConstantUtf8(signature);
    }
    if (constantValue != null) {
      this.constantValueIndex = symbolTable.addConstant(constantValue).index;
    }
  }
}
```

#### methods

第四个部分，`FieldWriter`类定义的方法有哪些。在`FieldWriter`类当中，有两个重要的方法：`computeFieldInfoSize()`和`putFieldInfo()`方法。这两个方法会在`ClassWriter`类的`toByteArray()`方法内使用到。

```java
final class FieldWriter extends FieldVisitor {
  /**
   * Returns the size of the field_info JVMS structure generated by this FieldWriter. Also adds the
   * names of the attributes of this field in the constant pool.
   *
   * @return the size in bytes of the field_info JVMS structure.
   */
  int computeFieldInfoSize() {
    // The access_flags, name_index, descriptor_index and attributes_count fields use 8 bytes.
    int size = 8;
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    if (constantValueIndex != 0) {
      // ConstantValue attributes always use 8 bytes.
      symbolTable.addConstantUtf8(Constants.CONSTANT_VALUE);
      size += 8;
    }
    // ......
    return size;
  }

  /**
   * Puts the content of the field_info JVMS structure generated by this FieldWriter into the given
   * ByteVector.
   *
   * @param output where the field_info structure must be put.
   */
  void putFieldInfo(final ByteVector output) {
    boolean useSyntheticAttribute = symbolTable.getMajorVersion() < Opcodes.V1_5;
    // Put the access_flags, name_index and descriptor_index fields.
    int mask = useSyntheticAttribute ? Opcodes.ACC_SYNTHETIC : 0;
    output.putShort(accessFlags & ~mask).putShort(nameIndex).putShort(descriptorIndex);
    // Compute and put the attributes_count field.
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    int attributesCount = 0;
    if (constantValueIndex != 0) {
      ++attributesCount;
    }
    // ......
    output.putShort(attributesCount);
    // Put the field_info attributes.
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    if (constantValueIndex != 0) {
      output
        .putShort(symbolTable.addConstantUtf8(Constants.CONSTANT_VALUE))
        .putInt(2)
        .putShort(constantValueIndex);
    }
    // ......
  }
}
```

### 2.5.2 FieldWriter类的使用

关于`FieldWriter`类的使用，它主要出现在`ClassWriter`类当中的`visitField()`和`toByteArray()`方法内。

#### visitField方法

在`ClassWriter`类当中，`visitField()`方法代码如下：

```java
public class ClassWriter extends ClassVisitor {
  public final FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    FieldWriter fieldWriter = new FieldWriter(symbolTable, access, name, descriptor, signature, value);
    if (firstField == null) {
      firstField = fieldWriter;
    } else {
      lastField.fv = fieldWriter;
    }
    return lastField = fieldWriter;
  }
}
```

#### toByteArray方法

在`ClassWriter`类当中，`toByteArray()`方法代码如下：

```java
public class ClassWriter extends ClassVisitor {
  public byte[] toByteArray() {

    // First step: compute the size in bytes of the ClassFile structure.
    // The magic field uses 4 bytes, 10 mandatory fields (minor_version, major_version,
    // constant_pool_count, access_flags, this_class, super_class, interfaces_count, fields_count,
    // methods_count and attributes_count) use 2 bytes each, and each interface uses 2 bytes too.
    int size = 24 + 2 * interfaceCount;
    int fieldsCount = 0;
    FieldWriter fieldWriter = firstField;
    while (fieldWriter != null) {
      ++fieldsCount;
      size += fieldWriter.computeFieldInfoSize();    // 这里是对FieldWriter.computeFieldInfoSize()方法的调用
      fieldWriter = (FieldWriter) fieldWriter.fv;
    }
    // ......


    // Second step: allocate a ByteVector of the correct size (in order to avoid any array copy in
    // dynamic resizes) and fill it with the ClassFile content.
    ByteVector result = new ByteVector(size);
    result.putInt(0xCAFEBABE).putInt(version);
    symbolTable.putConstantPool(result);
    int mask = (version & 0xFFFF) < Opcodes.V1_5 ? Opcodes.ACC_SYNTHETIC : 0;
    result.putShort(accessFlags & ~mask).putShort(thisClass).putShort(superClass);
    result.putShort(interfaceCount);
    for (int i = 0; i < interfaceCount; ++i) {
      result.putShort(interfaces[i]);
    }
    result.putShort(fieldsCount);
    fieldWriter = firstField;
    while (fieldWriter != null) {
      fieldWriter.putFieldInfo(result);             // 这里是对FieldWriter.putFieldInfo()方法的调用
      fieldWriter = (FieldWriter) fieldWriter.fv;
    }
    // ......

    // Third step: replace the ASM specific instructions, if any.
    if (hasAsmInstructions) {
      return replaceAsmInstructions(result.data, hasFrames);
    } else {
      return result.data;
    }
  }
}
```

### 2.5.3 小结

本文主要对`FieldWriter`类进行介绍，内容总结如下：

- 第一点，对于`FieldWriter`类的各个不同部分进行介绍，以便从整体上来理解`FieldWriter`类。
- 第二点，关于`FieldWriter`类的使用，它主要出现在`ClassWriter`类当中的`visitField()`和`toByteArray()`方法内。
- 第三点，从ASM应用的角度来说，只需要知道`FieldWriter`类的存在就可以了，不需要深究，我们平常写ASM代码的时候，由于它不带有`public`标识，所以不会直接用到它；从理解ASM源码的角度来说，`FieldWriter`类则值得研究，可以重点关注一下`computeFieldInfoSize()`和`putFieldInfo()`这两个方法。

## 2.6 MethodVisitor介绍

​	通过调用`ClassVisitor`类的`visitMethod()`方法，会返回一个`MethodVisitor`类型的对象。

```java
public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions);
```

### 2.6.1 MethodVisitor类

​	从类的结构来说，`MethodVisitor`类与`ClassVisitor`类和`FieldVisitor`类是非常相似性的。

#### class info

```java
/**
 * A visitor to visit a Java method. The methods of this class must be called in the following
 * order: ( {@code visitParameter} )* [ {@code visitAnnotationDefault} ] ( {@code visitAnnotation} |
 * {@code visitAnnotableParameterCount} | {@code visitParameterAnnotation} {@code
 * visitTypeAnnotation} | {@code visitAttribute} )* [ {@code visitCode} ( {@code visitFrame} |
 * {@code visit<i>X</i>Insn} | {@code visitLabel} | {@code visitInsnAnnotation} | {@code
 * visitTryCatchBlock} | {@code visitTryCatchAnnotation} | {@code visitLocalVariable} | {@code
 * visitLocalVariableAnnotation} | {@code visitLineNumber} )* {@code visitMaxs} ] {@code visitEnd}.
 * In addition, the {@code visit<i>X</i>Insn} and {@code visitLabel} methods must be called in the
 * sequential order of the bytecode instructions of the visited code, {@code visitInsnAnnotation}
 * must be called <i>after</i> the annotated instruction, {@code visitTryCatchBlock} must be called
 * <i>before</i> the labels passed as arguments have been visited, {@code
 * visitTryCatchBlockAnnotation} must be called <i>after</i> the corresponding try catch block has
 * been visited, and the {@code visitLocalVariable}, {@code visitLocalVariableAnnotation} and {@code
 * visitLineNumber} methods must be called <i>after</i> the labels passed as arguments have been
 * visited.
 *
 * @author Eric Bruneton
 */
public abstract class MethodVisitor {
}
```

#### fields

第二个部分，`MethodVisitor`类定义的字段有哪些。

```java
public abstract class MethodVisitor {
  /**
   * The ASM API version implemented by this visitor. The value of this field must be one of the
   * {@code ASM}<i>x</i> values in {@link Opcodes}.
   */
  protected final int api;
  /**
   * The method visitor to which this visitor must delegate method calls. May be {@literal null}.
   */
  protected MethodVisitor mv;
}
```

#### constructors

第三个部分，`MethodVisitor`类定义的构造方法有哪些。

```java
public abstract class MethodVisitor {
    public MethodVisitor(final int api) {
        this(api, null);
    }

    public MethodVisitor(final int api, final MethodVisitor methodVisitor) {
        this.api = api;
        this.mv = methodVisitor;
    }
}
```

#### methods

第四个部分，`MethodVisitor`类定义的方法有哪些。在`MethodVisitor`类当中，定义了许多的`visitXxx()`方法，我们列出了其中的一些方法，内容如下：

```java
public abstract class MethodVisitor {
  // 1. visitCode 方法体的开始位置
  public void visitCode();

  // 2. 方法体内的代码逻辑
  public void visitInsn(final int opcode);
  public void visitIntInsn(final int opcode, final int operand);
  public void visitVarInsn(final int opcode, final int var);
  public void visitTypeInsn(final int opcode, final String type);
  public void visitFieldInsn(final int opcode, final String owner, final String name, final String descriptor);
  public void visitMethodInsn(final int opcode, final String owner, final String name, final String descriptor,
                              final boolean isInterface);
  public void visitInvokeDynamicInsn(final String name, final String descriptor, final Handle bootstrapMethodHandle, final Object... bootstrapMethodArguments);
  public void visitJumpInsn(final int opcode, final Label label);
  public void visitLabel(final Label label);
  public void visitLdcInsn(final Object value);
  public void visitIincInsn(final int var, final int increment);
  public void visitTableSwitchInsn(final int min, final int max, final Label dflt, final Label... labels);
  public void visitLookupSwitchInsn(final Label dflt, final int[] keys, final Label[] labels);
  public void visitMultiANewArrayInsn(final String descriptor, final int numDimensions);
  // try-catch 代码，也可以归结为 方法体内的代码逻辑
  public void visitTryCatchBlock(final Label start, final Label end, final Label handler, final String type);

  // 3. 方法体结束的位置
  public void visitMaxs(final int maxStack, final int maxLocals);
  // 4. 最后调用的visitEnd方法，表示 操作结束
  public void visitEnd();

  // ......
}
```

对于这些`visitXxx()`方法，它们分别有什么作用呢？我们有三方面的资料可能参阅：

- 第一，从ASM API的角度来讲，我们可以查看API文档，来具体了解某一个方法是要实现什么样的作用，该方法所接收的参数代表什么含义。
- 第二，**从ClassFile的角度来讲，这些`visitXxxInsn()`方法的本质就是组装instruction的内容**。我们可以参考[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的[Chapter 6. The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)部分。
- 第三，《[Java ASM系列二：OPCODE](https://lsieun.github.io/java/asm/java-asm-season-02.html)》，主要是对opcode进行介绍。

### 2.6.2 方法的调用顺序

在`MethodVisitor`类当中，定义了许多的`visitXxx()`方法，这些方法的调用，也要遵循一定的顺序。

```java
(visitParameter)*
[visitAnnotationDefault]
(visitAnnotation | visitAnnotableParameterCount | visitParameterAnnotation | visitTypeAnnotation | visitAttribute)*
[
    visitCode
    (
        visitFrame |
        visitXxxInsn |
        visitLabel |
        visitInsnAnnotation |
        visitTryCatchBlock |
        visitTryCatchAnnotation |
        visitLocalVariable |
        visitLocalVariableAnnotation |
        visitLineNumber
    )*
    visitMaxs
]
visitEnd
```

我们可以把这些`visitXxx()`方法分成三组：

- 第一组，在`visitCode()`方法之前的方法。这一组的方法，主要负责parameter、annotation和attributes等内容，这些内容并不是方法当中“必不可少”的一部分；在当前课程当中，我们暂时不去考虑这些内容，可以忽略这一组方法。
- 第二组，**在`visitCode()`方法和`visitMaxs()`方法之间的方法。这一组的方法，主要负责当前方法的“方法体”内的opcode内容**。<u>其中，`visitCode()`方法，标志着方法体的开始，而`visitMaxs()`方法，标志着方法体的结束</u>。
- 第三组，是`visitEnd()`方法。这个`visitEnd()`方法，是最后一个进行调用的方法。

对这些`visitXxx()`方法进行精简之后，内容如下：

```java
[
    visitCode
    (
        visitFrame |
        visitXxxInsn |
        visitLabel |
        visitTryCatchBlock
    )*
    visitMaxs
]
visitEnd
```

这些方法的调用顺序，可以记忆如下：

- 第一步，调用`visitCode()`方法，调用一次。
- 第二步，调用`visitXxxInsn()`方法，可以调用多次。对这些方法的调用，就是在构建方法的“方法体”。
- 第三步，调用`visitMaxs()`方法，调用一次。
- 第四步，调用`visitEnd()`方法，调用一次。

### 2.6.3 小结

本文是对`MethodVisitor`类进行了介绍，内容总结如下：

- 第一点，对于`MethodVisitor`类的各个不同部分进行介绍，以便从整体上来理解`MethodVisitor`类。
- 第二点，在`MethodVisitor`类当中，`visitXxx()`方法也需要遵循一定的调用顺序。

另外，需要注意两点内容：

- 第一点，`ClassVisitor`类有自己的`visitXxx()`方法，`MethodVisitor`类也有自己的`visitXxx()`方法，两者是不一样的，要注意区分。
- 第二点，**`ClassVisitor.visitMethod()`方法提供的是“方法头”（Method Header）所需要的信息，它会返回一个`MethodVisitor`对象，这个`MethodVisitor`对象就用来实现“方法体”里面的代码逻辑**。

## 2.7 MethodWriter介绍

`MethodWriter`类的父类是`MethodVisitor`类。在`ClassWriter`类里，`visitMethod()`方法的实现就是通过`MethodWriter`类来实现的。

### 2.7.1 MethodWriter类

#### class info

第一个部分，`MethodWriter`类的父类是`MethodVisitor`类。<u>需要注意的是，`MethodWriter`类并不带有`public`修饰，因此它的有效访问范围只局限于它所处的package当中，不能像其它的`public`类一样被外部所使用</u>。

```java
/**
 * A {@link MethodVisitor} that generates a corresponding 'method_info' structure, as defined in the
 * Java Virtual Machine Specification (JVMS).
 *
 * @see <a href="https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.6">JVMS
 *     4.6</a>
 * @author Eric Bruneton
 * @author Eugene Kuleshov
 */
final class MethodWriter extends MethodVisitor {
}
```

#### fields

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.6) => 4.6. Methods

第二个部分，`MethodWriter`类定义的字段有哪些。

在`MethodWriter`类当中，定义了很多的字段。下面的几个字段，是与方法的访问标识（access flag）、方法名（method name）和描述符（method descriptor）等直接相关的字段：

```java
final class MethodWriter extends MethodVisitor {
    private final int accessFlags; // 访问修饰符 access_flags
    private final int nameIndex; // 方法名在常量池的索引下标 name_index
    private final String name; // 方法名
    private final int descriptorIndex; // 方法描述信息在常量池中的索引下标 descriptor_index，
    private final String descriptor; // 方法的描述信息, 即 方法参数 和 返回值
    private Attribute firstAttribute; // 方法的 属性()
}
```

这些字段与`ClassFile`当中的`method_info`也是对应的：

```java
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

下面的几个字段，是与“方法体”直接相关的几个字段：

```java
final class MethodWriter extends MethodVisitor {
    private int maxStack; // max_stack
    private int maxLocals; // max_locals
    private final ByteVector code = new ByteVector(); // code_length code[code_length]
    private Handler firstHandler; // exception_table的start_pc
    private Handler lastHandler; // exception_table的end_pc
    private final int numberOfExceptions;  // The number_of_exceptions field of the Exceptions attribute.
    private final int[] exceptionIndexTable; // The exception_index_table array of the Exceptions attribute, or null.
    private Attribute firstCodeAttribute; // attributes_count attributes[attributes_count]
}
```

这些字段对应于`Code`属性结构：

```java
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### constructors

第三个部分，`MethodWriter`类定义的构造方法有哪些。

```java
final class MethodWriter extends MethodVisitor {
  MethodWriter(SymbolTable symbolTable, int access, String name, String descriptor, String signature, String[] exceptions, int compute) {
    super(Opcodes.ASM9);
    this.symbolTable = symbolTable;
    this.accessFlags = "<init>".equals(name) ? access | Constants.ACC_CONSTRUCTOR : access;
    this.nameIndex = symbolTable.addConstantUtf8(name);
    this.name = name;
    this.descriptorIndex = symbolTable.addConstantUtf8(descriptor);
    this.descriptor = descriptor;
    this.signatureIndex = signature == null ? 0 : symbolTable.addConstantUtf8(signature);
    if (exceptions != null && exceptions.length > 0) {
      numberOfExceptions = exceptions.length;
      this.exceptionIndexTable = new int[numberOfExceptions];
      for (int i = 0; i < numberOfExceptions; ++i) {
        this.exceptionIndexTable[i] = symbolTable.addConstantClass(exceptions[i]).index;
      }
    } else {
      numberOfExceptions = 0;
      this.exceptionIndexTable = null;
    }
    this.compute = compute;
    if (compute != COMPUTE_NOTHING) {
      // Update maxLocals and currentLocals.
      int argumentsSize = Type.getArgumentsAndReturnSizes(descriptor) >> 2;
      if ((access & Opcodes.ACC_STATIC) != 0) {
        --argumentsSize;
      }
      maxLocals = argumentsSize;
      currentLocals = argumentsSize;
      // Create and visit the label for the first basic block.
      firstBasicBlock = new Label();
      visitLabel(firstBasicBlock);
    }
  }
}
```

#### methods

第四个部分，`MethodWriter`类定义的方法有哪些。

在`MethodWriter`类当中，也有两个重要的方法：`computeMethodInfoSize()`和`putMethodInfo()`方法。这两个方法也是在`ClassWriter`类的`toByteArray()`方法内使用到。

```java
final class MethodWriter extends MethodVisitor {
  int computeMethodInfoSize() {
    // ......
    // 2 bytes each for access_flags, name_index, descriptor_index and attributes_count.
    int size = 8;
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    if (code.length > 0) {
      if (code.length > 65535) {
        throw new MethodTooLargeException(symbolTable.getClassName(), name, descriptor, code.length);
      }
      symbolTable.addConstantUtf8(Constants.CODE);
      // The Code attribute has 6 header bytes, plus 2, 2, 4 and 2 bytes respectively for max_stack,
      // max_locals, code_length and attributes_count, plus the ByteCode and the exception table.
      size += 16 + code.length + Handler.getExceptionTableSize(firstHandler);
      if (stackMapTableEntries != null) {
        boolean useStackMapTable = symbolTable.getMajorVersion() >= Opcodes.V1_6;
        symbolTable.addConstantUtf8(useStackMapTable ? Constants.STACK_MAP_TABLE : "StackMap");
        // 6 header bytes and 2 bytes for number_of_entries.
        size += 8 + stackMapTableEntries.length;
      }
      // ......
    }
    if (numberOfExceptions > 0) {
      symbolTable.addConstantUtf8(Constants.EXCEPTIONS);
      size += 8 + 2 * numberOfExceptions;
    }
    //......
    return size;
  }

  void putMethodInfo(final ByteVector output) {
    boolean useSyntheticAttribute = symbolTable.getMajorVersion() < Opcodes.V1_5;
    int mask = useSyntheticAttribute ? Opcodes.ACC_SYNTHETIC : 0;
    output.putShort(accessFlags & ~mask).putShort(nameIndex).putShort(descriptorIndex);
    // ......
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    int attributeCount = 0;
    if (code.length > 0) {
      ++attributeCount;
    }
    if (numberOfExceptions > 0) {
      ++attributeCount;
    }
    // ......
    // For ease of reference, we use here the same attribute order as in Section 4.7 of the JVMS.
    output.putShort(attributeCount);
    if (code.length > 0) {
      // 2, 2, 4 and 2 bytes respectively for max_stack, max_locals, code_length and
      // attributes_count, plus the ByteCode and the exception table.
      int size = 10 + code.length + Handler.getExceptionTableSize(firstHandler);
      int codeAttributeCount = 0;
      if (stackMapTableEntries != null) {
        // 6 header bytes and 2 bytes for number_of_entries.
        size += 8 + stackMapTableEntries.length;
        ++codeAttributeCount;
      }
      // ......
      output
        .putShort(symbolTable.addConstantUtf8(Constants.CODE))
        .putInt(size)
        .putShort(maxStack)
        .putShort(maxLocals)
        .putInt(code.length)
        .putByteArray(code.data, 0, code.length);
      Handler.putExceptionTable(firstHandler, output);
      output.putShort(codeAttributeCount);
      // ......
    }
    if (numberOfExceptions > 0) {
      output
        .putShort(symbolTable.addConstantUtf8(Constants.EXCEPTIONS))
        .putInt(2 + 2 * numberOfExceptions)
        .putShort(numberOfExceptions);
      for (int exceptionIndex : exceptionIndexTable) {
        output.putShort(exceptionIndex);
      }
    }
    // ......
  }
}
```

### 2.7.2 MethodWriter类的使用

关于`MethodWriter`类的使用，它主要出现在`ClassWriter`类当中的`visitMethod()`和`toByteArray()`方法内。

#### visitMethod方法

在`ClassWriter`类当中，`visitMethod()`方法代码如下：

```java
public class ClassWriter extends ClassVisitor {
  public final MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodWriter methodWriter = new MethodWriter(symbolTable, access, name, descriptor, signature, exceptions, compute);
    if (firstMethod == null) {
      firstMethod = methodWriter;
    } else {
      lastMethod.mv = methodWriter;
    }
    return lastMethod = methodWriter;
  }
}
```

#### toByteArray方法

```java
public class ClassWriter extends ClassVisitor {
  public byte[] toByteArray() {

    // First step: compute the size in bytes of the ClassFile structure.
    // The magic field uses 4 bytes, 10 mandatory fields (minor_version, major_version,
    // constant_pool_count, access_flags, this_class, super_class, interfaces_count, fields_count,
    // methods_count and attributes_count) use 2 bytes each, and each interface uses 2 bytes too.
    int size = 24 + 2 * interfaceCount;
    // ......
    int methodsCount = 0;
    MethodWriter methodWriter = firstMethod;
    while (methodWriter != null) {
      ++methodsCount;
      size += methodWriter.computeMethodInfoSize();        // 这里是对MethodWriter.computeMethodInfoSize()方法的调用
      methodWriter = (MethodWriter) methodWriter.mv;
    }
    // ......

    // Second step: allocate a ByteVector of the correct size (in order to avoid any array copy in
    // dynamic resizes) and fill it with the ClassFile content.
    ByteVector result = new ByteVector(size);
    result.putInt(0xCAFEBABE).putInt(version);
    symbolTable.putConstantPool(result);
    int mask = (version & 0xFFFF) < Opcodes.V1_5 ? Opcodes.ACC_SYNTHETIC : 0;
    result.putShort(accessFlags & ~mask).putShort(thisClass).putShort(superClass);
    result.putShort(interfaceCount);
    for (int i = 0; i < interfaceCount; ++i) {
      result.putShort(interfaces[i]);
    }
    // ......
    result.putShort(methodsCount);
    boolean hasFrames = false;
    boolean hasAsmInstructions = false;
    methodWriter = firstMethod;
    while (methodWriter != null) {
      hasFrames |= methodWriter.hasFrames();
      hasAsmInstructions |= methodWriter.hasAsmInstructions();
      methodWriter.putMethodInfo(result);                    // 这里是对MethodWriter.putMethodInfo()方法的调用
      methodWriter = (MethodWriter) methodWriter.mv;
    }
    // ......

    // Third step: replace the ASM specific instructions, if any.
    if (hasAsmInstructions) {
      return replaceAsmInstructions(result.data, hasFrames);
    } else {
      return result.data;
    }
  }
}

```

### 2.7.3 小结

本文主要对`MethodWriter`类进行介绍，内容总结如下：

- 第一点，对于`MethodWriter`类的各个不同部分进行介绍，以便从整体上来理解`MethodWriter`类。
- 第二点，关于`MethodWriter`类的使用，它主要出现在`ClassWriter`类当中的`visitMethod()`和`toByteArray()`方法内。
- 第三点，从应用ASM的角度来说，只需要知道`MethodWriter`类的存在就可以了，不需要深究；从理解ASM源码的角度来说，`MethodWriter`类也是值得研究的。

## 2.8 方法的初始Frame

### 2.8.1 Frame内存结构

**JVM Architecture由Class Loader SubSystem(类加载子系统)、Runtime Data Areas(运行时数据区域)和Execution Engine(执行引擎)三个主要部分组成**，如下图所示。

其中，Runtime Data Areas包括Method Area(方法区)、Heap Area(堆)、**Stack Area**(栈)、PC Registers(程序计数器)和Native Method Stack(本地方法栈)等部分。

![Java ASM系列：（013）方法的初始Frame_ASM](https://s2.51cto.com/images/20210623/1624446191162652.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

<u>在程序运行的过程中，每一个线程（Thread）都对应一个属于自己的**JVM Stack**。当一个新线程（Thread）开始的时候，就会在内存上分配一个属于自己的JVM Stack；当该线程（Thread）执行结束的时候，相应的JVM Stack内存空间也就被回收了</u>。

在JVM Stack当中，是栈的结构，里面存储的是frames；每一个frame空间可以称之为**Stack Frame**。<u>当调用一个新方法的时候，就会在JVM Stack上分配一个frame空间（入栈操作）；当方法退出时，相应的frame空间也会JVM Stack上进行清除掉（出栈操作）</u>。

在Stack Frame内存空间当中，有两个重要的结构，即**local variables**和**operand stack**。

<u>上图中Stack Frame下面的LVA指本地变量表，OS指操作数栈，FO指Frame Data，其存储和方法相关的一些数据，非重点所以下图省略。</u>

![Java ASM系列：（013）方法的初始Frame_ByteCode_02](https://s2.51cto.com/images/20210623/1624446269398923.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在Stack Frame当中，**operand stack是一个栈的结构**，遵循“后进先出”（LIFO）的规则，**local variables则是一个数组**，索引从`0`开始。

<u>对于每一个方法来说，它都是在自己的Stack Frame上来运行的</u>：

- **在编译的时候（compile time），local variables和operand stack的空间大小(对应ClassFile中`method_info`的`Code`属性中的`max_locals`和`max_stack`)就确定下来了**。比如说，一个`.java`文件经过编译之后，得到`.class`文件，对于其中的某一个方法来说，它的local variable占用10个slot空间，operand stack占用4个slot空间。
- 在运行的时候（run-time），在local variables和operand stack上存放的数据，会随着方法的执行，不断发生变化。

那么，在运行的时候（run-time），刚进入方法，但还没有执行任何指令（instruction），那么，此时、此刻，local variables和operand stack是一个什么样的状态呢？

> 在Stack Frame空间当中，local variables和operand stack会有**一个开始的状态**和**一个结束的状态**。

### 2.8.2 方法的初始Frame

在方法刚开始的时候，operand stack是空的，不需要存储任何的数据，而local variables的初始状态，则需要考虑两个因素：

1. 是否需要存储`this`? 通过判断当前方法是否为static方法。

   - 如果当前方法是static方法，则不需要存储`this`。

   - **如果当前方法是non-static方法，则需要在local variables索引为`0`的位置存在一个`this`变量**。

2. 当前方法是否接收参数。方法接收的参数，会<u>按照参数的声明顺序</u>放到local variables当中。

   - 如果方法参数不是`long`和`double`类型，那么它在local variable当中占用1个位置。

   - **如果方法的参数是`long`或`double`类型，那么它在local variable当中占用2个位置**。

#### static方法

假设`HelloWorld`当中有一个静态`add(int, int)`方法，如下所示：

```java
public class HelloWorld {
  public static int add(int a, int b) {
    return a + b;
  }
}
```

我们可以通过运行`HelloWorldFrameCore`类，来查看`add(int, int)`方法的初始Frame：

```java
[int, int] []
```

在上面的结果中，第一个`[]`中存放的是local variables的数据，在第二个`[]`中存放的是operand stack的数据。

该方法包含的Instruction内容如下（使用`javap -c sample.HelloWorld`命令查看）：

```java
 public static int add(int, int);
    Code:
       0: iload_0
       1: iload_1
       2: iadd
       3: ireturn
```

该方法整体的Frame变化如下（运行learn-java-asm项目的`HelloWorldFrameCore`类的main方法）：

```java
add(II)I
[int, int] [] // 方法按照顺序传入参数a 和 b
[int, int] [int] // iload_0, 将 int变量a 加载到 operand stack 中
[int, int] [int, int] // iload_1, 将 int变量b 加载到 oerpand stack 中
[int, int] [int] // iadd, 将 b 和 a 出栈，然后相加后的到sum重新放入 operand stack 中
[] [] // ireturn 表示返回 int值，执行return操作时，会设置local variables 和 operand stack 为null
```

`HelloWorldFrameCore`类中new了`MethodStackMapFrameVisitor`对象，这里查看`MethodStackMapFrameVisitor`类里面的`MethodStackMapFrameAdapter`，查看其继承的`AnalyzerAdapter`的`visitInsn()`方法实现：

```java
// org.objectweb.asm.commons.AnalyzerAdapter#visitInsn
public void visitInsn(final int opcode) {
  super.visitInsn(opcode);
  execute(opcode, 0, null);
  // 可以看出来只要执行 return 操作，就会把locals和stack设置为null
  if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN) || opcode == Opcodes.ATHROW) {
    this.locals = null;
    this.stack = null;
  }
}
// Opcode 里代码节选
/**
int JSR = 168; // -
  int RET = 169; // visitVarInsn
  int TABLESWITCH = 170; // visiTableSwitchInsn
  int LOOKUPSWITCH = 171; // visitLookupSwitch
  int IRETURN = 172; // visitInsn
  int LRETURN = 173; // -
  int FRETURN = 174; // -
  int DRETURN = 175; // -
  int ARETURN = 176; // -
  int RETURN = 177; // -
  int GETSTATIC = 178; // visitFieldInsn
  int PUTSTATIC = 179; // -
  int GETFIELD = 180; // -
  int PUTFIELD = 181; // -
*/
```

#### non-static方法

假设`HelloWorld`当中有一个非静态`add(int, int)`方法，如下所示：

```java
public class HelloWorld {
    public int add(int a, int b) {
        return a + b;
    }
}
```

我们可以通过运行`HelloWorldFrameCore`类，来查看`add(int, int)`方法的初始Frame：

```java
[sample/HelloWorld, int, int] []
```

该方法包含的Instruction内容如下：

```java
public int add(int, int);
  Code:
     0: iload_1
     1: iload_2
     2: iadd
     3: ireturn
```

该方法整体的Frame变化如下：

```java
add(II)I
[sample/HelloWorld, int, int] [] // non-static方法，local variables数组的下标0位置存储 this 指针, 后续分别为a和b
[sample/HelloWorld, int, int] [int] // iload_1, 将 local variables 下标1的int数据a加载到 operand stack
[sample/HelloWorld, int, int] [int, int] // iload_2, 将 local variables 下标2的int数据b加载到 operand stack
[sample/HelloWorld, int, int] [int] // iadd, 将operand stack栈顶的b和a弹出后相加的到sum，然后放回 operand stack顶部
[] [] // ireturn
```

#### long和double类型

假设`HelloWorld`当中有一个非静态`add(long, long)`方法，如下所示：

```java
public class HelloWorld {
  public long add(long a, long b) {
    return a + b;
  }
}
```

我们可以通过运行`HelloWorldFrameCore`类，来查看`add(long, long)`方法的初始Frame：

```java
[sample/HelloWorld, long, top, long, top] []
```

该方法包含的Instruction内容如下：

```java
public long add(long, long);
  Code:
     0: lload_1
     1: lload_3
     2: ladd
     3: lreturn
```

该方法整体的Frame变化如下：

> 回顾"2.3.2 示例二：生成接口+字段+方法 > 小结 > 描述符（desciptor） 可以知道J表示long"

```java
add(JJ)J
[sample/HelloWorld, long, top, long, top] [] // non-static方法，local variables下标0位置存储this指针
[sample/HelloWorld, long, top, long, top] [long, top] // lload_1, top用于说明long占用2个slot(1个slot 32bit)
[sample/HelloWorld, long, top, long, top] [long, top, long, top] // lload_3, 因为 第二个long参数 b 占用2个slot，起始位置就是 local variables 下标3的位置
[sample/HelloWorld, long, top, long, top] [long, top] // ladd, b和a出栈相加后相加，sum放回 operand stack
[] [] // lreturn
```

### 2.8.3 小结

本文对方法初始的Frame进行了介绍，内容总结如下：

- **第一点，在JVM当中，每一个方法的调用都会分配一个Stack Frame内存空间；在Stack Frame内存空间当中，有local variables和operand stack两个重要结构；在Java文件进行编译的时候，方法对应的local variables和operand stack的大小就决定了**。
- **第二点，如何计算方法的初始Frame。在方法刚开始的时候，Stack Frame中的operand stack是空的，而只需要计算local variables的初始状态；而计算local variables的初始状态，则需要考虑当前方法是否为static方法、是否接收方法参数、方法参数中是否有`long`和`double`类型。**

## 2.9 MethodVisitor代码示例

![Java ASM系列：（014）MethodVisitor代码示例_ClassFile](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

> 在当前阶段，我们只能进行Class Generation的操作。

### 2.9.1 示例一：`<init>()`方法

在`.class`文件中，构造方法的名字是`<init>`，它表示instance **init**ialization method的缩写。

#### 预期目标

```java
public class HelloWorld {
}
```

或者（上面代码效果等同于下面，JVM会自动在构造方法开头添加对父类构造方法的调用逻辑）

```java
public class HelloWorld {
  public HelloWorld() {
    super();
  }
}
```

#### 编码实现

注意点：

+ 平时java代码在构造方法内会自动帮我们调用父类构造方法，ASM代码则需要我们显式调用父类构造方法

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv.visitCode();
      mv.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv.visitInsn(RETURN);
      // 方法体结束
      mv.visitMaxs(1, 1);
      mv.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);
  }
}
```

#### Frame的变化

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aload)
>
> aload: Load `reference` from local variable

对于`HelloWorld`类中`<init>()`方法对应的Instruction内容如下：

```shell
$ javap -c sample.HelloWorld
public sample.HelloWorld();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return
```

该方法对应的Frame变化情况如下：

```java
<init>()V
[uninitialized_this] [] // non-static方法，第一个参数是this，但是这里this未初始化，只是分配了内存空间，所以是 uninitialized_this
[uninitialized_this] [uninitialized_this]  // aload_0, 把 未初始化的实例this指针加载到 operand stack
[sample/HelloWorld] [] // invokespecial 初始化(内部还调用父类构造方法), this指针从 栈顶弹出
[] [] // return
```

**在这里，我们看到一个很“不一样”的变量，就是`uninitialized_this`，它就是一个“引用”，它指向的内存空间还没有初始化；等经过初始化之后，`uninitialized_this`变量就变成`this`变量**。

#### 小结

通过上面的示例，我们注意四个知识点：

- 第一点，如何使用`ClassWriter`类。
  - 第一步，创建`ClassWriter`类的实例。
  - 第二步，调用`ClassWriter`类的`visitXxx()`方法。
  - 第三步，调用`ClassWriter`类的`toByteArray()`方法。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
  - 第一步，调用`visitCode()`方法，调用一次
  - 第二步，调用`visitXxxInsn()`方法，可以调用多次
  - 第三步，调用`visitMaxs()`方法，调用一次
  - 第四步，调用`visitEnd()`方法，调用一次
- **第三点，在`.class`文件中，构造方法的名字是`<init>`。从Instruction的角度来讲，调用构造方法会用到`invokespecial`指令**。
- **第四点，从Frame的角度来讲，在构造方法`<init>()`中，local variables当中索引为`0`的位置存储的是什么呢？如果还没有进行初始化操作，就是`uninitialized_this`变量；如果已经进行了初始化操作，就是`this`变量。**

### 2.9.2 示例二：`<clinit>`方法

在`.class`文件中，静态初始化方法的名字是`<clinit>`，它表示**cl**ass **init**ialization method的缩写。

#### 预期目标

```java
public class HelloWorld {
  static {
    System.out.println("class initialization method");
  }
}
```

#### 编码实现

注意点：

+ 静态代码块，访问修饰符`ACC_STATIC`就够了（添加`ACC_PUBLIC`也能正常运作，只是javap会看到flags会多一个`ACC_PUBLIC`）
+ 调用实例的方法，使用`INVOKESPECIAL`

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // 构造方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv.visitCode();
      mv.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv.visitInsn(RETURN);
      // 方法体结束
      mv.visitMaxs(1, 1);
      mv.visitEnd();
    }

    // static 静态初始化方法
    {
      // 注意这里访问修饰符 没有 ACC_PUBLIC, 只需要 ACC_STATIC
      MethodVisitor mv = cw.visitMethod(ACC_STATIC, "<clinit>", "()V", null, null);
      mv.visitCode();
      // 访问 System 类的 static 字段 out
      mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      // 常量池字符串
      mv.visitLdcInsn("class initialization method");
      // 调用 System.out 的println方法
      mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv.visitInsn(RETURN);
      mv.visitMaxs(2, 0);
      mv.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);
  }
}
```

#### Frame的变化

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aload)
>
> ldc: Push item from run-time constant pool

对于`HelloWorld`类中`<clinit>()`方法对应的Instruction内容如下：

```shell
$ javap -c sample.HelloWorld
static {};
  Code:
     0: getstatic     #18                 // Field java/lang/System.out:Ljava/io/PrintStream;
     3: ldc           #20                 // String class initialization method
     5: invokevirtual #26                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     8: return
```

该方法对应的Frame变化情况如下：

```java
<clinit>()V
[] [] // static 方法，所以 local variabls 不需要在下标0位置填充this指针
[] [java/io/PrintStream] // getstatic , 将System的static成员变量out入 operand stack
[] [java/io/PrintStream, java/lang/String] // ldc 加载运行时常量池中的字符串到 operand stack
[] [] // invokevirtual 调用 System.out的println方法
[] [] // return
```

#### 小结

通过上面的示例，我们注意三个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- **第三点，在`.class`文件中，静态初始化方法的名字是`<clinit>`，它的方法描述符是`()V`。**

### 2.9.3 示例三：创建对象

#### 预期目标

假如有一个`GoodChild`类，内容如下：

```java
public class GoodChild {
  public String name;
  public int age;

  public GoodChild(String name, int age) {
    this.name = name;
    this.age = age;
  }
}
```

我们的预期目标是生成一个`HelloWorld`类：

```java
public class HelloWorld {
  public void test() {
    GoodChild child = new GoodChild("Lucy", 8);
  }
}
```

#### 编码实现

注意点：

+ 这里`DUP`因为后续调用构造函数会将栈中的GoodChild指针以及参数弹出，而构造方法的返回值是void，所以DUP一份GoodChild指针，后续才能将其从operand stack存储回local variables

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // 构造方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv.visitCode();
      mv.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv.visitInsn(RETURN);
      // 方法体结束
      mv.visitMaxs(1, 1);
      mv.visitEnd();
    }


    {
      MethodVisitor mv  = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv.visitCode();
      // new GoodChild
      mv.visitTypeInsn(NEW, "sample/GoodChild");
      // 栈顶复制一份，即 现在有两个未初始化的 GoodChild 指针
      mv.visitInsn(DUP);
      // 构造方法参数
      mv.visitLdcInsn("Lucy");
      mv.visitIntInsn(BIPUSH, 8);
      // 调用 GoodChild 构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "sample/GoodChild", "<init>", "(Ljava/lang/String;I)V", false);
      // 将初始化后的 GoodChild 实例指针存到 local variables
      mv.visitVarInsn(ASTORE, 1);
      mv.visitInsn(RETURN);
      mv.visitMaxs(4, 2);
      mv.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

#### 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Object obj = clazz.newInstance();

    Method m = clazz.getDeclaredMethod("test");
    m.invoke(obj);
  }
}
```

#### Frame的变化

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.dup)
>
> dup: Duplicate the top operand stack value
>
> ldc: Push item from run-time constant pool
>
> bipush: Push byte

对于`HelloWorld`类中`test()`方法对应的Instruction内容如下：

```shell
	$ javap -c sample.HelloWorld
public void test();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=4, locals=2, args_size=1
         0: new           #11                 // class sample/GoodChild
         3: dup
         4: ldc           #13                 // String Lucy
         6: bipush        8
         8: invokespecial #16                 // Method sample/GoodChild."<init>":(Ljava/lang/String;I)V
        11: astore_1
        12: return
```

该方法对应的Frame变化情况如下：

```java
test()V
[sample/HelloWorld] [] // non-static 方法，local variables的下标0位置存this指针
[sample/HelloWorld] [uninitialized_sample/GoodChild] // new, operand statck放入 GoodChild未初始化对象(只分配了内存空间)的指针
[sample/HelloWorld] [uninitialized_sample/GoodChild, uninitialized_sample/GoodChild] // dup, 复制栈顶元素
[sample/HelloWorld] [uninitialized_sample/GoodChild, uninitialized_sample/GoodChild, java/lang/String] // ldc, 常量池中字符串放入 operand stack
[sample/HelloWorld] [uninitialized_sample/GoodChild, uninitialized_sample/GoodChild, java/lang/String, int] // bipush, 把8放入栈
[sample/HelloWorld] [sample/GoodChild] // invokespecial, 将构造方法需要的 对象指针以及参数出栈, 调用 GoodChild 构造方法(内部进行初始化)
[sample/HelloWorld, sample/GoodChild] [] // astore_1, 将初始化后的GoodChild指针存入 local variables
[] []
```

#### 小结

通过上面的示例，我们注意四个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- **第三点，从Instruction的角度来讲，创建对象的指令集合：**
  - **`new`**
  - **`dup`**
  - **`invokespecial`**
- **第四点，从Frame的角度来讲，在创建新对象的时候，执行`new`指令之后，它是uninitialized状态，执行`invokespecial`指令之后，它是一个“合格”的对象。**

### 2.9.4 示例四：调用方法

#### 预期目标

```java
public class HelloWorld {
  public void test(int a, int b) {
    int val = Math.max(a, b); // 对static方法进行调用
    System.out.println(val);  // 对non-static方法进行调用
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // 构造方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv.visitCode();
      mv.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv.visitInsn(RETURN);
      // 方法体结束
      mv.visitMaxs(1, 1);
      mv.visitEnd();
    }


    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "test", "(II)V", null, null);
      mv.visitCode();
      // 加载 参数到 operand stack栈中
      mv.visitVarInsn(ILOAD, 1);
      mv.visitVarInsn(ILOAD, 2);
      // 调用 static 方法
      mv.visitMethodInsn(INVOKESTATIC, "java/lang/Math", "max", "(II)I", false);
      // 将 Math.max()返回值 存入 local variables
      mv.visitVarInsn(ISTORE, 3);
      // 后续就是 System 打印结果
      mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv.visitVarInsn(ILOAD, 3);
      mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
      mv.visitInsn(RETURN);
      mv.visitMaxs(2, 4);
      mv.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

#### 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Object obj = clazz.newInstance();

    Method m = clazz.getDeclaredMethod("test", int.class, int.class);
    m.invoke(obj, 10, 20);
  }
}
```

#### Frame的变化

对于`HelloWorld`类中`test()`方法对应的Instruction内容如下：

```shell
$ javap -c sample.HelloWorld
public void test(int, int);
    descriptor: (II)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=3
         0: iload_1
         1: iload_2
         2: invokestatic  #16                 // Method java/lang/Math.max:(II)I
         5: istore_3
         6: getstatic     #22                 // Field java/lang/System.out:Ljava/io/PrintStream;
         9: iload_3
        10: invokevirtual #28                 // Method java/io/PrintStream.println:(I)V
        13: return

```

该方法对应的Frame变化情况如下：

```java
test(II)V
[sample/HelloWorld, int, int] [] // non-static 方法,local variabls 0下标为this指针，1是a, 2是b
[sample/HelloWorld, int, int] [int] // iload_1, 加载b到 operand stack
[sample/HelloWorld, int, int] [int, int] // iload_2, 加载b到 operand stack
[sample/HelloWorld, int, int] [int] // invokestatic 调用 static 方法 Math.max,弹出 栈顶b和a，计算结果后返回值入栈
[sample/HelloWorld, int, int, int] [] // istore_3, 将 返回值存入 local variables的下标3的位置
[sample/HelloWorld, int, int, int] [java/io/PrintStream] // getstatic, 将 System.out 加载到 栈中
[sample/HelloWorld, int, int, int] [java/io/PrintStream, int] // iload_3 加载local variables 下标3的数据到operand stack
[sample/HelloWorld, int, int, int] [] // invokevirtual 调用 println方法(返回值void, 所以 operand stack空的)
[] [] // return
```

#### 小结

通过上面的示例，我们注意四个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- **第三点，从Instruction的角度来讲，调用static方法是使用`invokestatic`指令，调用non-static方法一般使用`invokevirtual`指令**。
- **第四点，从Frame的角度来讲，实现方法的调用，需要先将`this`变量和方法接收的参数放到operand stack上**。

### 2.9.5 示例五：不调用visitMaxs()方法

在创建`ClassWriter`对象时，使用了`ClassWriter.COMPUTE_FRAMES`选项。

```java
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
```

**使用`ClassWriter.COMPUTE_FRAMES`后，ASM会自动计算max stacks、max locals和stack map frames的具体值。 从代码的角度来说，使用`ClassWriter.COMPUTE_FRAMES`，会忽略我们在代码中`visitMaxs()`方法和`visitFrame()`方法传入的具体参数值。 换句话说，无论我们传入的参数值是否正确，ASM会帮助我们从新计算一个正确的值，代替我们在代码中传入的参数**。

- 第1种情况，在创建`ClassWriter`对象时，`flags`参数使用`ClassWriter.COMPUTE_FRAMES`值，在调用`mv.visitMaxs(0, 0)`方法之后，仍然能得到一个正确的`.class`文件。
- 第2种情况，在创建`ClassWriter`对象时，`flags`参数使用`0`值，在调用`mv.visitMaxs(0, 0)`方法之后，得到的`.class`文件就不能正确运行。

**需要注意的是，在创建`ClassWriter`对象时，`flags`参数使用`ClassWriter.COMPUTE_FRAMES`值，我们可以给`visitMaxs()`方法传入一个错误的值，但是不能省略对于`visitMaxs()`方法的调用**。 如果我们省略掉`visitCode()`和`visitEnd()`方法，生成的`.class`文件也不会出错；当然，并不建议这么做。**但是，如果我们省略掉对于`visitMaxs()`方法的调用，生成的`.class`文件就会出错。**

如果省略掉对于`visitMaxs()`方法的调用，会出现如下错误：

```shell
Exception in thread "main" java.lang.VerifyError: Operand stack overflow
```

### 2.9.6 示例六：不同的MethodVisitor交叉使用

假如我们有两个`MethodVisitor`对象`mv1`和`mv2`，如下所示：

```java
MethodVisitor mv1 = cw.visitMethod(...);
MethodVisitor mv2 = cw.visitMethod(...);
```

同时，我们也知道`MethodVisitor`类里的`visitXxx()`方法需要遵循一定的调用顺序：

- 第一步，调用`visitCode()`方法，调用一次
- 第二步，调用`visitXxxInsn()`方法，可以调用多次
- 第三步，调用`visitMaxs()`方法，调用一次
- 第四步，调用`visitEnd()`方法，调用一次

<u>对于`mv1`和`mv2`这两个对象来说，它们的`visitXxx()`方法的调用顺序是彼此独立的、不会相互干扰</u>。

一般情况下，我们可以如下写代码，这样逻辑比较清晰：

```java
MethodVisitor mv1 = cw.visitMethod(...);
mv1.visitCode(...);
mv1.visitXxxInsn(...)
mv1.visitMaxs(...);
mv1.visitEnd();

MethodVisitor mv2 = cw.visitMethod(...);
mv2.visitCode(...);
mv2.visitXxxInsn(...)
mv2.visitMaxs(...);
mv2.visitEnd();
```

但是，我们也可以这样来写代码：

```java
MethodVisitor mv1 = cw.visitMethod(...);
MethodVisitor mv2 = cw.visitMethod(...);

mv1.visitCode(...);
mv2.visitCode(...);

mv2.visitXxxInsn(...)
mv1.visitXxxInsn(...)

mv1.visitMaxs(...);
mv1.visitEnd();
mv2.visitMaxs(...);
mv2.visitEnd();
```

在上面的代码中，`mv1`和`mv2`这两个对象的`visitXxx()`方法交叉调用，这是可以的。 换句话说，只要每一个`MethodVisitor`对象在调用`visitXxx()`方法时，遵循了调用顺序，那结果就是正确的； 不同的`MethodVisitor`对象，是相互独立的、不会彼此影响。

那么，可能有的同学会问：`MethodVisitor`对象交叉使用有什么作用呢？有没有什么场景下的应用呢？回答是“有的”。 在ASM当中，有一个`org.objectweb.asm.commons.StaticInitMerger`类，其中有一个`MethodVisitor mergedClinitVisitor`字段，它就是一个很好的示例，在后续内容中，我们会介绍到这个类。

#### 预期目标

```java
import java.util.Date;

public class HelloWorld {
  public void test() {
    System.out.println("This is a test method.");
  }

  public void printDate() {
    Date now = new Date();
    System.out.println(now);
  }
}
```

#### 编码实现（第一种方式，顺序）

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // 构造方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv.visitCode();
      mv.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv.visitInsn(RETURN);
      // 方法体结束
      mv.visitMaxs(1, 1);
      mv.visitEnd();
    }

		// test()方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv.visitCode();
      mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv.visitLdcInsn("This is a test method.");
      mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv.visitInsn(RETURN);
      mv.visitMaxs(2, 1);
      mv.visitEnd();
    }

    // printDate()方法
    {
      MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "printDate", "()V", null, null);
      mv.visitCode();
      mv.visitTypeInsn(NEW, "java/util/Date");
      mv.visitInsn(DUP);
      mv.visitMethodInsn(INVOKESPECIAL, "java/util/Date", "<init>", "()V", false);
      mv.visitVarInsn(ASTORE, 1);
      mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv.visitVarInsn(ALOAD, 1);
      mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
      mv.visitInsn(RETURN);
      mv.visitMaxs(2, 2);
      mv.visitEnd();
    }


    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}

```

#### 编码实现（第二种方式，交叉）

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import utils.FileUtils;

import static org.objectweb.asm.Opcodes.*;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/11 5:56 PM
 */
public class HelloWorldGenerateCore {
  public static void main(String[] args) {
    String relativePath = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relativePath);

    // (1) 生成 byte[] 内容 (符合ClassFile的.class二进制数据)
    byte[] bytes = dump();

    // (2) 保存 byte[] 到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() {
    // (1) 创建 ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用 visitXxx() 方法
    cw.visit(V17, ACC_PUBLIC | ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    // 构造方法
    {
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      // 方法体内代码
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      // 调用父类构造方法
      mv1.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      // 方法体结束
      mv1.visitMaxs(1, 1);
      mv1.visitEnd();
    }


    {
      // 第1部分，mv2
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);

      // 第2部分，mv3
      MethodVisitor mv3 = cw.visitMethod(ACC_PUBLIC, "printDate", "()V", null, null);
      mv3.visitCode();
      mv3.visitTypeInsn(NEW, "java/util/Date");
      mv3.visitInsn(DUP);
      mv3.visitMethodInsn(INVOKESPECIAL, "java/util/Date", "<init>", "()V", false);

      // 第3部分，mv2
      mv2.visitCode();
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("This is a test method.");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

      // 第4部分，mv3
      mv3.visitVarInsn(ASTORE, 1);
      mv3.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv3.visitVarInsn(ALOAD, 1);
      mv3.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);

      // 第5部分，mv2
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 1);
      mv2.visitEnd();

      // 第6部分，mv3
      mv3.visitInsn(RETURN);
      mv3.visitMaxs(2, 2);
      mv3.visitEnd();
    }


    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Object instance = clazz.newInstance();
    invokeMethod(clazz, "test", instance);
    invokeMethod(clazz, "printDate", instance);
  }

  public static void invokeMethod(Class<?> clazz, String methodName, Object instance) throws Exception {
    Method m = clazz.getDeclaredMethod(methodName);
    m.invoke(instance);
  }
}
```

### 2.9.7 总结

本文主要介绍了`MethodVisitor`类的示例，内容总结如下：

- 第一点，要注意`MethodVisitor`类里`visitXxx()`的调用顺序
  - 第一步，调用`visitCode()`方法，调用一次
  - 第二步，调用`visitXxxInsn()`方法，可以调用多次
  - 第三步，调用`visitMaxs()`方法，调用一次
  - 第四步，调用`visitEnd()`方法，调用一次
- **第二点，在`.class`文件当中，构造方法的名字是`<init>`，静态初始化方法的名字是`<clinit>`。**
- 第三点，针对方法里包含的Instruction内容，需要放到Frame当中才能更好的理解。对每一条Instruction来说，它都有可能引起local variables和operand stack的变化。
- **第四点，在使用`COMPUTE_FRAMES`的前提下，我们可以给`visitMaxs()`方法参数传入错误的值，但不能忽略对于`visitMaxs()`方法的调用。**
- **第五点，不同的`MethodVisitor`对象，它们的`visitXxx()`方法是彼此独立的，只要各自遵循方法的调用顺序，就能够得到正确的结果。**

​	最后，本文列举的代码示例是有限的，能够讲到`visitXxxInsn()`方法也是有限的。针对于某一个具体的`visitXxxInsn()`方法，我们可能不太了解它的作用和如何使用它，这个是需要我们在日后的使用过程中一点一点积累和熟悉起来的。

## 2.10 Label介绍

