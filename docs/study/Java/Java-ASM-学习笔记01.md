

# Java-ASM-学习笔记01

> 视频：[Java ASM系列：（001）课程介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Py4y1g7dN/?spm_id_from=333.788&vd_source=ba4d176271299cb334816d3c4cbc885f)
> 视频对应的gitee：[learn-java-asm: 学习Java ASM (gitee.com)](https://gitee.com/lsieun/learn-java-asm)
> 视频对应的文章：[Java ASM系列一：Core API | lsieun](https://lsieun.github.io/java/asm/java-asm-season-01.html) <= 笔记内容大部分都摘自该文章，下面不再重复声明
>
> ASM官网：[ASM (ow2.io)](https://asm.ow2.io/)
> ASM中文手册：[ASM4 使用手册(中文版) (yuque.com)](https://www.yuque.com/mikaelzero/asm)
>
> Java语言规范和JVM规范-官方文档：[Java SE Specifications (oracle.com)](https://docs.oracle.com/javase/specs/index.html)

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

![Java ASM系列：（003）ASM与ClassFile_Core API_02](https://s2.51cto.com/images/20210619/1624105555567528.png)

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

```java
public abstract class ClassVisitor {
}
```

同时，由于`ClassVisitor`类是一个`abstract`类，要想使用它，就必须有具体的子类来继承它。

第一个比较常见的`ClassVisitor`子类是`ClassWriter`类，属于Core API：

```java
public class ClassWriter extends ClassVisitor {
}
```

第二个比较常见的`ClassVisitor`子类是`ClassNode`类，属于Tree API：

```java
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

![Java ASM系列：（006）ClassVisitor介绍_ASM](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在Java程序中，有三种基本控制结构：顺序、选择和循环。

```pseudocode
                                        ┌─── sequence
                                        │
Java: three basic control structures ───┼─── selection (if, switch)
                                        │
                                        └─── looping (for, while)
```

在Bytecode层面，只存在顺序（sequence）和跳转（jump）两种指令（Instruction）执行顺序：

```pseudocode
                          ┌─── sequence
                          │
Bytecode: control flow ───┤
                          │                ┌─── selection (if, switch)
                          └─── jump ───────┤
                                           └─── looping (for, while)
```

那么，`Label`类起到一个什么样的作用呢？我们现在已经知道，`MethodVisitor`类是用于生成方法体的代码，

- 如果没有`Label`类的参与，那么`MethodVisitor`类只能生成“顺序”结构的代码；
- 如果有`Label`类的参与，那么`MethodVisitor`类就能生成“选择”和“循环”结构的代码。

在本文当中，我们来介绍`Label`类。

如果查看`Label`类的API文档，就会发现下面的描述，分成了三个部分：

- 第一部分，`Label`类上是什么（What）；
- 第二部分，在哪些地方用到`Label`类（Where）；
- 第三部分，在编写ASM代码过程中，如何使用`Label`类（How），或者说，`Label`类与Instruction的关系。
- A position in the bytecode of a method.
- Labels are used for jump, goto, and switch instructions, and for try catch blocks.
- **A label designates the instruction that is just after. Note however that there can be other elements between a label and the instruction it designates (such as other labels, stack map frames, line numbers, etc.).**

如果是刚刚接触`Label`类，那么可能对于上面的三部分英文描述没有太多的“感受”或“理解”；但是，如果接触`Label`类一段时间之后，就会发现它描述的内容很“精髓”。本文的内容也是围绕着这三部分来展开的。

> 这里 labal声明其后面第一个方法体内代码instruction在java字节码中位置，但是其他非instruction的操作也可以放在label和instruction之间，也就是上面提到的other labels, stack map frames, line numbers等。

### 2.10.1 Label类

在`Label`类当中，定义了很多的字段和方法。为了方便，将`Label`类简化一下，内容如下：

```java
public class Label {
  int bytecodeOffset;

  public Label() {
    // Nothing to do.
  }

  public int getOffset() {
    return bytecodeOffset;
  }
}
```

经过这样简单之后，`Label`类当中就只包含一个`bytecodeOffset`字段，那么这个字段代表什么含义呢？**`bytecodeOffset`字段就是a position in the bytecode of a method**。

举例子来说明一下。假如有一个`test(boolean flag)`方法，它包含的Instruction内容如下：

> 这里运行java8-classfile-tutorial项目的`run.K_Code_Locals`的main方法生成以下内容

```shell
=== === ===  === === ===  === === ===
Method test:(Z)V
=== === ===  === === ===  === === ===
max_stack = 2
max_locals = 2
code_length = 24
code = 1B99000EB200021203B60004A7000BB200021205B60004B1
=== === ===  === === ===  === === ===
0000: iload_1              // 1B
0001: ifeq            14   // 99000E
0004: getstatic       #2   // B20002     || java/lang/System.out:Ljava/io/PrintStream;
0007: ldc             #3   // 1203       || value is true
0009: invokevirtual   #4   // B60004     || java/io/PrintStream.println:(Ljava/lang/String;)V
0012: goto            11   // A7000B
0015: getstatic       #2   // B20002     || java/lang/System.out:Ljava/io/PrintStream;
0018: ldc             #5   // 1205       || value is false
0020: invokevirtual   #4   // B60004     || java/io/PrintStream.println:(Ljava/lang/String;)V
0023: return               // B1
=== === ===  === === ===  === === ===
LocalVariableTable:
index  start_pc  length  name_and_type
    0         0      24  this:Lsample/HelloWorld;
    1         0      24  flag:Z
```

**那么，`Label`类当中的`bytecodeOffset`字段，就表示当前Instruction“索引值”。（也就是对应上面代码二进制表现`code`内容中，instruction对应的索引，上图`code`中的索引有`0000`,  `0001`, `0004`,  `0007`, `0009`,  `0012`,  `0015`, `0018`,  `0020`, `0023`）**

**那么，这个`bytecodeOffset`字段是做什么用的呢？它用来计算一个“相对偏移量”。比如说，`bytecodeOffset`字段的值是`15`，它标识了`getstatic`指令的位置，而在索引值为`1`的位置是`ifeq`指令，`ifeq`后面跟的`14`，这个`14`就是一个“相对偏移量”。换一个角度来说，由于`ifeq`的索引位置是`1`，“相对偏移量”是`14`，那么`1+14＝15`，也就是说，如果`ifeq`的条件成立，那么下一条执行的指令就是索引值为`15`的`getstatic`指令了。**

> **注意，这里java字节码中实际存储的是"相对偏移量"，但是后面用`javap -c`分解字节码查看instructions时，其帮忙计算好了"相对偏移量"，直接在shell中对开发者展示计算后的索引值。**

### 2.10.2 Label类能够做什么？

在ASM当中，`Label`类可以用于实现选择（if、switch）、循环（for、while）和try-catch语句。

在编写ASM代码的过程中，我们所要表达的是一种代码的跳转逻辑，就是从一个地方跳转到另外一个地方；在这两者之间，可以编写其它的代码逻辑，可能长一些，也可能短一些，所以，Instruction所对应的“索引值”还不确定。

**`Label`类的出现，就是代表一个“抽象的位置”，也就是将来要跳转的目标。 当我们调用`ClassWriter.toByteArray()`方法时，这些ASM代码会被转换成`byte[]`，在这个过程中，需要计算出`Label`对象中`bytecodeOffset`字段的值到底是多少，从而再进一步计算出跳转的相对偏移量（`offset`）。**

### 2.10.3 如何使用Label类

从编写代码的角度来说，`Label`类是属于`MethodVisitor`类的一部分：通过调用`MethodVisitor.visitLabel(Label)`方法，来为代码逻辑添加一个潜在的“跳转目标”。

我们先来看一个简单的示例代码：

```java
public class HelloWorld {
  public void test(boolean flag) {
    if (flag) {
      System.out.println("value is true");
    }
    else {
      System.out.println("value is false");
    }
  }
}
```

那么，`test(boolean flag)`方法对应的ASM代码如下：

```java
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "test", "(Z)V", null, null);
Label elseLabel = new Label();      // 首先，准备两个Label对象
Label returnLabel = new Label();

// 第1段
mv.visitCode();
mv.visitVarInsn(ILOAD, 1);
// 相当于 false 则跳转
mv.visitJumpInsn(IFEQ, elseLabel);
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("value is true");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
mv.visitJumpInsn(GOTO, returnLabel);

// 第2段
mv.visitLabel(elseLabel);         // 将第一个Label放到这里
mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("value is false");
mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

// 第3段
mv.visitLabel(returnLabel);      // 将第二个Label放到这里
mv.visitInsn(RETURN);
mv.visitMaxs(2, 2);
mv.visitEnd();
```

如何使用`Label`类：

- 首先，创建`Label`类的实例；
- 其次，确定label的位置。通过`MethodVisitor.visitLabel()`方法，确定label的位置。
- 最后，与label建立联系，实现程序的逻辑跳转。在条件合适的情况下，通过`MethodVisitor`类跳转相关的方法（例如，`visitJumpInsn()`）与label建立联系。

A label designates the instruction that is just after. Note however that there can be other elements between a label and the instruction it designates (such as other labels, stack map frames, line numbers, etc.).

上面这段英文描述，是在我们编写ASM代码过程中，label和instruction的位置关系：label在前，instruction在后。

```pseudocode
|          |     instruction     |
|          |     instruction     |
|  label1  |     instruction     |
|          |     instruction     |
|          |     instruction     |
|  label2  |     instruction     |
|          |     instruction     |
```

### 2.10.4 Frame的变化

对于`HelloWorld`类中`test()`方法对应的Instruction内容如下：

```java
public void test(boolean);
  Code:
     0: iload_1
     1: ifeq          15
     4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
     7: ldc           #3                  // String value is true
     9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    12: goto          23
    15: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
    18: ldc           #5                  // String value is false
    20: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    23: return
```

该方法对应的Frame变化情况如下：

```java
test(Z)V
[sample/HelloWorld, int] [] // non-static 方法，所以 local variables 下标0为this指针，boolean byte short char 都会被当作 int处理(统一大小方便处理)
[sample/HelloWorld, int] [int] // iload_1, 将 boolean参数(被当作int处理) 从 local variables 放入 operand stack
[sample/HelloWorld, int] [] // ifeq, operand stack 弹出 boolean值 做跳转判断
[sample/HelloWorld, int] [java/io/PrintStream] // getstatic 为true时 将 System.out 入 operand stack
[sample/HelloWorld, int] [java/io/PrintStream, java/lang/String] // ldc 从常量池加载字符串 入 operand stack
[sample/HelloWorld, int] [] // invokevirtual 调用 println方法，消耗掉栈顶两个参数(实例方法需要消耗实例指针+方法参数;静态方法则只需要消耗方法参数)
[] [] // goto 23, 这里注意 javap -c 直接将计算好的"相对偏移量"+"当前索引"="实际偏移量"展示到开发者眼前，实际class中还是存储的"相对偏移量", goto 和 return一样, ASM 会设置 local variables 和 operand stack 为null
[sample/HelloWorld, int] [java/io/PrintStream]                      // 注意，从上一行到这里是“非线性”的变化, getstatic, 这里是 ifeq 为false时跳转过来的, 将 System.out 入 operand stack
[sample/HelloWorld, int] [java/io/PrintStream, java/lang/String] // ldc 从常量池加载字符串 入 operand stack
[sample/HelloWorld, int] [] // invokevirtual 调用 println方法，消耗掉栈顶两个参数(实例方法需要消耗实例指针+方法参数;静态方法则只需要消耗方法参数)
[] [] // return
```

通过上面的输出结果，我们希望大家能够看到：**由于程序代码逻辑发生了跳转，那么相应的local variables和operand stack结构也发生了“非线性”的变化**。这部分内容与`MethodVisitor.visitFrame()`方法有关系。

> goto 清空本地变量表和操作数栈的代码，可以见`org.objectweb.asm.commons.AnalyzerAdapter#visitJumpInsn`方法
>
> ```java
> @Override
>   public void visitJumpInsn(final int opcode, final Label label) {
>     super.visitJumpInsn(opcode, label);
>     execute(opcode, 0, null);
>     if (opcode == Opcodes.GOTO) {
>       this.locals = null;
>       this.stack = null;
>     }
>   }
> ```

### 2.10.5 小结

本文主要对`Label`类进行了介绍，内容总结如下：

- 第一点，`Label`类是什么（What）。将`Label`类精简之后，就只剩下一个`bytecodeOffset`字段。这个`bytecodeOffset`字段就是`Label`类最精髓的内容，它代表了某一条Instruction的位置。
- **第二点，在哪里用到`Label`类（Where）。简单来说，`Label`类是为了方便程序的跳转，例如实现if、switch、for和try-catch等语句。**
- 第三点，从编写ASM代码的角度来讲，如何使用`Label`类（How）。首先，定义`Label`类的实例；其次，通过`MethodVisitor.visitLabel()`方法确定label的位置；最后，在条件合适的情况下，通过`MethodVisitor`类跳转相关的方法（例如，`visitJumpInsn()`）与label建立联系。
- 第四点，从Frame的角度来讲，由于程序代码逻辑发生了跳转，那么相应的local variables和operand stack结构也发生了“非线性”的变化。

## 2.11 Label代码示例

### 2.11.1 示例一：if语句

#### 预期目标

```java
public class HelloWorld {
  public void test(int value) {
    if (value == 0) {
      System.out.println("value is 0");
    }
    else {
      System.out.println("value is not 0");
    }
  }
}
```

#### 编码实现

注意点：

+ `mv2.visitJumpInsn(IFNE, ifValueNotZero);`这里`IFNE`没有指定value参数和0对比，因为默认就是和0值做比较（C语言中，0为false；非0为true）
+ if代码块内的代码执行完后，记得`GOTO`到return语句，类似汇编语言（如果不GOTO，就相当于继续执行后续代码，也就是执行到else代码块里的代码了）
+ else代码块内的代码执行完后，不需要再显式`GOTO`到return语句，因为后续本来就会执行到return语句

> `if<cond>`: Branch if `int` comparison with zero succeeds
>
> The *value* must be of type `int`. It is popped from the operand stack and compared against zero. All comparisons are signed. The results of the comparisons are as follows:
>
> - *ifeq* succeeds if and only if *value* = 0
> - *ifne* succeeds if and only if *value* ≠ 0
> - *iflt* succeeds if and only if *value* < 0
> - *ifle* succeeds if and only if *value* ≤ 0
> - *ifgt* succeeds if and only if *value* > 0
> - *ifge* succeeds if and only if *value* ≥ 0

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
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

    // test方法
    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "(I)V", null,null);
      Label returnLabel = new Label();
      Label ifValueNotZero = new Label();
      mv2.visitCode();
      // if (value == 0)
      mv2.visitVarInsn(ILOAD, 1);
      mv2.visitJumpInsn(IFNE, ifValueNotZero);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("value is 0");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println","(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO,returnLabel);
      // else
      mv2.visitLabel(ifValueNotZero);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("value is not 0");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println","(Ljava/lang/String;)V", false);

      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 2);

      mv2.visitEnd();
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

    Method method = clazz.getDeclaredMethod("test", int.class);
    method.invoke(obj, 0);
    method.invoke(obj, 1);
  }
}
```

```shell
$ javap -c sample.HelloWorld
public void test(int);
    Code:
       0: iload_1
       1: ifne          15
       4: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #18                 // String value is 0
       9: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: goto          23
      15: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #26                 // String value is not 0
      20: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: return
```

#### 小结

通过上面的示例，我们注意三个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- 第三点，如何通过`Label`类来实现if语句。

### 2.11.2 示例二：switch语句

从Instruction的角度来说，实现switch语句可以使用`lookupswitch`或`tableswitch`指令。

#### 预期目标

```java
public class HelloWorld {
  public void test(int val) {
    switch (val) {
      case 1:
        System.out.println("val = 1");
        break;
      case 2:
        System.out.println("val = 2");
        break;
      case 3:
        System.out.println("val = 3");
        break;
      case 4:
        System.out.println("val = 4");
        break;
      default:
        System.out.println("val is unknown");
    }
  }
}
```

#### 编码实现

注意点：

+ default后没有逻辑了，所以不需要再`GOTO`执行return语句的位置，因为后续自然会走return语句
+ `mv2.visitVarInsn(ILOAD, 1);`需要在`mv2.visitTableSwitchInsn(...)`之前，否则抛出异常
+ 这里和if不同的是，不需要再声明jump操作，switch会按照`visitTableSwitchInsn`内参数声明的顺序，映射1～4对应的case的Label。

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
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

    // test方法
    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "(I)V", null, null);
      Label returnLabel = new Label();
      Label case1Label = new Label();
      Label case2Label = new Label();
      Label case3Label = new Label();
      Label case4Label = new Label();
      Label defaultCaseLabel = new Label();

      mv2.visitCode();
      mv2.visitVarInsn(ILOAD, 1);
      // 下面 visitLookupSwitchInsn 的使用等同于下面 visitTableSwitchInsn 的使用
      // mv2.visitTableSwitchInsn(1, 4, defaultCaseLabel, case1Label, case2Label, case3Label, case4Label);
      mv2.visitLookupSwitchInsn(defaultCaseLabel, new int[]{1, 2, 3, 4}, new Label[]{case1Label, case2Label, case3Label, case4Label});
      // 如果写在 switch之后，则抛出异常 Exception in thread "main" java.lang.NegativeArraySizeException: -1
      // mv2.visitVarInsn(ILOAD, 1);

      // case 1:
      mv2.visitLabel(case1Label);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("val = 1");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      // case 2:
      mv2.visitLabel(case2Label);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("val = 2");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      // case 3:
      mv2.visitLabel(case3Label);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("val = 3");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      // case 4:
      mv2.visitLabel(case4Label);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("val = 4");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      // default:
      mv2.visitLabel(defaultCaseLabel);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("val is unknown");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      // default 后面本来就没有逻辑，直接就执行return了
      //            mv2.visitJumpInsn(GOTO,returnLabel);

      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 2);

      mv2.visitEnd();
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

    Method method = clazz.getDeclaredMethod("test", int.class);
    for (int i = 1; i < 6; i++) {
      method.invoke(obj, i);
    }
  }
}
```

```shell
$ javap -c sample.HelloWorld
 public void test(int);
    Code:
       0: iload_1
       1: lookupswitch  { // 4
                     1: 44
                     2: 55
                     3: 66
                     4: 77
               default: 88
          }
      44: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      47: ldc           #18                 // String val = 1
      49: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      52: goto          96
      55: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      58: ldc           #26                 // String val = 2
      60: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      63: goto          96
      66: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      69: ldc           #28                 // String val = 3
      71: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      74: goto          96
      77: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      80: ldc           #30                 // String val = 4
      82: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      85: goto          96
      88: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      91: ldc           #32                 // String val is unknown
      93: invokevirtual #24                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      96: return
```

#### 小结

通过上面的示例，我们注意三个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- **第三点，如何通过`Label`类来实现switch语句。在本示例当中，使用了`MethodVisitor.visitTableSwitchInsn()`方法，也可以使用`MethodVisitor.visitLookupSwitchInsn()`方法。**

### 2.11.3 示例三：for语句

#### 预期目标

```java
public class HelloWorld {
  public void test() {
    for (int i = 0; i < 10; i++) {
      System.out.println(i);
    }
  }
}
```

#### 编码实现

注意点：

+ `mv2.visitIntInsn(BIPUSH, 10);` 直接将10放入 operand stack
+ `mv2.visitVarInsn(ISTORE, 1);` 在for之前先将operand stack的i变量放回local variables，因为后续for循环需要读取
+ `mv2.visitIincInsn(1,1);` local variables指定下标位置的int数据+1

> Iinc: Increment local variable by constant
>
> The *index* is an unsigned byte that must be an index into the local variable array of the current frame ([§2.6](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6)). The *const* is an immediate signed byte. The local variable at *index* must contain an `int`. The value *const* is first sign-extended to an `int`, and then the local variable at *index* is incremented by that amount.

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
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

    // test方法
    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      Label returnLabel = new Label();
      Label forLabel = new Label();

      mv2.visitCode();
      // i = 0
      mv2.visitInsn(ICONST_0);
      // non-static, 0=>this, 1=>i (存回local variables是因为后续for循环要取出和10比较)
      mv2.visitVarInsn(ISTORE, 1);
      // for 循环
      mv2.visitLabel(forLabel);
      mv2.visitVarInsn(ILOAD, 1);
      mv2.visitIntInsn(BIPUSH, 10);
      // i >= 10 则跳出循环体，循环体外没有其他代码，所以跳转到 return的位置
      mv2.visitJumpInsn(IF_ICMPGE, returnLabel);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ILOAD,1);
      // 注意这里 descriptor 需要改成 "(I)V" 否则报错 Exception in thread "main" java.lang.VerifyError: Bad type on operand stack
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
      // i+=1
      mv2.visitIincInsn(1,1);
      mv2.visitJumpInsn(GOTO, forLabel);


      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 2);
      mv2.visitEnd();
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

    Method method = clazz.getDeclaredMethod("test");
    method.invoke(obj);
  }
}
```

```shell
$ javap -c sample.HelloWorld
public void test();
    Code:
       0: iconst_0
       1: istore_1
       2: iload_1
       3: bipush        10
       5: if_icmpge     21
       8: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
      11: iload_1
      12: invokevirtual #21                 // Method java/io/PrintStream.println:(I)V
      15: iinc          1, 1
      18: goto          2
      21: return
```

#### 小结

通过上面的示例，我们注意三个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- 第三点，如何通过`Label`类来实现for语句。

### 2.11.4 示例四：try-catch语句

#### 预期目标

```java
public class HelloWorld {
  public void test() {
    try {
      System.out.println("Before Sleep");
      Thread.sleep(1000);
      System.out.println("After Sleep");
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
}
```

#### 编码实现

注意点：

+ try代码块前后需要`visitLabel`标记try代码块起始、结束位置
+ `visitTryCatchBlock` 对应 `Code_attribute` 的 `exception_table`

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
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

    // test方法
    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      Label returnLabel = new Label();
      Label tryStartLabel = new Label();
      Label tryEndLabel = new Label();
      Label catchLabel = new Label();

      mv2.visitCode();

      // try 代码块
      // start, end, handler, type (指异常的类型)
      mv2.visitTryCatchBlock(tryStartLabel, tryEndLabel, catchLabel, "java/lang/InterruptedException");
      mv2.visitLabel(tryStartLabel);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("Before Sleep");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitLdcInsn(1000L);
      mv2.visitMethodInsn(INVOKESTATIC, "java/lang/Thread", "sleep", "(J)V", false);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("After Sleep");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitLabel(tryEndLabel);
      mv2.visitJumpInsn(GOTO, returnLabel);

      // catch 代码块
      mv2.visitLabel(catchLabel);
      // 获取 异常对象
      mv2.visitVarInsn(ASTORE, 1);
      mv2.visitVarInsn(ALOAD, 1);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/lang/InterruptedException", "printStackTrace", "()V", false);

      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);

      // visitTryCatchBlock也可以在这里访问
      // mv2.visitTryCatchBlock(tryStartLabel, tryEndLabel, catchLabel, "java/lang/InterruptedException");
      mv2.visitMaxs(2, 2);
      mv2.visitEnd();
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

    Method method = clazz.getDeclaredMethod("test");
    method.invoke(obj);
  }
}
```

```shell
$ javap -c sample.HelloWorld
public void test();
    Code:
       0: getstatic     #17                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #19                 // String Before Sleep
       5: invokevirtual #25                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: ldc2_w        #26                 // long 1000l
      11: invokestatic  #33                 // Method java/lang/Thread.sleep:(J)V
      14: getstatic     #17                 // Field java/lang/System.out:Ljava/io/PrintStream;
      17: ldc           #35                 // String After Sleep
      19: invokevirtual #25                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      22: goto          30
      25: astore_1
      26: aload_1
      27: invokevirtual #38                 // Method java/lang/InterruptedException.printStackTrace:()V
      30: return
    Exception table:
       from    to  target type
           0    22    25   Class java/lang/InterruptedException
```

#### 小结

通过上面的示例，我们注意三个知识点：

- 第一点，如何使用`ClassWriter`类。
- 第二点，在使用`MethodVisitor`类时，其中`visitXxx()`方法需要遵循的调用顺序。
- 第三点，如何通过`Label`类来实现try-catch语句。

有一个问题，`visitTryCatchBlock()`方法为什么可以在后边的位置调用呢？这与`Code`属性的结构有关系：

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

**因为instruction的内容（对应于`visitXxxInsn()`方法的调用）存储于`Code`结构当中的`code[]`内，而try-catch的内容（对应于`visitTryCatchBlock()`方法的调用），存储在`Code`结构当中的`exception_table[]`内，所以`visitTryCatchBlock()`方法的调用时机，可以早一点，也可以晚一点，只要整体上遵循`MethodVisitor`类对就于`visitXxx()`方法调用的顺序要求就可以了**。

> try-catch 和 code(方法体里其他代码逻辑)是分开字段存储的，所以彼此之间顺序不严格要求。

### 2.11.5 总结

本文主要对`Label`类的示例进行介绍，内容总结如下：

- **第一点，`Label`类的主要作用是实现程序代码的跳转，例如，if语句、switch语句、for语句和try-catch语句**。
- **第二点，在生成try-catch语句时，`visitTryCatchBlock()`方法的调用时机，可以早一点，也可以晚一点，只要整体上遵循`MethodVisitor`类对就于`visitXxx()`方法调用的顺序就可以了。**

## 2.12 frame介绍

### 2.12.1 ClassFile中的StackMapTable

​	在`ClassFile`结构中，有一个`StackMapTable`结构，它们关系如下。在`ClassFile`结构中，每一个方法都对应于`method_info`结构；在`method_info`结构中，方法体的代码存储在`Code`结构内；**在`Code`结构中，frame的变化存储在`StackMapTable`结构中**。

![Java ASM系列：（017）frame介绍_ByteCode](https://s2.51cto.com/images/20210621/1624254673117990.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

假如有一个`HelloWorld`类，内容如下

```java
public class HelloWorld {
  public void test(boolean flag) {
    if (flag) {
      System.out.println("value is true");
    }
    else {
      System.out.println("value is false");
    }
  }
}
```

#### 查看Instruction

在`.class`文件中，方法体的内容会被编译成一条一条的instruction。我们可以通过使用`javap -c sample.HelloWorld`来查看Instruction的内容。

```shell
public void test(boolean);
    Code:
       0: iload_1
       1: ifeq          15
       4: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #13                 // String value is true
       9: invokevirtual #15                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: goto          23
      15: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #21                 // String value is false
      20: invokevirtual #15                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: return
```

#### 查看Frame

在方法当中，每一条Instruction都有对应的Frame。

我们可以通过运行`HelloWorldFrameCore`来查看Frame的具体情况：

```java
test(Z)V
[sample/HelloWorld, int] [] // non-static, local variables 下标0存储this指针，1存储boolean参数（比int占用空间小的基础类型都被转成int处理了）
[sample/HelloWorld, int] [int] // iload_1, 将 local variables 下标1 位置的int数据加载到 operand stack顶部
[sample/HelloWorld, int] [] // ifeq, 弹出 operand stack 栈顶元素，和0比较，如果相同则跳转
[sample/HelloWorld, int] [java/io/PrintStream] // getstatic, 加载 System.out 这个 静态成员变量到 operand stack顶部
[sample/HelloWorld, int] [java/io/PrintStream, java/lang/String] // ldc 从常量池获取常量字符串 到 operand stack顶部
[sample/HelloWorld, int] [] // invokevirtual 调用 System.out 的实例方法 println，消耗 operand stack栈顶的参数和后续的对象指针
[] [] // goto 23, .class中实际存的是偏移量，javap -c 帮忙计算好了偏移后需要跳转的位置，goto指令会清空 local variabls 和 operand stack
[sample/HelloWorld, int] [java/io/PrintStream] // getstatic 
[sample/HelloWorld, int] [java/io/PrintStream, java/lang/String] // ldc
[sample/HelloWorld, int] [] // invokevirtual
[] [] // return, 会清空 local variables 和 operand stack
```

**严格的来说，每一条Instruction都对应两个frame，一个是instruction执行之前的frame，另一个是instruction执行之后的frame**。<u>但是，当多个instruction放到一起的时候来说，第`n`个instruction执行之后的frame，就成为第`n+1`个instruction执行之前的frame，所以也可以理解成：每一条instruction对应一个frame</u>。

**这些frames是要存储起来的。我们知道，每一个instruction对应一个frame，如果都要存储起来，那么在`.class`文件中就会占用非常多的空间；而`.class`文件设计的一个主要目标就是尽量占用较小的存储空间，那么就需要对这些frames进行压缩**。

#### 压缩frames

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.4) <= 4.7.4. The `StackMapTable` Attribute，具体可以看java文档
>
> The `StackMapTable` attribute is a variable-length attribute in the `attributes` table of a `Code` attribute ([§4.7.3](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.3)). A `StackMapTable` attribute is used during the process of verification by type checking ([§4.10.1](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.10.1)).
>
> Each stack map frame described in the `entries` table relies on the previous frame for some of its semantics. **The first stack map frame of a method is implicit, and computed from the method descriptor by the type checker ([§4.10.1.6](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.10.1.6))**. The `stack_map_frame` structure at `entries[0]` therefore describes the second stack map frame of the method.
>
> 文档中也提到了初始状态的frame可以通过方法descriptor计算出来，所以实际上`stack_map_frame` entries数组的下标0的frame其实是第二个需要被存储的frame。

为了让`.class`文件占用的存储空间尽可能的小，因此要对frames进行压缩。

**对frames进行压缩，从本质上来说，就是忽略掉一些不重要的frames，而只留下一些重要的frames。**

那么，怎样区分哪些frames重要，哪些frames不重要呢？我们从instruction执行顺序的角度来看待这个问题。

**<u>如果说，instruction是按照“一个挨一个向下顺序执行”的，那么它们对应的frames就不重要；相应的，instruction在执行过程时，它是从某个地方“跳转”过来的，那么对应的frames就重要。</u>**

为什么说instruction按照“一个挨一个向下顺序执行”的frames不重要呢？因为这些instruction对应的frame可以很容易的推导出来。 相反，如果当前的instruction是从某个地方跳转过来的，就必须要记录它执行之前的frame的情况，否则就没有办法计算它执行之后的frame的情况。当然，我们这里讲的只是大体的思路，而不是具体的判断细节。

**经过压缩之后的frames，就存放在`ClassFile`的`StackMapTable`结构中**。

### 2.12.2 如何使用visitFrame()方法

#### 预期目标

```java
public class HelloWorld {
  public void test(boolean flag) {
    if (flag) {
      System.out.println("value is true");
    }
    else {
      System.out.println("value is false");
    }
  }
}
```

#### 编码实现

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.4)
>
> The frame type `same_frame` is represented by tags in the range [0-63]. This frame type indicates that the frame has exactly the same local variables as the previous frame and that the operand stack is empty. The `offset_delta` value for the frame is the value of the tag item, `frame_type`.

注意点：

+ `ClassWriter(ClassWriter.COMPUTE_MAXS);`，此时ASM只帮忙计算ClassFile的Code_attribute的max_stack和max_locals，但是不帮忙计算stack_map_frame，需要我们自己调用`visitFrame()`方法
+ 只需要在发生跳转的Label后面的第一个Insn之前调用`visitFrame()`，代码初始状态不需要`visitFrame()`，因为stack_map_frame中存储的是压缩后的frame信息，而代码初始状态可以通过方法签名推导出来local variables和operand stack，为减少`.class`的文件大小， 就不需要存储初始状态frame。而jump动作需要知道上一个被执行的指令的frame，所以需要记录到stack_map_frame中
+ `mv2.visitFrame(F_SAME,0, null, 0, null);`这里`F_SAME`表示和上一个指令执行后的frame状态一致（具体指local variables相同，且operand stack都为空）

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
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
    // (1) 创建 ClassWriter对象 (注意这里改成 ClassWriter.COMPUTE_MAXS, 只会计算 max_locals 和 max_stack，但是不会帮忙计算 stack map frames)
    // ClassWriter.COMPUTE_MAXS 则下面需要手动调用 visitFrame()
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);

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

    // test方法
    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "(Z)V", null, null);
      Label returnLabel = new Label();
      Label elseLabel = new Label();

      mv2.visitCode();
      mv2.visitVarInsn(ILOAD,1);
      mv2.visitJumpInsn(IFEQ, elseLabel);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System","out","Ljava/io/PrintStream;");
      mv2.visitLdcInsn("value is true");
      mv2.visitMethodInsn(INVOKEVIRTUAL,"java/io/PrintStream", "println", "(Ljava/lang/String;)V",false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      mv2.visitLabel(elseLabel);
      // 注意 visitFrame 在 visitLabel之后，下一个操作Insn之前
      // type, numLocal, local, numStack, stack
      mv2.visitFrame(F_SAME,0, null, 0, null);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System","out","Ljava/io/PrintStream;");
      mv2.visitLdcInsn("value is false");
      mv2.visitMethodInsn(INVOKEVIRTUAL,"java/io/PrintStream", "println", "(Ljava/lang/String;)V",false);

      mv2.visitLabel(returnLabel);
      // 注意 visitFrame 在 visitLabel之后，下一个操作Insn之前
      // type, numLocal, local, numStack, stack
      mv2.visitFrame(F_SAME,0, null, 0, null);
      mv2.visitInsn(RETURN);

      // visitTryCatchBlock也可以在这里访问
      // mv2.visitTryCatchBlock(tryStartLabel, tryEndLabel, catchLabel, "java/lang/InterruptedException");
      mv2.visitMaxs(2, 2);
      mv2.visitEnd();
    }


    cw.visitEnd();

    // (3) 调用toByteArray() 方法
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
package com.example;

import java.lang.reflect.Method;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("sample.HelloWorld");
        Object obj = clazz.newInstance();

        Method method = clazz.getDeclaredMethod("test", boolean.class);
        method.invoke(obj, true);
        method.invoke(obj, false);
    }
}
```

### 2.12.3 不推荐使用visitFrame()方法

为什么我们不推荐调用`MethodVisitor.visitFrame()`方法呢？原因是计算frame本身就很麻烦，还容易出错。

我们在创建`ClassWriter`对象的时候，使用了`ClassWriter.COMPUTE_FRAMES`参数：

```java
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
```

**在使用了`ClassWriter.COMPUTE_FRAMES`参数之后，ASM会忽略代码当中对于`MethodVisitor.visitFrame()`方法的调用，并且自动帮助我们计算stack map frame的具体内容。**

### 2.12.4 总结

本文主要对frame进行了介绍，内容总结如下：

- 第一点，在`ClassFile`结构中，`StackMapTable`结构是如何得到的。
- 第二点，不推荐使用`MethodVisitor.visitFrame()`方法，原因是frame的计算复杂，容易出错。我们可以在创建`ClassWriter`对象的时候，使用`ClassWriter.COMPUTE_FRAMES`参数，这样ASM就会帮助我们计算frame的值到底是多少。

## 2.13 Opcodes介绍

`Opcodes`是一个接口，它定义了许多字段。这些字段主要是在`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法中使用。

### 2.13.1 ClassVisitor

#### ASM Version

字段含义：`Opcodes.ASM4`~`Opcodes.ASM9`标识了ASM的版本信息。

应用场景：用于创建具体的`ClassVisitor`实例，例如`ClassVisitor(int api, ClassVisitor classVisitor)`中的`api`参数。

```java
public interface Opcodes {
  // ASM API versions.
  int ASM4 = 4 << 16 | 0 << 8;
  int ASM5 = 5 << 16 | 0 << 8;
  int ASM6 = 6 << 16 | 0 << 8;
  int ASM7 = 7 << 16 | 0 << 8;
  int ASM8 = 8 << 16 | 0 << 8;
  int ASM9 = 9 << 16 | 0 << 8;
}
```

#### Java Version

字段含义：`Opcodes.V1_1`~`Opcodes.V16`标识了`.class`文件的版本信息。

应用场景：用于`ClassVisitor.visit(int version, int access, ...)`的`version`参数。

```java
public interface Opcodes {
  // Java ClassFile versions
  // (the minor version is stored in the 16 most significant bits, and
  //  the major version in the 16 least significant bits).

  int V1_1 = 3 << 16 | 45;
  int V1_2 = 0 << 16 | 46;
  int V1_3 = 0 << 16 | 47;
  int V1_4 = 0 << 16 | 48;
  int V1_5 = 0 << 16 | 49;
  int V1_6 = 0 << 16 | 50;
  int V1_7 = 0 << 16 | 51;
  int V1_8 = 0 << 16 | 52;
  int V9 = 0 << 16 | 53;
  int V10 = 0 << 16 | 54;
  int V11 = 0 << 16 | 55;
  int V12 = 0 << 16 | 56;
  int V13 = 0 << 16 | 57;
  int V14 = 0 << 16 | 58;
  int V15 = 0 << 16 | 59;
  int V16 = 0 << 16 | 60;
  int V17 = 0 << 16 | 61;
  int V18 = 0 << 16 | 62;
  int V19 = 0 << 16 | 63;
  int V20 = 0 << 16 | 64;
}
```

#### Access Flags

字段含义：`Opcodes.ACC_PUBLIC`~`Opcodes.ACC_MODULE`标识了Class、Field、Method的访问标识（Access Flag）。

应用场景：

- `ClassVisitor.visit(int version, int access, ...)`的`access`参数。
- `ClassVisitor.visitField(int access, String name, ...)`的`access`参数。
- `ClassVisitor.visitMethod(int access, String name, ...)`的`access`参数。

```java
public interface Opcodes {
  int ACC_PUBLIC = 0x0001;       // class, field, method
  int ACC_PRIVATE = 0x0002;      // class, field, method
  int ACC_PROTECTED = 0x0004;    // class, field, method
  int ACC_STATIC = 0x0008;       // field, method
  int ACC_FINAL = 0x0010;        // class, field, method, parameter
  int ACC_SUPER = 0x0020;        // class
  int ACC_SYNCHRONIZED = 0x0020; // method
  int ACC_OPEN = 0x0020;         // module
  int ACC_TRANSITIVE = 0x0020;   // module requires
  int ACC_VOLATILE = 0x0040;     // field
  int ACC_BRIDGE = 0x0040;       // method
  int ACC_STATIC_PHASE = 0x0040; // module requires
  int ACC_VARARGS = 0x0080;      // method
  int ACC_TRANSIENT = 0x0080;    // field
  int ACC_NATIVE = 0x0100;       // method
  int ACC_INTERFACE = 0x0200;    // class
  int ACC_ABSTRACT = 0x0400;     // class, method
  int ACC_STRICT = 0x0800;       // method
  int ACC_SYNTHETIC = 0x1000;    // class, field, method, parameter, module *
  int ACC_ANNOTATION = 0x2000;   // class
  int ACC_ENUM = 0x4000;         // class(?) field inner
  int ACC_MANDATED = 0x8000;     // field, method, parameter, module, module *
  int ACC_MODULE = 0x8000;       // class
}
```

### 2.13.2 MethodVisitor

#### frame

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html#jvms-4.7.4)

**字段含义：`Opcodes.F_NEW`~`Opcodes.F_SAME1`标识了frame的状态，`Opcodes.TOP`~`Opcodes.UNINITIALIZED_THIS`标识了frame中某一个数据项的具体类型**。

应用场景：

- `Opcodes.F_NEW`~`Opcodes.F_SAME1`用在`MethodVisitor.visitFrame(int type, int numLocal, Object[] local, int numStack, Object[] stack)`方法中的`type`参数。
- `Opcodes.TOP`~`Opcodes.UNINITIALIZED_THIS`用在`MethodVisitor.visitFrame(int type, int numLocal, Object[] local, int numStack, Object[] stack)`方法中的`local`参数和`stack`参数。

```java
public interface Opcodes {
  // ASM specific stack map frame types, used in {@link ClassVisitor#visitFrame}.
  int F_NEW = -1;
  int F_FULL = 0;
  int F_APPEND = 1;
  int F_CHOP = 2;
  int F_SAME = 3;
  int F_SAME1 = 4;


  // Standard stack map frame element types, used in {@link ClassVisitor#visitFrame}.
  Integer TOP = Frame.ITEM_TOP; // TOP 就是 long、double 由于需要2个槽位(1槽位32bit), 用于占位(占第二个槽位)
  Integer INTEGER = Frame.ITEM_INTEGER;
  Integer FLOAT = Frame.ITEM_FLOAT;
  Integer DOUBLE = Frame.ITEM_DOUBLE;
  Integer LONG = Frame.ITEM_LONG;
  Integer NULL = Frame.ITEM_NULL;
  Integer UNINITIALIZED_THIS = Frame.ITEM_UNINITIALIZED_THIS;
}
```

#### opcodes

字段含义：`Opcodes.NOP`~`Opcodes.IFNONNULL`表示opcode的值。

应用场景：在`MethodVisitor.visitXxxInsn(opcode)`方法中的`opcode`参数中使用。

```java
public interface Opcodes {
  int NOP = 0; // visitInsn
  int ACONST_NULL = 1; // -
  int ICONST_M1 = 2; // -
  int ICONST_0 = 3; // -
  int ICONST_1 = 4; // -
  int ICONST_2 = 5; // -
  int ICONST_3 = 6; // -
  int ICONST_4 = 7; // -
  int ICONST_5 = 8; // -
  int LCONST_0 = 9; // -
  int LCONST_1 = 10; // -
  int FCONST_0 = 11; // -
  int FCONST_1 = 12; // -
  int FCONST_2 = 13; // -
  int DCONST_0 = 14; // -
  int DCONST_1 = 15; // -
  int BIPUSH = 16; // visitIntInsn
  int SIPUSH = 17; // -
  int LDC = 18; // visitLdcInsn
  int ILOAD = 21; // visitVarInsn
  int LLOAD = 22; // -
  int FLOAD = 23; // -
  int DLOAD = 24; // -
  int ALOAD = 25; // -
  int IALOAD = 46; // visitInsn
  int LALOAD = 47; // -
  int FALOAD = 48; // -
  int DALOAD = 49; // -
  int AALOAD = 50; // -
  int BALOAD = 51; // -
  int CALOAD = 52; // -
  int SALOAD = 53; // -
  int ISTORE = 54; // visitVarInsn
  int LSTORE = 55; // -
  int FSTORE = 56; // -
  int DSTORE = 57; // -
  int ASTORE = 58; // -
  int IASTORE = 79; // visitInsn
  int LASTORE = 80; // -
  int FASTORE = 81; // -
  int DASTORE = 82; // -
  int AASTORE = 83; // -
  int BASTORE = 84; // -
  int CASTORE = 85; // -
  int SASTORE = 86; // -
  int POP = 87; // -
  int POP2 = 88; // -
  int DUP = 89; // -
  int DUP_X1 = 90; // -
  int DUP_X2 = 91; // -
  int DUP2 = 92; // -
  int DUP2_X1 = 93; // -
  int DUP2_X2 = 94; // -
  int SWAP = 95; // -
  int IADD = 96; // -
  int LADD = 97; // -
  int FADD = 98; // -
  int DADD = 99; // -
  int ISUB = 100; // -
  int LSUB = 101; // -
  int FSUB = 102; // -
  int DSUB = 103; // -
  int IMUL = 104; // -
  int LMUL = 105; // -
  int FMUL = 106; // -
  int DMUL = 107; // -
  int IDIV = 108; // -
  int LDIV = 109; // -
  int FDIV = 110; // -
  int DDIV = 111; // -
  int IREM = 112; // -
  int LREM = 113; // -
  int FREM = 114; // -
  int DREM = 115; // -
  int INEG = 116; // -
  int LNEG = 117; // -
  int FNEG = 118; // -
  int DNEG = 119; // -
  int ISHL = 120; // -
  int LSHL = 121; // -
  int ISHR = 122; // -
  int LSHR = 123; // -
  int IUSHR = 124; // -
  int LUSHR = 125; // -
  int IAND = 126; // -
  int LAND = 127; // -
  int IOR = 128; // -
  int LOR = 129; // -
  int IXOR = 130; // -
  int LXOR = 131; // -
  int IINC = 132; // visitIincInsn
  int I2L = 133; // visitInsn
  int I2F = 134; // -
  int I2D = 135; // -
  int L2I = 136; // -
  int L2F = 137; // -
  int L2D = 138; // -
  int F2I = 139; // -
  int F2L = 140; // -
  int F2D = 141; // -
  int D2I = 142; // -
  int D2L = 143; // -
  int D2F = 144; // -
  int I2B = 145; // -
  int I2C = 146; // -
  int I2S = 147; // -
  int LCMP = 148; // -
  int FCMPL = 149; // -
  int FCMPG = 150; // -
  int DCMPL = 151; // -
  int DCMPG = 152; // -
  int IFEQ = 153; // visitJumpInsn
  int IFNE = 154; // -
  int IFLT = 155; // -
  int IFGE = 156; // -
  int IFGT = 157; // -
  int IFLE = 158; // -
  int IF_ICMPEQ = 159; // -
  int IF_ICMPNE = 160; // -
  int IF_ICMPLT = 161; // -
  int IF_ICMPGE = 162; // -
  int IF_ICMPGT = 163; // -
  int IF_ICMPLE = 164; // -
  int IF_ACMPEQ = 165; // -
  int IF_ACMPNE = 166; // -
  int GOTO = 167; // -
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
  int INVOKEVIRTUAL = 182; // visitMethodInsn
  int INVOKESPECIAL = 183; // -
  int INVOKESTATIC = 184; // -
  int INVOKEINTERFACE = 185; // -
  int INVOKEDYNAMIC = 186; // visitInvokeDynamicInsn
  int NEW = 187; // visitTypeInsn
  int NEWARRAY = 188; // visitIntInsn
  int ANEWARRAY = 189; // visitTypeInsn
  int ARRAYLENGTH = 190; // visitInsn
  int ATHROW = 191; // -
  int CHECKCAST = 192; // visitTypeInsn
  int INSTANCEOF = 193; // -
  int MONITORENTER = 194; // visitInsn
  int MONITOREXIT = 195; // -
  int MULTIANEWARRAY = 197; // visitMultiANewArrayInsn
  int IFNULL = 198; // visitJumpInsn
  int IFNONNULL = 199; // -
}
```

#### opcode: newarray

字段含义：`Opcodes.T_BOOLEAN`~`Opcodes.T_LONG`表示数组的类型。

应用场景：对于`MethodVisitor.visitIntInsn(opcode, operand)`方法，当`opcode`为`NEWARRAY`时，`operand`参数中使用。

```java
public interface Opcodes {
  // Possible values for the type operand of the NEWARRAY instruction.
  int T_BOOLEAN = 4;
  int T_CHAR = 5;
  int T_FLOAT = 6;
  int T_DOUBLE = 7;
  int T_BYTE = 8;
  int T_SHORT = 9;
  int T_INT = 10;
  int T_LONG = 11;
}
```

```java
public class HelloWorld {
  public void test() {
    byte[] bytes = new byte[10];
  }
}
```

#### opcode: invokedynamic

字段含义：`Opcodes.H_GETFIELD`~`Opcodes.H_INVOKEINTERFACE`表示MethodHandle的类型。

应用场景：在创建`Handle(int tag, String owner, String name, String descriptor, boolean isInterface)`时，`tag`参数中使用；而该`Handle`实例会在`MethodVisitor.visitInvokeDynamicInsn()`方法使用到。

```java
public interface Opcodes {
  // Possible values for the reference_kind field of CONSTANT_MethodHandle_info structures.
  int H_GETFIELD = 1;
  int H_GETSTATIC = 2;
  int H_PUTFIELD = 3;
  int H_PUTSTATIC = 4;
  int H_INVOKEVIRTUAL = 5;
  int H_INVOKESTATIC = 6;
  int H_INVOKESPECIAL = 7;
  int H_NEWINVOKESPECIAL = 8;
  int H_INVOKEINTERFACE = 9;
}
```

```java
import java.util.function.BiFunction;

public class HelloWorld {
  public void test() {
    BiFunction<Integer, Integer, Integer> func = Math::max;
  }
}
```

### 2.12.3 总结

本文主要对`Opcodes`接口里定义的字段进行介绍，内容总结如下：

- 第一点，在`Opcodes`类定义的字段，主要应用于`ClassVisitor`和`MethodVisitor`类的`visitXxx()`方法。
- 第二点，记忆方法。由于`Opcodes`类定义的字段很多，我们可以分成不同的批次和类别来进行理解，慢慢去掌握。

## 2.14 本章内容总结

![Java ASM系列：（019）第二章内容总结_ByteCode](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

本章内容是围绕着Class Generation（生成新的类）来展开，在这个过程当中，我们介绍了ASM Core API当中的一些类和接口。在本文当中，我们对这些内容进行一个总结。

### 2.14.1 Java ClassFile

如果我们想要生成一个`.class`文件，就需要先对`.class`文件所遵循的文件格式（或者说是数据结构）有所了解。`.class`文件所遵循的数据结构是由[ Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)定义的，其结构如下：

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

对上面的条目来进行一个简单的介绍：

- `magic`：表示magic number，是一个固定值`CAFEBABE`，它是一个标识信息，用来判断当前文件是否为ClassFile。其实，不只是`.class`文件有magic number。例如，`.pdf`文件的magic number是`%PDF`，`.png`文件的magic number是`PNG`。
- `minor_version`和`major_version`：表示当前`.class`文件的版本信息。因为Java语言不断发展，就存在不同版本之间的差异；记录`.class`文件的版本信息，是为了判断JVM的版本的`.class`文件的版本是否兼容。高版本的JVM可以执行低版本的`.class`文件，但是低版本的JVM不能执行高版本的`.class`文件。
- `constant_pool_count`和`constant_pool`：表示“常量池”信息，它是一个“资源仓库”。在这里面，存放了当前类的类名、父类的类名、所实现的接口名字，后面的`this_class`、`super_class`和`interfaces[]`存放的是一个索引值，该索引值指向常量池。
- `access_flags`、`this_class`、`super_class`、`interfaces_count`和`interfaces`：表示当前类的访问标识、类名、父类、实现接口的数量和具体的接口。
- `fields_count`和`fields`：表示字段的数量和具体的字段内容。
- `methods_count`和`methods`：表示方法的数量和具体的方法内容。
- `attributes_count`和`attributes`：表示属性的数量和具体的属性内容。

总结一下就是，magic number是为了区分不同产品（PDF、PNG、ClassFile）之间的差异，而version则是为了区分同一个产品在不同版本之间的差异。 接下来的Constant Pool、Class Info、Fields、Methods和Attributes则是实实在在的映射`.class`文件当中的内容。

我们可以把这个Java ClassFile和一个Java文件的内容来做一个对照：

```java
public class HelloWorld extends Object implements Cloneable {
    private int intValue = 10;
    private String strValue = "ABC";

    public int add(int a, int b) {
        return a + b;
    }

    public int sub(int a, int b) {
        return a - b;
    }
}
```

### 2.14.2 ASM Core API

要生成一个`.class`文件，直接使用记事本或十六进制编辑器，这是“不可靠的”，所以我们借助于ASM这个类库，使用其中Core API部分来帮助我们实现。

讲到任何的API，其实就是讲它的类、接口、方法等内容；谈到ASM Core API就是讲其中涉及到的类、接口和里面的方法。在ASM Core API中，有三个非常重要的类，即`ClassReader`、`ClassVisitor`和`ClassWriter`类。但是，在Class Generation过程中，不会用到`ClassReader`，所以我们就主要关注`ClassVisitor`和`ClassWriter`类。

![Java ASM系列：（019）第二章内容总结_ByteCode_02](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

#### ClassVisitor和ClassWriter

在`ClassVisitor`类当中，定义了许多的`visitXxx()`方法，并且这些`visitXxx()`方法要遵循一定的调用顺序。我们把这些`visitXxx()`方法进行精简，得到4个`visitXxx()`方法：

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

我们可以将这4个`visitXxx()`方法，与Java ClassFile进行比对，这样我们就能够理解“为什么会有这4个方法”以及“方法要接收参数的含义是什么”。

但是，`ClassVisitor`类是一个抽象类，我们需要它的一个具体子类。这时候，就引出了`ClassWriter`类，它是`ClassVisitor`类的子类，继承了`visitXxx()`方法。同时，`ClassWriter`类也定义了一个`toByteArray()`方法，它可以将`visitXxx()`方法执行后的结果转换成`byte[]`。在创建`ClassWriter(flags)`对象的时候，对于`flags`参数，我们推荐使用`ClassWriter.COMPUTE_FRAMES`。

使用`ClassWriter`生成一个Class文件，可以大致分成三个步骤：

- 第一步，创建`ClassWriter`对象。
- 第二步，调用`ClassWriter`对象的`visitXxx()`方法。
- 第三步，调用`ClassWriter`对象的`toByteArray()`方法。

示例代码如下：

```java
import org.objectweb.asm.ClassWriter;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateCore {
    public static byte[] dump () throws Exception {
        // (1) 创建ClassWriter对象
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

        // (2) 调用visitXxx()方法
        cw.visit();
        cw.visitField();
        cw.visitMethod();
        cw.visitEnd();       // 注意，最后要调用visitEnd()方法

        // (3) 调用toByteArray()方法
        byte[] bytes = cw.toByteArray();
        return bytes;
    }
}
```

我们在介绍Class Generation示例的时候，直接使用`ClassWriter`类就可以了。但是，除了`ClassVisitor`和`ClassWriter`类，我们还需要更多的类来“丰富”这个类的“细节信息”，比如说`FieldVisitor`和`MethodVisitor`，它们分别是为了“丰富”字段和方法的具体信息。

#### FieldVisitor和MethodVisitor

在`ClassVisitor`类当中，`visitField()`方法会返回一个`FieldVisitor`对象，`visitMethod()`方法会返回一个`MethodVisitor`对象。其实，`FieldVisitor`对象和`MethodVisitor`对象是为了让生成的字段和方法内容更“丰富、充实”。

相对来说，`FieldVisitor`类比较简单，在刚开始学的时候，我们只需要关注它的`visitEnd()`方法就可以了。

相对来说，`MethodVisitor`类就比较复杂，因为在调用`ClassVisitor.visitMethod()`方法的时候，只是说明了方法的名字、方法的参数类型、方法的描述符等信息，并没有说明方法的“方法体”信息，所以我们需要使用具体的`MethodVisitor`对象来实现具体的方法体。在`MethodVisitor`类当中，也定义了许多的`visitXxx()`方法。这里要注意一下，要注意与`ClassVisitor`类里定义的`visitXxx()`方法区分。**`ClassVisitor`类里的`visitXxx()`方法是提供类层面的信息，而`MethodVisitor`类里的`visitXxx()`方法是提供某一个具体方法里的信息。**

`MethodVisitor`类里的`visitXxx()`方法，也需要遵循一定的调用顺序，精简之后，如下：

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

我们可以按照下面来记忆`visitXxx()`方法的调用顺序：

- 第一步，调用`visitCode()`方法，调用一次
- 第二步，调用`visitXxxInsn()`方法，可以调用多次。对这些方法的调用，就是在构建方法的“方法体”。
- 第三步，调用`visitMaxs()`方法，调用一次
- 第四步，调用`visitEnd()`方法，调用一次

> `ClassVisitor`类里的`visitXxx()`方法需要遵循一定的调用顺序，`MethodVisitor`类里的`visitXxx()`方法也需要遵循一定的调用顺序。

另外，我们也需要特别注意一些特殊的方法名字，例如，**构造方法的名字是`<init>`，而静态初始化方法的名字是`<clinit>`。**

我们在使用`MethodVisitor`来编写方法体的代码逻辑时，不可避免的会遇到程序逻辑`true`和`false`判断和执行流程的跳转，而`Label`在ASM代码中就标志着跳转的位置。**借助于`Label`类，我们可以实现if语句、switch语句、for语句和try-catch语句**。添加label位置，是通过`MethodVisitor.visitLabel()`方法实现的。

在Java 6之后，为了对方法的代码进行校验，于是就增加了`StackMapTable`属性。**谈到`StackMapTable`属性，其实就是我们讲到的frame，就是记录某一条instruction所对应的local variables和operand stack的状态。我们不推荐大家自己计算frame，因此不推荐使用`MethodVisitor.visitFrame()`方法。**

无论是`Label`类，还是frame，它们都是`MethodVisitor`在实现“方法体”过程当中的“细节信息”，所以我们把这两者放到`MethodVisitor`一起来说明。

#### 常量池去哪儿了？

有细心的同学，可能会发现这样的问题：在ASM当中，常量池去哪儿了？为什么没有常量池相关的类和方法呢？

其实，**在ASM源码中，与常量池对应的是`SymbolTable`类**，但我们并没有对它进行介绍。为什么没有介绍呢？

- 第一个原因，在调用`ClassVisitor.visitXxx()`方法和`MethodVisitor.visitXxx()`方法的过程中，ASM会自动帮助我们去构建`SymbolTable`类里面具体的内容。
- 第二个原因，常量池中包含十几种具体的常量类型，内容多而复杂，需要结合Java ClassFile相关的知识才能理解。

我们的关注点还是在于如何使用Core API来进行Class Generation操作，ASM的内部实现会帮助我们处理好`SymbolTable`类的内容。

### 2.14.3 总结

本文是对第二章的整体内容进行总结，大家可以从两方面进行把握：一个是Java ClassFile的格式是什么的，另一个就是ASM Core API里的具体类和方法的作用。

# 3. 转换已有的类

## 3.1 ClassReader介绍

### 3.1.1 ClassReader类

`ClassReader`类和`ClassWriter`类，从功能角度来说，是完全相反的两个类，一个用于读取`.class`文件，另一个用于生成`.class`文件。

#### class info

第一个部分，`ClassReader`的父类是`Object`类。与`ClassWriter`类不同的是，`ClassReader`类并没有继承自`ClassVisitor`类。

`ClassReader`类的定义如下：

```java
/**
 * A parser to make a {@link ClassVisitor} visit a ClassFile structure, as defined in the Java
 * Virtual Machine Specification (JVMS). This class parses the ClassFile content and calls the
 * appropriate visit methods of a given {@link ClassVisitor} for each field, method and bytecode
 * instruction encountered.
 *
 * @see <a href="https://docs.oracle.com/javase/specs/jvms/se9/html/jvms-4.html">JVMS 4</a>
 * @author Eric Bruneton
 * @author Eugene Kuleshov
 */
public class ClassReader {
}
```

`ClassWriter`类的定义如下：

```java
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

第二个部分，`ClassReader`类定义的字段有哪些。我们选取出其中的3个字段进行介绍，即`classFileBuffer`字段、`cpInfoOffsets`字段和`header`字段。

```java
public class ClassReader {
  /**
   * A byte array containing the JVMS ClassFile structure to be parsed. <i>The content of this array
   * must not be modified. This field is intended for {@link Attribute} sub classes, and is normally
   * not needed by class visitors.</i>
   *
   * <p>NOTE: the ClassFile structure can start at any offset within this array, i.e. it does not
   * necessarily start at offset 0. Use {@link #getItem} and {@link #header} to get correct
   * ClassFile element offsets within this byte array.
   */  
  //第1组，真实的数据部分
  final byte[] classFileBuffer;

  /**
   * The offset in bytes, in {@link #classFileBuffer}, of each cp_info entry of the ClassFile's
   * constant_pool array, <i>plus one</i>. In other words, the offset of constant pool entry i is
   * given by cpInfoOffsets[i] - 1, i.e. its cp_info's tag field is given by b[cpInfoOffsets[i] -
   * 1].
   */
  //第2组，数据的索引信息
  private final int[] cpInfoOffsets;

  /** The offset in bytes of the ClassFile's access_flags field. */
  public final int header;
}
```

为什么选择这3个字段呢？因为这3个字段能够体现出`ClassReader`类处理`.class`文件的整体思路：

- **第1组，`classFileBuffer`字段：它里面包含的信息，就是从`.class`文件中读取出来的字节码数据。**
- **第2组，`cpInfoOffsets`字段和`header`字段：它们分别标识了`classFileBuffer`中数据里包含的常量池（constant pool）和访问标识（access flag）的位置信息。**

我们拿到`classFileBuffer`字段后，一个主要目的就是对它的内容进行修改，来实现一个新的功能。它处理的大体思路是这样的：

```
.class文件 --> ClassReader --> byte[] --> 经过各种转换 --> ClassWriter --> byte[] --> .class文件
```

- 第一，从一个`.class`文件（例如`HelloWorld.class`）开始，它可能存储于磁盘的某个位置；
- 第二，使用`ClassReader`类将这个`.class`文件的内容读取出来，其实这些内容（`byte[]`）就是`ClassReader`对象中的`classFileBuffer`字段的内容；
- 第三，为了增加某些功能，就对这些原始内容（`byte[]`）进行转换；
- 第四，等各种转换都完成之后，再交给`ClassWriter`类处理，调用它的`toByteArray()`方法，从而得到新的内容（`byte[]`）；
- 第五，将新生成的内容（`byte[]`）存储到一个具体的`.class`文件中，那么这个新的`.class`文件就具备了一些新的功能。

#### constructors

第三个部分，`ClassReader`类定义的构造方法有哪些。在`ClassReader`类当中定义了5个构造方法。但是，从本质上来说，这5个构造方法本质上是同一个构造方法的不同表现形式。其中，最常用的构造方法有两个：

- 第一个是`ClassReader cr = new ClassReader("sample.HelloWorld");`
- 第二个是`ClassReader cr = new ClassReader(bytes);`

```java
public class ClassReader {

    public ClassReader(final String className) throws IOException { // 第一个构造方法（常用）
        this(
            readStream(ClassLoader.getSystemResourceAsStream(className.replace('.', '/') + ".class"), true)
        );
    }

    public ClassReader(final byte[] classFile) { // 第二个构造方法（常用）
        this(classFile, 0, classFile.length);
    }

    public ClassReader(final byte[] classFileBuffer, final int classFileOffset, final int classFileLength) {
        this(classFileBuffer, classFileOffset, true);
    }

    ClassReader( // 这是最根本、最本质的构造方法
        final byte[] classFileBuffer,
        final int classFileOffset,
        final boolean checkClassVersion) {
        // ......
    }

    private static byte[] readStream(final InputStream inputStream, final boolean close) throws IOException {
        if (inputStream == null) {
            throw new IOException("Class not found");
        }
        try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
            byte[] data = new byte[INPUT_STREAM_DATA_CHUNK_SIZE];
            int bytesRead;
            while ((bytesRead = inputStream.read(data, 0, data.length)) != -1) {
                outputStream.write(data, 0, bytesRead);
            }
            outputStream.flush();
            return outputStream.toByteArray();
        } finally {
            if (close) {
                inputStream.close();
            }
        }
    }
}
```

所有构造方法，本质上都执行下面的逻辑：

```java
/**
   * Constructs a new {@link ClassReader} object. <i>This internal constructor must not be exposed
   * as a public API</i>.
   *
   * @param classFileBuffer a byte array containing the JVMS ClassFile structure to be read.
   * @param classFileOffset the offset in byteBuffer of the first byte of the ClassFile to be read.
   * @param checkClassVersion whether to check the class version or not.
   */
ClassReader(
  final byte[] classFileBuffer, final int classFileOffset, final boolean checkClassVersion) {
  this.classFileBuffer = classFileBuffer;
  this.b = classFileBuffer;
  // Check the class' major_version. This field is after the magic and minor_version fields, which
  // use 4 and 2 bytes respectively.
  if (checkClassVersion && readShort(classFileOffset + 6) > Opcodes.V20) {
    throw new IllegalArgumentException(
      "Unsupported class file major version " + readShort(classFileOffset + 6));
  }
  // Create the constant pool arrays. The constant_pool_count field is after the magic,
  // minor_version and major_version fields, which use 4, 2 and 2 bytes respectively.
  int constantPoolCount = readUnsignedShort(classFileOffset + 8);
  cpInfoOffsets = new int[constantPoolCount];
  constantUtf8Values = new String[constantPoolCount];
  // Compute the offset of each constant pool entry, as well as a conservative estimate of the
  // maximum length of the constant pool strings. The first constant pool entry is after the
  // magic, minor_version, major_version and constant_pool_count fields, which use 4, 2, 2 and 2
  // bytes respectively.
  int currentCpInfoIndex = 1;
  int currentCpInfoOffset = classFileOffset + 10;
  int currentMaxStringLength = 0;
  boolean hasBootstrapMethods = false;
  boolean hasConstantDynamic = false;
  // The offset of the other entries depend on the total size of all the previous entries.
  while (currentCpInfoIndex < constantPoolCount) {
    cpInfoOffsets[currentCpInfoIndex++] = currentCpInfoOffset + 1;
    int cpInfoSize;
    switch (classFileBuffer[currentCpInfoOffset]) {
      case Symbol.CONSTANT_FIELDREF_TAG:
      case Symbol.CONSTANT_METHODREF_TAG:
      case Symbol.CONSTANT_INTERFACE_METHODREF_TAG:
      case Symbol.CONSTANT_INTEGER_TAG:
      case Symbol.CONSTANT_FLOAT_TAG:
      case Symbol.CONSTANT_NAME_AND_TYPE_TAG:
        cpInfoSize = 5;
        break;
      case Symbol.CONSTANT_DYNAMIC_TAG:
        cpInfoSize = 5;
        hasBootstrapMethods = true;
        hasConstantDynamic = true;
        break;
      case Symbol.CONSTANT_INVOKE_DYNAMIC_TAG:
        cpInfoSize = 5;
        hasBootstrapMethods = true;
        break;
      case Symbol.CONSTANT_LONG_TAG:
      case Symbol.CONSTANT_DOUBLE_TAG:
        cpInfoSize = 9;
        currentCpInfoIndex++;
        break;
      case Symbol.CONSTANT_UTF8_TAG:
        cpInfoSize = 3 + readUnsignedShort(currentCpInfoOffset + 1);
        if (cpInfoSize > currentMaxStringLength) {
          // The size in bytes of this CONSTANT_Utf8 structure provides a conservative estimate
          // of the length in characters of the corresponding string, and is much cheaper to
          // compute than this exact length.
          currentMaxStringLength = cpInfoSize;
        }
        break;
      case Symbol.CONSTANT_METHOD_HANDLE_TAG:
        cpInfoSize = 4;
        break;
      case Symbol.CONSTANT_CLASS_TAG:
      case Symbol.CONSTANT_STRING_TAG:
      case Symbol.CONSTANT_METHOD_TYPE_TAG:
      case Symbol.CONSTANT_PACKAGE_TAG:
      case Symbol.CONSTANT_MODULE_TAG:
        cpInfoSize = 3;
        break;
      default:
        throw new IllegalArgumentException();
    }
    currentCpInfoOffset += cpInfoSize;
  }
  maxStringLength = currentMaxStringLength;
  // The Classfile's access_flags field is just after the last constant pool entry.
  header = currentCpInfoOffset;

  // Allocate the cache of ConstantDynamic values, if there is at least one.
  constantDynamicValues = hasConstantDynamic ? new ConstantDynamic[constantPoolCount] : null;

  // Read the BootstrapMethods attribute, if any (only get the offset of each method).
  bootstrapMethodOffsets =
    hasBootstrapMethods ? readBootstrapMethodsAttribute(currentMaxStringLength) : null;
}
```

上面的代码，要结合ClassFile的结构进行理解：

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

#### methods

第四个部分，`ClassReader`类定义的方法有哪些。

##### getXxx()方法

这里介绍的几个`getXxx()`方法，都是在`header`字段的基础上获得的：

```java
public class ClassReader {
  public int getAccess() {
    return readUnsignedShort(header);
  }

  public String getClassName() {
    // this_class is just after the access_flags field (using 2 bytes).
    return readClass(header + 2, new char[maxStringLength]);
  }

  public String getSuperName() {
    // super_class is after the access_flags and this_class fields (2 bytes each).
    return readClass(header + 4, new char[maxStringLength]);
  }

  public String[] getInterfaces() {
    // interfaces_count is after the access_flags, this_class and super_class fields (2 bytes each).
    int currentOffset = header + 6;
    int interfacesCount = readUnsignedShort(currentOffset);
    String[] interfaces = new String[interfacesCount];
    if (interfacesCount > 0) {
      char[] charBuffer = new char[maxStringLength];
      for (int i = 0; i < interfacesCount; ++i) {
        currentOffset += 2;
        interfaces[i] = readClass(currentOffset, charBuffer);
      }
    }
    return interfaces;
  }
}
```

同样，上面的几个`getXxx()`方法也需要参考ClassFile结构来理解：

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

假如，有如下一个类：

```java
import java.io.Serializable;

public class HelloWorld extends Exception implements Serializable, Cloneable {

}
```

输出结果：

```shell
access: 33
className: sample/HelloWorld
superName: java/lang/Exception
interfaces: [java/io/Serializable, java/lang/Cloneable]
```

##### accept()方法

在`ClassReader`类当中，有一个`accept()`方法，这个方法接收一个`ClassVisitor`类型的参数，因此`accept()`方法是将`ClassReader`和`ClassVisitor`进行连接的“桥梁”。<u>`accept()`方法的代码逻辑就是按照一定的顺序来调用`ClassVisitor`当中的`visitXxx()`方法。</u>

```java
public class ClassReader {
  // A flag to skip the Code attributes.
  public static final int SKIP_CODE = 1;

  // A flag to skip the SourceFile, SourceDebugExtension,
  // LocalVariableTable, LocalVariableTypeTable,
  // LineNumberTable and MethodParameters attributes.
  public static final int SKIP_DEBUG = 2;

  // A flag to skip the StackMap and StackMapTable attributes.
  public static final int SKIP_FRAMES = 4;

  // A flag to expand the stack map frames.
  public static final int EXPAND_FRAMES = 8;


  public void accept(final ClassVisitor classVisitor, final int parsingOptions) {
    accept(classVisitor, new Attribute[0], parsingOptions);
  }

  public void accept(
    final ClassVisitor classVisitor,
    final Attribute[] attributePrototypes,
    final int parsingOptions) {
    Context context = new Context();
    context.attributePrototypes = attributePrototypes;
    context.parsingOptions = parsingOptions;
    context.charBuffer = new char[maxStringLength];

    // Read the access_flags, this_class, super_class, interface_count and interfaces fields.
    char[] charBuffer = context.charBuffer;
    int currentOffset = header;
    int accessFlags = readUnsignedShort(currentOffset);
    String thisClass = readClass(currentOffset + 2, charBuffer);
    String superClass = readClass(currentOffset + 4, charBuffer);
    String[] interfaces = new String[readUnsignedShort(currentOffset + 6)];
    currentOffset += 8;
    for (int i = 0; i < interfaces.length; ++i) {
      interfaces[i] = readClass(currentOffset, charBuffer);
      currentOffset += 2;
    }

    // ......

    // Visit the class declaration. The minor_version and major_version fields start 6 bytes before
    // the first constant pool entry, which itself starts at cpInfoOffsets[1] - 1 (by definition).
    classVisitor.visit(readInt(cpInfoOffsets[1] - 7), accessFlags, thisClass, signature, superClass, interfaces);

    // ......

    // Visit the fields and methods.
    int fieldsCount = readUnsignedShort(currentOffset);
    currentOffset += 2;
    while (fieldsCount-- > 0) {
      currentOffset = readField(classVisitor, context, currentOffset);
    }
    int methodsCount = readUnsignedShort(currentOffset);
    currentOffset += 2;
    while (methodsCount-- > 0) {
      currentOffset = readMethod(classVisitor, context, currentOffset);
    }

    // Visit the end of the class.
    classVisitor.visitEnd();
  }

}
```

另外，我们也可以回顾一下`ClassVisitor`类中`visitXxx()`方法的调用顺序：

```shell
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

### 3.1.2 如何使用ClassReader类

The ASM core API for **generating** and **transforming** compiled Java classes is based on the `ClassVisitor` abstract class.

![Java ASM系列：（020）Cla***eader介绍_ByteCode](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在现阶段，我们接触了`ClassVisitor`、`ClassWriter`和`ClassReader`类，因此可以介绍Class Transformation的操作。

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassVisitor(api, cw) { /**/ };

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

代码的整体处理流程是如下这样的：

```shell
.class --> ClassReader --> ClassVisitor1 ... --> ClassVisitorN --> ClassWriter --> .class文件
```

我们可以将整体的处理流程想像成一条河流，那么

- 第一步，构建`ClassReader`。生成的`ClassReader`对象，它是这条“河流”的“源头”。
- 第二步，构建`ClassWriter`。生成的`ClassWriter`对象，它是这条“河流”的“归处”，它可以想像成是“百川东到海”中的“大海”。
- 第三步，串连`ClassVisitor`。生成的`ClassVisitor`对象，它是这条“河流”上的重要节点，可以想像成一个“水库”；可以有多个`ClassVisitor`对象，也就是在这条“河流”上存在多个“水库”，这些“水库”可以对“河水”进行一些处理，最终会这些“水库”的水会流向“大海”；也就是说多个`ClassVisitor`对象最终会连接到`ClassWriter`对象上。
- 第四步，结合`ClassReader`和`ClassVisitor`。在`ClassReader`类上，有一个`accept()`方法，它接收一个`ClassVisitor`类型的对象；换句话说，就是将“河流”的“源头”和后续的“水库”连接起来。
- 第五步，生成`byte[]`。到这一步，就是所有的“河水”都流入`ClassWriter`这个“大海”当中，这个时候我们调用`ClassWriter.toByteArray()`方法，就能够得到`byte[]`内容。

![Java ASM系列：（020）Cla***eader介绍_Java_02](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.1.3 parsingOptions参数

在`ClassReader`类当中，`accept()`方法接收一个`int`类型的`parsingOptions`参数。

```java
/**
   * Makes the given visitor visit the JVMS ClassFile structure passed to the constructor of this
   * {@link ClassReader}.
   *
   * @param classVisitor the visitor that must visit this class.
   * @param parsingOptions the options to use to parse this class. One or more of {@link
   *     #SKIP_CODE}, {@link #SKIP_DEBUG}, {@link #SKIP_FRAMES} or {@link #EXPAND_FRAMES}.
   */
public void accept(final ClassVisitor classVisitor, final int parsingOptions)
```

`parsingOptions`参数可以选取的值有以下5个：

- `0`
- `ClassReader.SKIP_CODE`
- `ClassReader.SKIP_DEBUG`
- `ClassReader.SKIP_FRAMES`
- `ClassReader.EXPAND_FRAMES`

推荐使用：

- 在调用`ClassReader.accept()`方法时，其中的`parsingOptions`参数，推荐使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`。
- 在创建`ClassWriter`对象时，其中的`flags`参数，推荐使用`ClassWriter.COMPUTE_FRAMES`。

示例代码如下：

```java
ClassReader cr = new ClassReader(bytes);
int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cv, parsingOptions);

ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
```

为什么我们推荐使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`呢？因为使用这样的一个值，可以生成最少的ASM代码，但是又能实现完整的功能。

- `0`：会生成所有的ASM代码，包括调试信息、frame信息和代码信息。
- `ClassReader.SKIP_CODE`：会忽略代码信息，例如，会忽略对于`MethodVisitor.visitXxxInsn()`方法的调用。
- `ClassReader.SKIP_DEBUG`：会忽略调试信息，例如，会忽略对于`MethodVisitor.visitParameter()`、`MethodVisitor.visitLineNumber()`和`MethodVisitor.visitLocalVariable()`等方法的调用。
- `ClassReader.SKIP_FRAMES`：会忽略frame信息，例如，会忽略对于`MethodVisitor.visitFrame()`方法的调用。
- `ClassReader.EXPAND_FRAMES`：会对frame信息进行扩展，例如，会对`MethodVisitor.visitFrame()`方法的参数有影响。（即会把当前Frame状态的参数更完整地填充到`visitFrame()`中）

简而言之，使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`的目的是**功能完整、代码少、复杂度低，**：

- 不使用`ClassReader.SKIP_CODE`，使代码的**功能**保持完整。
- 使用`ClassReader.SKIP_DEBUG`，减少不必要的调试信息，会使**代码量**减少。
- 使用`ClassReader.SKIP_FRAMES`，降低代码的**复杂度**。
- 不使用`ClassReader.EXPAND_FRAMES`，降低代码的**复杂度**。

对于这些参数的使用，我们可以在`ASMPrint`类的基础上进行实验。

我们使用`ClassReader.SKIP_DEBUG`的时候，就不会生成调试信息。因为这些调试信息主要是记录某一条instruction在代码当中的行数，以及变量的名字等信息；如果没有这些调试信息，也不会影响程序的正常运行，也就是说功能不受影响，因此省略这些信息，就会让ASM代码尽可能的简洁。

我们使用`ClassReader.SKIP_FRAMES`的时候，就会忽略frame的信息。为什么要忽略这些frame信息呢？因为frame计算的细节会很繁琐，需要处理的情况也有很多，总的来说，就是比较麻烦。我们解决这个麻烦的方式，就是让ASM帮助我们来计算frame的情况，也就是在创建`ClassWriter`对象的时候使用`ClassWriter.COMPUTE_FRAMES`选项。

**在刚开始学习ASM的时候，对于`parsingOptions`参数，我们推荐使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`的组合值。但是，以后，随着大家对ASM的知识越来越熟悉，或者随着功能需求的变化，大家可以尝试着使用其它的选项值。**

### 3.1.4 总结

本文主要对`ClassReader`类进行了介绍，内容总结如下：

- 第一点，了解`ClassReader`类的成员都有哪些。
- 第二点，如何使用`ClassReader`类，来进行Class Transformation的操作。
- 第三点，在`ClassReader`类当中，对于`accept()`方法的`parsingOptions`参数，我们推荐使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`。

## 3.2 ClassReader代码示例

在现阶段，我们接触了`ClassVisitor`、`ClassWriter`和`ClassReader`类，因此可以介绍Class Transformation的操作。

![Java ASM系列：（021）Cla***eader代码示例_Java](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.2.1 整体思路

对于一个`.class`文件进行Class Transformation操作，整体思路是这样的：

```
ClassReader --> ClassVisitor(1) --> ... --> ClassVisitor(N) --> ClassWriter
```

其中，

- `ClassReader`类，是ASM提供的一个类，可以直接拿来使用。
- `ClassWriter`类，是ASM提供的一个类，可以直接拿来使用。
- `ClassVisitor`类，是ASM提供的一个抽象类，因此需要写代码提供一个`ClassVisitor`的子类，在这个子类当中可以实现对`.class`文件进行各种处理操作。换句话说，**进行Class Transformation操作，编写`ClassVisitor`的子类是关键**。

### 3.2.2 修改类的信息

#### 示例一：修改类的版本

预期目标：假如有一个`HelloWorld.java`文件，经过Java 17编译之后，生成的`HelloWorld.class`文件的版本就是Java 17的版本，我们的目标是将`HelloWorld.class`由Java 17版本转换成Java 8版本。

```java
public class HelloWorld {
}
```

编码实现：

注意点：

+ 这里api指的是ASM API 版本，而version指的是Java ClassFile 版本

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.Opcodes;

/**
 * 将Class文件编译后的版本从 java17 改成 java8
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 5:42 PM
 */
public class ClassChangeVersionVisitor extends ClassVisitor {
  public ClassChangeVersionVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(Opcodes.V1_8, access, name, signature, superName, interfaces);
  }
}

```

进行转换：

```java
package com.example;

import core.ClassChangeVersionVisitor;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassChangeVersionVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}

```

验证结果：

```shell
$ javap -p -v sample.HelloWorld

# 前后对比 major version 值，61 是 java17，52是java8
# 在Opcodes接口中能看到
# int V1_8 = 0 << 16 | 52;
# int V17 = 0 << 16 | 61;
```

#### 示例二：修改类的接口

预期目标：在下面的`HelloWorld`类中，我们定义了一个`clone()`方法，但存在一个问题，也就是，如果没有实现`Cloneable`接口，`clone()`方法就会出错，我们的目标是希望通过ASM为`HelloWorld`类添加上`Cloneable`接口。

```java
public class HelloWorld {
  @Override
  public Object clone() throws CloneNotSupportedException {
    return super.clone();
  }
}
```

编码实现：

```java
package core;

import org.objectweb.asm.ClassVisitor;

/**
 * 给指定类 直接暴力添加实现 Cloneable 接口 (暂不考虑 原有类已经实现了 Cloneable 的情况)
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 6:09 PM
 */
public class ClassCloneVisitor extends ClassVisitor {
  public ClassCloneVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    // 暴力指 实现 Cloneable 接口
    super.visit(version, access, name, signature, superName, new String[] {"java/lang/Cloneable"});
  }
}
```

注意：`ClassCloneVisitor`这个类写的比较简单，直接添加`java/lang/Cloneable`接口信息；在`learn-java-asm`项目代码当中，有一个`ClassAddInterfaceVisitor`类，实现更灵活。

```java
package com.example;

import core.ClassChangeVersionVisitor;
import core.ClassCloneVisitor;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassCloneVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

验证结果：

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld obj = new HelloWorld();
    Object anotherObj = obj.clone();
    System.out.println(anotherObj);
  }
}
```

#### 小结

我们看到上面的两个例子，一个是修改类的版本信息，另一个是修改类的接口信息，那么这两个示例都是基于`ClassVisitor.visit()`方法实现的：

```java
public void visit(int version, int access, String name, String signature, String superName, String[] interfaces)
```

这两个示例，就是通过修改`visit()`方法的参数实现的：

- 修改类的版本信息，是通过修改`version`这个参数实现的
- 修改类的接口信息，是通过修改`interfaces`这个参数实现的

其实，在`visit()`方法当中的其它参数也可以修改：

- 修改`access`参数，也就是修改了类的访问标识信息。
- 修改`name`参数，也就是修改了类的名称。但是，<u>在大多数的情况下，不推荐修改`name`参数。因为调用类里的方法，都是先找到类，再找到相应的方法；如果将当前类的类名修改成别的名称，那么其它类当中可能就找不到原来的方法了，因为类名已经改了。但是，也有少数的情况，可以修改`name`参数，比如说对代码进行混淆（obfuscate）操作</u>。
- 修改`superName`参数，也就是修改了当前类的父类信息。

### 3.2.3 修改字段信息

#### 示例三：删除字段

预期目标：删除掉`HelloWorld`类里的`String strValue`字段。

```java
public class HelloWorld {
  public int intValue;
  public String strValue; // 删除这个字段
}
```

编码实现：

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;

/**
 * 判断当 字段名+字段类型 和实例传入的参数一致时，则忽略(删除)该字段
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 6:27 PM
 */
public class ClassRemoveFieldVisitor extends ClassVisitor {
    /**
     * 预删除的字段 的 字段名
     */
    private final String fieldName;
    /**
     * 预删除的字段 的 描述符(字段类型)
     */
    private final String fieldDescriptor;

    public ClassRemoveFieldVisitor(int api, ClassVisitor classVisitor, String fieldName, String fieldDescriptor) {
        super(api, classVisitor);
        this.fieldName = fieldName;
        this.fieldDescriptor = fieldDescriptor;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        // 如果和 预删除的 字段一致，则忽略(删除)该字段
        if (name.equals(fieldName) && descriptor.equals(fieldDescriptor)) {
            return null;
        }
        return super.visitField(access, name, descriptor, signature, value);
    }
}
```

**上面代码思路的关键就是`ClassVisitor.visitField()`方法。在正常的情况下，`ClassVisitor.visitField()`方法返回一个`FieldVisitor`对象；但是，如果`ClassVisitor.visitField()`方法返回的是`null`，就么能够达到删除该字段的效果**。

我们之前说过一个形象的类比，就是将`ClassReader`类比喻成河流的“源头”，而`ClassVisitor`类比喻成河流的经过的路径上的“水库”，而`ClassWriter`类则比喻成“大海”，也就是河水的最终归处。如果说，其中一个“水库”拦截了一部分水流，那么这部分水流就到不了“大海”了；这就相当于`ClassVisitor.visitField()`方法返回的是`null`，从而能够达到删除该字段的效果。

或者说，换一种类比，用信件的传递作类比。将`ClassReader`类想像成信件的“发出地”，将`ClassVisitor`类想像成信件运送途中经过的“驿站”，将`ClassWriter`类想像成信件的“接收地”；如果是在某个“驿站”中将其中一封邮件丢失了，那么这封信件就抵达不了“接收地”了。

```java
package com.example;

import core.ClassChangeVersionVisitor;
import core.ClassCloneVisitor;
import core.ClassRemoveFieldVisitor;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassRemoveFieldVisitor(api, cw, "strValue", "Ljava/lang/String;");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

验证结果：

```java
import java.lang.reflect.Field;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz.getName());

    Field[] declaredFields = clazz.getDeclaredFields();
    for (Field f : declaredFields) {
      System.out.println("    " + f.getName());
    }
  }
}
```

#### 示例四：添加字段

预期目标：为了`HelloWorld`类添加一个`Object objValue`字段。

```java
public class HelloWorld {
    public int intValue;
    public String strValue;
    // 添加一个Object objValue字段
}
```

编码实现：

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;

/**
 * 和删除字段不同的是，这里需要在 visitEnd里面再添加字段 <br/>
 * 因为遍历现有.class时，visitField需要遍历所有字段，会调用多次，而visitEnd只会在最后调用一次 <br/>
 * 在只调用一次的 visitEnd 中实际添加字段，保证只会添加一次字段
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 6:38 PM
 */
public class ClassAddFieldVisitor extends ClassVisitor {
  // 预添加的 字段的 访问修饰符，字段名，字段描述符
  private final int fieldAccess;
  private final String fieldName;
  private final String fieldDescriptor;
  // 判断是否已经有该字段(避免重复添加字段)
  private boolean isFieldPresent;

  public ClassAddFieldVisitor(int api, ClassVisitor classVisitor, int fieldAccess, String fieldName, String fieldDescriptor) {
    super(api, classVisitor);
    this.fieldAccess = fieldAccess;
    this.fieldName = fieldName;
    this.fieldDescriptor = fieldDescriptor;
  }

  @Override
  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    // java类中不允许有同名字段，所以仅判断 name 就够了
    if (name.equals(fieldName)) {
      isFieldPresent = true;
    }
    return super.visitField(access, name, descriptor, signature, value);
  }

  @Override
  public void visitEnd() {
    // 如果java类中没有 预添加的字段，则添加
    if (!isFieldPresent) {
      FieldVisitor fieldVisitor = super.visitField(fieldAccess, fieldName, fieldDescriptor, null, null);
      // 记得调用 visitEnd (因为后续的ClassVisitor也可能在 visitEnd 里写了另外的处理逻辑)
      if(null != fieldVisitor) {
        fieldVisitor.visitEnd();
      }
    }
    super.visitEnd();
  }
}
```

上面的代码思路：第一步，在`visitField()`方法中，判断某个字段是否已经存在，其结果存在于`isFieldPresent`字段当中；第二步，就是在`visitEnd()`方法中，根据`isFieldPresent`字段的值，来决定是否添加新的字段。

```java
package com.example;

import core.ClassAddFieldVisitor;
import core.ClassChangeVersionVisitor;
import core.ClassCloneVisitor;
import core.ClassRemoveFieldVisitor;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassAddFieldVisitor(api, cw, Opcodes.ACC_PUBLIC,"objValue", "Ljava/lang/Object;");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

验证结果：

```java
import java.lang.reflect.Field;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz.getName());

    Field[] declaredFields = clazz.getDeclaredFields();
    for (Field f : declaredFields) {
      System.out.println("    " + f.getName());
    }
  }
}
```

#### 小结

对于字段的操作，都是基于`ClassVisitor.visitField()`方法来实现的：

```java
public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value);
```

那么，对于字段来说，可以进行哪些操作呢？有三种类型的操作：

- 修改现有的字段。例如，修改字段的名字、修改字段的类型、修改字段的访问标识，这些需要通过修改`visitField()`方法的参数来实现。
- **删除已有的字段。在`visitField()`方法中，返回`null`值，就能够达到删除字段的效果**。
- 添加新的字段。在`visitField()`方法中，判断该字段是否已经存在；在`visitEnd()`方法中，如果该字段不存在，则添加新字段。

**一般情况下来说，不推荐“修改已有的字段”，也不推荐“删除已有的字段”**，原因如下：

- 不推荐“修改已有的字段”，因为这可能会引起字段的名字不匹配、字段的类型不匹配，从而导致程序报错。例如，假如在`HelloWorld`类里有一个`intValue`字段，而且`GoodChild`类里也使用到了`HelloWorld`类的这个`intValue`字段；如果我们将`HelloWorld`类里的`intValue`字段名字修改为`myValue`，那么`GoodChild`类就再也找不到`intValue`字段了，这个时候，程序就会出错。当然，如果我们把`GoodChild`类里对于`intValue`字段的引用修改成`myValue`，那也不会出错了。但是，我们要保证所有使用`intValue`字段的地方，都要进行修改，这样才能让程序不报错。
- 不推荐“删除已有的字段”，因为一般来说，类里的字段都是有作用的，如果随意的删除就会造成字段缺失，也会导致程序报错。

**为什么不在`ClassVisitor.visitField()`方法当中来添加字段呢？如果在`ClassVisitor.visitField()`方法，就可能添加重复的字段，这样就不是一个合法的ClassFile了**。

### 3.2.4 修改方法信息

#### 示例五：删除方法

预期目标：删除掉`HelloWorld`类里的`add()`方法。

```java
public class HelloWorld {
  public int add(int a, int b) { // 删除add方法
    return a + b;
  }

  public int sub(int a, int b) {
    return a - b;
  }
}
```

编码实现：

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

/**
 * 若指定的 方法名 和 描述符 和现有的方法一致，则忽略(删除)该方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 7:31 PM
 */
public class ClassRemoveMethodVisitor extends ClassVisitor {
  private final String methodName;
  private final String methodDescriptor;

  public ClassRemoveMethodVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDescriptor) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDescriptor = methodDescriptor;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    if (name.equals(methodName) && descriptor.equals(methodDescriptor)) {
      return null;
    }
    return super.visitMethod(access, name, descriptor, signature, exceptions);
  }
}
```

上面删除方法的代码思路，与删除字段的代码思路是一样的。

```java
package com.example;

import core.*;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassRemoveMethodVisitor(api,cw,"add", "(II)I");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

验证结果：

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz.getName());

    Method[] declaredMethods = clazz.getDeclaredMethods();
    for (Method m : declaredMethods) {
      System.out.println("    " + m.getName());
    }
  }
}
```

#### 示例六：添加方法

预期目标：为`HelloWorld`类添加一个`mul()`方法。

```java
public class HelloWorld {
  public int add(int a, int b) {
    return a + b;
  }

  public int sub(int a, int b) {
    return a - b;
  }

  // TODO: 添加一个乘法
}
```

编码实现：

注意点：

+ 和添加字段不同的是，添加方法，还需要填充方法体逻辑

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

/**
 * 如果 指定的 类中不存在 指定的mul方法，则添加 <br/>
 * 与添加 字段不同的是，添加方法，还需要实现方法体内的逻辑
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/23 7:39 PM
 */
public abstract class ClassAddMethodVisitor extends ClassVisitor {
  private final int methodAccess;
  private final String methodName;
  private final String methodDescriptor;
  private final String methodSignature;
  private final String[] methodExceptions;
  private boolean isMethodPresent;

  protected ClassAddMethodVisitor(int api, ClassVisitor classVisitor, int methodAccess, String methodName, String methodDescriptor, String methodSignature, String[] methodExceptions) {
    super(api, classVisitor);
    this.methodAccess = methodAccess;
    this.methodName = methodName;
    this.methodDescriptor = methodDescriptor;
    this.methodSignature = methodSignature;
    this.methodExceptions = methodExceptions;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    if(name.equals(methodName) && descriptor.equals(methodDescriptor)){
      isMethodPresent = true;
    }
    return super.visitMethod(access, name, descriptor, signature, exceptions);
  }

  @Override
  public void visitEnd() {
    if(!isMethodPresent) {
      MethodVisitor methodVisitor = super.visitMethod(methodAccess, methodName, methodDescriptor, methodSignature, methodExceptions);
      if(null != methodVisitor) {
        // 填充方法体逻辑
        generateMethodBody(methodVisitor);
      }
    }
    super.visitEnd();
  }

  protected abstract void generateMethodBody(MethodVisitor mv);
}

```

<u>添加新的方法，和添加新的字段的思路，在前期，两者是一样的，都是先要判断该字段或该方法是否已经存在；但是，在后期，两者会有一些差异，因为方法需要有“方法体”，在上面的代码中，我们定义了一个`generateMethodBody()`方法，这个方法需要在子类当中进行实现。</u>

```java
package com.example;

import core.ClassAddMethodVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassAddMethodVisitor(api, cw, Opcodes.ACC_PUBLIC, "mul", "(II)I", null, null) {
      @Override
      protected void generateMethodBody(MethodVisitor mv) {
        mv.visitCode();
        mv.visitVarInsn(Opcodes.ILOAD,1);
        mv.visitVarInsn(Opcodes.ILOAD,2);
        mv.visitInsn(Opcodes.IMUL);
        mv.visitInsn(Opcodes.IRETURN);
        mv.visitMaxs(2,3);
        mv.visitEnd();
      }
    };

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}

```

验证结果：

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz.getName());

    Method[] declaredMethods = clazz.getDeclaredMethods();
    for (Method m : declaredMethods) {
      System.out.println("    " + m.getName());
    }

    Object obj = clazz.newInstance();
    Method mul = clazz.getDeclaredMethod("mul", int.class, int.class);
    System.out.println(mul.invoke(obj, 2, 5));
  }
}
```

#### 小结

对于方法的操作，都是基于`ClassVisitor.visitMethod()`方法来实现的：

```java
public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions);
```

与字段操作类似，对于方法来说，可以进行的操作也有三种类型：

- 修改现有的方法。
- 删除已有的方法。
- 添加新的方法。

**我们不推荐“删除已有的方法”，因为这可能会引起方法调用失败，从而导致程序报错。**

<u>另外，对于“修改现有的方法”，我们不建议修改方法的名称、方法的类型（接收参数的类型和返回值的类型），因为别的地方可能会对该方法进行调用，修改了方法名或方法的类型，就会使方法调用失败。但是，我们可以“修改现有方法”的“方法体”，也就是方法的具体实现代码。</u>

### 3.2.5 总结

本文主要是使用`ClassReader`类进行Class Transformation的代码示例进行介绍，内容总结如下：

- 第一点，类层面的信息，例如，类名、父类、接口等，可以通过`ClassVisitor.visit()`方法进行修改。
- 第二点，字段层面的信息，例如，添加新字段、删除已有字段等，可能通过`ClassVisitor.visitField()`方法进行修改。
- 第三点，方法层面的信息，例如，添加新方法、删除已有方法等，可以通过`ClassVisitor.visitMethod()`方法进行修改。

但是，对于方法层面来说，还有一个重要的方面没有涉及，也就是对于现有方法里面的代码进行修改，我们在后续内容中会有介绍。

## 3.3 Class Transformation的原理

我们使用`ClassReader`、`ClassVisitor`和`ClassWriter`类来进行Class Transformation操作的整体思路是这样的：

```pseudocode
ClassReader --> ClassVisitor(1) --> ... --> ClassVisitor(N) --> ClassWriter
```

其中，`ClassReader`类负责“读”Class，`ClassWriter`负责“写”Class，而`ClassVisitor`则负责进行“转换”（Transformation）。在Class Transformation过程中，可以有多个`ClassVisitor`参与。不过要注意，`ClassVisitor`类是一个抽象类，我们需要写代码来实现一个`ClassVisitor`类的子类才能使用。

![Java ASM系列：（022）Class Transformation的原理_Java](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

为了解释清楚Class Transformation是如何工作的，我们从两个问题来入手：

- 第一个问题，在编写代码的时候（程序未运行），`ClassReader`、`ClassVisitor`和`ClassWriter`三个类的实例之间，是如何建立联系的？
- 第二个问题，在执行代码的时候（程序开始运行），类内部`visitXxx()`方法的调用顺序是什么样的？

### 3.3.1 Class-Reader/Visitor/Writer

#### 建立联系

我们在进行Class Transformation操作时，一般是这样写代码的：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv1 = new ClassVisitor1(api, cw);
    ClassVisitor cv2 = new ClassVisitor2(api, cv1);
    // ...
    ClassVisitor cvn = new ClassVisitorN(api, cvm);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cvn, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

##### ClassReader-ClassVisitor

第一步，将`ClassReader`类与`ClassVisitor`类建立联系是通过`ClassReader.accept()`方法实现的。

```java
public void accept(ClassVisitor classVisitor, int parsingOptions)
```

**在`accept()`方法中，`ClassReader`类会不断调用`ClassVisitor`类当中的`visitXxx()`方法**。

```java
public void accept(ClassVisitor classVisitor, int parsingOptions) {
  //......
  classVisitor.visit(readInt(cpInfoOffsets[1] - 7), accessFlags, thisClass, signature, superClass, interfaces);
  //...

  // Visit the fields and methods.
  int fieldsCount = readUnsignedShort(currentOffset);
  currentOffset += 2;
  while (fieldsCount-- > 0) {
    currentOffset = readField(classVisitor, context, currentOffset);
  }

  int methodsCount = readUnsignedShort(currentOffset);
  currentOffset += 2;
  while (methodsCount-- > 0) {
    currentOffset = readMethod(classVisitor, context, currentOffset);
  }

  // Visit the end of the class.
  classVisitor.visitEnd();
}
```

##### ClassVisitor-ClassVisitor

第二步，就是将一个`ClassVisitor`类与另一个`ClassVisitor`类建立联系。因为在进行Class Transformation的过程中，可能需要多个`ClassVisitor`类的参与。

**当前`ClassVisitor`类与下一个`ClassVisitor`类建立初步联系（或外部联系，指一个类与另一个类之间的联系），是通过构造方法来实现的**：

```java
public abstract class ClassVisitor {
  protected final int api;
  protected ClassVisitor cv;

  public ClassVisitor(final int api, final ClassVisitor classVisitor) {
    this.api = api;
    this.cv = classVisitor;
  }
}
```

**当前`ClassVisitor`类与下一个`ClassVisitor`类建立后续联系（或内部联系，指字段、方法等层面的联系），是通过调用`visitXxx()`方法来实现的**：

```java
public abstract class ClassVisitor {
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    if (cv != null) {
      cv.visit(version, access, name, signature, superName, interfaces);
    }
  }

  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    if (cv != null) {
      return cv.visitField(access, name, descriptor, signature, value);
    }
    return null;
  }

  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    if (cv != null) {
      return cv.visitMethod(access, name, descriptor, signature, exceptions);
    }
    return null;
  }

  public void visitEnd() {
    if (cv != null) {
      cv.visitEnd();
    }
  }
}
```

##### ClassVisitor-ClassWriter

第三步，就是将一个`ClassVisitor`类与一个`ClassWriter`类建立联系。由于`ClassWriter`类是继承自`ClassVisitor`类，所以第三步的原理和第二步的原理是一样的。不过，**`ClassWriter`类是一个特殊的`ClassVisitor`类，它的主要作用是得到一个符合ClassFile结构的字节数组（`byte[]`）**。

#### 执行顺序

对于Class Transformation来说，它具体的代码执行顺序如下图所示：

![Java ASM系列：（022）Class Transformation的原理_ASM_02](https://s2.51cto.com/images/20210628/1624882000742512.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

为了方便理解代码的执行顺序，我们也结合生活当中的一些经验，来进行这样的类比：在生活当中，天下降下了雨水，这些雨水会慢慢的向下渗透，穿过不同的地层，最后形成地下水；这种“雨水向下渗透，穿过不同地层”的形式，它与多个`ClassVisitor`对象调用同名的`visitXxx()`方法是非常相似的。

![Java ASM系列：（022）Class Transformation的原理_ClassFile_03](https://s2.51cto.com/images/20210628/1624882032736187.jpg?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

- 第一步，可以将`ClassReader`类想像成最上面的土层
- 第二步，可以将`ClassWriter`类想像成最下面的土层
- 第三步，可以将`ClassVisitor`类想像成中间的多个土层

当调用`visitXxx()`方法的时，就像水在多个土层之间渗透：由最上面的`ClassReader`“土层”开始，经历一个一个的`ClassVisitor`中间“土层”，最后进入最下面的`ClassWriter`“土层”。

#### 执行顺序的代码演示

为了查看代码的执行顺序，我们可以添加一个自定义的`ClassVisitor`类。在这个类当中，我们可以添加一些打印语句，将类名、方法名以及方法的接收的参数都打印出来，这样我们就能知道代码的执行过程了。

首先，我们来定义一个`InfoClassVisitor`类，它继承自`ClassVisitor`；在`InfoClassVisitor`类当中，我们只打印了`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`这4个方法的信息。

- 在`visitField()`方法中，我们自定义了一个`InfoFieldVisitor`对象
- 在`visitMethod()`方法中，我们也自定义了一个`InfoMethodVisitor`对象

另外，在`InfoClassVisitor`类当中，也定义了一个`getAccess()`方法，为了简便，我们只判断了`public`、`protected`和`private`标识符。大家可以根据自己的兴趣，来对这个类进行扩展。

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class InfoClassVisitor extends ClassVisitor {
  public InfoClassVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    String line = String.format("ClassVisitor.visit(%d, %s, %s, %s, %s, %s);", 
                                version, getAccess(access), name, signature, superName, Arrays.toString(interfaces));
    System.out.println(line);
    super.visit(version, access, name, signature, superName, interfaces);
  }

  @Override
  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    String line = String.format("ClassVisitor.visitField(%s, %s, %s, %s, %s);", 
                                getAccess(access), name, descriptor, signature, value);
    System.out.println(line);

    FieldVisitor fv = super.visitField(access, name, descriptor, signature, value);
    return new InfoFieldVisitor(api, fv);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    String line = String.format("ClassVisitor.visitMethod(%s, %s, %s, %s, %s);",
                                getAccess(access), name, descriptor, signature, exceptions);
    System.out.println(line);

    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    return new InfoMethodVisitor(api, mv);
  }

  @Override
  public void visitEnd() {
    String line = String.format("ClassVisitor.visitEnd();");
    System.out.println(line);
    super.visitEnd();
  }

  private String getAccess(int access) {
    List<String> list = new ArrayList<>();
    if ((access & Opcodes.ACC_PUBLIC) != 0) {
      list.add("ACC_PUBLIC");
    }
    else if ((access & Opcodes.ACC_PROTECTED) != 0) {
      list.add("ACC_PROTECTED");
    }
    else if ((access & Opcodes.ACC_PRIVATE) != 0) {
      list.add("ACC_PRIVATE");
    }
    return list.toString();
  }
}
```

接着，是`InfoFieldVisitor`类，它继承自`FieldVisitor`类。这个类很简单，它只打印其中的`visitEnd()`方法。

```java
import org.objectweb.asm.FieldVisitor;

public class InfoFieldVisitor extends FieldVisitor {
  public InfoFieldVisitor(int api, FieldVisitor fieldVisitor) {
    super(api, fieldVisitor);
  }

  @Override
  public void visitEnd() {
    String line = String.format("    FieldVisitor.visitEnd();");
    System.out.println(line);
    super.visitEnd();
  }
}
```

再接下来，是`InfoMethodVisitor`类，它继承自`MethodVisitor`类。在这个类当中，需要打印的方法比较多，但是这些方法分为4个类型：

- `visitCode()`
- `visitXxxInsn()`
- `visitMaxs()`
- `visitEnd()`

```java
import org.objectweb.asm.Label;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.util.Printer;

public class InfoMethodVisitor extends MethodVisitor {
  public InfoMethodVisitor(int api, MethodVisitor methodVisitor) {
    super(api, methodVisitor);
  }

  @Override
  public void visitCode() {
    String line = String.format("    MethodVisitor.visitCode();");
    System.out.println(line);
    super.visitCode();
  }

  @Override
  public void visitInsn(int opcode) {
    String line = String.format("    MethodVisitor.visitInsn(%s);", Printer.OPCODES[opcode]);
    System.out.println(line);
    super.visitInsn(opcode);
  }

  @Override
  public void visitIntInsn(int opcode, int operand) {
    String line = String.format("    MethodVisitor.visitIntInsn(%s, %s);", Printer.OPCODES[opcode], operand);
    System.out.println(line);
    super.visitIntInsn(opcode, operand);
  }

  @Override
  public void visitVarInsn(int opcode, int var) {
    String line = String.format("    MethodVisitor.visitVarInsn(%s, %s);", Printer.OPCODES[opcode], var);
    System.out.println(line);
    super.visitVarInsn(opcode, var);
  }

  @Override
  public void visitTypeInsn(int opcode, String type) {
    String line = String.format("    MethodVisitor.visitTypeInsn(%s, %s);", Printer.OPCODES[opcode], type);
    System.out.println(line);
    super.visitTypeInsn(opcode, type);
  }

  @Override
  public void visitFieldInsn(int opcode, String owner, String name, String descriptor) {
    String line = String.format("    MethodVisitor.visitFieldInsn(%s, %s, %s, %s);",
                                Printer.OPCODES[opcode], owner, name, descriptor);
    System.out.println(line);
    super.visitFieldInsn(opcode, owner, name, descriptor);
  }

  @Override
  public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
    String line = String.format("    MethodVisitor.visitMethodInsn(%s, %s, %s, %s, %s);",
                                Printer.OPCODES[opcode], owner, name, descriptor, isInterface);
    System.out.println(line);
    super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
  }

  @Override
  public void visitJumpInsn(int opcode, Label label) {
    String line = String.format("    MethodVisitor.visitJumpInsn(%s, %s);", Printer.OPCODES[opcode], label);
    System.out.println(line);
    super.visitJumpInsn(opcode, label);
  }

  @Override
  public void visitLabel(Label label) {
    String line = String.format("    MethodVisitor.visitLabel(%s);", label);
    System.out.println(line);
    super.visitLabel(label);
  }

  @Override
  public void visitLdcInsn(Object value) {
    String line = String.format("    MethodVisitor.visitLdcInsn(%s);", value);
    System.out.println(line);
    super.visitLdcInsn(value);
  }

  @Override
  public void visitIincInsn(int var, int increment) {
    String line = String.format("    MethodVisitor.visitIincInsn(%s, %s);", var, increment);
    System.out.println(line);
    super.visitIincInsn(var, increment);
  }

  @Override
  public void visitMaxs(int maxStack, int maxLocals) {
    String line = String.format("    MethodVisitor.visitMaxs(%s, %s);", maxStack, maxLocals);
    System.out.println(line);
    super.visitMaxs(maxStack, maxLocals);
  }

  @Override
  public void visitEnd() {
    String line = String.format("    MethodVisitor.visitEnd();");
    System.out.println(line);
    super.visitEnd();
  }
}
```

现在，准备工作已经做好了，我们只需要将自定义的`InfoClassVisitor`类加入到Class Transformation的过程中就可以了：

```java
import lsieun.core.info.InfoClassVisitor;
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new InfoClassVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

准备一个示例代码：

```java
public class HelloWorld {
  public int intValue;
  public String strValue;

  public int add(int a, int b) {
    return a + b;
  }

  public int sub(int a, int b) {
    return a - b;
  }
}
```

输出结果：

```shell
ClassVisitor.visit(52, [ACC_PUBLIC], sample/HelloWorld, null, java/lang/Object, []);
ClassVisitor.visitField([ACC_PUBLIC], intValue, I, null, null);
    FieldVisitor.visitEnd();
ClassVisitor.visitField([ACC_PUBLIC], strValue, Ljava/lang/String;, null, null);
    FieldVisitor.visitEnd();
ClassVisitor.visitMethod([ACC_PUBLIC], <init>, ()V, null, null);
    MethodVisitor.visitCode();
    MethodVisitor.visitVarInsn(ALOAD, 0);
    MethodVisitor.visitMethodInsn(INVOKESPECIAL, java/lang/Object, <init>, ()V, false);
    MethodVisitor.visitInsn(RETURN);
    MethodVisitor.visitMaxs(1, 1);
    MethodVisitor.visitEnd();
ClassVisitor.visitMethod([ACC_PUBLIC], add, (II)I, null, null);
    MethodVisitor.visitCode();
    MethodVisitor.visitVarInsn(ILOAD, 1);
    MethodVisitor.visitVarInsn(ILOAD, 2);
    MethodVisitor.visitInsn(IADD);
    MethodVisitor.visitInsn(IRETURN);
    MethodVisitor.visitMaxs(2, 3);
    MethodVisitor.visitEnd();
ClassVisitor.visitMethod([ACC_PUBLIC], sub, (II)I, null, null);
    MethodVisitor.visitCode();
    MethodVisitor.visitVarInsn(ILOAD, 1);
    MethodVisitor.visitVarInsn(ILOAD, 2);
    MethodVisitor.visitInsn(ISUB);
    MethodVisitor.visitInsn(IRETURN);
    MethodVisitor.visitMaxs(2, 3);
    MethodVisitor.visitEnd();
ClassVisitor.visitEnd();
```

### 3.3.2 串联的Field/MethodVisitors

经过上面内容的讲解，相信大家已经了解到多个`ClassVisitor`之间是相互连接的，或者说是串联到一起的。

![Java ASM系列：（022）Class Transformation的原理_ClassFile_04](https://s2.51cto.com/images/20210628/1624882119265917.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

其实，还有一些“细微之处”的连接，我们也要注意到：不同`ClassVisitor`对象里，对应同一个字段的多个`FieldVisitor`对象，也是串联到一起；不同`ClassVisitor`对象里，对应同一个方法的多个`MethodVisitor`对象，也是串联到一起，如下图所示。

![多个FieldVisitor和MethodVisitor串联到一起](https://lsieun.github.io/assets/images/java/asm/multiple-field-method-vistors-connected.png)

<u>我们在讲删除“字段”和删除“方法”的时候，就是其中的某一个`FieldVisitor`或`MethodVisitor`不工作了，也就是不向后“传递数据”了，那么，相应的“字段”和“方法”就丢失了，就达到了“删除”的效果</u>。类似的，添加“字段”和“方法”，其实就是“传递了额外的数据”，那么就会出现新的字段和方法，就达到了添加字段和方法的效果。

### 3.3.3 Class Transformation的本质

对于Class Transformation来说，它的本质就是“中间人攻击”（Man-in-the-middle attack）。

在[Wiki](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)当中，是这样描述Man-in-the-middle attack的：

> In cryptography and computer security, a man-in-the-middle(MITM) attack is a cyberattack where the attacker secretly relays and possibly alters the communications between two parties who believe that they are directly communicating with each other.

![Java ASM系列：（022）Class Transformation的原理_Java_06](https://s2.51cto.com/images/20210628/1624882180784922.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在计算机安全领域，我们应该尽量的避免遭受到“中间人攻击”，这样我们的信息就不会被窃取和篡改。但是，在Java ASM当中，Class Transformation的本质就是利用了“中间人攻击”的方式来实现对已有的Class文件进行修改或转换。

更详细的来说，我们自己定义的`ClassVisitor`类就是一个“中间人”，那么这个“中间人”可以做什么呢？可以做三种类型的事情：

- 对“原有的信息”进行篡改，就可以实现“修改”的效果。对应到ASM代码层面，就是对`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`的参数值进行修改。
- 对“原有的信息”进行扔掉，就可以实现“删除”的效果。对应到ASM代码层面，将原本的`FieldVisitor`和`MethodVisitor`对象实例替换成`null`值，或者对原本的一些`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法不去调用了。
- 伪造一条“新的信息”，就可以实现“添加”的效果。对应到ASM代码层面，就是在原来的基础上，添加一些对于`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法的调用。

### 3.3.4 总结

本文主要对Class Transformation的工作原理进行介绍，内容总结如下：

- 第一点，在代码开始执行之前，`ClassReader`、`ClassVisitor`和`ClassWriter`这三者之间是如何建立最初的联系的。例如，`ClassReader`与`ClassVisitor`建立联系是通过`ClassReader.accept()`方法来实现的；而`ClassVisitor`与`ClassWriter`建立联系是通过`ClassVisitor`的构造方法来建立联系的。
- 第二点，在代码的执行过程中，其中涉及到`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法的执行顺序是什么样的。
- 第三点，在进行Class Transformation的过程中，内在的多个FieldVisitor和MethodVisitor是串联到一起的。
- 第四点，Class Transformation的本质就是“中间人攻击”（Man-in-the-middle attack）。

## 3.4 Type介绍

### 3.4.1 为什么会存在Type类

> [Java Language and Virtual Machine Specifications](https://docs.oracle.com/javase/specs/)
>
> [Type 类型 泛型 反射 Class ParameterizedType](https://www.cnblogs.com/baiqiantao/p/7460580.html)

在ASM的代码中，有一个`Type`类（`org.objectweb.asm.Type`）。为什么会有这样一个`Type`类呢？

大家知道，在JDK当中有一个`java.lang.reflect.Type`类。对于`java.lang.reflect.Type`类来说，它是一个接口，它有一个我们经常使用的子类，即`java.lang.Class`；相应的，在ASM当中有一个`org.objectweb.asm.Type`类。

|      | JDK                      | ASM                      |
| ---- | ------------------------ | ------------------------ |
| 类名 | `java.lang.reflect.Type` | `org.objectweb.asm.Type` |
| 位置 | `rt.jar`                 | `asm.jar`                |

```java
package java.lang.reflect;

/**
 * Type is the common superinterface for all types in the Java
 * programming language. These include raw types, parameterized types,
 * array types, type variables and primitive types.
 *
 * @jls 4.1 The Kinds of Types and Values
 * @jls 4.2 Primitive Types and Values
 * @jls 4.3 Reference Types and Values
 * @jls 4.4 Type Variables
 * @jls 4.5 Parameterized Types
 * @jls 4.8 Raw Types
 * @jls 4.9 Intersection Types
 * @jls 10.1 Array Types
 * @since 1.5
 */
public interface Type {
    /**
     * Returns a string describing this type, including information
     * about any type parameters.
     *
     * @implSpec The default implementation calls {@code toString}.
     *
     * @return a string describing this type
     * @since 1.8
     */
    default String getTypeName() {
        return toString();
    }
}
```

```java
package org.objectweb.asm;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

/**
 * A Java field or method type. This class can be used to make it easier to manipulate type and
 * method descriptors.
 *
 * @author Eric Bruneton
 * @author Chris Nokleberg
 */
public final class Type {
  // ...
}
```

在编写代码层面，如果我们不能区分出`java.lang.reflect.Type`类和`org.objectweb.asm.Type`类，我们也不能很好的使用它们。

- **Java File：具体表现为`.java`文件，在里面使用Java语言编写代码，它是属于Java Language Specification的范畴。**
- **Class File：具体表现为`.class`文件，它里面的内容遵循ClassFile的结构，它是属于JVM Specification的范畴**。
- ASM：它是一个类库。我们在编写ASM代码的时候，是在`.java`文件中编写，使用的是Java语言，而它所操作的对象却是`.class`文件。

换句话说，ASM实现，从本质上来说，是一只脚踩在Java Language Specification的范畴，而另一只脚却踩在JVM Specification的范畴。ASM，在这两个范畴中，扮演的一个非常重要的角色，就是将Java Language Specification范畴的概念和JVM Specification范畴的概念进行转换。

![Java ASM系列：（023）Type介绍_Core API](https://s2.51cto.com/images/20210628/1624892139273785.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

这两个范畴，是相关的，但是又不是那种密不可分的关系。比如说，Java语言编写的程序可以运行在JVM上，Scala语言编写的程序也可以运行在JVM上，甚至Python语言编写的程序也可以编写在JVM上；也就是说，某一种编程语言和JVM之间，并不是一种非常强的依赖关系。

![Java ASM系列：（023）Type介绍_Core API_02](https://s2.51cto.com/images/20210628/1624892168978847.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

| Java Language Specification | ASM                                    | JVM Specification  |
| --------------------------- | -------------------------------------- | ------------------ |
| `int`                       | <--- 向左转换 ---Type--- 向右转换 ---> | `I`                |
| `float`                     | <--- 向左转换 ---Type--- 向右转换 ---> | `F`                |
| `java.lang.String`          | <--- 向左转换 ---Type--- 向右转换 ---> | `java/lang/String` |

<u>在`.java`文件中，我们经常使用`java.lang.Class`类；而在`.class`文件中，需要经常用到internal name、type descriptor和method descriptor</u>；而在ASM中，`org.objectweb.asm.Type`类就是帮助我们进行两者之间的转换。

![Java ASM系列：（023）Type介绍_ClassFile_03](https://s2.51cto.com/images/20210628/1624892335847287.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.4.2 Type类

#### class info

第一个部分，`Type`类继承自`Object`类，而且带有`final`标识，所以不会存在子类。

```java
public final class Type {
}
```

#### fields

第二个部分，`Type`类定义的字段有哪些。这里我们列出了4个字段，这4个字段可以分成两组。

- 第一组，只包括`sort`字段，是`int`类型，它标识了`Type`类的类别。
- 第二组，包括`valueBuffer`、`valueBegin`和`valueEnd`字段，这3个字段组合到一起表示一个value值，本质上就是一个字符串。

```java
public final class Type {
    // 标识类型
  /**
   * The sort of this type. Either {@link #VOID}, {@link #BOOLEAN}, {@link #CHAR}, {@link #BYTE},
   * {@link #SHORT}, {@link #INT}, {@link #FLOAT}, {@link #LONG}, {@link #DOUBLE}, {@link #ARRAY},
   * {@link #OBJECT}, {@link #METHOD} or {@link #INTERNAL}.
   */
    private final int sort;

    // 标识内容
  /**
   * A buffer containing the value of this field or method type. This value is an internal name for
   * {@link #OBJECT} and {@link #INTERNAL} types, and a field or method descriptor in the other
   * cases.
   *
   * <p>For {@link #OBJECT} types, this field also contains the descriptor: the characters in
   * [{@link #valueBegin},{@link #valueEnd}) contain the internal name, and those in [{@link
   * #valueBegin} - 1, {@link #valueEnd} + 1) contain the descriptor.
   */
    private final String valueBuffer;
    private final int valueBegin;
    private final int valueEnd;
}
```

#### constructors

第三个部分，`Type`类定义的构造方法有哪些。由于`Type`类的构造方法用`private`修饰，因此“外界”不能使用`new`关键字创建`Type`对象。

```java
/**
   * Constructs a reference type.
   *
   * @param sort the sort of this type, see {@link #sort}.
   * @param valueBuffer a buffer containing the value of this field or method type.
   * @param valueBegin the beginning index, inclusive, of the value of this field or method type in
   *     valueBuffer.
   * @param valueEnd the end index, exclusive, of the value of this field or method type in
   *     valueBuffer.
   */
public final class Type {
    private Type(final int sort, final String valueBuffer, final int valueBegin, final int valueEnd) {
        this.sort = sort;
        this.valueBuffer = valueBuffer;
        this.valueBegin = valueBegin;
        this.valueEnd = valueEnd;
    }
}
```

#### methods

第四个部分，`Type`类定义的方法有哪些。在`Type`类里，定义了一些方法，这些方法是与字段有直接关系的。

```java
public final class Type {
    public int getSort() {
        return sort == INTERNAL ? OBJECT : sort;
    }

    public String getClassName() {
        switch (sort) {
            case VOID:
                return "void";
            case BOOLEAN:
                return "boolean";
            case CHAR:
                return "char";
            case BYTE:
                return "byte";
            case SHORT:
                return "short";
            case INT:
                return "int";
            case FLOAT:
                return "float";
            case LONG:
                return "long";
            case DOUBLE:
                return "double";
            case ARRAY:
                StringBuilder stringBuilder = new StringBuilder(getElementType().getClassName());
                for (int i = getDimensions(); i > 0; --i) {
                    stringBuilder.append("[]");
                }
                return stringBuilder.toString();
            case OBJECT:
            case INTERNAL:
                return valueBuffer.substring(valueBegin, valueEnd).replace('/', '.');
            default:
                throw new AssertionError();
        }
    }

    public String getInternalName() {
        return valueBuffer.substring(valueBegin, valueEnd);
    }

    public String getDescriptor() {
        if (sort == OBJECT) {
            return valueBuffer.substring(valueBegin - 1, valueEnd + 1);
        } else if (sort == INTERNAL) {
            return 'L' + valueBuffer.substring(valueBegin, valueEnd) + ';';
        } else {
            return valueBuffer.substring(valueBegin, valueEnd);
        }
    }
}
```

关于这些方法的使用，示例如下：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Type t = Type.getType("Ljava/lang/String;");

        int sort = t.getSort();                    // ASM
        String className = t.getClassName();       // Java File
        String internalName = t.getInternalName(); // Class File
        String descriptor = t.getDescriptor();     // Class File

        System.out.println(sort);         // 10，它对应于Type.OBJECT字段
        System.out.println(className);    // java.lang.String   注意，分隔符是“.”
        System.out.println(internalName); // java/lang/String   注意，分隔符是“/”
        System.out.println(descriptor);   // Ljava/lang/String; 注意，分隔符是“/”，前有“L”，后有“;”
    }
}
```

### 3.4.3 静态成员

#### 静态字段

在`Type`类里，定义了一些常量字段，有`int`类型，也有`String`类型。

```java
public final class Type {
    public static final int VOID = 0;
    public static final int BOOLEAN = 1;
    public static final int CHAR = 2;
    public static final int BYTE = 3;
    public static final int SHORT = 4;
    public static final int INT = 5;
    public static final int FLOAT = 6;
    public static final int LONG = 7;
    public static final int DOUBLE = 8;
    public static final int ARRAY = 9;
    public static final int OBJECT = 10;
  	/** The sort of method types. See {@link #getSort}. */
    public static final int METHOD = 11;
  	/** The (private) sort of object reference types represented with an internal name. */
    private static final int INTERNAL = 12;
		/** The descriptors of the primitive types. */
    private static final String PRIMITIVE_DESCRIPTORS = "VZCBSIFJD";
}
```

在`Type`类里，也定义了一些`Type`类型的字段，这些字段是由上面的`int`和`String`类型的字段组合得到。

```java
public final class Type {
    public static final Type VOID_TYPE = new Type(VOID, PRIMITIVE_DESCRIPTORS, VOID, VOID + 1);
    public static final Type BOOLEAN_TYPE = new Type(BOOLEAN, PRIMITIVE_DESCRIPTORS, BOOLEAN, BOOLEAN + 1);
    public static final Type CHAR_TYPE = new Type(CHAR, PRIMITIVE_DESCRIPTORS, CHAR, CHAR + 1);
    public static final Type BYTE_TYPE = new Type(BYTE, PRIMITIVE_DESCRIPTORS, BYTE, BYTE + 1);
    public static final Type SHORT_TYPE = new Type(SHORT, PRIMITIVE_DESCRIPTORS, SHORT, SHORT + 1);
    public static final Type INT_TYPE = new Type(INT, PRIMITIVE_DESCRIPTORS, INT, INT + 1);
    public static final Type FLOAT_TYPE = new Type(FLOAT, PRIMITIVE_DESCRIPTORS, FLOAT, FLOAT + 1);
    public static final Type LONG_TYPE = new Type(LONG, PRIMITIVE_DESCRIPTORS, LONG, LONG + 1);
    public static final Type DOUBLE_TYPE = new Type(DOUBLE, PRIMITIVE_DESCRIPTORS, DOUBLE, DOUBLE + 1);
}
```

#### 静态方法

这里介绍的几个`get*Type()`方法，是静态（`static`）方法。这几个方法的主要目的就是得到一个`Type`对象。

```java
public final class Type {
  public static Type getType(final Class<?> clazz) {
    if (clazz.isPrimitive()) {
      if (clazz == Integer.TYPE) {
        return INT_TYPE;
      } else if (clazz == Void.TYPE) {
        return VOID_TYPE;
      } else if (clazz == Boolean.TYPE) {
        return BOOLEAN_TYPE;
      } else if (clazz == Byte.TYPE) {
        return BYTE_TYPE;
      } else if (clazz == Character.TYPE) {
        return CHAR_TYPE;
      } else if (clazz == Short.TYPE) {
        return SHORT_TYPE;
      } else if (clazz == Double.TYPE) {
        return DOUBLE_TYPE;
      } else if (clazz == Float.TYPE) {
        return FLOAT_TYPE;
      } else if (clazz == Long.TYPE) {
        return LONG_TYPE;
      } else {
        throw new AssertionError();
      }
    } else {
      return getType(getDescriptor(clazz));
    }
  }

  public static Type getType(final Constructor<?> constructor) {
    return getType(getConstructorDescriptor(constructor));
  }

  public static Type getType(final Method method) {
    return getType(getMethodDescriptor(method));
  }

  public static Type getType(final String typeDescriptor) {
    return getTypeInternal(typeDescriptor, 0, typeDescriptor.length());
  }

  public static Type getMethodType(final String methodDescriptor) {
    return new Type(METHOD, methodDescriptor, 0, methodDescriptor.length());
  }

  public static Type getObjectType(final String internalName) {
    return new Type(internalName.charAt(0) == '[' ? ARRAY : INTERNAL, internalName, 0, internalName.length());
  }
}
```

#### 获取Type对象

`Type`类有一个`private`的构造方法，因此`Type`对象实例不能通过`new`关键字来创建。但是，`Type`类提供了static method和static field来获取对象。

##### 方式一：java.lang.Class

从一个`java.lang.Class`对象来获取`Type`对象：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Type t = Type.getType(String.class);
        System.out.println(t); // Ljava/lang/String;
    }
}
```

##### 方式二：descriptor

从一个描述符（descriptor）来获取`Type`对象：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Type t1 = Type.getType("Ljava/lang/String;");
        System.out.println(t1); // Ljava/lang/String;

        // 这里是方法的描述符
        Type t2 = Type.getMethodType("(II)I");
        System.out.println(t2); // (II)I
    }
}
```

##### 方式三：internal name

从一个internal name来获取`Type`对象：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Type t = Type.getObjectType("java/lang/String");
        System.out.println(t); // Ljava/lang/String;
    }
}
```

##### 方式四：static field

从一个`Type`类的静态字段来获取`Type`对象：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
    public static void main(String[] args) throws Exception {
        Type t = Type.INT_TYPE;
        System.out.println(t); // I
    }
}
```

### 3.4.4 特殊的方法

#### array-related methods

这里介绍的两个方法与数组类型相关：

- `getDimensions()`方法，用于获取数组的维度
- `getElementType()`方法，用于获取数组的元素的类型

```java
public final class Type {
  /**
   * Returns the number of dimensions of this array type. This method should only be used for an
   * array type.
   *
   * @return the number of dimensions of this array type.
   */
  public int getDimensions() {
    int numDimensions = 1;
    while (valueBuffer.charAt(valueBegin + numDimensions) == '[') {
      numDimensions++;
    }
    return numDimensions;
  }

  /**
   * Returns the type of the elements of this array type. This method should only be used for an
   * array type.
   *
   * @return Returns the type of the elements of this array type.
   */
  public Type getElementType() {
    final int numDimensions = getDimensions();
    return getTypeInternal(valueBuffer, valueBegin + numDimensions, valueEnd);
  }
}
```

示例代码：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type t = Type.getType("[[[[[Ljava/lang/String;");

    int dimensions = t.getDimensions();
    Type elementType = t.getElementType();

    System.out.println(dimensions);    // 5
    System.out.println(elementType);   // Ljava/lang/String;
  }
}
```

#### method-related methods

这里介绍的两个方法与“方法”相关：

- `getArgumentTypes()`方法，用于获取“方法”接收的参数类型
- `getReturnType()`方法，用于获取“方法”返回值的类型

```java
public final class Type {
  /**
   * Returns the argument types of methods of this type. This method should only be used for method
   * types.
   *
   * @return the argument types of methods of this type.
   */
  public Type[] getArgumentTypes() {
    return getArgumentTypes(getDescriptor());
  }
  
	/**
   * Returns the return type of methods of this type. This method should only be used for method
   * types.
   *
   * @return the return type of methods of this type.
   */
  public Type getReturnType() {
    return getReturnType(getDescriptor());
  }
}
```

示例代码：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type methodType = Type.getMethodType("(Ljava/lang/String;I)V");

    String descriptor = methodType.getDescriptor();
    Type[] argumentTypes = methodType.getArgumentTypes();
    Type returnType = methodType.getReturnType();

    System.out.println("Descriptor: " + descriptor);
    System.out.println("Argument Types:");
    for (Type t : argumentTypes) {
      System.out.println("    " + t);
    }
    System.out.println("Return Type: " + returnType);
  }
}
```

输出结果：

```java
Descriptor: (Ljava/lang/String;I)V
Argument Types:
    Ljava/lang/String;
    I
Return Type: V
```

#### size-related methods

这里列举的3个方法是与“类型占用slot空间的大小”相关：

- `getSize()`方法，用于返回某一个类型所占用的slot空间的大小。
- `getArgumentsAndReturnSizes()`方法，用于返回方法所对应的slot空间的大小。
  - 其中，参数大小`int argumentsSize = argumentsAndReturnSizes >> 2;`。注意，在`argumentsSize`当中包含了隐藏的`this`变量（non-static方法，static方法则无this参数）。
  - 其中，返回值大小`int returnSize = argumentsAndReturnSizes 0x03)`

```java
public final class Type {
  public int getSize() {
    switch (sort) {
      case VOID:
        return 0;
      case BOOLEAN:
      case CHAR:
      case BYTE:
      case SHORT:
      case INT:
      case FLOAT:
      case ARRAY:
      case OBJECT:
      case INTERNAL:
        return 1;
      case LONG:
      case DOUBLE:
        return 2;
      default:
        throw new AssertionError();
    }
  }

  public int getArgumentsAndReturnSizes() {
    return getArgumentsAndReturnSizes(getDescriptor());
  }

  public static int getArgumentsAndReturnSizes(final String methodDescriptor) {
    int argumentsSize = 1;
    // Skip the first character, which is always a '('.
    int currentOffset = 1;
    int currentChar = methodDescriptor.charAt(currentOffset);
    // Parse the argument types and compute their size, one at a each loop iteration.
    while (currentChar != ')') {
      if (currentChar == 'J' || currentChar == 'D') {
        currentOffset++;
        argumentsSize += 2;
      } else {
        while (methodDescriptor.charAt(currentOffset) == '[') {
          currentOffset++;
        }
        if (methodDescriptor.charAt(currentOffset++) == 'L') {
          // Skip the argument descriptor content.
          int semiColumnOffset = methodDescriptor.indexOf(';', currentOffset);
          currentOffset = Math.max(currentOffset, semiColumnOffset + 1);
        }
        argumentsSize += 1;
      }
      currentChar = methodDescriptor.charAt(currentOffset);
    }
    currentChar = methodDescriptor.charAt(currentOffset + 1);
    if (currentChar == 'V') {
      return argumentsSize << 2;
    } else {
      int returnSize = (currentChar == 'J' || currentChar == 'D') ? 2 : 1;
      return argumentsSize << 2 | returnSize;
    }
  }
}
```

示例代码：

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type t = Type.INT_TYPE;
    System.out.println(t.getSize()); // 1
  }
}
```

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type t = Type.LONG_TYPE;
    System.out.println(t.getSize()); // 2, long和double占用2个槽位，在字节码中用"TOP"占位(第二个槽位)
  }
}
```

```java
import org.objectweb.asm.Type;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type t = Type.getMethodType("(II)I");
    int value = t.getArgumentsAndReturnSizes();

    int argumentsSize = value >> 2;
    int returnSize = value & 0b11;

    System.out.println(argumentsSize); // 3, 因non-static方法，所以local variables下标0位置是this指针
    System.out.println(returnSize);    // 1
  }
}
```

#### opcode-related methods

这里介绍的方法与opcode相关：

- `getOpcode(int opcode)`方法，会让我们写代码的过程中更加方便。(根据Type自动将原Opcodes转换成对应当前Type的Opcodes)

但是，值得注意的一点：**参数`opcode`的取值是一些“指定的操作码（opcode）”，而不能随便乱写**。 这些“指定的操作码（opcode）”在方法的注释中写明了，必须是 `ILOAD`, `ISTORE`, `IALOAD`, `IASTORE`, `IADD`, `ISUB`, `IMUL`, `IDIV`, `IREM`, `INEG`, `ISHL`, `ISHR`, `IUSHR`, `IAND`, `IOR`, `IXOR` 和 `IRETURN`中的某一个值。

**作为初学者，我们一定要注意`getOpcode(opcode)`方法参数值的有效性。** 否则，它就会返回一个错误的结果，导致整个程序出现错误。 当我们去检查代码整体思路的时候，会觉得程序的代码逻辑上没有问题，但是生成的代码就是不能正确运行， 百思不得其解，很可能就是给`getOpcode(opcode)`方法传入了一个错误的参数值。

举个具体例子，有一次我给`getOpcode(opcode)`方法传入了`ICONST_0`值（这是一个错误的值）， 我期望得到`FCONST_0`或`DCONST_0`，最终没有达到预期的效果。 因为`ICONST_0`，对于`getOpcode(int opcode)`来说，不是一个合理的值。

```java
public final class Type {
  /**
     * Returns a JVM instruction opcode adapted to this {@link Type}. This method must not be used for
     * method types.
     *
     * @param opcode a JVM instruction opcode. This opcode must be one of ILOAD, ISTORE, IALOAD,
     *     IASTORE, IADD, ISUB, IMUL, IDIV, IREM, INEG, ISHL, ISHR, IUSHR, IAND, IOR, IXOR and
     *     IRETURN.
     * @return an opcode that is similar to the given opcode, but adapted to this {@link Type}. For
     *     example, if this type is {@code float} and {@code opcode} is IRETURN, this method returns
     *     FRETURN.
     */
  public int getOpcode(final int opcode) {
    if (opcode == Opcodes.IALOAD || opcode == Opcodes.IASTORE) {
      switch (sort) {
        case BOOLEAN:
        case BYTE:
          return opcode + (Opcodes.BALOAD - Opcodes.IALOAD);
        case CHAR:
          return opcode + (Opcodes.CALOAD - Opcodes.IALOAD);
        case SHORT:
          return opcode + (Opcodes.SALOAD - Opcodes.IALOAD);
        case INT:
          return opcode;
        case FLOAT:
          return opcode + (Opcodes.FALOAD - Opcodes.IALOAD);
        case LONG:
          return opcode + (Opcodes.LALOAD - Opcodes.IALOAD);
        case DOUBLE:
          return opcode + (Opcodes.DALOAD - Opcodes.IALOAD);
        case ARRAY:
        case OBJECT:
        case INTERNAL:
          return opcode + (Opcodes.AALOAD - Opcodes.IALOAD);
        case METHOD:
        case VOID:
          throw new UnsupportedOperationException();
        default:
          throw new AssertionError();
      }
    } else {
      switch (sort) {
        case VOID:
          if (opcode != Opcodes.IRETURN) {
            throw new UnsupportedOperationException();
          }
          return Opcodes.RETURN;
        case BOOLEAN:
        case BYTE:
        case CHAR:
        case SHORT:
        case INT:
          return opcode;
        case FLOAT:
          return opcode + (Opcodes.FRETURN - Opcodes.IRETURN);
        case LONG:
          return opcode + (Opcodes.LRETURN - Opcodes.IRETURN);
        case DOUBLE:
          return opcode + (Opcodes.DRETURN - Opcodes.IRETURN);
        case ARRAY:
        case OBJECT:
        case INTERNAL:
          if (opcode != Opcodes.ILOAD && opcode != Opcodes.ISTORE && opcode != Opcodes.IRETURN) {
            throw new UnsupportedOperationException();
          }
          return opcode + (Opcodes.ARETURN - Opcodes.IRETURN);
        case METHOD:
          throw new UnsupportedOperationException();
        default:
          throw new AssertionError();
      }
    }
  }
}
```

示例代码：

```java
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.util.Printer;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Type t = Type.FLOAT_TYPE;

    int[] opcodes = new int[]{
      Opcodes.IALOAD,
      Opcodes.IASTORE,
      Opcodes.ILOAD,
      Opcodes.ISTORE,
      Opcodes.IADD,
      Opcodes.ISUB,
      Opcodes.IRETURN,
    };

    for (int oldOpcode : opcodes) {
      int newOpcode = t.getOpcode(oldOpcode);

      String oldName = Printer.OPCODES[oldOpcode];
      String newName = Printer.OPCODES[newOpcode];

      System.out.printf("%-7s --- %-7s%n", oldName, newName);
    }
  }
}
```

输出结果：

```
IALOAD  --- FALOAD
IASTORE --- FASTORE
ILOAD   --- FLOAD
ISTORE  --- FSTORE
IADD    --- FADD
ISUB    --- FSUB
IRETURN --- FRETURN
```

### 3.4.5 总结

本文主要对`Type`类进行了介绍，内容总结如下：

- 第一点，`Type`类的作用是什么？**`Type`类是一个工具类，它的一个主要目的是将Java语言当中的概念转换成ClassFile当中的概念**。
- 第二点，学习`Type`类的方式就是“分而治之”。在`Type`类当中，定义了许多的字段和方法，它们是一个整体，内容也很繁杂；于是，我们将`Type`类分成不同的部分来讲解，就是希望大家能循序渐进的理解这个类的各个部分，方便以后对该类的使用。

当然，也不要求大家一下子把这个类的内容全部掌握，因为这里面的很多方法都是和ClassFile的结构密切相关的；如果大家对于ClassFile的结构不太了解，那么理解这些方法也会有一定的困难。总的来说，希望大家在以后使用的过程中，对这些方法慢慢熟悉起来。

## 3.5 修改已有的方法（添加－进入和退出）

### 3.5.1 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    System.out.println("this is a test method.");
  }
}
```

我们想实现的预期目标：对于`test()`方法，在“方法进入”时和“方法退出”时，添加一条打印语句。

第一种情况，在“方法进入”时，预期目标如下所示：

```java
public class HelloWorld {
  public void test() {
    System.out.println("Method Enter...");
    System.out.println("this is a test method.");
  }
}
```

第二种情况，在“方法退出”时，预期目标如下所示：

```java
public class HelloWorld {
  public void test() {
    System.out.println("this is a test method.");
    System.out.println("Method Exit...");
  }
}
```

现在，我们有了明确的预期目标；接下来，就是将这个预期目标转换成具体的ASM代码。那么，应该怎么实现呢？从哪里着手呢？

### 3.5.2 实现思路

我们知道，现在的内容是Class Transformation的操作，其中涉及到三个主要的类：`ClassReader`、`ClassVisitor`和`ClassWriter`。其中，`ClassReader`负责读取Class文件，`ClassWriter`负责生成Class文件，而具体的`ClassVisitor`负责进行Transformation的操作。换句话说，我们还是应该从`ClassVisitor`类开始。

第一步，回顾一下`ClassVisitor`类当中主要的`visitXxx()`方法有哪些。在`ClassVisitor`类当中，有`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`方法；这些`visitXxx()`方法与`.class`文件里的不同部分之间是有对应关系的，如下图：

![Java ASM系列：（024）修改已有的方法（添加－进入和退出）_ASM](https://s2.51cto.com/images/20210629/1624967397832240.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

根据我们的预期目标，现在想要修改的是“方法”的部分，那么就对应着`ClassVisitor`类的`visitMethod()`方法。<u>`ClassVisitor.visitMethod()`会返回一个`MethodVisitor`类的实例；而`MethodVisitor`类就是用来生成方法的“方法体”</u>。

第二步，回顾一下`MethodVisitor`类当中定义了哪些`visitXxx()`方法。

![Java ASM系列：（024）修改已有的方法（添加－进入和退出）_ASM_02](https://s2.51cto.com/images/20210629/1624967433536552.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在`MethodVisitor`类当中，定义的`visitXxx()`方法比较多，但是我们可以将这些`visitXxx()`方法进行分组：

- 第一组，`visitCode()`方法，标志着方法体（method body）的开始。
- 第二组，`visitXxxInsn()`方法，对应方法体（method body）本身，这里包含多个方法。
- 第三组，`visitMaxs()`方法，标志着方法体（method body）的结束。
- 第四组，`visitEnd()`方法，是最后调用的方法。

另外，我们也回顾一下，在`MethodVisitor`类中，`visitXxx()`方法的调用顺序：

- 第一步，调用`visitCode()`方法，调用一次。
- 第二步，调用`visitXxxInsn()`方法，可以调用多次。
- 第三步，调用`visitMaxs()`方法，调用一次。
- 第四步，调用`visitEnd()`方法，调用一次。

到了这一步，我们基本上就知道了：需要修改的内容就位于`visitCode()`和`visitMaxs()`方法之间，这是一个大概的范围。

第三步，精确定位。也就是说，在`MethodVisitor`类当中，要确定出要在哪一个`visitXxx()`方法里进行修改。

#### 方法进入

如果我们想在“方法进入”时，添加一些打印语句，那么我们有两个位置可以添加打印语句：

- 第一个位置，就是在`visitCode()`方法中。
- 第二个位置，就是在第1个`visitXxxInsn()`方法中。

在这两个位置当中，我们推荐使用`visitCode()`方法。因为`visitCode()`方法总是位于方法体（method body）的前面，而第1个`visitXxxInsn()`方法是不稳定的（因为每个方法第一个执行的`visitXxxInsn`都大不相同）。

```java
public void visitCode() {
    // 首先，处理自己的代码逻辑
    // TODO: 添加“方法进入”时的代码

    // 其次，调用父类的方法实现
    super.visitCode();
}
```

#### 方法退出

如果我们在“方法退出”时想添加的代码，是否可以添加到`visitMaxs()`方法内呢？这样做是不行的。因为在执行`visitMaxs()`方法之前，方法体（method body）已经执行过了：在方法体（method body）当中，里面会包含return语句；如果return语句一执行，后面的任何语句都不会再执行了；**换句话说，如果在`visitMaxs()`方法内添加的打印输出语句，由于前面方法体（method body）中已经执行了return语句，后面的任何语句就执行不到了**。

那么，到底是应该在哪里添加代码呢？为了回答这个问题，我们需要知道“方法退出”有哪几种情况。**方法的退出，有两种情况，一种是正常退出（执行return语句），另一种是异常退出（执行throw语句）**；接下来，就是将这两种退出情况应用到ASM的代码层面。

**在`MethodVisitor`类当中，无论是执行return语句，还是执行throw语句，都是通过`visitInsn(opcode)`方法来实现的**。所以，如果我们想在“方法退出”时，添加一些语句，那么这些语句放到`visitInsn(opcode)`方法中就可以了。

```java
public void visitInsn(int opcode) {
    // 首先，处理自己的代码逻辑
    if (opcode == Opcodes.ATHROW || (opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN)) {
        // TODO: 添加“方法退出”时的代码
    }

    // 其次，调用父类的方法实现
    super.visitInsn(opcode);
}
```

------

> 讲师有一个编程的习惯：在编写ASM代码的时候，如果写了一个类，它继承自`ClassVisitor`，那么就命名成`XxxVisitor`；如果写了一个类，它继承自`MethodVisitor`，那么就命名成`XxxAdapter`。通过类的名字，就可以区分出哪些类是继承自`ClassVisitor`，哪些类是继承自`MethodVisitor`。

### 3.5.3 示例一：方法进入

#### 编码实现

注意点：

+ 如果在`<init>`构造方法的方法初始位置插入逻辑，由于还没有调用`super()`进行初始化，所以此时调用`this`的实例方法会抛出异常
+ 需要在方法体开头插入逻辑，则只需重写MethodVisitor的`visitCode()`方法

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

/**
 * 目标：在方法体执行开始时打印 "Method Enter..."
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/27 3:37 PM
 */
public class MethodEnterCVisitor extends ClassVisitor {
  public MethodEnterCVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    // 先调用当前 ClassVisitor 实例 成员变量cv的 visitMethod(效果就是递归，从最早一个ClassVisitor的visitMethod开始执行)
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    // 抽象方法没方法体，native方法也没方法体，所以直接走原本的 visitMethod逻辑
    // 构造方法，this() 或 super() 必须是第一个执行的逻辑，所以也排除 <init> 构造方法
    if(null == mv || (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT || (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE || name.equals("<init>")) {
      return mv;
    }
    // 执行自定义的 MethodVisitor 内的逻辑
    return new MethodEnterMVisitor(api, mv);
  }

  /**
     * 实现在方法体执行开头插入自定义逻辑的 MethodVisitor
     */
  private static class MethodEnterMVisitor extends MethodVisitor {
    protected MethodEnterMVisitor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitCode() {
      // 执行我们自定义添加的逻辑，输出 "Method Enter..."
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitLdcInsn("Method Enter...");
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      // 继续递归执行 MethodVisitor成员变量mv的 visitCode() 方法
      super.visitCode();
    }
  }
}
```

在上面`MethodEnterAdapter`类的`visitCode()`方法中，主要是做两件事情：

- 首先，处理自己的代码逻辑。
- 其次，调用父类的方法实现。

在处理自己的代码逻辑中，有3行代码。这3条语句的作用就是添加`System.out.println("Method Enter...");`语句：

```java
super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
super.visitLdcInsn("Method Enter...");
super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

注意，上面的代码中使用了`super`关键字。

事实上，在`MethodVisitor`类当中，定义了一个`protected MethodVisitor mv;`字段。我们也可以使用`mv`这个字段，代码也可以这样写：

```java
mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv.visitLdcInsn("Method Enter...");
mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

**但是这样写，可能会遇到`mv`为`null`的情况，这样就会出现`NullPointerException`异常**。

如果使用`super`，就会避免`NullPointerException`异常的情况。因为使用`super`的情况下，就是调用父类定义的方法，在本例中其实就是调用`MethodVisitor`类里定义的方法。在`MethodVisitor`类里的`visitXxx()`方法中，会先对`mv`进行是否为`null`的判断，所以就不会出现`NullPointerException`的情况。

```java
public abstract class MethodVisitor {
  protected MethodVisitor mv;

  public void visitCode() {
    if (mv != null) {
      mv.visitCode();
    }
  }

  public void visitInsn(final int opcode) {
    if (mv != null) {
      mv.visitInsn(opcode);
    }
  }

  public void visitIntInsn(final int opcode, final int operand) {
    if (mv != null) {
      mv.visitIntInsn(opcode, operand);
    }
  }

  public void visitVarInsn(final int opcode, final int var) {
    if (mv != null) {
      mv.visitVarInsn(opcode, var);
    }
  }

  public void visitFieldInsn(final int opcode, final String owner, final String name, final String descriptor) {
    if (mv != null) {
      mv.visitFieldInsn(opcode, owner, name, descriptor);
    }
  }

  // ......

  public void visitMaxs(final int maxStack, final int maxLocals) {
    if (mv != null) {
      mv.visitMaxs(maxStack, maxLocals);
    }
  }

  public void visitEnd() {
    if (mv != null) {
      mv.visitEnd();
    }
  }
}
```

#### 进行转换

```java
package com.example;

import core.ClassAddMethodVisitor;
import core.MethodEnterCVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodEnterCVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Method m = clazz.getDeclaredMethod("test");

    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

#### 特殊情况：`<init>()`方法

在`.class`文件中，`<init>()`方法，就表示类当中的构造方法。

我们在“方法进入”时，有一个对于`<init>`的判断：

```java
if (mv != null && !"<init>".equals(name)) {
    // ......
}
```

为什么要对`<init>()`方法进行特殊处理呢？

> Java requires that if you call this() or super() in a constructor, it must be the first statement.

```java
public class HelloWorld {
    public HelloWorld() {
        System.out.println("Method Enter...");
        super(); // 报错：Call to 'super()' must be first statement in constructor body
    }
}
```

大家可以做个实践，就是去掉对于`<init>()`方法的判断，会发现它好像也是可以正常执行的。（可以执行，因为没有调用当前未初始化的实例的方法，所以没事）

但是，如果我们换一下添加的语句，就会出错了：

```java
super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
super.visitVarInsn(Opcodes.ALOAD, 0);
super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Object", "toString", "()Ljava/lang/String;", false);
super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

因为如上代码，通过`javap -p -v sample.HelloWorld ` 可以看出来，尝试在`invokespecial`初始化this指针指向的内存之前，就调用this对象的方法，所以会抛出异常。（JVM先分配内存，然后调用`super()`时会对内存进行初始化，此时`this`才是一个有效的实例的指针，否则是未完成对内存数据初始化的`uninitialized_this`）

```java
Exception in thread "main" java.lang.VerifyError: Bad type on operand stack
Exception Details:
  Location:
    sample/HelloWorld.<init>()V @4: invokevirtual
  Reason:
    Type uninitializedThis (current frame, stack[1]) is not assignable to 'java/lang/Object'
  Current Frame:
    bci: @4
    flags: { flagThisUninit }
    locals: { uninitializedThis }
    stack: { 'java/io/PrintStream', uninitializedThis }
  Bytecode:
    0000000: b200 0c2a b600 10b6 0016 2ab7 0018 b1
```

### 3.5.4 示例二：方法退出

#### 编码实现

注意点：

+ 不能通过重写MethodVisitor的`visitMaxs()`方法来完成在方法正常/异常返回时插入自定义逻辑，因为此时逻辑已经执行完return或者throw了，不会再往下执行代码
+ 由于return和throw都在`visitInsn()`方法中，所以重写该方法，可以实现在return或者throw之前执行自定义的逻辑
+ 由于构造方法执行完`super()`后，this指针指向的内存就被初始化了，所以可以不过滤`name.equals("<init>")`的情况，在构造方法返回之前，也可以执行自定义的逻辑
+ 如果重写`visitMaxs()`方法，会发现里面就算插入自定义逻辑也不会生效，`javap -p -v sample.HelloWorld`指令查看后会发现自定义逻辑被替换成`nop`
+ 通过javap指令，可以看到如果代码中throw异常，我们插入的自定义逻辑会被放在`athrow`之前。

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

/**
 * 目标：在每个方法之前完，正常/异常返回之前，输出 "Method Exit..."
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/27 4:33 PM
 */
public class MethodExitCVisitor extends ClassVisitor {
  public MethodExitCVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    // 方法 正常return 或 异常返回之前增加逻辑，这里 构造方法结束前加逻辑也是ok的，所以不用过滤 name.equals("<init>") 的情况
    if(null == mv || (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT || (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE) {
      return mv;
    }
    return new MethodExitMVisitor(api,mv);
  }

  /**
     * 在每个方法 正常return 或 异常返回之前，添加 自定义逻辑 <br/>
     * 这里不能在 visitMaxs() 里写逻辑，因为到这一步的时候，方法已经执行完 返回的逻辑了，不接受其他逻辑的执行 <br/>
     * 因为和方法返回有关的操作，都在 visitInsn()方法里，所以只要Override该方法，插入自定义的逻辑即可
     */
  private static class MethodExitMVisitor extends MethodVisitor {
    protected MethodExitMVisitor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitInsn(int opcode) {
      // 正常退出方法 or 异常退出方法时，执行自定义逻辑
      if(opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN || opcode == Opcodes.ATHROW) {
        super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        super.visitLdcInsn("Method Exit...");
        super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
			 // 下面代码是能正常运行的
       // super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
       // super.visitVarInsn(Opcodes.ALOAD, 0);
       // super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Object", "toString", "()Ljava/lang/String;", false);
       // super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      }
      // 继续递归执行 MethodVisitor 实例的成员变量mv的 visitInsn()方法
      super.visitInsn(opcode);
    }

    @Override
    public void visitMaxs(int maxStack, int maxLocals) {
      // 这里会通过 javap -v -p sample.HelloWord 指令，可以发现 下面代码被替换成 nop，并没有真正按照我们的想法执行相关逻辑
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitLdcInsn("no executed, Method Enter...");
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      // 继续递归执行 MethodVisitor成员变量mv的 visitMaxs() 方法
      super.visitMaxs(maxStack, maxLocals);
    }
  }
}
```

#### 进行转换

```java
package com.example;

import core.ClassAddMethodVisitor;
import core.MethodEnterCVisitor;
import core.MethodExitCVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodExitCVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}

```

#### 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Method m = clazz.getDeclaredMethod("test");

    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

输出结果：

```shell
this is a test method.
Method Exit...
```

### 3.5.5 示例三：方法进入和方法退出

#### 方式一：串联多个ClassVisitor

第一种方式，就是将多个`ClassVisitor`类串联起来。

```java
package com.example;

import core.ClassAddMethodVisitor;
import core.MethodEnterCVisitor;
import core.MethodExitCVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv1 = new MethodExitCVisitor(api, cw);
    ClassVisitor cv = new MethodEnterCVisitor(api, cv1);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}

```

#### 方式二：实现单个ClassVisitor

第二种方式，就是将所有的代码都放到一个`ClassVisitor`类里面。

编码实现：

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

/**
 * 目标：在方法原有的逻辑执行前输出 "Method Enter...", 方法正常/异常返回前输出 "Method Exit..."
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/27 5:28 PM
 */
public class MethodAroundCVisitor extends ClassVisitor {
  public MethodAroundCVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    // 先调用当前 ClassVisitor 实例 成员变量cv的 visitMethod(效果就是递归，从最早一个ClassVisitor的visitMethod开始执行)
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    // 抽象方法没方法体，native方法也没方法体，所以直接走原本的 visitMethod逻辑
    // 构造方法，this() 或 super() 必须是第一个执行的逻辑，所以也排除 <init> 构造方法
    if(null == mv || (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT || (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE || name.equals("<init>")) {
      return mv;
    }
    // 执行自定义的 MethodVisitor 内的逻辑
    return new MethodAroundMVisitor(api, mv);
  }

  /**
     * 在原有的方法执行逻辑前 和 返回前，插入自定义逻辑
     */
  private class MethodAroundMVisitor extends MethodVisitor {

    protected MethodAroundMVisitor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitCode() {
      // 执行我们自定义添加的逻辑，输出 "Method Enter..."
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitLdcInsn("Method Enter...");
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      // 继续递归执行 MethodVisitor成员变量mv的 visitCode() 方法
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      // 正常退出方法 or 异常退出方法时，执行自定义逻辑
      if(opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN || opcode == Opcodes.ATHROW) {
        super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        super.visitLdcInsn("Method Exit...");
        super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      }
      // 继续递归执行 MethodVisitor 实例的成员变量mv的 visitInsn()方法
      super.visitInsn(opcode);
    }
  }
}
```

进行转换

```java
package com.example;

import core.ClassAddMethodVisitor;
import core.MethodAroundCVisitor;
import core.MethodEnterCVisitor;
import core.MethodExitCVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodAroundCVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 3.5.6 总结

本文主要是对“方法进入”和“方法退出”添加代码进行介绍，内容总结如下：

- 第一点，在“方法进入”时和“方法退出”时添加代码，应该如何实现？
  - **在“方法进入”时添加代码，是在`visitCode()`方法当中完成；**
  - **在“方法退出”添加代码时，是在`visitInsn(opcode)`方法中，判断`opcode`为return或throw的情况下完成。**
- 第二点，在“方法进入”时和“方法退出”时添加代码，有一些特殊的情况，需要小心处理：
  - **接口，是否需要处理？接口当中的抽象方法没有方法体，但也可能有带有方法体的default方法。**
  - 带有特殊修饰符的方法：
    - **抽象方法，是否需要处理？不只是接口当中有抽象方法，抽象类里也可能有抽象方法。抽象方法，是没有方法体的。**
    - **native方法，是否需要处理？native方法是没有方法体的。**
  - 名字特殊的方法，例如，构造方法（`<init>()`）和静态初始化方法（`<clinit>()`），是否需要处理？

*另外，在编写代码的时候，我们遵循一个“规则”：如果是`ClassVisitor`的子类，就取名为`XxxVisitor`类；如果是`MethodVisitor`的子类，就取名为`XxxAdapter`类。（这个是讲师的个人编码习惯，我这边自己敲的代码没这么搞）*

**在后续的内容中，我们会介绍`AdviceAdapter`类，它能很容易帮助我们在“方法进入”时和“方法退出”时添加代码**。 那么，这就带来有一个问题，既然使用`AdviceAdapter`类实现起来很容易，那么为什么还要讲本文的实现方式呢？有两个原因。

- 第一个原因，本文的介绍方式侧重于让大家理解“工作原理”，而`AdviceAdapter`则侧重于“应用”，`AdviceAdapter`的实现也是基于`visitCode()`和`visitInsn(opcode)`方法实现的，在理解上有一个步步递进的关系。
- 第二个原因，虽然`AdviceAdapter`在“方法进入”时和“方法退出”时添加代码比较容易，大多数情况，都是能正常工作，但也有极其特殊的情况下，它会失败。这个时候，我们还是要回归到本文介绍的实现方式。

## 3.6 修改已有的方法（添加－进入和退出－打印方法参数和返回值）

### 3.6.1 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public int test(String name, int age, long idCard, Object obj) {
    int hashCode = 0;
    hashCode += name.hashCode();
    hashCode += age;
    hashCode += (int) (idCard % Integer.MAX_VALUE);
    hashCode += obj.hashCode();
    return hashCode;
  }
}
```

我们想实现的预期目标：打印出“方法接收的参数值”和“方法的返回值”。

```java
public class HelloWorld {
  public int test(String name, int age, long idCard, Object obj) {
    System.out.println(name);
    System.out.println(age);
    System.out.println(idCard);
    System.out.println(obj);
    int hashCode = 0;
    hashCode += name.hashCode();
    hashCode += age;
    hashCode += (int) (idCard % Integer.MAX_VALUE);
    hashCode += obj.hashCode();
    System.out.println(hashCode);
    return hashCode;
  }
}
```

实现这个功能的思路：在“方法进入”的时候，打印出“方法接收的参数值”；在“方法退出”的时候，打印出“方法的返回值”。

### 3.6.2 实现一：针对test方法硬编码

我们要实现的第一个版本是比较简单的，它是在`MethodAroundCVisitor`类基础上直接修改得到的。

#### 编码实现

注意点：

+ long占两个槽位（第二个槽位用top占位标记）
+ 这里HelloWorld方法体中的hashCode本地变量被放置到了local variables下标6的位置，所以输出返回值的时候`ILOAD 6`

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

/**
 * 目标：在进入方法时输出方法的参数，退出方法前输出返回值
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/29 4:15 PM
 */
public class MethodAroundCVisitor extends ClassVisitor {
  public MethodAroundCVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    // 先调用当前 ClassVisitor 实例 成员变量cv的 visitMethod(效果就是递归，从最早一个ClassVisitor的visitMethod开始执行)
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    // 抽象方法没方法体，native方法也没方法体，所以直接走原本的 visitMethod逻辑
    // 构造方法，this() 或 super() 必须是第一个执行的逻辑，所以也排除 <init> 构造方法
    if(null == mv || (access & Opcodes.ACC_ABSTRACT) == Opcodes.ACC_ABSTRACT || (access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE || name.equals("<init>")) {
      return mv;
    }
    // 执行自定义的 MethodVisitor 内的逻辑
    return new MethodAroundMVisitor(api, mv);
  }

  /**
     * 在原有的方法执行逻辑前 和 返回前，插入自定义逻辑
     */
  private class MethodAroundMVisitor extends MethodVisitor {

    protected MethodAroundMVisitor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitCode() {
      // 针对 HelloWorld类的test()方法硬编码 请求参数的输出逻辑 int test(String name, int age, long idCard, Object obj )
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitVarInsn(Opcodes.ALOAD, 1);
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitVarInsn(Opcodes.ILOAD, 2);
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitVarInsn(Opcodes.LLOAD, 3);
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(J)V", false);
      super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitVarInsn(Opcodes.ALOAD, 5);
      super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
      // 继续递归执行 MethodVisitor成员变量mv的 visitCode() 方法
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      // 正常退出方法 or 异常退出方法时，执行自定义逻辑
      if(opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN || opcode == Opcodes.ATHROW) {
        super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        // 通过 HelloWorldFrameCore 查看 Frame 变化，结合 javap -p -v sample.HelloWorld, 可知 方法体中最后返回的hashCode，中间通过istore存储到了 local variables 下标 6位置
        super.visitVarInsn(Opcodes.ILOAD, 6);
        super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
      }
      // 继续递归执行 MethodVisitor 实例的成员变量mv的 visitInsn()方法
      super.visitInsn(opcode);
    }
  }
}
```

#### 进行转换

```java
package com.example;

import core.ClassAddMethodVisitor;
import core.MethodAroundCVisitor;
import core.MethodEnterCVisitor;
import core.MethodExitCVisitor;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodAroundCVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    int hashCode = instance.test("Tomcat", 10, System.currentTimeMillis(), new Object());
    int remainder = hashCode % 2;

    if (remainder == 0) {
      System.out.println("hashCode is even number.");
    }
    else {
      System.out.println("hashCode is odd number.");
    }
  }
}
```

#### 小结

缺点：不灵活。如果方法参数的数量和类型发生改变，这种方法就会失效。

那么，有没有办法来自动适应方法参数的数量和类型变化呢？答案是“有”。这个时候，就是`Type`类（`org.objectweb.asm.Type`）来发挥作用的地方。

### 3.6.3 实现二：通过Type灵活适配参数

#### 编码实现

注意点：

+ 需要先判断方法是否static，来判断是否local variables下标0位置存储this指针
+ 这里通过方法的Type类获取其请求参数`Type[]`数组，然后通过`int getOpcode(final int opcode)`自动适配当前Type所需的指令，将参数数据加载到operand stack，再根据不同类型进行不同的打印逻辑
+ 这里打印方法入参时，出现频繁的`SWAP`操作，以及为了实现多个槽位的SWAP而组合的`DUP_X2`、`POP`操作，原因是正常打印数据，应该先将out对象加载到operand stack，其次才是需要打印的数据。这里为了保证能正常调用println方法，所以需要对operand stack顶部的数据进行swap。
+ 打印返回值之前，需要先`DUP`操作，因为需要保证operand stack顶部复制一份返回值用于打印（调用`println`时会消耗operand stack栈顶的参数和out对象指针）。我们需要保证自定义逻辑不破坏原有的Frame结构（local variables 和 operand stack）

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;

import static org.objectweb.asm.Opcodes.*;

public class MethodParameterVisitor extends ClassVisitor {
  public MethodParameterVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !name.equals("<init>")) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodParameterAdapter(api, mv, access, name, descriptor);
      }
    }
    return mv;
  }

  private static class MethodParameterAdapter extends MethodVisitor {
    private final int methodAccess;
    private final String methodName;
    private final String methodDesc;

    public MethodParameterAdapter(int api, MethodVisitor mv, int methodAccess, String methodName, String methodDesc) {
      super(api, mv);
      this.methodAccess = methodAccess;
      this.methodName = methodName;
      this.methodDesc = methodDesc;
    }

    @Override
    public void visitCode() {
      // 首先，处理自己的代码逻辑
      boolean isStatic = ((methodAccess & ACC_STATIC) != 0);
      int slotIndex = isStatic ? 0 : 1;

      printMessage("Method Enter: " + methodName + methodDesc);

      Type methodType = Type.getMethodType(methodDesc);
      Type[] argumentTypes = methodType.getArgumentTypes();
      for (Type t : argumentTypes) {
        int sort = t.getSort();
        int size = t.getSize();
        String descriptor = t.getDescriptor();

        int opcode = t.getOpcode(ILOAD);
        super.visitVarInsn(opcode, slotIndex);

        if (sort == Type.BOOLEAN) {
          printBoolean();
        }
        else if (sort == Type.CHAR) {
          printChar();
        }
        else if (sort == Type.BYTE || sort == Type.SHORT || sort == Type.INT) {
          printInt();
        }
        else if (sort == Type.FLOAT) {
          printFloat();
        }
        else if (sort == Type.LONG) {
          printLong();
        }
        else if (sort == Type.DOUBLE) {
          printDouble();
        }
        else if (sort == Type.OBJECT && "Ljava/lang/String;".equals(descriptor)) {
          printString();
        }
        else if (sort == Type.OBJECT) {
          printObject();
        }
        else {
          printMessage("No Support");
          if (size == 1) {
            super.visitInsn(Opcodes.POP);
          }
          else {
            super.visitInsn(Opcodes.POP2);
          }
        }
        slotIndex += size;
      }

      // 其次，调用父类的方法实现
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      // 首先，处理自己的代码逻辑
      if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        printMessage("Method Exit:");
        if (opcode == IRETURN) {
          super.visitInsn(DUP);
          Type methodType = Type.getMethodType(methodDesc);
          Type returnType = methodType.getReturnType();
          if (returnType == Type.BOOLEAN_TYPE) {
            printBoolean();
          }
          else if (returnType == Type.CHAR_TYPE) {
            printChar();
          }
          else {
            printInt();
          }
        }
        else if (opcode == FRETURN) {
          super.visitInsn(DUP);
          printFloat();
        }
        else if (opcode == LRETURN) {
          super.visitInsn(DUP2);
          printLong();
        }
        else if (opcode == DRETURN) {
          super.visitInsn(DUP2);
          printDouble();
        }
        else if (opcode == ARETURN) {
          super.visitInsn(DUP);
          printObject();
        }
        else if (opcode == RETURN) {
          printMessage("    return void");
        }
        else {
          printMessage("    abnormal return");
        }
      }

      // 其次，调用父类的方法实现
      super.visitInsn(opcode);
    }

    private void printBoolean() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Z)V", false);
    }

    private void printChar() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(C)V", false);
    }

    private void printInt() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
    }

    private void printFloat() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(F)V", false);
    }

    private void printLong() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(DUP_X2);
      super.visitInsn(POP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(J)V", false);
    }

    private void printDouble() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(DUP_X2);
      super.visitInsn(POP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(D)V", false);
    }

    private void printString() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
    }

    private void printObject() {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitInsn(SWAP);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
    }

    private void printMessage(String str) {
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitLdcInsn(str);
      super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodParameterVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 小结

这种方式的特点就是，结合着`Type`类来使用，为方法参数的“类型”（`getSort()`）和“数量”（`getArgumentTypes()`和`getSize()`）赋予“灵魂”，让方法灵活起来。

##### Frame的初始状态

在JVM执行的过程中，在内存空间中，每一个运行的方法（method）都对应一个frame空间（省略 Frame Data）；在frame空间当中，有两个重要的结构，即local variables和operand stack，如下图所示。其中，local variables是一个数组结构，它通过索引来读取或设置数据；而operand stack是一个栈结构，符合“后进先出”（LIFO, Last in, First out）的规则。

![Java ASM系列：（025）修改已有的方法（添加－进入和退出－打印方法参数和返回值）_ClassFile](https://s2.51cto.com/images/20210623/1624446269398923.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在方法刚进入时，operand stack的初始状态是什么样的呢？回答：operand stack是空的，换句话说，“栈上没有任何元素”。

在方法刚进入时，local variables的初始状态是什么样的？相对来说，会比较复杂一些，因此我们重点说一下。对于local variables来说，我们把握以下三点：

- 第一点，local variables是通过索引（index）来确定里的元素的，它的索引（index）是从`0`开始计算的，每一个位置可以称之为slot。
- 第二点，在local variables中，存放数据的位置：this-方法接收的参数－方法内定义的局部变量。
  - **对于非静态方法（non-static method）来说，索引位置为`0`的位置存放的是`this`变量；**
  - 对于静态方法（static method）来说，索引位置为`0`的位置则不需要存储`this`变量。
- **第三点，在local variables中，`boolean`、`byte`、`char`、`short`、`int`、`float`和`reference`类型占用1个slot，而`long`和`double`类型占用2个slot。**（并且`boolean`、`byte`、`char`、`short`、`int`都会被直接当作int处理，使用`ILOAD`、`ISTORE`）

##### 打印语句

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dup_x2)
>
> dup_x2: Duplicate the top operand stack value and insert two or three values down

一般情况下，我们想打印一个字符串，可以如下写ASM代码：

```java
private void printMessage(String str) {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitLdcInsn(str);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
}
```

但是，有些情况下，我们想要打印的值已经位于operand stack上了，此时可以这样：

```java
private void printString() {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitInsn(SWAP);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
}
```

对于`int`类型的数据，它在operand stack当中占用1个位置，我们使用`swap`指令来完成`int`与`System.out`的位置互换。

```java
private void printInt() {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitInsn(SWAP);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
}
```

对于`long`类型的数据，它在operand stack当中占用2个位置，我们使用`dup_x2`和`pop`指令来完成`long`与`System.out`的位置互换。

```java
private void printLong() {
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitInsn(DUP_X2);
    super.visitInsn(POP);
    super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(J)V", false);
}
```

![img](https://lsieun.github.io/assets/images/java/asm/swap-int-and-long.png)

### 3.6.4 实现三：输出逻辑封装为工具类

#### 编码实现

首先，我们添加一个`ParameterUtils`类，在这个类定义了许多print方法，这些print方法可以打印不同类型的数据。

```java
package sample;

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
      System.out.println("    " + Arrays.toString((char[])value));
    }
    else {
      System.out.println("    " + value.getClass() + ": " + value.toString());
    }
  }

  public static void printText(String str) {
    System.out.println(str);
  }
}
```

在下面的`MethodParameterVisitor2`类当中，我们将使用`ParameterUtils`类帮助我们打印信息。

注意点：

+ 这里提取输出工具类后，原本的MethodVisitor实现类就少了很多繁琐的类型判断逻辑，并且无需编写一堆调用`println`的ASM代码。取而代之的是编写更易理解的java源代码。
+ 这里先拆分基础类型和非基础类型两大类，调用输出工具的方法，而针对Object又分类处理。

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;

import static org.objectweb.asm.Opcodes.*;

public class MethodParameterVisitor2 extends ClassVisitor {
  public MethodParameterVisitor2(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !name.equals("<init>")) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodParameterAdapter2(api, mv, access, name, descriptor);
      }
    }
    return mv;
  }

  private static class MethodParameterAdapter2 extends MethodVisitor {
    private final int methodAccess;
    private final String methodName;
    private final String methodDesc;

    public MethodParameterAdapter2(int api, MethodVisitor mv, int methodAccess, String methodName, String methodDesc) {
      super(api, mv);
      this.methodAccess = methodAccess;
      this.methodName = methodName;
      this.methodDesc = methodDesc;
    }

    @Override
    public void visitCode() {
      // 首先，处理自己的代码逻辑
      boolean isStatic = ((methodAccess & ACC_STATIC) != 0);
      int slotIndex = isStatic ? 0 : 1;

      printMessage("Method Enter: " + methodName + methodDesc);

      Type methodType = Type.getMethodType(methodDesc);
      Type[] argumentTypes = methodType.getArgumentTypes();
      for (Type t : argumentTypes) {
        int sort = t.getSort();
        int size = t.getSize();
        String descriptor = t.getDescriptor();
        int opcode = t.getOpcode(ILOAD);
        super.visitVarInsn(opcode, slotIndex);
        if (sort >= Type.BOOLEAN && sort <= Type.DOUBLE) {
          String methodDesc = String.format("(%s)V", descriptor);
          printValueOnStack(methodDesc);
        }
        else {
          printValueOnStack("(Ljava/lang/Object;)V");
        }

        slotIndex += size;
      }

      // 其次，调用父类的方法实现
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      // 首先，处理自己的代码逻辑
      if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        printMessage("Method Exit: " + methodName + methodDesc);
        if (opcode >= IRETURN && opcode <= DRETURN) {
          Type methodType = Type.getMethodType(methodDesc);
          Type returnType = methodType.getReturnType();
          int size = returnType.getSize();
          String descriptor = returnType.getDescriptor();

          if (size == 1) {
            super.visitInsn(DUP);
          }
          else {
            super.visitInsn(DUP2);
          }
          String methodDesc = String.format("(%s)V", descriptor);
          printValueOnStack(methodDesc);
        }
        else if (opcode == ARETURN) {
          super.visitInsn(DUP);
          printValueOnStack("(Ljava/lang/Object;)V");
        }
        else if (opcode == RETURN) {
          printMessage("    return void");
        }
        else {
          printMessage("    abnormal return");
        }
      }

      // 其次，调用父类的方法实现
      super.visitInsn(opcode);
    }

    private void printMessage(String str) {
      super.visitLdcInsn(str);
      super.visitMethodInsn(INVOKESTATIC, "sample/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);
    }

    private void printValueOnStack(String descriptor) {
      super.visitMethodInsn(INVOKESTATIC, "sample/ParameterUtils", "printValueOnStack", descriptor, false);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodParameterVisitor2(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 小结

这种方式的特点就是将“打印工作”放到一个单独的类里面。在这个单独的类里面，我们可以把内容打印出来，也可以输出到文件中，可以根据自己的需要进行修改。（在我个人看来，还有个好处就是将输出逻辑以java源代码形式维护，尽量减少了复杂的ASM代码编写，降低项目维护成本）

### 3.6.5 总结

本文主要介绍了如何实现打印方法的参数和返回值，我们提供了三个版本：

- 第一个版本，它的特点是代码固定、不够灵活。
- 第二个版本，它的特点是结合`Type`来使用，为方法参数的“类型”和“数量”赋予“灵魂”，让方法灵活起来。
- 第三个版本，它的特点是将“打印工作”移交给“专业人员”来处理。

本文内容总结如下：

- 第一点，从实现思路的角度来说，打印方法的参数和返回值，是在“方法进入”和“方法退出”的基础上实现的。在“方法进入”的时候，先将方法的参数打印出来；在“方法退出”的时候，再将方法的返回值打印出来。
- 第二点，我们呈现三个版本的目的，是为了让大家理解一步一步迭代的过程。如果大家日后用到类似的功能，直接参照第三个版本实现就可以了。

## 3.7 修改已有的方法（添加－进入和退出－方法计时）

### 3.7.1 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
import java.util.Random;

public class HelloWorld {

  public int add(int a, int b) throws InterruptedException {
    int c = a + b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(300);
    Thread.sleep(100 + num);
    return c;
  }

  public int sub(int a, int b) throws InterruptedException {
    int c = a - b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(400);
    Thread.sleep(100 + num);
    return c;
  }
}
```

我们想实现的预期目标：计算出方法的运行时间。这里有两种实现方式：

- 计算所有方法的运行时间
- 计算每个方法的运行时间

第一种方式，计算所有方法的运行时间，将该时间记录在`timer`字段当中：

```java
import java.util.Random;

public class HelloWorld {
  public static long timer;

  public int add(int a, int b) throws InterruptedException {
    timer -= System.currentTimeMillis();
    int c = a + b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(300);
    Thread.sleep(100 + num);
    timer += System.currentTimeMillis();
    return c;
  }

  public int sub(int a, int b) throws InterruptedException {
    timer -= System.currentTimeMillis();
    int c = a - b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(400);
    Thread.sleep(100 + num);
    timer += System.currentTimeMillis();
    return c;
  }
}
```

第二种方式，计算每个方法的运行时间，将每个方法的运行时间单独记录在对应的字段当中：

```java
import java.util.Random;

public class HelloWorld {
  public static long timer_add;
  public static long timer_sub;

  public int add(int a, int b) throws InterruptedException {
    timer_add -= System.currentTimeMillis();
    int c = a + b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(300);
    Thread.sleep(100 + num);
    timer_add += System.currentTimeMillis();
    return c;
  }

  public int sub(int a, int b) throws InterruptedException {
    timer_sub -= System.currentTimeMillis();
    int c = a - b;
    Random rand = new Random(System.currentTimeMillis());
    int num = rand.nextInt(400);
    Thread.sleep(100 + num);
    timer_sub += System.currentTimeMillis();
    return c;
  }
}
```

实现这个功能的思路：在“方法进入”的时候，减去一个时间戳；在“方法退出”的时候，加上一个时间戳，在这个过程当中就记录一个时间差。

有一个问题，我们为什么要计算方法的运行时间呢？如果我们想对现有的程序进行优化，那么需要对程序的整体性能有所了解，而方法的运行时间是衡量程序性能的一个重要参考。

### 3.7.2 实现一：计算所有方法的运行时间

#### 编码实现

注意点：

+ 需要先给类生成`timer`字段

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

/**
 * 目标：计算所有方法的执行耗时
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/29 8:42 PM
 */
public class MethodTimerVisitor extends ClassVisitor {
  /**
     * 当前类名
     */
  private String owner;
  /**
     * 当前类是否是接口
     */
  private boolean isInterface;

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    owner = name;
    isInterface = (access & ACC_INTERFACE) == ACC_INTERFACE;
  }

  public MethodTimerVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (isInterface || null == mv
        || ((access & ACC_ABSTRACT) == ACC_ABSTRACT) || ((access & ACC_NATIVE) == ACC_NATIVE)
        || "<init>".equals(name) || "<clinit>".equals(name)) {
      return mv;
    }
    return new MethodTimerMVisitor(api, mv, owner);
  }

  @Override
  public void visitEnd() {
    if (isInterface) {
      super.visitEnd();
			return;
    }
    // 给指定类生成timer成员变量(暂不考虑已经存在的情况), 计算方法执行耗时
    FieldVisitor fv = super.visitField(ACC_PUBLIC | ACC_STATIC, "timer", "J", null, null);
    if (null != fv) {
      fv.visitEnd();
    }
  }

  private static class MethodTimerMVisitor extends MethodVisitor {
    /**
         * 当前类名
         */
    private final String owner;

    protected MethodTimerMVisitor(int api, MethodVisitor methodVisitor, String owner) {
      super(api, methodVisitor);
      this.owner = owner;
    }

    @Override
    public void visitCode() {
      super.visitFieldInsn(GETSTATIC, owner, "timer", "J");
      super.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
      super.visitInsn(LSUB);
      super.visitFieldInsn(PUTSTATIC, owner, "timer", "J");
      // 继续调用 mv成员变量的 visitCode()方法
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      if (opcode >= IRETURN && opcode <= RETURN || opcode == ATHROW) {
        super.visitFieldInsn(GETSTATIC, owner, "timer", "J");
        super.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        super.visitInsn(LADD);
        super.visitFieldInsn(PUTSTATIC, owner, "timer", "J");
      }
      // 继续调用 mv成员变量的 visitInsn()方法
      super.visitInsn(opcode);
    }
  }
}
```

#### 进行转换

```java
package com.example;

import core.*;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodTimerVisitor(api, cw);

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}

```

#### 验证结果

```java
import java.lang.reflect.Field;
import java.util.Random;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    // 第一部分，先让“子弹飞一会儿”，让程序运行一段时间
    HelloWorld instance = new HelloWorld();
    Random rand = new Random(System.currentTimeMillis());
    for (int i = 0; i < 10; i++) {
      boolean flag = rand.nextBoolean();
      int a = rand.nextInt(50);
      int b = rand.nextInt(50);
      if (flag) {
        int c = instance.add(a, b);
        String line = String.format("%d + %d = %d", a, b, c);
        System.out.println(line);
      }
      else {
        int c = instance.sub(a, b);
        String line = String.format("%d - %d = %d", a, b, c);
        System.out.println(line);
      }
    }

    // 第二部分，来查看方法运行的时间
    Class<?> clazz = HelloWorld.class;
    Field[] declaredFields = clazz.getDeclaredFields();
    for (Field f : declaredFields) {
      String fieldName = f.getName();
      f.setAccessible(true);
      if (fieldName.startsWith("timer")) {
        Object FieldValue = f.get(null);
        System.out.println(fieldName + " = " + FieldValue);
      }
    }
  }
}
```

### 3.7.3 实现二：计算每个方法的运行时间 

#### 编码实现

注意点：

+ 这里针对每个方法都生成`timer_XXX`字段，所以在ClassVisitor的`visitMethod()`方法内生成字段
+ 这里MethodVisitor需要感知自己的方法名，所以需要在构造方法里要求传递方法名

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

/**
 * 目标：计算所有方法的执行耗时
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/29 9:08 PM
 */
public class MethodTimerVisitor2 extends ClassVisitor {
  /**
     * 当前类名
     */
  private String owner;
  /**
     * 当前类是否是接口
     */
  private boolean isInterface;

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    owner = name;
    isInterface = (access & ACC_INTERFACE) == ACC_INTERFACE;
  }

  public MethodTimerVisitor2(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (isInterface || null == mv
        || ((access & ACC_ABSTRACT) == ACC_ABSTRACT) || ((access & ACC_NATIVE) == ACC_NATIVE)
        || "<init>".equals(name) || "<clinit>".equals(name)) {
      return mv;
    }
    // 针对每个方法，都生成一个timer字段用于统计执行耗时
    // 给指定类生成timer成员变量(暂不考虑已经存在的情况), 计算方法执行耗时
    FieldVisitor fv = super.visitField(ACC_PUBLIC | ACC_STATIC, genFieldName(name), "J", null, null);
    if (null != fv) {
      fv.visitEnd();
    }
    return new MethodTimerMVisitor(api, mv, owner, name);
  }

  private String genFieldName(String methodName) {
    return "timer_" + methodName;
  }

  private class MethodTimerMVisitor extends MethodVisitor {
    /**
         * 当前类名
         */
    private final String owner;
    /**
         * 当前方法名
         */
    private final String methodName;

    protected MethodTimerMVisitor(int api, MethodVisitor methodVisitor, String owner,String methodName) {
      super(api, methodVisitor);
      this.owner = owner;
      this.methodName = methodName;
    }

    @Override
    public void visitCode() {
      super.visitFieldInsn(GETSTATIC, owner, genFieldName(methodName), "J");
      super.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
      super.visitInsn(LSUB);
      super.visitFieldInsn(PUTSTATIC, owner, genFieldName(methodName), "J");
      // 继续调用 mv成员变量的 visitCode()方法
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      if (opcode >= IRETURN && opcode <= RETURN || opcode == ATHROW) {
        super.visitFieldInsn(GETSTATIC, owner, genFieldName(methodName), "J");
        super.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        super.visitInsn(LADD);
        super.visitFieldInsn(PUTSTATIC, owner, genFieldName(methodName), "J");
      }
      // 继续调用 mv成员变量的 visitInsn()方法
      super.visitInsn(opcode);
    }
  }
}

```

### 3.7.4 总结

本文主要介绍了如何计算方法的运行时间，内容总结如下：

- 第一点，从实现思路的角度来说，计算方法的运行时间，是在“方法进入”和“方法退出”的基础上实现的。在“方法进入”的时候，减去一个时间戳；在“方法退出”的时候，加上一个时间戳。
- 第二点，我们提供了两种实现方式
  - 第一种实现方式，计算类里面所有方法的总运行时间
  - 第二种实现方式，计算类里面每个方法的单独运行时间

其实，遵循同样的思路，我们也可以计算方法运行的总次数；再进一步，我们可以计算出方法多次运行后的平均执行时间。

## 3.8 修改已有的方法（删除－移除Instruction）

### 3.8.1 如何移除Instruction

在修改方法体的代码时，**如何移除一条Instruction呢**？其实，很简单，就是**让中间的某一个`MethodVisitor`对象不向后“传递该instruction”就可以了**。

![Java ASM系列：（027）修改已有的方法（删除－移除Instruction）_Core API](https://s2.51cto.com/images/20210628/1624882148334418.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

但是，需要要注意一点：**无论是添加instruction，还是删除instruction，还是要替换instruction，都要保持operand stack修改前和修改后是一致的**。这句话该怎么理解呢？我们举个例子来进行说明。

假如，有一条打印语句，如下：

```java
System.out.println("Hello World");
```

这条打印语句，对应着三个instruction，如下：

```java
GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
LDC "Hello World"
INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
```

上面三条instruction的执行过程如下：

- 第一步，执行`GETSTATIC java/lang/System.out : Ljava/io/PrintStream;`，会把一个`System.out`对象push到operand stack上。
- 第二步，执行`LDC "Hello World"`，会将一个字符串对象push到operand stack上。
- 第三步，执行`INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V`，会消耗掉operand stack栈顶上的两个元素，然后打印出结果。

<u>如果我们只想删除第三条`INVOKEVIRTUAL`对应的instruction，是不行的，因为它会让operand stack栈顶上多出两个元素。这三条instruction应该一起删除，才能保证operand stack在修改前和修改后是一致的</u>。

### 3.8.2 示例：移除NOP

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.nop)
>
> nop: Do nothing.

为了让示例容易一些，我们先来处理一个比较简单的情况，那就是移除`NOP`指令。那么，为什么要移除`NOP`指令呢？因为`NOP`表示no operation，它是一个单独的指令，本身也不做什么操作，我们删除它不会影响任何实质性的操作，也不会牵连其它的instruction。

当然，一般情况下，由`.java`编译生成的`.class`文件中不会包含`NOP`指令。那么，我们就自己生成一个`.class`文件，让它带有`NOP`指令。

#### 预期目标

我们想实现的预期目标：删除代码当中的`NOP`指令。

首先，我们来生成一个包含`NOP`指令的`.class`文件，如下：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateCore {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用visitXxx()方法
    cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld",
             null, "java/lang/Object", null);

    {
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      mv1.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      mv1.visitMaxs(1, 1);
      mv1.visitEnd();
    }

    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv2.visitCode();
      mv2.visitInsn(NOP);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitInsn(NOP);
      mv2.visitLdcInsn("Hello World");
      mv2.visitInsn(NOP);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitInsn(NOP);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 1);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

查看生成后的效果：

```java
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
  public sample.HelloWorld();
    Code:
       0: aload_0
       1: invokespecial #8                  // Method java/lang/Object."<init>":()V
       4: return

  public void test();
    Code:
       0: nop
       1: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
       4: nop
       5: ldc           #17                 // String Hello World
       7: nop
       8: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      11: nop
      12: return
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

public class MethodRemoveNopVisitor extends ClassVisitor {
  public MethodRemoveNopVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodRemoveNopAdapter(api, mv);
      }

    }
    return mv;
  }

  private static class MethodRemoveNopAdapter extends MethodVisitor {
    public MethodRemoveNopAdapter(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitInsn(int opcode) {
      // if (opcode == NOP) {
      //     // do nothing
      // }
      // else {
      //     super.visitInsn(opcode);
      // }
      if (opcode != NOP) {
        super.visitInsn(opcode);
      }
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodRemoveNopVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
  public sample.HelloWorld();
    Code:
       0: aload_0
       1: invokespecial #8                  // Method java/lang/Object."<init>":()V
       4: return

  public void test();
    Code:
       0: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #17                 // String Hello World
       5: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

### 3.8.3 总结

本文主要对移除Instruction进行了介绍，内容总结如下：

- 第一点，移除Instruction方式，就是让中间的某一个`MethodVisitor`对象不向后“传递该instruction”就可以了。
- 第二点，**在移除instruction的过程中，要保证operand stack在修改前和修改后是一致的**。

在后面的内容当中，我们会介绍到如何删除打印语句，因为它要经历一个“模式识别”的过程，相对复杂一些，所以我们放到后面的内容再来讨论。

## 3.9 修改已有的方法（删除－清空方法体）

### 3.9.1 如何清空方法体

在有些情况下，我们可能想清空整个方法体的内容，那该怎么做呢？其实，有两个思路。

- 第一种思路，就是将instruction一条一条的移除掉，直到最后只剩下return语句。（不推荐）
- **第二种思路，就是忽略原来的方法体，重新生成一个新的方法体。（推荐使用）**

![Java ASM系列：（028）修改已有的方法（删除－清空方法体）_Core API](https://s2.51cto.com/images/20210628/1624882148334418.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

对于第二种思路，“忽略原来的方法体，重新生成一个新的方法体”，想法很好，具体如何实现呢？假设有一个中间的`MethodVisitor`来负责做这个工作，通过两个步骤来实现：

- 第一步，对于它“前面”的`MethodVisitor`，它返回`null`值，就相当于原来的方法丢失了；
- 第二步，对于它“后面”的`MethodVisitor`，它添加同名、同类型的方法，然后生成新的方法体，这就相当于又添加了一个新的方法。

需要注意的一点：**清空方法体，并不是一条instruction也没有，它至少要有一条return语句。**

- 如果方法返回值是`void`类型，那至少要有一个return；
- 如果方法返回值不是`void`类型（例如，`int`、`String`），这个时候，就要考虑返回一个什么样的值比较合适了。

同时，我们也要**计算local variables和operand stack的大小**：

- 计算local variables的大小。在local variables中，主要是用于存储`this`变量和方法的参数，只要计算`this`和方法参数的大小就可以了。
- 计算operand stack的大小。
  - 如果方法有返回值，则需要先放到operand stack上去，再进行返回，那么operand stack的大小与返回值的类型密切相关；
  - 如果方法没有返回值，清空方法体后，那么operand stack的大小为`0`。

计算local variables和operand stack的大小，可以由我们自己编码来实现，也可以由ASM帮助我们实现。

### 3.9.2. 示例：绕过验证机制

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void verify(String username, String password) throws IllegalArgumentException {
    if ("tomcat".equals(username) && "123456".equals(password)) {
      return;
    }
    throw new IllegalArgumentException("username or password is not correct");
  }
}
```

我们想实现的预期目标：清空`verify`方法的方法体，无论输入什么样的值，它都不会报错。

```java
public class HelloWorld {
  public void verify(String username, String password) throws IllegalArgumentException {
    return;
  }
}
```

#### 编码实现

注意点：

+ 清空方法体，这里visitMethod遇到指定方法时，先给下一个MethodVisitor增加同名方法，然后当前返回null，以达到当前MethodVisitor删除方法，而下一个MethodVisitor增加同名方法（空方法体，仅保留return语句）的效果。

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;

import static org.objectweb.asm.Opcodes.*;

/**
 * 目标：清空 HelloWorld 类的 verify 方法的方法体
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/30 9:51 AM
 */
public class MethodEmptyBodyVisitor extends ClassVisitor {
  /**
     * 当前类名
     */
  private String owner;
  /**
     * 需要被清空方法体的方法名
     */
  private String methodName;
  /**
     * 需要被清空方法体的方法的描述符
     */
  private String methodDescriptor;

  public MethodEmptyBodyVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDescriptor) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDescriptor = methodDescriptor;
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    // 不需要特输处理的情况
    if (null == mv || (access & ACC_ABSTRACT) == ACC_ABSTRACT || (access & ACC_NATIVE) == ACC_NATIVE
        || "<init>".equals(name) || "<clinit>".equals(name)) {
      return mv;
    }
    // 找到需要 清空方法体的方法
    if (methodName.equals(name) && methodDescriptor.equals(descriptor)) {
      // 给下一个 MethodVisitor增加同名方法
      generateNewBody(mv, owner, access, name, descriptor);
      // 当前MethodVisitor忽略(删除)该方法
      return null;
    }
    // 其他正常方法，不做处理
    return mv;
  }

  /**
     * 给当前类绑定的 下一个 MethodVisitor 添加同名方法，仅保留return逻辑 <br/>
     * 需要判断原本的返回值类型，做不同的返回处理
     */
  private void generateNewBody(MethodVisitor mv, String owner, int methodAccess, String methodName, String methodDesc) {
    // (1) method argument types and return type
    Type methodType = Type.getType(methodDescriptor);
    Type[] argumentTypes = methodType.getArgumentTypes();
    Type returnType = methodType.getReturnType();

    // (2) compute the size of local variables and operand stack
    boolean isStaticMethod = (methodAccess & ACC_STATIC) == ACC_STATIC;
    // non-static 方法的local variables 下标0存放this指针
    int localSize = isStaticMethod ? 0 : 1;
    for(Type argType : argumentTypes) {
      localSize += argType.getSize();
    }
    int stackSize = returnType.getSize();

    // (3) method body
    mv.visitCode();
    if (returnType.getSort() == Type.VOID) {
      mv.visitInsn(RETURN);
    }
    else if (returnType.getSort() >= Type.BOOLEAN && returnType.getSort() <= Type.INT) {
      mv.visitInsn(ICONST_1);
      mv.visitInsn(IRETURN);
    }
    else if (returnType.getSort() == Type.LONG) {
      mv.visitInsn(LCONST_0);
      mv.visitInsn(LRETURN);
    }
    else if (returnType.getSort() == Type.FLOAT) {
      mv.visitInsn(FCONST_0);
      mv.visitInsn(FRETURN);
    }
    else if (returnType.getSort() == Type.DOUBLE) {
      mv.visitInsn(DCONST_0);
      mv.visitInsn(DRETURN);
    }
    else {
      mv.visitInsn(ACONST_NULL);
      mv.visitInsn(ARETURN);
    }
    mv.visitMaxs(stackSize, localSize);
    mv.visitEnd();
  }
}
```

#### 进行转换

```java
package com.example;

import core.*;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodEmptyBodyVisitor(api, cw, "verify", "(Ljava/lang/String;Ljava/lang/String;)V");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.verify("jerry", "123");
  }
}
```

### 3.9.3 总结

本文主要对清空方法体进行了介绍，内容总结如下：

- 第一点，**清空方法体，它的思路是，忽略原来的方法体，然后重新生成新的方法体**。
- 第二点，清空方法体过程中，要注意的事情是，方法体当中要包含return相关的语句，同时要计算local variables和operand stack的大小。

## 3.10 修改已有的方法（修改－替换方法调用）

### 3.10.1 如何替换Instruction

有的时候，我们想替换掉某一条instruction，那应该如何实现呢？其实，实现起来也很简单，就是**先找到该instruction，然后在同样的位置替换成另一个instruction就可以了**。

![Java ASM系列：（029）修改已有的方法（修改－替换方法调用）_ByteCode](https://s2.51cto.com/images/20210628/1624882148334418.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

同样，我们也要注意：**在替换instruction的过程当中，operand stack在修改前和修改后是一致的**。

在方法当中，替换Instruction，有什么样的使用场景呢？比如说，第三方提供的jar包当中，可能在某一个`.class`文件当中调用了一个方法。这个方法，从某种程度上来说，你可能对它“不满意”。假如说，这个方法是一个验证逻辑的方法，你想替换成自己的验证逻辑，又或者说，它实现的功能比较简单，你想替换成功能更完善的方法，就可以把这个方法对应的Instruction替换掉。

### 3.10.2 示例：替换方法调用

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public void test(int a, int b) {
        int c = Math.max(a, b);
        System.out.println(c);
    }
}
```

其中，`test()`方法对应的指令如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: invokestatic  #2                  // Method java/lang/Math.max:(II)I
       5: istore_3
       6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: iload_3
      10: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      13: return
}
```

我们想实现的预期目标有两个：

- 第一个，就是将静态方法`Math.max()`方法替换掉。
- 第二个，就是将非静态方法`PrintStream.println()`方法替换掉。

#### 编码实现

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.ACC_ABSTRACT;
import static org.objectweb.asm.Opcodes.ACC_NATIVE;

public class MethodReplaceInvokeVisitor extends ClassVisitor {
  private final String oldOwner;
  private final String oldMethodName;
  private final String oldMethodDesc;

  private final int newOpcode;
  private final String newOwner;
  private final String newMethodName;
  private final String newMethodDesc;

  public MethodReplaceInvokeVisitor(int api, ClassVisitor classVisitor,
                                    String oldOwner, String oldMethodName, String oldMethodDesc,
                                    int newOpcode, String newOwner, String newMethodName, String newMethodDesc) {
    super(api, classVisitor);
    this.oldOwner = oldOwner;
    this.oldMethodName = oldMethodName;
    this.oldMethodDesc = oldMethodDesc;

    this.newOpcode = newOpcode;
    this.newOwner = newOwner;
    this.newMethodName = newMethodName;
    this.newMethodDesc = newMethodDesc;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodReplaceInvokeAdapter(api, mv);
      }
    }
    return mv;
  }

  private class MethodReplaceInvokeAdapter extends MethodVisitor {
    public MethodReplaceInvokeAdapter(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
      if (oldOwner.equals(owner) && oldMethodName.equals(name) && oldMethodDesc.equals(descriptor)) {
        // 注意，最后一个参数是false，会不会太武断呢？
        super.visitMethodInsn(newOpcode, newOwner, newMethodName, newMethodDesc, false);
      }
      else {
        super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
      }
    }
  }
}
```

在`MethodReplaceInvokeAdapter`类当中，`visitMethodInsn()`方法有这么一行代码：

```java
// 注意，最后一个参数是false，会不会太武断呢？
super.visitMethodInsn(newOpcode, newOwner, newMethodName, newMethodDesc, false);
```

在`visitMethodInsn()`方法中，最后一个参数是`boolean isInterface`，它可以取值为`true`，也可以取值为`false`。如果它的值为`true`，表示调用的方法是一个接口里的方法；如果它的值为`false`，则表示调用的方法是类里面的方法。换句话说，这个`boolean isInterface`参数，本来可以有两个可选值，即`true`或`false`；但是，我们直接提供一个固定值`false`，完成没有考虑`true`的情况，这么做是不是太过武断了呢？

之所以要这么做，是因为一般情况下，替换后的方法是“自己写的某一个方法”，那么对于“这个方法”，我们有很大的“自主权”，可以把它放到一个接口里，也可以放在一个类里，可以把它写成一个non-static方法，也可以写成一个static方法。这样，我们完全可以把“这个方法”写成一个在类里的static方法。

#### 进行转换

##### 替换static方法

在替换static方法的时候，要保证一点：**替换方法前，和替换方法后，要保持“方法接收的参数”和“方法的返回类型”是一致的**。

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
    public static void main(String[] args) {
        String relative_path = "sample/HelloWorld.class";
        String filepath = FileUtils.getFilePath(relative_path);
        byte[] bytes1 = FileUtils.readBytes(filepath);

        //（1）构建ClassReader
        ClassReader cr = new ClassReader(bytes1);

        //（2）构建ClassWriter
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

        //（3）串连ClassVisitor
        int api = Opcodes.ASM9;
        ClassVisitor cv = new MethodReplaceInvokeVisitor(api, cw,
                "java/lang/Math", "max", "(II)I",
                Opcodes.INVOKESTATIC, "java/lang/Math", "min", "(II)I");

        //（4）结合ClassReader和ClassVisitor
        int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
        cr.accept(cv, parsingOptions);

        //（5）生成byte[]
        byte[] bytes2 = cw.toByteArray();

        FileUtils.writeBytes(filepath, bytes2);
    }
}
```

##### 替换non-static方法

对于non-static方法来说，它有一个隐藏的`this`变量。我们在替换non-static方法的时候，要把`this`变量给“消耗”掉。

```java
public class ParameterUtils {
  public static void output(PrintStream printStream, int val) {
    printStream.println("ParameterUtils: " + val);
  }
}
```

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
    public static void main(String[] args) {
        String relative_path = "sample/HelloWorld.class";
        String filepath = FileUtils.getFilePath(relative_path);
        byte[] bytes1 = FileUtils.readBytes(filepath);

        //（1）构建ClassReader
        ClassReader cr = new ClassReader(bytes1);

        //（2）构建ClassWriter
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

        //（3）串连ClassVisitor
        int api = Opcodes.ASM9;
        ClassVisitor cv = new MethodReplaceInvokeVisitor(api, cw,
                "java/io/PrintStream", "println", "(I)V",
                Opcodes.INVOKESTATIC, "sample/ParameterUtils", "output", "(Ljava/io/PrintStream;I)V");

        //（4）结合ClassReader和ClassVisitor
        int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
        cr.accept(cv, parsingOptions);

        //（5）生成byte[]
        byte[] bytes2 = cw.toByteArray();

        FileUtils.writeBytes(filepath, bytes2);
    }
}
```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test(10, 20);
  }
}
```

### 3.10.3 总结

本文主要对替换Instruction进行了介绍，内容总结如下：

- 第一点，替换Instruction，实现思路是，先找到该instruction，然后在同样的位置替换成另一个instruction就可以了。
- 第二点，替换Instruction，要注意的地方是，保持operand stack在修改前和修改后是一致的。对于static方法和non-static方法，我们需要考虑是否要处理`this`变量。

其实，按照相同的思路，我们也可以将“对于字段的调用”替换成“对于方法的调用”。

## 3.11 查找已有的方法（查找－方法调用）

### 3.11.1 查找Instruction

#### 如何查找Instruction

在方法当中，查找某一个特定的Instruction，那么应该怎么做呢？简单来说，就是**通过`MethodVisitor`类当中定义的`visitXxxInsn()`方法来查找**。

让我们回顾一下`MethodVisitor`类当中定义了哪些`visitXxx()`方法。

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_ClassFile](https://s2.51cto.com/images/20210629/1624967433536552.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在`MethodVisitor`类当中，定义的主要`visitXxx()`方法可以分成四组：

- 第一组，`visitCode()`方法，标志着方法体（method body）的开始。
- 第二组，`visitXxxInsn()`方法，对应方法体（method body）本身，这里包含多个方法。
- 第三组，`visitMaxs()`方法，标志着方法体（method body）的结束。
- 第四组，`visitEnd()`方法，是最后调用的方法。

在方法当中，任何一条Instruction，放在ASM代码中，它都是通过调用`MethodVisitor.visitXxxInsn()`方法的形式来呈现的。换句话说，想去找某一条特定的Instruction，分成两个步骤：

- 第一步，找到该Instruction对应的`visitXxxInsn()`方法。
- 第二步，对该`visitXxxInsn()`方法接收的`opcode`和其它参数进行判断。

**简而言之，查找Instruction的过程，就是对`visitXxxInsn()`方法接收的参数进行检查的过程。** 举一个形象的例子，平时我们坐地铁，随身物品要过安检，其实就是对书包（方法）里的物品（参数）进行检查，如下图：

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_Java_02](https://s2.51cto.com/images/20210701/1625137506187777.jpg?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

#### Class Analysis

**查找Instruction的过程，** 并不属于Class Transformation（因为没有生成新的类），而**是属于Class Analysis。** 在下图当中，Class Analysis包括find potential bugs、detect unused code和reverse engineer code等操作。但是，这些分析操作（analysis）是比较困难的，它需要编程经验的积累和对问题模式的识别，需要编码处理各种不同情况，所以不太容易实现。

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_ASM_03](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

但是，Class Analysis，并不是只包含复杂的分析操作，也包含一些简单的分析操作。例如，当前方法里调用了哪些其它的方法、当前的方法被哪些别的方法所调用。对于方法的调用，就对应着`MethodVisitor.visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface)`方法。

另外，要注意一点：**在Class Transformation当中，需要用到`ClassReader`、`ClassVisitor`和`ClassWriter`类；但是，在Class Analysis中，我们只需要用到`ClassReader`和`ClassVisitor`类，而不需要用到`ClassWriter`类**。

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_Java_04](https://s2.51cto.com/images/20210618/1624028333369109.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.11.2 示例一：调用了哪些方法

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int a, int b) {
    int c = Math.addExact(a, b);
    String line = String.format("%d + %d = %d", a, b, c);
    System.out.println(line);
  }
}
```

我们想要实现的预期目标：打印出`test()`方法当中调用了哪些方法。

在编写ASM代码之前，可以使用`javap`命令查看`test()`方法所包含的Instruction内容：

```java
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: invokestatic  #2                  // Method java/lang/Math.addExact:(II)I
       5: istore_3
       6: ldc           #3                  // String %d + %d = %d
       8: iconst_3
       9: anewarray     #4                  // class java/lang/Object
      12: dup
      13: iconst_0
      14: iload_1
      15: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      18: aastore
      19: dup
      20: iconst_1
      21: iload_2
      22: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      25: aastore
      26: dup
      27: iconst_2
      28: iload_3
      29: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      32: aastore
      33: invokestatic  #6                  // Method java/lang/String.format:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;
      36: astore        4
      38: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      41: aload         4
      43: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      46: return
}
```

#### 编码实现

注意点：

+ 这里做的是code analysis，所以只需要ClassReader和ClassVisitor，不需要ClassWriter

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.util.Printer;

import java.util.ArrayList;
import java.util.List;

/**
 * 目标，打印指定的方法在方法体中 调用到的方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/30 2:13 PM
 */
public class MethodFindInvokeVisitor extends ClassVisitor {

  private final String methodName;
  private final String methodDescriptor;

  public MethodFindInvokeVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDescriptor) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDescriptor = methodDescriptor;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    return name.equals(methodName) && descriptor.equals(methodDescriptor) ? new MethodFindInvokeMVisitor(api, mv) : mv;
  }

  private static class MethodFindInvokeMVisitor extends MethodVisitor {
    /**
         * 方法的方法体内调用过的方法(已去重)
         */
    private List<String> methodCallList = new ArrayList<>();

    protected MethodFindInvokeMVisitor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
      String info = String.format("%s %s.%s%s", Printer.OPCODES[opcode], owner, name, descriptor);
      if (!methodCallList.contains(info)) {
        methodCallList.add(info);
      }
      super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
    }

    @Override
    public void visitEnd() {
      // 输出指定方法的方法体中 调用过的方法
      for(String methodCall : methodCallList) {
        System.out.println(methodCall);
      }
      super.visitEnd();
    }
  }
}

```

#### 进行分析

```java
package com.example;

import core.*;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    //        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor (这里是进行 Code Analysis，不需要ClassWriter)
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodFindInvokeVisitor(api, null, "test", "(II)V");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    //        byte[] bytes2 = cw.toByteArray();

    //        FileUtils.writeBytes(filepath, bytes2);
  }
}
```

输出结果：

```shell
INVOKESTATIC java/lang/Math.addExact(II)I
INVOKESTATIC java/lang/Integer.valueOf(I)Ljava/lang/Integer;
INVOKESTATIC java/lang/String.format(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;
INVOKEVIRTUAL java/io/PrintStream.println(Ljava/lang/String;)V
```

### 3.11.3 示例二：被哪些方法所调用

在IDEA当中，有一个Find Usages功能：在类名、字段名、或方法名上，右键之后，选择Find Usages，就可以查看该项内容在哪些地方被使用了。

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_ByteCode_05](https://s2.51cto.com/images/20210701/1625137638539075.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

查找结果如下：

![Java ASM系列：（030）查找已有的方法（查找－方法调用）_ClassFile_06](https://s2.51cto.com/images/20210701/1625137667714870.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

这样一个功能，如果我们自己来实现，那该怎么编写ASM代码呢？

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public int add(int a, int b) {
    int c = a + b;
    test(a, b, c);
    return c;
  }

  public int sub(int a, int b) {
    int c = a - b;
    test(a, b, c);
    return c;
  }

  public int mul(int a, int b) {
    return a * b;
  }

  public int div(int a, int b) {
    return a / b;
  }

  public void test(int a, int b, int c) {
    String line = String.format("a = %d, b = %d, c = %d", a, b, c);
    System.out.println(line);
  }
}
```

我们想要实现的预期目标：找出是哪些方法对`test()`方法进行了调用。

#### 编码实现

```java
package core;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import java.util.ArrayList;
import java.util.List;

import static org.objectweb.asm.Opcodes.ACC_ABSTRACT;
import static org.objectweb.asm.Opcodes.ACC_NATIVE;

/**
 * 目标：输出 调用过 指定的A方法的 B、C、D等调用方的方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/30 2:31 PM
 */
public class MethodFindRefVisitor extends ClassVisitor {
  /**
     * 被调用方-类名(需要知道方法的归属)
     */
  private final String targetClassName;
  /**
     * 被调用方-方法名
     */
  private final String targetMethodName;
  /**
     * 被调用方-方法描述符
     */
  private final String targetMethodDescriptor;
  /**
     * 当前类-类名
     */
  private String currentClassName;
  /**
     * 方法调用记录集合(去重)
     */
  private List<String> methodCallList = new ArrayList<>();

  public MethodFindRefVisitor(int api, ClassVisitor classVisitor, String targetClassName, String targetMethodName, String targetMethodDescriptor) {
    super(api, classVisitor);
    this.targetClassName = targetClassName;
    this.targetMethodName = targetMethodName;
    this.targetMethodDescriptor = targetMethodDescriptor;
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.currentClassName = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    return (access & ACC_ABSTRACT) == ACC_ABSTRACT || (access & ACC_NATIVE) == ACC_NATIVE ? mv : new MethodFindRefMVisitor(api, mv, currentClassName, name, descriptor);
  }

  @Override
  public void visitEnd() {
    // 输出 所有调用过 目标方法 的记录
    for (String methodCall : methodCallList) {
      System.out.println(methodCall);
    }
    super.visitEnd();
  }

  private class MethodFindRefMVisitor extends MethodVisitor {
    /**
         * 调用方-类名
         */
    private final String currentClassName;
    /**
         * 调用方-方法名
         */
    private final String currentMethodName;
    /**
         * 调用方-方法描述符
         */
    private final String currentMethodDescriptor;

    protected MethodFindRefMVisitor(int api, MethodVisitor methodVisitor, String currentClassName, String currentMethodName, String currentMethodDescriptor) {
      super(api, methodVisitor);
      this.currentClassName = currentClassName;
      this.currentMethodName = currentMethodName;
      this.currentMethodDescriptor = currentMethodDescriptor;
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
      // 指定的方法被调用，则记录当前方法信息
      if (owner.equals(targetClassName) && name.equals(targetMethodName) && descriptor.equals(targetMethodDescriptor)) {
        String info = String.format("%s.%s%s", currentClassName, currentMethodName, currentMethodDescriptor);
        if (!methodCallList.contains(info)) {
          methodCallList.add(info);
        }
      }
      super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
    }

  }
}
```

#### 进行分析

```java
package com.example;

import core.*;
import org.objectweb.asm.*;
import utils.FileUtils;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/4/7 10:34 PM
 */
public class HelloWorldRun {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassVisitor
    //        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3) 串联ClassVisitor  (这里是进行 Code Analysis，不需要ClassWriter)
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodFindRefVisitor(api, null, "sample/HelloWorld", "test", "(III)V");

    // (4) 结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    // (5) 生成byte[]
    //        byte[] bytes2 = cw.toByteArray();

    //        FileUtils.writeBytes(filepath, bytes2);
  }
}
```

输出结果：

```shell
sample/HelloWorld.add(II)I
sample/HelloWorld.sub(II)I
```

### 3.11.4 总结

本文主要对查找Instruction进行了介绍，内容总结如下：

- 第一点，**查找Instruction的过程，就是对`MethodVisitor`类的`visitXxxInsn()`方法及参数进行判断的过程**。
- 第二点，**查找Instruction，并不属于Class Transformation，而属于Class Analysis。Class Analysis，只需要用到`ClassReader`和`ClassVisitor`类，而不需要用到`ClassWriter`类**。
- 第三点，在两个代码示例中，都是围绕着“方法调用”展开，而“方法调用”就对应着`MethodVisitor.visitMethodInsn()`方法。

## 3.12 修改已有的方法（优化－删除－复杂的变换）

### 3.12.1 复杂的变换

#### stateless transformations

The stateless transformation does not depend on **the instructions that have been visited before the current one**.

举几个关于stateless transformation的例子：

- 添加指令：在方法进入和方法退出时，打印方法的参数和返回值、计算方法的运行时间。
- 删除指令：移除NOP、清空方法体。
- 修改指令：替换调用的方法。

这种stateless transformation实现起来比较容易，所以也被称为simple transformations。

#### stateful transformations

The **stateful transformation** require memorizing **some state** about **the instructions that have been visited before the current one**. This requires storing state inside the method adapter.

举几个关于stateful transformation的例子：

- 删除指令：移除`ICONST_0 IADD`。例如，`int d = c + 0;`与`int d = c;`两者效果是一样的，所以`+ 0`的部分可以删除掉。
- 删除指令：移除`ALOAD_0 ALOAD_0 GETFIELD PUTFIELD`。例如，`this.val = this.val;`，将字段的值赋值给字段本身，无实质意义。
- 删除指令：移除`GETSTATIC LDC INVOKEVIRTUAL`。例如，`System.out.println("Hello World");`，删除打印信息。

这种stateful transformation实现起来比较困难，所以也被称为complex transformations。

那么，为什么stateless transformation实现起来比较容易，而stateful transformation会实现起来比较困难呢？做个类比，stateless transformation就类似于“一人吃饱，全家不饿”，不用考虑太多，所以实现起来就比较简单；而stateful transformation类似于“成家之后，要考虑一家人的生活状态”，考虑的事情就多一点，所以实现起来就比较困难。难归难，但是我们还是应该想办法进行实现。

那么，stateful transformation到底该如何开始着手实现呢？在stateful transformation过程中，一般都是涉及到对多个指令（Instruction）同时判断，这多个指令是一个“组合”，不能轻易拆散。我们通过三个步骤来进行实现：

- **第一步，就是将问题本身转换成Instruction指令，然后对多个指令组合的“特征”或遵循的“模式”进行总结。**
- 第二步，就是使用这个总结出来的“特征”或“模式”对指令进行识别。在识别的过程当中，每一条Instruction的加入，都会引起原有状态（state）的变化，这就对应着`stateful`的部分；
- 第三步，识别成功之后，要对Class文件进行转换，这就对应着`transformation`的部分。谈到transformation，无非就是对Instruction的内容进行增加、删除和修改等操作。

到这里，就有一个新的问题产生：如何去记录第二步当中的状态（state）变化呢？我们的回答就是，借助于state machine(有限状态机)。

#### state machine

首先，我们回答一个问题：什么是state machine？

A state machine is a behavior model. It consists of **a finite number of states** and is therefore also called finite-state machine (FSM). Based on the **current state** and **a given input** the machine performs **state transitions** and produces outputs.

对于state machine，我想到了这句话：“吾生也有涯，而知也无涯。以有涯随无涯，殆已”。这句话的意思是讲，人们的生命是有限的，而知识却是无限的。以有限的生命去追求无限的知识，势必体乏神伤。我觉得，state machine的聪明之处，就是将“无限”的操作步骤给限定在“有限”的状态里来思考。

接下来，就是给出一个具体的state machine。也就是说，下面的`MethodPatternAdapter`类，就是一个原始的state machine，我们从三个层面来把握它：

- 第一个层面，class info。`MethodPatternAdapter`类，继承自`MethodVisitor`类，本身也是一个抽象类。
- 第二个层面，fields。`MethodPatternAdapter`类，定义了两个字段，其中`SEEN_NOTHING`字段，是一个常量值，表示一个“初始状态”，而`state`字段则是用于记录不断变化的状态。
- 第三个层面，methods。<u>`MethodPatternAdapter`类定义`visitXxxInsn()`方法，都会去调用一个自定义的`visitInsn()`方法。`visitInsn()`方法，是一个抽象方法，它的作用就是让所有的其它状态（state）都回归“初始状态”</u>。

那么，应该怎么使用`MethodPatternAdapter`类呢？我们就是写一个`MethodPatternAdapter`类的子类，这个子类就是一个更“先进”的state machine，它做以下三件事情：

- 第一件事情，从字段层面，根据处理的问题，来定义更多的状态；也就是，类似于`SEEN_NOTHING`的字段。这里就是**对a finite number of states进行定义**。
- 第二件事情，从方法层面，处理好`visitXxxInsn()`的调用，对于`state`字段状态的影响。也就是，输入新的指令（Instruction），都会对`state`字段产生影响。这里就是**构建状态（state）变化的机制**。
- 第三件事情，从方法层面，实现`visitInsn()`方法，根据`state`字段的值，如何回归到“初始状态”。这里就是添加一个“恢复出厂设置”的功能，**让状态（state）归零**，回归到一个初始状态。**让状态（state）归零**，是**构建状态（state）变化的机制**一个比较特殊的环节。结合生活来说，生活中有不顺的地方，就从新开始，从零开始。

```java
import org.objectweb.asm.Handle;
import org.objectweb.asm.Label;
import org.objectweb.asm.MethodVisitor;

public abstract class MethodPatternAdapter extends MethodVisitor {
  protected final static int SEEN_NOTHING = 0;
  protected int state;

  public MethodPatternAdapter(int api, MethodVisitor methodVisitor) {
    super(api, methodVisitor);
  }

  @Override
  public void visitInsn(int opcode) {
    visitInsn();
    super.visitInsn(opcode);
  }

  @Override
  public void visitIntInsn(int opcode, int operand) {
    visitInsn();
    super.visitIntInsn(opcode, operand);
  }

  @Override
  public void visitVarInsn(int opcode, int var) {
    visitInsn();
    super.visitVarInsn(opcode, var);
  }

  @Override
  public void visitTypeInsn(int opcode, String type) {
    visitInsn();
    super.visitTypeInsn(opcode, type);
  }

  @Override
  public void visitFieldInsn(int opcode, String owner, String name, String descriptor) {
    visitInsn();
    super.visitFieldInsn(opcode, owner, name, descriptor);
  }

  @Override
  public void visitMethodInsn(int opcode, String owner, String name, String descriptor) {
    visitInsn();
    super.visitMethodInsn(opcode, owner, name, descriptor);
  }

  @Override
  public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
    visitInsn();
    super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
  }

  @Override
  public void visitInvokeDynamicInsn(String name, String descriptor, Handle bootstrapMethodHandle, Object... bootstrapMethodArguments) {
    visitInsn();
    super.visitInvokeDynamicInsn(name, descriptor, bootstrapMethodHandle, bootstrapMethodArguments);
  }

  @Override
  public void visitJumpInsn(int opcode, Label label) {
    visitInsn();
    super.visitJumpInsn(opcode, label);
  }

  @Override
  public void visitLdcInsn(Object value) {
    visitInsn();
    super.visitLdcInsn(value);
  }

  @Override
  public void visitIincInsn(int var, int increment) {
    visitInsn();
    super.visitIincInsn(var, increment);
  }

  @Override
  public void visitTableSwitchInsn(int min, int max, Label dflt, Label... labels) {
    visitInsn();
    super.visitTableSwitchInsn(min, max, dflt, labels);
  }

  @Override
  public void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels) {
    visitInsn();
    super.visitLookupSwitchInsn(dflt, keys, labels);
  }

  @Override
  public void visitMultiANewArrayInsn(String descriptor, int numDimensions) {
    visitInsn();
    super.visitMultiANewArrayInsn(descriptor, numDimensions);
  }

  @Override
  public void visitTryCatchBlock(Label start, Label end, Label handler, String type) {
    visitInsn();
    super.visitTryCatchBlock(start, end, handler, type);
  }

  @Override
  public void visitLabel(Label label) {
    visitInsn();
    super.visitLabel(label);
  }

  @Override
  public void visitFrame(int type, int numLocal, Object[] local, int numStack, Object[] stack) {
    visitInsn();
    super.visitFrame(type, numLocal, local, numStack, stack);
  }

  @Override
  public void visitMaxs(int maxStack, int maxLocals) {
    visitInsn();
    super.visitMaxs(maxStack, maxLocals);
  }

  protected abstract void visitInsn();
}
```

### 3.12.2 示例一：加零

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public void test(int a, int b) {
        int c = a + b;
        int d = c + 0;
        System.out.println(d);
    }
}
```

我们想要实现的预期目标：将`int d = c + 0;`转换成`int d = c;`。

> 从javap来看，`iconst_0`和接下去的`iadd`是多余的，即多余的加0操作，也就是我们想要删除的指令

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: istore_3
       4: iload_3
       5: iconst_0
       6: iadd
       7: istore        4
       9: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      12: iload         4
      14: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
      17: return
}
```

#### 编码实现

注意点：

+ 这里除了父类`MethodPatternAdapter`已经定义了的`SEEN_NOTHING`用于表示初始状态，只额外在`MethodRemoveAddZeroAdapter`类中添加了一个状态`SEEN_ICONST_0`。因为需要判断的指令只有两个（`iconst_0`和`iadd`），遇到`iconst_0`时转换到`SEEN_ICONST_0`状态，接下去如果遇到`iadd`则表示匹配到了，可以直接再转换到初始状态（即使再多定义一个状态，比如`SEEN_ICONST_0_IADD`，下一步也还是要回归初始状态，所以没必要多定义一个状态）。

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

public class MethodRemoveAddZeroVisitor extends ClassVisitor {
  public MethodRemoveAddZeroVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = cv.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodRemoveAddZeroAdapter(api, mv);
      }
    }
    return mv;
  }

  private class MethodRemoveAddZeroAdapter extends MethodPatternAdapter {
    private static final int SEEN_ICONST_0 = 1;

    public MethodRemoveAddZeroAdapter(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitInsn(int opcode) {
      // 第一，对于感兴趣的状态进行处理
      switch (state) {
        case SEEN_NOTHING:
          if (opcode == ICONST_0) {
            state = SEEN_ICONST_0;
            return;
          }
          break;
        case SEEN_ICONST_0:
          if (opcode == IADD) {
            state = SEEN_NOTHING;
            return;
          }
          // 这里是考虑到有出现多次iconst_0的情况，放弃记录前面的iconst_0，即直接向后传递该指令使其生效
          else if (opcode == ICONST_0) {
            mv.visitInsn(ICONST_0);
            return;
          }
          break;
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitInsn(opcode);
    }

    @Override
    protected void visitInsn() {
      // 当 iconst_0 下一个指令不是 iadd时，匹配失败，重新回归到初始状态(这里覆盖父类MethodPatternAdapter的方法，即每次调用visitXxxInsn()方法时都会尝试判断是否需要回归到初始状态)
      // 回归到初始状态时，需要将指令向后传递
      if (state == SEEN_ICONST_0) {
        mv.visitInsn(ICONST_0);
      }
      state = SEEN_NOTHING;
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodRemoveAddZeroVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: istore_3
       4: iload_3
       5: istore        4
       7: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
      10: iload         4
      12: invokevirtual #22                 // Method java/io/PrintStream.println:(I)V
      15: return
}
```

### 3.12.3 示例二：字段赋值

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public int val;

    public void test(int a, int b) {
        int c = a + b;
        this.val = this.val;
        System.out.println(c);
    }
}
```

我们想要实现的预期目标：删除掉`this.val = this.val;`语句。

> 这里希望删除的，对应指令`aload_0` => `aload_0` => `getfield` => `putfield`

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  public int val;

...

  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: istore_3
       4: aload_0
       5: aload_0
       6: getfield      #2                  // Field val:I
       9: putfield      #2                  // Field val:I
      12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      15: iload_3
      16: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      19: return
}
```

#### 编码实现

![Java ASM系列：（031）修改已有的方法（优化－删除－复杂的变换）_ByteCode](https://s2.51cto.com/images/20210702/1625221942271769.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

public class MethodRemoveGetFieldPutFieldVisitor extends ClassVisitor {
  public MethodRemoveGetFieldPutFieldVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = cv.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodRemoveGetFieldPutFieldAdapter(api, mv);
      }
    }
    return mv;
  }

  private class MethodRemoveGetFieldPutFieldAdapter extends MethodPatternAdapter {
    private final static int SEEN_ALOAD_0 = 1;
    private final static int SEEN_ALOAD_0_ALOAD_0 = 2;
    private final static int SEEN_ALOAD_0_ALOAD_0_GETFIELD = 3;

    private String fieldOwner;
    private String fieldName;
    private String fieldDesc;

    public MethodRemoveGetFieldPutFieldAdapter(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitVarInsn(int opcode, int var) {
      // 第一，对于感兴趣的状态进行处理
      switch (state) {
        case SEEN_NOTHING:
          if (opcode == ALOAD && var == 0) {
            state = SEEN_ALOAD_0;
            return;
          }
          break;
        case SEEN_ALOAD_0:
          if (opcode == ALOAD && var == 0) {
            state = SEEN_ALOAD_0_ALOAD_0;
            return;
          }
          break;
          // 考虑重复 aload_0 的场景，放弃匹配最早出现的aload_0，即直接向后传递该指令
        case SEEN_ALOAD_0_ALOAD_0:
          if (opcode == ALOAD && var == 0) {
            mv.visitVarInsn(opcode, var);
            return;
          }
          break;
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitVarInsn(opcode, var);
    }

    @Override
    public void visitFieldInsn(int opcode, String owner, String name, String descriptor) {
      // 第一，对于感兴趣的状态进行处理
      switch (state) {
        case SEEN_ALOAD_0_ALOAD_0:
          if (opcode == GETFIELD) {
            // 这里需要记录本次 getfield 的字段，后续如果紧接着 putfield ，需要操作同一个字段，才算匹配规则
            state = SEEN_ALOAD_0_ALOAD_0_GETFIELD;
            fieldOwner = owner;
            fieldName = name;
            fieldDesc = descriptor;
            return;
          }
          break;
        case SEEN_ALOAD_0_ALOAD_0_GETFIELD:
          if (opcode == PUTFIELD && name.equals(fieldName)) {
            state = SEEN_NOTHING;
            return;
          }
          break;
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitFieldInsn(opcode, owner, name, descriptor);
    }
		
    // 不匹配 or 匹配到中间失败，则状态重制到初始状态，然后将之前暂时劫持滞留的操作向后传递
    @Override
    protected void visitInsn() {
      switch (state) {
        case SEEN_ALOAD_0:
          mv.visitVarInsn(ALOAD, 0);
          break;
        case SEEN_ALOAD_0_ALOAD_0:
          mv.visitVarInsn(ALOAD, 0);
          mv.visitVarInsn(ALOAD, 0);
          break;
        case SEEN_ALOAD_0_ALOAD_0_GETFIELD:
          mv.visitVarInsn(ALOAD, 0);
          mv.visitVarInsn(ALOAD, 0);
          mv.visitFieldInsn(GETFIELD, fieldOwner, fieldName, fieldDesc);
          break;
      }
      state = SEEN_NOTHING;
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodRemoveGetFieldPutFieldVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
  public int val;

  public sample.HelloWorld();
    Code:
       0: aload_0
       1: invokespecial #10                 // Method java/lang/Object."<init>":()V
       4: return

  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: istore_3
       4: getstatic     #18                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_3
       8: invokevirtual #24                 // Method java/io/PrintStream.println:(I)V
      11: return
}
```

### 3.12.4 示例三：删除打印语句

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public void test(int a, int b) {
        System.out.println("Before a + b");
        int c = a + b;
        System.out.println("After a + b");
        System.out.println(c);
    }
}
```

我们想要实现的预期目标：删除掉打印字符串的语句。

> 即消除下面的`getstatic`=>`ldc`=>`invokevirtual`

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String Before a + b
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: iload_1
       9: iload_2
      10: iadd
      11: istore_3
      12: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      15: ldc           #5                  // String After a + b
      17: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      20: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: iload_3
      24: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
      27: return
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;

import static org.objectweb.asm.Opcodes.*;

public class MethodRemovePrintVisitor extends ClassVisitor {
  public MethodRemovePrintVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = cv.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodRemovePrintAdaptor(api, mv);
      }
    }
    return mv;
  }

  private class MethodRemovePrintAdaptor extends MethodPatternAdapter {
    private static final int SEEN_GETSTATIC = 1;
    private static final int SEEN_GETSTATIC_LDC = 2;

    private String message;

    public MethodRemovePrintAdaptor(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitFieldInsn(int opcode, String owner, String name, String descriptor) {
      // 第一，对于感兴趣的状态进行处理
      boolean flag = (opcode == GETSTATIC && owner.equals("java/lang/System") && name.equals("out") 
                      && descriptor.equals("Ljava/io/PrintStream;"));
      switch (state) {
        case SEEN_NOTHING:
          if (flag) {
            state = SEEN_GETSTATIC;
            return;
          }
          break;
        case SEEN_GETSTATIC:
          if (flag) {
            mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            return;
          }
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitFieldInsn(opcode, owner, name, descriptor);
    }

    @Override
    public void visitLdcInsn(Object value) {
      // 第一，对于感兴趣的状态进行处理
      switch (state) {
        case SEEN_GETSTATIC:
          if (value instanceof String) {
            state = SEEN_GETSTATIC_LDC;
            message = (String) value;
            return;
          }
          break;
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitLdcInsn(value);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
      // 第一，对于感兴趣的状态进行处理
      switch (state) {
        case SEEN_GETSTATIC_LDC:
          if (opcode == INVOKEVIRTUAL && owner.equals("java/io/PrintStream") &&
              name.equals("println") && descriptor.equals("(Ljava/lang/String;)V")) {
            state = SEEN_NOTHING;
            return;
          }
          break;
      }

      // 第二，对于不感兴趣的状态，交给父类进行处理
      super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
    }

    @Override
    protected void visitInsn() {
      switch (state) {
        case SEEN_GETSTATIC:
          mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
          break;
        case SEEN_GETSTATIC_LDC:
          mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
          mv.visitLdcInsn(message);
          break;
      }

      state = SEEN_NOTHING;
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodRemovePrintVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: istore_3
       4: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_3
       8: invokevirtual #22                 // Method java/io/PrintStream.println:(I)V
      11: return
}
```

### 3.12.5 总结

本文对stateful transformations进行介绍，内容总结如下：

- **第一点，stateful transformations可以实现复杂的操作，它是借助于state machine来进行实现的**。
- 第二点，对于`MethodPatternAdapter`类来说，它是一个原始的state machine，本身是一个抽象类；我们写一个具体的子类，作为更“先进”的state machine，在子类当中主要做三件事情：
  - 第一件事情，从字段层面，根据处理的问题，来定义更多的状态。
  - 第二件事情，从方法层面，处理好`visitXxxInsn()`的调用，对于`state`字段状态的影响。
  - 第三件事情，从方法层面，实现`visitInsn()`方法，根据`state`字段的值，如何回归到“初始状态”。

## 3.13 本章内容总结

在本章当中，从Core API的角度来说（第二个层次），我们介绍了`asm.jar`当中的`ClassReader`和`Type`两个类；同时，从应用的角度来说（第一个层次），我们也介绍了Class Transformation的原理和示例。

![Java ASM系列：（032）第三章内容总结_Core API](https://s2.51cto.com/images/20210619/1624105643863231.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.13.1 Class Transformation的原理

在Class Transformation的过程中，主要使用到了`ClassReader`、`ClassVisitor`和`ClassWriter`三个类；其中`ClassReader`类负责“读”Class，`ClassWriter`负责“写”Class，而`ClassVisitor`则负责进行“转换”（Transformation）。

![Java ASM系列：（032）第三章内容总结_ByteCode_02](https://s2.51cto.com/images/20210628/1624882119265917.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在Java ASM当中，Class Transformation的本质就是利用了“中间人攻击”的方式来实现对已有的Class文件进行修改或转换。

![Java ASM系列：（032）第三章内容总结_ClassFile_03](https://s2.51cto.com/images/20210628/1624882180784922.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

详细的来说，我们自己定义的`ClassVisitor`类就是一个“中间人”，那么这个“中间人”可以做什么呢？可以做三种类型的事情：

- 对“原有的信息”进行篡改，就可以实现“修改”的效果。对应到ASM代码层面，就是对`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`的参数值进行修改。
- 对“原有的信息”进行扔掉，就可以实现“删除”的效果。对应到ASM代码层面，将原本的`FieldVisitor`和`MethodVisitor`对象实例替换成`null`值，或者对原本的一些`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法不去调用了。
- 伪造一条“新的信息”，就可以实现“添加”的效果。对应到ASM代码层面，就是在原来的基础上，添加一些对于`ClassVisitor.visitXxx()`和`MethodVisitor.visitXxx()`方法的调用。

### 3.13.2 ASM能够做哪些转换操作

#### 类层面的修改

在类层面所做的修改，主要是通过`ClassVisitor`类来完成。我们将类层面可以修改的信息，分成以下三个方面：

- 类自身信息：修改当前类、父类、接口的信息，通过`ClassVisitor.visit()`方法实现。
- 字段：添加一个新的字段、删除已有的字段，通过`ClassVisitor.visitField()`方法实现。
- 方法：添加一个新的方法、删除已有的方法，通过`ClassVisitor.visitMethod()`方法实现。

```java
public class HelloWorld extends Object implements Cloneable {
  public int intValue;
  public String strValue;

  public int add(int a, int b) {
    return a + b;
  }

  public int sub(int a, int b) {
    return a - b;
  }

  @Override
  public Object clone() throws CloneNotSupportedException {
    return super.clone();
  }
}
```

为了让大家更明确的知道需要修改哪一个`visitXxx()`方法的参数，我们做了如下总结：

- `ClassVisitor.visit(int version, int access, String name, String signature, String superName, String[] interfaces)`
  - `version`: 修改当前Class版本的信息
  - `access`: 修改当前类的访问标识（access flag）信息。
  - `name`: 修改当前类的名字。
  - `signature`: 修改当前类的泛型信息。
  - `superName`: 修改父类。
  - `interfaces`: 修改接口信息。
- `ClassVisitor.visitField(int access, String name, String descriptor, String signature, Object value)`
  - `access`: 修改当前字段的访问标识（access flag）信息。
  - `name`: 修改当前字段的名字。
  - `descriptor`: 修改当前字段的描述符。
  - `signature`: 修改当前字段的泛型信息。
  - `value`: 修改当前字段的常量值。
- `ClassVisitor.visitMethod(int access, String name, String descriptor, String signature, String[] exceptions)`
  - `access`: 修改当前方法的访问标识（access flag）信息。
  - `name`: 修改当前方法的名字。
  - `descriptor`: 修改当前方法的描述符。
  - `signature`: 修改当前方法的泛型信息。
  - `exceptions`: 修改当前方法可以抛出的异常信息。

再有，**如何删除一个字段或者方法呢？**其实很简单，我们只要让中间的某一个`ClassVisitor`在遇到该字段或方法时，不向后传递就可以了。在具体的代码实现上，我们只要让`visitField()`或`visitMethod()`方法返回一个`null`值就可以了。

![Java ASM系列：（032）第三章内容总结_Core API_04](https://s2.51cto.com/images/20210628/1624882148334418.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

最后，**如何添加一个字段或方法呢？**我们只要让中间的某一个`ClassVisitor`向后多传递一个字段和方法就可以了。<u>在具体的代码实现上，我们是在`visitEnd()`方法完成对字段或方法的添加，而不是在`visitField()`或`visitMethod()`当中添加。因为我们要避免“一个类里有重复的字段和方法出现”，在`visitField()`或`visitMethod()`方法中，我们要判断该字段或方法是否已经存在；如果该字段或方法不存在，那我们就在`visitEnd()`方法进行添加；如果该字段或方法存在，那么我们就不需要在`visitEnd()`方法中添加了</u>。

#### 方法体层面的修改

在方法体层面所做的修改，主要是通过`MethodVisitor`类来完成。

在方法体层面的修改，更准确的地说，就是对方法体内包含的Instruction进行修改。就像数据库的操作“增删改查”一样，我们也可以对Instruction进行添加、删除、修改和查找。

为了让大家更直观的理解，我们假设有如下代码：

```java
public class HelloWorld {
    public int test(String name, int age) {
        int hashCode = name.hashCode();
        hashCode = hashCode + age * 31;
        return hashCode;
    }
}
```

其中，`test()`方法的方法体包含的Instruction内容如下：

```shell
  public test(Ljava/lang/String;I)I
    ALOAD 1
    INVOKEVIRTUAL java/lang/String.hashCode ()I
    ISTORE 3
    ILOAD 3
    ILOAD 2
    BIPUSH 31
    IMUL
    IADD
    ISTORE 3
    ILOAD 3
    IRETURN
    MAXSTACK = 3
    MAXLOCALS = 4
```

有的时候，我们想实现某个功能，但是感觉无从下手。这个时候，我们需要解决两个问题。第一个问题，就是要明确需要修改什么？第二个问题，就是“定位”方法，也就是要使用哪个方法进行修改。我们可以结合这两个问题，和下面的示例应用来理解。

- 添加
  - 在“方法进入”时和“方法退出”时，
    - 打印方法参数和返回值
    - 方法计时
- 删除
  - 移除`NOP`
  - 移除打印语句、加零、字段赋值
  - 清空方法体
- 修改
  - 替换方法调用（静态方法和非静态方法）
- 查找
  - 当前方法调用了哪些方法
  - 当前方法被哪些方法所调用

由于`MethodVisitor`类里定义了很多的`visitXxxInsn()`方法，我们就不详细介绍了。但是，大家可以的看一下[asm4-guide.pdf](https://asm.ow2.io/asm4-guide.pdf)的一段描述：

Methods can be transformed, i.e. by using a method adapter that forwards the method calls it receives with some modifications:

- changing arguments can be used to change individual instructions,
- not forwarding a received call removes an instruction,
- and inserting calls between the received ones adds new instructions.

需要要注意一点：**无论是添加instruction，还是删除instruction，还是要替换instruction，都要保持operand stack修改前和修改后是一致的**。

### 3.13.3 总结

本文内容总结如下：

- 第一点，希望大家可以理解Class Transformation的原理。
- 第二点，在Class Transformation中，ASM究竟能够帮助我们修改哪些信息。

# 4. 工具类和常用类

## 4.1 asm-util和asm-commons

### 4.1.1 asm-util

在`asm-util.jar`当中，主要介绍`CheckClassAdapter`和`TraceClassVisitor`类。在`TraceClassVisitor`类当中，会涉及到`Printer`、`ASMifier`和`Textifier`类。

![Java ASM系列：（033）asm-util和asm-commons_ASM](https://s2.51cto.com/images/20210703/1625323958764266.png)

- 其中，`CheckClassAdapter`类，主要负责检查（Check）生成的`.class`文件内容是否正确。（但是能力有限，不能全指望靠其发现生成的字节码的问题）
- 其中，`TraceClassVisitor`类，主要负责将`.class`文件的内容打印成文字输出。根据输出的文字信息，可以探索或追踪（Trace）`.class`文件的内部信息。
  - `Printer`负责将`.class`文件的内容转成文字（`ASMifiler`转成ASM代码文字，`Textifier`转成JVM指令文字）
  - `PrinterWriter`负责输出文字

> "1.5.1 ASMPrint类"讲解到的`ASMPrint`类就使用到了`TraceClassVisitor`

### 4.1.2 asm-commons

在`asm-commons.jar`当中，包括的类比较多，我们就不一一介绍每个类的作用了。但是，我们可以这些类可以分成两组，一组是`ClassVisitor`的子类，另一组是`MethodVisitor`的子类。

- 其中，`ClassVisitor`的子类有`ClassRemapper`、`StaticInitMerger`和`SerialVersionUIDAdder`类；
- 其中，`MethodVisitor`的子类有`LocalVariablesSorter`、`GeneratorAdapter`、`AdviceAdapter`、`AnalyzerAdapter`和`InstructionAdapter`类。

![Java ASM系列：（033）asm-util和asm-commons_ByteCode_02](https://s2.51cto.com/images/20210703/1625323993793949.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 4.1.3 util和commons的区别

那么，**asm-util.jar**与**asm-commons.jar**有什么区别呢？在`asm-util.jar`里，它提供的是通用性的功能，没有特别明确的应用场景；而在`asm-commons.jar`里，它提供的功能，都是为解决某一种特定场景中出现的问题而提出的解决思路。

- the `org.objectweb.asm.util` package, in the `asm-util.jar` archive, provides various tools based on the core API that can be used during the development and debuging of ASM applications.
  - 在`CheckClassAdapter`类的代码实现中，它会依赖于`org.objectweb.asm.tree`和`org.objectweb.asm.tree.analysis`的内容。因此，`asm-util.jar`除了依赖Core API之外，也依赖于Tree API的内容。
- the `org.objectweb.asm.commons` package provides several useful pre-defined class transformers, mostly based on the core API. It is contained in the `asm-commons.jar` archive.

在[learn-java-asm](https://gitee.com/lsieun/learn-java-asm)当中，各个Jar包之间的依赖关系如下：

![img](https://lsieun.github.io/assets/images/java/asm/learn-java-asm-depdendencies.png)

由上图，我们可以看到：`asm-util.jar`和`asm-commons.jar`两者都对`asm.jar`、`asm-tree.jar`、`asm-analysis.jar`有依赖。

![img](https://lsieun.github.io/assets/images/java/asm/relation-of-asm-jars.png)

### 4.1.4 编程习惯

> 在编写ASM代码的过程中，讲师经常遵循一个命名的习惯：如果添加一个新的类，它继承自`ClassVisitor`，那么就命名成`XxxVisitor`；如果添加一个新的类，它继承自`MethodVisitor`，那么就命名成`XxxAdapter`。通过类的名字，我们就可以区分出哪些类是继承自`ClassVisitor`，哪些类是继承自`MethodVisitor`。
>
> 其实，将`MethodVisitor`类的子类命名成`XxxAdapter`就是参考了`GeneratorAdapter`、`AdviceAdapter`、`AnalyzerAdapter`和`InstructionAdapter`类的名字。但是，`CheckClassAdapter`类是个例外，它是继承自`ClassVisitor`类。

## 4.2 CheckClassAdapter介绍

The `CheckClassAdapter` class checks that its **methods are called** in the **appropriate order**, and with **valid arguments**, before delegating to the next visitor.

即CheckClassAdapter主要检查方法调用时的JVM指令是否正确顺序，以及参数是否合法有效，整体主要做方法层面指令的校验。

> 详细的说明，可以直接看CheckClassAdapter类上面的注释，下面截取部分：
>
> A ClassVisitor that checks that its methods are properly used. More precisely this class adapter checks each method call individually, based only on its arguments, but does not check the sequence of method calls. For example, the invalid sequence visitField(ACC_PUBLIC, "i", "I", null) visitField(ACC_PUBLIC, "i", "D", null) will not be detected by this class adapter.
>
> CheckClassAdapter can be also used to verify bytecode transformations in order to make sure that the transformed bytecode is sane. For example:
>
> ...

### 4.2.1 CheckClassAdapter使用方式

到目前为止，我们主要介绍了Class Generation和Class Transformation操作。我们可以借助于`CheckClassAdapter`类来检查生成的字节码内容是否正确，主要有两种使用方式：

- 在生成类或转换类的**过程中**进行检查
- 在生成类或转换类的**结束后**进行检查

#### 方式一：生成/转换类"过程中"检查

第一种使用方式，是在生成类（Class Generation）或转换类（Class Transformation）的**过程中**进行检查：

```java
// 第一步，应用于Class Generation
// 串联ClassVisitor：cv --- cca --- cw
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
CheckClassAdapter cca = new CheckClassAdapter(cw);
ClassVisitor cv = new MyClassVisitor(cca);

// 第二步，应用于Class Transformation
byte[] bytes = ... // 这里是class file bytes
ClassReader cr = new ClassReader(bytes);
cr.accept(cv, 0);
```

示例如下：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.util.CheckClassAdapter;

import static org.objectweb.asm.Opcodes.*;

public class CheckClassAdapterExample01Generate {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    ClassVisitor cv = new CheckClassAdapter(cw);

    // (2) 调用visitXxx()方法
    cv.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld",
             null, "java/lang/Object", null);

    {
      FieldVisitor fv = cv.visitField(ACC_PRIVATE, "intValue", "I", null, null);
      fv.visitEnd();
    }

    {
      MethodVisitor mv1 = cv.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      mv1.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      mv1.visitMaxs(1, 1);
      mv1.visitEnd();
    }

    {
      MethodVisitor mv2 = cv.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv2.visitCode();
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("Hello World");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 1);
      mv2.visitEnd();
    }
    cv.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

#### 方式二：生成/转换类"结束后"检查

第二种使用方式，是在生成类（Class Generation）或转换类（Class Transformation）的**结束后**进行检查：

```java
byte[] bytes = ... // 这里是class file bytes
PrintWriter printWriter = new PrintWriter(System.out);
CheckClassAdapter.verify(new ClassReader(bytes), false, printWriter);
```

示例如下：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.util.CheckClassAdapter;

import java.io.PrintWriter;

import static org.objectweb.asm.Opcodes.*;

public class CheckClassAdapterExample02Generate {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);

    // (3) 检查
    PrintWriter printWriter = new PrintWriter(System.out);
    CheckClassAdapter.verify(new ClassReader(bytes), false, printWriter);
  }

  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用visitXxx()方法
    cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld",
             null, "java/lang/Object", null);

    {
      FieldVisitor fv = cw.visitField(ACC_PRIVATE, "intValue", "I", null, null);
      fv.visitEnd();
    }

    {
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      mv1.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      mv1.visitMaxs(1, 1);
      mv1.visitEnd();
    }

    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv2.visitCode();
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("Hello World");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 1);
      mv2.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

### 4.2.2 检测案例

> 下面以 方式二：生成/转换类"结束后"检查 来举例，因为其提示文案更近人意

#### 检测：方法调用顺序

> 方式一：生成/转换类"过程中"检查 ，检测不出来

如果将`mv2.visitLdcInsn()`和`mv2.visitFieldInsn()`顺序调换：

```java
mv2.visitLdcInsn("Hello World");
mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
```

会出现如下错误：

```shell
Method owner: expected Ljava/io/PrintStream;, but found Ljava/lang/String;
```

#### 检测：方法参数不对

如果将方法的描述符（`(Ljava/lang/String;)V`）修改成`(I)V`：

```java
mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
mv2.visitLdcInsn("Hello World");
mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);
```

会出现如下错误：

```shell
Argument 1: expected I, but found Ljava/lang/String;
```

#### 检测：没有return语句

如果注释掉`mv2.visitInsn(RETURN);`语句，会出现如下错误：

```shell
org.objectweb.asm.tree.analysis.AnalyzerException: Execution can fall off the end of the code
```

#### 检测：不调用visitMaxs()方法

如果注释掉`mv2.visitMaxs(2, 1);`语句，会出现如下错误：

```shell
org.objectweb.asm.tree.analysis.AnalyzerException: Error at instruction 0: Insufficient maximum stack size.
```

#### 检测不出：重复类成员

如果出现重复的字段或者重复的方法，`CheckClassAdapter`类是检测不出来的（方式一和方式二都检测不出来）：

```java
{
  FieldVisitor fv = cw.visitField(ACC_PRIVATE, "intValue", "I", null, null);
  fv.visitEnd();
}

{
  FieldVisitor fv = cw.visitField(ACC_PRIVATE, "intValue", "I", null, null);
  fv.visitEnd();
}
```

## 4.3 TraceClassVisitor介绍

`TraceClassVisitor` class extends the `ClassVisitor` class, and builds **a textual representation of the visited class**.

### 4.3.1 TraceClassVisitor类

#### class info

第一个部分，`TraceClassVisitor`类继承自`ClassVisitor`类，而且有`final`修饰，因此不会存在子类。

```java
public final class TraceClassVisitor extends ClassVisitor {
}
```

#### fields

第二个部分，`TraceClassVisitor`类定义的字段有哪些。`TraceClassVisitor`类有两个重要的字段，一个是`PrintWriter printWriter`用于打印；另一个是`Printer p`将class转换成文字信息。

```java
public final class TraceClassVisitor extends ClassVisitor {
  /** The print writer to be used to print the class. May be {@literal null}. */
  private final PrintWriter printWriter; // 真正打印输出的类
  /** The printer to convert the visited class into text. */
  public final Printer p; // 信息采集器
}
```

#### constructors

第三个部分，`TraceClassVisitor`类定义的构造方法有哪些。

```java
public final class TraceClassVisitor extends ClassVisitor {
  public TraceClassVisitor(final PrintWriter printWriter) {
    this(null, printWriter);
  }

  public TraceClassVisitor(final ClassVisitor classVisitor, final PrintWriter printWriter) {
    this(classVisitor, new Textifier(), printWriter);
  }

  public TraceClassVisitor(final ClassVisitor classVisitor, final Printer printer, final PrintWriter printWriter) {
    super(Opcodes.ASM10_EXPERIMENTAL, classVisitor);
    this.printWriter = printWriter;
    this.p = printer;
  }
}
```

#### methods

第四个部分，`TraceClassVisitor`类定义的方法有哪些。对于`TraceClassVisitor`类的`visit()`、`visitField()`、`visitMethod()`和`visitEnd()`方法，会分别调用`Printer.visit()`、`Printer.visitField()`、`Printer.visitMethod()`和`Printer.visitClassEnd()`方法。

> TraceClassVisitor的`visitEnd()`里才是真正输出逻辑。正如前面所说，平时记得调用`visitEnd()`，毕竟你不知道后面串联的ClassVisitor是否在`visitEnd()`也有处理逻辑。

```java
public final class TraceClassVisitor extends ClassVisitor {
  @Override
  public void visit(final int version, final int access, final String name, final String signature,
                    final String superName, final String[] interfaces) {
    p.visit(version, access, name, signature, superName, interfaces);
    super.visit(version, access, name, signature, superName, interfaces);
  }

  @Override
  public FieldVisitor visitField(final int access, final String name, final String descriptor,
                                 final String signature, final Object value) {
    Printer fieldPrinter = p.visitField(access, name, descriptor, signature, value);
    return new TraceFieldVisitor(super.visitField(access, name, descriptor, signature, value), fieldPrinter);
  }

  @Override
  public MethodVisitor visitMethod(final int access, final String name, final String descriptor,
                                   final String signature, final String[] exceptions) {
    Printer methodPrinter = p.visitMethod(access, name, descriptor, signature, exceptions);
    return new TraceMethodVisitor(super.visitMethod(access, name, descriptor, signature, exceptions), methodPrinter);
  }

  @Override
  public void visitEnd() {
    p.visitClassEnd();
    if (printWriter != null) {
      p.print(printWriter); // Printer和PrintWriter进行结合
      printWriter.flush();
    }
    super.visitEnd();
  }
}
```

### 4.3.2 如何使用TraceClassVisitor类

使用`TraceClassVisitor`类，很重点的一点就是选择`Printer`类的具体实现，可以选择`ASMifier`类，也可以选择`Textifier`类（默认）：

```java
boolean flag = true or false;
Printer printer = flag ? new ASMifier() : new Textifier();
PrintWriter printWriter = new PrintWriter(System.out, true);
TraceClassVisitor traceClassVisitor = new TraceClassVisitor(null, printer, printWriter);
```

#### 生成新的类

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.PrintWriter;

import static org.objectweb.asm.Opcodes.*;

public class TraceClassVisitorExample01Generate {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    PrintWriter printWriter = new PrintWriter(System.out);
    TraceClassVisitor cv = new TraceClassVisitor(cw, printWriter);

    // (2) 调用visitXxx()方法
    cv.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    {
      FieldVisitor fv1 = cv.visitField(ACC_PRIVATE, "intValue", "I", null, null);
      fv1.visitEnd();
    }

    {
      FieldVisitor fv2 = cv.visitField(ACC_PRIVATE, "strValue", "Ljava/lang/String;", null, null);
      fv2.visitEnd();
    }

    {
      MethodVisitor mv1 = cv.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      mv1.visitCode();
      mv1.visitVarInsn(ALOAD, 0);
      mv1.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
      mv1.visitInsn(RETURN);
      mv1.visitMaxs(0, 0);
      mv1.visitEnd();
    }

    {
      MethodVisitor mv2 = cv.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv2.visitCode();
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("Hello World");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 1);
      mv2.visitEnd();
    }

    cv.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

#### 修改已有的类

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.PrintWriter;

public class TraceClassVisitorExample02Transform {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    PrintWriter printWriter = new PrintWriter(System.out);
    TraceClassVisitor tcv = new TraceClassVisitor(cw, printWriter);
    ClassVisitor cv = new MethodTimerVisitor(api, tcv);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 打印ASM代码

```java
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.util.ASMifier;
import org.objectweb.asm.util.Printer;
import org.objectweb.asm.util.Textifier;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.IOException;
import java.io.PrintWriter;

/**
 * 这里的代码是参考自{@link org.objectweb.asm.util.Printer#main}
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

### 4.3.3 如何使用TraceMethodVisitor类

有的时候，我们想打印出某一个具体方法包含的指令。在实现这个功能的时候，使用Tree API比较容易一些，因为`MethodNode`类（Tree API）就代表一个单独的方法。那么，结合`MethodNode`和`TraceMethodVisitor`类，我们就可以打印出该方法包含的指令。

```java
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.AbstractInsnNode;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.util.Textifier;
import org.objectweb.asm.util.TraceMethodVisitor;

import java.io.PrintWriter;
import java.util.List;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode();

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）找到某个具体的方法
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);

    //（4）打印输出
    Textifier printer = new Textifier();
    TraceMethodVisitor tmv = new TraceMethodVisitor(printer);

    InsnList instructions = mn.instructions;
    for (AbstractInsnNode node : instructions) {
      node.accept(tmv);
    }
    List<Object> list = printer.text;
    printList(list);
  }

  private static void printList(List<?> list) {
    PrintWriter writer = new PrintWriter(System.out);
    printList(writer, list);
    writer.flush();
  }

  // 下面这段代码来自org.objectweb.asm.util.Printer.printList()方法
  private static void printList(final PrintWriter printWriter, final List<?> list) {
    for (Object o : list) {
      if (o instanceof List) {
        printList(printWriter, (List<?>) o);
      }
      else {
        printWriter.print(o);
      }
    }
  }
}
```

### 4.3.4 总结

本文对`TraceClassVisitor`类进行了介绍，内容总结如下：

- 第一点，从整体上来说，`TraceClassVisitor`类的作用是什么。它能够将class文件的内容转换成文字输出。
- 第二点，从结构上来说，`TraceClassVisitor`类的各个部分包含哪些信息。
- 第三点，从使用上来说，`TraceClassVisitor`类的输出结果依赖于`Printer`类的具体实现，可以选择`ASMifier`类输出ASM代码，也可以选择`Textifier`类输出Instruction信息。

## 4.4 Printer/ASMifier/Textifier介绍

### 4.4.1 Printer类

#### class info

第一个部分，`Printer`类是一个`abstract`类，它有两个子类：`ASMifier`类和`Textifier`类。

```java
/**
 * An abstract converter from visit events to text.
 *
 * @author Eric Bruneton
 */
public abstract class Printer {
}
```

#### fields

第二个部分，`Printer`类定义的字段有哪些。

```java
public abstract class Printer {
  protected final int api;

  // The builder used to build strings in the various visit methods.
  protected final StringBuilder stringBuilder;

  // The text to be printed.
  public final List<Object> text;
}
```

#### constructors

第三个部分，`Printer`类定义的构造方法有哪些。

```java
public abstract class Printer {
  protected Printer(final int api) {
    this.api = api;
    this.stringBuilder = new StringBuilder();
    this.text = new ArrayList<>();
  }
}
```

#### methods

第四个部分，`Printer`类定义的方法有哪些。

##### visitXxx方法

`Printer`类定义的`visitXxx`方法是与`ClassVisitor`和`MethodVisitor`类里定义的方法有很大的相似性。

> 这里不是`visitEnd()`，取而代之的是`visitClassEnd()`和`visitMethodEnd()`

```java
public abstract class Printer {
  // Classes，这部分方法可与ClassVisitor内定义的方法进行对比
  public abstract void visit(int version, int access, String name, String signature, String superName, String[] interfaces);
  public abstract Printer visitField(int access, String name, String descriptor, String signature, Object value);
  public abstract Printer visitMethod(int access, String name, String descriptor, String signature, String[] exceptions);
  public abstract void visitClassEnd();
  // ......


  // Methods，这部分方法可与MethodVisitor内定义的方法进行对比
  public abstract void visitCode();
  public abstract void visitInsn(int opcode);
  public abstract void visitIntInsn(int opcode, int operand);
  public abstract void visitVarInsn(int opcode, int var);
  public abstract void visitTypeInsn(int opcode, String type);
  public abstract void visitFieldInsn(int opcode, String owner, String name, String descriptor);
  public void visitMethodInsn(final int opcode, final String owner, final String name, final String descriptor, final boolean isInterface);
  public abstract void visitJumpInsn(int opcode, Label label);
  // ......
  public abstract void visitMaxs(int maxStack, int maxLocals);
  public abstract void visitMethodEnd();
}
```

##### print方法

下面这个`print(PrintWriter)`方法会在`TraceClassVisitor.visitEnd()`方法中调用。

- `print(PrintWriter)`方法的作用：打印出`text`字段的值，将采集的内容进行输出。
- `print(PrintWriter)`方法的调用时机：在`TraceClassVisitor.visitEnd()`方法中。

```java
public abstract class Printer {
  public void print(final PrintWriter printWriter) {
    printList(printWriter, text);
  }

  static void printList(final PrintWriter printWriter, final List<?> list) {
    for (Object o : list) {
      if (o instanceof List) {
        printList(printWriter, (List<?>) o);
      } else {
        printWriter.print(o.toString());
      }
    }
  }
}
```

### 4.4.2 ASMifier类和Textifier类

对于`ASMifier`类和`Textifier`类来说，它们的父类是`Printer`类。

```java
public class ASMifier extends Printer {
}
```

```java
public class Textifier extends Printer {
}
```

在这里，我们不对`ASMifier`类和`Textifier`类的成员信息进行展开，因为它们的内容非常多。但是，这么多的内容都是为了一个共同的目的：通过对`visitXxx()`方法的调用，将class的内容转换成文字的表示形式。

除了`ASMifier`和`Textifier`这两个类，如果有什么好的想法，我们也可以写一个自定义的`Printer`类进行使用。

### 4.4.3 如何使用

对于`ASMifier`和`Textifier`这两个类来说，它们的使用方法是非常相似的。换句话说，知道了如何使用`ASMifier`类，也就知道了如何使用`Textifier`类；反过来说，知道了如何使用`Textifier`类，也就知道了如何使用`ASMifier`类。

#### 从命令行使用

Linux分隔符是“:”

```shell
$ java -classpath asm.jar:asm-util.jar org.objectweb.asm.util.ASMifier java.lang.Runnable
```

Windows分隔符是“;”

```shell
$ java -classpath asm.jar;asm-util.jar org.objectweb.asm.util.ASMifier java.lang.Runnable
```

Cygwin分隔符是“\;”

```shell
$ java -classpath asm.jar\;asm-util.jar org.objectweb.asm.util.ASMifier java.lang.Runnable
```

#### 从代码中使用

无论是`ASMifier`类里的`main()`方法，还是`Textifier`类里的`main()`方法，它们本质上都是调用了`Printer`类里的`main()`方法。在`Printer`类里的`main()`方法里，代码的功能也是通过`TraceClassVisitor`类来实现的。

在Java ASM 9.0版本当中，使用`-debug`选项：

```java
import org.objectweb.asm.util.ASMifier;

import java.io.IOException;

public class HelloWorldRun {
  public static void main(String[] args) throws IOException {
    String[] array = new String[] {
      "-debug",
      "sample.HelloWorld"
    };
    ASMifier.main(array);
  }
}
```

在Java ASM 9.1或9.2及之后版本当中，使用`-nodebug`选项：（这一点，要感谢[4ra1n](https://4ra1n.love/)同学指出错误并纠正）

```java
import org.objectweb.asm.util.ASMifier;

import java.io.IOException;

public class HelloWorldRun {
  public static void main(String[] args) throws IOException {
    String[] array = new String[] {
      "-nodebug",
      "sample.HelloWorld"
    };
    ASMifier.main(array);
  }
}
```

在[Versions](https://asm.ow2.io/versions.html)当中，提到：

```shell
6 February 2021: ASM 9.1 (tag ASM_9_1)

－ Replace -debug flag in Printer with -nodebug (-debug continues to work)
```

但是，在 Java ASM 9.1和9.2版本中，我测试了一下`-debug`选项，它是不能用的。

### 4.4.4 总结

本文对`Printer`、`ASMifier`和`Textifier`这三个类进行介绍，内容总结如下：

- 第一点，了解这三个类的主要目的是为了方便理解`TraceClassVisitor`类的工作原理。
- 第二点，如何从命令行使用`ASMifier`类和`Textifier`类。

## 4.5 AdviceAdapter介绍

对于 `AdviceAdapter` 类来说，能够很容易的实现在“方法进入”和“方法退出”时添加代码。

`AdviceAdapter` 类的特点：引入了 `onMethodEnter()` 方法和 `onMethodExit()` 方法。

### 4.5.1 AdviceAdapter类

#### class info

第一个部分，`AdviceAdapter` 类是一个抽象的（`abstract`）、特殊的 `MethodVisitor` 类。

具体来说，`AdviceAdapter` 类继承自 `GeneratorAdapter` 类，而 `GeneratorAdapter` 类继承自 `LocalVariablesSorter` 类， `LocalVariablesSorter` 类继承自 `MethodVisitor` 类。

- org.objectweb.asm.MethodVisitor
  - org.objectweb.asm.commons.LocalVariablesSorter
    - org.objectweb.asm.commons.GeneratorAdapter
      - org.objectweb.asm.commons.AdviceAdapter

由于 `AdviceAdapter` 类是抽象类（`abstract`），如果我们想使用这个类，那么就需要实现一个具体的子类。

```java
/**
 * A {@link MethodVisitor} to insert before, after and around advices in methods and constructors.
 * For constructors, the code keeps track of the elements on the stack in order to detect when the
 * super class constructor is called (note that there can be multiple such calls in different
 * branches). {@code onMethodEnter} is called after each super class constructor call, because the
 * object cannot be used before it is properly initialized.
 *
 * @author Eugene Kuleshov
 * @author Eric Bruneton
 */
public abstract class AdviceAdapter extends GeneratorAdapter implements Opcodes {
}
```

#### fields

第二个部分，`AdviceAdapter` 类定义的字段有哪些。其中， `isConstructor` 字段是判断当前方法是不是构造方法。如果当前方法是构造方法，在“方法进入”时添加代码，需要特殊处理。

```java
public abstract class AdviceAdapter extends GeneratorAdapter implements Opcodes {
  /** The access flags of the visited method. */
  protected int methodAccess;
  /** The descriptor of the visited method. */
  protected String methodDesc;

  /** Whether the visited method is a constructor. */
  private final boolean isConstructor;
}
```

#### constructors

第三个部分，`AdviceAdapter` 类定义的构造方法有哪些。 需要注意的是，`AdviceAdapter` 的构造方法是用 `protected` 修饰，因此这个构造方法只能在子类当中访问。 换句话说，在外界不能用 `new` 关键字来创建对象。

```java
public abstract class AdviceAdapter extends GeneratorAdapter implements Opcodes {
    protected AdviceAdapter(final int api, final MethodVisitor methodVisitor,
                            final int access, final String name, final String descriptor) {
        super(api, methodVisitor, access, name, descriptor);
        methodAccess = access;
        methodDesc = descriptor;
        isConstructor = "<init>".equals(name);
    }
}
```

#### methods

第四个部分，`AdviceAdapter` 类定义的方法有哪些。

在 `AdviceAdapter` 类的方法中，定义了两个重要的方法：`onMethodEnter()` 方法和 `onMethodExit()` 方法。

```java
public abstract class AdviceAdapter extends GeneratorAdapter implements Opcodes {
    // Generates the "before" advice for the visited method.
    // The default implementation of this method does nothing.
    // Subclasses can use or change all the local variables, but should not change state of the stack.
    // This method is called at the beginning of the method or
    // after super class constructor has been called (in constructors).
    protected void onMethodEnter() {}

    // Generates the "after" advice for the visited method.
    // The default implementation of this method does nothing.
    // Subclasses can use or change all the local variables, but should not change state of the stack.
    // This method is called at the end of the method, just before return and athrow instructions.
    // The top element on the stack contains the return value or the exception instance.
    protected void onMethodExit(final int opcode) {}
}
```

对于 `onMethodEnter()` 和 `onMethodExit()` 这两个方法，我们从三个角度来把握它们：

- 第一个角度，应用场景。
  - `onMethodEnter()` 方法：在“方法进入”的时候，添加一些代码逻辑。
  - `onMethodExit()` 方法：在“方法退出”的时候，添加一些代码逻辑。
- 第二个角度，注意事项。
  - 第一点，对于 `onMethodEnter()` 和 `onMethodExit()` 这两个方法，都要注意 Subclasses can use or change all the local variables, but should not change state of the stack。也就是说，要保持operand stack在修改前和修改后是一致的。
  - 第二点，对于 `onMethodExit()` 方法，要注意 The top element on the stack contains the return value or the exception instance。也就是说，“方法退出”的时候，operand stack 上有返回值或异常对象，不要忘记处理，不要弄丢了它们。
- 第三个角度，工作原理。
  - 对于 `onMethodEnter()` 方法，它是借助于 `visitCode()` 方法来实现的。**使用 `onMethodEnter()` 方法的优势在于，它能够处理 `<init>()` 的复杂情况，而直接使用 `visitCode()` 方法则可能导致 `<init>()` 方法出现错误。**（直接使用`visitCode()`的话，还需要自己考虑怎么保证原本的构造方法中`super()`或`this()`怎么仍然在第一条执行，否则会报错）
  - 对于 `onMethodExit()` 方法，它是借助于 `visitInsn(int opcode)` 方法来实现的。

### 4.5.2 示例: 打印方法参数和返回值

#### 预期目标

假如有一个 `HelloWorld` 类，代码如下：

```java
public class HelloWorld {
  private String name;
  private int age;

  public HelloWorld(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public void test(long idCard, Object obj) {
    int hashCode = 0;
    hashCode += name.hashCode();
    hashCode += age;
    hashCode += (int) (idCard % Integer.MAX_VALUE);
    hashCode += obj.hashCode();
    hashCode = Math.abs(hashCode);
    System.out.println("Hash Code is " + hashCode);
    if (hashCode % 2 == 1) {
      throw new RuntimeException("illegal");
    }
  }
}
```

我们想实现的预期目标：打印出构造方法（`<init>()`）和 `test()` 的参数和返回值。

#### 编码实现

```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

public class ParameterUtils {
  private static final DateFormat fm = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

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
      System.out.println("    " + Arrays.toString((char[])value));
    }
    else {
      System.out.println("    " + value.getClass() + ": " + value.toString());
    }
  }

  public static void printText(String str) {
    System.out.println(str);
  }
}
```

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AdviceAdapter;

import static org.objectweb.asm.Opcodes.*;

public class ClassPrintParameterVisitor extends ClassVisitor {
  public ClassPrintParameterVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodPrintParameterAdapter(api, mv, access, name, descriptor);
      }
    }
    return mv;
  }

  public static class MethodPrintParameterAdapter extends AdviceAdapter {
    public MethodPrintParameterAdapter(int api, MethodVisitor mv, int access, String name, String descriptor) {
      super(api, mv, access, name, descriptor);
    }

    @Override
    protected void onMethodEnter() {
      printMessage("Method Enter: " + getName() + methodDesc);

      Type[] argumentTypes = getArgumentTypes();
      for (int i = 0; i < argumentTypes.length; i++) {
        Type t = argumentTypes[i];
        loadArg(i);
        box(t);
        printValueOnStack("(Ljava/lang/Object;)V");
      }
    }

    @Override
    protected void onMethodExit(int opcode) {
      printMessage("Method Exit: " + getName() + methodDesc);

      if (opcode == ATHROW) {
        super.visitLdcInsn("abnormal return");
      }
      else if (opcode == RETURN) {
        super.visitLdcInsn("return void");
      }
      else if (opcode == ARETURN) {
        dup();
      }
      else {
        if (opcode == LRETURN || opcode == DRETURN) {
          dup2();
        }
        else {
          dup();
        }
        box(getReturnType());
      }
      printValueOnStack("(Ljava/lang/Object;)V");
    }

    private void printMessage(String str) {
      super.visitLdcInsn(str);
      super.visitMethodInsn(INVOKESTATIC, "sample/ParameterUtils", "printText", "(Ljava/lang/String;)V", false);
    }

    private void printValueOnStack(String descriptor) {
      super.visitMethodInsn(INVOKESTATIC, "sample/ParameterUtils", "printValueOnStack", descriptor, false);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassPrintParameterVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
import java.util.Date;

public class HelloWorldRun {
  public static void main(String[] args) {
    HelloWorld instance = new HelloWorld("tomcat", 10);
    instance.test(441622197605122816L, new Date());
  }
}
```

### 4.5.3 AdviceAdapter VS. MethodVisitor

`AdviceAdapter` 类的特点：引入了 `onMethodEnter()` 方法和 `onMethodExit()` 方法。

```
                 ┌─── onMethodEnter()
AdviceAdapter ───┤
                 └─── onMethodExit()
```

同时，回顾一下 `MethodVisitor` 类里的四个主要方法：

```
                 ┌─── visitCode()
                 │
                 ├─── visitXxxInsn()
MethodVisitor ───┤
                 ├─── visitMaxs()
                 │
                 └─── visitEnd()
```

将 `AdviceAdapter` 类和 `MethodVisitor` 类进行一下对比：

- `AdviceAdapter.onMethodEnter()` 方法对应着 `MethodVisitor.visitCode()` 方法
- `AdviceAdapter.onMethodExit()` 方法对应着 `MethodVisitor.visitInsn(opcode)` 方法当中 opcode 为 `return` 或 `athrow` 的情况

![img](https://lsieun.github.io/assets/images/java/asm/advice-adapter-vs-method-visitor.png)

### 4.5.4 总结

本文对 `AdviceAdapter` 类进行介绍，内容总结如下：

- 第一点，在`AdviceAdapter`类当中，有两个关键的方法，即`onMethodEnter()`和`onMethodExit()`方法。我们可以从三个角度来把握这两个方法：
  - **从使用场景的角度来说，`AdviceAdapter` 类能够很容易的在“方法进入”时和“方法退出”时添加一些代码**。
  - **从注意事项的角度来说， Subclasses can use or change all the local variables, but should not change state of the stack.（即子类可以在方法执行时修改Frame的local variables，但是需要保证逻辑执行后，原本Frame的operand stack还是保持原样）**
  - **从工作原理的角度来说，`onMethodEnter()` 方法是借助于 `visitCode()` 方法来实现的；`onMethodExit()` 方法是借助于 `visitInsn(int opcode)` 方法来实现的。**
- 第二点，特殊的情况的处理。
  - <u>有些时候，使用 `AdviceAdapter` 类的 `onMethodEnter()` 和 `onMethodExit()` 方法是不能正常工作的。比如说，一些代码经过混淆（obfuscate）之后，ByteCode的内容就会变得复杂，就会出现处理不了的情况。这个时候，我们还是应该回归到 `visitCode()` 和 `visitInsn(opcode)` 方法来解决问题</u>。

## 4.6 GeneratorAdapter介绍

对于`GeneratorAdapter`类来说，它非常重要的一个特点：将一些`visitXxxInsn()`方法封装成一些常用的方法。

### 4.6.1 GeneratorAdapter类

#### class info

第一个部分，`GeneratorAdapter`类继承自`LocalVariablesSorter`类。

- org.objectweb.asm.MethodVisitor
  - org.objectweb.asm.commons.LocalVariablesSorter
    - org.objectweb.asm.commons.GeneratorAdapter
      - org.objectweb.asm.commons.AdviceAdapter

```java
public class GeneratorAdapter extends LocalVariablesSorter {
}
```

#### fields

第二个部分，`GeneratorAdapter`类定义的字段有哪些。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  /** The access flags of the visited method. */
  private final int access;
  /** The name of the visited method. */
  private final String name;
  /** The return type of the visited method. */
  private final Type returnType;
  /** The argument types of the visited method. */
  private final Type[] argumentTypes;
}
```

#### constructors

第三个部分，`GeneratorAdapter`类定义的构造方法有哪些。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  public GeneratorAdapter(final MethodVisitor methodVisitor,
                          final int access, final String name, final String descriptor) {
    this(Opcodes.ASM9, methodVisitor, access, name, descriptor);
  }

  protected GeneratorAdapter(final int api, final MethodVisitor methodVisitor,
                             final int access, final String name, final String descriptor) {
    super(api, access, descriptor, methodVisitor);
    this.access = access;
    this.name = name;
    this.returnType = Type.getReturnType(descriptor);
    this.argumentTypes = Type.getArgumentTypes(descriptor);
  }
}
```

#### methods

第四个部分，`GeneratorAdapter`类定义的方法有哪些。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  public int getAccess() {
    return access;
  }

  public String getName() {
    return name;
  }

  public Type getReturnType() {
    return returnType;
  }

  public Type[] getArgumentTypes() {
    return argumentTypes.clone();
  }
}
```

### 4.6.2 特殊方法举例

`GeneratorAdapter`类的特点是将一些`visitXxxInsn()`方法封装成一些常用的方法。在这里，我们给大家举几个有代表性的例子，更多的方法可以参考`GeneratorAdapter`类的源码。

#### loadThis

在`GeneratorAdapter`类当中，`loadThis()`方法的本质是`mv.visitVarInsn(Opcodes.ALOAD, 0)`；但是，要注意static方法并不需要`this`变量。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  /** Generates the instruction to load 'this' on the stack. */
  public void loadThis() {
    if ((access & Opcodes.ACC_STATIC) != 0) { // 注意，静态方法没有this
      throw new IllegalStateException("no 'this' pointer within static method");
    }
    mv.visitVarInsn(Opcodes.ALOAD, 0);
  }
}
```

#### arg

在`GeneratorAdapter`类当中，定义了一些与方法参数相关的方法。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  private int getArgIndex(final int arg) {
    int index = (access & Opcodes.ACC_STATIC) == 0 ? 1 : 0;
    for (int i = 0; i < arg; i++) {
      index += argumentTypes[i].getSize();
    }
    return index;
  }

  private void loadInsn(final Type type, final int index) {
    mv.visitVarInsn(type.getOpcode(Opcodes.ILOAD), index);
  }

  private void storeInsn(final Type type, final int index) {
    mv.visitVarInsn(type.getOpcode(Opcodes.ISTORE), index);
  }

  public void loadArg(final int arg) {
    loadInsn(argumentTypes[arg], getArgIndex(arg));
  }

  public void loadArgs(final int arg, final int count) {
    int index = getArgIndex(arg);
    for (int i = 0; i < count; ++i) {
      Type argumentType = argumentTypes[arg + i];
      loadInsn(argumentType, index);
      index += argumentType.getSize();
    }
  }

  public void loadArgs() {
    loadArgs(0, argumentTypes.length);
  }

  public void loadArgArray() {
    push(argumentTypes.length);
    newArray(OBJECT_TYPE);
    for (int i = 0; i < argumentTypes.length; i++) {
      dup();
      push(i);
      loadArg(i);
      box(argumentTypes[i]);
      arrayStore(OBJECT_TYPE);
    }
  }

  public void storeArg(final int arg) {
    storeInsn(argumentTypes[arg], getArgIndex(arg));
  }
}
```

#### boxing and unboxing

在`GeneratorAdapter`类当中，定义了一些与boxing和unboxing相关的操作。

```java
public class GeneratorAdapter extends LocalVariablesSorter {
  public void box(final Type type) {
    if (type.getSort() == Type.OBJECT || type.getSort() == Type.ARRAY) {
      return;
    }
    if (type == Type.VOID_TYPE) {
      push((String) null);
    } else {
      Type boxedType = getBoxedType(type);
      newInstance(boxedType);
      if (type.getSize() == 2) {
        dupX2();
        dupX2();
        pop();
      } else {
        dupX1();
        swap();
      }
      invokeConstructor(boxedType, new Method("<init>", Type.VOID_TYPE, new Type[] {type}));
    }
  }

  public void unbox(final Type type) {
    Type boxedType = NUMBER_TYPE;
    Method unboxMethod;
    switch (type.getSort()) {
      case Type.VOID:
        return;
      case Type.CHAR:
        boxedType = CHARACTER_TYPE;
        unboxMethod = CHAR_VALUE;
        break;
      case Type.BOOLEAN:
        boxedType = BOOLEAN_TYPE;
        unboxMethod = BOOLEAN_VALUE;
        break;
      case Type.DOUBLE:
        unboxMethod = DOUBLE_VALUE;
        break;
      case Type.FLOAT:
        unboxMethod = FLOAT_VALUE;
        break;
      case Type.LONG:
        unboxMethod = LONG_VALUE;
        break;
      case Type.INT:
      case Type.SHORT:
      case Type.BYTE:
        unboxMethod = INT_VALUE;
        break;
      default:
        unboxMethod = null;
        break;
    }
    if (unboxMethod == null) {
      checkCast(type);
    } else {
      checkCast(boxedType);
      invokeVirtual(boxedType, unboxMethod);
    }
  }

  public void valueOf(final Type type) {
    if (type.getSort() == Type.OBJECT || type.getSort() == Type.ARRAY) {
      return;
    }
    if (type == Type.VOID_TYPE) {
      push((String) null);
    } else {
      Type boxedType = getBoxedType(type);
      invokeStatic(boxedType, new Method("valueOf", boxedType, new Type[] {type}));
    }
  }

  private static Type getBoxedType(final Type type) {
    switch (type.getSort()) {
      case Type.BYTE:
        return BYTE_TYPE;
      case Type.BOOLEAN:
        return BOOLEAN_TYPE;
      case Type.SHORT:
        return SHORT_TYPE;
      case Type.CHAR:
        return CHARACTER_TYPE;
      case Type.INT:
        return INTEGER_TYPE;
      case Type.FLOAT:
        return FLOAT_TYPE;
      case Type.LONG:
        return LONG_TYPE;
      case Type.DOUBLE:
        return DOUBLE_TYPE;
      default:
        return type;
    }
  }
}
```

### 4.6.3 示例：生成类

#### 预期目标

我们想实现的预期目标：生成一个`HelloWorld`类，代码如下所示。

```java
public class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello World!");
  }
}
```

#### 编码实现

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.GeneratorAdapter;
import org.objectweb.asm.commons.Method;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.PrintStream;
import java.io.PrintWriter;

import static org.objectweb.asm.Opcodes.*;

public class GeneratorAdapterExample01 {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    PrintWriter printWriter = new PrintWriter(System.out);
    TraceClassVisitor cv = new TraceClassVisitor(cw, printWriter);

    cv.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld", null, "java/lang/Object", null);

    {
      Method m1 = Method.getMethod("void <init> ()");
      GeneratorAdapter mg = new GeneratorAdapter(ACC_PUBLIC, m1, null, null, cv);
      mg.loadThis();
      mg.invokeConstructor(Type.getType(Object.class), m1);
      mg.returnValue();
      mg.endMethod();
    }

    {
      Method m2 = Method.getMethod("void main (String[])");
      GeneratorAdapter mg = new GeneratorAdapter(ACC_PUBLIC + ACC_STATIC, m2, null, null, cv);
      mg.getStatic(Type.getType(System.class), "out", Type.getType(PrintStream.class));
      mg.push("Hello World!");
      mg.invokeVirtual(Type.getType(PrintStream.class), Method.getMethod("void println (String)"));
      mg.returnValue();
      mg.endMethod();
    }

    cv.visitEnd();

    return cw.toByteArray();
  }
}
```

### 4.6.4 总结

本文对`GeneratorAdapter`类进行介绍，内容总结如下：

- **第一点，`GeneratorAdapter`类的特点是将一些`visitXxxInsn()`方法封装成一些常用的方法**。
- <u>第二点，`GeneratorAdapter`类定义的新方法，并不是十分必要的；如果熟悉`MethodVisitor.visitXxxInsn()`方法，可以完全不考虑使用`GeneratorAdapter`类。</u>

## 4.7 LocalVariablesSorter介绍

对于`LocalVariablesSorter`类来说，它的特点是“可以引入新的局部变量，并且能够对局部变量重新排序”。

### 4.7.1 LocalVariablesSorter类

#### class info

第一个部分，`LocalVariablesSorter`类继承自`MethodVisitor`类。

- org.objectweb.asm.MethodVisitor
  - org.objectweb.asm.commons.LocalVariablesSorter
    - org.objectweb.asm.commons.GeneratorAdapter
      - org.objectweb.asm.commons.AdviceAdapter

```java
/**
 * A {@link MethodVisitor} that renumbers local variables in their order of appearance. This adapter
 * allows one to easily add new local variables to a method. It may be used by inheriting from this
 * class, but the preferred way of using it is via delegation: the next visitor in the chain can
 * indeed add new locals when needed by calling {@link #newLocal} on this adapter (this requires a
 * reference back to this {@link LocalVariablesSorter}).
 *
 * @author Chris Nokleberg
 * @author Eugene Kuleshov
 * @author Eric Bruneton
 */
public class LocalVariablesSorter extends MethodVisitor {
}
```

#### fields

第二个部分，`LocalVariablesSorter`类定义的字段有哪些。在理解`LocalVariablesSorter`类时，一个要记住的核心点：**处理好”新变量“与”旧变量“的位置关系**。换句话说，要给”新变量“在local variables当中找一个位置存储，”旧变量“也要在local variables当中找一个位置存储，它们的位置不能发生冲突。对于local variables当中某一个具体的位置，要么存储的是”新变量“，要么存储的是”旧变量“，不可能在同一个位置既存储”新变量“，又存储”旧变量“。

- `remappedVariableIndices`字段，是一个`int[]`数组，其中所有元素的初始值为0。
  - `remappedVariableIndices`字段的作用：**只关心“旧变量”，它记录“旧变量”的新位置**。
  - `remappedVariableIndices`字段使用的算法，有点奇怪和特别。
- `remappedLocalTypes`字段，将“旧变量”和“新变量”整合到一起之后，记录它们的类型信息。
- `firstLocal`字段，记录“方法体”中“第一个变量”在local variables当中的索引值，由于带有`final`标识，所以赋值之后，就不再发生变化了。(即记录除了non-static方法才有的this指针，方法的入参外，方法体中出现的需要存在local variables的变量)
- `nextLocal`字段，记录local variables中可以未分配变量的位置，无论是“新变量”，还是“旧变量”，它们都是由`nextLocal`字段来分配位置；分配变量之后，`nextLocal`字段值会发生变化，重新指向local variables中未分配变量的位置。

```java
public class LocalVariablesSorter extends MethodVisitor {
  // The mapping from old to new local variable indices.
  // A local variable at index i of size 1 is remapped to 'mapping[2*i]',
  // while a local variable at index i of size 2 is remapped to 'mapping[2*i+1]'.
  private int[] remappedVariableIndices = new int[40];

  // The local variable types after remapping.
  private Object[] remappedLocalTypes = new Object[20];
	/** The index of the first local variable, after formal parameters. */
  protected final int firstLocal;
  /** The index of the next local variable to be created by {@link #newLocal}. */
  protected int nextLocal;
}
```

#### constructors

第三个部分，`LocalVariablesSorter`类定义的构造方法有哪些。

```java
public class LocalVariablesSorter extends MethodVisitor {
  public LocalVariablesSorter(final int access, final String descriptor, final MethodVisitor methodVisitor) {
    this(Opcodes.ASM9, access, descriptor, methodVisitor);
  }

  protected LocalVariablesSorter(final int api, final int access, final String descriptor,
                                 final MethodVisitor methodVisitor) {
    super(api, methodVisitor);
    nextLocal = (Opcodes.ACC_STATIC & access) == 0 ? 1 : 0;
    for (Type argumentType : Type.getArgumentTypes(descriptor)) {
      nextLocal += argumentType.getSize();
    }
    firstLocal = nextLocal;
  }
}
```

#### methods

第四个部分，`LocalVariablesSorter`类定义的方法有哪些。`LocalVariablesSorter`类要处理好“新变量”与“旧变量”之间的关系。

##### newLocal method

`newLocal()`方法就是为“新变量”来分配位置。

```java
/**
   * Constructs a new local variable of the given type.
   *
   * @param type the type of the local variable to be created.
   * @return the identifier of the newly created local variable.
   */
public class LocalVariablesSorter extends MethodVisitor {
  public int newLocal(final Type type) {
    Object localType;
    switch (type.getSort()) {
      case Type.BOOLEAN:
      case Type.CHAR:
      case Type.BYTE:
      case Type.SHORT:
      case Type.INT:
        localType = Opcodes.INTEGER;
        break;
      case Type.FLOAT:
        localType = Opcodes.FLOAT;
        break;
      case Type.LONG:
        localType = Opcodes.LONG;
        break;
      case Type.DOUBLE:
        localType = Opcodes.DOUBLE;
        break;
      case Type.ARRAY:
        localType = type.getDescriptor();
        break;
      case Type.OBJECT:
        localType = type.getInternalName();
        break;
      default:
        throw new AssertionError();
    }
    int local = newLocalMapping(type);
    setLocalType(local, type);
    setFrameLocal(local, localType);
    return local;
  }

  protected int newLocalMapping(final Type type) {
    int local = nextLocal;
    nextLocal += type.getSize();
    return local;
  }

  protected void setLocalType(final int local, final Type type) {
    // The default implementation does nothing.
  }

  private void setFrameLocal(final int local, final Object type) {
    int numLocals = remappedLocalTypes.length;
    if (local >= numLocals) { // 这里是处理分配空间不足的情况
      Object[] newRemappedLocalTypes = new Object[Math.max(2 * numLocals, local + 1)];
      System.arraycopy(remappedLocalTypes, 0, newRemappedLocalTypes, 0, numLocals);
      remappedLocalTypes = newRemappedLocalTypes;
    }
    remappedLocalTypes[local] = type; // 真正的处理逻辑只有这一句代码
  }
}
```

##### local variables method

`visitVarInsn()`和`visitIincInsn()`方法就是为“旧变量”来重新分配位置，这两个方法都会去调用`remap(var, type)`方法。

> 这里只涉及两个方法，因为对于local variables的操作可以归类3种：
>
> 1. 将变量从operand stack存到local variables
> 2. 将变量从local variables加载到operand stack
> 3. local variables的某个索引位置的变量自增某个值

```java
public class LocalVariablesSorter extends MethodVisitor {
  @Override
  public void visitVarInsn(final int opcode, final int var) {
    Type varType;
    switch (opcode) {
      case Opcodes.LLOAD:
      case Opcodes.LSTORE:
        varType = Type.LONG_TYPE;
        break;
      case Opcodes.DLOAD:
      case Opcodes.DSTORE:
        varType = Type.DOUBLE_TYPE;
        break;
      case Opcodes.FLOAD:
      case Opcodes.FSTORE:
        varType = Type.FLOAT_TYPE;
        break;
      case Opcodes.ILOAD:
      case Opcodes.ISTORE:
        varType = Type.INT_TYPE;
        break;
      case Opcodes.ALOAD:
      case Opcodes.ASTORE:
      case Opcodes.RET:
        varType = OBJECT_TYPE;
        break;
      default:
        throw new IllegalArgumentException("Invalid opcode " + opcode);
    }
    super.visitVarInsn(opcode, remap(var, varType));
  }

  @Override
  public void visitIincInsn(final int var, final int increment) {
    super.visitIincInsn(remap(var, Type.INT_TYPE), increment);
  }

  private int remap(final int var, final Type type) {
    // 第一部分，处理方法的输入参数
    if (var + type.getSize() <= firstLocal) {
      return var;
    }

    // 第二部分，处理方法体内定义的局部变量
    int key = 2 * var + type.getSize() - 1;
    int size = remappedVariableIndices.length;
    if (key >= size) { // 这段代码，主要是处理分配空间不足的情况。我们可以假设分配的空间一直是足够的，那么可以忽略此段代码
      int[] newRemappedVariableIndices = new int[Math.max(2 * size, key + 1)];
      System.arraycopy(remappedVariableIndices, 0, newRemappedVariableIndices, 0, size);
      remappedVariableIndices = newRemappedVariableIndices;
    }
    int value = remappedVariableIndices[key];
    if (value == 0) { // 如果是0，则表示还没有记录下来
      value = newLocalMapping(type);
      setLocalType(value, type);
      remappedVariableIndices[key] = value + 1;
    } else { // 如果不是0，则表示有具体的值
      value--;
    }
    return value;
  }

  protected int newLocalMapping(final Type type) {
    int local = nextLocal;
    nextLocal += type.getSize();
    return local;
  }
}
```

### 4.7.2 工作原理

对于`LocalVariablesSorter`类的工作原理，主要依赖于三个字段：`firstLocal`、`nextLocal`和`remappedVariableIndices`字段。

```java
public class LocalVariablesSorter extends MethodVisitor {
  // The mapping from old to new local variable indices.
  // A local variable at index i of size 1 is remapped to 'mapping[2*i]',
  // while a local variable at index i of size 2 is remapped to 'mapping[2*i+1]'.
  private int[] remappedVariableIndices = new int[40];

  protected final int firstLocal;
  protected int nextLocal;
}
```

首先，我们来看一下`firstLocal`和`nextLocal`初始化，它发生在`LocalVariablesSorter`类的构造方法中。其中，`firstLocal`是一个`final`类型的字段，一次赋值之后就不能变化了；而`nextLocal`字段的取值可以继续变化。

```java
public class LocalVariablesSorter extends MethodVisitor {
  protected LocalVariablesSorter(final int api, final int access, final String descriptor,
                                 final MethodVisitor methodVisitor) {
    super(api, methodVisitor);
    nextLocal = (Opcodes.ACC_STATIC & access) == 0 ? 1 : 0; // 首先，判断是不是静态方法
    for (Type argumentType : Type.getArgumentTypes(descriptor)) { // 接着，循环方法接收的参数
      nextLocal += argumentType.getSize();
    }
    firstLocal = nextLocal; // 最后，为firstLocal字段赋值。
  }
}
```

对于上面的代码，主要是对两方面内容进行判断：

- 第一方面，是否需要处理`this`变量。
- 第二方面，对方法接收的参数进行处理。

在执行完`LocalVariablesSorter`类的构造方法后，`firstLocal`和`nextLocal`的值是一样的，其值表示下一个方法体中的变量在local variables当中的位置。接下来，就是该考虑第三方面的事情了：

- 第三方面，方法体内定义的变量。对于这些变量，又分成两种情况：
  - 第一种情况，程序代码中原来定义的变量。
  - 第二种情况，程序代码中新定义的变量。

**对于`LocalVariablesSorter`类来说，它要处理的一个关键性的工作，就是处理好“旧变量”和“新变量”之间的关系**。其实，不管是“新变量”，还是“旧变量”，它都是通过`newLocalMapping(type)`方法来找到自己的位置。**`newLocalMapping(type)`方法的逻辑就是“先到先得”**。有一个形象的例子，可以帮助我们理解`newLocalMapping(type)`方法的作用。高考之后，过一段时间，大学就会开学，新生就会来报到；不管新学生来自于什么地方，第一个来到学校的学生就分配`001`的编号，第二个来到学校的学生就分配`002`的编号，依此类推。

我们先来说明第二种情况，也就是在程序代码中添加新的变量。

#### 添加新变量

如果要添加新的变量，那么需要调用`newLocal(type)`方法。

- 在`newLocal(type)`方法中，它会进一步调用`newLocalMapping(type)`方法；
- 在`newLocalMapping(type)`方法中，首先会记录`nextLocal`的值到`local`局部变量中，接着会更新`nextLocal`的值（即加上`type.getSize()`的值），最后返回`local`的值。那么，`local`的值就是新变量在local variables当中存储的位置。

```java
public class LocalVariablesSorter extends MethodVisitor {
  public int newLocal(final Type type) {
    int local = newLocalMapping(type);
    return local;
  }

  protected int newLocalMapping(final Type type) {
    int local = nextLocal;
    nextLocal += type.getSize();
    return local;
  }
}
```

#### 处理旧变量

如果要处理“旧变量”，那么需要调用`visitVarInsn(opcode, var)`或`visitIincInsn(var, increment)`方法。在这两个方法中，会进一步调用`remap(var, type)`方法。其中，`remap(var, type)`方法的主要作用，就是实现“旧变量”的原位置向新位置的映射。

```java
public class LocalVariablesSorter extends MethodVisitor {
  @Override
  public void visitVarInsn(final int opcode, final int var) {
    Type varType;
    switch (opcode) {
      case Opcodes.LLOAD:
      case Opcodes.LSTORE:
        varType = Type.LONG_TYPE;
        break;
      case Opcodes.DLOAD:
      case Opcodes.DSTORE:
        varType = Type.DOUBLE_TYPE;
        break;
      case Opcodes.FLOAD:
      case Opcodes.FSTORE:
        varType = Type.FLOAT_TYPE;
        break;
      case Opcodes.ILOAD:
      case Opcodes.ISTORE:
        varType = Type.INT_TYPE;
        break;
      case Opcodes.ALOAD:
      case Opcodes.ASTORE:
      case Opcodes.RET:
        varType = OBJECT_TYPE;
        break;
      default:
        throw new IllegalArgumentException("Invalid opcode " + opcode);
    }
    super.visitVarInsn(opcode, remap(var, varType));
  }

  @Override
  public void visitIincInsn(final int var, final int increment) {
    super.visitIincInsn(remap(var, Type.INT_TYPE), increment);
  }

  private int remap(final int var, final Type type) {
    // 第一部分，处理方法的输入参数
    if (var + type.getSize() <= firstLocal) {
      return var;
    }

    // 第二部分，处理方法体内定义的局部变量
    int key = 2 * var + type.getSize() - 1;
    int value = remappedVariableIndices[key];
    if (value == 0) { // 如果是0，则表示还没有记录下来
      value = newLocalMapping(type);
      remappedVariableIndices[key] = value + 1;
    } else { // 如果不是0，则表示有具体的值
      value--;
    }
    return value;
  }

  protected int newLocalMapping(final Type type) {
    int local = nextLocal;
    nextLocal += type.getSize();
    return local;
  }
}
```

在`remap(var, type)`方法中，有两部分主要逻辑：

- 第一部分，是处理方法的输入参数。方法接收的参数，它们在local variables当中的索引位置是不会变化的，所以处理起来也比较简单，直接返回`var`的值。
- 第二部分，是处理方法体内定义的局部变量。在这个部分，就是`remappedVariableIndices`字段发挥作用的地方，也会涉及到`nextLocal`字段。

在`remap(var, type)`方法中，我们重点关注第二部分，代码处理的步骤是：

- 第一步，计算出`remappedVariableIndices`字段的一个索引值`key`，即`int key = 2 * var + type.getSize() - 1`。假设有一个变量的索引是`i`，如果该变量的大小是1，那么它在`remappedVariableIndices`字段中的索引位置是`2*i`；如果该变量（`long`或`double`类型）的大小是2，那么它在`remappedVariableIndices`字段中的索引位置是`2*i+1`。（因为long和double需要占用2个slot，第二个slot用top标记占用）
- 第二步，根据`key`值，取出`remappedVariableIndices`字段当中的`value`值。大家注意，`int[] remappedVariableIndices = new int[40]`，也就是说，`remappedVariableIndices`字段是一个数组，所有元素的默认值是0。
  - 如果`value`的值是`0`，说明还没有记录“旧变量”的新位置；那么，就通过`value = newLocalMapping(type)`计算出新的位置，将`value + 1`赋值给`remappedVariableIndices`字段中`key`位置。
  - 如果`value`的值不是`0`，说明已经记录“旧变量”的新位置；这个时候，要进行`value--`操作。
- 第三步，返回`value`的值。那么，这个`value`值就是“旧变量”的新位置。

在上面的代码当中，我们可以看到`remap`方法里有`value + 1`和`value--`的代码：

![img](https://lsieun.github.io/assets/images/java/asm/local-variable-sorter-remap-plus-one-minus-one.png)

<u>为什么进行这样的处理呢？我们来思考这样的问题：当创建一个新的`int[]`时，其中的每一个元素的默认值都是`0`；在local variable当中，0是一个有效的索引值； 那么，如果从`int[]`数组当中取出一个元素，它的值是`0`，那它是代表元素的“默认值”，还是local variable当中的一个有效的索引值`0`呢？ 为了进行区分，它加一个`offset`值，而在代码中这个`offset`的值是`1`，我觉得，将`offset`取值成`100`也能得到一个正确的结果</u>。

### 4.7.3 示例

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
import java.util.Random;

public class HelloWorld {
  public void test(int a, int b) throws Exception {
    int c = a + b;
    int d = c * 10;
    Random rand = new Random();
    int value = rand.nextInt(d);
    Thread.sleep(value);
  }
}
```

我们想实现的预期目标：添加一个新的局部变量`t`，然后使用变量`t`计算方法的运行时间。

```java
import java.util.Random;

public class HelloWorld {
  public void test(int a, int b) throws Exception {
    long t = System.currentTimeMillis();

    int c = a + b;
    int d = c * 10;
    Random rand = new Random();
    int value = rand.nextInt(d);
    Thread.sleep(value);

    t = System.currentTimeMillis() - t;
    System.out.println("test method execute: " + t);
  }
}
```

#### 编码实现

注意点：

+ `slotIndex`用来记录新增的本地变量在local variables中的下标索引值

下面的`MethodTimerAdapter3`类继承自`LocalVariablesSorter`类。

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.LocalVariablesSorter;

import static org.objectweb.asm.Opcodes.*;

public class MethodTimerVisitor3 extends ClassVisitor {
  public MethodTimerVisitor3(int api, ClassVisitor cv) {
    super(api, cv);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);

    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodTimerAdapter3(api, access, name, descriptor, mv);
      }
    }
    return mv;
  }

  private static class MethodTimerAdapter3 extends LocalVariablesSorter {
    private final String methodName;
    private final String methodDesc;
    private int slotIndex;

    public MethodTimerAdapter3(int api, int access, String name, String descriptor, MethodVisitor methodVisitor) {
      super(api, access, descriptor, methodVisitor);
      this.methodName = name;
      this.methodDesc = descriptor;
    }

    @Override
    public void visitCode() {
      // 首先，实现自己的逻辑
      slotIndex = newLocal(Type.LONG_TYPE);
      mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
      mv.visitVarInsn(LSTORE, slotIndex);

      // 其次，调用父类的实现
      super.visitCode();
    }

    @Override
    public void visitInsn(int opcode) {
      // 首先，实现自己的逻辑
      if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        mv.visitVarInsn(LLOAD, slotIndex);
        mv.visitInsn(LSUB);
        mv.visitVarInsn(LSTORE, slotIndex);
        mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
        mv.visitInsn(DUP);
        mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
        mv.visitLdcInsn(methodName + methodDesc + " method execute: ");
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitVarInsn(LLOAD, slotIndex);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      }

      // 其次，调用父类的实现
      super.visitInsn(opcode);
    }
  }
}
```

**需要注意的是，我们使用的是`mv.visitVarInsn(opcode, var)`方法，而不是使用`super.visitVarInsn(opcode, var)`方法。为什么要使用`mv`，而不使用`super`呢？因为使用`super.visitVarInsn(opcode, var)`方法，实质上是调用了`LocalVariablesSorter.visitVarInsn(opcode, var)`，它会进一步调用`remap(var, type)`方法，这就可能导致新添加的变量在local variables中的位置发生“位置偏移”。**

下面的`MethodTimerAdapter4`类继承自`AdviceAdapter`类。

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AdviceAdapter;

import static org.objectweb.asm.Opcodes.ACC_ABSTRACT;
import static org.objectweb.asm.Opcodes.ACC_NATIVE;

public class MethodTimerVisitor4 extends ClassVisitor {
  public MethodTimerVisitor4(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodTimerAdapter4(api, mv, access, name, descriptor);
      }
    }
    return mv;
  }

  private static class MethodTimerAdapter4 extends AdviceAdapter {
    private int slotIndex;

    public MethodTimerAdapter4(int api, MethodVisitor mv, int access, String name, String descriptor) {
      super(api, mv, access, name, descriptor);
    }

    @Override
    protected void onMethodEnter() {
      slotIndex = newLocal(Type.LONG_TYPE);
      mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
      mv.visitVarInsn(LSTORE, slotIndex);
    }

    @Override
    protected void onMethodExit(int opcode) {
      if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        mv.visitVarInsn(LLOAD, slotIndex);
        mv.visitInsn(LSUB);
        mv.visitVarInsn(LSTORE, slotIndex);
        mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
        mv.visitInsn(DUP);
        mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
        mv.visitLdcInsn(getName() + methodDesc + " method execute: ");
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitVarInsn(LLOAD, slotIndex);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      }
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodTimerVisitor4(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test(10, 20);
  }
}
```

### 4.7.4 总结

本文对`LocalVariablesSorter`类进行介绍，内容总结如下：

- 第一点，了解`LocalVariablesSorter`类的各个部分，都有哪些信息。
- 第二点，理解`LocalVariablesSorter`类的工作原理。
- 第三点，如何使用`LocalVariablesSorter`类添加新的变量。

## 4.8 AnalyzerAdapter介绍

对于`AnalyzerAdapter`类来说，它的特点是“可以模拟frame的变化”，或者说“可以模拟local variables和operand stack的变化”。

The `AnalyzerAdapter` is a `MethodVisitor` that keeps track of stack map frame changes between `visitFrame(int, int, Object[], int, Object[])` calls. **This `AnalyzerAdapter` adapter must be used with the `ClassReader.EXPAND_FRAMES` option**.

This method adapter computes the **stack map frames** before each instruction, based on the frames visited in `visitFrame`. Indeed, `visitFrame` is only called before some specific instructions in a method, in order to save space, and because “the other frames can be easily and quickly inferred from these ones”. This is what this adapter does.

![Java ASM系列：（040）AnalyzerAdapter介绍_ByteCode](https://s2.51cto.com/images/20210623/1624446269398923.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 4.8.1 AnalyzerAdapter类

#### class info

第一个部分，`AnalyzerAdapter`类的父类是`MethodVisitor`类。

```java
public class AnalyzerAdapter extends MethodVisitor {
}
```

#### fields

第二个部分，`AnalyzerAdapter`类定义的字段有哪些。

我们将以下列出的字段分成3个组：

- 第1组，包括`locals`、`stack`、`maxLocals`和`maxStack`字段，它们是与local variables和operand stack直接相关的字段。
- <u>第2组，包括`labels`和`uninitializedTypes`字段，它们记录的是未初始化的对象类型，是属于一些特殊情况</u>。
- 第3组，是`owner`字段，表示当前类的名字。

```java
public class AnalyzerAdapter extends MethodVisitor {
  // 第1组字段：local variables和operand stack
  public List<Object> locals;
  public List<Object> stack;
  private int maxLocals;
  private int maxStack;

  // 第2组字段：uninitialized类型
  private List<Label> labels;
  public Map<Object, Object> uninitializedTypes;

  // 第3组字段：类的名字
  private String owner;
}
```

#### constructors

第三个部分，`AnalyzerAdapter`类定义的构造方法有哪些。

有一个问题：`AnalyzerAdapter`类的构造方法，到底是想实现一个什么样的代码逻辑呢？回答：它想<u>构建方法刚进入时的Frame状态</u>。在方法刚进入时，Frame的初始状态是什么样的呢？其中，operand stack上没有任何元素，而local variables则需要考虑存储`this`和方法的参数信息。在`AnalyzerAdapter`类的构造方法中，主要就是围绕着`locals`字段来展开，它需要将`this`和方法参数添加进入。

```java
public class AnalyzerAdapter extends MethodVisitor {
  public AnalyzerAdapter(String owner, int access, String name, String descriptor, MethodVisitor methodVisitor) {
    this(Opcodes.ASM9, owner, access, name, descriptor, methodVisitor);
  }

  protected AnalyzerAdapter(int api, String owner, int access, String name, String descriptor, MethodVisitor methodVisitor) {
    super(api, methodVisitor);
    this.owner = owner;
    locals = new ArrayList<>();
    stack = new ArrayList<>();
    uninitializedTypes = new HashMap<>();

    // 首先，判断是不是static方法、是不是构造方法，来更新local variables的初始状态
    if ((access & Opcodes.ACC_STATIC) == 0) {
      if ("<init>".equals(name)) {
        locals.add(Opcodes.UNINITIALIZED_THIS);
      } else {
        locals.add(owner);
      }
    }

    // 其次，根据方法接收的参数，来更新local variables的初始状态
    for (Type argumentType : Type.getArgumentTypes(descriptor)) {
      switch (argumentType.getSort()) {
        case Type.BOOLEAN:
        case Type.CHAR:
        case Type.BYTE:
        case Type.SHORT:
        case Type.INT:
          locals.add(Opcodes.INTEGER);
          break;
        case Type.FLOAT:
          locals.add(Opcodes.FLOAT);
          break;
        case Type.LONG:
          locals.add(Opcodes.LONG);
          locals.add(Opcodes.TOP);
          break;
        case Type.DOUBLE:
          locals.add(Opcodes.DOUBLE);
          locals.add(Opcodes.TOP);
          break;
        case Type.ARRAY:
          locals.add(argumentType.getDescriptor());
          break;
        case Type.OBJECT:
          locals.add(argumentType.getInternalName());
          break;
        default:
          throw new AssertionError();
      }
    }
    maxLocals = locals.size();
  }
}
```

#### methods

第四个部分，`AnalyzerAdapter`类定义的方法有哪些。

##### execute方法

在`AnalyzerAdapter`类当中，多数的`visitXxxInsn()`方法都会去调用`execute()`方法；而`execute()`方法是模拟每一条instruction对于local variables和operand stack的影响。

```java
public class AnalyzerAdapter extends MethodVisitor {
  private void execute(final int opcode, final int intArg, final String stringArg) {
    // ......
  }
}
```

##### return和throw

当遇到`return`或`throw`时，会将`locals`字段和`stack`字段设置为`null`。如果遇到`return`之后，就代表了“正常结束”，方法的代码执行结束了；如果遇到`throw`之后，就代表了“出现异常”，方法处理不了某种情况而退出。

```java
public class AnalyzerAdapter extends MethodVisitor {
  // 这里对应return语句
  public void visitInsn(final int opcode) {
    super.visitInsn(opcode);
    execute(opcode, 0, null);
    if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN) || opcode == Opcodes.ATHROW) {
      this.locals = null;
      this.stack = null;
    }
  }
}
```

##### jump

当遇到`goto`、`switch`（`tableswitch`和`lookupswitch`）时，也会将`locals`字段和`stack`字段设置为`null`。遇到jump相关的指令，意味着代码的逻辑要进行“跳转”，从一个地方跳转到另一个地方执行。

```java
public class AnalyzerAdapter extends MethodVisitor {
  // 这里对应goto语句
  public void visitJumpInsn(final int opcode, final Label label) {
    super.visitJumpInsn(opcode, label);
    execute(opcode, 0, null);
    if (opcode == Opcodes.GOTO) {
      this.locals = null;
      this.stack = null;
    }
  }

  // 这里对应switch语句
  public void visitTableSwitchInsn(int min, int max, Label dflt, Label... labels) {
    super.visitTableSwitchInsn(min, max, dflt, labels);
    execute(Opcodes.TABLESWITCH, 0, null);
    this.locals = null;
    this.stack = null;
  }

  // 这里对应switch语句
  public void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels) {
    super.visitLookupSwitchInsn(dflt, keys, labels);
    execute(Opcodes.LOOKUPSWITCH, 0, null);
    this.locals = null;
    this.stack = null;
  }
}
```

##### visitFrame方法

当遇到jump相关的指令后，程序的代码会发生跳转。那么，跳转到新位置之后，就需要给local variables和operand stack重新设置一个新的状态；而`visitFrame()`方法，是将local variables和operand stack设置成某一个状态。跳转之后的代码，就是在这个新状态的基础上发生变化。

```java
public class AnalyzerAdapter extends MethodVisitor {
  public void visitFrame(int type, int numLocal, Object[] local, int numStack, Object[] stack) {
    if (type != Opcodes.F_NEW) { // Uncompressed frame.
      throw new IllegalArgumentException("AnalyzerAdapter only accepts expanded frames (see ClassReader.EXPAND_FRAMES)");
    }

    super.visitFrame(type, numLocal, local, numStack, stack);

    // 先清空
    if (this.locals != null) {
      this.locals.clear();
      this.stack.clear();
    } else {
      this.locals = new ArrayList<>();
      this.stack = new ArrayList<>();
    }

    // 再重新设置
    visitFrameTypes(numLocal, local, this.locals);
    visitFrameTypes(numStack, stack, this.stack);
    maxLocals = Math.max(maxLocals, this.locals.size());
    maxStack = Math.max(maxStack, this.stack.size());
  }

  private static void visitFrameTypes(int numTypes, Object[] frameTypes, List<Object> result) {
    for (int i = 0; i < numTypes; ++i) {
      Object frameType = frameTypes[i];
      result.add(frameType);
      if (frameType == Opcodes.LONG || frameType == Opcodes.DOUBLE) {
        result.add(Opcodes.TOP);
      }
    }
  }
}
```

##### new和invokespecial

在执行程序代码的时候，有些特殊的情况需要处理：

- 当遇到`new`时，会创建`Label`对象来表示“未初始化的对象”，并将label存储到`uninitializedTypes`字段内；
- 当遇到`invokespecial`时，会把“未初始化的对象”从`uninitializedTypes`字段内取出来，转换成“经过初始化之后的对象”，然后同步到`locals`字段和`stack`字段内。

```java
public class AnalyzerAdapter extends MethodVisitor {
  // 对应于new
  public void visitTypeInsn(final int opcode, final String type) {
    if (opcode == Opcodes.NEW) {
      if (labels == null) {
        Label label = new Label();
        labels = new ArrayList<>(3);
        labels.add(label);
        if (mv != null) {
          mv.visitLabel(label);
        }
      }
      for (Label label : labels) {
        uninitializedTypes.put(label, type);
      }
    }
    super.visitTypeInsn(opcode, type);
    execute(opcode, 0, type);
  }

  // 对应于invokespecial
  public void visitMethodInsn(int opcodeAndSource, String owner, String name, String descriptor, boolean isInterface) {
    super.visitMethodInsn(opcodeAndSource, owner, name, descriptor, isInterface);
    int opcode = opcodeAndSource & ~Opcodes.SOURCE_MASK;

    if (this.locals == null) {
      labels = null;
      return;
    }
    pop(descriptor);
    if (opcode != Opcodes.INVOKESTATIC) {
      Object value = pop();
      if (opcode == Opcodes.INVOKESPECIAL && name.equals("<init>")) {
        Object initializedValue;
        if (value == Opcodes.UNINITIALIZED_THIS) {
          initializedValue = this.owner;
        } else {
          initializedValue = uninitializedTypes.get(value);
        }
        for (int i = 0; i < locals.size(); ++i) {
          if (locals.get(i) == value) {
            locals.set(i, initializedValue);
          }
        }
        for (int i = 0; i < stack.size(); ++i) {
          if (stack.get(i) == value) {
            stack.set(i, initializedValue);
          }
        }
      }
    }
    pushDescriptor(descriptor);
    labels = null;
  }
}
```

### 4.8.2 工作原理

在上面的内容，我们分别介绍了`AnalyzerAdapter`类的各个部分的信息，那么在这里，我们的目标是按照一个抽象的逻辑顺序来将各个部分组织到一起。那么，这个抽象的逻辑是什么呢？就是local variables和operand stack的状态变化，从初始状态，到中间状态，再到结束状态。

一个类能够为外界提供什么样的“信息”，只要看它的`public`成员就可以了。如果我们仔细观察一下`AnalyzerAdapter`类，就会发现：除了从`MethodVisitor`类继承的`visitXxxInsn()`方法，`AnalyzerAdapter`类自己只定义了三个`public`类型的字段，即`locals`、`stack`和`uninitializedTypes`。如果我们想了解和使用`AnalyzerAdapter`类，只要把握住这三个字段就可以了。

`AnalyzerAdapter`类的主要作用就是记录stack map frame的变化情况；在frame当中，有两个重要的结构，即local variables和operand stack。结合刚才的三个字段，其中`locals`和`stack`分别表示local variables和operand stack；而`uninitializedTypes`则是记录一种特殊的状态，这个状态就是“对象已经通过new创建了，但是还没有调用它的构造方法”，这个状态只是一个“临时”的状态，等后续调用它的构造方法之后，它就是一个真正意义上的对象了。举一个例子，一个人拿到了大学录取通知书，可以笼统的叫作”大学生“，但是还不是真正意义上的”大学生“，是一种”临时“的过渡状态，等到去大学报到之后，才成为真正意义上的大学生。

```java
public class AnalyzerAdapter extends MethodVisitor {
    // 第1组字段：local variables和operand stack
    public List<Object> locals;
    public List<Object> stack;

    // 第2组字段：uninitialized类型
    private List<Label> labels;
    public Map<Object, Object> uninitializedTypes;
}
```

我们在研究local variables和operand stack的变化时，遵循下面的思路就可以了：

- 首先，初始状态。也就是说，最开始的时候，local variables和operand stack是如何布局的。
- 其次，中间状态。local variables和operand stack会随着Instruction的执行而发生变化。按照Instruction执行的顺序，我们这里又分成两种情况：
  - 第一种情况，Instruction按照顺序一条一条的向下执行。在这第一种情况里，还有一种特殊情况，就是new对象时，出现的特殊状态下的对象，也就是“已经分配内存空间，但还没有调用构造方法的对象”。
  - 第二种情况，遇到jump相关的Instruction，程序代码逻辑要发生跳转。
- 最后，结束状态。方法退出，可以是正常退出（return），也可以异常退出（throw）。

这三种状态，可以与“生命体”作一个类比。在这个世界上，大多数的生命体，都会经历出生、成长、衰老和死亡的变化。

------

**在Java语言当中，流程控制语句有三种，分别是顺序（sequential structure）、选择（selective structure）和循环（cycle structure）。但是，如果进入到ByteCode层面或Instruction层面，那么选择（selective structure）和循环（cycle structure）本质上是一样的，都是跳转（Jump）**。

#### 初始状态

首先，就是local variables和operand stack的初始状态，它是通过`AnalyzerAdapter`类的构造方法来为`locals`和`stack`字段赋值。

```java
public class AnalyzerAdapter extends MethodVisitor {
  protected AnalyzerAdapter(int api, String owner, int access, String name, String descriptor, MethodVisitor methodVisitor) {
    super(api, methodVisitor);
    this.owner = owner;
    locals = new ArrayList<>();
    stack = new ArrayList<>();
    uninitializedTypes = new HashMap<>();

    // 首先，判断是不是static方法、是不是构造方法，来更新local variables的初始状态
    if ((access & Opcodes.ACC_STATIC) == 0) {
      if ("<init>".equals(name)) {
        locals.add(Opcodes.UNINITIALIZED_THIS);
      } else {
        locals.add(owner);
      }
    }

    // 其次，根据方法接收的参数，来更新local variables的初始状态
    for (Type argumentType : Type.getArgumentTypes(descriptor)) {
      switch (argumentType.getSort()) {
        case Type.BOOLEAN:
        case Type.CHAR:
        case Type.BYTE:
        case Type.SHORT:
        case Type.INT:
          locals.add(Opcodes.INTEGER);
          break;
        case Type.FLOAT:
          locals.add(Opcodes.FLOAT);
          break;
        case Type.LONG:
          locals.add(Opcodes.LONG);
          locals.add(Opcodes.TOP);
          break;
        case Type.DOUBLE:
          locals.add(Opcodes.DOUBLE);
          locals.add(Opcodes.TOP);
          break;
        case Type.ARRAY:
          locals.add(argumentType.getDescriptor());
          break;
        case Type.OBJECT:
          locals.add(argumentType.getInternalName());
          break;
        default:
          throw new AssertionError();
      }
    }
    maxLocals = locals.size();
  }
}
```

在上面的构造方法中，operand stack的初始状态是空的；而local variables的初始状态需要考虑两方面的内容：

- 第一方面，当前方法是不是static方法、当前方法是不是`<init>()`方法。
- 第二方面，方法接收的参数。

#### 中间状态

##### 顺序执行

接着，就是instruction的执行会使得local variables和operand stack状态发生变化。在这个过程中，`visitXxxInsn()`方法大多是通过调用`execute(opcode, intArg, stringArg)`方法来完成。

```java
public class AnalyzerAdapter extends MethodVisitor {
  private void execute(final int opcode, final int intArg, final String stringArg) {
    // ......
  }
}
```

##### 发生跳转

当遇到jump相关的指令时，程序代码会从一个地方跳转到另一个地方。

**当程序跳转完成之后，需要通过`visitFrame()`方法为`locals`和`stack`字段赋一个新的初始值**。再往下执行，可能就进入到“顺序执行”的过程了。

##### 特殊情况：new对象

对于“未初始化的对象类型”，我们来举个例子，比如说`new String()`会创建一个`String`类型的对象，但是对应到ByteCode层面是3条instruction：

```shell
NEW java/lang/String
DUP
INVOKESPECIAL java/lang/String.<init> ()V
```

- 第1条instruction，是`NEW java/lang/String`，会为即将创建的对象分配内存空间，确切的说是在堆（heap）上分配内存空间，同时将一个`reference`放到operand stack上，这个`reference`就指向这块内存空间。由于这块内存空间还没有进行初始化，所以这个`reference`对应的内容并不能确切的叫作“对象”，只能叫作“未初始化的对象”，也就是“uninitialized object”。
- 第2条instruction，是`DUP`，会将operand stack上的原有的`reference`复制一份，这时候operand stack上就有两个`reference`，这两个`reference`都指向那块未初始化的内存空间，这两个`reference`的内容都对应于同一个“uninitialized object”。
- 第3条instruction，是`INVOKESPECIAL java/lang/String.<init> ()V`，会将那块内存空间进行初始化，同时会“消耗”掉operand stack最上面的`reference`，那么就只剩下一个`reference`了。由于那块内存空间进行了初始化操作，那么剩下的`reference`对应的内容就是一个“经过初始化的对象”，就是一个平常所说的“对象”了。

#### 结束状态

**从JVM内存空间的角度来说，每一个方法都有对应的frame内存空间：当方法开始的时候，就会创建相应的frame内存空间；当方法结束的时候，就会清空相应的frame内存空间。换句话说，当方法结束的时候，frame内存空间的local variables和operand stack也就被清空了。所以，从JVM内存空间的角度来说，结束状态，就是local variables和operand stack所占用的内存空间都“消失了”。**

从Java代码的角度来说，方法的退出，就对应于`visitInsn(opcode)`方法中`return`和`throw`的情况。

对于local variables和operand stack的结束状态，它又重要，又不重要：

- 它不重要，是因为它的内存空间被回收了或“消失了”，不需要我们花费太多的时间去思考它，这是从“自身所包含内容的多与少”的角度来考虑。
- 它重要，是因为它在“初始状态－中间状态－结束状态”这个环节当中是必不可少的一部分，这是从“整体性”的角度上来考虑。

### 4.8.3 示例：打印方法的Frame

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
import java.util.Random;

public class HelloWorld {
  public HelloWorld() {
    super();
  }

  public boolean getFlag() {
    Random rand = new Random();
    return rand.nextBoolean();
  }

  public void test(boolean flag) {
    if (flag) {
      System.out.println("value is true");
    }
    else {
      System.out.println("value is false");
    }
  }

  public static void main(String[] args) {
    HelloWorld instance = new HelloWorld();
    boolean flag = instance.getFlag();
    instance.test(flag);
  }
}
```

我们想实现的预期目标：打印出`HelloWorld`类当中各个方法的frame变化情况。

#### 编码实现

```java
import org.objectweb.asm.*;
import org.objectweb.asm.commons.AnalyzerAdapter;

import java.util.Arrays;
import java.util.List;

public class MethodStackMapFrameVisitor extends ClassVisitor {
  private String owner;

  public MethodStackMapFrameVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    return new MethodStackMapFrameAdapter(api, owner, access, name, descriptor, mv);
  }

  private static class MethodStackMapFrameAdapter extends AnalyzerAdapter {
    private final String methodName;
    private final String methodDesc;

    public MethodStackMapFrameAdapter(int api, String owner, int access, String name, String descriptor, MethodVisitor methodVisitor) {
      super(api, owner, access, name, descriptor, methodVisitor);
      this.methodName = name;
      this.methodDesc = descriptor;
    }

    @Override
    public void visitCode() {
      super.visitCode();
      System.out.println();
      System.out.println(methodName + methodDesc);
      printStackMapFrame();
    }

    @Override
    public void visitInsn(int opcode) {
      super.visitInsn(opcode);
      printStackMapFrame();
    }

    @Override
    public void visitIntInsn(int opcode, int operand) {
      super.visitIntInsn(opcode, operand);
      printStackMapFrame();
    }

    @Override
    public void visitVarInsn(int opcode, int var) {
      super.visitVarInsn(opcode, var);
      printStackMapFrame();
    }

    @Override
    public void visitTypeInsn(int opcode, String type) {
      super.visitTypeInsn(opcode, type);
      printStackMapFrame();
    }

    @Override
    public void visitFieldInsn(int opcode, String owner, String name, String descriptor) {
      super.visitFieldInsn(opcode, owner, name, descriptor);
      printStackMapFrame();
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
      super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
      printStackMapFrame();
    }

    @Override
    public void visitInvokeDynamicInsn(String name, String descriptor, Handle bootstrapMethodHandle, Object... bootstrapMethodArguments) {
      super.visitInvokeDynamicInsn(name, descriptor, bootstrapMethodHandle, bootstrapMethodArguments);
      printStackMapFrame();
    }

    @Override
    public void visitJumpInsn(int opcode, Label label) {
      super.visitJumpInsn(opcode, label);
      printStackMapFrame();
    }

    @Override
    public void visitLdcInsn(Object value) {
      super.visitLdcInsn(value);
      printStackMapFrame();
    }

    @Override
    public void visitIincInsn(int var, int increment) {
      super.visitIincInsn(var, increment);
      printStackMapFrame();
    }

    @Override
    public void visitTableSwitchInsn(int min, int max, Label dflt, Label... labels) {
      super.visitTableSwitchInsn(min, max, dflt, labels);
      printStackMapFrame();
    }

    @Override
    public void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels) {
      super.visitLookupSwitchInsn(dflt, keys, labels);
      printStackMapFrame();
    }

    @Override
    public void visitMultiANewArrayInsn(String descriptor, int numDimensions) {
      super.visitMultiANewArrayInsn(descriptor, numDimensions);
      printStackMapFrame();
    }

    @Override
    public void visitTryCatchBlock(Label start, Label end, Label handler, String type) {
      super.visitTryCatchBlock(start, end, handler, type);
      printStackMapFrame();
    }

    private void printStackMapFrame() {
      String locals_str = locals == null ? "[]" : list2Str(locals);
      String stack_str = stack == null ? "[]" : list2Str(stack);
      String line = String.format("%s %s", locals_str, stack_str);
      System.out.println(line);
    }

    private String list2Str(List<Object> list) {
      if (list == null || list.size() == 0) return "[]";
      int size = list.size();
      String[] array = new String[size];
      for (int i = 0; i < size; i++) {
        Object item = list.get(i);
        array[i] = item2Str(item);
      }
      return Arrays.toString(array);
    }

    private String item2Str(Object obj) {
      if (obj == Opcodes.TOP) {
        return "top";
      }
      else if (obj == Opcodes.INTEGER) {
        return "int";
      }
      else if (obj == Opcodes.FLOAT) {
        return "float";
      }
      else if (obj == Opcodes.DOUBLE) {
        return "double";
      }
      else if (obj == Opcodes.LONG) {
        return "long";
      }
      else if (obj == Opcodes.NULL) {
        return "null";
      }
      else if (obj == Opcodes.UNINITIALIZED_THIS) {
        return "uninitialized_this";
      }
      else if (obj instanceof Label) {
        Object value = uninitializedTypes.get(obj);
        if (value == null) {
          return obj.toString();
        }
        else {
          return "uninitialized_" + value;
        }
      }
      else {
        return obj.toString();
      }
    }
  }
}
```

#### 验证结果

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;

public class HelloWorldFrameCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new MethodStackMapFrameVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.EXPAND_FRAMES; // 注意，这里使用了EXPAND_FRAMES
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 4.8.4 总结

本文对`AnalyzerAdapter`类进行介绍，内容总结如下：

- 第一点，了解`AnalyzerAdapter`类的各个不同部分。
- 第二点，理解`AnalyzerAdapter`类的代码原理，它是围绕着local variables和operand stack如何变化来展开的。
- 第三点，需要注意的一点是，在使用`AnalyzerAdapter`类时，要记得用`ClassReader.EXPAND_FRAMES`选项。

`AnalyzerAdapter`类，更多的是具有“学习特性”，而不是“实用特性”。所谓的“学习特性”，具体来说，就是`AnalyzerAdapter`类让我们能够去学习local variables和operand stack随着instruction的向下执行而发生变化。所谓的“实用特性”，就是像`AdviceAdapter`类那样，它有明确的使用场景，能够在“方法进入”的时候和“方法退出”的时候来添加一些代码逻辑。

## 4.9 InstructionAdapter介绍

对于`InstructionAdapter`类来说，它的特点是“添加了许多与opcode同名的方法”，更接近“原汁原味”的JVM Instruction Set。

### 4.9.1 为什么有InstructionAdapter类

`InstructionAdapter`类继承自`MethodVisitor`类，它提供了更详细的API用于generate和transform。

在JVM Specification中，一共定义了200多个opcode，在ASM的`MethodVisitor`类当中定义了15个`visitXxxInsn()`方法。这说明一个问题，也就是在`MethodVisitor`类的每个`visitXxxInsn()`方法都会对应JVM Specification当中多个opcode。

那么，`InstructionAdapter`类起到一个什么样的作用呢？`InstructionAdapter`类继承了`MethodVisitor`类，也就继承了那些`visitXxxInsn()`方法，同时它也添加了80多个新的方法，这些新的方法与opcode更加接近。

从功能上来说，`InstructionAdapter`类和`MethodVisitor`类是一样的，两者没有差异。对于`InstructionAdapter`类来说，它可能更适合于熟悉opcode的人来使用。但是，如果我们已经熟悉`MethodVisitor`类里的`visitXxxInsn()`方法，那就完全可以不去使用`InstructionAdapter`类。

### 4.9.2 InstructionAdapter类

#### class info

第一个部分，`InstructionAdapter`类的父类是`MethodVisitor`类。

```java
/**
 * A {@link MethodVisitor} providing a more detailed API to generate and transform instructions.
 *
 * @author Eric Bruneton
 */
public class InstructionAdapter extends MethodVisitor {
}
```

#### fields

第二个部分，`InstructionAdapter`类定义的字段有哪些。我们可以看到，`InstructionAdapter`类定义了一个`OBJECT_TYPE`静态字段。

```java
public class InstructionAdapter extends MethodVisitor {
    public static final Type OBJECT_TYPE = Type.getType("Ljava/lang/Object;");
}
```

#### constructors

第三个部分，`InstructionAdapter`类定义的构造方法有哪些。

```java
public class InstructionAdapter extends MethodVisitor {
  public InstructionAdapter(final MethodVisitor methodVisitor) {
    this(Opcodes.ASM9, methodVisitor);
    if (getClass() != InstructionAdapter.class) {
      throw new IllegalStateException();
    }
  }

  protected InstructionAdapter(final int api, final MethodVisitor methodVisitor) {
    super(api, methodVisitor);
  }
}
```

#### methods

> [The Java® Virtual Machine Specification (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/index.html)

第四个部分，`InstructionAdapter`类定义的方法有哪些。除了从`MethodVisitor`类继承的`visitXxxInsn()`方法，<u>`InstructionAdapter`类还定义了许多与opcode相关的新方法，这些新方法本质上就是调用`visitXxxInsn()`方法来实现的</u>。

```java
public class InstructionAdapter extends MethodVisitor {
  public void nop() {
    mv.visitInsn(Opcodes.NOP);
  }

  public void aconst(final Object value) {
    if (value == null) {
      mv.visitInsn(Opcodes.ACONST_NULL);
    } else {
      mv.visitLdcInsn(value);
    }
  }

  // ......
}
```

### 4.9.3 示例

#### 预期目标

我们想实现的预期目标：生成如下的`HelloWorld`类。

```java
public class HelloWorld {
  public void test() {
    System.out.println("Hello World");
  }
}
```

#### 编码实现

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.InstructionAdapter;

import static org.objectweb.asm.Opcodes.*;

public class InstructionAdapterExample01 {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用visitXxx()方法
    cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, "sample/HelloWorld",
             null, "java/lang/Object", null);

    {
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
      InstructionAdapter ia = new InstructionAdapter(mv1);
      ia.visitCode();
      ia.load(0, InstructionAdapter.OBJECT_TYPE);
      ia.invokespecial("java/lang/Object", "<init>", "()V", false);
      ia.areturn(Type.VOID_TYPE);
      ia.visitMaxs(1, 1);
      ia.visitEnd();
    }

    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      InstructionAdapter ia = new InstructionAdapter(mv2);
      ia.visitCode();
      ia.getstatic("java/lang/System", "out", "Ljava/io/PrintStream;");
      ia.aconst("Hello World");
      ia.invokevirtual("java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      ia.areturn(Type.VOID_TYPE);
      ia.visitMaxs(2, 1);
      ia.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

### 4.9.3 总结

本文对`InstructionAdapter`类进行了介绍，内容总结如下：

- 第一点，`InstructionAdapter`类的特点就是引入了一些与opcode有关的新方法，这些新方法本质上还是调用`MethodVisitor.visitXxxInsn()`来实现的。
- 第二点，如果已经熟悉`MethodVisitor`类的使用，可以完全不考虑使用`InstructionAdapter`类。

## 4.10 ClassRemapper介绍

`ClassRemapper`类的特点是，可以实现从“一个类”向“另一个类”的映射。借助于这个类，我们可以将class文件进行简单的混淆处理（obfuscate）：

![Java ASM系列：（042）Cla***emapper介绍_Core API](https://s2.51cto.com/images/20210707/1625603371740043.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 4.10.1 ClassRemapper类

#### class info

第一个部分，`ClassRemapper`类继承自`ClassVisitor`类。

```java
/**
 * A {@link ClassVisitor} that remaps types with a {@link Remapper}.
 *
 * <p><i>This visitor has several limitations</i>. A non-exhaustive list is the following:
 *
 * <ul>
 *   <li>it cannot remap type names in dynamically computed strings (remapping of type names in
 *       static values is supported).
 *   <li>it cannot remap values derived from type names at compile time, such as
 *       <ul>
 *         <li>type name hashcodes used by some Java compilers to implement the string switch
 *             statement.
 *         <li>some compound strings used by some Java compilers to implement lambda
 *             deserialization.
 *       </ul>
 * </ul>
 *
 * @author Eugene Kuleshov
 */
public class ClassRemapper extends ClassVisitor {
}
```

#### fields

第二个部分，`ClassRemapper`类定义的字段有哪些。在`ClassRemapper`类当中，定义了两个字段：`remapper`字段和`className`字段。

- `remapper`字段是实现从“一个类”向“另一个类”映射的关键部分；它的类型是`Remapper`类。
- `className`字段则表示当前类的名字。

```java
public class ClassRemapper extends ClassVisitor {
  /** The remapper used to remap the types in the visited class. */
  protected final Remapper remapper;
  /** The internal name of the visited class. */
  protected String className;
}
```

`Remapper`类是一个抽象类，它有一个具体的子类`SimpleRemapper`类；这个`SimpleRemapper`类从本质上来说是一个`Map`，在实现上比较简单。

```java
public abstract class Remapper {
}

public class SimpleRemapper extends Remapper {
  private final Map<String, String> mapping;
  // 这个方法上有很详细的注释，可以自己到源码上看，上面说明了该按照什么规则填写该 映射关系 Map
  public SimpleRemapper(final Map<String, String> mapping) {
    this.mapping = mapping;
  }

  public SimpleRemapper(final String oldName, final String newName) {
    this.mapping = Collections.singletonMap(oldName, newName);
  }    
}
```

#### constructors

第三个部分，`ClassRemapper`类定义的构造方法有哪些。

```java
public class ClassRemapper extends ClassVisitor {
  public ClassRemapper(final ClassVisitor classVisitor, final Remapper remapper) {
    this(Opcodes.ASM9, classVisitor, remapper);
  }

  protected ClassRemapper(final int api, final ClassVisitor classVisitor, final Remapper remapper) {
    super(api, classVisitor);
    this.remapper = remapper;
  }
}
```

#### methods

第四个部分，`ClassRemapper`类定义的方法有哪些。

*可以看到在原本的visitXxx()方法调用中，都增加了映射替换的逻辑。*

```java
public class ClassRemapper extends ClassVisitor {
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    this.className = name;
    super.visit(
      version,
      access,
      remapper.mapType(name),
      remapper.mapSignature(signature, false),
      remapper.mapType(superName),
      interfaces == null ? null : remapper.mapTypes(interfaces));
  }

  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    FieldVisitor fieldVisitor =
      super.visitField(
      access,
      remapper.mapFieldName(className, name, descriptor),
      remapper.mapDesc(descriptor),
      remapper.mapSignature(signature, true),
      (value == null) ? null : remapper.mapValue(value));
    return fieldVisitor == null ? null : createFieldRemapper(fieldVisitor);
  }

  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    String remappedDescriptor = remapper.mapMethodDesc(descriptor);
    MethodVisitor methodVisitor =
      super.visitMethod(
      access,
      remapper.mapMethodName(className, name, descriptor),
      remappedDescriptor,
      remapper.mapSignature(signature, false),
      exceptions == null ? null : remapper.mapTypes(exceptions));
    return methodVisitor == null ? null : createMethodRemapper(methodVisitor);
  }
}
```

### 4.10.2 示例

#### 示例一：修改类名

+ 预期目标

修改前：

```java
public class HelloWorld {
  public void test() {
    System.out.println("Hello World");
  }
}
```

修改后：

```java
public class GoodChild {
  public void test() {
    System.out.println("Hello World");
  }
}
```

---

+ 编码实现

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.commons.ClassRemapper;
import org.objectweb.asm.commons.Remapper;
import org.objectweb.asm.commons.SimpleRemapper;

public class ClassRemapperExample01 {
  public static void main(String[] args) {
    String origin_name = "sample/HelloWorld";
    String target_name = "sample/GoodChild";
    String origin_filepath = getFilePath(origin_name);
    byte[] bytes1 = FileUtils.readBytes(origin_filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    Remapper remapper = new SimpleRemapper(origin_name, target_name);
    ClassVisitor cv = new ClassRemapper(cw, remapper);

    //（4）两者进行结合
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）重新生成Class
    byte[] bytes2 = cw.toByteArray();

    String target_filepath = getFilePath(target_name);
    FileUtils.writeBytes(target_filepath, bytes2);
  }

  public static String getFilePath(String internalName) {
    String relative_path = String.format("%s.class", internalName);
    return FileUtils.getFilePath(relative_path);
  }
}
```

---

+ 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.GoodChild");
    Method m = clazz.getDeclaredMethod("test");

    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

#### 示例二：修改字段名和方法名

+ 预期目标

修改前：

```java
public class HelloWorld {
  private int intValue;

  public HelloWorld() {
    this.intValue = 100;
  }

  public void test() {
    System.out.println("field value: " + intValue);
  }
}
```

修改后：

```java
public class GoodChild {
  private int a;

  public GoodChild() {
    this.a = 100;
  }

  public void b() {
    System.out.println("field value: " + this.a);
  }
}
```

---

+ 编码实现

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.commons.ClassRemapper;
import org.objectweb.asm.commons.Remapper;
import org.objectweb.asm.commons.SimpleRemapper;

import java.util.HashMap;
import java.util.Map;

public class ClassRemapperExample02 {
  public static void main(String[] args) {
    String origin_name = "sample/HelloWorld";
    String target_name = "sample/GoodChild";
    String origin_filepath = getFilePath(origin_name);
    byte[] bytes1 = FileUtils.readBytes(origin_filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    Map<String, String> mapping = new HashMap<>();
    mapping.put(origin_name, target_name);
    mapping.put(origin_name + ".intValue", "a");
    mapping.put(origin_name + ".test()V", "b");
    Remapper mapper = new SimpleRemapper(mapping);
    ClassVisitor cv = new ClassRemapper(cw, mapper);

    //（4）两者进行结合
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）重新生成Class
    byte[] bytes2 = cw.toByteArray();

    String target_filepath = getFilePath(target_name);
    FileUtils.writeBytes(target_filepath, bytes2);
  }

  public static String getFilePath(String internalName) {
    String relative_path = String.format("%s.class", internalName);
    return FileUtils.getFilePath(relative_path);
  }
}
```

---

+ 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.GoodChild");
    Method m = clazz.getDeclaredMethod("b");

    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

#### 示例三：修改两个类

+ 预期目标

修改前：

```java
public class GoodChild {
  public void study() {
    System.out.println("Start where you are. Use what you have. Do what you can. – Arthur Ashe");
  }
}
```

修改后：

```java
public class BBB {
  public void b() {
    System.out.println("Start where you are. Use what you have. Do what you can. – Arthur Ashe");
  }
}
```

修改前：

```java
public class HelloWorld {
  public void test() {
    GoodChild child = new GoodChild();
    child.study();
  }
}
```

修改后：

```java
public class AAA {
  public void a() {
    BBB var1 = new BBB();
    var1.b();
  }
}
```

---

+ 编码实现

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.commons.ClassRemapper;
import org.objectweb.asm.commons.Remapper;
import org.objectweb.asm.commons.SimpleRemapper;

import java.util.HashMap;
import java.util.Map;

public class ClassRemapperExample03 {
  public static void main(String[] args) {
    Map<String, String> mapping = new HashMap<>();
    mapping.put("sample/HelloWorld", "sample/AAA");
    mapping.put("sample/GoodChild", "sample/BBB");
    mapping.put("sample/HelloWorld.test()V", "a");
    mapping.put("sample/GoodChild.study()V", "b");
    obfuscate("sample/HelloWorld", "sample/AAA", mapping);
    obfuscate("sample/GoodChild", "sample/BBB", mapping);
  }

  public static void obfuscate(String origin_name, String target_name, Map<String, String> mapping) {
    String origin_filepath = getFilePath(origin_name);
    byte[] bytes1 = FileUtils.readBytes(origin_filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    Remapper mapper = new SimpleRemapper(mapping);
    ClassVisitor cv = new ClassRemapper(cw, mapper);

    //（4）两者进行结合
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）重新生成Class
    byte[] bytes2 = cw.toByteArray();

    String target_filepath = getFilePath(target_name);
    FileUtils.writeBytes(target_filepath, bytes2);
  }

  public static String getFilePath(String internalName) {
    String relative_path = String.format("%s.class", internalName);
    return FileUtils.getFilePath(relative_path);
  }
}
```

---

+ 验证结果

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.AAA");
    Method m = clazz.getDeclaredMethod("a");

    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

### 4.10.3 总结

本文对`ClassRemapper`类进行介绍，内容总结如下：

- 第一点，`ClassRemapper`类的特点是，可以实现从“一个类”向“另一个类”的映射。
- 第二点，了解`ClassRemapper`类各个部分的信息。使用`ClassRemapper`类时，关键的地方就是创建一个`Remapper`对象。`Remapper`类，是一个抽象类，它有一个具体实现`SimpleRemapper`类，它记录的是从“一个类”向“另一个类”的映射关系。其实，我们也可以提供一个自己的`Remapper`类的子类，来完成一些特殊的转换规则。
- 第三点，<u>从功能（强弱）的角度来说，`ClassRemapper`类可以实现对class文件进行简单的混淆处理。但是，`ClassRemapper`类进行混淆处理的程度是比较低的，远不及一些专业的代码混淆工具（obfuscator）</u>。

## 4.11 StaticInitMerger介绍

`StaticInitMerger`类的特点是，可以实现将多个`<clinit>()`方法合并到一起。

### 4.11.1 如何合并两个类文件

**首先，什么是合并两个类文件？** 假如有两个类，一个是`sample.HelloWorld`类，另一个是`sample.GoodChild`类，我们想将`sample.GoodChild`类里面定义的字段（fields）和方法（methods）全部放到`sample.HelloWorld`类里面，这样就是将两个类合并成一个新的`sample.HelloWorld`类。

**其次，合并两个类文件，有哪些应用场景呢？** 假如`sample/HelloWorld.class`是来自于第三方的软件产品，但是，我们可能会发现它的功能有些不足，所以想对这个类进行扩展。

- 第一种情况，如果扩展的功能比较简单，那么可以直接使用`ClassVisitor`和`MethodVisitor`类可以进行Class Transformation操作。
- 第二种情况，如果扩展的功能比较复杂，例如，需要添加的方法比较多、方法实现的代码逻辑比较复杂，那么使用`ClassVisitor`和`MethodVisitor`类直接修改就会变得比较麻烦。这个时候，如果我们把想添加的功能，使用Java语言编写代码，放到一个全新的`sample.GoodChild`类，将其编译成`sample/GoodChild.class`文件；再接下来，只要我们将`sample/GoodChild.class`定义的字段（fields）和方法（methods）全部迁移到`sample/HelloWorld.class`就可以了。

**再者，合并两个类文件，需要注意哪些地方呢？** 因为主要迁移的内容有接口（interface）、字段（fields）和方法（methods），那么就应该避免出现“重复”的接口、字段和方法。

- 第一点，在编写Java代码时，在编写`sample.GoodChild`类时，应该避免定义重复的字段和方法。
- 第二点，在合并两个类时，对于重复的接口信息进行处理。
- 第三点，在合并两个类时，对于重复的`<init>()`方法进行处理。对于`sample/GoodChild.class`里面的`<init>()`方法直接忽略就好了，只保留`sample/HelloWorld.class`定义的`<init>()`方法。
- 第四点，在合并两个类之后，对于重复的`<clinit>()`方法进行处理。对于`sample/HelloWorld.class`定义的`<clinit>()`方法和`sample/GoodChild.class`里面定义的`<clinit>()`方法，则需要合并到一起。**`StaticInitMerger`类的作用，就是将多个`<clinit>()`方法进行合并。**

**最后，合并两个类文件，需要经历哪些步骤呢？** 在这里，我们列出了四个步骤：

- 第一步，读取两个类文件。
- 第二步，将`sample.GoodChild`类重命名为`sample.HelloWorld`。在代码实现上，会用到`ClassRemapper`类。
- 第三步，合并两个类。在这个过程中，要对重复的接口（interface）和`<init>()`方法。在代码实现上，会用到`ClassNode`类（Tree API）。
- 第四步，处理重复的`<clinit>()`方法。在代码实现上，会用到`StaticInitMerger`类。

![Java ASM系列：（043）StaticInitMerger介绍_ASM](https://s2.51cto.com/images/20210707/1625660086263737.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 4.11.2 StaticInitMerger类

#### class info

第一个部分，`StaticInitMerger`类继承自`ClassVisitor`类。

```java
/**
 * A {@link ClassVisitor} that merges &lt;clinit&gt; methods into a single one. All the existing
 * &lt;clinit&gt; methods are renamed, and a new one is created, which calls all the renamed
 * methods.
 *
 * @author Eric Bruneton
 */
public class StaticInitMerger extends ClassVisitor {
}
```

#### fields

第二个部分，`StaticInitMerger`类定义的字段有哪些。

在`StaticInitMerger`类里面，定义了4个字段：

- `owner`字段，表示当前类的名字。
- `renamedClinitMethodPrefix`字段和`numClinitMethods`字段一起来确定方法的新名字。
- `mergedClinitVisitor`字段，负责生成新的`<clinit>()`方法。

```java
public class StaticInitMerger extends ClassVisitor {
  // 当前类的名字
  private String owner;

  // 新方法的名字
  private final String renamedClinitMethodPrefix;
  private int numClinitMethods;

  // 生成新方法的MethodVisitor
  private MethodVisitor mergedClinitVisitor;
}
```

#### constructors

第三个部分，`StaticInitMerger`类定义的构造方法有哪些。

```java
public class StaticInitMerger extends ClassVisitor {
  public StaticInitMerger(final String prefix, final ClassVisitor classVisitor) {
    this(Opcodes.ASM9, prefix, classVisitor);
  }

  protected StaticInitMerger(final int api, final String prefix, final ClassVisitor classVisitor) {
    super(api, classVisitor);
    this.renamedClinitMethodPrefix = prefix;
  }
}
```

#### methods

第四个部分，`StaticInitMerger`类定义的方法有哪些。

在`StaticInitMerger`类里面，定义了3个`visitXxx()`方法：

- `visit()`方法，负责将当前类的名字记录到`owner`字段
- `visitMethod()`方法，负责将原来的`<clinit>()`方法进行重新命名成`renamedClinitMethodPrefix + numClinitMethods`，并在新的`<clinit>()`方法中对`renamedClinitMethodPrefix + numClinitMethods`方法进行调用。
- `visitEnd()`方法，为新的`<clinit>()`方法添加`return`语句。

```java
public class StaticInitMerger extends ClassVisitor {
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    this.owner = name;
  }

  // 注意这里 MethodVisitor没有按照规范在方法开始时调用visitCode()，在方法结束时调用visitEnd()
  // 这里简言之就是 把原本的 <clinit> 重命名，然后重新生成一个 <clinit>方法调用 原本被重命名的<clinit>方法
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor methodVisitor;
    if ("<clinit>".equals(name)) {
      int newAccess = Opcodes.ACC_PRIVATE + Opcodes.ACC_STATIC;
      String newName = renamedClinitMethodPrefix + numClinitMethods++;
      methodVisitor = super.visitMethod(newAccess, newName, descriptor, signature, exceptions);

      if (mergedClinitVisitor == null) {
        mergedClinitVisitor = super.visitMethod(newAccess, name, descriptor, null, null);
      }
      mergedClinitVisitor.visitMethodInsn(Opcodes.INVOKESTATIC, owner, newName, descriptor, false);
    } else {
      methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);
    }
    return methodVisitor;
  }

  public void visitEnd() {
    if (mergedClinitVisitor != null) {
      mergedClinitVisitor.visitInsn(Opcodes.RETURN);
      mergedClinitVisitor.visitMaxs(0, 0);
    }
    super.visitEnd();
  }
}
```

### 4.11.3 示例：合并两个类文件

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  static {
    System.out.println("This is static initialization method");
  }

  private String name;
  private int age;

  public HelloWorld() {
    this("tomcat", 10);
  }

  public HelloWorld(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public void test() {
    System.out.println("This is test method.");
  }

  @Override
  public String toString() {
    return String.format("HelloWorld { name='%s', age=%d }", name, age);
  }
}
```

假如有一个`GoodChild`类，代码如下：

```java
import java.io.Serializable;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;

public class GoodChild implements Serializable {
  private static final DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

  public void printDate() {
    Date now = new Date();
    String str = df.format(now);
    System.out.println(str);
  }
}
```

我们想实现的预期目标：将`HelloWorld`类和`GoodChild`类合并成一个新的`HelloWorld`类。

#### 编码实现

下面的`ClassMergeVisitor`类的作用是负责将两个类合并到一起。我们需要注意以下三点：

- 第一点，`ClassNode`、`FieldNode`和`MethodNode`都是属于ASM的Tree API部分。
- 第二点，将两个类进行合并的代码逻辑，放在了`visitEnd()`方法内。为什么要把代码逻辑放在`visitEnd()`方法内呢？因为参照`ClassVisitor`类里的`visitXxx()`方法调用的顺序，`visitField()`方法和`visitMethod()`方法正好位于`visitEnd()`方法的前面。
- 第三点，在`visitEnd()`方法的代码逻辑中，忽略掉了`<init>()`方法，这样就避免新生成的类当中包含重复的`<init>()`方法。

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.FieldNode;
import org.objectweb.asm.tree.MethodNode;

import java.util.List;

public class ClassMergeVisitor extends ClassVisitor {
  private final ClassNode anotherClass;

  public ClassMergeVisitor(int api, ClassVisitor classVisitor, ClassNode anotherClass) {
    super(api, classVisitor);
    this.anotherClass = anotherClass;
  }

  @Override
  public void visitEnd() {
    List<FieldNode> fields = anotherClass.fields;
    for (FieldNode fn : fields) {
      fn.accept(this);
    }

    List<MethodNode> methods = anotherClass.methods;
    for (MethodNode mn : methods) {
      String methodName = mn.name;
      if ("<init>".equals(methodName)) {
        continue;
      }
      mn.accept(this);
    }
    super.visitEnd();
  }
}
```

下面的`ClassAddInterfaceVisitor`类是负责为类添加“接口信息”。

```java
import org.objectweb.asm.ClassVisitor;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

public class ClassAddInterfaceVisitor extends ClassVisitor {
  private final String[] newInterfaces;

  public ClassAddInterfaceVisitor(int api, ClassVisitor cv, String[] newInterfaces) {
    super(api, cv);
    this.newInterfaces = newInterfaces;
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    Set<String> set = new HashSet<>(); // 注意，这里使用Set是为了避免出现重复接口
    if (interfaces != null) {
      set.addAll(Arrays.asList(interfaces));
    }
    if (newInterfaces != null) {
      set.addAll(Arrays.asList(newInterfaces));
    }
    super.visit(version, access, name, signature, superName, set.toArray(new String[0]));
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.commons.ClassRemapper;
import org.objectweb.asm.commons.Remapper;
import org.objectweb.asm.commons.SimpleRemapper;
import org.objectweb.asm.commons.StaticInitMerger;
import org.objectweb.asm.tree.ClassNode;

import java.util.List;

public class StaticInitMergerExample01 {
  private static final int API_VERSION = Opcodes.ASM9;

  public static void main(String[] args) {
    // 第一步，读取两个类文件
    String first_class = "sample/HelloWorld";
    String second_class = "sample/GoodChild";

    String first_class_filepath = getFilePath(first_class);
    byte[] bytes1 = FileUtils.readBytes(first_class_filepath);

    String second_class_filepath = getFilePath(second_class);
    byte[] bytes2 = FileUtils.readBytes(second_class_filepath);

    // 第二步，将sample/GoodChild类重命名为sample/HelloWorld
    byte[] bytes3 = renameClass(second_class, first_class, bytes2);

    // 第三步，合并两个类
    byte[] bytes4 = mergeClass(bytes1, bytes3);

    // 第四步，处理重复的class initialization method
    byte[] bytes5 = removeDuplicateStaticInitMethod(bytes4);
    FileUtils.writeBytes(first_class_filepath, bytes5);
  }

  public static String getFilePath(String internalName) {
    String relative_path = String.format("%s.class", internalName);
    return FileUtils.getFilePath(relative_path);
  }

  public static byte[] renameClass(String origin_name, String target_name, byte[] bytes) {
    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    Remapper remapper = new SimpleRemapper(origin_name, target_name);
    ClassVisitor cv = new ClassRemapper(cw, remapper);

    //（4）两者进行结合
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）重新生成Class
    return cw.toByteArray();
  }

  public static byte[] mergeClass(byte[] bytes1, byte[] bytes2) {
    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    ClassNode cn = getClassNode(bytes2);
    List<String> interface_list = cn.interfaces;
    int size = interface_list.size();
    String[] interfaces = new String[size];
    for (int i = 0; i < size; i++) {
      String item = interface_list.get(i);
      interfaces[i] = item;
    }
    ClassMergeVisitor cmv = new ClassMergeVisitor(API_VERSION, cw, cn);
    ClassAddInterfaceVisitor cv = new ClassAddInterfaceVisitor(API_VERSION, cmv, interfaces);

    //（4）两者进行结合
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）重新生成Class
    return cw.toByteArray();
  }

  public static ClassNode getClassNode(byte[] bytes) {
    ClassReader cr = new ClassReader(bytes);
    ClassNode cn = new ClassNode();
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);
    return cn;
  }

  public static byte[] removeDuplicateStaticInitMethod(byte[] bytes) {
    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    ClassVisitor cv = new StaticInitMerger("class_init$", cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
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
    System.out.println(instance);
    invokeMethod(clazz, "test", instance);
    invokeMethod(clazz, "printDate", instance);
  }

  public static void invokeMethod(Class<?> clazz, String methodName, Object instance) throws Exception {
    Method m = clazz.getDeclaredMethod(methodName);
    m.invoke(instance);
  }
}
```

### 4.11.4 总结

本文对`StaticInitMerger`类进行了介绍，内容总结如下：

- 第一点，**`StaticInitMerger`类的特点是可以将多个`<clinit>()`方法合并**。
- 第二点，了解`StaticInitMerger`类各个部分的信息，以便理解`StaticInitMerger`类的工作原理。
- 第三点，将两个类文件合并，有什么样的应用场景、需要注意哪些内容、经过哪些步骤，以及如何编码进行实现。

## 4.12 SerialVersionUIDAdder介绍

`SerialVersionUIDAdder`类的特点是可以为Class文件添加一个`serialVersionUID`字段。

### 4.12.1 SerialVersionUIDAdder类

当`SerialVersionUIDAdder`类计算`serialVersionUID`值的时候，它有一套自己的算法，对于算法本身就不进行介绍了。那么，`serialVersionUID`字段有什么作用呢？

Simply put, we use the `serialVersionUID` attribute to **remember versions of a `Serializable` class** to verify that **a loaded class and the serialized object are compatible**.

#### class info

第一个部分，`SerialVersionUIDAdder`类的父类是`ClassVisitor`类。

```java
public class SerialVersionUIDAdder extends ClassVisitor {
}
```

#### fields

第二个部分，`SerialVersionUIDAdder`类定义的字段有哪些。

`SerialVersionUIDAdder`类的字段分成三组：

- 第一组，`computeSvuid`和`hasSvuid`字段，用于判断是否需要添加`serialVersionUID`信息。
- 第二组，`access`、`name`和`interfaces`字段，用于记录当前类的访问标识、类的名字和实现的接口。
- 第三组，`svuidFields`、`svuidMethods`、`svuidConstructors`和`hasStaticInitializer`字段，用于记录字段、构造方法、普通方法、是否有`<clinit>()`方法。

第二组和第三组部分的字段都是用于计算`serialVersionUID`值的，而第一组字段是则先判断是否有必要去计算`serialVersionUID`值。

```java
public class SerialVersionUIDAdder extends ClassVisitor {
  // 第一组，用于判断是否需要添加serialVersionUID信息
  private boolean computeSvuid;
  private boolean hasSvuid;

  // 第二组，用于记录当前类的访问标识、类的名字和实现的接口
  private int access;
  private String name;
  private String[] interfaces;

  // 第三组，用于记录字段、构造方法、普通方法
  private Collection<Item> svuidFields;
  private boolean hasStaticInitializer;
  private Collection<Item> svuidConstructors;
  private Collection<Item> svuidMethods;
}
```

其中，`svuidFields`、`svuidMethods`和`svuidConstructors`字段都涉及到`Item`类。`Item`类的定义如下：

```java
private static final class Item implements Comparable<Item> {
  final String name;
  final int access;
  final String descriptor;

  Item(final String name, final int access, final String descriptor) {
    this.name = name;
    this.access = access;
    this.descriptor = descriptor;
  }
}
```

#### constructors

第三个部分，`SerialVersionUIDAdder`类定义的构造方法有哪些。

```java
public class SerialVersionUIDAdder extends ClassVisitor {
  public SerialVersionUIDAdder(final ClassVisitor classVisitor) {
    this(Opcodes.ASM9, classVisitor);
  }

  protected SerialVersionUIDAdder(final int api, final ClassVisitor classVisitor) {
    super(api, classVisitor);
  }
}
```

#### methods

第四个部分，`SerialVersionUIDAdder`类定义的方法有哪些。

```java
public class SerialVersionUIDAdder extends ClassVisitor {
  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    // Get the class name, access flags, and interfaces information (step 1, 2 and 3) for SVUID computation.
    // 对于枚举类型，不计算serialVersionUID
    computeSvuid = (access & Opcodes.ACC_ENUM) == 0;

    if (computeSvuid) {
      // 记录第二组字段的信息
      this.name = name;
      this.access = access;
      this.interfaces = interfaces.clone();
      this.svuidFields = new ArrayList<>();
      this.svuidConstructors = new ArrayList<>();
      this.svuidMethods = new ArrayList<>();
    }

    super.visit(version, access, name, signature, superName, interfaces);
  }

  @Override
  public FieldVisitor visitField(int access, String name, String desc, String signature, Object value) {
    // Get the class field information for step 4 of the algorithm. Also determine if the class already has a SVUID.
    if (computeSvuid) {
      if ("serialVersionUID".equals(name)) {
        // Since the class already has SVUID, we won't be computing it.
        computeSvuid = false;
        hasSvuid = true;
      }
      // Collect the non private fields. Only the ACC_PUBLIC, ACC_PRIVATE, ACC_PROTECTED,
      // ACC_STATIC, ACC_FINAL, ACC_VOLATILE, and ACC_TRANSIENT flags are used when computing
      // serialVersionUID values.
      if ((access & Opcodes.ACC_PRIVATE) == 0
          || (access & (Opcodes.ACC_STATIC | Opcodes.ACC_TRANSIENT)) == 0) {
        int mods = access
          & (Opcodes.ACC_PUBLIC
             | Opcodes.ACC_PRIVATE
             | Opcodes.ACC_PROTECTED
             | Opcodes.ACC_STATIC
             | Opcodes.ACC_FINAL
             | Opcodes.ACC_VOLATILE
             | Opcodes.ACC_TRANSIENT);
        svuidFields.add(new Item(name, mods, desc));
      }
    }

    return super.visitField(access, name, desc, signature, value);
  }


  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    // Get constructor and method information (step 5 and 7). Also determine if there is a class initializer (step 6).
    if (computeSvuid) {
      // 这里是对静态初始方法进行判断
      if (CLINIT.equals(name)) {
        hasStaticInitializer = true;
      }
      // Collect the non private constructors and methods. Only the ACC_PUBLIC, ACC_PRIVATE,
      // ACC_PROTECTED, ACC_STATIC, ACC_FINAL, ACC_SYNCHRONIZED, ACC_NATIVE, ACC_ABSTRACT and
      // ACC_STRICT flags are used.
      int mods = access
        & (Opcodes.ACC_PUBLIC
           | Opcodes.ACC_PRIVATE
           | Opcodes.ACC_PROTECTED
           | Opcodes.ACC_STATIC
           | Opcodes.ACC_FINAL
           | Opcodes.ACC_SYNCHRONIZED
           | Opcodes.ACC_NATIVE
           | Opcodes.ACC_ABSTRACT
           | Opcodes.ACC_STRICT);

      if ((access & Opcodes.ACC_PRIVATE) == 0) {
        if ("<init>".equals(name)) {
          // 这里是对构造方法进行判断
          svuidConstructors.add(new Item(name, mods, descriptor));
        } else if (!CLINIT.equals(name)) {
          // 这里是对普通方法进行判断
          svuidMethods.add(new Item(name, mods, descriptor));
        }
      }
    }

    return super.visitMethod(access, name, descriptor, signature, exceptions);
  }

  @Override
  public void visitEnd() {
    // Add the SVUID field to the class if it doesn't have one.
    if (computeSvuid && !hasSvuid) {
      try {
        addSVUID(computeSVUID());
      } catch (IOException e) {
        throw new IllegalStateException("Error while computing SVUID for " + name, e);
      }
    }

    super.visitEnd();
  }

  protected void addSVUID(final long svuid) {
    FieldVisitor fieldVisitor =
      super.visitField(Opcodes.ACC_FINAL + Opcodes.ACC_STATIC, "serialVersionUID", "J", null, svuid);
    if (fieldVisitor != null) {
      fieldVisitor.visitEnd();
    }
  }

  protected long computeSVUID() throws IOException {
    // ...
  }
}
```

### 4.12.2 示例

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
import java.io.Serializable;

public class HelloWorld implements Serializable {
  public String name;
  public int age;

  public HelloWorld(String name, int age) {
    this.name = name;
    this.age = age;
  }

  @Override
  public String toString() {
    return String.format("HelloWorld { name='%s', age=%d }", name, age);
  }
}
```

我们想实现的预期目标：为`HelloWorld`类添加`serialVersionUID`字段。

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.commons.SerialVersionUIDAdder;

public class SerialVersionUIDAdderExample01 {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new SerialVersionUIDAdder(cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
import lsieun.utils.FileUtils;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class HelloWorldRun {
  private static final String FILE_DATA = "obj.data";

  public static void main(String[] args) throws Exception {
    HelloWorld obj = new HelloWorld("Tomcat", 10);
    writeObject(obj);
    readObject();
  }

  public static void readObject() throws Exception {
    String filepath = FileUtils.getFilePath(FILE_DATA);
    byte[] bytes = FileUtils.readBytes(filepath);
    ByteArrayInputStream bai = new ByteArrayInputStream(bytes);
    ObjectInputStream in = new ObjectInputStream(bai);
    Object obj = in.readObject();
    System.out.println(obj);
  }

  public static void writeObject(Object obj) throws Exception {
    ByteArrayOutputStream bao = new ByteArrayOutputStream();
    ObjectOutputStream out = new ObjectOutputStream(bao);
    out.writeObject(obj);
    out.flush();
    out.close();
    byte[] bytes = bao.toByteArray();

    String filepath = FileUtils.getFilePath(FILE_DATA);
    FileUtils.writeBytes(filepath, bytes);
  }
}
```

### 4.12.3 总结

本文对`SerialVersionUIDAdder`类进行介绍，内容总结如下：

- 第一点，`SerialVersionUIDAdder`类的特点是可以为Class文件添加一个`serialVersionUID`字段。
- 第二点，了解`SerialVersionUIDAdder`类的各个部分的信息，以便理解它的工作原理。
- 第三点，如何使用`SerialVersionUIDAdder`类添加`serialVersionUID`字段。
