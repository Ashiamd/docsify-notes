

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

## 1.4 ClassFile结构

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
    u2 max_stack; // 栈相关参数，后续介绍后再回来补充说明
    u2 max_locals; // 栈相关参数，后续介绍后再回来补充说明
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
