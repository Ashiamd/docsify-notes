# Java-ASM-学习笔记03

> 视频：[Java ASM系列：（201）课程介绍-lsieun-lsieun-哔哩哔哩视频 (bilibili.com)](https://www.bilibili.com/list/1321054247?bvid=BV1Dq4y1P78G&oid=548331545)
> 视频对应的gitee：[learn-java-asm: 学习Java ASM (gitee.com)](https://gitee.com/lsieun/learn-java-asm)
> 视频对应的文章：[Java ASM系列三：Tree API | lsieun](https://lsieun.github.io/java/asm/java-asm-season-03.html) <= 笔记内容大部分都摘自该文章，下面不再重复声明
>
> ASM官网：[ASM (ow2.io)](https://asm.ow2.io/)
> ASM中文手册：[ASM4 使用手册(中文版) (yuque.com)](https://www.yuque.com/mikaelzero/asm)
>
> Java语言规范和JVM规范-官方文档：[Java SE Specifications (oracle.com)](https://docs.oracle.com/javase/specs/index.html)

# 1. 基础

## 1.1 Tree API介绍

> 其实学过Core API后，Tree API整体理解起来还是比较简单的，主要就是大致熟悉下Tree API的使用。毕竟Tree API就是在Core API的基础上衍生出来的产物。

### 1.1.1 ASM的两个组成部分

从组成结构上来说，ASM分成两部分，一部分为Core API，另一部分为Tree API。

- 其中，Core API包括`asm.jar`、`asm-util.jar`和`asm-commons.jar`；
- 其中，Tree API包括`asm-tree.jar`和`asm-analysis.jar`。

![Java ASM系列：（068）Tree API介绍_Bytecode](https://s2.51cto.com/images/20210618/1624028194528190.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

从两者的关系来说，Core API是基础，而Tree API是在Core API的这个基础上构建起来的。

### 1.1.2 Tree API概览

#### asm-tree.jar

在`asm-tree.jar`(9.0版本)当中，一共包含了36个类，我们会涉及到其中的20个类，这20个类就构成`asm-tree.jar`主体部分。

在`asm-tree.jar`当中，这20个类具体内容如下：

- ClassNode
- FieldNode
- MethodNode
- InsnList
- AbstractInsnNode
  - FieldInsnNode
  - IincInsnNode
  - InsnNode
  - IntInsnNode
  - InvokeDynamicInsnNode
  - JumpInsnNode
  - LabelNode
  - LdcInsnNode
  - LookupSwitchInsnNode
  - MethodInsnNode
  - MultiANewArrayInsnNode
  - TableSwitchInsnNode
  - TypeInsnNode
  - VarInsnNode
- TryCatchBlockNode

----

为了方便于理解，我们可以将这些类按“包含”关系组织起来：

- ClassNode（类）
  - FieldNode（字段）
  - MethodNode（方法）
    - InsnList（有序的指令集合）
      - AbstractInsnNode（单条指令）
    - TryCatchBlockNode（异常处理）

我们可以用文字来描述它们之间的关系：

- 第一点，类（`ClassNode`）包含字段（`FieldNode`）和方法（`MethodNode`）。
- 第二点，方法（`MethodNode`）包含有序的指令集合（`InsnList`）和异常处理（`TryCatchBlockNode`）。
- 第三点，有序的指令集合（`InsnList`）由多个单条指令（`AbstractInsnNode`）组合而成。

#### asm-analysis.jar

在`asm-analysis.jar`(9.0版本)当中，一共包含了13个类，我们会涉及到其中的10个类：

- Analyzer
- BasicInterpreter
- BasicValue
- BasicVerifier
- Frame
- Interpreter
- SimpleVerifier
- SourceInterpreter
- SourceValue
- Value

同样，为了方便于理解，我们也可以将这些类按“包含”关系组织起来：

- Analyzer
  - Frame
  - Interpreter + Value
    - BasicInterpreter + BasicValue
    - BasicVerifier + BasicValue
    - SimpleVerifier + BasicValue
    - SourceInterpreter + SourceValue

接着，我们用文字来描述它们之间的关系：

- 第一点，`Analyzer`是一个“胶水”，它起到一个粘合的作用，它就是将`Frame`（不变的部分）和`Interpreter`（变化的部分）组织到了一起。`Frame`类表示的就是一种不变的规则，就类似于现实世界的物理规则；而`Interpreter`类则是在这个规则基础上衍生的变化，就比如说能否造出一个圆珠笔，能否造出一个火箭飞上天空。
- 第二点，`Frame`类并不是绝对的“不变”，它本身也包含“不变”和“变化”的部分。其中，“不变”的部分就是指`Frame.execute()`方法，它就是模拟opcode在执行过程中，对于local variable和operand stack的影响，这是[JVM文档](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)所规定的内容；而“变化”的部分就是指`Frame`类的`V[] values`字段，它需要记录每一个opcode执行前、后的具体值是什么。
- 第三点，`Interpreter`类就是体现“个人创造力”的一个类。比如说，同样是受这个现实物理世界规则的约束，在不同的历史阶段过程中，有人发明了风筝（纸鸢），有人发明了烟花，有人发明了滑翔机，有人发明了喷气式飞机，有人发明了宇宙飞船。同样，我们可以实现具体的`Interpreter`子类来完成特定的功能，那么到底能够达到一种什么样的效果，还是会取决于我们自身的“创造力”。

> *另外，我们也不会涉及`Subroutine`类。因为`Subroutine`类对应于`jsr`这条指令，而`jsr`指令是处于“以前确实有用，现在已经过时”的状态。*

## 1.2 Core API VS. Tree API

### 1.2.1 ASM能够做什么

ASM is an all-purpose(多用途的；通用的) Java ByteCode **manipulation** and **analysis** framework. It can be used to modify existing classes or to dynamically generate classes, directly in binary form.

The goal of the ASM library is to **generate**, **transform** and **analyze** compiled Java classes, represented as byte arrays (as they are stored on disk and loaded in the Java Virtual Machine).

![Java ASM系列：（069）Core API VS. Tree API_ClassFile](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

- **Program analysis**, which can range from a simple syntactic parsing to a full semantic analysis, can be used to find potential bugs in applications, to detect unused code, to reverse engineer code, etc.
- **Program generation** is used in compilers. This includes traditional compilers, but also stub or skeleton compilers used for distributed programming, Just in Time compilers, etc.
- **Program transformation** can be used to optimize or obfuscate programs, to insert debugging or performance monitoring code into applications, for aspect oriented programming, etc.

总结：**无论是Core API，还是Tree API，都可以用来进行Class Generation、Class Transformation和Class Analysis操作**。

### 1.2.2 Core API和Tree API的区别

#### 整体认识

首先，我们借用电影的片段来解释Core API和Tree API之间的区别，让大家有一个**整体的、模糊的认识**。

电影《龙门飞甲》雨化田介绍“东厂”和“西厂”的概念：

```
一句话，东厂管得了的我要管，东厂管不了的我更要管，先斩后奏，皇权特许，这就是西厂，够不够清楚。
```

我们镜像上面这句话，说明一下“Core API”和“Tree API”的区别：

```
一句话，Core API做得了的我要做，Core API做不了的我更要做，简单易用，功能更强，这就是Tree API，够不够清楚。
```

总结一下：

- Tree API的优势:
  - 易用性：如果一个人在之前并没有接触过Core API和Tree API，那么Tree API更容易入手。
  - 功能性：在实现比较复杂的功能时，Tree API比Core API更容易实现。
- Core API的优势：
  - 执行效率：**在实现相同功能的前提下，Core API要比Tree API执行效率高，花费时间少**。
  - 内存使用：**Core API比Tree API占用的内存空间少**。

ASM provides both APIs because **there is no best API**. Indeed **each API has its own advantages and drawbacks**:

- The Core API is faster and requires less memory than the Tree API, since there is no need to create and store in memory a tree of objects representing the class.
- However implementing class transformations can be more diffcult with the Core API, since only one element of the class is available at any given time (the element that corresponds to the current event), while the whole class is available in memory with the Tree API.

再接下来，我们就从**技术的细节角度**来看Core API和Tree API之间的区别。

#### Class Generation

假如我们想生成下面一个`HelloWorld`类：

```java
public interface HelloWorld extends Cloneable {
  int LESS = -1;
  int EQUAL = 0;
  int GREATER = 1;
  int compareTo(Object o);
}
```

接下来，我们分别使用Core API和Tree API来生成这个类。

如果使用Core API来进行生成，在代码中调用多个`visitXxx()`方法，相应的代码如下：

```java
public class HelloWorldGenerateCore {
  public static byte[] dump() throws Exception {
    // (1) 创建ClassWriter对象
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (2) 调用visitXxx()方法
    cw.visit(V1_8, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE, "sample/HelloWorld",
             null, "java/lang/Object", new String[]{"java/lang/Cloneable"});

    {
      FieldVisitor fv1 = cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS", "I", null, -1);
      fv1.visitEnd();
    }

    {
      FieldVisitor fv2 = cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL", "I", null, 0);
      fv2.visitEnd();
    }

    {
      FieldVisitor fv3 = cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER", "I", null, 1);
      fv3.visitEnd();
    }

    {
      MethodVisitor mv1 = cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "compareTo", "(Ljava/lang/Object;)I", null, null);
      mv1.visitEnd();
    }


    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

如果使用Tree API来进行生成，其特点是代码实现过程就是为`ClassNode`类的字段进行赋值的过程，相应的代码如下：

```java
public class HelloWorldGenerateTree {
  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";
    cn.interfaces.add("java/lang/Cloneable");

    {
      FieldNode fieldNode = new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "LESS", "I", null, new Integer(-1));
      cn.fields.add(fieldNode);
    }

    {
      FieldNode fieldNode = new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "EQUAL", "I", null, new Integer(0));
      cn.fields.add(fieldNode);
    }

    {
      FieldNode fieldNode = new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "GREATER", "I", null, new Integer(1));
      cn.fields.add(fieldNode);
    }

    {
      MethodNode methodNode = new MethodNode(ACC_PUBLIC | ACC_ABSTRACT, "compareTo", "(Ljava/lang/Object;)I", null, null);
      cn.methods.add(methodNode);
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
    return cw.toByteArray();
  }
}
```

虽然Core API和Tree API在代码的表现形式上有差异，前者是调用`visitXxx()`方法，后者是给`ClassNode`定义的字段赋值，但是它们所需要的信息是一致的，都需要提供类名、父类名、接口名、字段、方法等信息。

Generating a class with the tree API simply consists in creating a `ClassNode` object and in initializing its fields.

**Using the tree API to generate a class takes about 30% more time and consumes more memory than using the core API**. But **it makes it possible to generate the class elements in any order**, which can be convenient in some cases.

> 尽管Tree API平均下来需要比Core API多消耗30%时间，以及占用更多内存。但是Tree API使用时可以以任何顺序添加元素，而不需要像Core API一样遵循一定的方法调用顺序

#### Class Transformation

Like for **class generation**, using the tree API to transform classes **takes more time and consumes more memory** than using the core API. But it makes it possible to implement some transformations more easily.

现在，我们想给下面的`HelloWorld`类添加一个digital signature信息（这是一个Class Transformation的过程）：

```java
public class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello ASM");
  }

  public void test() {
    System.out.println("test");
  }
}
```

这里面涉及到两个问题：

- 第一个问题，如何计算digital signature的信息呢？ 我们用一个简单的方法，就是将`class name + field name + method name`拼接成一个字符串，然后求这个字符中的hash code，那么这个hash code就作为digital signature。大家可以根据自己的需求，来换成一个更复杂的方法。
- 第二个问题，如何将计算出的digital signature添加到`HelloWorld`类里面去？ 我们通过一个自定义的Attribute来添加。

其实，这个例子来自于`asm4-guide.pdf`文件，我们稍稍做了一些修改：原文是添加一个Annotation，现在是添加一个Attribute。

This is the case, for example, of **a transformation that adds to a class an annotation containing a digital signature of its content**. With the core API the digital signature can be computed only when all the class has been visited, but then it is too late to add an annotation containing it, because annotations must be visited before class members. With the tree API this problem disappears because there is no such constraint in this case.

In fact, it is possible to implement the `AddDigitialSignature` example with the core API, but then the class must be transformed in two passes.

- During the first pass the class is visited with a `ClassReader` (and no `ClassWriter`), in order to compute the digital signature based on the class content.
- During the second pass the same `ClassReader` is reused to do a second visit of the class, this time with an `AddAnnotationAdapter` chained to a `ClassWriter`.

那么，我们先来使用Core API来进行实现。

我们添加一个`ClassGetAttributeContentVisitor`类，它用来获取digital signature的内容：

```java
public class ClassGetAttributeContentVisitor extends ClassVisitor {
  private final StringBuilder attr = new StringBuilder();

  public ClassGetAttributeContentVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    attr.append(name);
    super.visit(version, access, name, signature, superName, interfaces);
  }

  @Override
  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    attr.append(name);
    return super.visitField(access, name, descriptor, signature, value);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    attr.append(name);
    return super.visitMethod(access, name, descriptor, signature, exceptions);
  }

  public String getAttributeContent() {
    return attr.toString();
  }
}
```

接着，我们添加一个`ClassAddCustomAttributeVisitor`类，它用来添加自定义的Attribute：

```java
public class ClassAddCustomAttributeVisitor extends ClassVisitor {
  private final String attrName;
  private final String attrContent;
  private boolean isAttrPresent;

  public ClassAddCustomAttributeVisitor(int api, ClassVisitor classVisitor, String attrName, String attrContent) {
    super(api, classVisitor);
    this.attrName = attrName;
    this.attrContent = attrContent;
    this.isAttrPresent = false;
  }

  @Override
  public void visitAttribute(Attribute attribute) {
    if (attribute.type.equals(attrName)) {
      isAttrPresent = true;
    }
    super.visitAttribute(attribute);
  }

  @Override
  public void visitNestMember(String nestMember) {
    addAttribute();
    super.visitNestMember(nestMember);
  }

  @Override
  public void visitInnerClass(String name, String outerName, String innerName, int access) {
    addAttribute();
    super.visitInnerClass(name, outerName, innerName, access);
  }

  @Override
  public RecordComponentVisitor visitRecordComponent(String name, String descriptor, String signature) {
    addAttribute();
    return super.visitRecordComponent(name, descriptor, signature);
  }

  @Override
  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    addAttribute();
    return super.visitField(access, name, descriptor, signature, value);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    addAttribute();
    return super.visitMethod(access, name, descriptor, signature, exceptions);
  }

  @Override
  public void visitEnd() {
    addAttribute();
    super.visitEnd();
  }

  private void addAttribute() {
    if (!isAttrPresent) {
      int hashCode = attrContent.hashCode();
      byte[] info = ByteUtils.intToByteArray(hashCode);
      Attribute attr = new CustomAttribute(attrName, info);
      super.visitAttribute(attr);
      isAttrPresent = true;
    }
  }
}
```

再接着，我们经过两次处理来完成转换：

```java
public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    int api = Opcodes.ASM9;
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;

    //（3）第一次处理
    ClassGetAttributeContentVisitor cv1 = new ClassGetAttributeContentVisitor(api, null);
    cr.accept(cv1, parsingOptions);
    String attributeContent = cv1.getAttributeContent();

    //（4）第二次处理
    ClassVisitor cv2 = new ClassAddCustomAttributeVisitor(api, cw, "cn.lsieun.MyAttribute", attributeContent);
    cr.accept(cv2, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

再接下来，我们看看如何使用Tree API来进行转换。

```java
public class ClassAddCustomAttributeNode extends ClassNode {
  private final String attrName;

  public ClassAddCustomAttributeNode(int api, ClassVisitor cv, String attrName) {
    super(api);
    this.cv = cv;
    this.attrName = attrName;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    StringBuilder sb = new StringBuilder();
    sb.append(name);
    for (FieldNode fn : fields) {
      sb.append(fn.name);
    }
    for (MethodNode mn : methods) {
      sb.append(mn.name);
    }
    int hashCode = sb.toString().hashCode();
    byte[] info = ByteUtils.intToByteArray(hashCode);
    Attribute customAttribute = new CustomAttribute(attrName, info);
    if (attrs == null) {
      attrs = new ArrayList<>();
    }
    attrs.add(customAttribute);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }
}
```

```java
public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassAddCustomAttributeNode(api, cw, "cn.lsieun.MyAttribute");

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

> 这里`javap -p -v sample.HelloWorld`会提示出现未知字段，但是`java sample.HelloWorld`仍能正常运行`.class`文件
>
> ```shell
> cn.lsieun.MyAttribute: length = 0x8 (unknown attribute)
>    C0 DE B1 0B 00 5A 4C 70
> ```

By generalizing this argument we see that, in fact, **any transformation can be implemented with the core API alone, by using several passes if necessary.** But this increases the transformation code complexity, this requires to store state between passes (which can be as complex as a full tree representation!), and parsing the class several times has a cost, which must be compared to the cost of constructing the corresponding `ClassNode`.

The conclusion is that **the tree API is generally used for transformations that cannot be implemented in one pass with the core API.** But there are of course exceptions. For example an obfuscator cannot be implemented in one pass, because you cannot transform classes before the mapping from original to obfuscated names is fully constructed, which requires to parse all classes. But the tree API is not a good solution either, because it would require keeping in memory the object representation of all the classes to obfuscate. In this case it is better to use the core API with two passes: one to compute the mapping between original and obfuscated names (a simple hash table that requires much less memory than a full object representation of all the classes), and one to transform the classes based on this mapping.

> Tree API通常用在Core API需要多次转换的场景（即若用Core API需要串联多个ClassVisitor才能实现的复杂功能，用Tree API通常一次或少次转换即可实现）。
>
> 例外就是**代码混淆**的实现，因为你在混淆前后的名称完成映射前，不能通过仅一次实现就完成类转换。且这种场景下Tree API不是好选择，因为它需要在内存中保存类的所有信息。使用Core API完成代码混淆更好，第一次ClassVisitor计算混淆前后的名称映射，第二次ClassVisitor根据映射完成类的转换。

### 1.2.3 总结

本文内容总结如下：

- 第一点，在ASM当中，不管是Core API，还是Tree API，都能够进行Class Generation、Class Transformation和Class Analysis操作。
- 第二点，Core API和Tree API是两者有各自的优势。Tree API易于使用、更容易实现复杂的操作；Core API执行速度更快、占用内存空间更少。

> Tree API 需构造一个树形结构存储当前访问的class的信息；而Core API每次只访问class信息的其中一部分

## 1.3 如何编写ASM代码

在ASM当中，有一个`ASMifier`类，它可以打印出Core API对应的代码；但是，ASM并没有提供打印Tree API对应代码的类，因此我们就写了一个类来实现该功能。

### 1.3.1 PrintASMCodeTree类

我们可以从两个方面来理解`PrintASMCodeTree`类：

- 从功能上来说，`PrintASMCodeTree`类就是用来打印生成类的Tree API代码。
- 从实现上来说，`PrintASMCodeTree`类是通过调用`TreePrinter`类来实现的。

```java
public class PrintASMCodeTree {
  public static void main(String[] args) throws IOException {
    // (1) 设置参数
    String className = "sample.HelloWorld";
    int parsingOptions = ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG;

    // (2) 打印结果
    Printer printer = new TreePrinter();
    PrintWriter printWriter = new PrintWriter(System.out, true);
    TraceClassVisitor traceClassVisitor = new TraceClassVisitor(null, printer, printWriter);
    new ClassReader(className).accept(traceClassVisitor, parsingOptions);
  }
}
```

首先，我们来看一下`PrintASMCodeTree`类的功能。假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    if (val == 0) {
      System.out.println("val is 0");
    }
    else {
      System.out.println("val is not 0");
    }
  }
}
```

接着，我们来看一下`TreePrinter`类，这个类是[项目](https://gitee.com/lsieun/learn-java-asm)当中提供的一个类，继承自`org.objectweb.asm.util.Printer`类。`TreePrinter`类，实现比较简单，功能也非常有限，还可能存在问题（bug）。因此，对于这个类，我们可以使用它，但也应该保持一份警惕。也就是说，使用这个类生成代码之后，我们应该检查一下生成的代码是否正确。

另外，还要注意区分下面五个类的作用：

- `ASMPrint`类：生成类的Core API代码或类的内容（功能二合一）
- `PrintASMCodeCore`类：生成类的Core API代码（由`ASMPrint`类拆分得到，功能单一）
- `PrintASMCodeTree`类：生成类的Tree API代码
- `PrintASMTextClass`类：查看类的内容（由`ASMPrint`类拆分得到，功能单一）
- `PrintASMTextLambda`类：查看Lambda表达式生成的匿名类的内容

这五个类的共同点就是都使用到了`org.objectweb.asm.util.Printer`类的子类。

### 1.3.2 ControlFlowGraphRun类

除了打印ASM Tree API的代码，我们也提供一个`ControlFlowGraphRun`类，可以查看方法的控制流程图：

```java
public class ControlFlowGraphRun {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    ClassNode cn = new ClassNode();

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）查找方法
    String methodName = "test";
    MethodNode targetNode = null;
    for (MethodNode mn : cn.methods) {
      if (mn.name.equals(methodName)) {
        targetNode = mn;
        break;
      }
    }
    if (targetNode == null) {
      System.out.println("Can not find method: " + methodName);
      return;
    }

    //（4）进行图形化显示
    System.out.println("Origin:");
    display(cn.name, targetNode, ControlFlowGraphType.NONE);
    System.out.println("Control Flow Graph:");
    display(cn.name, targetNode, ControlFlowGraphType.STANDARD);

    //（5）打印复杂度
    int complexity = CyclomaticComplexity.getCyclomaticComplexity(cn.name, targetNode);
    String line = String.format("%s:%s complexity: %d", targetNode.name, targetNode.desc, complexity);
    System.out.println(line);
  }
}
```

对于上面的`HelloWorld`类，可以使用`javap`命令查看`test`方法包含的instructions内容：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: iload_1
       1: ifne          15
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String val is 0
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: goto          23
      15: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #5                  // String val is not 0
      20: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: return
}
```

运行`ControlFlowGraphRun`类之后，文字输出程序的复杂度：

```pseudocode
test:(I)V complexity: 2
```

同时，也会有文本图形显示，将instructions内容分成不同的子部分，并显示出子部分之间的跳转关系：

![Java ASM系列：（070）如何编写ASM代码_Bytecode](https://s2.51cto.com/images/20211004/1633322979869068.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

另外，图形部分的代码位于`lsieun.graphics`包内，这些代码是从[ Simple Java Graphics](https://horstmann.com/sjsu/graphics/)网站复制而来的。

### 1.3.2 总结

- 第一点，通过运行`PrintASMCodeTree`类，可以生成类的Tree API代码。
- 第二点，通过运行`ControlFlowGraphRun`类，可以打印方法的复杂度，也可以显示控制流程图。

# 2. Class Generation

## 2.1 ClassNode介绍

The ASM tree API for **generating and transforming compiled Java classes** is based on the `ClassNode` class.

### 2.1.1 ClassNode

#### class info

第一个部分，`ClassNode`类继承自`ClassVisitor`类。

```java
/**
 * A node that represents a class.
 *
 * @author Eric Bruneton
 */
public class ClassNode extends ClassVisitor {
}
```

#### fields

第二个部分，`ClassNode`类定义的字段有哪些。

```java
public class ClassNode extends ClassVisitor {
  public int version;
  public int access;
  public String name;
  public String signature; // 泛型，ClassFile attributes中的一部分
  public String superName;
  public List<String> interfaces;

  public List<FieldNode> fields;
  public List<MethodNode> methods;
}
```

这些字段与`ClassFile`结构中定义的条目有对应关系：

```pseudocode
ClassFile {
    u4             magic; // CAFEBABE
    u2             minor_version; // version
    u2             major_version; // version
    u2             constant_pool_count; // 常量池
    cp_info        constant_pool[constant_pool_count-1]; // 常量池
    u2             access_flags; // access
    u2             this_class; // name
    u2             super_class; // superName
    u2             interfaces_count; // interfaces
    u2             interfaces[interfaces_count]; // interfaces
    u2             fields_count; // fields
    field_info     fields[fields_count]; // fields
    u2             methods_count; // methods
    method_info    methods[methods_count]; // methods
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### constructors

第三个部分，`ClassNode`类定义的构造方法有哪些。

```java
public class ClassNode extends ClassVisitor {
  public ClassNode() {
    this(Opcodes.ASM9);
    if (getClass() != ClassNode.class) {
      throw new IllegalStateException();
    }
  }

  public ClassNode(final int api) {
    super(api);
    this.interfaces = new ArrayList<>();
    this.fields = new ArrayList<>();
    this.methods = new ArrayList<>();
  }
}
```

这两个构造方法的区别：

- **`ClassNode()`主要用于Class Generation**。这个构造方法不适用于子类构造方法中调用，因为它对`getClass() != ClassNode.class`进行了判断；如果是子类调用这个构造方法，就一定会抛出`IllegalStateException`。
- **`ClassNode(int)`主要用于Class Transformation**。这个构造方法适用于子类构造方法中调用。

#### methods

第四个部分，`ClassNode`类定义的方法有哪些。

##### visitXxx方法

**在`ClassNode`类当中，`visitXxx()`方法的目的是将方法的参数转换成`ClassNode`类中字段的值**。

```java
public class ClassNode extends ClassVisitor {
  @Override
  public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
    this.version = version;
    this.access = access;
    this.name = name;
    this.signature = signature;
    this.superName = superName;
    this.interfaces = Util.asArrayList(interfaces);
  }

  @Override
  public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
    FieldNode field = new FieldNode(access, name, descriptor, signature, value);
    fields.add(field);
    return field;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodNode method = new MethodNode(access, name, descriptor, signature, exceptions);
    methods.add(method);
    return method;
  }

  @Override
  public void visitEnd() {
    // Nothing to do.
  }
}
```

##### accept方法

**在`ClassNode`类当中，`accept()`方法的目的是将`ClassNode`类中字段的值传递给下一个`ClassVisitor`类实例**。

```java
public class ClassNode extends ClassVisitor {
  public void accept(final ClassVisitor classVisitor) {
    // Visit the header.
    String[] interfacesArray = new String[this.interfaces.size()];
    this.interfaces.toArray(interfacesArray);
    classVisitor.visit(version, access, name, signature, superName, interfacesArray);
    // ...
    // Visit the fields.
    for (int i = 0, n = fields.size(); i < n; ++i) {
      fields.get(i).accept(classVisitor);
    }
    // Visit the methods.
    for (int i = 0, n = methods.size(); i < n; ++i) {
      methods.get(i).accept(classVisitor);
    }
    classVisitor.visitEnd();
  }    
}
```

### 2.1.2 如何使用ClassNode

我们知道，`ClassNode`是Java ASM类库当中的一个类；而在一个具体的`.class`文件中，它包含的是字节码数据，可以表现为`byte[]`的形式。那么`byte[]`和`ClassNode`类之间是不是可以相互转换呢？

![Java ASM系列：（071）ClassNode介绍_Bytecode](https://s2.51cto.com/images/20211006/1633495164198644.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

#### 将字节转换成ClassNode

借助于`ClassReader`类，可以将`byte[]`内容转换成`ClassNode`类实例：

```java
int api = Opcodes.ASM9;
ClassNode cn = new ClassNode(api);

byte[] bytes = ...; // from a .class file
ClassReader cr = new ClassReader(bytes);
cr.accept(cn, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
```

这样操作之后，一个具体的`.class`文件中的`byte[]`内容会转换成`ClassNode`类中字段的具体值。

#### 将ClassNode转换成字节

相应的，借助于`ClassWriter`类，可以将`ClassNode`类实例转换成`byte[]`：

```java
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
cn.accept(cw);
byte[] bytes = cw.toByteArray();
```

#### Class Generation模板

使用`ClassNode`类进行Class Generation（生成类）操作分成两个步骤：

- 第一步，创建`ClassNode`类实例，为其类的字段内容进行赋值，这是收集数据的过程。
  - 首先，设置类层面的信息，包括类名、父类、实现的接口等。
  - 接着，设置字段层面的信息。
  - 最后，设置方法层面的信息。
- 第二步，借助于`ClassWriter`类，将`ClassNode`对象实例转换成`byte[]`，这是输出结果的过程。

```java
public static byte[] dump() throws Exception {
  // (1) 使用ClassNode类收集数据
  ClassNode cn = new ClassNode();
  cn.version = V1_8;
  cn.access = ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE;
  cn.name = "sample/HelloWorld";
  cn.superName = "java/lang/Object";
  // ...

  // (2) 使用ClassWriter类生成字节码
  ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
  cn.accept(cw);
  return cw.toByteArray();
}
```

### 2.1.3 示例：生成接口

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public interface HelloWorld {
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE;
    cn.name = "sample/HelloWorld";
    cn.superName = "java/lang/Object";

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);

    Field[] declaredFields = clazz.getDeclaredFields();
    if (declaredFields.length > 0) {
      System.out.println("fields:");
      for (Field f : declaredFields) {
        System.out.println("    " + f.getName());
      }
    }

    Method[] declaredMethods = clazz.getDeclaredMethods();
    if (declaredMethods.length > 0) {
      System.out.println("methods:");
      for (Method m : declaredMethods) {
        System.out.println("    " + m.getName());
      }
    }
  }
}
```

### 2.1.4 总结

本文内容总结如下：

- 第一点，介绍`ClassNode`类各个部分的信息。
- 第二点，如何使用`ClassNode`类。
- 第三点，代码示例，使用`ClassNode`类生成一个接口。

## 2.2 FieldNode介绍

### 2.2.1 FieldNode类

#### class info

第一个部分，`FieldNode`类继承自`FieldVisitor`类。

```java
/**
 * A node that represents a field.
 *
 * @author Eric Bruneton
 */
public class FieldNode extends FieldVisitor {
}
```

#### fields

第二个部分，`FieldNode`类定义的字段有哪些。

```java
public class FieldNode extends FieldVisitor {
    public int access;
    public String name;
    public String desc;
    public String signature; // 泛型信息
    public Object value; // 字段值
    // ... 下面还有一些关于annotation注解的字段，省略
}
```

这些字段与`field_info`结构相对应：

```pseudocode
field_info {
    u2             access_flags; // access
    u2             name_index; // name
    u2             descriptor_index; // desc
    u2             attributes_count;
    attribute_info attributes[attributes_count]; // signature, value 等属性
}
```

#### constructors

第三个部分，`FieldNode`类定义的构造方法有哪些。

```java
public class FieldNode extends FieldVisitor {
  public FieldNode(int access, String name, String descriptor, String signature, Object value) {
    this(Opcodes.ASM9, access, name, descriptor, signature, value);
    if (getClass() != FieldNode.class) {
      throw new IllegalStateException();
    }
  }

  public FieldNode(int api, int access, String name, String descriptor, String signature, Object value) {
    super(api);
    this.access = access;
    this.name = name;
    this.desc = descriptor;
    this.signature = signature;
    this.value = value;
  }
}
```

> 和ClassNode类似的，这里子类得调用第二个构造方法，第一个构造方法会检查当前实例的类信息是否为`FieldNode.class`

#### methods

第四个部分，`FieldNode`类定义的方法有哪些。

```java
public class FieldNode extends FieldVisitor {
  public void accept(final ClassVisitor classVisitor) {
    FieldVisitor fieldVisitor = classVisitor.visitField(access, name, desc, signature, value);
    if (fieldVisitor == null) {
      return;
    }
    // ... 省略注解和属性相关的visit和accept方法调用说明
    fieldVisitor.visitEnd();
  }
}
```

> 整体即将当前FieldNode的信息向当前绑定的(下一个)ClassVisitor传递。

### 2.2.2 示例：接口+字段

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public interface HelloWorld extends Cloneable {
  int LESS = -1;
  int EQUAL = 0;
  int GREATER = 1;
  int compareTo(Object o);
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_ABSTRACT | ACC_INTERFACE;
    cn.name = "sample/HelloWorld";
    cn.superName = "java/lang/Object";
    cn.interfaces.add("java/lang/Cloneable");

    cn.fields.add(new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "LESS", "I", null, -1));
    cn.fields.add(new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "EQUAL", "I", null, 0));
    cn.fields.add(new FieldNode(ACC_PUBLIC | ACC_FINAL | ACC_STATIC, "GREATER", "I", null, 1));
    cn.methods.add(new MethodNode(ACC_PUBLIC | ACC_ABSTRACT, "compareTo", "(Ljava/lang/Object;)I", null, null));

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);

    Field[] declaredFields = clazz.getDeclaredFields();
    if (declaredFields.length > 0) {
      System.out.println("fields:");
      for (Field f : declaredFields) {
        System.out.println("    " + f.getName());
      }
    }

    Method[] declaredMethods = clazz.getDeclaredMethods();
    if (declaredMethods.length > 0) {
      System.out.println("methods:");
      for (Method m : declaredMethods) {
        System.out.println("    " + m.getName());
      }
    }
  }
}
```

### 2.2.3 总结

本文内容总结如下：

- 第一点，介绍`FieldNode`类各个部分的信息。
- 第二点，代码示例，使用`FieldNode`类生成字段。

## 2.3 MethodNode介绍

### 2.3.1 MethodNode

#### class info

第一个部分，`MethodNode`类继承自`MethodVisitor`类。

```java
/**
 * A node that represents a method.
 *
 * @author Eric Bruneton
 */
public class MethodNode extends MethodVisitor {
}
```

#### fields

第二个部分，`MethodNode`类定义的字段有哪些。

方法，由方法头(Method Header)和方法体(Method Body)组成。在这里，我们将这些字段分成两部分：

- 第一部分，与方法头(Method Header)相关的字段。
- 第二部分，与方法体(Method Body)相关的字段。

首先，我们来看与方法头(Method Header)相关的字段：

```java
public class MethodNode extends MethodVisitor {
  public int access;
  public String name;
  public String desc;
  public String signature; // 泛型信息
  public List<String> exceptions; // 方法声明抛出的异常
}
```

这些字段与`method_info`结构相对应：

```pseudocode
method_info {
    u2             access_flags; // access
    u2             name_index; // name
    u2             descriptor_index; // desc
    u2             attributes_count;
    attribute_info attributes[attributes_count]; // signature, exceptions
}
```

接着，我们来看与方法体(Method Body)相关的字段：

```java
public class MethodNode extends MethodVisitor {
  public InsnList instructions; // 方法体内具体的操作(操作码+操作数)
  public List<TryCatchBlockNode> tryCatchBlocks; // 方法体内的异常处理逻辑
  public int maxStack; // Frame的operand stack所需申请的大小
  public int maxLocals; // Frame的local variables所需申请的大小
}
```

在上面的字段中，我们看到`InsnList`和`TryCatchBlockNode`两个类：

- `InsnList`类，表示有序的指令集合，是方法体的具体实现。它与`Code`结构当中的`code_length`和`code[]`相对应。
- `TryCatchBlockNode`类，表示方法内异常处理的逻辑。它与`Code`结构当中的`exception_table_length`和`exception_table[]`相对应。

这些字段与`Code_attribute`结构相对应：

```pseudocode
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack; // maxStack
    u2 max_locals; // maxLocals
    u4 code_length;
    u1 code[code_length]; // instructions
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length]; // tryCatchBlocks
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### constructors

第三个部分，`MethodNode`类定义的构造方法有哪些。

```java
public class MethodNode extends MethodVisitor {
  public MethodNode() {
    this(Opcodes.ASM9);
    if (getClass() != MethodNode.class) {
      throw new IllegalStateException();
    }
  }

  public MethodNode(final int api) {
    super(api);
    this.instructions = new InsnList();
  }

  public MethodNode(int access, String name, String descriptor, String signature, String[] exceptions) {
    this(Opcodes.ASM9, access, name, descriptor, signature, exceptions);
    if (getClass() != MethodNode.class) {
      throw new IllegalStateException();
    }
  }

  public MethodNode(int api, int access, String name, String descriptor, String signature, String[] exceptions) {
    super(api);
    this.access = access;
    this.name = name;
    this.desc = descriptor;
    this.signature = signature;
    this.exceptions = Util.asArrayList(exceptions);

    this.tryCatchBlocks = new ArrayList<>();
    this.instructions = new InsnList();
  }
}
```

> 和ClassNode、FieldNode相同，子类需要调用传递api参数的构造方法，因为其他几个构造方法会校验当前实例的类信息是否为`MethodNode.class`

#### methods

第四个部分，`MethodNode`类定义的方法有哪些。

##### visitXxx方法

这里介绍的`visitXxx()`方法，就是将指令存储到`InsnList instructions`字段内。

```java
public class MethodNode extends MethodVisitor {
  @Override
  public void visitCode() {
    // Nothing to do.
  }

  @Override
  public void visitInsn(final int opcode) {
    instructions.add(new InsnNode(opcode));
  }

  @Override
  public void visitIntInsn(final int opcode, final int operand) {
    instructions.add(new IntInsnNode(opcode, operand));
  }

  // ...

  @Override
  public void visitMaxs(final int maxStack, final int maxLocals) {
    this.maxStack = maxStack;
    this.maxLocals = maxLocals;
  }

  @Override
  public void visitEnd() {
    // Nothing to do.
  }
}
```

##### accept方法

在`MethodNode`类，有两个`accept`方法：一个接收`ClassVisitor`类型的参数，另一个接收`MethodVisitor`参数。

```java
public class MethodNode extends MethodVisitor {
  public void accept(ClassVisitor classVisitor) {
    String[] exceptionsArray = exceptions == null ? null : exceptions.toArray(new String[0]);
    MethodVisitor methodVisitor = classVisitor.visitMethod(access, name, desc, signature, exceptionsArray);
    if (methodVisitor != null) {
      accept(methodVisitor);
    }
  }

  public void accept(MethodVisitor methodVisitor) {
    // ...
    // Visit the code.
    if (instructions.size() > 0) {
      methodVisitor.visitCode();
      // Visits the try catch blocks.
      if (tryCatchBlocks != null) {
        for (int i = 0, n = tryCatchBlocks.size(); i < n; ++i) {
          tryCatchBlocks.get(i).updateIndex(i);
          tryCatchBlocks.get(i).accept(methodVisitor);
        }
      }
      // Visit the instructions.
      instructions.accept(methodVisitor);
      // ...
      methodVisitor.visitMaxs(maxStack, maxLocals);
      visited = true;
    }
    methodVisitor.visitEnd();
  }
}
```

### 2.3.2 示例：类

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public class HelloWorld {
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);

    Field[] declaredFields = clazz.getDeclaredFields();
    if (declaredFields.length > 0) {
      System.out.println("fields:");
      for (Field f : declaredFields) {
        System.out.println("    " + f.getName());
      }
    }

    Method[] declaredMethods = clazz.getDeclaredMethods();
    if (declaredMethods.length > 0) {
      System.out.println("methods:");
      for (Method m : declaredMethods) {
        System.out.println("    " + m.getName());
      }
    }
  }
}
```

### 2.3.3 总结

本文内容总结如下：

- 第一点，介绍`MethodNode`类各个部分的信息。
- 第二点，代码示例，使用`MethodNode`类生成方法。
- 第三点，方法内代码实现需要使用到`InsnList`类。

## 2.4 InsnList介绍

### 2.4.1 InsnList

#### class info

第一个部分，`InsnList`类实现了`Iterable<AbstractInsnNode>`接口。

- 从含义上来说，`InsnList`类表示一个有序的指令集合，而`AbstractInsnNode`则表示单条指令。
- 从结构上来说，`InsnList`类是一个存储`AbstractInsnNode`的双向链表。

```java
/**
 * A doubly linked list of {@link AbstractInsnNode} objects. <i>This implementation is not thread
 * safe</i>.
 */
public class InsnList implements Iterable<AbstractInsnNode> {
}
```

我们可以使用foreach语句对`InsnList`对象进行循环：

```java
ClassNode cn = new ClassNode();
// ...
MethodNode mn = cn.methods.get(0);
InsnList instructions = mn.instructions;
for (AbstractInsnNode insn : instructions) {
    System.out.println(insn);
}
```

#### fields

第二个部分，`InsnList`类定义的字段有哪些。

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  /** The number of instructions in this list. */
  private int size;

  /** The first instruction in this list. May be {@literal null}. */
  private AbstractInsnNode firstInsn;
  private AbstractInsnNode lastInsn;

  // A cache of the instructions of this list.
  // This cache is used to improve the performance of the get method.
  AbstractInsnNode[] cache;
}
```

#### constructors

第三个部分，`InsnList`类定义的构造方法有哪些。

```java
public class InsnList implements Iterable<AbstractInsnNode> {
    // 没有提供
}
```

#### methods

第四个部分，`InsnList`类定义的方法有哪些。

##### getter方法

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public int size() {
    return size;
  }

  public AbstractInsnNode getFirst() {
    return firstInsn;
  }

  public AbstractInsnNode getLast() {
    return lastInsn;
  }

  public AbstractInsnNode get(int index) {
    if (index < 0 || index >= size) {
      throw new IndexOutOfBoundsException();
    }
    if (cache == null) {
      cache = toArray();
    }
    return cache[index];
  }

  public AbstractInsnNode[] toArray() {
    int currentInsnIndex = 0;
    AbstractInsnNode currentInsn = firstInsn;
    AbstractInsnNode[] insnNodeArray = new AbstractInsnNode[size];
    while (currentInsn != null) {
      insnNodeArray[currentInsnIndex] = currentInsn;
      currentInsn.index = currentInsnIndex++;
      currentInsn = currentInsn.nextInsn;
    }
    return insnNodeArray;
  }
}
```

##### accept方法

在`InsnList`中，`accept`方法的作用就是将其包含的指令全部发送给下一个`MethodVisitor`对象。

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void accept(MethodVisitor methodVisitor) {
    AbstractInsnNode currentInsn = firstInsn;
    while (currentInsn != null) {
      currentInsn.accept(methodVisitor);
      currentInsn = currentInsn.nextInsn;
    }
  }
}
```

### 2.4.2 方法分类

在下面所介绍的方法也都是`InsnList`所定义的方法，我们把它们单独的拿出来有两点原因：一是内容确实比较多，二是为了分成不同的类别以方便记忆。

我们将这些方法分成“遍历－增加－删除－修改－查询”共五个类别。

#### 遍历

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public ListIterator<AbstractInsnNode> iterator() {
    return iterator(0);
  }

  public ListIterator<AbstractInsnNode> iterator(int index) {
    return new InsnListIterator(index);
  }
}
```

由于`InsnList`类实现了`Iterable`接口，我们可以直接对`InsnList`类进行foreach遍历：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.AbstractInsnNode;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.InsnList;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    // 读取字节数组byte[]
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    // 将byte[]转换成ClassNode
    ClassReader cr = new ClassReader(bytes);
    ClassNode cn = new ClassNode(Opcodes.ASM9);
    cr.accept(cn, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);

    // 遍历InsnList
    InsnList instructions = cn.methods.get(0).instructions;
    for (AbstractInsnNode insn : instructions) {
      System.out.println(insn);
    }
  }
}
```

#### 增加：开头

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void insert(AbstractInsnNode insnNode) {
    ++size;
    if (firstInsn == null) {
      firstInsn = insnNode;
      lastInsn = insnNode;
    } else {
      firstInsn.previousInsn = insnNode;
      insnNode.nextInsn = firstInsn;
    }
    firstInsn = insnNode;
    cache = null;
    insnNode.index = 0; // insnNode now belongs to an InsnList.
  }

  public void insert(InsnList insnList) {
    if (insnList.size == 0) {
      return;
    }
    size += insnList.size;
    if (firstInsn == null) {
      firstInsn = insnList.firstInsn;
      lastInsn = insnList.lastInsn;
    } else {
      AbstractInsnNode lastInsnListElement = insnList.lastInsn;
      firstInsn.previousInsn = lastInsnListElement;
      lastInsnListElement.nextInsn = firstInsn;
      firstInsn = insnList.firstInsn;
    }
    cache = null;
    insnList.removeAll(false);
  }
}
```

#### 增加：结尾

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void add(AbstractInsnNode insnNode) {
    ++size;
    if (lastInsn == null) {
      firstInsn = insnNode;
      lastInsn = insnNode;
    } else {
      lastInsn.nextInsn = insnNode;
      insnNode.previousInsn = lastInsn;
    }
    lastInsn = insnNode;
    cache = null;
    insnNode.index = 0; // insnNode now belongs to an InsnList.
  }

  public void add(InsnList insnList) {
    if (insnList.size == 0) {
      return;
    }
    size += insnList.size;
    if (lastInsn == null) {
      firstInsn = insnList.firstInsn;
      lastInsn = insnList.lastInsn;
    } else {
      AbstractInsnNode firstInsnListElement = insnList.firstInsn;
      lastInsn.nextInsn = firstInsnListElement;
      firstInsnListElement.previousInsn = lastInsn;
      lastInsn = insnList.lastInsn;
    }
    cache = null;
    insnList.removeAll(false);
  }
}
```

#### 增加：插队

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void insert(AbstractInsnNode previousInsn, AbstractInsnNode insnNode) {
    ++size;
    AbstractInsnNode nextInsn = previousInsn.nextInsn;
    if (nextInsn == null) {
      lastInsn = insnNode;
    } else {
      nextInsn.previousInsn = insnNode;
    }
    previousInsn.nextInsn = insnNode;
    insnNode.nextInsn = nextInsn;
    insnNode.previousInsn = previousInsn;
    cache = null;
    insnNode.index = 0; // insnNode now belongs to an InsnList.
  }

  public void insert(AbstractInsnNode previousInsn, InsnList insnList) {
    if (insnList.size == 0) {
      return;
    }
    size += insnList.size;
    AbstractInsnNode firstInsnListElement = insnList.firstInsn;
    AbstractInsnNode lastInsnListElement = insnList.lastInsn;
    AbstractInsnNode nextInsn = previousInsn.nextInsn;
    if (nextInsn == null) {
      lastInsn = lastInsnListElement;
    } else {
      nextInsn.previousInsn = lastInsnListElement;
    }
    previousInsn.nextInsn = firstInsnListElement;
    lastInsnListElement.nextInsn = nextInsn;
    firstInsnListElement.previousInsn = previousInsn;
    cache = null;
    insnList.removeAll(false);
  }

  public void insertBefore(AbstractInsnNode nextInsn, AbstractInsnNode insnNode) {
    ++size;
    AbstractInsnNode previousInsn = nextInsn.previousInsn;
    if (previousInsn == null) {
      firstInsn = insnNode;
    } else {
      previousInsn.nextInsn = insnNode;
    }
    nextInsn.previousInsn = insnNode;
    insnNode.nextInsn = nextInsn;
    insnNode.previousInsn = previousInsn;
    cache = null;
    insnNode.index = 0; // insnNode now belongs to an InsnList.
  }

  public void insertBefore(AbstractInsnNode nextInsn, InsnList insnList) {
    if (insnList.size == 0) {
      return;
    }
    size += insnList.size;
    AbstractInsnNode firstInsnListElement = insnList.firstInsn;
    AbstractInsnNode lastInsnListElement = insnList.lastInsn;
    AbstractInsnNode previousInsn = nextInsn.previousInsn;
    if (previousInsn == null) {
      firstInsn = firstInsnListElement;
    } else {
      previousInsn.nextInsn = firstInsnListElement;
    }
    nextInsn.previousInsn = lastInsnListElement;
    lastInsnListElement.nextInsn = nextInsn;
    firstInsnListElement.previousInsn = previousInsn;
    cache = null;
    insnList.removeAll(false);
  }
}
```

#### 删除

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void remove(AbstractInsnNode insnNode) {
    --size;
    AbstractInsnNode nextInsn = insnNode.nextInsn;
    AbstractInsnNode previousInsn = insnNode.previousInsn;
    if (nextInsn == null) {
      if (previousInsn == null) {
        firstInsn = null;
        lastInsn = null;
      } else {
        previousInsn.nextInsn = null;
        lastInsn = previousInsn;
      }
    } else {
      if (previousInsn == null) {
        firstInsn = nextInsn;
        nextInsn.previousInsn = null;
      } else {
        previousInsn.nextInsn = nextInsn;
        nextInsn.previousInsn = previousInsn;
      }
    }
    cache = null;
    insnNode.index = -1; // insnNode no longer belongs to an InsnList.
    insnNode.previousInsn = null;
    insnNode.nextInsn = null;
  }

  void removeAll(boolean mark) {
    if (mark) {
      AbstractInsnNode currentInsn = firstInsn;
      while (currentInsn != null) {
        AbstractInsnNode next = currentInsn.nextInsn;
        currentInsn.index = -1; // currentInsn no longer belongs to an InsnList.
        currentInsn.previousInsn = null;
        currentInsn.nextInsn = null;
        currentInsn = next;
      }
    }
    size = 0;
    firstInsn = null;
    lastInsn = null;
    cache = null;
  }

  public void clear() {
    removeAll(false);
  }
}
```

#### 修改

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public void set(AbstractInsnNode oldInsnNode, AbstractInsnNode newInsnNode) {
    // 处理与后续指令之间的关系
    AbstractInsnNode nextInsn = oldInsnNode.nextInsn;
    newInsnNode.nextInsn = nextInsn;
    if (nextInsn != null) {
      nextInsn.previousInsn = newInsnNode;
    } else {
      lastInsn = newInsnNode;
    }

    // 处理与前面指令之间的关系
    AbstractInsnNode previousInsn = oldInsnNode.previousInsn;
    newInsnNode.previousInsn = previousInsn;
    if (previousInsn != null) {
      previousInsn.nextInsn = newInsnNode;
    } else {
      firstInsn = newInsnNode;
    }

    // 新指令获取索引信息
    if (cache != null) {
      int index = oldInsnNode.index;
      cache[index] = newInsnNode;
      newInsnNode.index = index;
    } else {
      newInsnNode.index = 0; // newInsnNode now belongs to an InsnList.
    }

    // 旧指令移除索引信息
    oldInsnNode.index = -1; // oldInsnNode no longer belongs to an InsnList.
    oldInsnNode.previousInsn = null;
    oldInsnNode.nextInsn = null;
  }
}
```

#### 查询

```java
public class InsnList implements Iterable<AbstractInsnNode> {
  public boolean contains(AbstractInsnNode insnNode) {
    AbstractInsnNode currentInsn = firstInsn;
    while (currentInsn != null && currentInsn != insnNode) {
      currentInsn = currentInsn.nextInsn;
    }
    return currentInsn != null;
  }

  public int indexOf(AbstractInsnNode insnNode) {
    if (cache == null) {
      cache = toArray();
    }
    return insnNode.index;
  }
}
```

### 2.4.3 InsnList类的特点

An `InsnList` is a doubly linked list of instructions, whose links are stored in the `AbstractInsnNode` objects themselves. This point is extremely important because it has many consequences on the way instruction objects and instruction lists must be used:

- An `AbstractInsnNode` object cannot appear more than once in an instruction list.
- An `AbstractInsnNode` object cannot belong to several instruction lists at the same time.
- As a consequence, adding an `AbstractInsnNode` to a list requires removing it from the list to which it belonged, if any.
- As another consequence, adding all the elements of a list into another one clears the first list.

> 简言之，InsnList是一个双向链表，存储的是AbstractInsnNode，由于其记录了前后节点的信息，所以操作时需要注意修改前后节点信息，避免指针混乱的问题。

### 2.4.4 总结

本文内容总结如下：

- 第一点，介绍`InsnList`类各个部分的信息。
- 第二点，对`InsnList`类中的方法进行分类，分成遍历、增、删、改、查五个类别，目的是方便从概括的角度来把握这些方法。
- 第三点，在`InsnList`类中，`AbstractInsnNode`表现出来的特点就是“一臣不能事二主”。

## 2.5 AbstractInsnNode介绍

### 2.5.1 AbstractInsnNode

#### class info

第一个部分，`AbstractInsnNode`类是一个抽象（`abstract`）类。

```java
/**
 * A node that represents a bytecode instruction. <i>An instruction can appear at most once in at
 * most one {@link InsnList} at a time</i>.
 *
 * @author Eric Bruneton
 */
public abstract class AbstractInsnNode {
}
```

#### fields

第二个部分，`AbstractInsnNode`类定义的字段有哪些。

- `opcode`字段，记录当前指令是什么。
- `previousInsn`和`nextInsn`字段，用来记录不同指令之间的关联关系。
- `index`字段，用来记录当前指令在`InsnList`对象实例中索引值。
  - 如果当前指令没有加入任何`InsnList`对象实例，其`index`值为`-1`。
  - 如果当前指令刚加入某个`InsnList`对象实例时，其`index`值为`0`；在调用`InsnList.toArray()`方法后，会更新其`index`值。

```java
public abstract class AbstractInsnNode {
  protected int opcode;

  AbstractInsnNode previousInsn;
  AbstractInsnNode nextInsn;

  int index;
}
```

#### constructors

第三个部分，`AbstractInsnNode`类定义的构造方法有哪些。

```java
public abstract class AbstractInsnNode {
  protected AbstractInsnNode(final int opcode) {
    this.opcode = opcode;
    this.index = -1;
  }
}
```

#### methods

第四个部分，`AbstractInsnNode`类定义的方法有哪些。

##### getter方法

```java
public abstract class AbstractInsnNode {
  public int getOpcode() {
    return opcode;
  }

  public AbstractInsnNode getPrevious() {
    return previousInsn;
  }

  public AbstractInsnNode getNext() {
    return nextInsn;
  }
}
```

##### 抽象方法

我们知道，`AbstractInsnNode`本身就是一个抽象类，它里面有两个抽象方法：

- `getType()`方法，用来获取当前指令的类型。
- `accept(MethodVisitor)`方法，用来将当前指令发送给下一个`MethodVisitor`对象实例。

```java
public abstract class AbstractInsnNode {
  public abstract int getType();

  public abstract void accept(MethodVisitor methodVisitor);
}
```

### 2.5.2 指令分类

在`AbstractInsnNode`类当中，`getType()`是一个抽象方法，它具体的取值范围位于`0~15`之间，一共16个类别。这16个类型，也对应了16个具体的子类实现；由于这些子类的数量较多，并且代码实现也比较简单，我们就不一一进行介绍了。

```java
public abstract class AbstractInsnNode {
  // The type of InsnNode instructions.
  public static final int INSN = 0;

  // The type of IntInsnNode instructions.
  public static final int INT_INSN = 1;

  // The type of VarInsnNode instructions.
  public static final int VAR_INSN = 2;

  // The type of TypeInsnNode instructions.
  public static final int TYPE_INSN = 3;

  // The type of FieldInsnNode instructions.
  public static final int FIELD_INSN = 4;

  // The type of MethodInsnNode instructions.
  public static final int METHOD_INSN = 5;

  // The type of InvokeDynamicInsnNode instructions.
  public static final int INVOKE_DYNAMIC_INSN = 6;

  // The type of JumpInsnNode instructions.
  public static final int JUMP_INSN = 7;

  // The type of LabelNode "instructions".
  public static final int LABEL = 8;

  // The type of LdcInsnNode instructions.
  public static final int LDC_INSN = 9;

  // The type of IincInsnNode instructions.
  public static final int IINC_INSN = 10;

  // The type of TableSwitchInsnNode instructions.
  public static final int TABLESWITCH_INSN = 11;

  // The type of LookupSwitchInsnNode instructions.
  public static final int LOOKUPSWITCH_INSN = 12;

  // The type of MultiANewArrayInsnNode instructions.
  public static final int MULTIANEWARRAY_INSN = 13;

  // The type of FrameNode "instructions".
  public static final int FRAME = 14;

  // The type of LineNumberNode "instructions".
  public static final int LINE = 15; // 用于调试，可以忽略
}
```

### 2.5.3 示例：打印字符串

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    System.out.println("Hello World");
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    {
      MethodNode mn2 = new MethodNode(ACC_PUBLIC, "test", "()V", null, null);
      cn.methods.add(mn2);

      InsnList il = mn2.instructions;
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("Hello World"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new InsnNode(RETURN));

      mn2.maxStack = 2;
      mn2.maxLocals = 1;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
    return cw.toByteArray();
  }
}
```

#### 验证结果

```java
import sample.HelloWorld;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test();
  }
}
```

### 2.5.4 总结

本文内容总结如下：

- 第一点，介绍`AbstractInsnNode`类各个部分的信息。
- 第二点，`AbstractInsnNode`是一个抽象类，它有16个具体子类。
- 第三点，代码示例，使用`AbstractInsnNode`的子类生成打印字符串的代码。

## 2.6 if和switch示例

实现if语句，要用到`JumpInsnNode`类；而实现switch语句，则需要用到`TableSwitchInsnNode`或`LookupSwitchInsnNode`类。

### 2.6.1 示例：if语句

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    if (val == 0) {
      System.out.println("val is 0");
    }
    else {
      System.out.println("val is not 0");
    }
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    {
      MethodNode mn2 = new MethodNode(ACC_PUBLIC, "test", "(I)V", null, null);
      cn.methods.add(mn2);

      LabelNode elseLabelNode = new LabelNode();
      LabelNode returnLabelNode = new LabelNode();

      // 第1段
      InsnList il = mn2.instructions;
      il.add(new VarInsnNode(ILOAD, 1));
      il.add(new JumpInsnNode(IFNE, elseLabelNode));

      // 第2段
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val is 0"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第3段
      il.add(elseLabelNode);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val is not 0"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));

      // 第4段
      il.add(returnLabelNode);
      il.add(new InsnNode(RETURN));

      mn2.maxStack = 2;
      mn2.maxLocals = 2;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
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
    Method m = clazz.getDeclaredMethod("test", int.class);
    Object instance = clazz.newInstance();
    m.invoke(instance, 0);
  }
}
```

### 2.6.2 示例：tableswitch

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

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

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    {
      MethodNode mn2 = new MethodNode(ACC_PUBLIC, "test", "(I)V", null, null);
      cn.methods.add(mn2);

      LabelNode caseLabelNode1 = new LabelNode();
      LabelNode caseLabelNode2 = new LabelNode();
      LabelNode caseLabelNode3 = new LabelNode();
      LabelNode caseLabelNode4 = new LabelNode();
      LabelNode defaultLabelNode = new LabelNode();
      LabelNode returnLabelNode = new LabelNode();

      // 第1段
      InsnList il = mn2.instructions;
      il.add(new VarInsnNode(ILOAD, 1));
      il.add(new TableSwitchInsnNode(1, 4, defaultLabelNode, new LabelNode[] { caseLabelNode1, caseLabelNode2, caseLabelNode3, caseLabelNode4 }));

      // 第2段
      il.add(caseLabelNode1);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 1"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第3段
      il.add(caseLabelNode2);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 2"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第4段
      il.add(caseLabelNode3);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 3"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第5段
      il.add(caseLabelNode4);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 4"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第6段
      il.add(defaultLabelNode);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val is unknown"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));

      // 第7段
      il.add(returnLabelNode);
      il.add(new InsnNode(RETURN));

      mn2.maxStack = 2;
      mn2.maxLocals = 2;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
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
    Method m = clazz.getDeclaredMethod("test", int.class);
    Object instance = clazz.newInstance();
    for (int i = 0; i < 5; i++) {
      m.invoke(instance, i);
    }
  }
}
```

### 2.6.3 示例：lookupswitch

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    switch (val) {
      case 10:
        System.out.println("val = 10");
        break;
      case 20:
        System.out.println("val = 20");
        break;
      case 30:
        System.out.println("val = 30");
        break;
      case 40:
        System.out.println("val = 40");
        break;
      default:
        System.out.println("val is unknown");
    }
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    {
      MethodNode mn2 = new MethodNode(ACC_PUBLIC, "test", "(I)V", null, null);
      cn.methods.add(mn2);

      LabelNode caseLabelNode1 = new LabelNode();
      LabelNode caseLabelNode2 = new LabelNode();
      LabelNode caseLabelNode3 = new LabelNode();
      LabelNode caseLabelNode4 = new LabelNode();
      LabelNode defaultLabelNode = new LabelNode();
      LabelNode returnLabelNode = new LabelNode();

      // 第1段
      InsnList il = mn2.instructions;
      il.add(new VarInsnNode(ILOAD, 1));
      il.add(new LookupSwitchInsnNode(defaultLabelNode, new int[] { 10, 20, 30, 40 }, 
                                      new LabelNode[] { caseLabelNode1, caseLabelNode2, caseLabelNode3, caseLabelNode4 }));

      // 第2段
      il.add(caseLabelNode1);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 10"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第3段
      il.add(caseLabelNode2);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 20"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第4段
      il.add(caseLabelNode3);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 30"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第5段
      il.add(caseLabelNode4);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val = 40"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      // 第6段
      il.add(defaultLabelNode);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("val is unknown"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));

      // 第7段
      il.add(returnLabelNode);
      il.add(new InsnNode(RETURN));

      mn2.maxStack = 2;
      mn2.maxLocals = 2;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
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
    Method m = clazz.getDeclaredMethod("test", int.class);
    Object instance = clazz.newInstance();
    for (int i = 0; i < 5; i++) {
      m.invoke(instance, i * 10);
    }
  }
}
```

### 2.6.4 总结

本文内容总结如下：

- 第一点，实现if语句，要用到`JumpInsnNode`类。
- 第二点，实现switch语句，要用到`TableSwitchInsnNode`或`LookupSwitchInsnNode`类。

> 循环语句在底层实际就是通过if和jump跳转实现的，所以没有单独的指令

## 2.7 TryCatchBlockNode介绍

### 2.7.1 TryCatchBlockNode

#### class info

第一个部分，**`TryCatchBlockNode`类直接继承自`Object`类。注意：`TryCatchBlockNode`的父类并不是`AbstractInsnNode`类**。(毕竟在JVM规范中，`Code_attribute`的`code[code_length]`存储方法体中的指令，而`exception_table[exception_table_length]`存储方法体中异常处理的信息，两者是分开的字段)

```java
/**
 * A node that represents a try catch block.
 *
 * @author Eric Bruneton
 */
public class TryCatchBlockNode {
}
```

#### fields

第二个部分，`TryCatchBlockNode`类定义的字段有哪些。

我们可以将字段分成两组：

- 第一组字段，包括`start`和`end`字段，用来标识异常处理的范围（`start~end`）。
- 第二组字段，包括`handler`和`type`字段，用来标识异常处理的类型（`type`字段）和手段（`handler`字段）。

```java
public class TryCatchBlockNode {
  // 第一组字段
  public LabelNode start;
  public LabelNode end;

  // 第二组字段
  public LabelNode handler;
  public String type;
}
```

#### constructors

第三个部分，`TryCatchBlockNode`类定义的构造方法有哪些。

```java
public class TryCatchBlockNode {
  public TryCatchBlockNode(LabelNode start, LabelNode end, LabelNode handler, String type) {
    this.start = start;
    this.end = end;
    this.handler = handler;
    this.type = type;
  }
}
```

#### methods

第四个部分，`TryCatchBlockNode`类定义的方法有哪些。在这里，我们只关注`accept`方法，它接收一个`MethodVisitor`类型的参数。

```java
public class TryCatchBlockNode {
  public void accept(MethodVisitor methodVisitor) {
    methodVisitor.visitTryCatchBlock(start.getLabel(), end.getLabel(), handler == null ? null : handler.getLabel(), type);
  }
}
```

### 2.7.2 示例：try-catch

#### 预期目标

我们想实现的预期目标是生成`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    try {
      System.out.println("Before Sleep");
      Thread.sleep(1000L);
      System.out.println("After Sleep");
    }
    catch (InterruptedException ex) {
      ex.printStackTrace();
    }
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldGenerateTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);

    // (1) 生成byte[]内容
    byte[] bytes = dump();

    // (2) 保存byte[]到文件
    FileUtils.writeBytes(filepath, bytes);
  }

  public static byte[] dump() throws Exception {
    // (1) 使用ClassNode类收集数据
    ClassNode cn = new ClassNode();
    cn.version = V1_8;
    cn.access = ACC_PUBLIC | ACC_SUPER;
    cn.name = "sample/HelloWorld";
    cn.signature = null;
    cn.superName = "java/lang/Object";

    {
      MethodNode mn1 = new MethodNode(ACC_PUBLIC, "<init>", "()V", null, null);
      cn.methods.add(mn1);

      InsnList il = mn1.instructions;
      il.add(new VarInsnNode(ALOAD, 0));
      il.add(new MethodInsnNode(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false));
      il.add(new InsnNode(RETURN));

      mn1.maxStack = 1;
      mn1.maxLocals = 1;
    }

    {
      MethodNode mn2 = new MethodNode(ACC_PUBLIC, "test", "()V", null, null);
      cn.methods.add(mn2);

      LabelNode startLabelNode = new LabelNode();
      LabelNode endLabelNode = new LabelNode();
      LabelNode exceptionHandlerLabelNode = new LabelNode();
      LabelNode returnLabelNode = new LabelNode();

      InsnList il = mn2.instructions;
      mn2.tryCatchBlocks.add(new TryCatchBlockNode(startLabelNode, endLabelNode, exceptionHandlerLabelNode, "java/lang/InterruptedException"));

      il.add(startLabelNode);
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("Before Sleep"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      il.add(new LdcInsnNode(new Long(1000L)));
      il.add(new MethodInsnNode(INVOKESTATIC, "java/lang/Thread", "sleep", "(J)V", false));
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("After Sleep"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));

      il.add(endLabelNode);
      il.add(new JumpInsnNode(GOTO, returnLabelNode));

      il.add(exceptionHandlerLabelNode);
      il.add(new VarInsnNode(ASTORE, 1));
      il.add(new VarInsnNode(ALOAD, 1));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/lang/InterruptedException", "printStackTrace", "()V", false));

      il.add(returnLabelNode);
      il.add(new InsnNode(RETURN));

      mn2.maxStack = 2;
      mn2.maxLocals = 2;
    }

    // (2) 使用ClassWriter类生成字节码
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);
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
    Method m = clazz.getDeclaredMethod("test");
    Object instance = clazz.newInstance();
    m.invoke(instance);
  }
}
```

### 2.7.3 总结

本文内容总结如下：

- 第一点，介绍`TryCatchBlockNode`类各个部分的信息。
- 第二点，代码示例，如何使用`TryCatchBlockNode`类生成try-catch语句。

# 3. Class Transformation

## 3.1 Tree Based Class Transformation

### 3.1.1 Core Based Class Transformation

在Core API当中，使用`ClassReader`、`ClassVisitor`和`ClassWriter`类来进行Class Transformation操作的整体思路是这样的：

```pseudocode
ClassReader --> ClassVisitor(1) --> ... --> ClassVisitor(N) --> ClassWriter
```

在这些类当中，它们有各自的职责：

- `ClassReader`类负责“读”Class。
- `ClassWriter`类负责“写”Class。
- `ClassVisitor`类负责进行“转换”（Transformation）。

因此，我们可以说，`ClassVisitor`类是Class Transformation的核心操作。

### 3.1.2 Class Transformation的本质

对于Class Transformation来说，它的本质就是“中间人攻击”（Man-in-the-middle attack）。

在[Wiki](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)当中，是这样描述Man-in-the-middle attack的：

> In cryptography and computer security, a man-in-the-middle(MITM) attack is a cyberattack where the attacker secretly relays and possibly alters the communications between two parties who believe that they are directly communicating with each other.

![Java ASM系列：（078）Tree Based Class Transformation_Bytecode](https://s2.51cto.com/images/20210628/1624882180784922.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 3.1.3 Tree Based Class Transformation

首先，思考一个问题：基于Tree API的Class Transformation要怎么进行呢？它是要完全开发一套全新的处理流程，还是利用已有的Core API的Class Transformation流程呢？

回答：要实现Tree API的Class Transformation，Java ASM利用了已有的Core API的Class Transformation流程。

```pseudocode
ClassReader --> ClassVisitor(1) --> ... --> ClassNode(M) --> ... --> ClassVisitor(N) --> ClassWriter
```

因为`ClassNode`类（Tree API）是继承自`ClassVisitor`类（Core API），因此这里的处理流程和上面的处理流程本质上一样的。

虽然处理流程本质上是一样的，但是还有三个具体的技术细节需要处理：

- 第一个，如何将Core API（`ClassReader`和`ClassVisitor`）转换成Tree API（`ClassNode`）。
- 第二个，如何将Tree API（`ClassNode`）转换成Core API（`ClassVisitor`和`ClassWriter`）。
- 第三个，如何对`ClassNode`进行转换。

#### 从Core API到Tree API

从Core API到Tree API的转换，有两种情况。

第一种情况，将`ClassReader`类转换成`ClassNode`类，要依赖于`ClassReader`的`accept(ClassVisitor)`方法：

```java
ClassNode cn = new ClassNode(Opcodes.ASM9);

ClassReader cr = new ClassReader(bytes);
int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cn, parsingOptions);
```

第二种情况，将`ClassVisitor`类转换成`ClassNode`类，要依赖于`ClassVisitor`的构造方法：

```java
int api = Opcodes.ASM9;
ClassNode cn = new ClassNode();
ClassVisitor cv = new XxxClassVisitor(api, cn);
```

#### 从Tree API到Core API

从Tree API到Core API的转换，要依赖于`ClassNode`的`accept(ClassVisitor)`方法：

```java
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
cn.accept(cw);
```

#### 如何对ClassNode进行转换

对`ClassNode`对象实例进行转换，其实就是对其字段的值进行修改。

##### 第一个版本

首先，我们来看第一个版本，就是在拿到`ClassNode cn`之后，直接对`cn`里的字段值进行修改。

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);
    if (bytes1 == null) {
      throw new RuntimeException("bytes1 is null");
    }

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);
    cr.accept(cn, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);

    // (3) 进行transform
    cn.interfaces.add("java/lang/Cloneable");

    // (4) 构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);

    // (5) 生成byte[]内容输出
    byte[] bytes2 = cw.toByteArray();
    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

这个版本有点“鲁莽”和“原始”，如果进行变换的内容复杂，代码就会变得很“臃肿”，缺少一点面向对象的“美”。那么，怎么改进呢？

##### 第二个版本

在第二个版本中，我们就引入一个`ClassTransformer`类，它的作用就是将“需要进行的变换”封装成一个“类”。

```java
import org.objectweb.asm.tree.ClassNode;

public class ClassTransformer {
  protected ClassTransformer ct;

  public ClassTransformer(ClassTransformer ct) {
    this.ct = ct;
  }

  public void transform(ClassNode cn) {
    if (ct != null) {
      ct.transform(cn);
    }
  }
}
```

对于`ClassTransformer`类，我们主要理解两点内容：

- 第一点，`transform()`方法是主要的关注点，它的作用是对某一个`ClassNode`对象进行转换。
- 第二点，`ct`字段是次要的关注点，它的作用是将多个`ClassTransformer`对象连接起来，这就能够对某一个`ClassNode`对象进行连续多次处理。

代码片段：

```java
// (1)构建ClassReader
ClassReader cr = new ClassReader(bytes1);

// (2) 构建ClassNode
int api = Opcodes.ASM9;
ClassNode cn = new ClassNode(api);
cr.accept(cn, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);

// (3) 进行transform
ClassTransformer ct = ...;
ct.transform(cn);

// (4) 构建ClassWriter
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
cn.accept(cw);

// (5) 生成byte[]内容输出
byte[] bytes2 = cw.toByteArray();
```

完整代码示例：

```java
import lsieun.asm.tree.transformer.ClassAddFieldTransformer;
import lsieun.asm.tree.transformer.ClassAddMethodTransformer;
import lsieun.asm.tree.transformer.ClassTransformer;
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.InsnNode;
import org.objectweb.asm.tree.MethodNode;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);
    if (bytes1 == null) {
      throw new RuntimeException("bytes1 is null");
    }

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2) 构建ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);
    cr.accept(cn, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);

    // (3) 进行transform
    ClassTransformer ct1 = new ClassAddFieldTransformer(null, Opcodes.ACC_PUBLIC, "intValue", "I");
    ClassTransformer ct2 = new ClassAddMethodTransformer(ct1, Opcodes.ACC_PUBLIC, "abc", "()V") {
      @Override
      protected void generateMethodBody(MethodNode mn) {
        InsnList il = mn.instructions;
        il.add(new InsnNode(Opcodes.RETURN));
        mn.maxStack = 0;
        mn.maxLocals = 1;
      }
    };
    ct2.transform(cn);

    // (4) 构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    cn.accept(cw);

    // (5) 生成byte[]内容输出
    byte[] bytes2 = cw.toByteArray();
    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

第二个版本，其实挺好的，但是仍然有改进的余地，就是“学会内敛”。“学会内敛”是什么意思呢？我们结合生活当中的例子来说一下，有一句话叫“腹有诗书气自华”。 如果你有才华，但到处卖弄，会非常招人讨厌；但如果你将才华藏于自身，不轻易示人，这样你的才华就会体现你的气质中。

那么，应该怎么进一步改进呢？在ASM的官方文档（[asm4-guide.pdf](https://asm.ow2.io/asm4-guide.pdf)）提出了两种Common Patterns。

### 3.1.4 Two Common Patterns

#### First Pattern

The first pattern uses inheritance:

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;

public class MyClassNode extends ClassNode {
  public MyClassNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    // put your transformation code here
    // 使用ClassTransformer进行转换

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }
}
```

那么，可以对`MyClassNode`按如下思路进行使用：

```java
// (1)构建ClassReader
ClassReader cr = new ClassReader(bytes1);

// (2)构建ClassWriter
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

// (3)串连ClassNode
int api = Opcodes.ASM9;
ClassNode cn = new MyClassNode(api, cw);

//（4）结合ClassReader和ClassNode
int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cn, parsingOptions);

// (5) 生成byte[]
byte[] bytes2 = cw.toByteArray();
```

#### Second Pattern

The second pattern uses delegation instead of inheritance:

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;

public class MyClassVisitor extends ClassVisitor {
  private final ClassVisitor next;

  public MyClassVisitor(int api, ClassVisitor classVisitor) {
    super(api, new ClassNode());    // 注意一：这里创建了一个ClassNode对象
    this.next = classVisitor;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassNode cn = (ClassNode) cv;    // 注意二：这里获取的是上面创建的ClassNode对象
    // put your transformation code here
    // 使用ClassTransformer进行转换

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (next != null) {
      cn.accept(next);
    }
  }
}
```

那么，可以对`MyClassVisitor`按如下思路进行使用：

```java
//（1）构建ClassReader
ClassReader cr = new ClassReader(bytes1);

//（2）构建ClassWriter
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

//（3）串连ClassVisitor
int api = Opcodes.ASM9;
ClassVisitor cv = new MyClassVisitor(api, cw);

//（4）结合ClassReader和ClassVisitor
int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cv, parsingOptions);

//（5）生成byte[]
byte[] bytes2 = cw.toByteArray();
```

### 3.1.5 总结

本文内容总结如下：

- 第一点，介绍Core Based Class Transformation的处理流程是什么。
- 第二点，Class Transformation的本质是什么。（中间人攻击）
- 第三点，Tree Based Class Transformation是利用了已有的Core API处理流程，在过程中需要解决三个技术细节问题：
  - 第一个问题，如何将Core API转换成Tree API
  - 第二个问题，如何将Tree API转换成Core API
  - 第三个问题，如何对`ClassNode`类进行转换
- 第四点，使用Tree API进行Class Transformation的两种Pattern是什么。

在刚接触Tree Based Class Transformation的时候，可能不知道如何开始着手，我们可以按下面的步骤来进行思考：

- 第一步，读取具体的`.class`文件，是使用`ClassReader`类，它属于Core API的内容。
- 第二步，思考如何将Core API转换成Tree API。
- 第三步，思考如何使用Tree API进行Class Transformation操作。
- 第四步，思考如何将Tree API转换成Core API。
- 第五步，最后落实到`ClassWriter`类，调用其`toByteArray()`方法来生成`byte[]`内容。

## 3.2 Tree Based Class Transformation示例

### 3.2.1 整体思路

使用Tree API进行Class Transformation的思路：

```pseudocode
ClassReader --> ClassNode --> ClassWriter
```

其中，

- `ClassReader`类负责“读”Class。
- `ClassWriter`类负责“写”Class。
- `ClassNode`类负责进行“转换”（Transformation）。

### 3.2.2 示例一：删除字段

#### 预期目标

预期目标：删除掉`HelloWorld`类里的`String strValue`字段。

```java
public class HelloWorld {
  public int intValue;
  public String strValue; // 删除这个字段
}
```

#### 编码实现

```java
import lsieun.asm.tree.transformer.ClassTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;

public class ClassRemoveFieldNode extends ClassNode {
  private final String fieldName;
  private final String fieldDesc;

  public ClassRemoveFieldNode(int api, ClassVisitor cv, String fieldName, String fieldDesc) {
    super(api);
    this.cv = cv;
    this.fieldName = fieldName;
    this.fieldDesc = fieldDesc;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassTransformer ct = new ClassRemoveFieldTransformer(null, fieldName, fieldDesc);
    ct.transform(this);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class ClassRemoveFieldTransformer extends ClassTransformer {
    private final String fieldName;
    private final String fieldDesc;

    public ClassRemoveFieldTransformer(ClassTransformer ct, String fieldName, String fieldDesc) {
      super(ct);
      this.fieldName = fieldName;
      this.fieldDesc = fieldDesc;
    }

    @Override
    public void transform(ClassNode cn) {
      // 首先，处理自己的代码逻辑
      cn.fields.removeIf(fn -> fieldName.equals(fn.name) && fieldDesc.equals(fn.desc));

      // 其次，调用父类的方法实现
      super.transform(cn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassRemoveFieldNode(api, cw, "strValue", "Ljava/lang/String;");

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    System.out.println(clazz);

    Field[] declaredFields = clazz.getDeclaredFields();
    if (declaredFields.length > 0) {
      System.out.println("fields:");
      for (Field f : declaredFields) {
        System.out.println("    " + f.getName());
      }
    }

    Method[] declaredMethods = clazz.getDeclaredMethods();
    if (declaredMethods.length > 0) {
      System.out.println("methods:");
      for (Method m : declaredMethods) {
        System.out.println("    " + m.getName());
      }
    }
  }
}
```

### 3.2.3 示例二：添加字段

#### 预期目标

预期目标：为了`HelloWorld`类添加一个`Object objValue`字段。

```java
public class HelloWorld {
  public int intValue;
  public String strValue;
  // 添加一个Object objValue字段
}
```

#### 编码实现

```java
import lsieun.asm.tree.transformer.ClassTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.FieldNode;

public class ClassAddFieldNode extends ClassNode {
  private final int fieldAccess;
  private final String fieldName;
  private final String fieldDesc;

  public ClassAddFieldNode(int api, ClassVisitor cv,
                           int fieldAccess, String fieldName, String fieldDesc) {
    super(api);
    this.cv = cv;
    this.fieldAccess = fieldAccess;
    this.fieldName = fieldName;
    this.fieldDesc = fieldDesc;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassTransformer ct = new ClassAddFieldTransformer(null, fieldAccess, fieldName, fieldDesc);
    ct.transform(this);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class ClassAddFieldTransformer extends ClassTransformer {
    private final int fieldAccess;
    private final String fieldName;
    private final String fieldDesc;

    public ClassAddFieldTransformer(ClassTransformer ct, int fieldAccess, String fieldName, String fieldDesc) {
      super(ct);
      this.fieldAccess = fieldAccess;
      this.fieldName = fieldName;
      this.fieldDesc = fieldDesc;
    }

    @Override
    public void transform(ClassNode cn) {
      // 首先，处理自己的代码逻辑
      boolean isPresent = false;
      for (FieldNode fn : cn.fields) {
        if (fieldName.equals(fn.name)) {
          isPresent = true;
          break;
        }
      }
      if (!isPresent) {
        cn.fields.add(new FieldNode(fieldAccess, fieldName, fieldDesc, null, null));
      }

      // 其次，调用父类的方法实现
      super.transform(cn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassAddFieldNode(api, cw, Opcodes.ACC_PUBLIC, "objValue", "Ljava/lang/Object;");

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 3.2.4 示例三：删除方法

#### 预期目标

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

#### 编码实现

```java
import lsieun.asm.tree.transformer.ClassTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;

public class ClassRemoveMethodNode extends ClassNode {
  private final String methodName;
  private final String methodDesc;

  public ClassRemoveMethodNode(int api, ClassVisitor cv, String methodName, String methodDesc) {
    super(api);
    this.cv = cv;
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassTransformer ct = new ClassRemoveMethodTransformer(null, methodName, methodDesc);
    ct.transform(this);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class ClassRemoveMethodTransformer extends ClassTransformer {
    private final String methodName;
    private final String methodDesc;

    public ClassRemoveMethodTransformer(ClassTransformer ct, String methodName, String methodDesc) {
      super(ct);
      this.methodName = methodName;
      this.methodDesc = methodDesc;
    }

    @Override
    public void transform(ClassNode cn) {
      // 首先，处理自己的代码逻辑
      cn.methods.removeIf(mn -> methodName.equals(mn.name) && methodDesc.equals(mn.desc));

      // 其次，调用父类的方法实现
      super.transform(cn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassRemoveMethodNode(api, cw, "add", "(II)I");

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 3.2.5 示例四：添加方法

#### 预期目标

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

#### 编码实现

```java
import lsieun.asm.tree.transformer.ClassTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.MethodNode;

import java.util.function.Consumer;

public class ClassAddMethodNode extends ClassNode {
  private final int methodAccess;
  private final String methodName;
  private final String methodDesc;
  private final Consumer<MethodNode> methodBody;

  public ClassAddMethodNode(int api, ClassVisitor cv,
                            int methodAccess, String methodName, String methodDesc,
                            Consumer<MethodNode> methodBody) {
    super(api);
    this.cv = cv;
    this.methodAccess = methodAccess;
    this.methodName = methodName;
    this.methodDesc = methodDesc;
    this.methodBody = methodBody;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassTransformer ct = new ClassAddMethodTransformer(null, methodAccess, methodName, methodDesc, methodBody);
    ct.transform(this);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class ClassAddMethodTransformer extends ClassTransformer {
    private final int methodAccess;
    private final String methodName;
    private final String methodDesc;
    private final Consumer<MethodNode> methodBody;

    public ClassAddMethodTransformer(ClassTransformer ct,
                                     int methodAccess, String methodName, String methodDesc,
                                     Consumer<MethodNode> methodBody) {
      super(ct);
      this.methodAccess = methodAccess;
      this.methodName = methodName;
      this.methodDesc = methodDesc;
      this.methodBody = methodBody;
    }

    @Override
    public void transform(ClassNode cn) {
      // 首先，处理自己的代码逻辑
      boolean isPresent = false;
      for (MethodNode mn : cn.methods) {
        if (methodName.equals(mn.name) && methodDesc.equals(mn.desc)) {
          isPresent = true;
          break;
        }
      }
      if (!isPresent) {
        MethodNode mn = new MethodNode(methodAccess, methodName, methodDesc, null, null);
        cn.methods.add(mn);

        if (methodBody != null) {
          methodBody.accept(mn);
        }
      }

      // 其次，调用父类的方法实现
      super.transform(cn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.*;

import java.util.function.Consumer;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    Consumer<MethodNode> methodBody = (mn) -> {
      InsnList il = mn.instructions;
      il.add(new VarInsnNode(ILOAD, 1));
      il.add(new VarInsnNode(ILOAD, 2));
      il.add(new InsnNode(IMUL));
      il.add(new InsnNode(IRETURN));

      mn.maxStack = 2;
      mn.maxLocals = 3;
    };
    ClassNode cn = new ClassAddMethodNode(api, cw, ACC_PUBLIC, "mul", "(II)I", methodBody);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 3.2.6 总结

本文内容总结如下：

- 第一点，代码示例，如何删除和添加字段。
- 第二点，代码示例，如何删除和添加方法。

## 3.3 Tree Based Method Transformation

在在ASM的官方文档（[asm4-guide.pdf](https://asm.ow2.io/asm4-guide.pdf)）中，对Tree-Based Transformation分成了两个不同的层次：

- 类层面
- 方法层面

```
                                                ┌─── ClassTransformer
                             ┌─── ClassNode ────┤
                             │                  └─── Two Common Patterns
Tree-Based Transformation ───┤
                             │                  ┌─── MethodTransformer
                             └─── MethodNode ───┤
                                                └─── Two Common Patterns
```

为什么没有“字段层面”呢？主要是因为字段的处理比较简单；而方法的处理则要复杂很多，方法有方法头（method header）和方法体（method body），方法体里有指令（`InsnList`）和异常处理的逻辑（`TryCatchBlockNode`），有足够的理由成为一个单独的讨论话题。

值得一提的是，对于Tree-Based Transformation来说，类层面和方法层面，两者虽然在使用细节上有差异，但在“整体的处理思路”上有非常大的相似性。

### 3.3.1 MethodTransformer

```java
import org.objectweb.asm.tree.MethodNode;

public class MethodTransformer {
  protected MethodTransformer mt;

  public MethodTransformer(MethodTransformer mt) {
    this.mt = mt;
  }

  public void transform(MethodNode mn) {
    if (mt != null) {
      mt.transform(mn);
    }
  }
}
```

### 3.3.2 Two Common Patterns

#### First Pattern

```java
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.tree.MethodNode;

public class MyMethodNode extends MethodNode {
  public MyMethodNode(int access, String name, String descriptor,
                      String signature, String[] exceptions,
                      MethodVisitor mv) {
    super(access, name, descriptor, signature, exceptions);
    this.mv = mv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    // put your transformation code here

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续MethodVisitor传递
    if (mv != null) {
      accept(mv);
    }
  }
}
```

#### Second Pattern

```java
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.tree.MethodNode;

public class MyMethodAdapter extends MethodVisitor {
  private final MethodVisitor next;

  public MyMethodAdapter(int api, int access, String name, String desc,
                         String signature, String[] exceptions, MethodVisitor mv) {
    super(api, new MethodNode(access, name, desc, signature, exceptions));
    this.next = mv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    MethodNode mn = (MethodNode) mv;
    // put your transformation code here

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (next != null) {
      mn.accept(next);
    }
  }
}
```

### 3.3.3 编写代码的习惯

在这里，主要是讲使用`MethodNode`进行Tree-Based Transformation过程中经常使用的编码习惯：

- 在遍历`InsnList`的过程当中，对instruction进行修改。
- 当需要添加多个instruction时，可以创建一个临时的`InsnList`，作为一个整体加入到方法当中。

Transforming a method with the tree API simply consists in modifying the fields of a `MethodNode` object, and in particular the `instructions` list.

#### modify it while iterating

Although this list can be modified in arbitrary ways, **a common pattern is to modify it while iterating over it.**

<u>Indeed, unlike with the general `ListIterator` contract, the `ListIterator` returned by an `InsnList` supports many concurrent list modifications.</u>

> `InsnList`支持并发修改，即遍历时也支持添加、删除操作，和普通的List不同

In fact, you can use the `InsnList` methods to **remove** one or more elements before and including the current one, to **remove** one or more elements after the next element(i.e. not just after the current element, but after its successor), or to **insert** one or more elements before the current one or after its successor. These changes will be reflected in the iterator, i.e. the elements inserted (resp. removed) after the next element will be seen (resp. not seen) in the iterator.

#### temporary instruction list

Another common pattern to modify an instruction list, when you need to insert several instructions after an instruction `i` inside a list, is

- to add these new instructions in a **temporary instruction list**,
- and to insert **this temporary list** inside the main one in one step

```java
InsnList il = new InsnList();
il.add(...);
...
il.add(...);
mn.instructions.insert(i, il);
```

<u>Inserting the instructions one by one is also possible but more cumbersome(麻烦), because the insertion point must be updated after each insertion.</u>

### 3.3.4 总结

本文内容总结如下：

- 第一点，对方法进行转换的时，经常使用的两种模式。
- 第二点，引入`MethodTransformer`类，它帮助进行转换具体的方法。
- 第三点，在处理`InsnList`时，经常遵循的编码习惯。

## 3.4 Tree Based Method Transformation示例

### 3.4.1 示例一：方法计时

#### 预期目标

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

我们想实现的预期目标：计算出方法的运行时间。

经过转换之后的结果，主要体现在三方面：

- 第一点，添加了一个`timer`字段，是`long`类型，访问标识为`public`和`static`。
- 第二点，在方法进入之后，`timer`字段减去一个时间戳：`timer -= System.currentTimeMillis();`。
- 第三点，在方法退出之前，`timer`字段加上一个时间戳：`timer += System.currentTimeMillis();`。

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

#### 编码实现

```java
import lsieun.asm.tree.transformer.ClassTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class ClassAddTimerNode extends ClassNode {
  public ClassAddTimerNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    ClassTransformer ct = new ClassAddTimerTransformer(null);
    ct.transform(this);

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class ClassAddTimerTransformer extends ClassTransformer {
    public ClassAddTimerTransformer(ClassTransformer ct) {
      super(ct);
    }

    @Override
    public void transform(ClassNode cn) {
      for (MethodNode mn : cn.methods) {
        if ("<init>".equals(mn.name) || "<clinit>".equals(mn.name)) {
          continue;
        }
        InsnList instructions = mn.instructions;
        // 跳过没方法体的方法（比如抽象方法）
        if (instructions.size() == 0) {
          continue;
        }
        for (AbstractInsnNode item : instructions) {
          int opcode = item.getOpcode();
          // 在方法退出之前，加上当前时间戳
          if ((opcode >= IRETURN && opcode <= RETURN) || (opcode == ATHROW)) {
            InsnList il = new InsnList();
            il.add(new FieldInsnNode(GETSTATIC, cn.name, "timer", "J"));
            il.add(new MethodInsnNode(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J"));
            il.add(new InsnNode(LADD));
            il.add(new FieldInsnNode(PUTSTATIC, cn.name, "timer", "J"));
            instructions.insertBefore(item, il);
          }
        }

        // 在方法刚进入之后，减去当前时间戳
        InsnList il = new InsnList();
        il.add(new FieldInsnNode(GETSTATIC, cn.name, "timer", "J"));
        il.add(new MethodInsnNode(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J"));
        il.add(new InsnNode(LSUB));
        il.add(new FieldInsnNode(PUTSTATIC, cn.name, "timer", "J"));
        instructions.insert(il);

        // local variables的大小，保持不变
        // mn.maxLocals = mn.maxLocals;
        // operand stack的大小，增加4个位置
        mn.maxStack += 4;
      }

      int acc = ACC_PUBLIC | ACC_STATIC;
      cn.fields.add(new FieldNode(acc, "timer", "J", null, null));
      super.transform(cn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassAddTimerNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```java
import sample.HelloWorld;

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

### 3.4.2 示例二：移除字段赋值

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

通过`javap`命令，可以查看`HelloWorld`类的instructions，该语句对应的指令组合是`aload_0 aload0 getfield putfield`：

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

```java
import lsieun.asm.tree.transformer.MethodTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.*;

import java.util.ListIterator;

import static org.objectweb.asm.Opcodes.*;

public class RemoveGetFieldPutFieldNode extends ClassNode {
  public RemoveGetFieldPutFieldNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    MethodTransformer mt = new MethodRemoveGetFieldPutFieldTransformer(null);
    for (MethodNode mn : methods) {
      if ("<init>".equals(mn.name) || "<clinit>".equals(mn.name)) {
        continue;
      }
      InsnList instructions = mn.instructions;
      if (instructions.size() == 0) {
        continue;
      }
      mt.transform(mn);
    }

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class MethodRemoveGetFieldPutFieldTransformer extends MethodTransformer {
    public MethodRemoveGetFieldPutFieldTransformer(MethodTransformer mt) {
      super(mt);
    }

    @Override
    public void transform(MethodNode mn) {
      // 首先，处理自己的代码逻辑
      InsnList instructions = mn.instructions;
      ListIterator<AbstractInsnNode> it = instructions.iterator();
      while (it.hasNext()) {
        AbstractInsnNode node1 = it.next();
        if (isALOAD0(node1)) {
          AbstractInsnNode node2 = getNext(node1);
          if (node2 != null && isALOAD0(node2)) {
            AbstractInsnNode node3 = getNext(node2);
            if (node3 != null && node3.getOpcode() == GETFIELD) {
              AbstractInsnNode node4 = getNext(node3);
              if (node4 != null && node4.getOpcode() == PUTFIELD) {
                // 只有 instance.x = instance.x 这种无意义的情况需要删除
                if (sameField(node3, node4)) {
                  while (it.next() != node4) {
                  }
                  instructions.remove(node1);
                  instructions.remove(node2);
                  instructions.remove(node3);
                  instructions.remove(node4);
                }
              }
            }
          }
        }
      }

      // 其次，调用父类的方法实现
      super.transform(mn);
    }

    private static AbstractInsnNode getNext(AbstractInsnNode insn) {
      do {
        insn = insn.getNext();
        if (insn != null && !(insn instanceof LineNumberNode)) {
          break;
        }
      } while (insn != null);
      return insn;
    }

    private static boolean isALOAD0(AbstractInsnNode insnNode) {
      return insnNode.getOpcode() == ALOAD && ((VarInsnNode) insnNode).var == 0;
    }

    private static boolean sameField(AbstractInsnNode oneInsnNode, AbstractInsnNode anotherInsnNode) {
      if (!(oneInsnNode instanceof FieldInsnNode)) return false;
      if (!(anotherInsnNode instanceof FieldInsnNode)) return false;
      FieldInsnNode fieldInsnNode1 = (FieldInsnNode) oneInsnNode;
      FieldInsnNode fieldInsnNode2 = (FieldInsnNode) anotherInsnNode;
      String name1 = fieldInsnNode1.name;
      String name2 = fieldInsnNode2.name;
      return name1.equals(name2);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new RemoveGetFieldPutFieldNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
  public int val;
...
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

### 3.4.3 示例三：优化跳转

#### 预期目标

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    System.out.println(val == 0 ? "val is 0" : "val is not 0");
  }
}
```

接着，我们查看`test`方法所包含的instructions内容：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: iload_1
       4: ifne          12
       7: ldc           #3                  // String val is 0
       9: goto          14
      12: ldc           #4                  // String val is not 0
      14: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      17: return
}
```

转换成流程图：

```pseudocode
┌───────────────────────────────────┐
│ getstatic System.out              │
│ iload_1                           │
│ ifne L0                           ├───┐
└────────────────┬──────────────────┘   │
                 │                      │
┌────────────────┴──────────────────┐   │
│ ldc "val is 0"                    │   │
│ goto L1                           ├───┼──┐
└───────────────────────────────────┘   │  │
                                        │  │
┌───────────────────────────────────┐   │  │
│ L0                                ├───┘  │
│ ldc "val is not 0"                │      │
└────────────────┬──────────────────┘      │
                 │                         │
┌────────────────┴──────────────────┐      │
│ L1                                ├──────┘
│ invokevirtual PrintStream.println │
│ return                            │
└───────────────────────────────────┘
```

在保证`test`方法正常运行的前提下，打乱内部instructions之间的顺序：

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
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "(I)V", null, null);

      Label startLabel = new Label();
      Label middleLabel = new Label();
      Label endLabel = new Label();
      Label ifLabel = new Label();
      Label elseLabel = new Label();
      Label printLabel = new Label();
      Label returnLabel = new Label();

      mv2.visitCode();
      mv2.visitJumpInsn(GOTO, middleLabel);
      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);

      mv2.visitLabel(startLabel);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ILOAD, 1);
      mv2.visitJumpInsn(GOTO, ifLabel);

      mv2.visitLabel(middleLabel);
      mv2.visitJumpInsn(GOTO, endLabel);

      mv2.visitLabel(ifLabel);
      mv2.visitJumpInsn(IFNE, elseLabel);
      mv2.visitLdcInsn("val is 0");
      mv2.visitJumpInsn(GOTO, printLabel);

      mv2.visitLabel(elseLabel);
      mv2.visitLdcInsn("val is not 0");

      mv2.visitLabel(printLabel);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      mv2.visitLabel(endLabel);
      mv2.visitJumpInsn(GOTO, startLabel);

      mv2.visitMaxs(2, 2);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

接着，我们查看`test`方法包含的instructions内容：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: goto          11
       3: return
       4: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_1
       8: goto          14
      11: goto          30
      14: ifne          22
      17: ldc           #18                 // String val is 0
      19: goto          24
      22: ldc           #20                 // String val is not 0
      24: invokevirtual #26                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: goto          3
      30: goto          4
}
```

转换成流程图：

```pseudocode
┌───────────────────────────────────┐
│ goto L0                           ├───┐
└───────────────────────────────────┘   │
                                        │
┌───────────────────────────────────┐   │
│ L1                                ├───┼────────────────────┐
│ return                            │   │                    │
└───────────────────────────────────┘   │                    │
                                        │                    │
┌───────────────────────────────────┐   │                    │
│ L2                                ├───┼────────────────────┼──┐
│ getstatic System.out              │   │                    │  │
│ iload_1                           │   │                    │  │
│ goto L3                           ├───┼─────┐              │  │
└───────────────────────────────────┘   │     │              │  │
                                        │     │              │  │
┌───────────────────────────────────┐   │     │              │  │
│ L0                                ├───┘     │              │  │
│ goto L4                           ├─────────┼──┐           │  │
└───────────────────────────────────┘         │  │           │  │
                                              │  │           │  │
┌───────────────────────────────────┐         │  │           │  │
│ L3                                ├─────────┘  │           │  │
│ ifne L5                           ├────────────┼──┐        │  │
└────────────────┬──────────────────┘            │  │        │  │
                 │                               │  │        │  │
┌────────────────┴──────────────────┐            │  │        │  │
│ ldc "val is 0"                    │            │  │        │  │
│ goto L6                           ├────────────┼──┼──┐     │  │
└───────────────────────────────────┘            │  │  │     │  │
                                                 │  │  │     │  │
┌───────────────────────────────────┐            │  │  │     │  │
│ L5                                ├────────────┼──┘  │     │  │
│ ldc "val is not 0"                │            │     │     │  │
└────────────────┬──────────────────┘            │     │     │  │
                 │                               │     │     │  │
┌────────────────┴──────────────────┐            │     │     │  │
│ L6                                ├────────────┼─────┘     │  │
│ invokevirtual PrintStream.println │            │           │  │
│ goto L1                           ├────────────┼───────────┘  │
└───────────────────────────────────┘            │              │
                                                 │              │
┌───────────────────────────────────┐            │              │
│ L4                                ├────────────┘              │
│ goto L2                           ├───────────────────────────┘
└───────────────────────────────────┘
```

我们想要实现的预期目标：优化instruction的跳转。

#### 编码实现

> 这里实现优化逻辑不是关键，主要是了解整体编写思路

```java
import lsieun.asm.tree.transformer.MethodTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class OptimizeJumpNode extends ClassNode {
  public OptimizeJumpNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    MethodTransformer mt = new MethodOptimizeJumpTransformer(null);
    for (MethodNode mn : methods) {
      if ("<init>".equals(mn.name) || "<clinit>".equals(mn.name)) {
        continue;
      }
      InsnList instructions = mn.instructions;
      if (instructions.size() == 0) {
        continue;
      }
      mt.transform(mn);
    }

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class MethodOptimizeJumpTransformer extends MethodTransformer {
    public MethodOptimizeJumpTransformer(MethodTransformer mt) {
      super(mt);
    }

    @Override
    public void transform(MethodNode mn) {
      // 首先，处理自己的代码逻辑
      InsnList instructions = mn.instructions;
      for (AbstractInsnNode insnNode : instructions) {
        if (insnNode instanceof JumpInsnNode) {
          JumpInsnNode jumpInsnNode = (JumpInsnNode) insnNode;
          LabelNode label = jumpInsnNode.label;
          AbstractInsnNode target;
          while (true) {
            target = label;
            while (target != null && target.getOpcode() < 0) {
              target = target.getNext();
            }

            if (target != null && target.getOpcode() == GOTO) {
              label = ((JumpInsnNode) target).label;
            }
            else {
              break;
            }
          }

          // update target
          jumpInsnNode.label = label;
          // if possible, replace jump with target instruction
          if (insnNode.getOpcode() == GOTO && target != null) {
            int opcode = target.getOpcode();
            if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
              instructions.set(insnNode, target.clone(null));
            }
          }
        }
      }

      // 其次，调用父类的方法实现
      super.transform(mn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new OptimizeJumpNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

验证结果，一方面要保证程序仍然能够正常运行：

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Object instance = clazz.newInstance();
    Method m = clazz.getDeclaredMethod("test", int.class);
    m.invoke(instance, 0);
    m.invoke(instance, 1);
  }
}
```

另一方面，要验证“是否对跳转进行了优化”。那么，我们通过`javap`命令来验证：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: goto          4
       3: athrow
       4: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_1
       8: goto          14
      11: nop
      12: nop
      13: athrow
      14: ifne          22
      17: ldc           #18                 // String val is 0
      19: goto          24
      22: ldc           #20                 // String val is not 0
      24: invokevirtual #26                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: return
      28: nop
      29: nop
      30: athrow
}
```

转换成流程图：

```pseudocode
┌───────────────────────────────────┐
│ goto L0                           ├───┐
└───────────────────────────────────┘   │
                                        │
┌───────────────────────────────────┐   │
│ return                            │   │
└───────────────────────────────────┘   │
                                        │
┌───────────────────────────────────┐   │
│ L0                                ├───┘
│ getstatic System.out              │
│ iload_1                           │
│ goto L1                           ├─────────┐
└───────────────────────────────────┘         │
                                              │
┌───────────────────────────────────┐         │
│ goto L0                           │         │
└───────────────────────────────────┘         │
                                              │
┌───────────────────────────────────┐         │
│ L1                                ├─────────┘
│ ifne L2                           ├───────────────┐
└────────────────┬──────────────────┘               │
                 │                                  │
┌────────────────┴──────────────────┐               │
│ ldc "val is 0"                    │               │
│ goto L3                           ├───────────────┼──┐
└───────────────────────────────────┘               │  │
                                                    │  │
┌───────────────────────────────────┐               │  │
│ L2                                ├───────────────┘  │
│ ldc "val is not 0"                │                  │
└────────────────┬──────────────────┘                  │
                 │                                     │
┌────────────────┴──────────────────┐                  │
│ L3                                ├──────────────────┘
│ invokevirtual PrintStream.println │
│ return                            │
└───────────────────────────────────┘
 
┌───────────────────────────────────┐
│ goto L0                           │
└───────────────────────────────────┘
```

### 3.4.4 总结

本文内容总结如下：

- 第一点，代码示例一（方法计时），使用了`ClassTransformer`的子类，因为既要增加字段，又要对方法进行修改。
- 第二点，代码示例二（移除字段给自身赋值），使用了`MethodTransformer`的子类，需要删除方法内的`aload_0 aload0 getfield putfield`指令组合。
- 第三点，代码示例三（优化跳转），使用了`MethodTransformer`的子类，需要对方法内的instruction替换跳转目标。

## 3.5 混合使用Core API和Tree API进行类转换

混合使用Core API和Tree API进行类转换，要分成两种情况：

- 第一种情况，先是Core API处理，然后使用Tree API处理。
- 第二种情况，先是Tree API处理，然后使用Core API处理。

先来说第一种情况，先前是Core API，接着用Tree API进行处理，对“类”和“方法”进行转换：

- ClassVisitor –> ClassNode –> ClassTransformer/MethodTransformer –> 回归Core API
- MethodVisitor –> MethodNode –> MethodTransformer –> 回归Core API

再来说第二种情况，先前是Tree API，接着用Core API进行处理，对“类”和“方法”进行转换：

- ClassNode –> ClassVisitor –> 回归Tree API
- MethodNode –> MethodVisitor –> 回归Tree API

### 3.5.1 类层面：ClassVisitor和ClassNode

假如有`HelloWorld`类，内容如下：

```java
public class HelloWorld {
  public int intValue;
}
```

我们的预期目标：使用Core API添加一个`String strValue`字段，使用Tree API添加一个`Object objValue`字段。

#### 先Core API后Tree API

思路：

```pseudocode
ClassReader --> ClassVisitor（Core API，添加strValue字段） --> ClassNode（Tree API，添加objValue字段） --> ClassWriter
```

代码片段：

```java
int api = Opcodes.ASM9;
ClassNode cn = new ClassAddFieldNode(api, cw, Opcodes.ACC_PUBLIC, "objValue", "Ljava/lang/Object;");
ClassVisitor cv = new ClassAddFieldVisitor(api, cn, Opcodes.ACC_PUBLIC, "strValue", "Ljava/lang/String;");

int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cv, parsingOptions);
```

#### 先Tree API后Core API

思路：

```pseudocode
ClassReader --> ClassNode（Tree API，添加objValue字段） --> ClassVisitor（Core API，添加strValue字段） --> ClassWriter
```

代码片段：

```java
int api = Opcodes.ASM9;
ClassVisitor cv = new ClassAddFieldVisitor(api, cw, Opcodes.ACC_PUBLIC, "strValue", "Ljava/lang/String;");
ClassNode cn = new ClassAddFieldNode(api, cv, Opcodes.ACC_PUBLIC, "objValue", "Ljava/lang/Object;");

int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
cr.accept(cn, parsingOptions);
```

### 3.5.2 方法层面：MethodVisitor和MethodNode

#### 先Core API后Tree API

思路：

```pseudocode
MethodVisitor(Core API) --> MethodNode(Tree API) --> MethodVisitor(Core API)
```

编码实现：

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.tree.*;

import static org.objectweb.asm.Opcodes.*;

public class MixCore2TreeVisitor extends ClassVisitor {
  public MixCore2TreeVisitor(int api, ClassVisitor classVisitor) {
    super(api, classVisitor);
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && !"<clinit>".equals(name)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodEnterNode(api, access, name, descriptor, signature, exceptions, mv);
      }
    }
    return mv;
  }

  private static class MethodEnterNode extends MethodNode {
    public MethodEnterNode(int api, int access, String name, String descriptor,
                           String signature, String[] exceptions,
                           MethodVisitor mv) {
      super(api, access, name, descriptor, signature, exceptions);
      this.mv = mv;
    }

    @Override
    public void visitEnd() {
      // 首先，处理自己的代码逻辑
      InsnList il = new InsnList();
      il.add(new FieldInsnNode(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;"));
      il.add(new LdcInsnNode("Method Enter"));
      il.add(new MethodInsnNode(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false));
      instructions.insert(il);

      // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
      super.visitEnd();

      // 最后，向后续MethodVisitor传递
      if (mv != null) {
        accept(mv);
      }
    }
  }
}
```

进行转换：

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
    ClassVisitor cv = new MixCore2TreeVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 先Tree API后Core API

思路：

```pseudocode
MethodNode(Tree API) --> MethodVisitor(Core API) --> MethodNode(Tree API)
```

编码实现：

```java
import lsieun.asm.template.MethodEnteringAdapter;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.MethodNode;

public class MixTree2CoreNode extends ClassNode {
  public MixTree2CoreNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    int size = methods.size();
    for (int i = 0; i < size; i++) {
      MethodNode mn = methods.get(i);
      if ("<init>".equals(mn.name) || "<clinit>".equals(mn.name)) {
        continue;
      }
      InsnList instructions = mn.instructions;
      if (instructions.size() == 0) {
        continue;
      }

      int api = Opcodes.ASM9;
      MethodNode newMethodNode = new MethodNode(api, mn.access, mn.name, mn.desc, mn.signature, mn.exceptions.toArray(new String[0]));
      MethodVisitor mv = new MethodEnteringAdapter(api, newMethodNode, mn.access, mn.name, mn.desc);
      mn.accept(mv);
      methods.set(i, newMethodNode);
    }

    // 其次，调用父类的方法实现（根据实际情况，选择保留，或删除）
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }
}
```

进行转换：

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.*;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new MixTree2CoreNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 3.5.3 总结

本文内容总结如下：

- 第一点，混合使用Core API和Tree API，分成两种情况，一种是从Core API到Tree API，另一种是从Tree API到Core API。
- 第二点，这种混合使用又常体现在两个层面：类层面和方法层面。

其中，Core API和Tree API是从ASM的角度来进行区分，而类层面和方法层面是一个`ClassFile`的结构来划分。

在混合使用Core API和Tree API的初期，可能会觉得无从下手，这个时候可以让思路慢下来：

- 首先，思考一下当前是在什么位置。
- 其次，思考一下将要去往什么位置。
- 最后，逐步补充中间需要的步骤就可以了。

# 4. Method Analysis

## 4.1 Method Analysis

### 4.1.1 Method Analysis

#### 类的主要分析对象

Java ASM是一个操作字节码（bytecode）的工具，而字节码（bytecode）的一种具体存在形式就是一个`.class`文件。现在，我们要进行分析，就可以称之为Class Analysis。

![Java ASM系列：（083）Method Analysis_java-bytecode-asm](https://s2.51cto.com/images/20210618/1624005632705532.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在类（Class）当中，主要由字段（Field）和方法（Method）组成。如果我们仔细思考一下，其实字段（Field）本身没有什么太多内容可以分析的，**主要的分析对象是方法（Method）**。因为在方法（Method）中，它包含了主要的代码处理逻辑。

因此，我们可以粗略的认为Class Analysis和Method Analysis指代同一个事物，不做严格区分。

#### 方法的主要分析对象

**在方法分析（method analysis）中有三个主要的分析对象：Instruction、Frame和Control Flow Graph。**

```java
public class HelloWorld {
  public void test(int val) {
    if (val == 0) {
      System.out.println("val is 0");
    }
    else {
      System.out.println("val is unknown");
    }
  }
}
```

![Java ASM系列：（083）Method Analysis_java-bytecode-asm_02](https://s2.51cto.com/images/20211104/1636009199533148.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

对于Frame的分析就是**data flow analysis**，对于control flow graph的分析就是**control flow analysis**。

#### DFA和CFA的区别

A **data flow analysis** consists in computing **the state of the execution frames** of a method, for each instruction of this method. This state can be represented in a more or less abstract way. For example reference values can be represented by a single value, by one value per class, by three possible values in the `{ null, not null, may be null }` set, etc.

A **control flow analysis** consists in computing **the control flow graph** of a method, and in performing analysis on this graph. The control flow graph is a graph whose nodes are instructions, and whose oriented edges connect two instructions `i → j` if `j` can be executed just after `i`.

那么，data flow analysis和control flow analysis的区别：

- <u>data flow analysis注重于“细节”，它需要明确计算出每一个instruction在local variable和operand stack当中的值。</u>
- <u>control flow analysis注重于“整体”，它不关注具体的值，而是关注于整体上指令之间前后连接或跳转的逻辑关系。</u>

接下来，举一个比喻的例子来帮助理解。在《礼记·大学》里谈到几件事情：正心、修身、齐家、治国、平天下。这几件事情，很容易让人们感受到它们处在不同的层次上，但是本质上又是贯通的。那么，大家也将data flow analysis和control flow analysis可以理解为“同一件事物在不同层次上的表达”：两者都是在方法的instructions的基础上生成的，data flow analysis注重每一条instruction对应的Frame的状态，类似于“齐家”层次，而control flow analysis注意多个instructions之间的连接/跳转关系，类似于“治国”层次。

#### 方法分析的分类

对于方法的分析，分成两种类型：

- **第一种，就是静态分析（static analysis），不需要运行程序，可以直接针对源码或字节码（bytecode）进行分析。**
- **第二种，就是动态分析（dynamic analysis），需要运行程序，是在运行过程中获取数据来进行分析。**
- **static analysis** is the analysis of computer software that is performed without actually executing programs.
- **dynamic analysis** is the analysis of computer software that is performed by executing programs on a real or virtual processor.

我们主要介绍data flow analysis和control flow analysis，但这两种analysis都是属于static analysis：

- Method Analysis
  - static analysis
    - data flow analysis
    - control flow analysis
  - dynamic analysis

另外，data flow analysis和control flow analysis有一些不适合的场景：

- 不适用于反射（reflection），例如通过反射调用某一个具体的方法。
- <u>不适用于动态绑定（dynamic binding），例如子类覆写了父类的方法，方法在执行的时候体现出子类的行为</u>。

因为静态分析，是在程序进入JVM之前发生的；动态分析，是在程序进入JVM之后发生的。上面这两种情况都是在程序运行过程中，才是它们发挥作用的时候，使用静态分析的技术不容易解决这样的问题。当使用到某个语言特性的时候，可以看看它是什么时候发挥作用的。

### 4.1.2 asm-analysis.jar

The ASM API for code analysis is in the `org.objectweb.asm.tree.analysis` package. As the package name implies, it is based on the tree API.

在上面介绍的data flow analysis和control flow analysis就是通过`asm-analysis.jar`当中定义的类来实现的。

#### 涉及到哪些类

在学习`asm-analysis.jar`时，我们的重点是理解`Analyzer`、`Frame`、`Interpreter`和`Value`这四个类之间的关系：

- `Interpreter`类，依赖于`Value`类
- `Frame`类，依赖于`Interpreter`和`Value`类
- `Analyzer`类，依赖于`Frame`、`Interpreter`和`Value`类

这四个类的依赖关系也可以表示成如下：

- `Analyzer`
  - `Frame`
  - `Interpreter`
    - `Value`

除了这四个主要的类，还有一些类是`Interpreter`和`Value`的子类：

```
┌───┬───────────────────┬─────────────┐
│ 0 │    Interpreter    │    Value    │
├───┼───────────────────┼─────────────┤
│ 1 │ BasicInterpreter  │ BasicValue  │
├───┼───────────────────┼─────────────┤
│ 2 │   BasicVerifier   │ BasicValue  │
├───┼───────────────────┼─────────────┤
│ 3 │  SimpleVerifier   │ BasicValue  │
├───┼───────────────────┼─────────────┤
│ 4 │ SourceInterpreter │ SourceValue │
└───┴───────────────────┴─────────────┘
```

> 这里不介绍Subroutine，因为这个对应jsr指令，jdk7或更早版本在遇到try-catch-finally语句生成jsr指令。但是后续有更优的方案，jsr已经是弃用指令。

#### 四个类如何协作

在`asm-analysis.jar`当中，是如何实现data flow analysis和control flow analysis的呢？

> `org.objectweb.asm.tree.analysis`包提供的是从前(之前的指令)往后(之后的指令)推测/分析的能力（forward anlysis）

Two types of **data flow analysis** can be performed:

- a **forward analysis** computes, for each instruction, the state of the execution frame after this instruction, from the state before its execution.
- a **backward analysis** computes, for each instruction, the state of the execution frame before this instruction, from the state after its execution.

In fact, the `org.objectweb.asm.tree.analysis` package provides a framework for doing **forward data flow analysis**.

In order to be able to perform various data flow analysis, with more or less precise sets of values, the **data flow analysis algorithm** is split in two parts: **one is fixed and is provided by the framework**, **the other is variable and provided by users**. More precisely:

- The overall data flow analysis algorithm, and the task of popping from the stack, and pushing back to the stack, the appropriate number of values, is implemented once and for all in the `Analyzer` and `Frame` classes.
- The task of combining values and of computing unions of value sets is performed by user defined subclasses of the `Interpreter` and `Value` abstract classes. Several predefined subclasses are provided.

`Analyzer`和`Frame`是属于“固定”的部分，而`Interpreter`和`Value`类是属于“变化”的部分。

```pseudocode
┌──────────┬─────────────┐
│          │  Analyzer   │
│  Fixed   ├─────────────┤
│          │    Frame    │
├──────────┼─────────────┤
│          │ Interpreter │
│ Variable ├─────────────┤
│          │    Value    │
└──────────┴─────────────┘
```

Although the primary goal of the framework is to perform **data flow analysis**, the `Analyzer` class can also construct the **control flow graph** of the analysed method. This can be done by overriding the `newControlFlowEdge` and `newControlFlowExceptionEdge` methods of this class, which by default do nothing. The result can be used for doing **control flow analysis**.

```pseudocode
            ┌─── data flow analysis
            │
Analyzer ───┤
            │
            └─── control flow analysis
```

#### 主要讲什么

在本章当中，我们会围绕着`asm-analysis.jar`来展开，那么我们主要讲什么内容呢？主要讲以下两行代码：

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

不管我们讲多少的内容细节，它们的最终落角点仍然是这两行代码，它是贯穿这一章内容的核心点。我们的目的就是拿到这个`frames`值，然后用它进行分析。

### 4.1.3 HelloWorldFrameTree类

在[项目](https://gitee.com/lsieun/learn-java-asm)当中，有一个`HelloWorldFrameTree`类，它的作用就是打印出Instruction和Frame的信息。

```java
public class HelloWorldFrameTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    // (2) 构建ClassNode
    ClassNode cn = new ClassNode();
    cr.accept(cn, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);

    // (3) 查看方法Instruction和Frame
    String owner = cn.name;
    List<MethodNode> methods = cn.methods;
    for (MethodNode mn : methods) {
      print(owner, mn, 3);
    }
  }
}
```

为了展示`HelloWorldFrameTree`类的功能，让我们准备一个`HelloWorld`类：

```java
public class HelloWorld {
  public void test(boolean flag, int val) {
    Object obj;
    if (flag) {
      obj = Integer.valueOf(val);
    }
    else {
      obj = Long.valueOf(val);
    }
    System.out.println(obj);
  }
}
```

#### BasicInterpreter

如果使用`BasicInterpreter`类，所有的引用类型（reference type）都使用`R`来表示：

```shell
test:(ZI)V
000:    iload_1                                 {R, I, I, .} | {}
001:    ifeq L0                                 {R, I, I, .} | {I}
002:    iload_2                                 {R, I, I, .} | {}
003:    invokestatic Integer.valueOf            {R, I, I, .} | {I}
004:    astore_3                                {R, I, I, .} | {R}
005:    goto L1                                 {R, I, I, R} | {}
006:    L0                                      {R, I, I, .} | {}
007:    iload_2                                 {R, I, I, .} | {}
008:    i2l                                     {R, I, I, .} | {I}
009:    invokestatic Long.valueOf               {R, I, I, .} | {J}
010:    astore_3                                {R, I, I, .} | {R}
011:    L1                                      {R, I, I, R} | {}
012:    getstatic System.out                    {R, I, I, R} | {}
013:    aload_3                                 {R, I, I, R} | {R}
014:    invokevirtual PrintStream.println       {R, I, I, R} | {R, R}
015:    return                                  {R, I, I, R} | {}
================================================================
```

#### SimpleVerifier

如果使用`SimpleVerifier`类，每一个不同的引用类型（reference type）都有自己的表示形式：

```shell
test:(ZI)V
000:    iload_1                                 {HelloWorld, I, I, .} | {}
001:    ifeq L0                                 {HelloWorld, I, I, .} | {I}
002:    iload_2                                 {HelloWorld, I, I, .} | {}
003:    invokestatic Integer.valueOf            {HelloWorld, I, I, .} | {I}
004:    astore_3                                {HelloWorld, I, I, .} | {Integer}
005:    goto L1                                 {HelloWorld, I, I, Integer} | {}
006:    L0                                      {HelloWorld, I, I, .} | {}
007:    iload_2                                 {HelloWorld, I, I, .} | {}
008:    i2l                                     {HelloWorld, I, I, .} | {I}
009:    invokestatic Long.valueOf               {HelloWorld, I, I, .} | {J}
010:    astore_3                                {HelloWorld, I, I, .} | {Long}
011:    L1                                      {HelloWorld, I, I, Number} | {}
012:    getstatic System.out                    {HelloWorld, I, I, Number} | {}
013:    aload_3                                 {HelloWorld, I, I, Number} | {PrintStream}
014:    invokevirtual PrintStream.println       {HelloWorld, I, I, Number} | {PrintStream, Number}
015:    return                                  {HelloWorld, I, I, Number} | {}
================================================================
```

#### SourceInterpreter

如果使用`SourceInterpreter`类，可以查看指令（Instruction）与Frame（local variable和operand stack）值的关系：

```shell
test:(ZI)V
000:    iload_1                                 {[], [], [], []} | {}
001:    ifeq L0                                 {[], [], [], []} | {[iload_1]}
002:    iload_2                                 {[], [], [], []} | {}
003:    invokestatic Integer.valueOf            {[], [], [], []} | {[iload_2]}
004:    astore_3                                {[], [], [], []} | {[invokestatic Integer.valueOf]}
005:    goto L1                                 {[], [], [], [astore_3]} | {}
006:    L0                                      {[], [], [], []} | {}
007:    iload_2                                 {[], [], [], []} | {}
008:    i2l                                     {[], [], [], []} | {[iload_2]}
009:    invokestatic Long.valueOf               {[], [], [], []} | {[i2l]}
010:    astore_3                                {[], [], [], []} | {[invokestatic Long.valueOf]}
011:    L1                                      {[], [], [], [astore_3, astore_3]} | {}
012:    getstatic System.out                    {[], [], [], [astore_3, astore_3]} | {}
013:    aload_3                                 {[], [], [], [astore_3, astore_3]} | {[getstatic System.out]}
014:    invokevirtual PrintStream.println       {[], [], [], [astore_3, astore_3]} | {[getstatic System.out], [aload_3]}
015:    return                                  {[], [], [], [astore_3, astore_3]} | {}
================================================================
```

在`011`行，有`[astore_3, astore_3]`，那么为什么有两个`astore_3`呢？

为了回答这个问题，我们也可以换一种方式来显示，查看指令索引（Instruction Index）与Frame值之间的关系：

> 下面数字[4, 10]表示当前 local variables指定位置的值可能来自指令4，也可能来之指令10

```shell
test:(ZI)V
000:    iload_1                                 {[], [], [], []} | {}
001:    ifeq L0                                 {[], [], [], []} | {[0]}
002:    iload_2                                 {[], [], [], []} | {}
003:    invokestatic Integer.valueOf            {[], [], [], []} | {[2]}
004:    astore_3                                {[], [], [], []} | {[3]}
005:    goto L1                                 {[], [], [], [4]} | {}
006:    L0                                      {[], [], [], []} | {}
007:    iload_2                                 {[], [], [], []} | {}
008:    i2l                                     {[], [], [], []} | {[7]}
009:    invokestatic Long.valueOf               {[], [], [], []} | {[8]}
010:    astore_3                                {[], [], [], []} | {[9]}
011:    L1                                      {[], [], [], [4, 10]} | {}
012:    getstatic System.out                    {[], [], [], [4, 10]} | {}
013:    aload_3                                 {[], [], [], [4, 10]} | {[12]}
014:    invokevirtual PrintStream.println       {[], [], [], [4, 10]} | {[12], [13]}
015:    return                                  {[], [], [], [4, 10]} | {}
================================================================
```

### 4.1.4 ControlFlowGraphRun类

使用`ControlFlowGraphRun`类，可以生成指令的流程图，重点是修改`display`方法的第三个参数，推荐使用`ControlFlowGraphType.STANDARD`。

```java
public class ControlFlowGraphRun {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    ClassNode cn = new ClassNode();

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）查找方法
    String methodName = "test";
    MethodNode targetNode = null;
    for (MethodNode mn : cn.methods) {
      if (mn.name.equals(methodName)) {
        targetNode = mn;
        break;
      }
    }
    if (targetNode == null) {
      System.out.println("Can not find method: " + methodName);
      return;
    }

    //（4）进行图形化显示
    System.out.println("Origin:");
    display(cn.name, targetNode, ControlFlowGraphType.NONE);
    System.out.println("Control Flow Graph:");
    display(cn.name, targetNode, ControlFlowGraphType.STANDARD);

    //（5）打印复杂度
    int complexity = CyclomaticComplexity.getCyclomaticComplexity(cn.name, targetNode);
    String line = String.format("%s:%s complexity: %d", targetNode.name, targetNode.desc, complexity);
    System.out.println(line);
  }
}
```

### 4.1.5 总结

本文内容总结如下：

- 第一点，在Method Analysis中，主要的分析对象是Instruction、Frame和Control Flow Graph。
- 第二点，在`asm-analysis.jar`当中，主要有`Analyzer`、`Frame`、`Interpreter`和`Value`四个类来进行data flow analysis和control flow analysis。
- 第三点，使用`HelloWorldFrameTree`类来查看instructions对应的不同的frames形式。
- 第四点，使用`ControlFlowGraphRun`类来查看instructions对应的control flow graph。

## 4.2 Frame/Interpreter/Value

在本文当中，我们介绍`Frame`、`Interpreter`和`Value`三个类。

![img](https://lsieun.github.io/assets/images/java/asm/analyzer-frame-interpreter.png)

### 4.2.1 Frame类

从源码的角度来讲，`Frame`是一个泛型类，它的定义如下：

```java
/**
 * The input and output stack map frames of a basic block.
 *
 * <p>Stack map frames are computed in two steps:
 *
 * <ul>
 *   <li>During the visit of each instruction in MethodWriter, the state of the frame at the end of
 *       the current basic block is updated by simulating the action of the instruction on the
 *       previous state of this so called "output frame".
 *   <li>After all instructions have been visited, a fix point algorithm is used in MethodWriter to
 *       compute the "input frame" of each basic block (i.e. the stack map frame at the beginning of
 *       the basic block). See {@link MethodWriter#computeAllFrames}.
 * </ul>
 *
 * <p>Output stack map frames are computed relatively to the input frame of the basic block, which
 * is not yet known when output frames are computed. It is therefore necessary to be able to
 * represent abstract types such as "the type at position x in the input frame locals" or "the type
 * at position x from the top of the input frame stack" or even "the type at position x in the input
 * frame, with y more (or less) array dimensions". This explains the rather complicated type format
 * used in this class, explained below.
 *
 * <p>The local variables and the operand stack of input and output frames contain values called
 * "abstract types" hereafter. An abstract type is represented with 4 fields named DIM, KIND, FLAGS
 * and VALUE, packed in a single int value for better performance and memory efficiency:
 *
 * <pre>
 *   =====================================
 *   |...DIM|KIND|.F|...............VALUE|
 *   =====================================
 * </pre>
 */
public class Frame<V extends Value> {
  //...
}
```

为了方便理解，我们可以暂时忽略掉它的泛型信息。也就是说，我们将泛型`V`直接替换成`Value`类型。

#### class info

第一个部分，`Frame`类继承自`Object`类。

```java
public class Frame {
}
```

#### fields

第二个部分，`Frame`类定义的字段有哪些。其中，

- `returnValue`和`values`

  字段分别对应于方法的“返回值”和“方法参数”。

  - **其实，`values`字段，除了包含“方法参数”，它更确切的可以理解为local variable和operand stack拼接之后的结果。**

- `numLocals`字段表示local variable的大小，而`numStack`字段表示当前operand stack上有多少个元素。

```java
public class Frame {
    private Value returnValue;
    private Value[] values;

    private int numLocals;
    private int numStack;
}
```

#### constructors

第三个部分，`Frame`类定义的构造方法有哪些。在`Frame`当中，一共定义了两个构造方法。

先来看第一个构造方法，用来创建一个全新的`Frame`对象：

- 方法参数：`int numLocals`和`int numStack`分别表示local variable和operand stack的大小。
- 方法体：
  - 初始化`values`数组的大小。
  - 记录`numLocals`字段的值。
  - 另外，注意没有给`numStack`赋值，其初始值则为`0`，表示在刚进入方法的时候，operand stack上没有任何元素。

```java
public class Frame {
  public Frame(int numLocals, int numStack) {
    this.values = new Value[numLocals + numStack];
    this.numLocals = numLocals;
    // 注意，这里并没有对numStack字段进行赋值。
  }
}
```

第二个构造方法，是对已有的frame进行复制：

- 方法参数：接收一个`Frame`类型的参数
- 方法体：
  - 调用`this(int,int)`构造方法，来初始化`values`字段和`numLocals`字段。
  - 调用`init(Frame)`方法，为`values`数组中元素赋值，并为`numStack`字段赋值。

```java
public class Frame {
  public Frame(Frame frame) {
    this(frame.numLocals, frame.values.length - frame.numLocals);
    init(frame);
  }

  public Frame init(Frame frame) {
    returnValue = frame.returnValue;
    System.arraycopy(frame.values, 0, values, 0, values.length);
    numStack = frame.numStack;
    return this;
  }
}
```

#### methods

第四个部分，`Frame`类定义的方法有哪些。

##### locals相关方法

下面这些方法是与local variable相关的方法：

- `getLocals()`方法：获取local variable的大小
- `getLocal(int)`方法：获取local variable当中某一个元素的值。
- `setLocal(int, Value)`方法：给local variable当中的某一个元素进行赋值。

```java
public class Frame {
  public int getLocals() {
    return numLocals;
  }

  public Value getLocal(int index) {
    if (index >= numLocals) {
      throw new IndexOutOfBoundsException("Trying to get an inexistant local variable " + index);
    }
    return values[index];
  }

  public void setLocal(int index, Value value) {
    if (index >= numLocals) {
      throw new IndexOutOfBoundsException("Trying to set an inexistant local variable " + index);
    }
    values[index] = value;
  }
}
```

##### stack相关方法

下面这些方法是与operand stack相关的方法：

- `getMaxStackSize()`方法：获取operand stack的总大小。
- `getStackSize()`方法：获取operand stack的当前大小。
- `clearStack()`方法：将operand stack的当前大小设置为`0`值。
- `getStack(int)`方法：获取operand stack的某一个元素。
- `setStack(int, Value)`方法：设置operand stack的某一个元素。
- `pop()`方法：将operand stack最上面的元素进行出栈。
- `push(Value)`方法：将某一个元素压进operand stack当中。

```java
public class Frame {
  public int getMaxStackSize() {
    return values.length - numLocals;
  }

  public int getStackSize() {
    return numStack;
  }

  public void clearStack() {
    numStack = 0;
  }

  public Value getStack(int index) {
    return values[numLocals + index];
  }

  public void setStack(int index, Value value) {
    values[numLocals + index] = value;
  }

  public Value pop() {
    if (numStack == 0) {
      throw new IndexOutOfBoundsException("Cannot pop operand off an empty stack.");
    }
    return values[numLocals + (--numStack)];
  }

  public void push(Value value) {
    if (numLocals + numStack >= values.length) {
      throw new IndexOutOfBoundsException("Insufficient maximum stack size.");
    }
    values[numLocals + (numStack++)] = value;
  }
}
```

##### init方法

`init`方法用来将另一个Frame里的值复制到当前Frame当中。

```java
public class Frame {
  public Frame init(final Frame frame) {
    returnValue = frame.returnValue;
    System.arraycopy(frame.values, 0, values, 0, values.length);
    numStack = frame.numStack;
    return this;
  }
}
```

##### execute方法

下面的`execute(AbstractInsnNode, Interpreter)`方法，是<u>模拟某一条instruction对local variable和operand stack的影响</u>。针对某一个具体的instruction，它具体有什么样的操作，可以参考[Chapter 6. The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)。

同时，我们也要注意到`execute`方法也会用到`Interpreter`类，那么`Interpreter`类起到一个什么样的作用呢？**`Interpreter`类，就是使用当前的指令（`insn`）和相关参数（`value1`、`value2`、`value3`、`value4`）计算出一个新的值**。

```java
public class Frame {
  public void execute(AbstractInsnNode insn, Interpreter interpreter) throws AnalyzerException {
    Value value1;
    Value value2;
    Value value3;
    Value value4;
    int var;

    switch (insn.getOpcode()) {
      case Opcodes.NOP:
        break;
      case Opcodes.ACONST_NULL:
      case Opcodes.ICONST_M1:
      case Opcodes.ICONST_0:
      case Opcodes.ICONST_1:
      case Opcodes.ICONST_2:
      case Opcodes.ICONST_3:
      case Opcodes.ICONST_4:
      case Opcodes.ICONST_5:
      case Opcodes.LCONST_0:
      case Opcodes.LCONST_1:
      case Opcodes.FCONST_0:
      case Opcodes.FCONST_1:
      case Opcodes.FCONST_2:
      case Opcodes.DCONST_0:
      case Opcodes.DCONST_1:
      case Opcodes.BIPUSH:
      case Opcodes.SIPUSH:
      case Opcodes.LDC:
        // operand stack入栈：1
        // 首先，由Interpreter解析出一个Value值
        // 其次，将该Value值入栈到operand stack上
        push(interpreter.newOperation(insn));
        break;
      case Opcodes.ILOAD:
      case Opcodes.LLOAD:
      case Opcodes.FLOAD:
      case Opcodes.DLOAD:
      case Opcodes.ALOAD:
        // operand stack入栈：1
        // 首先，从local variable当中取出一个Value值
        // 其次，由Interpreter将旧的Value值解析成一个新的Value值
        // 最后，将新Value值入栈到operand stack上
        push(interpreter.copyOperation(insn, getLocal(((VarInsnNode) insn).var)));
        break;
      case Opcodes.ISTORE:
      case Opcodes.LSTORE:
      case Opcodes.FSTORE:
      case Opcodes.DSTORE:
      case Opcodes.ASTORE:
        // operand stack出栈：1
        // 首先，从operand stack当中出栈一个Value值
        // 其次，由Interpreter将旧的Value值解析成一个新的Value值
        // 最后，将新Value值存储到local variable上
        value1 = interpreter.copyOperation(insn, pop());
        var = ((VarInsnNode) insn).var;
        setLocal(var, value1);
        if (value1.getSize() == 2) {
          setLocal(var + 1, interpreter.newEmptyValue(var + 1));
        }
        if (var > 0) {
          Value local = getLocal(var - 1);
          if (local != null && local.getSize() == 2) {
            setLocal(var - 1, interpreter.newEmptyValue(var - 1));
          }
        }
        break;
      case Opcodes.IASTORE:
      case Opcodes.LASTORE:
      case Opcodes.FASTORE:
      case Opcodes.DASTORE:
      case Opcodes.AASTORE:
      case Opcodes.BASTORE:
      case Opcodes.CASTORE:
      case Opcodes.SASTORE:
        // operand stack出栈：3
        // 首先，从operand stack当中出栈三个Value值
        // 其次，由Interpreter将三个旧的Value值解析成一个新的Value值
        // 最后，扔掉这个新生成的Value值，因为用不到它
        value3 = pop();
        value2 = pop();
        value1 = pop();
        interpreter.ternaryOperation(insn, value1, value2, value3);
        break;
      case Opcodes.POP:
        // operand stack出栈：1
        // 首先，从operand stack当中出栈一个Value值
        // 其次，扔掉这个Value值，因为用不到它
        if (pop().getSize() == 2) {
          throw new AnalyzerException(insn, "Illegal use of POP");
        }
        break;
      case Opcodes.POP2:
        // operand stack出栈：1
        // 首先，从operand stack当中出栈一个Value值
        // 其次，扔掉这个Value值，因为用不到它
        if (pop().getSize() == 1 && pop().getSize() != 1) {
          throw new AnalyzerException(insn, "Illegal use of POP2");
        }
        break;
      case Opcodes.DUP:
        // operand stack出栈：1，入栈：2
        // 首先，从operand stack当中出栈一个Value值
        // 其次，将该Value值再入栈到operand stack上
        // 接着，由Interpreter将旧的Value值解析成一个新的Value值
        // 最后，将这个新的Value值入栈到operand stack上
        value1 = pop();
        if (value1.getSize() != 1) {
          throw new AnalyzerException(insn, "Illegal use of DUP");
        }
        push(value1);
        push(interpreter.copyOperation(insn, value1));
        break;
      case Opcodes.DUP_X1:
        // operand stack出栈：2，入栈：3
        value1 = pop();
        value2 = pop();
        if (value1.getSize() != 1 || value2.getSize() != 1) {
          throw new AnalyzerException(insn, "Illegal use of DUP_X1");
        }
        push(interpreter.copyOperation(insn, value1));
        push(value2);
        push(value1);
        break;
      case Opcodes.DUP_X2:
        value1 = pop();
        if (value1.getSize() == 1 && executeDupX2(insn, value1, interpreter)) {
          break;
        }
        throw new AnalyzerException(insn, "Illegal use of DUP_X2");
      case Opcodes.DUP2:
        value1 = pop();
        if (value1.getSize() == 1) {
          value2 = pop();
          if (value2.getSize() == 1) {
            push(value2);
            push(value1);
            push(interpreter.copyOperation(insn, value2));
            push(interpreter.copyOperation(insn, value1));
            break;
          }
        } else {
          push(value1);
          push(interpreter.copyOperation(insn, value1));
          break;
        }
        throw new AnalyzerException(insn, "Illegal use of DUP2");
      case Opcodes.DUP2_X1:
        value1 = pop();
        if (value1.getSize() == 1) {
          value2 = pop();
          if (value2.getSize() == 1) {
            value3 = pop();
            if (value3.getSize() == 1) {
              push(interpreter.copyOperation(insn, value2));
              push(interpreter.copyOperation(insn, value1));
              push(value3);
              push(value2);
              push(value1);
              break;
            }
          }
        } else {
          value2 = pop();
          if (value2.getSize() == 1) {
            push(interpreter.copyOperation(insn, value1));
            push(value2);
            push(value1);
            break;
          }
        }
        throw new AnalyzerException(insn, "Illegal use of DUP2_X1");
      case Opcodes.DUP2_X2:
        value1 = pop();
        if (value1.getSize() == 1) {
          value2 = pop();
          if (value2.getSize() == 1) {
            value3 = pop();
            if (value3.getSize() == 1) {
              value4 = pop();
              if (value4.getSize() == 1) {
                push(interpreter.copyOperation(insn, value2));
                push(interpreter.copyOperation(insn, value1));
                push(value4);
                push(value3);
                push(value2);
                push(value1);
                break;
              }
            } else {
              push(interpreter.copyOperation(insn, value2));
              push(interpreter.copyOperation(insn, value1));
              push(value3);
              push(value2);
              push(value1);
              break;
            }
          }
        } else if (executeDupX2(insn, value1, interpreter)) {
          break;
        }
        throw new AnalyzerException(insn, "Illegal use of DUP2_X2");
      case Opcodes.SWAP:
        // operand stack出栈：2，入栈：2
        value2 = pop();
        value1 = pop();
        if (value1.getSize() != 1 || value2.getSize() != 1) {
          throw new AnalyzerException(insn, "Illegal use of SWAP");
        }
        push(interpreter.copyOperation(insn, value2));
        push(interpreter.copyOperation(insn, value1));
        break;
      case Opcodes.IALOAD:
      case Opcodes.LALOAD:
      case Opcodes.FALOAD:
      case Opcodes.DALOAD:
      case Opcodes.AALOAD:
      case Opcodes.BALOAD:
      case Opcodes.CALOAD:
      case Opcodes.SALOAD:
      case Opcodes.IADD:
      case Opcodes.LADD:
      case Opcodes.FADD:
      case Opcodes.DADD:
      case Opcodes.ISUB:
      case Opcodes.LSUB:
      case Opcodes.FSUB:
      case Opcodes.DSUB:
      case Opcodes.IMUL:
      case Opcodes.LMUL:
      case Opcodes.FMUL:
      case Opcodes.DMUL:
      case Opcodes.IDIV:
      case Opcodes.LDIV:
      case Opcodes.FDIV:
      case Opcodes.DDIV:
      case Opcodes.IREM:
      case Opcodes.LREM:
      case Opcodes.FREM:
      case Opcodes.DREM:
      case Opcodes.ISHL:
      case Opcodes.LSHL:
      case Opcodes.ISHR:
      case Opcodes.LSHR:
      case Opcodes.IUSHR:
      case Opcodes.LUSHR:
      case Opcodes.IAND:
      case Opcodes.LAND:
      case Opcodes.IOR:
      case Opcodes.LOR:
      case Opcodes.IXOR:
      case Opcodes.LXOR:
      case Opcodes.LCMP:
      case Opcodes.FCMPL:
      case Opcodes.FCMPG:
      case Opcodes.DCMPL:
      case Opcodes.DCMPG:
        // operand stack出栈：2，入栈：1
        value2 = pop();
        value1 = pop();
        push(interpreter.binaryOperation(insn, value1, value2));
        break;
      case Opcodes.INEG:
      case Opcodes.LNEG:
      case Opcodes.FNEG:
      case Opcodes.DNEG:
        // operand stack出栈：1，入栈：1
        push(interpreter.unaryOperation(insn, pop()));
        break;
      case Opcodes.IINC:
        var = ((IincInsnNode) insn).var;
        setLocal(var, interpreter.unaryOperation(insn, getLocal(var)));
        break;
      case Opcodes.I2L:
      case Opcodes.I2F:
      case Opcodes.I2D:
      case Opcodes.L2I:
      case Opcodes.L2F:
      case Opcodes.L2D:
      case Opcodes.F2I:
      case Opcodes.F2L:
      case Opcodes.F2D:
      case Opcodes.D2I:
      case Opcodes.D2L:
      case Opcodes.D2F:
      case Opcodes.I2B:
      case Opcodes.I2C:
      case Opcodes.I2S:
        // operand stack出栈：1，入栈：1
        push(interpreter.unaryOperation(insn, pop()));
        break;
      case Opcodes.IFEQ:
      case Opcodes.IFNE:
      case Opcodes.IFLT:
      case Opcodes.IFGE:
      case Opcodes.IFGT:
      case Opcodes.IFLE:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      case Opcodes.IF_ICMPEQ:
      case Opcodes.IF_ICMPNE:
      case Opcodes.IF_ICMPLT:
      case Opcodes.IF_ICMPGE:
      case Opcodes.IF_ICMPGT:
      case Opcodes.IF_ICMPLE:
      case Opcodes.IF_ACMPEQ:
      case Opcodes.IF_ACMPNE:
      case Opcodes.PUTFIELD:
        // operand stack出栈：2
        value2 = pop();
        value1 = pop();
        interpreter.binaryOperation(insn, value1, value2);
        break;
      case Opcodes.GOTO:
        break;
      case Opcodes.JSR:
        push(interpreter.newOperation(insn));
        break;
      case Opcodes.RET:
        break;
      case Opcodes.TABLESWITCH:
      case Opcodes.LOOKUPSWITCH:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      case Opcodes.IRETURN:
      case Opcodes.LRETURN:
      case Opcodes.FRETURN:
      case Opcodes.DRETURN:
      case Opcodes.ARETURN:
        value1 = pop();
        interpreter.unaryOperation(insn, value1);
        interpreter.returnOperation(insn, value1, returnValue);
        break;
      case Opcodes.RETURN:
        if (returnValue != null) {
          throw new AnalyzerException(insn, "Incompatible return type");
        }
        break;
      case Opcodes.GETSTATIC:
        // operand stack入栈：1
        push(interpreter.newOperation(insn));
        break;
      case Opcodes.PUTSTATIC:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      case Opcodes.GETFIELD:
        // operand stack出栈：1，入栈：1
        push(interpreter.unaryOperation(insn, pop()));
        break;
      case Opcodes.INVOKEVIRTUAL:
      case Opcodes.INVOKESPECIAL:
      case Opcodes.INVOKESTATIC:
      case Opcodes.INVOKEINTERFACE:
        executeInvokeInsn(insn, ((MethodInsnNode) insn).desc, interpreter);
        break;
      case Opcodes.INVOKEDYNAMIC:
        executeInvokeInsn(insn, ((InvokeDynamicInsnNode) insn).desc, interpreter);
        break;
      case Opcodes.NEW:
        // operand stack入栈：1
        push(interpreter.newOperation(insn));
        break;
      case Opcodes.NEWARRAY:
      case Opcodes.ANEWARRAY:
      case Opcodes.ARRAYLENGTH:
        // operand stack出栈：1，入栈：1
        push(interpreter.unaryOperation(insn, pop()));
        break;
      case Opcodes.ATHROW:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      case Opcodes.CHECKCAST:
      case Opcodes.INSTANCEOF:
        // operand stack出栈：1，入栈：1
        push(interpreter.unaryOperation(insn, pop()));
        break;
      case Opcodes.MONITORENTER:
      case Opcodes.MONITOREXIT:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      case Opcodes.MULTIANEWARRAY:
        // operand stack出栈：n，入栈：1
        List<Value> valueList = new ArrayList<>();
        for (int i = ((MultiANewArrayInsnNode) insn).dims; i > 0; --i) {
          valueList.add(0, pop());
        }
        push(interpreter.naryOperation(insn, valueList));
        break;
      case Opcodes.IFNULL:
      case Opcodes.IFNONNULL:
        // operand stack出栈：1
        interpreter.unaryOperation(insn, pop());
        break;
      default:
        throw new AnalyzerException(insn, "Illegal opcode " + insn.getOpcode());
    }
  }

  private void executeInvokeInsn(AbstractInsnNode insn, String methodDescriptor, Interpreter interpreter) {
    ArrayList<Value> valueList = new ArrayList<>();
    // 添加方法的参数
    for (int i = Type.getArgumentTypes(methodDescriptor).length; i > 0; --i) {
      valueList.add(0, pop());
    }

    // 考虑是否添加this变量
    if (insn.getOpcode() != Opcodes.INVOKESTATIC && insn.getOpcode() != Opcodes.INVOKEDYNAMIC) {
      valueList.add(0, pop());
    }

    if (Type.getReturnType(methodDescriptor) == Type.VOID_TYPE) {
      // 返回void类型
      interpreter.naryOperation(insn, valueList);
    } else {
      // 返回值不为void类型，需要将返回值加载到operand stack上
      push(interpreter.naryOperation(insn, valueList));
    }
  }
}
```

##### initJumpTarget方法

`initJumpTarget`方法从两个方面来把握：它的作用是什么和它的调用时机是什么。

- 作用：提供一个修改当前Frame中值的机会，这样就可以对Frame的内容进行精细的管理。Overriding this method and changing the frame values allows implementing branch-sensitive analyses.
- 调用时机：遇到当前要执行的指令
  - 首先，调用`init`方法，复制当前指令执行之前的状态
  - 其次，调用`execute`方法，对当前的指令进行执行，Frame获得一个新的状态
  - 再接着，判断一下当前指令是否是一个跳转指令（`if`、`switch`），那么就会执行`initJumpTarget`方法，它允许我们对于当前Frame再进一步调整
  - 最后，调用`merge`方法，将当前的Frame“存储”起来。注意，这里不是直接存储，而是进行merge操作，就类似于将两个文件夹的内容放到一个文件夹里面。

```java
public class Frame {
  public void initJumpTarget(final int opcode, final LabelNode target) {
    // Does nothing by default.
  }
}
```

##### merge方法

下面的`merge`方法是将两个`Frame`对象的内容进行合并（merge），它会进一步的调用`Interpreter.merge()`方法。

```java
public class Frame {
  public boolean merge(Frame frame, final Interpreter interpreter) throws AnalyzerException {
    if (numStack != frame.numStack) {
      throw new AnalyzerException(null, "Incompatible stack heights");
    }
    boolean changed = false;
    for (int i = 0; i < numLocals + numStack; ++i) {
      Value v = interpreter.merge(values[i], frame.values[i]);
      if (!v.equals(values[i])) {
        values[i] = v;
        changed = true;
      }
    }
    return changed;
  }
}
```

还有另外一个`merge`方法，我们故意将它省略了，因为它与`jsr`（subroutine）相关，而`jsr`指令不推荐使用了。

### 4.2.2 Interpreter类

与`Frame`类似，`Interpreter`也是一个泛型类，我们也可以将泛型`V`替换成`Value`值，以简化思考。

#### class info

第一个部分，`Interpreter`类是一个抽象类，它继承自`Object`类。

```java
/**
 * A semantic bytecode interpreter. More precisely, this interpreter only manages the computation of
 * values from other values: it does not manage the transfer of values to or from the stack, and to
 * or from the local variables. This separation allows a generic bytecode {@link Analyzer} to work
 * with various semantic interpreters, without needing to duplicate the code to simulate the
 * transfer of values.
 *
 * @param <V> type of the Value used for the analysis.
 * @author Eric Bruneton
 */
public abstract class Interpreter {
}
```

#### fields

第二个部分，`Interpreter`类定义的字段有哪些。

```java
public abstract class Interpreter {
  protected final int api;
}
```

#### constructors

第三个部分，`Interpreter`类定义的构造方法有哪些。

```java
public abstract class Interpreter {
  protected Interpreter(int api) {
    this.api = api;
  }
}
```

#### methods

第四个部分，`Interpreter`类定义的方法有哪些。

##### newValue相关方法

如果我们仔细观察下面几个方法，会发现它们都指向同一个方法，即`Value newValue(Type)`方法：该方法是<u>将ASM当中的`Type`类型向Frame当中的`Value`类型进行映射</u>。

```java
public abstract class Interpreter {
  public Value newParameterValue(boolean isInstanceMethod, int local, Type type) {
    return newValue(type);
  }

  public Value newReturnTypeValue(Type type) {
    return newValue(type);
  }

  public Value newEmptyValue(int local) {
    return newValue(null);
  }

  public Value newExceptionValue(
    TryCatchBlockNode tryCatchBlockNode,
    Frame handlerFrame,
    Type exceptionType) {
    return newValue(exceptionType);
  }

  public abstract Value newValue(Type type);    
}
```

具体来说，这几个方法的用途：

- `newParameterValue()`方法：在方法头中，对“方法接收的参数类型”进行转换。
- `newReturnTypeValue()`方法：在方法头中，对“方法的返回值类型”进行转换。
- `newEmptyValue()`方法：将local variable当中某个位置设置成空值。
- `newExceptionValue()`方法：在方法体中，执行的时候，可能会出现异常，这里就是对“异常的类型”进行转换。

在local variable当中，为什么会有空值的出现呢？有两种情况：

- 第一种情况，假如local variable的总大小是10，而方法的接收参数只占前3个位置，那么剩下的7个位置的初始值就是空值。
- 第二种情况，在local variable当中，`long`和`double`类型占用2个位置，在进行模拟的时候，第1个位置就记录了`long`和`double`类型，第2个位置就用空值来表示。

接着，我们解释一个这样的问题：为什么`Value newValue(Type type)`方法要将`Type`类型转换成`Value`类型？

| 领域 | ClassFile                | ASM模拟类型或描述符                                | ASM模拟Stack Frame(local variable+operand stack) |
| ---- | ------------------------ | -------------------------------------------------- | ------------------------------------------------ |
| 类型 | Internal Name/Descriptor | Type                                               | Value                                            |
| 示例 | `Ljava/lang/String;`     | `Type t = Type.getObjectType("java/lang/String");` | `BasicValue val = BasicValue.INT_VALUE;`         |

我们使用ASM编写代码，遇到的类型就是`Type`类型，接下来要做的就是模拟instruction执行过程中对local variable和operand stack的影响，因此需要将`Type`类型转换成`Value`类型。

举个例子，现在你持有中国的货币，接下来你想投资美国的市场，那么需要先将中国的货币兑换成美国的货币，然后才能去投资。

另外，我们要注意`Type`和`Value`类型的实例分别以`_TYPE`和`_VALUE`为结尾：

- `Type`类型的实例：`Type.VOID_TYPE`、`Type.BOOLEAN_TYPE`、`Type.CHAR_TYPE`、`Type.INT_TYPE`、`Type.FLOAT_TYPE`、`Type.LONG_TYPE`、`Type.DOUBLE_TYPE`等。
- `Value`类型的实例：`BasicValue.UNINITIALIZED_VALUE`、`BasicValue.INT_VALUE`、`BasicValue.FLOAT_VALUE`、`BasicValue.LONG_VALUE`、`BasicValue.DOUBLE_VALUE`和`BasicValue.REFERENCE_VALUE`等。

除了以上的几个方法用到了`newValue`方法，还有哪些地方也会用到`newValue`方法：

- 在`BasicInterpreter`类当中：
  - `ACONST_NULL`指令：`newValue(NULL_TYPE);`
  - `CHECKCAST`指令：`newValue(Type.getObjectType(((TypeInsnNode) insn).desc));`
  - `LDC`指令：`newValue(Type.getObjectType("java/lang/String"))`
  - `GETFIELD`指令：`newValue(Type.getType(((FieldInsnNode) insn).desc))`
  - `GETSTATIC`指令：`newValue(Type.getType(((FieldInsnNode) insn).desc))`
  - `NEW`指令：`newValue(Type.getObjectType(((TypeInsnNode) insn).desc))`
  - `NEWARRAY`指令：`newValue(Type.getType("[Z"))`、`newValue(Type.getType("[C"))`等
  - `ANEWARRAY`指令：`newValue(Type.getType("[" + Type.getObjectType(((TypeInsnNode) insn).desc)))`
  - `MULTIANEWARRAY`指令：`newValue(Type.getType(((MultiANewArrayInsnNode) insn).desc))`
  - `INVOKEDYNAMIC`指令：`newValue(Type.getReturnType(((InvokeDynamicInsnNode) insn).desc))`
  - `INVOKEVIRTUAL`、`INVOKESPECIAL`、`INVOKESTATIC`、`INVOKEINTERFACE`指令：`newValue(Type.getReturnType(((InvokeDynamicInsnNode) insn).desc))`
- …（省略）

##### opcode相关方法

下面7个方法，就是结合指令（`AbstractInsnNode`类型）和多个元素值（`Value`类型）来计算出一个新的元素值（`Value`类型）。

```java
public abstract class Interpreter {
  public abstract Value newOperation(AbstractInsnNode insn) throws AnalyzerException;

  public abstract Value copyOperation(AbstractInsnNode insn, Value value) throws AnalyzerException;

  public abstract Value unaryOperation(AbstractInsnNode insn, Value value) throws AnalyzerException;

  public abstract Value binaryOperation(AbstractInsnNode insn, Value value1, Value value2) throws AnalyzerException;

  public abstract Value ternaryOperation(AbstractInsnNode insn, Value value1, Value value2, Value value3) throws AnalyzerException;

  public abstract Value naryOperation(AbstractInsnNode insn, List<Value> values) throws AnalyzerException;

  public abstract void returnOperation(AbstractInsnNode insn, Value value, Value expected) throws AnalyzerException;
}
```

具体来说，这几个方法的作用：

- `newOperation()`方法：处理opcode和0个元素值（`Value`类型）之间的关系，这是一步从`0`到`1`的操作。
- `copyOperation()`方法：处理opcode和1个元素值（`Value`类型）之间的关系，这是一步从`1`到`1`的操作。copy是“复制”，一个`int`类型的值，复制一份之后，仍然是`int`类型的值。
- `unaryOperation()`方法：处理opcode和1个元素值（`Value`类型）之间的关系，这是一步从`1`到`1`的操作。一个`int`类型的值，经过`i2f`指令运算，就会变成`float`类型的值。
- `binaryOperation()`方法：处理opcode和2个元素值（`Value`类型）之间的关系，这是一步从`2`到`1`的操作。
- `ternaryOperation()`方法：处理opcode和3个元素值（`Value`类型）之间的关系，这是一步从`3`到`1`的操作。
- `naryOperation()`方法：处理opcode和n个元素值（`Value`类型）之间的关系，这是一步从`n`到`1`的操作。
- `returnOperation()`方法：处理return的期望类型和实际类型之间的关系，这是一步从`2`到`0`的操作。

为什么有这些方法呢？因为opcode有200个左右，如果一个类里面定义200个方法，记忆起来就不太方便了。那么，按照“消耗”的元素值（`Value`类型）的数量多少，分成7个不同的方法，这就大大简化了方法的整体数量。

**注意：在这里，我们用操作数（operand）和元素（element）表示不同的概念。**

- **`AbstractInsnNode`类型，是指instruction，包含opcode和operand；这里的operand，体现为具体`AbstractInsnNode`类型的字段值。**
- **元素值（`Value`类型），是指local variable和operand stack上某一个元素（element）的值。**

```pseudocode
instruction = opcode + operand
```

##### merge方法

这里是`merge`方法，它的作用是将两个`Value`值合并为一个新的`Value`值。

```java
public abstract class Interpreter {
  public abstract Value merge(Value value1, Value value2);
}
```

那么，为什么需要将两个`Value`值合并为一个新的`Value`值呢？我们举个例子：

```java
public class HelloWorld {
  public void test(int a, int b) {
    Object obj;
    int c = a + b;

    if (c > 10) {
      obj = Integer.valueOf(20);
      System.out.println(obj);
    }
    else {
      obj = Float.valueOf(5);
      System.out.println(obj);
    }
    int hashCode = obj.hashCode();
    System.out.println(hashCode);
  }
}
```

我们可以查看instruction对应的local variable和operand stack的变化：

```shell
test:(II)V
                               // {this, int, int} | {}
0000: iload_1                  // {this, int, int} | {int}
0001: iload_2                  // {this, int, int} | {int, int}
0002: iadd                     // {this, int, int} | {int}
0003: istore          4        // {this, int, int, top, int} | {}
0005: iload           4        // {this, int, int, top, int} | {int}
0007: bipush          10       // {this, int, int, top, int} | {int, int}
0009: if_icmple       19       // {this, int, int, top, int} | {}
0012: bipush          20       // {this, int, int, top, int} | {int}
0014: invokestatic    #2       // {this, int, int, top, int} | {Integer}
0017: astore_3                 // {this, int, int, Integer, int} | {}
0018: getstatic       #3       // {this, int, int, Integer, int} | {PrintStream}
0021: aload_3                  // {this, int, int, Integer, int} | {PrintStream, Integer}
0022: invokevirtual   #4       // {this, int, int, Integer, int} | {}
0025: goto            16       // {} | {}
                               // {this, int, int, top, int} | {}
0028: ldc             #5       // {this, int, int, top, int} | {float}
0030: invokestatic    #6       // {this, int, int, top, int} | {Float}
0033: astore_3                 // {this, int, int, Float, int} | {}
0034: getstatic       #3       // {this, int, int, Float, int} | {PrintStream}
0037: aload_3                  // {this, int, int, Float, int} | {PrintStream, Float}
0038: invokevirtual   #4       // {this, int, int, Float, int} | {}
                               // {this, int, int, Object, int} | {}
0041: aload_3                  // {this, int, int, Object, int} | {Object}
0042: invokevirtual   #7       // {this, int, int, Object, int} | {int}
0045: istore          5        // {this, int, int, Object, int, int} | {}
0047: getstatic       #3       // {this, int, int, Object, int, int} | {PrintStream}
0050: iload           5        // {this, int, int, Object, int, int} | {PrintStream, int}
0052: invokevirtual   #8       // {this, int, int, Object, int, int} | {}
0055: return                   // {} | {}
```

在local variable当中，在`(offset=17, local=3)`的位置是`Integer`类型，在`(offset=33, local=3)`的位置是`Float`类型，它们merge之后是`Object`类型。

### 4.2.3 Value类

现在我们来看`Value`，它是一个接口，定义了一个`getSize()`方法。

```java
public interface Value {
    int getSize();
}
```

### 4.2.4 总结

本文内容总结如下：

- 第一点，`Frame`类的从字段和方法两方面来把握：
  - `Frame`类的字段，主要是`values`字段，用来记录local variable和operand stack的状态，其中存储的数据是`Value`类型。
  - `Frame`类的方法，`init`、`execute`、`initJumpTarget`和`merge`方法，理解这几个方法，就能理解随着指令（instruction）的执行Frame的状态是如何变化的。
- 第二点，`Interpreter`类的作用就像一个`Value`的工厂，要么从无到有的创建一个新的`Value`对象出来，要么多个`Value`对象合并成一个新的`Value`对象。
- 第三点，`Value`是一个接口，它非常简单。

## 4.3 Analyzer

![Java ASM系列：（085）Analyzer_java-bytecode-asm](https://s2.51cto.com/images/20211107/1636265255925384.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

### 4.3.1 Analyzer类

#### class info

第一个部分，`Analyzer`类实现了`Opcodes`接口。

```java
/**
 * A semantic bytecode analyzer. <i>This class does not fully check that JSR and RET instructions
 * are valid.</i>
 *
 * @param <V> type of the Value used for the analysis.
 * @author Eric Bruneton
 */
public class Analyzer<V extends Value> implements Opcodes {
}
```

#### fields

第二个部分，`Analyzer`类定义的字段有哪些。

| 领域 | ClassFile                         | ASM                                         | JVM Frame(local variable + operand stack) |
| ---- | --------------------------------- | ------------------------------------------- | ----------------------------------------- |
| 元素 | `Code = code[] + exception_table` | `MethodNode = InsnList + TryCatchBlockNode` | Interpreter + Frame                       |

这字段分成三组：

- 第一组，`insnList`表示指令集合，`insnListSize`表示指令集的大小，`handlers`表示异常处理的机制，这三个字段都是属于“方法”的内容。
- 第二组，`interpreter`、`frames`和泛型`V`则是用来模拟“local variable和operand stack”的内容。
- 第三组，`inInstructionsToProcess`、`instructionsToProcess`和`numInstructionsToProcess`三个字段都是“临时的数据容器”，用来记录过程中的内部状态。

**粗略地理解：第一组字段，可以理解为“数据输入”，第三组字段，可以理解为“中间状态”，第二组字段，可以理解为“输出结果”。**

```java
public class Analyzer<V extends Value> implements Opcodes {
  // 第一组，指令和异常处理，是属于“方法”的内容。
  private InsnList insnList;
  private int insnListSize;
  private List<TryCatchBlockNode>[] handlers;

  // 第二组，Interpreter和Frame，是属于“local variable和operand stack”的内容。
  private final Interpreter<V> interpreter;
  private Frame<V>[] frames;

  // 第三组，在运行过程中，记录临时状态的变化量
  private boolean[] inInstructionsToProcess;
  private int[] instructionsToProcess;
  private int numInstructionsToProcess;
}
```

#### constructors

第三个部分，`Analyzer`类定义的构造方法有哪些。

```java
public class Analyzer<V extends Value> implements Opcodes {
  public Analyzer(Interpreter<V> interpreter) {
    this.interpreter = interpreter;
  }
}
```

#### methods

第四个部分，`Analyzer`类定义的方法有哪些。

##### getter方法

```java
public class Analyzer<V extends Value> implements Opcodes {
  public Frame<V>[] getFrames() {
    return frames;
  }

  public List<TryCatchBlockNode> getHandlers(int insnIndex) {
    return handlers[insnIndex];
  }
}
```

##### analyze方法

在`Analyzer`类当中，最重要的方法就是`analyze`方法。在`analyze`方法中，会进一步调用其它的方法。

```java
public class Analyzer<V extends Value> implements Opcodes {
  public Frame<V>[] analyze(String owner, MethodNode method) throws AnalyzerException {
    // 如果方法是abstract或native方法，则直接返回
    if ((method.access & (ACC_ABSTRACT | ACC_NATIVE)) != 0) {
      frames = (Frame<V>[]) new Frame<?>[0];
      return frames;
    }

    // 为各个字段进行赋值
    insnList = method.instructions;
    insnListSize = insnList.size();
    handlers = (List<TryCatchBlockNode>[]) new List<?>[insnListSize];
    frames = (Frame<V>[]) new Frame<?>[insnListSize];
    inInstructionsToProcess = new boolean[insnListSize];
    instructionsToProcess = new int[insnListSize];
    numInstructionsToProcess = 0;
    // ...

    // 方法头：计算方法的初始Frame
    Frame<V> currentFrame = computeInitialFrame(owner, method);
    merge(0, currentFrame, null);
    init(owner, method);

    // 方法体：通过循环对每一条指令进行处理
    while (numInstructionsToProcess > 0) {
      // 计算每一条指令对应的Frame的状态
      // ...
    }

    return frames;
  }
}
```

##### computeInitialFrame方法

`computeInitialFrame`方法根据方法的访问标识（access flag）和方法的描述符（method descriptor）来生成初始的Frame。

```java
public class Analyzer<V extends Value> implements Opcodes {
  private Frame<V> computeInitialFrame(final String owner, final MethodNode method) {
    // 准备工作：创建一个新的Frame，用来存储下面的数据
    Frame<V> frame = newFrame(method.maxLocals, method.maxStack);
    int currentLocal = 0;

    // 首先，考虑是否需要存储this
    boolean isInstanceMethod = (method.access & ACC_STATIC) == 0;
    if (isInstanceMethod) {
      Type ownerType = Type.getObjectType(owner);
      frame.setLocal(currentLocal, interpreter.newParameterValue(isInstanceMethod, currentLocal, ownerType));
      currentLocal++;
    }

    // 接着，存储方法接收的参数
    Type[] argumentTypes = Type.getArgumentTypes(method.desc);
    for (Type argumentType : argumentTypes) {
      frame.setLocal(currentLocal, interpreter.newParameterValue(isInstanceMethod, currentLocal, argumentType));
      currentLocal++;
      if (argumentType.getSize() == 2) {
        frame.setLocal(currentLocal, interpreter.newEmptyValue(currentLocal));
        currentLocal++;
      }
    }

    // 再者，如果local variable仍有空间，则设置成empty值。
    while (currentLocal < method.maxLocals) {
      frame.setLocal(currentLocal, interpreter.newEmptyValue(currentLocal));
      currentLocal++;
    }

    // 最后，设置返回值。
    frame.setReturn(interpreter.newReturnTypeValue(Type.getReturnType(method.desc)));
    return frame;
  }
}
```

##### init方法

在`Analyzer`类当中，`init`方法是一个空实现，它也是提供了一个机会，一个做“准备工作”的机会。至于做什么样的“准备工作”，需要结合具体的使用场景来决定。

```java
public class Analyzer<V extends Value> implements Opcodes {
  /**
     * Initializes this analyzer.
     * This method is called just before the execution of control flow analysis loop in #analyze.
     * The default implementation of this method does nothing.
     */
  protected void init(final String owner, final MethodNode method) throws AnalyzerException {
    // Nothing to do.
  }
}
```

##### newFrame方法

`newFrame`方法的主要作用是创建一个新的Frame对象。

下面的两个`newFrame`方法就是直接调用`Frame`类的构造方法，非常直白，那么这两个`newFrame`方法有什么隐含的用途呢？

```java
public class Analyzer<V extends Value> implements Opcodes {
  protected Frame<V> newFrame(int numLocals, int numStack) {
    return new Frame<>(numLocals, numStack);
  }

  protected Frame<V> newFrame(Frame<? extends V> frame) {
    return new Frame<>(frame);
  }
}
```

以前的时候，我们介绍过：`Analyzer`和`Frame`是属于“固定”的部分，而`Interpreter`和`Value`类是属于“变化”的部分。

```pseudocode
┌──────────┬─────────────┐
│          │  Analyzer   │
│  Fixed   ├─────────────┤
│          │    Frame    │
├──────────┼─────────────┤
│          │ Interpreter │
│ Variable ├─────────────┤
│          │    Value    │
└──────────┴─────────────┘
```

但是，`Analyzer`和`Frame`是相对的“固定”，而不是绝对的“固定”。在有些情况下，ASM提供的`Frame`类可能不满足实际的应用要求，那么我们就可以修改这两个`newFrame`方法来替换成我们自己想要使用的`Frame`的子类。

##### merge方法

`merge`方法的主要作用是合并两个Frame对象。

从代码实现上来说，这个`merge`方法会调用`Frame.merge()`方法，会再进一步调用`Interpreter.merge()`方法。

同时，也要注意，我们省略了另一个`merge`方法，因为该`merge`方法是处理`jsr`指令的，而`jsr`指令已经不推荐使用了。

```java
public class Analyzer<V extends Value> implements Opcodes {
  private void merge(final int insnIndex, final Frame<V> frame, final Subroutine subroutine) throws AnalyzerException {
    boolean changed;

    // 获取旧的Frame对象
    Frame<V> oldFrame = frames[insnIndex];
    if (oldFrame == null) {
      // 如果旧的Frame是null，那么将新的Frame直接存储起来
      frames[insnIndex] = newFrame(frame);
      changed = true;
    } else {
      // 如果旧的Frame不是null，那么将旧的Frame和新的Frame进行merge操作
      changed = oldFrame.merge(frame, interpreter);
    }

    // ... 此处省略一段代码，这段代码与jsr指令相关，而jsr指令不推荐使用了。

    // 记录下一个需要处理的指令
    if (changed && !inInstructionsToProcess[insnIndex]) {
      inInstructionsToProcess[insnIndex] = true;
      instructionsToProcess[numInstructionsToProcess++] = insnIndex;
    }
  }
}
```

##### ControlFlowEdge方法

下面的`newControlFlowEdge`和`newControlFlowExceptionEdge`方法就是实现control flow analysis的关键。

- `newControlFlowEdge`方法：记录正常的跳转：指令的顺序执行、指令的跳转执行（if、switch）。
- `newControlFlowExceptionEdge`方法：记录异常的跳转：当程序出现异常的时候，程序应该跳向哪里执行。

```java
public class Analyzer<V extends Value> implements Opcodes {
  protected void newControlFlowEdge(int insnIndex, int successorIndex) {
    // Nothing to do.
  }

  protected boolean newControlFlowExceptionEdge(int insnIndex, int successorIndex) {
    return true;
  }

  protected boolean newControlFlowExceptionEdge(int insnIndex, TryCatchBlockNode tryCatchBlock) {
    return newControlFlowExceptionEdge(insnIndex, insnList.indexOf(tryCatchBlock.handler));
  }
}
```

### 4.3.2 实现原理MockAnalyzer

在刚开始接触`Analyzer`类的时候，不太容易理解。因此，在[项目](https://gitee.com/lsieun/learn-java-asm)当中，我们提供了`MockAnalyzer`类，它是对`Analyzer`类的简化，理解起来更容易一些。借助于`MockAnalyzer`这个类，我们来讲解一下`Analyzer`类的实现原理。

#### class info

第一个部分，`MockAnalyzer`类实现了`Opcodes`类。

```java
public class MockAnalyzer<V extends Value> implements Opcodes {
}
```

#### fields

第二个部分，`MockAnalyzer`类定义的字段有哪些。在这个类当中，我们定义了原来的`interpreter`字段，因为其它的字段都可以转换成局部变量。

```java
public class MockAnalyzer<V extends Value> implements Opcodes {
  private final Interpreter<V> interpreter;
}
```

#### constructors

第三个部分，`MockAnalyzer`类定义的构造方法有哪些。

```java
public class MockAnalyzer<V extends Value> implements Opcodes {
  public MockAnalyzer(Interpreter<V> interpreter) {
    this.interpreter = interpreter;
  }
}
```

#### methods

第四个部分，`MockAnalyzer`类定义的方法有哪些。

```java
public class MockAnalyzer<V extends Value> implements Opcodes {
  public Frame<V>[] analyze(String owner, MethodNode method) throws AnalyzerException {
    // 第一步，如果是abstract或native方法，则直接返回。
    if ((method.access & (ACC_ABSTRACT | ACC_NATIVE)) != 0) {
      return (Frame<V>[]) new Frame<?>[0];
    }

    // 第二步，定义局部变量
    // （1）数据输入：获取指令集
    InsnList insnList = method.instructions;
    int size = insnList.size();

    // （2）中间状态：记录需要哪一个指令需要处理
    boolean[] instructionsToProcess = new boolean[size];

    // （3）数据输出：最终的返回结果
    Frame<V>[] frames = (Frame<V>[]) new Frame<?>[size];

    // 第三步，开始计算
    // （1）开始计算：根据方法的参数，计算方法的初始Frame
    Frame<V> currentFrame = computeInitialFrame(owner, method);
    merge(frames, 0, currentFrame, instructionsToProcess);

    // （2）开始计算：根据方法的每一条指令，计算相应的Frame
    while (getCount(instructionsToProcess) > 0) {
      // 获取需要处理的指令索引（insnIndex）和旧的Frame（oldFrame）
      int insnIndex = getFirst(instructionsToProcess);
      Frame<V> oldFrame = frames[insnIndex];
      instructionsToProcess[insnIndex] = false;

      // 模拟每一条指令的执行
      try {
        AbstractInsnNode insnNode = method.instructions.get(insnIndex);
        int insnOpcode = insnNode.getOpcode();
        int insnType = insnNode.getType();

        // 这三者并不是真正的指令，分别表示Label、LineNumberTable和Frame
        if (insnType == AbstractInsnNode.LABEL
            || insnType == AbstractInsnNode.LINE
            || insnType == AbstractInsnNode.FRAME) {
          merge(frames, insnIndex + 1, oldFrame, instructionsToProcess);
        }
        else {
          // 这里是真正的指令
          currentFrame.init(oldFrame).execute(insnNode, interpreter);

          if (insnNode instanceof JumpInsnNode) {
            JumpInsnNode jumpInsn = (JumpInsnNode) insnNode;
            // if之后的语句
            if (insnOpcode != GOTO) {
              merge(frames, insnIndex + 1, currentFrame, instructionsToProcess);
            }

            // if和goto跳转之后的位置
            int jumpInsnIndex = insnList.indexOf(jumpInsn.label);
            merge(frames, jumpInsnIndex, currentFrame, instructionsToProcess);
          }
          else if (insnNode instanceof LookupSwitchInsnNode) {
            LookupSwitchInsnNode lookupSwitchInsn = (LookupSwitchInsnNode) insnNode;

            // lookupswitch的default情况
            int targetInsnIndex = insnList.indexOf(lookupSwitchInsn.dflt);
            merge(frames, targetInsnIndex, currentFrame, instructionsToProcess);

            // lookupswitch的各种case情况
            for (int i = 0; i < lookupSwitchInsn.labels.size(); ++i) {
              LabelNode label = lookupSwitchInsn.labels.get(i);
              targetInsnIndex = insnList.indexOf(label);
              merge(frames, targetInsnIndex, currentFrame, instructionsToProcess);
            }
          }
          else if (insnNode instanceof TableSwitchInsnNode) {
            TableSwitchInsnNode tableSwitchInsn = (TableSwitchInsnNode) insnNode;

            // tableswitch的default情况
            int targetInsnIndex = insnList.indexOf(tableSwitchInsn.dflt);
            merge(frames, targetInsnIndex, currentFrame, instructionsToProcess);

            // tableswitch的各种case情况
            for (int i = 0; i < tableSwitchInsn.labels.size(); ++i) {
              LabelNode label = tableSwitchInsn.labels.get(i);
              targetInsnIndex = insnList.indexOf(label);
              merge(frames, targetInsnIndex, currentFrame, instructionsToProcess);
            }
          }
          else if (insnOpcode != ATHROW && (insnOpcode < IRETURN || insnOpcode > RETURN)) {
            merge(frames, insnIndex + 1, currentFrame, instructionsToProcess);
          }
        }
      }
      catch (AnalyzerException e) {
        throw new AnalyzerException(e.node, "Error at instruction " + insnIndex + ": " + e.getMessage(), e);
      }
    }

    return frames;
  }

  private int getCount(boolean[] array) {
    int count = 0;
    for (boolean flag : array) {
      if (flag) {
        count++;
      }
    }
    return count;
  }

  private int getFirst(boolean[] array) {
    int length = array.length;
    for (int i = 0; i < length; i++) {
      boolean flag = array[i];
      if (flag) {
        return i;
      }
    }
    return -1;
  }

  private Frame<V> computeInitialFrame(String owner, MethodNode method) {
    Frame<V> frame = new Frame<>(method.maxLocals, method.maxStack);
    int currentLocal = 0;

    // 第一步，判断是否需要存储this变量
    boolean isInstanceMethod = (method.access & ACC_STATIC) == 0;
    if (isInstanceMethod) {
      Type ownerType = Type.getObjectType(owner);
      V value = interpreter.newParameterValue(isInstanceMethod, currentLocal, ownerType);
      frame.setLocal(currentLocal, value);
      currentLocal++;
    }

    // 第二步，将方法的参数存入到local variable内
    Type[] argumentTypes = Type.getArgumentTypes(method.desc);
    for (Type argumentType : argumentTypes) {
      V value = interpreter.newParameterValue(isInstanceMethod, currentLocal, argumentType);
      frame.setLocal(currentLocal, value);
      currentLocal++;
      if (argumentType.getSize() == 2) {
        frame.setLocal(currentLocal, interpreter.newEmptyValue(currentLocal));
        currentLocal++;
      }
    }

    // 第三步，将local variable的剩余位置填补上空值
    while (currentLocal < method.maxLocals) {
      frame.setLocal(currentLocal, interpreter.newEmptyValue(currentLocal));
      currentLocal++;
    }

    // 第四步，设置返回值类型
    frame.setReturn(interpreter.newReturnTypeValue(Type.getReturnType(method.desc)));
    return frame;
  }

  /**
     * Merge old frame with new frame.
     *
     * @param frames 所有的frame信息。
     * @param insnIndex 当前指令的索引。
     * @param newFrame 新的frame
     * @param instructionsToProcess 记录哪一条指令需要处理
     * @throws AnalyzerException 分析错误，抛出此异常
     */
  private void merge(Frame<V>[] frames, int insnIndex, Frame<V> newFrame, boolean[] instructionsToProcess) throws AnalyzerException {
    boolean changed;
    Frame<V> oldFrame = frames[insnIndex];
    if (oldFrame == null) {
      frames[insnIndex] = new Frame<>(newFrame);
      changed = true;
    }
    else {
      changed = oldFrame.merge(newFrame, interpreter);
    }

    if (changed && !instructionsToProcess[insnIndex]) {
      instructionsToProcess[insnIndex] = true;
    }
  }
}
```

### 4.3.3 如何使用Analyzer类

使用Analyzer类非常简单，它的主要目的就是分析，也就是调用它的`analyze`方法。

虽然下面的两行代码比较简单，但是它包含了`Analyzer`、`Frame`、`Interpreter`和`Value`四个类的应用：

```pseudocode
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

另外，如果我们想验证一下`MockAnalyzer`类是否可以正常工作，可以将`HelloWorldFrameTree`类当中的`Analyzer`类替换成`MockAnalyzer`类：

```java
MockAnalyzer<V> analyzer = new MockAnalyzer<>(interpreter);
Frame<V>[] frames = analyzer.analyze(owner, mn);
```

### 4.3.4 总结

本文内容总结如下：

- 第一点，介绍`Analyzer`类的主要组成部分。
- 第二点，借助于一个简化的`Analyzer`类（`MockAnalyzer`），来理解data flow analysis的工作原理。虽然我们没有介绍control flow analysis，但是大家只要追踪一下`newControlFlowEdge`和`newControlFlowExceptionEdge`方法就知道了。
- **第三点，如何使用Analyzer类。其实，就是调用Analyzer类的`analyze`方法，得到生成的`Frame[]`信息，然后再利用这个`Frame[]`信息做进一步的分析。**

```pseudocode
┌──────────┬─────────────┐
│          │  Analyzer   │
│  Fixed   ├─────────────┤
│          │    Frame    │
├──────────┼─────────────┤
│          │ Interpreter │
│ Variable ├─────────────┤
│          │    Value    │
└──────────┴─────────────┘
```

## 4.4 BasicValue-BasicInterpreter

在本章内容当中，最核心的内容就是下面两行代码。这两行代码包含了`asm-analysis.jar`当中`Analyzer`、`Frame`、`Interpreter`和`Value`最重要的四个类：

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

在本文当中，我们将介绍`BasicInterpreter`和`BasicValue`类：

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

### 4.4.1 BasicValue

#### class info

第一个部分，`BasicValue`类实现了`Value`接口。

```java
public class BasicValue implements Value {
}
```

#### fields

第二个部分，`BasicValue`类定义的字段有哪些。

```java
public class BasicValue implements Value {
  private final Type type;

  public Type getType() {
    return type;
  }
}
```

通过以下三点来理解`BasicValue`和`Type`之间的关系：

- 第一点，`BasicValue`是`asm-analysis.jar`当中定义的类，是local variable和operand stack当中存储的数据类型。
- 第二点，`Type`是`asm.jar`当中定义的类，它是对具体的`.class`文件当中的Internal Name和Descriptor的一种ASM表示方式。
- 第三点，`BasicValue`类就是对`Type`类的封装。

| 领域 | ClassFile                | ASM                                                | Frame(local variable+operand stack)      |
| ---- | ------------------------ | -------------------------------------------------- | ---------------------------------------- |
| 类型 | Internal Name/Descriptor | Type                                               | Value                                    |
| 示例 | `Ljava/lang/String;`     | `Type t = Type.getObjectType("java/lang/String");` | `BasicValue val = BasicValue.INT_VALUE;` |

```pseudocode
  ┌─────────────────────────────┐     |        ┌──────────────────────────────────────────────┐
  │          asm.jar            │     |        │            asm-analysis.jar                  │
  │  ClassReader   Type         │     |        │                                              │
  │  ClassVisitor  ClassWriter  │     |        │  Value Interpreter Frame Analyzer            │
  └─────────────────────────────┘     |        └──────────────────────────────────────────────┘
---------------------------------------------------------------------------------------------------
                                      |    ┌──────────────────────────────────────────────────────┐
                                      |    │     operand stack                                    │
    ┌────────────────────────┐        |    │    ┌──────────────┐              JVM                 │
    │     Internal Name      │        |    │    ├──────────────┤           Stack Frame            │
    │       Descriptor       │        |    │    ├──────────────┤                                  │
    │                        │        |    │    ├──────────────┤                                  │
    │        .class          │        |    │    ├──────────────┤                                  │
    │       ClassFile        │        |    │    ├──────────────┤         local variable           │
    │                        │        |    │    ├──────────────┤     ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐     │
    └────────────────────────┘        |    │    └──────────────┘     └──┘ └──┘ └──┘ └──┘ └──┘     │
                                      |    │                           0    1    2    3    4      │
                                      |    └──────────────────────────────────────────────────────┘
```

#### constructors

第三个部分，`BasicValue`类定义的构造方法有哪些。

```java
public class BasicValue implements Value {
  public BasicValue(Type type) {
    this.type = type;
  }
}
```

#### methods

> [关于字节码：哪些Java编译器使用jsr指令，以及用于什么目的？ | 码农家园 (codenong.com)](https://www.codenong.com/21150154/)
>
> In Oracle's implementation of a compiler for the Java programming language prior to Java SE 6, the jsr instruction was used with the ret instruction in the implementation of the finally clause

第四个部分，`BasicValue`类定义的方法有哪些。

```java
public class BasicValue implements Value {
  @Override
  public int getSize() {
    return type == Type.LONG_TYPE || type == Type.DOUBLE_TYPE ? 2 : 1;
  }

  public boolean isReference() {
    return type != null && (type.getSort() == Type.OBJECT || type.getSort() == Type.ARRAY);
  }

  @Override
  public String toString() {
    if (this == UNINITIALIZED_VALUE) {
      return ".";
      // A 这个 对应 jsr 指令，jdk7即往后已经弃用
    } else if (this == RETURNADDRESS_VALUE) {
      return "A";
    } else if (this == REFERENCE_VALUE) {
      return "R";
    } else {
      return type.getDescriptor();
    }
  }
}
```

#### static fields

在`BasicValue`类当中，定义了7个静态字段：

- `UNINITIALIZED_VALUE` means “all possible values”.
- `INT_VALUE` means “all int, short, byte, boolean or char values”.
- `FLOAT_VALUE` means “all float values”.
- `LONG_VALUE` means “all long values”.
- `DOUBLE_VALUE` means “all double values”.
- `REFERENCE_VALUE` means “all object and array values”.
- `RETURNADDRESS_VALUE` is used for subroutines.（对应JSR指令，jdk7已经废弃）

```java
public class BasicValue implements Value {
  public static final BasicValue UNINITIALIZED_VALUE = new BasicValue(null);

  public static final BasicValue INT_VALUE = new BasicValue(Type.INT_TYPE);
  public static final BasicValue FLOAT_VALUE = new BasicValue(Type.FLOAT_TYPE);
  public static final BasicValue LONG_VALUE = new BasicValue(Type.LONG_TYPE);
  public static final BasicValue DOUBLE_VALUE = new BasicValue(Type.DOUBLE_TYPE);

  public static final BasicValue REFERENCE_VALUE = new BasicValue(Type.getObjectType("java/lang/Object"));
  public static final BasicValue RETURNADDRESS_VALUE = new BasicValue(Type.VOID_TYPE);//对应JSR指令, jdk7已经废弃
}
```

在这里，我们要注意：虽然`BasicValue`类定义了这7个静态字段，但是并不是表示说`BasicValue`只能有这7个字段的值，它还可以创建许许多的对象实例。为什么我们要说这样一件事情呢？因为在刚开始，我们会经常用到这7个静态字段的值，很容易误导我们，让我们觉得只有这7个静态字段的值。实际上，只有`BasicInterpreter`和`BasicVerifier`两个类完全限定于使用这7个静态字段，而`SimpleVerifier`就会创建许许多的`BasicValue`对象。

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

用一个比喻来加深印象。`BasicInterpreter`和`BasicVerifier`两个类就像两个小孩儿，他们只会7个单词，说的所有的话都是由这7个单词组成；而`SimpleVerifier`就像是一个词汇量丰富的初中学生，可以描述事物的具体细节。

### 4.4.2 BasicInterpreter

The `BasicInterpreter` class is one of the predefined subclass of the `Interpreter` abstract class. It simulates the effect of bytecode instructions by using seven sets of values, defined in the `BasicValue` class.

#### class info

第一个部分，`BasicInterpreter`类继承自`Interpreter<BasicValue>`类。

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
}
```

#### fields

第二个部分，`BasicInterpreter`类定义的字段有哪些。

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
  /**
   * Special type used for the {@literal null} literal. This is an object reference type with
   * descriptor 'Lnull;'.
   */  
  public static final Type NULL_TYPE = Type.getObjectType("null");
}
```

#### constructors

第三个部分，`BasicInterpreter`类定义的构造方法有哪些。

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
  public BasicInterpreter() {
    super(ASM9);
    if (getClass() != BasicInterpreter.class) {
      throw new IllegalStateException();
    }
  }

  protected BasicInterpreter(int api) {
    super(api);
  }
}
```

#### methods

> 方法内的处理逻辑不是重点，知道返回值类型是BasicValue即可

第四个部分，`BasicInterpreter`类定义的方法有哪些。下面这几个方法都是对`Interpreter`定义的方法进行重写。

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
  @Override
  public BasicValue newValue(Type type) {
    // 返回BasicValue定义的7个静态字段之一
  }

  @Override
  public BasicValue newOperation(AbstractInsnNode insn) throws AnalyzerException {
    // 返回BasicValue定义的7个静态字段之一
  }

  @Override
  public BasicValue copyOperation(AbstractInsnNode insn, BasicValue value) throws AnalyzerException {
    return value;
  }

  public BasicValue unaryOperation(AbstractInsnNode insn, BasicValue value) throws AnalyzerException {
    // 返回BasicValue定义的7个静态字段之一
  }

  public BasicValue binaryOperation(AbstractInsnNode insn, BasicValue value1, BasicValue value2) throws AnalyzerException {
    // 返回BasicValue定义的7个静态字段之一
  }

  @Override
  public BasicValue ternaryOperation(AbstractInsnNode insn, BasicValue value1, BasicValue value2, BasicValue value3) throws AnalyzerException {
    return null;
  }

  @Override
  public BasicValue naryOperation(AbstractInsnNode insn, List<? extends BasicValue> values) throws AnalyzerException {
    // 返回BasicValue定义的7个静态字段之一
  }

  @Override
  public void returnOperation(AbstractInsnNode insn, BasicValue value, BasicValue expected) throws AnalyzerException {
    // Nothing to do.
  }

  @Override
  public BasicValue merge(BasicValue value1, BasicValue value2) {
    if (!value1.equals(value2)) {
      return BasicValue.UNINITIALIZED_VALUE;
    }
    return value1;
  }
}
```

### 4.4.3 示例：打印Frame的状态

Get type info for each variable and stack slot for each method instruction

```java
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(className, mn);

for (Frame<BasicValue> f : frames) {
  for (int i = 0; i < f.getLocals(); ++i) {
    BasicValue local = f.getLocal(i);
    // ... local.getType()
  }

  for (int j = 0; j < f.getStackSize(); ++j) {
    BasicValue stack = f.getStack(j);
    // ...
  }
}
```

打印每条instruction对应的Frame状态：

```java
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.analysis.Analyzer;
import org.objectweb.asm.tree.analysis.BasicInterpreter;
import org.objectweb.asm.tree.analysis.BasicValue;
import org.objectweb.asm.tree.analysis.Frame;

import java.util.ArrayList;
import java.util.List;

public class HelloWorldAnalysisTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）进行分析
    String className = cn.name;
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);

    Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
    Frame<BasicValue>[] frames = analyzer.analyze(className, mn);

    for (Frame<BasicValue> f : frames) {
      List<BasicValue> localList = new ArrayList<>();
      for (int i = 0; i < f.getLocals(); ++i) {
        BasicValue local = f.getLocal(i);
        localList.add(local);
      }

      List<BasicValue> stackList = new ArrayList<>();
      for (int j = 0; j < f.getStackSize(); ++j) {
        BasicValue stack = f.getStack(j);
        stackList.add(stack);
      }

      String line = FrameUtils.toLine(localList, stackList);
      System.out.println(line);
    }
  }
}
```

### 4.4.4 细节：top、null和void的处理

在`BasicInterpreter`类当中，对于top值、null值和void进行了处理，但是其中有一些让人容易混淆的地方。

#### 三者分别指什么

- **`top`：在JVM文档的[4.7.4. The StackMapTable Attribute](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.4)定义的一个类型，它表示当前local variable当中的某一个位置处于未初始化的状态。**
- `null`或`aconst_null`：在Java语言当中，表现为`null`值；在`.class`文件中，表现为`aconst_null`，是[JVM文档](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.aconst_null)定义的一个指令，将一个`null`加载到operand stack上。（null在class中也是一个类型，即type null，且JVM不要求给null类型赋具体值）
- `void`：方法的返回值是`void`类型。
  - 如果方法的返回值类型不是`void`类型，假如说是一个`int`类型，那么它会将返回的`int`值加载到operand stack上。
  - 如果返回值类型是`void`类型，那么则不需要加载任何值到operand stack上。

#### 概念领域的转换

刚刚介绍的`top`、`null`和`void`都是在`.class`文件所存在的内容，接下来它进行转换成ASM当中的`Type`类型，再接着转换成ASM当中的`Value`类型，之后就可以结合`Frame`类来模拟local variable和operand stack的变化了。

```pseudocode
.class --> ASM Type --> ASM Value
```

三个概念领域的关系可以表示成下图：

```pseudocode
  ┌─────────────────────────────┐     |        ┌──────────────────────────────────────────────┐
  │          asm.jar            │     |        │            asm-analysis.jar                  │
  │  ClassReader   Type         │     |        │                                              │
  │  ClassVisitor  ClassWriter  │     |        │  Value Interpreter Frame Analyzer            │
  └─────────────────────────────┘     |        └──────────────────────────────────────────────┘
---------------------------------------------------------------------------------------------------
                                      |    ┌──────────────────────────────────────────────────────┐
                                      |    │     operand stack                                    │
    ┌────────────────────────┐        |    │    ┌──────────────┐              JVM                 │
    │     Internal Name      │        |    │    ├──────────────┤           Stack Frame            │
    │       Descriptor       │        |    │    ├──────────────┤                                  │
    │                        │        |    │    ├──────────────┤                                  │
    │ top, aconst_null, void │        |    │    ├──────────────┤                                  │
    │                        │        |    │    ├──────────────┤         local variable           │
    │        .class          │        |    │    ├──────────────┤     ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐     │
    │       ClassFile        │        |    │    └──────────────┘     └──┘ └──┘ └──┘ └──┘ └──┘     │
    └────────────────────────┘        |    │                           0    1    2    3    4      │
                                      |    └──────────────────────────────────────────────────────┘
```

#### 转换过程

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.2) 中对`aconst_null`操作码的描述为Push the `null` object `reference` onto the operand stack. 也就是说null也是算 object 引用类型。这样比较好理解为啥ASM中 `BasicValue.REFERENCE_VALUE`对应`aconst_null`

在下表当中，就是`top`、`null`和`void`三者相对应的转换值：

```pseudocode
┌─────────────┬────────────────────────────┬────────────────────────────────┐
│   .class    │          ASM Type          │      ASM Value in Frame        │
├─────────────┼────────────────────────────┼────────────────────────────────┤
│     top     │            null            │ BasicValue.UNINITIALIZED_VALUE │
├─────────────┼────────────────────────────┼────────────────────────────────┤
│ aconst_null │ BasicInterpreter.NULL_TYPE │   BasicValue.REFERENCE_VALUE   │
├─────────────┼────────────────────────────┼────────────────────────────────┤
│    void     │       Type.VOID_TYPE       │              null              │
└─────────────┴────────────────────────────┴────────────────────────────────┘
```

在上表当中，容易混淆的地方就是：第一列有`aconst_null`，第二列有`null`，第三列有`null`，但是这三者不在同一行，且所隶属的“领域”是不同的。

首先，是`top`的查找过程:

- `Analyzer.analyze()`方法 –> `Analyzer.computeInitialFrame()`方法
- `Interpreter.newEmptyValue()`方法
- `BasicInterpreter.newValue()`方法

```java
public abstract class Interpreter<V extends Value> {
  public V newEmptyValue(int local) {
    return newValue(null);
  }
}
```

其次，是`null`或`aconst_null`的查找过程:

- `Analyzer.analyze()`方法
- `Frame.execute()`方法的`ACONST_NULL`指令
- `BasicInterpreter.newOperation()`方法当中的`ACONST_NULL`指令 –> `BasicInterpreter.newValue()`方法

最后，是`void`的查找过程:

- `Analyzer.analyze()`方法
- `Frame.execute()`方法调用方法的指令 –> `Frame.executeInvokeInsn()`方法
- `BasicInterpreter.naryOperation()`方法 –> `BasicInterpreter.newValue()`方法

如果遇到`void`类型的时候，它不应该向Frame当中添加任何值，因此其相应的`BasicValue`值为`null`。

在`BasicInterpreter`类当中，定义了一个`NULL_TYPE`字段：

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
    public static final Type NULL_TYPE = Type.getObjectType("null");
}
```

这个`NULL_TYPE`字段在`newOperation`方法中处理`aconst_null`指令时用到。

```java
public class BasicInterpreter extends Interpreter<BasicValue> implements Opcodes {
  @Override
  public BasicValue newOperation(final AbstractInsnNode insn) throws AnalyzerException {
    switch (insn.getOpcode()) {
      case ACONST_NULL:
        return newValue(NULL_TYPE);      // aconst_null --> NULL_TYPE
        // ...
      case GETSTATIC:
        return newValue(Type.getType(((FieldInsnNode) insn).desc));
      case NEW:
        return newValue(Type.getObjectType(((TypeInsnNode) insn).desc));
      default:
        throw new AssertionError();
    }
  }

  @Override
  public BasicValue newValue(final Type type) {
    if (type == null) {
      return BasicValue.UNINITIALIZED_VALUE; // top --> null --> UNINITIALIZED_VALUE
    }
    switch (type.getSort()) {
      case Type.VOID:
        return null; // void --> Type.VOID_TYPE --> null
      case Type.BOOLEAN:
      case Type.CHAR:
      case Type.BYTE:
      case Type.SHORT:
      case Type.INT:
        return BasicValue.INT_VALUE;
      case Type.FLOAT:
        return BasicValue.FLOAT_VALUE;
      case Type.LONG:
        return BasicValue.LONG_VALUE;
      case Type.DOUBLE:
        return BasicValue.DOUBLE_VALUE;
      case Type.ARRAY:
      case Type.OBJECT:
        return BasicValue.REFERENCE_VALUE;
      default:
        throw new AssertionError();
    }
  }

  @Override
  public BasicValue naryOperation(AbstractInsnNode insn, final List<? extends BasicValue> values)
    throws AnalyzerException {
    int opcode = insn.getOpcode();
    if (opcode == MULTIANEWARRAY) {
      return newValue(Type.getType(((MultiANewArrayInsnNode) insn).desc));
    } else if (opcode == INVOKEDYNAMIC) {
      return newValue(Type.getReturnType(((InvokeDynamicInsnNode) insn).desc));
    } else {
      return newValue(Type.getReturnType(((MethodInsnNode) insn).desc));
    }
  }
}
```

### 4.4.5 总结

本文内容总结如下：

- 第一点，`BasicValue`类实现了`Value`接口，它本质是在`Type`类型基础的进一步封装。在`BasicValue`类当中，定义了7个静态字段。
- 第二点，`BasicInterpreter`类继承自`Interpreter`类，它就是利用`BasicValue`类当中定义的7个静态字段进行运算，得到的结果是这7个字段当中的任意一个值，或者是`null`值。
- 第三点，代码示例，打印出Frame(local variable和operand stack)当中的元素。
- 第四点，从细节上来说，对top、null和void进行区分。

## 4.5 BasicValue-BasicInterpreter示例：移除Dead Code

### 4.5.1 如何移除Dead Code

This `BasicInterpreter` is not very useful in itself, but it can be used as an “empty” `Interpreter` implementation in order to construct an `Analyzer`. This analyzer can then be used to detect unreachable code in a method.

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

Indeed, even by following both branches in jumps instructions, it is not possible to reach code that cannot be reached from the first instruction. The consequence is that, after an analysis, and whatever the `Interpreter` implementation, the computed frames – returned by the `Analyzer.getFrames` method - are `null` for instructions that cannot be reached.

那么，为什么可以移除dead code呢？

- 其实，`BasicInterpreter`类本身并没有太多的作用，因为所有的引用类型都使用`BasicValue.REFERENCE_VALUE`来表示，类型非常单一，没有太多的操作空间。
- 相反的，`Analyzer`类才是起到主要作用的类，`Frame<V>[] frames = (Frame<V>[]) new Frame<?>[insnListSize];`。那么，在默认情况下，`frames`里所有元素都是`null`值。如果某一条instruction可以执行到，那么就会设置对应的`frames`当中的值，就不再为`null`了。反过来说，如果`frames`当中的某一个元素为`null`，那么也就表示对应的instruction就是dead code。

使用`BasicInterpreter`类时，指令（Instruction）和Frame之间的关系：

```shell
test:(I)V
000:                                 goto L0    {R, I} | {}
001:                                  athrow    {} | {}
002:                                      L0    {R, I} | {}
003:                    getstatic System.out    {R, I} | {}
004:                                 iload_1    {R, I} | {R}
005:                                 goto L1    {R, I} | {R, I}
006:                                     nop    {} | {}
007:                                     nop    {} | {}
008:                                  athrow    {} | {}
009:                                      L1    {R, I} | {R, I}
010:                                 ifne L2    {R, I} | {R, I}
011:                          ldc "val is 0"    {R, I} | {R}
012:                                 goto L3    {R, I} | {R, R}
013:                                      L2    {R, I} | {R}
014:                      ldc "val is not 0"    {R, I} | {R}
015:                                      L3    {R, I} | {R, R}
016:       invokevirtual PrintStream.println    {R, I} | {R, R}
017:                                  return    {R, I} | {}
018:                                     nop    {} | {}
019:                                     nop    {} | {}
020:                                  athrow    {} | {}
================================================================
```

### 4.5.2 代码示例

#### 预期目标

在之前的Tree Based Class Transformation示例当中，有一个示例是“优化跳转”，经过优化之后，有一些instruction就成为dead code。

我们的预期目标就是移除那些成为dead code的instruction。

#### 编码实现

```java
import lsieun.asm.tree.transformer.MethodTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.tree.AbstractInsnNode;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.LabelNode;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.analysis.*;

public class RemoveDeadCodeNode extends ClassNode {
  public RemoveDeadCodeNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    MethodTransformer mt = new MethodRemoveDeadCodeTransformer(name, null);
    for (MethodNode mn : methods) {
      mt.transform(mn);
    }

    // 其次，调用父类的方法实现
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class MethodRemoveDeadCodeTransformer extends MethodTransformer {
    private final String owner;

    public MethodRemoveDeadCodeTransformer(String owner, MethodTransformer mt) {
      super(mt);
      this.owner = owner;
    }

    @Override
    public void transform(MethodNode mn) {
      // 首先，处理自己的代码逻辑
      Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
      try {
        analyzer.analyze(owner, mn);
        Frame<BasicValue>[] frames = analyzer.getFrames();
        AbstractInsnNode[] insnNodes = mn.instructions.toArray();
        for (int i = 0; i < frames.length; i++) {
          // Label不算真正的指令，但是可以用来标记跳转位置
          if (frames[i] == null && !(insnNodes[i] instanceof LabelNode)) {
            mn.instructions.remove(insnNodes[i]);
          }
        }
      }
      catch (AnalyzerException ex) {
        ex.printStackTrace();
      }

      // 其次，调用父类的方法实现
      super.transform(mn);
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.*;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new RemoveDeadCodeNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
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
  public void test(int);
    Code:
       0: goto          3
       3: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
       6: iload_1
       7: goto          10
      10: ifne          18
      13: ldc           #18                 // String val is 0
      15: goto          20
      18: ldc           #20                 // String val is not 0
      20: invokevirtual #26                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23
```

### 4.5.3 解决方式二：抛弃Frame信息

There are more efficient ways, but they require writing more code.

```pseudocode
            ┌─── data flow analysis
            │
Analyzer ───┤
            │
            └─── control flow analysis
```

#### ControlFlowAnalyzer

这个`ControlFlowAnalyzer`是模仿`org.objectweb.asm.tree.analysis.Analyzer`写的，但是不带有泛型信息，也不带有`Frame`相关的信息，只是将control flow analysis功能抽取出来。

```java
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.*;
import org.objectweb.asm.tree.analysis.AnalyzerException;

import java.util.ArrayList;
import java.util.List;

public class ControlFlowAnalyzer implements Opcodes {

  private InsnList insnList;
  private int insnListSize;
  private List<TryCatchBlockNode>[] handlers;

  /**
     * 记录需要处理的instructions.
     *
     * NOTE: 这三个字段为一组，应该一起处理，最好是放到同一个方法里来处理。
     * 因此，我就加了三个新方法。
     * {@link #initInstructionsToProcess()}、{@link #addInstructionsToProcess(int)}和
     * {@link #removeInstructionsToProcess()}
     *
     * @see #initInstructionsToProcess()
     * @see #addInstructionsToProcess(int)
     * @see #removeInstructionsToProcess()
     */
  private boolean[] inInstructionsToProcess;
  private int[] instructionsToProcess;
  private int numInstructionsToProcess;

  public ControlFlowAnalyzer() {
  }

  public List<TryCatchBlockNode> getHandlers(final int insnIndex) {
    return handlers[insnIndex];
  }


  @SuppressWarnings("unchecked")
  // NOTE: analyze方法的返回值类型变成了void类型。
  public void analyze(final String owner, final MethodNode method) throws AnalyzerException {
    if ((method.access & (ACC_ABSTRACT | ACC_NATIVE)) != 0) {
      return;
    }
    insnList = method.instructions;
    insnListSize = insnList.size();
    handlers = (List<TryCatchBlockNode>[]) new List<?>[insnListSize];

    initInstructionsToProcess();

    // For each exception handler, and each instruction within its range, record in 'handlers' the
    // fact that execution can flow from this instruction to the exception handler.
    for (int i = 0; i < method.tryCatchBlocks.size(); ++i) {
      TryCatchBlockNode tryCatchBlock = method.tryCatchBlocks.get(i);
      int startIndex = insnList.indexOf(tryCatchBlock.start);
      int endIndex = insnList.indexOf(tryCatchBlock.end);
      for (int j = startIndex; j < endIndex; ++j) {
        List<TryCatchBlockNode> insnHandlers = handlers[j];
        if (insnHandlers == null) {
          insnHandlers = new ArrayList<>();
          handlers[j] = insnHandlers;
        }
        insnHandlers.add(tryCatchBlock);
      }
    }


    // Initializes the data structures for the control flow analysis.
    // NOTE: 调用addInstructionsToProcess方法，传入参数0，启动整个循环过程。
    addInstructionsToProcess(0);
    init(owner, method);

    // Control flow analysis.
    while (numInstructionsToProcess > 0) {
      // Get and remove one instruction from the list of instructions to process.
      int insnIndex = removeInstructionsToProcess();

      // Simulate the execution of this instruction.
      AbstractInsnNode insnNode = method.instructions.get(insnIndex);
      int insnOpcode = insnNode.getOpcode();
      int insnType = insnNode.getType();

      if (insnType == AbstractInsnNode.LABEL
          || insnType == AbstractInsnNode.LINE
          || insnType == AbstractInsnNode.FRAME) {
        newControlFlowEdge(insnIndex, insnIndex + 1);
      }
      else {
        if (insnNode instanceof JumpInsnNode) {
          JumpInsnNode jumpInsn = (JumpInsnNode) insnNode;
          if (insnOpcode != GOTO && insnOpcode != JSR) {
            newControlFlowEdge(insnIndex, insnIndex + 1);
          }
          int jumpInsnIndex = insnList.indexOf(jumpInsn.label);
          newControlFlowEdge(insnIndex, jumpInsnIndex);
        }
        else if (insnNode instanceof LookupSwitchInsnNode) {
          LookupSwitchInsnNode lookupSwitchInsn = (LookupSwitchInsnNode) insnNode;
          int targetInsnIndex = insnList.indexOf(lookupSwitchInsn.dflt);
          newControlFlowEdge(insnIndex, targetInsnIndex);
          for (int i = 0; i < lookupSwitchInsn.labels.size(); ++i) {
            LabelNode label = lookupSwitchInsn.labels.get(i);
            targetInsnIndex = insnList.indexOf(label);
            newControlFlowEdge(insnIndex, targetInsnIndex);
          }
        }
        else if (insnNode instanceof TableSwitchInsnNode) {
          TableSwitchInsnNode tableSwitchInsn = (TableSwitchInsnNode) insnNode;
          int targetInsnIndex = insnList.indexOf(tableSwitchInsn.dflt);
          newControlFlowEdge(insnIndex, targetInsnIndex);
          for (int i = 0; i < tableSwitchInsn.labels.size(); ++i) {
            LabelNode label = tableSwitchInsn.labels.get(i);
            targetInsnIndex = insnList.indexOf(label);
            newControlFlowEdge(insnIndex, targetInsnIndex);
          }
        }
        else if (insnOpcode != ATHROW && (insnOpcode < IRETURN || insnOpcode > RETURN)) {
          newControlFlowEdge(insnIndex, insnIndex + 1);
        }
      }

      List<TryCatchBlockNode> insnHandlers = handlers[insnIndex];
      if (insnHandlers != null) {
        for (TryCatchBlockNode tryCatchBlock : insnHandlers) {
          newControlFlowExceptionEdge(insnIndex, tryCatchBlock);
        }
      }

    }
  }


  protected void init(final String owner, final MethodNode method) throws AnalyzerException {
    // Nothing to do.
  }

  // NOTE: 这是一个新添加的方法
  private void initInstructionsToProcess() {
    inInstructionsToProcess = new boolean[insnListSize];
    instructionsToProcess = new int[insnListSize];
    numInstructionsToProcess = 0;
  }

  // NOTE: 这是一个新添加的方法
  private int removeInstructionsToProcess() {
    int insnIndex = this.instructionsToProcess[--numInstructionsToProcess];
    inInstructionsToProcess[insnIndex] = false;
    return insnIndex;
  }

  // NOTE: 这是一个新添加的方法
  private void addInstructionsToProcess(final int insnIndex) {
    if (!inInstructionsToProcess[insnIndex]) {
      inInstructionsToProcess[insnIndex] = true;
      instructionsToProcess[numInstructionsToProcess++] = insnIndex;
    }
  }

  protected void newControlFlowEdge(final int insnIndex, final int successorIndex) {
    // Nothing to do.
    addInstructionsToProcess(successorIndex);
  }

  protected void newControlFlowExceptionEdge(final int insnIndex, final TryCatchBlockNode tryCatchBlock) {
    newControlFlowExceptionEdge(insnIndex, insnList.indexOf(tryCatchBlock.handler));
  }

  protected void newControlFlowExceptionEdge(final int insnIndex, final int successorIndex) {
    // Nothing to do.
    addInstructionsToProcess(successorIndex);
  }
}
```

#### 编码实现

```java
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.analysis.AnalyzerException;

public class DeadCodeDiagnosis {
  public static int[] diagnose(String className, MethodNode mn) throws AnalyzerException {
    InsnList instructions = mn.instructions;
    int size = instructions.size();

    boolean[] flags = new boolean[size];
    ControlFlowAnalyzer analyzer = new ControlFlowAnalyzer() {
      @Override
      protected void newControlFlowEdge(int insnIndex, int successorIndex) {
        // 首先，处理自己的代码逻辑
        flags[insnIndex] = true;
        flags[successorIndex] = true;

        // 其次，调用父类的实现
        super.newControlFlowEdge(insnIndex, successorIndex);
      }
    };
    analyzer.analyze(className, mn);


    TIntArrayList intArrayList = new TIntArrayList();
    for (int i = 0; i < size; i++) {
      boolean flag = flags[i];
      if (!flag) {
        intArrayList.add(i);
      }
    }

    return intArrayList.toNativeArray();
  }
}
```

对于`DeadCodeDiagnosis`类，我们从三点来把握：

- 第一点，实现逻辑。通过重写`ControlFlowAnalyzer`类的`newControlFlowEdge`方法实现，将能够访问到了指令索引（instruction index）存放到一个`boolean[] flags`当中。
- 第二点，类名的命名规则。
  - 如果当前类是自己写的，那么类名的命名规则为`XxxDiagnosis`，其中定义的主要方法叫`diagnose`方法（诊断）。
  - 如果当前类继承自`Analyzer`类，或者说模仿`Analyzer`类，那么类命名规则为`XxxAnalyzer`类，其主要的方法为`analyze`方法（分析）。
- 第三点，`TIntArrayList`类的使用。
  - 类的作用：它的操作类似于`List<int>`类型，里面的每一个元素都是`int`类型，比使用`List<Integer>`更高效。
  - 类的来源：`TIntArrayList`类来源于trove4j类库。Trove is a high-performance library which provides primitive collections for Java.
  - 类的优势：The greatest benefit of using `TIntArrayList` is performance and memory consumption gains. No additional boxing/unboxing is needed as it stores the data inside of an `int[]` array.
  - 类的引用：在实际项目当中，可以在`pom.xml`文件中添加maven依赖

```xml
<dependency>
    <groupId>org.jetbrains.intellij.deps</groupId>
    <artifactId>trove4j</artifactId>
    <version>1.0.20200330</version>
</dependency>
```

#### 进行分析

```java
public class HelloWorldAnalysisTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）进行分析
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);
    int[] array = DeadCodeDiagnosis.diagnose(cn.name, mn);
    System.out.println(Arrays.toString(array));
    BoxDrawingUtils.printInstructionLinks(mn.instructions, array);
  }
}
```

输出结果：

```shell
[1, 6, 7, 8, 18, 19, 20]
      000: goto L0
┌──── 001: athrow
│     002: L0
│     003: getstatic System.out
│     004: iload_1
│     005: goto L1
├──── 006: nop
├──── 007: nop
├──── 008: athrow
│     009: L1
│     010: ifne L2
│     011: ldc "val is 0"
│     012: goto L3
│     013: L2
│     014: ldc "val is not 0"
│     015: L3
│     016: invokevirtual PrintStream.println
│     017: return
├──── 018: nop
├──── 019: nop
└──── 020: athrow
```

#### 逻辑上的dead code

上面介绍的两种解决问题是没有办法处理代码逻辑上的dead code的：

```java
public class HelloWorld {
  public void test(int val) {
    if (val > 0) {
      if (val < -1) {
        // 这里是不可能执行的
        System.out.println(val);
      }
    }
  }
}
```

### 4.5.4 总结

本文内容总结如下：

- 第一点，移除dead code的识别标志：如果某一条指令（instruction）是dead code，那么它对应的frame是`null`。
- 第二点，如果不借助于frame信息，那么我们可以使用`Analyzer`类的control flow analysis部分进行实现。
- 第三点，当前介绍的两种解决方法有局限性，不能处理代码逻辑上的dead code。

## 4.6 BasicValue-BasicVerifier

在本章内容当中，最核心的内容就是下面两行代码。这两行代码包含了`asm-analysis.jar`当中`Analyzer`、`Frame`、`Interpreter`和`Value`最重要的四个类：

```
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicVerifier());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

在本文当中，我们将介绍`BasicVerifier`类：

```
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

在上面这个表当中，我们关注以下三点：

- 第一点，类的继承关系。`BasicVerifier`类继承自`BasicInterpreter`类，而`BasicInterpreter`类继承自`Interpreter`抽象类。另外，`SimpleVerifier`类是当前`BasicVerifier`类的子类。
- 第二点，类的合作关系。`BasicVerifier`与`BasicValue`类是一起使用的。
- 第三点，类的表达能力。`BasicVerifier`类能够使用的`BasicValue`对象有7个，也就是`BasicValue`类定义的7个静态字段值。

值得注意的是，`BasicInterpreter`类和`BasicVerifier`类为什么会使用相同的`BasicValue`类定义的7个静态字段呢？因为`BasicVerifier`类沿用了`BasicInterpreter`类的`newValue()`方法；而`newValue()`就是生产`Value`对象的工厂。相应的，`SimpleVerifier`重新实现了`newValue()`方法，因此可以生成很多个`BasicValue`类的实例。

### 4.6.1 BasicVerifier

在介绍`BasicVerifier`类具体组成部分之前，我们来提出两个问题：

- 第一个问题，类名的含义是什么呢？
- 第二个问题，这个类的作用大吗？

首先，回答第一个问题。`BasicVerifier`类名当中带有verify（检查）的含义。那么，检查什么内容呢？（What）检查的是指令（instruction）使用的是否正确。怎么检查呢？（How）指令（instruction）自身会带有类型信息（期望类型），让它与local variable和operand stack当中的数据类型（实际类型）是否兼容。那么，理解`BasicVerifier`类重点就在于，看看它是如何实现类型检查（verify）的。

- For instance it checks that the operands of an `IADD` instruction are `BasicValue.INT_VALUE` values (while `BasicInterpreter`

   just returns the result, i.e. `BasicValue.INT_VALUE`).

  - 在`BasicVerifier`类当中，`binaryOperation`方法处理`IADD`指令时，会检查两个operand是否是`BasicValue.INT_VALUE`类型，然而再调用父类（`BasicInterpreter`）的实现。
  - 在`BasicInterpreter`类当中，`binaryOperation`方法处理`IADD`指令时，会直接返回`BasicValue.INT_VALUE`类型。

- For instance this class can detect that the `ISTORE 1 ALOAD 1` sequence is invalid.

  - 在`BasicVerifier`类当中，`copyOperation`方法处理`ISTORE`指令时，会检查参数`value`是否为`BasicValue.INT_VALUE`类型。
  - 在`BasicVerifier`类当中，`copyOperation`方法处理`ALOAD`指令时，会检查参数`value`是否为引用类型（`isReference()`方法）。

其次，回答第二个问题。从实际应用的角度来说，`BasicVerifier`类的价值不大。为什么价值不大呢？因为它只能使用`BasicValue`所定义的7个静态字段值，对问题的表达能力非常有限，能够检查的也是简单的错误。那么，我们着重，学习和模仿`BasicVerifier`检查类型的处理思路。

#### class info

第一个部分，`BasicVerifier`类继承自`BasicInterpreter`类。

```java
/**
 * An extended {@link BasicInterpreter} that checks that bytecode instructions are correctly used.
 *
 * @author Eric Bruneton
 * @author Bing Ran
 */
public class BasicVerifier extends BasicInterpreter {
}
```

#### fields

第二个部分，`BasicVerifier`类定义的字段有哪些。

```java
public class BasicVerifier extends BasicInterpreter {
    // 没有定义字段
}
```

#### constructors

第三个部分，`BasicVerifier`类定义的构造方法有哪些。

```java
public class BasicVerifier extends BasicInterpreter {
  public BasicVerifier() {
    super(ASM9);
    if (getClass() != BasicVerifier.class) {
      throw new IllegalStateException();
    }
  }

  protected BasicVerifier(int api) {
    super(api);
  }
}
```

#### methods

第四个部分，`BasicVerifier`类定义的方法有哪些。

##### opcode相关的方法

在`Interpreter`类当中，定义了7个与opcode相关的抽象方法（abstract method）：

1. newOperation
2. copyOperation
3. unaryOperation
4. binaryOperation
5. ternaryOperation
6. naryOperation
7. returnOperation

在`BasicInterpreter`类当中，对这7个方法进行了实现。

在`BasicVerifier`类当中，对6个方法进行了重新实现，有1个方法（`newOperation`）沿用了`BasicInterpreter`类的实现。

在这6个重新实现的方法当中，它们的整体思路是一样的：将期望值（期望类型）和实际值（实际类型）进行比较。这里以`copyOperation`方法为例，其中`value`是实际值，`expected`是期望值，如果两者不匹配，则抛出`AnalyzerException`类型的异常。

```java
public class BasicVerifier extends BasicInterpreter {
  @Override
  public BasicValue copyOperation(final AbstractInsnNode insn, final BasicValue value) throws AnalyzerException {
    Value expected;
    switch (insn.getOpcode()) {
      case ILOAD:
      case ISTORE:
        expected = BasicValue.INT_VALUE;
        break;
      case FLOAD:
      case FSTORE:
        expected = BasicValue.FLOAT_VALUE;
        break;
      case LLOAD:
      case LSTORE:
        expected = BasicValue.LONG_VALUE;
        break;
      case DLOAD:
      case DSTORE:
        expected = BasicValue.DOUBLE_VALUE;
        break;
      case ALOAD:
        if (!value.isReference()) {
          throw new AnalyzerException(insn, null, "an object reference", value);
        }
        return value;
      case ASTORE:
        if (!value.isReference() && !BasicValue.RETURNADDRESS_VALUE.equals(value)) {
          throw new AnalyzerException(insn, null, "an object reference or a return address", value);
        }
        return value;
      default:
        return value;
    }
    if (!expected.equals(value)) {
      throw new AnalyzerException(insn, null, expected, value);
    }
    return value;
  }
}
```

##### 自定义的protected方法

在`BasicVerifier`类当中，它除了继承自`Interpreter`的方法之外，还定义了一些自己的`protected`方法。虽然这些`protected`方法比较简单，但是子类（例如，`SimpleVerifier`）可以对这些方法进行扩展。

```java
public class BasicVerifier extends BasicInterpreter {
  protected boolean isArrayValue(final BasicValue value) {
    return value.isReference();
  }
  protected BasicValue getElementValue(final BasicValue objectArrayValue) throws AnalyzerException {
    return BasicValue.REFERENCE_VALUE;
  }
  protected boolean isSubTypeOf(final BasicValue value, final BasicValue expected) {
    return value.equals(expected);
  }
}
```

数组类型：`BasicVerifier.isArrayValue(BasicValue value)`和`BasicVerifier.getElementValue(BasicValue objectArrayValue)`方法

- 遇到`arraylength`指令时，判断operand stack栈顶是不是一个数组。
- 遇到`aaload`指令时，根据当前的数组类型，获取其中的元素类型。

两个类型之间的兼容性：`BasicVerifier.isSubTypeOf(BasicValue value, BasicValue expected)`

- 遇到`invokevirtual`等指令时，实际返回的类型和方法的返回值类型之间是否兼容。

引用类型：`BasicValue.isReference()`

- 遇到`ifnull`或`areturn`这类指令的时候，它需要一个引用类型的值，不能是primitive type类型的值。

### 4.6.2 示例：检测不合理指令组合

假如我们想实现下面的`test`方法：

```java
package sample;

public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a + b;
  }
}
```

我们可以使用`BasicVerifier`来检验出`ISTORE 1 ALOAD 1`指令组合是不合理的。

```java
import org.objectweb.asm.tree.*;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.analysis.*;

import static org.objectweb.asm.Opcodes.*;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    MethodNode mn = new MethodNode(ACC_PUBLIC, "test", "()V", null, null);

    InsnList il = mn.instructions;
    il.add(new InsnNode(ICONST_1));
    il.add(new VarInsnNode(ISTORE, 1));
    il.add(new InsnNode(ICONST_2));
    il.add(new VarInsnNode(ISTORE, 2));
    il.add(new VarInsnNode(ILOAD, 1)); // 将ILOAD替换成ALOAD，就会报错
    il.add(new VarInsnNode(ILOAD, 2));
    il.add(new InsnNode(IADD));
    il.add(new VarInsnNode(ISTORE, 3));
    il.add(new InsnNode(RETURN));

    mn.maxStack = 2;
    mn.maxLocals = 4;

    Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicVerifier());
    analyzer.analyze("sample/HelloWorld", mn);
  }
}
```

### 4.6.3 总结

本文内容总结如下：

- 第一点，介绍`BasicVerifier`类，它是属于`Interpreter`的部分，它的特点是进行类型检查。
  - 进行类型检查的方式：将一个实际值和一个期望值进行比较，判断两者是否兼容（对应于`isSubTypeOf()`方法，或者相等，或者说父类和子类的关系）；如果兼容，则表示类型没有错误；如果不兼容，则表示类型出现错误。
- 第二点，代码示例，学习`BasicVerifier`检查类型的思路。

## 4.7 BasicValue-SimpleVerifier

在本章内容当中，最核心的内容就是下面两行代码。这两行代码包含了`asm-analysis.jar`当中`Analyzer`、`Frame`、`Interpreter`和`Value`最重要的四个类：

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new SimpleVerifier());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

在本文当中，我们将介绍`SimpleVerifier`类：

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

在上面这个表当中，我们关注以下三点：

- 第一点，类的继承关系。`SimpleVerifier`类继承自`BasicVerifier`类，`BasicVerifier`类继承自`BasicInterpreter`类，而`BasicInterpreter`类继承自`Interpreter`抽象类。
- 第二点，类的合作关系。`SimpleVerifier`与`BasicValue`类是一起使用的。
- 第三点，类的表达能力。`SimpleVerifier`类能够使用的`BasicValue`对象有很多个。因为`SimpleVerifier`类重新实现了`newValue()`方法。每个类都有自己的表示形式。

The `SimpleVerifier` class extends the `BasicVerifier` class. It uses **more sets** to simulate the execution of bytecode instructions: indeed **each class is represented by its own set, representing all possible objects of this class**.

This class uses the **Java reflection API** in order to perform verifications and computations related to the class hierarchy. It therefore loads the classes referenced by a method into the JVM. This default behavior can be changed by overriding the protected methods of this class.

### 4.7.1 SimpleVerifier

理解`SimpleVerifier`类，从两个方面来把握：

- 第一点，`SimpleVerifier`类可以模拟的`Value`值有多少个？为什么关注这个问题呢？因为这些`Value`值会存储在`Frame`内，它体现了`Frame`的表达能力。
- 第二点，`SimpleVerifier`类如何实现验证（Verify）的功能？

#### class info

第一个部分，`SimpleVerifier`类继承自`BasicVerifier`类。

```java
/**
 * An extended {@link BasicVerifier} that performs more precise verifications. This verifier
 * computes exact class types, instead of using a single "object reference" type (as done in {@link
 * BasicVerifier}).
 *
 * @author Eric Bruneton
 * @author Bing Ran
 */
public class SimpleVerifier extends BasicVerifier {
}
```

#### fields

第二个部分，`SimpleVerifier`类定义的字段有哪些。

```java
public class SimpleVerifier extends BasicVerifier {
  // 第一组字段，当前类、父类、接口
  private Type currentClass;
  private Type currentSuperClass;
  private List<Type> currentClassInterfaces;
  private boolean isInterface;

  // 第二组字段，ClassLoader
  private ClassLoader loader = getClass().getClassLoader();
}
```

第一组字段，是记录当前类的相关信息。同时，我们也要注意到`Analyzer.analyze(owner, mn)`方法只提供了当前类的名字（`owner`），提供的类相关信息太少；而`SimpleVerifier`这个类记录了当前类、父类、接口和当前类是不是接口的信息。

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │                                   │
   │        └── Value                           └── 只提供了当前类的名字信息
   └── Frame
```

第二组字段，`loader`字段通过反射的方式将某个类加载进来，从而进一步判断类的继承关系。另外，`SimpleVerifier`也提供了一个`setter`方法来设置`loader`字段的值。

```java
public class SimpleVerifier extends BasicVerifier {
  public void setClassLoader(ClassLoader loader) {
    this.loader = loader;
  }    
}
```

#### constructors

第三个部分，`SimpleVerifier`类定义的构造方法有哪些。

注意，前三个构造方法都不适合由子类继承，因为它会判断`getClass() != SimpleVerifier.class`是否成立；如果是子类实现，就会抛出`IllegalStateException`类型的异常。

```java
public class SimpleVerifier extends BasicVerifier {
  public SimpleVerifier() {
    this(null, null, false);
  }

  public SimpleVerifier(Type currentClass, Type currentSuperClass, boolean isInterface) {
    this(currentClass, currentSuperClass, null, isInterface);
  }

  public SimpleVerifier(Type currentClass, Type currentSuperClass, List<Type> currentClassInterfaces, boolean isInterface) {
    this(ASM9, currentClass, currentSuperClass, currentClassInterfaces, isInterface);
    if (getClass() != SimpleVerifier.class) {
      throw new IllegalStateException();
    }
  }

  protected SimpleVerifier(int api, Type currentClass, Type currentSuperClass, List<Type> currentClassInterfaces, boolean isInterface) {
    super(api);
    this.currentClass = currentClass;
    this.currentSuperClass = currentSuperClass;
    this.currentClassInterfaces = currentClassInterfaces;
    this.isInterface = isInterface;
  }
}
```

#### methods

第四个部分，`SimpleVerifier`类定义的方法有哪些。

##### Interpreter.newValue方法

这个`newValue`方法原本是在`Interpreter`类里定义的。

在下面的表当中，我们可以看到：

- 在`BasicInterpreter`和`BasicVerifier`类当中，可以使用的`BasicValue`值有`7`个。
- 在`SimpleVerifier`类当中，可以使用的`BasicValue`值有`N`个。

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

那么，为什么`SimpleVerifier`类可以使用更多的`BasicValue`值呢？原因就在于`newValue`方法。我们从两点来把握：

- 第一点，在`SimpleVerifier`类当中，`newValue`方法使用`new BasicValue(Type)`创建新的对象，因此使得`BasicValue`值多样化，每个不同的类都有一个对应的`BasicValue`值。
- 第二点，<u>在`SimpleVerifier`类当中，`new BasicValue(Type)`的调用只在`newValue`方法中发生，不会在其它方法中调用。也就是说，创建新的`BasicValue`值，只会在`newValue`方法发生</u>。

```java
public class SimpleVerifier extends BasicVerifier {
  @Override
  public BasicValue newValue(Type type) {
    if (type == null) {
      return BasicValue.UNINITIALIZED_VALUE;
    }

    boolean isArray = type.getSort() == Type.ARRAY;
    if (isArray) {
      switch (type.getElementType().getSort()) {
        case Type.BOOLEAN:
        case Type.CHAR:
        case Type.BYTE:
        case Type.SHORT:
          return new BasicValue(type);
        default:
          break;
      }
    }

    BasicValue value = super.newValue(type);
    if (BasicValue.REFERENCE_VALUE.equals(value)) {
      if (isArray) {
        value = newValue(type.getElementType());
        StringBuilder descriptor = new StringBuilder();
        for (int i = 0; i < type.getDimensions(); ++i) {
          descriptor.append('[');
        }
        descriptor.append(value.getType().getDescriptor());
        value = new BasicValue(Type.getType(descriptor.toString()));
      } else {
        value = new BasicValue(type);
      }
    }
    return value;
  }
}
```

##### Interpreter.merge方法

其实，`newValue`和`merge`方法都是`Interpreter`类所定义的方法。

- 在`Interpreter`类当中，`merge`方法是一个抽象方法。
- 在`BasicInterpreter`类当中，为`merge`方法提供了一个简单实现。
- 在`BasicVerifier`类当中，继承了`BasicInterpreter`类的`merge`方法没有做任何改变。
- 在`SimpleVerifier`类当中，对`merge`方法的代码逻辑进行了重新编写。

在`BasicInterpreter`和`BasicVerifier`类当中，只使用7个`BasicValue`值，因此`merge`方法实现起来很简单。然而，在`SimpleVerifier`类当中，可以使用N个`BasicValue`值；也就是说，每个类型都有一个对应的`BasicValue`值，那么相应的merge操作就变得复杂起来了。

```java
public class SimpleVerifier extends BasicVerifier {
  @Override
  public BasicValue merge(BasicValue value1, BasicValue value2) {
    if (!value1.equals(value2)) {
      Type type1 = value1.getType();
      Type type2 = value2.getType();
      if (type1 != null
          && (type1.getSort() == Type.OBJECT || type1.getSort() == Type.ARRAY)
          && type2 != null
          && (type2.getSort() == Type.OBJECT || type2.getSort() == Type.ARRAY)) {
        if (type1.equals(NULL_TYPE)) {
          return value2;
        }
        if (type2.equals(NULL_TYPE)) {
          return value1;
        }
        if (isAssignableFrom(type1, type2)) {
          return value1;
        }
        if (isAssignableFrom(type2, type1)) {
          return value2;
        }
        int numDimensions = 0;
        if (type1.getSort() == Type.ARRAY
            && type2.getSort() == Type.ARRAY
            && type1.getDimensions() == type2.getDimensions()
            && type1.getElementType().getSort() == Type.OBJECT
            && type2.getElementType().getSort() == Type.OBJECT) {
          numDimensions = type1.getDimensions();
          type1 = type1.getElementType();
          type2 = type2.getElementType();
        }
        while (true) {
          if (type1 == null || isInterface(type1)) {
            return newArrayValue(Type.getObjectType("java/lang/Object"), numDimensions);
          }
          type1 = getSuperClass(type1);
          if (isAssignableFrom(type1, type2)) {
            return newArrayValue(type1, numDimensions);
          }
        }
      }
      return BasicValue.UNINITIALIZED_VALUE;
    }
    return value1;
  }    
}
```

##### BasicVerifier方法

下面几个方法是从`BasicVerifier`继承来的方法

```java
public class SimpleVerifier extends BasicVerifier {
  @Override
  protected boolean isArrayValue(BasicValue value) {
    Type type = value.getType();
    return type != null && (type.getSort() == Type.ARRAY || type.equals(NULL_TYPE));
  }

  @Override
  protected BasicValue getElementValue(BasicValue objectArrayValue) throws AnalyzerException {
    Type arrayType = objectArrayValue.getType();
    if (arrayType != null) {
      if (arrayType.getSort() == Type.ARRAY) {
        return newValue(Type.getType(arrayType.getDescriptor().substring(1)));
      } else if (arrayType.equals(NULL_TYPE)) {
        return objectArrayValue;
      }
    }
    throw new AssertionError();
  }

  @Override
  protected boolean isSubTypeOf(BasicValue value, BasicValue expected) {
    Type expectedType = expected.getType();
    Type type = value.getType();
    switch (expectedType.getSort()) {
      case Type.INT:
      case Type.FLOAT:
      case Type.LONG:
      case Type.DOUBLE:
        return type.equals(expectedType);
      case Type.ARRAY:
      case Type.OBJECT:
        // 注意 null 类型也算是引用类型 (毕竟primitive type不允许null值，只有reference type允许null值)
        if (type.equals(NULL_TYPE)) {
          return true;
        } else if (type.getSort() == Type.OBJECT || type.getSort() == Type.ARRAY) {
          if (isAssignableFrom(expectedType, type)) {
            return true;
          } else if (getClass(expectedType).isInterface()) {
            // The merge of class or interface types can only yield class types (because it is not
            // possible in general to find an unambiguous common super interface, due to multiple
            // inheritance). Because of this limitation, we need to relax the subtyping check here
            // if 'value' is an interface.
            return Object.class.isAssignableFrom(getClass(type));
          } else {
            return false;
          }
        } else {
          return false;
        }
      default:
        throw new AssertionError();
    }
  }
}
```

##### protected方法

下面几个方法是`SimpleVerifier`自定义的`protected`方法：

- `isAssignableFrom`方法：判断两个`Type`之间是否兼容。
- `isInterface`方法：判断某个`Type`是否为接口。
- `getSuperClass`方法：获取某个`Type`的父类型。
- `getClass`方法：通过反射的方式加载某个`Type`类型，是`loader`字段发挥作用的地方。另外，`isAssignableFrom`、`isInterface`和`getSuperClass`方法都会调用`getClass`方法。

```java
public class SimpleVerifier extends BasicVerifier {
  protected boolean isAssignableFrom(Type type1, Type type2) {
    if (type1.equals(type2)) {
      return true;
    }
    if (currentClass != null && currentClass.equals(type1)) {
      if (getSuperClass(type2) == null) {
        return false;
      } else {
        if (isInterface) {
          return type2.getSort() == Type.OBJECT || type2.getSort() == Type.ARRAY;
        }
        return isAssignableFrom(type1, getSuperClass(type2));
      }
    }
    if (currentClass != null && currentClass.equals(type2)) {
      if (isAssignableFrom(type1, currentSuperClass)) {
        return true;
      }
      if (currentClassInterfaces != null) {
        for (Type currentClassInterface : currentClassInterfaces) {
          if (isAssignableFrom(type1, currentClassInterface)) {
            return true;
          }
        }
      }
      return false;
    }
    return getClass(type1).isAssignableFrom(getClass(type2));
  }

  protected boolean isInterface(Type type) {
    if (currentClass != null && currentClass.equals(type)) {
      return isInterface;
    }
    return getClass(type).isInterface();
  }

  protected Type getSuperClass(Type type) {
    if (currentClass != null && currentClass.equals(type)) {
      return currentSuperClass;
    }
    Class<?> superClass = getClass(type).getSuperclass();
    return superClass == null ? null : Type.getType(superClass);
  }

  protected Class<?> getClass(Type type) {
    try {
      if (type.getSort() == Type.ARRAY) {
        return Class.forName(type.getDescriptor().replace('/', '.'), false, loader);
      }
      return Class.forName(type.getClassName(), false, loader);
    } catch (ClassNotFoundException e) {
      throw new TypeNotPresentException(e.toString(), e);
    }
  }    
}
```

### 4.7.2 SimpleVerifier的表达能力

#### primitive type无法区分

SimpleVerifier的表达能力可以描述成这样：

- 可以区分不同的引用类型（Reference Type），例如`String`、`Object`类型
- 可以区分同一个引用类型的不同对象实例，例如”AAA”和”BBB”是`String`类型的不同对象实例
- 但是，不能够区分同一种primitive type的不同值

例如，下面的`HelloWorld`类当中，`str1`和`str2`这两个变量是`String`类型，它们分别使用不同的`BasicValue`对象来表示；相应的，`a`和`b`都是`int`类型（primitive type），它们分别是`1`和`2`两个值，但是它们都是用`BasicValue.INT_VALUE`来表示，没有办法进行区分。

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    String str1 = "AAA";
    String str2 = "BBB";
  }
}
```

#### 如何验证

为了验证上面的内容是否正确，我们可以使用`HelloWorldFrameTree`类查看frame当中存储的数据：看一看`int`类型的`a`和`b`能不能区分，看一个`String`类型的`str1`和`str2`能不能区分开？

第一次尝试（错误示例，不能区分）：

```java
print(owner, mn, new SimpleVerifier(), null);
```

输出结果：`a`和`b`都用`I`表示，`str1`和`str2`都用`Ljava/lang/String;`，看不出`str1`和`str2`有什么区别

```shell
test:()V
000:    iconst_1                                {Lsample/HelloWorld;, ., ., ., .} | {}
001:    istore_1                                {Lsample/HelloWorld;, ., ., ., .} | {I}
002:    iconst_2                                {Lsample/HelloWorld;, I, ., ., .} | {}
003:    istore_2                                {Lsample/HelloWorld;, I, ., ., .} | {I}
004:    ldc "AAA"                               {Lsample/HelloWorld;, I, I, ., .} | {}
005:    astore_3                                {Lsample/HelloWorld;, I, I, ., .} | {Ljava/lang/String;}
006:    ldc "BBB"                               {Lsample/HelloWorld;, I, I, Ljava/lang/String;, .} | {}
007:    astore 4                                {Lsample/HelloWorld;, I, I, Ljava/lang/String;, .} | {Ljava/lang/String;}
008:    return                                  {Lsample/HelloWorld;, I, I, Ljava/lang/String;, Ljava/lang/String;} | {}
================================================================
```

第二次尝试（错误示例，不能区分）：

```java
print(owner, mn, new SimpleVerifier(), item -> item.toString() + "@" + item.hashCode());
```

输出结果：`a`和`b`都用`I@65`表示，`str1`和`str2`都用`Ljava/lang/String;@-689322901`，看不出`str1`和`str2`有什么区别

```shell
test:()V
000:    iconst_1                                {Lsample/HelloWorld;@1649535039, .@0, .@0, .@0, .@0} | {}
001:    istore_1                                {Lsample/HelloWorld;@1649535039, .@0, .@0, .@0, .@0} | {I@65}
002:    iconst_2                                {Lsample/HelloWorld;@1649535039, I@65, .@0, .@0, .@0} | {}
003:    istore_2                                {Lsample/HelloWorld;@1649535039, I@65, .@0, .@0, .@0} | {I@65}
004:    ldc "AAA"                               {Lsample/HelloWorld;@1649535039, I@65, I@65, .@0, .@0} | {}
005:    astore_3                                {Lsample/HelloWorld;@1649535039, I@65, I@65, .@0, .@0} | {Ljava/lang/String;@-689322901}
006:    ldc "BBB"                               {Lsample/HelloWorld;@1649535039, I@65, I@65, Ljava/lang/String;@-689322901, .@0} | {}
007:    astore 4                                {Lsample/HelloWorld;@1649535039, I@65, I@65, Ljava/lang/String;@-689322901, .@0} | {Ljava/lang/String;@-689322901}
008:    return                                  {Lsample/HelloWorld;@1649535039, I@65, I@65, Ljava/lang/String;@-689322901, Ljava/lang/String;@-689322901} | {}
================================================================
```

那么，为什么第二次尝试是错误的呢？是因为`BasicValue.hashCode()`是经过修改的，它会进一步调用`Type.hashCode()`；而`Type.hashCode()`会根据descritor来计算hash值，只要descriptor相同，那么hash值就相同。

第三次尝试（正确示例，能够区分）：借助于`System.identityHashCode()`方法

```java
print(owner, mn, new SimpleVerifier(), item -> item.toString() + "@" + System.identityHashCode(item));
```

输出结果：`a`和`b`都用`I@1267032364`表示，`str1`用`Ljava/lang/String;@661672156`表示，`str2`都用`Ljava/lang/String;@96639997`，可以看出`str1`和`str2`是不同的对象。

```shell
test:()V
000:    iconst_1                                {Lsample/HelloWorld;@1147985808, .@2040495657, .@2040495657, .@2040495657, .@2040495657} | {}
001:    istore_1                                {Lsample/HelloWorld;@1147985808, .@2040495657, .@2040495657, .@2040495657, .@2040495657} | {I@1267032364}
002:    iconst_2                                {Lsample/HelloWorld;@1147985808, I@1267032364, .@2040495657, .@2040495657, .@2040495657} | {}
003:    istore_2                                {Lsample/HelloWorld;@1147985808, I@1267032364, .@2040495657, .@2040495657, .@2040495657} | {I@1267032364}
004:    ldc "AAA"                               {Lsample/HelloWorld;@1147985808, I@1267032364, I@1267032364, .@2040495657, .@2040495657} | {}
005:    astore_3                                {Lsample/HelloWorld;@1147985808, I@1267032364, I@1267032364, .@2040495657, .@2040495657} | {Ljava/lang/String;@661672156}
006:    ldc "BBB"                               {Lsample/HelloWorld;@1147985808, I@1267032364, I@1267032364, Ljava/lang/String;@661672156, .@2040495657} | {}
007:    astore 4                                {Lsample/HelloWorld;@1147985808, I@1267032364, I@1267032364, Ljava/lang/String;@661672156, .@2040495657} | {Ljava/lang/String;@96639997}
008:    return                                  {Lsample/HelloWorld;@1147985808, I@1267032364, I@1267032364, Ljava/lang/String;@661672156, Ljava/lang/String;@96639997} | {}
================================================================
```

那么，我们为什么要将三次尝试都记录下来呢？因为大家在自己尝试的过程当中，可能也会想去确定local variable和operand stack上的某两个位置的值到底是不是同一个元素呢？如果说具体的`Value`值修改过`hashCode()`方法，那么可能就检测不出来。**为了正确的检测两个位置的值是不是同一个对象，我们可以借助于`System.identityHashCode()`方法**。

> [hashCode和identityHashCode的区别你知道吗？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/62109977)
>
> + `System.identityHashCode()`永远返回根据对象物理内存地址产生的hash值。
> + 如果没重写`hashCode()`方法，那么`hashCode()`返回值会和`System.identityHashCode()`一致

### 4.7.3 总结

本文内容总结如下：

- 第一点，介绍`SimpleVerifier`类的各个部分。
- 第二点，理解`SimpleVerifier`类的表达能力。

## 4.8 BasicValue-SimpleVerifier示例：移除checkcast

### 4.8.1 示例：移除不必要的转换

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.2)
>
> `checkcast indexbyte1 indexbyte2`
>
> **Operand Stack**
>
> ```pseudocode
> ..., objectref →
> 
> ..., objectref
> ```
>
> **Operation**
>
> Check whether object is of given type
>
> **Description**
>
> The *objectref* must be of type `reference`. The unsigned *indexbyte1* and *indexbyte2* are used to construct an index into the run-time constant pool of the current class ([§2.6](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.6)), where the value of the index is (*indexbyte1* `<<` 8) | *indexbyte2*. <u>The run-time constant pool entry at the index must be a symbolic reference to a class, array, or interface type.</u>
>
> <u>If *objectref* is `null`, then the operand stack is unchanged.</u>
>
> Otherwise, the named class, array, or interface type is resolved ([§5.4.3.1](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.1)). If *objectref* can be cast to the resolved class, array, or interface type, the operand stack is unchanged; otherwise, the *checkcast* instruction throws a `ClassCastException`.
>
> The following rules are used to determine whether an *objectref* that is not `null` can be cast to the resolved type. If S is the type of the object referred to by *objectref*, and T is the resolved class, array, or interface type, then *checkcast* determines whether *objectref* can be cast to type T as follows:
>
> - If S is a class type, then:
>   - If T is a class type, then S must be the same class as T, or S must be a subclass of T;
>   - If T is an interface type, then S must implement interface T.
> - If S is an array type SC`[]`, that is, an array of components of type SC, then:
>   - If T is a class type, then T must be `Object`.
>   - If T is an interface type, then T must be one of the interfaces implemented by arrays (JLS §4.10.3, [Chapter 4. Types, Values, and Variables (oracle.com)](https://docs.oracle.com/javase/specs/jls/se20/html/jls-4.html#jls-4.10.3)).
>   - If T is an array type TC`[]`, that is, an array of components of type TC, then one of the following must be true:
>     - TC and SC are the same primitive type.
>     - TC and SC are reference types, and type SC can be cast to TC by recursive application of these rules.
>
> **Linking Exceptions**
>
> During resolution of the symbolic reference to the class, array, or interface type, any of the exceptions documented in [§5.4.3.1](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-5.html#jvms-5.4.3.1) can be thrown.
>
> **Run-time Exception**
>
> Otherwise, if *objectref* cannot be cast to the resolved class, array, or interface type, the *checkcast* instruction throws a `ClassCastException`.
>
> **Notes**
>
> The *checkcast* instruction is very similar to the *instanceof* instruction ([§*instanceof*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.instanceof)). It differs in its treatment of `null`, its behavior when its test fails (*checkcast* throws an exception, *instanceof* pushes a result code), and its effect on the operand stack.

#### 预期目标

```java
public class HelloWorld {
  public void test() {
    Object obj = "ABC";
    String val = (String) obj;
    System.out.println(val);
  }
}
```

我们可以使用`javap`指令查看`test`方法包含的instructions内容：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #2                  // String ABC
       2: astore_1
       3: aload_1
       4: checkcast     #3                  // class java/lang/String
       7: astore_2
       8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: aload_2
      12: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      15: return
}
```

我们想实现的目标：移除不必要的`checkcast`指令。

```shell
test:()V
000:    ldc "ABC"                               {HelloWorld, ., .} | {}
001:    astore_1                                {HelloWorld, ., .} | {String}
002:    aload_1                                 {HelloWorld, String, .} | {}
003:    checkcast String                        {HelloWorld, String, .} | {String}
004:    astore_2                                {HelloWorld, String, .} | {String}
005:    getstatic System.out                    {HelloWorld, String, String} | {}
006:    aload_2                                 {HelloWorld, String, String} | {PrintStream}
007:    invokevirtual PrintStream.println       {HelloWorld, String, String} | {PrintStream, String}
008:    return                                  {HelloWorld, String, String} | {}
================================================================
```

#### 编码实现

```java
import lsieun.asm.tree.transformer.MethodTransformer;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.Type;
import org.objectweb.asm.tree.*;
import org.objectweb.asm.tree.analysis.*;

import static org.objectweb.asm.Opcodes.CHECKCAST;

public class RemoveUnusedCastNode extends ClassNode {
  public RemoveUnusedCastNode(int api, ClassVisitor cv) {
    super(api);
    this.cv = cv;
  }

  @Override
  public void visitEnd() {
    // 首先，处理自己的代码逻辑
    MethodTransformer mt = new MethodRemoveUnusedCastTransformer(name, null);
    for (MethodNode mn : methods) {
      if ("<init>".equals(mn.name) || "<clinit>".equals(mn.name)) {
        continue;
      }
      InsnList insns = mn.instructions;
      if (insns.size() == 0) {
        continue;
      }
      mt.transform(mn);
    }

    // 其次，调用父类的方法实现
    super.visitEnd();

    // 最后，向后续ClassVisitor传递
    if (cv != null) {
      accept(cv);
    }
  }

  private static class MethodRemoveUnusedCastTransformer extends MethodTransformer {
    private final String owner;

    public MethodRemoveUnusedCastTransformer(String owner, MethodTransformer mt) {
      super(mt);
      this.owner = owner;
    }

    @Override
    public void transform(MethodNode mn) {
      // 首先，处理自己的代码逻辑
      Analyzer<BasicValue> analyzer = new Analyzer<>(new SimpleVerifier());
      try {
        Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
        AbstractInsnNode[] insnNodes = mn.instructions.toArray();
        for (int i = 0; i < insnNodes.length; i++) {
          AbstractInsnNode insn = insnNodes[i];
          if (insn.getOpcode() == CHECKCAST) {
            Frame<BasicValue> f = frames[i];
            if (f != null && f.getStackSize() > 0) {
              BasicValue operand = f.getStack(f.getStackSize() - 1);
              Class<?> to = getClass(((TypeInsnNode) insn).desc);
              Class<?> from = getClass(operand.getType());
              if (to.isAssignableFrom(from)) {
                // 如果验证确实可转换，那么移除 checkcast 指令
                mn.instructions.remove(insn);
              }
            }
          }
        }
      }
      catch (AnalyzerException ex) {
        ex.printStackTrace();
      }

      // 其次，调用父类的方法实现
      super.transform(mn);
    }

    private static Class<?> getClass(String desc) {
      try {
        return Class.forName(desc.replace('/', '.'));
      }
      catch (ClassNotFoundException ex) {
        throw new RuntimeException(ex.toString());
      }
    }

    private static Class<?> getClass(Type t) {
      if (t.getSort() == Type.OBJECT) {
        return getClass(t.getInternalName());
      }
      return getClass(t.getDescriptor());
    }
  }
}
```

#### 进行转换

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.*;

public class HelloWorldTransformTree {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);

    // (1)构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    // (2)构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    // (3)串连ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new RemoveUnusedCastNode(api, cw);

    //（4）结合ClassReader和ClassNode
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    // (5) 生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

#### 验证结果

一方面，验证`test`方法的功能是否正常：

```java
import java.lang.reflect.Method;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    Class<?> clazz = Class.forName("sample.HelloWorld");
    Object instance = clazz.newInstance();
    Method m = clazz.getDeclaredMethod("test");
    m.invoke(instance);
  }
}
```

另一方面，验证`test`方法是否包含`checkcast`指令：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #11                 // String ABC
       2: astore_1
       3: aload_1
       4: astore_2
       5: getstatic     #17                 // Field java/lang/System.out:Ljava/io/PrintStream;
       8: aload_2
       9: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: return
}
```

### 4.8.2 解决方式二：使用Core API

这个实现主要是依赖于Core API当中介绍的`AnalyzerAdapter`类。

#### 编码实现

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.commons.AnalyzerAdapter;

import static org.objectweb.asm.Opcodes.*;

public class RemoveUnusedCastVisitor extends ClassVisitor {
  private String owner;

  public RemoveUnusedCastVisitor(int api, ClassVisitor classVisitor) {
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
    if (mv != null && !name.equals("<init>")) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        RemoveUnusedCastAdapter adapter = new RemoveUnusedCastAdapter(api, mv);
        adapter.aa = new AnalyzerAdapter(owner, access, name, descriptor, adapter);
        return adapter.aa;
      }
    }

    return mv;
  }

  private static class RemoveUnusedCastAdapter extends MethodVisitor {
    public AnalyzerAdapter aa;

    public RemoveUnusedCastAdapter(int api, MethodVisitor methodVisitor) {
      super(api, methodVisitor);
    }

    @Override
    public void visitTypeInsn(int opcode, String type) {
      if (opcode == CHECKCAST) {
        Class<?> to = getClass(type);
        if (aa.stack != null && aa.stack.size() > 0) {
          Object operand = aa.stack.get(aa.stack.size() - 1);
          if (operand instanceof String) {
            Class<?> from = getClass((String) operand);
            if (to.isAssignableFrom(from)) {
              //直接return 即不向后传递指令，最后效果即删除 checkcast指令
              return;
            }
          }
        }
      }
      super.visitTypeInsn(opcode, type);
    }

    private static Class<?> getClass(String desc) {
      try {
        return Class.forName(desc.replace('/', '.'));
      }
      catch (ClassNotFoundException ex) {
        throw new RuntimeException(ex.toString());
      }
    }
  }
}
```

#### 进行转换

需要注意的地方是，在使用`AnalyzerAdapter`类时，要使用`ClassReader.EXPAND_FRAMES`选项。

```java
import lsieun.utils.FileUtils;
import org.objectweb.asm.*;

public class HelloWorldTransformCore {
  public static void main(String[] args) {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes1 = FileUtils.readBytes(filepath);
    if (bytes1 == null) {
      throw new RuntimeException("bytes1 is null");
    }

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes1);

    //（2）构建ClassWriter
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    //（3）串连ClassVisitor
    int api = Opcodes.ASM9;
    ClassVisitor cv = new RemoveUnusedCastVisitor(api, cw);

    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.EXPAND_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

### 4.8.3 总结

本文内容总结如下：

- 第一点，移除`checkcast`指令，从本质上来说，就是判断checkcast带有的“预期类型”和operand stack上栈顶的“实际类型”是否兼容。
- 第二点，虽然使用的分别是Tree API和Core API的内容来进行实现，但两者的本质是一样的逻辑。

## 4.9 BasicValue-SimpleVerifier示例：冗余变量分析

### 4.9.1 如何判断变量是否冗余

如果在IntelliJ IDEA当中编写如下的代码，它会提示`str2`和`str3`局部变量是多余的：

```java
public class HelloWorld {
  public void test() {
    String str1 = "Hello ASM";
    Object obj1 = new Object();
    // Local variable 'str2' is redundant
    String str2 = str1;
    Object obj2 = new Object();
    // Local variable 'str3' is redundant
    String str3 = str2;
    Object obj3 = new Object();
    int length = str3.length();
    System.out.println(length);
  }
}
```

#### 整体思路

结合`Analyzer`和`SimpleVerifier`类，我们可以查看Frame的变化情况：

```shell
test:()V
000:                         ldc "Hello ASM"    {HelloWorld, ., ., ., ., ., ., .} | {}
001:                                astore_1    {HelloWorld, ., ., ., ., ., ., .} | {String}
002:                              new Object    {HelloWorld, String, ., ., ., ., ., .} | {}
003:                                     dup    {HelloWorld, String, ., ., ., ., ., .} | {Object}
004:             invokespecial Object.<init>    {HelloWorld, String, ., ., ., ., ., .} | {Object, Object}
005:                                astore_2    {HelloWorld, String, ., ., ., ., ., .} | {Object}
006:                                 aload_1    {HelloWorld, String, Object, ., ., ., ., .} | {}
007:                                astore_3    {HelloWorld, String, Object, ., ., ., ., .} | {String}
008:                              new Object    {HelloWorld, String, Object, String, ., ., ., .} | {}
009:                                     dup    {HelloWorld, String, Object, String, ., ., ., .} | {Object}
010:             invokespecial Object.<init>    {HelloWorld, String, Object, String, ., ., ., .} | {Object, Object}
011:                                astore 4    {HelloWorld, String, Object, String, ., ., ., .} | {Object}
012:                                 aload_3    {HelloWorld, String, Object, String, Object, ., ., .} | {}
013:                                astore 5    {HelloWorld, String, Object, String, Object, ., ., .} | {String}
014:                              new Object    {HelloWorld, String, Object, String, Object, String, ., .} | {}
015:                                     dup    {HelloWorld, String, Object, String, Object, String, ., .} | {Object}
016:             invokespecial Object.<init>    {HelloWorld, String, Object, String, Object, String, ., .} | {Object, Object}
017:                                astore 6    {HelloWorld, String, Object, String, Object, String, ., .} | {Object}
018:                                 aload 5    {HelloWorld, String, Object, String, Object, String, Object, .} | {}
019:             invokevirtual String.length    {HelloWorld, String, Object, String, Object, String, Object, .} | {String}
020:                                istore 7    {HelloWorld, String, Object, String, Object, String, Object, .} | {I}
021:                    getstatic System.out    {HelloWorld, String, Object, String, Object, String, Object, I} | {}
022:                                 iload 7    {HelloWorld, String, Object, String, Object, String, Object, I} | {PrintStream}
023:       invokevirtual PrintStream.println    {HelloWorld, String, Object, String, Object, String, Object, I} | {PrintStream, I}
024:                                  return    {HelloWorld, String, Object, String, Object, String, Object, I} | {}
================================================================
```

我们的整体思路是这样的：

- 在每一个Frame当中，它有local variable和operand stack两部分组成。
- 程序中定义的“变量”是存储在local variable当中。
- 在理想的情况下，一个“变量”对应于local variable当中的一个位置；如果一个“变量”对应于local variable当中的两个或多个位置，那么我们就认为“变量”出现了冗余。

那么，针对某一个具体的frame，相应的实现思路上是这样的：

- 判断`local[0]`和`local[1]`是否相同，如果相同，那么表示`local[1]`是冗余的变量。
- 判断`local[0]`和`local[2]`是否相同，如果相同，那么表示`local[2]`是冗余的变量。
- …
- 判断`local[0]`和`local[n]`是否相同，如果相同，那么表示`local[n]`是冗余的变量。
- 判断`local[1]`和`local[2]`是否相同，如果相同，那么表示`local[2]`是冗余的变量。
- 判断`local[1]`和`local[3]`是否相同，如果相同，那么表示`local[3]`是冗余的变量。
- …

需要注意的一点就是，如果local variable当中存储“未初始化的值”（`BasicValue.UNINITIALIZED_VALUE`），那么我们就不进行处理了。

具体来说，“未初始化的值”（`BasicValue.UNINITIALIZED_VALUE`）有两种情况：

- 第一种情况，在方法刚进入的时候，local variable有些位置存储的就是“未初始化的值”（`BasicValue.UNINITIALIZED_VALUE`）。
- 第二种情况，在存储`long`和`double`类型的数据时，它占用两个位置，其中第二个位置就是“未初始化的值”（`BasicValue.UNINITIALIZED_VALUE`）。

#### 为什么选择SimpleVerifier

在ASM当中，`Interpreter`类是一个抽象类，其中提供的子类有`BasicInterpreter`、`BasicVerifier`、`SimpleVerifier`和`SourceInterpreter`类。那么，我们到底应该选择哪一个呢？

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

首先，不能选择`BasicInterpreter`和`BasicVerifier`类。因为它们使用7个值（`BasicValue`类定义的7个静态字段）来模拟Frame的变化，这7个值的“表达能力”很弱。如果一个对象是`String`类型，另一个对象是`Object`类型，这两个对象都会被表示成`BasicValue.REFERENCE_VALUE`，没有办法进行区分。

其次，不能选择`SourceInterpreter`类。因为它定义的`copyOperation`方法中会创建一个新的对象（`new SourceValue(value.getSize(), insn)`），不能识别为同一个对象。

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  @Override
  public SourceValue copyOperation(final AbstractInsnNode insn, final SourceValue value) {
    return new SourceValue(value.getSize(), insn);
  }
}
```

为什么要关注这个`copyOperation`方法呢？因为`copyOperation`方法负责处理load和store相关的指令。

```java
public abstract class Interpreter<V extends Value> {
    /**
     * Interprets a bytecode instruction that moves a value on the stack or to or from local variables.
     * This method is called for the following opcodes:
     *
     * ILOAD, LLOAD, FLOAD, DLOAD, ALOAD,
     * ISTORE, LSTORE, FSTORE, DSTORE, ASTORE,
     * DUP, DUP_X1, DUP_X2, DUP2, DUP2_X1, DUP2_X2, SWAP
     *
     */
    public abstract V copyOperation(AbstractInsnNode insn, V value) throws AnalyzerException;
}
```

最后，选择`SimpleVerifier`是合适的。一方面，它能区分不同的类型（class）、区分不同的对象实例（object instance）；另一方面，在`copyOperation`方法中保证了对象的一致性，传入的是`value`，返回的仍然是`value`。更准确的来说，`SimpleVerifier`是继承了父类`BasicVerifier`类的`copyOperation`方法。

```java
public class BasicVerifier extends BasicInterpreter {
  @Override
  public BasicValue copyOperation(final AbstractInsnNode insn, final BasicValue value)
    throws AnalyzerException {
    //...
    return value;
  }
}
```

### 4.9.2 示例：冗余变量分析

#### 预期目标

在下面的代码中，会提示`str2`和`str3`局部变量是多余的：

```java
public class HelloWorld {
  public void test() {
    String str1 = "Hello ASM";
    Object obj1 = new Object();
    // Local variable 'str2' is redundant
    String str2 = str1;
    Object obj2 = new Object();
    // Local variable 'str3' is redundant
    String str3 = str2;
    Object obj3 = new Object();
    int length = str3.length();
    System.out.println(length);
  }
}
```

我们的预期目标：识别出`str2`和`str3`是冗余变量。

#### 编码实现

```java
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.AbstractInsnNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.VarInsnNode;
import org.objectweb.asm.tree.analysis.*;

import java.util.Arrays;

public class RedundantVariableDiagnosis {
  public static int[] diagnose(String className, MethodNode mn) throws AnalyzerException {
    // 第一步，准备工作。使用SimpleVerifier进行分析，得到frames信息
    Analyzer<BasicValue> analyzer = new Analyzer<>(new SimpleVerifier());
    Frame<BasicValue>[] frames = analyzer.analyze(className, mn);

    // 第二步，利用frames信息，查看local variable当中哪些slot数据出现了冗余
    TIntArrayList localIndexList = new TIntArrayList();
    for (Frame<BasicValue> f : frames) {
      int locals = f.getLocals();
      for (int i = 0; i < locals; i++) {
        BasicValue val1 = f.getLocal(i);
        if (val1 == BasicValue.UNINITIALIZED_VALUE) {
          continue;
        }
        for (int j = i + 1; j < locals; j++) {
          BasicValue val2 = f.getLocal(j);
          if (val2 == BasicValue.UNINITIALIZED_VALUE) {
            continue;
          }
          if (val1 == val2) {
            if (!localIndexList.contains(j)) {
              localIndexList.add(j);
            }
          }
        }
      }
    }

    // 第三步，将slot的索引值（local index）转换成instruction的索引值（insn index）
    TIntArrayList insnIndexList = new TIntArrayList();
    InsnList instructions = mn.instructions;
    int size = instructions.size();
    for (int i = 0; i < size; i++) {
      AbstractInsnNode node = instructions.get(i);
      int opcode = node.getOpcode();
      if (opcode >= Opcodes.ISTORE && opcode <= Opcodes.ASTORE) {
        VarInsnNode varInsnNode = (VarInsnNode) node;
        if (localIndexList.contains(varInsnNode.var)) {
          if (!insnIndexList.contains(i)) {
            insnIndexList.add(i);
          }
        }
      }
    }

    // 第四步，将insnIndexList转换成int[]形式
    int[] array = insnIndexList.toNativeArray();
    Arrays.sort(array);
    return array;
  }
}
```

#### 进行分析

```java
public class HelloWorldAnalysisTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）进行分析
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);
    int[] array = RedundantVariableDiagnosis.diagnose(cn.name, mn);
    System.out.println(Arrays.toString(array));
    BoxDrawingUtils.printInstructionLinks(mn.instructions, array);
  }
}
```

输出结果：

```shell
[7, 13]
      000: ldc "Hello ASM"
      001: astore_1
      002: new Object
      003: dup
      004: invokespecial Object.<init>
      005: astore_2
      006: aload_1
┌──── 007: astore_3
│     008: new Object
│     009: dup
│     010: invokespecial Object.<init>
│     011: astore 4
│     012: aload_3
└──── 013: astore 5
      014: new Object
      015: dup
      016: invokespecial Object.<init>
      017: astore 6
      018: aload 5
      019: invokevirtual String.length
      020: istore 7
      021: getstatic System.out
      022: iload 7
      023: invokevirtual PrintStream.println
      024: return
```

### 4.9.3 测试用例

#### primitive type - no

本文介绍的方法不适合对primitive type进行分析：

- 所有`int`类型的值都用`BasicValue.INT_VALUE`表示，不能对两个不同的值进行区分
- 所有`float`类型的值都用`BasicValue.FLOAT_VALUE`表示，不能对两个不同的值进行区分
- 所有`long`类型的值都用`BasicValue.LONG_VALUE`表示，不能对两个不同的值进行区分
- 所有`double`类型的值都用`BasicValue.DOUBLE_VALUE`表示，不能对两个不同的值进行区分

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a + b;
    int d = a - b;
    int e = c * d;
    System.out.println(e);
  }
}
```

输出结果（错误）：

```shell
[3, 7, 11, 15]
      000: iconst_1
      001: istore_1
      002: iconst_2
┌──── 003: istore_2
│     004: iload_1
│     005: iload_2
│     006: iadd
├──── 007: istore_3
│     008: iload_1
│     009: iload_2
│     010: isub
├──── 011: istore 4
│     012: iload_3
│     013: iload 4
│     014: imul
└──── 015: istore 5
      016: getstatic System.out
      017: iload 5
      018: invokevirtual PrintStream.println
      019: return
```

#### return-no

本文介绍的方法也不适用于`return`语句的判断。在下面的代码中，会提示`result`局部变量是多余的：

```java
public class HelloWorld {
  public Object test() {
    // Local variable 'result' is redundant
    Object result = new Object();
    return result;
  }
}
```

我觉得，可以使用`astore aload areturn`的指令组合来识别这种情况，不一定要使用Frame的分析做到。

### 4.9.4 总结

本文内容总结如下：

- 第一点，如何判断一个变量是否冗余呢？看看local variable当中是否有两个或多个相同的值。
- 第二点，代码示例，编码实现冗余变量分析。

## 4.10 SourceValue-SourceInterpreter

在本章内容当中，最核心的内容就是下面两行代码。这两行代码包含了`asm-analysis.jar`当中`Analyzer`、`Frame`、`Interpreter`和`Value`最重要的四个类：

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<SourceValue> analyzer = new Analyzer<>(new SourceInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

在本文当中，我们将介绍`SourceInterpreter`和`SourceValue`类：

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

在上面这个表当中，我们关注以下三点：

- 第一点，类的继承关系。`SourceInterpreter`类继承自`Interpreter`抽象类。
- 第二点，类的合作关系。`SourceInterpreter`与`SourceValue`类是一起使用的。
- 第三点，类的表达能力。`SourceInterpreter`类能够使用的`SourceValue`对象有很多个。

### 4.10.1 SourceValue

#### class info

第一个部分，`SourceValue`类实现了`Value`接口。

```java
/**
 * A {@link Value} which keeps track of the bytecode instructions that can produce it.
 *
 * @author Eric Bruneton
 */
public class SourceValue implements Value {
}
```

#### fields

第二个部分，`SourceValue`类定义的字段有哪些。

```java
public class SourceValue implements Value {
  public int size;

  public Set<AbstractInsnNode> insns;
}
```

#### constructors

第三个部分，`SourceValue`类定义的构造方法有哪些。这三个构造方法，有不同的应用场景：

- `SourceValue(int)`构造方法，**在指令（Instruction）开始执行之前**，设置local variable的初始值，例如`this`、方法接收的参数，这些不需要指令（Instruction）参数。
- `SourceValue(int, AbstractInsnNode)`构造方法，**在指令（Instruction）开始执行之后**，记录一条指令（`AbstractInsnNode`）和对应的local variable、operand stack上的值（`SourceValue`）之间的关系。
- `SourceValue(int, Set<AbstractInsnNode>)`构造方法，在**`SourceValue`值进行合并（merge）的时候**，记录多条指令（`Set<AbstractInsnNode>`）和对应的local variable、operand stack上的值（`SourceValue`）之间的关系。

```java
public class SourceValue implements Value {
  public SourceValue(int size) {
    this(size, new SmallSet<AbstractInsnNode>());
  }

  public SourceValue(int size, AbstractInsnNode insnNode) {
    this.size = size;
    this.insns = new SmallSet<>(insnNode);
  }

  public SourceValue(int size, Set<AbstractInsnNode> insnSet) {
    this.size = size;
    this.insns = insnSet;
  }
}
```

#### methods

第四个部分，`SourceValue`类定义的方法有哪些。

```java
public class SourceValue implements Value {
  @Override
  public int getSize() {
    return size;
  }
}
```

### 4.10.2 SourceInterpreter

`SourceInterpreter`的主要作用是记录指令（instruction）与Frame当中值（`SourceValue`）的关联关系。

#### class info

第一个部分，`SourceInterpreter`类实现了`Interpreter`抽象类。

```java
/**
 * An {@link Interpreter} for {@link SourceValue} values.
 *
 * @author Eric Bruneton
 */
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
}
```

#### fields

第二个部分，`SourceInterpreter`类定义的字段有哪些。

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  // 没有定义字段
}
```

#### constructors

第三个部分，`SourceInterpreter`类定义的构造方法有哪些。

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  public SourceInterpreter() {
    super(ASM9);
    if (getClass() != SourceInterpreter.class) {
      throw new IllegalStateException();
    }
  }

  protected SourceInterpreter(int api) {
    super(api);
  }
}
```

#### methods

第四个部分，`SourceInterpreter`类定义的方法有哪些。

##### newValue方法

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  @Override
  public SourceValue newValue(Type type) {
    if (type == Type.VOID_TYPE) {
      return null;
    }
    // 这里是SourceValue定义的第1个构造方法，不需要指令参与
    return new SourceValue(type == null ? 1 : type.getSize());
  }
}
```

##### opcode相关方法

下面7个方法中的6个，遵循一个共同特点：“创建SourceValue对象”。

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  @Override
  public SourceValue newOperation(AbstractInsnNode insn) {
    int size = 1; // 或者 size = 2
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(size, insn);
  }

  @Override
  public SourceValue copyOperation(AbstractInsnNode insn, SourceValue value) {
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(value.getSize(), insn);
  }

  @Override
  public SourceValue unaryOperation(AbstractInsnNode insn,
                                    SourceValue value) {
    int size = 1; // 或者 size = 2
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(size, insn);
  }

  @Override
  public SourceValue binaryOperation(AbstractInsnNode insn,
                                     SourceValue value1,
                                     SourceValue value2) {
    int size = 1; // 或者 size = 2
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(size, insn);
  }

  @Override
  public SourceValue ternaryOperation(AbstractInsnNode insn,
                                      SourceValue value1,
                                      SourceValue value2,
                                      SourceValue value3) {
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(1, insn);
  }

  @Override
  public SourceValue naryOperation(AbstractInsnNode insn,
                                   List<? extends SourceValue> values) {
    int size = 1; // 或者 size = 2
    // 这里是SourceValue定义的第2个构造方法，需要1个指令参与
    return new SourceValue(size, insn);
  }

  @Override
  public void returnOperation(AbstractInsnNode insn,
                              SourceValue value,
                              SourceValue expected) {
    // Nothing to do.
  }
}
```

##### merge方法

```java
public class SourceInterpreter extends Interpreter<SourceValue> implements Opcodes {
  @Override
  public SourceValue merge(SourceValue value1, SourceValue value2) {
    // 第一种情况，SmallSet类型
    if (value1.insns instanceof SmallSet && value2.insns instanceof SmallSet) {
      Set<AbstractInsnNode> setUnion = value1.insns + value2.insns;
      if (setUnion == value1.insns && value1.size == value2.size) {
        // value1包含了value2
        return value1;
      } else {
        // 这里是SourceValue定义的第3个构造方法，需要多个指令参与
        // value1不能包含value2，那就生成一个新的SourceValue对象
        return new SourceValue(Math.min(value1.size, value2.size), setUnion);
      }
    }

    // 第二种情况，其它类型，value1不能包含value2，那就生成一个新的SourceValue对象
    if (value1.size != value2.size || !containsAll(value1.insns, value2.insns)) {
      HashSet<AbstractInsnNode> setUnion = new HashSet<>();
      setUnion.addAll(value1.insns);
      setUnion.addAll(value2.insns);
      // 这里是SourceValue定义的第3个构造方法，需要多个指令参与
      return new SourceValue(Math.min(value1.size, value2.size), setUnion);
    }

    // 第三种情况，其它类型，value1包含了value2
    return value1;
  }
}
```

### 4.10.3 测试代码

#### simple

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a + b;
    System.out.println(c);
  }
}
```

对应的Frame变化如下：

```shell
test:()V
000:    iconst_1                                {[], [], [], []} | {}
001:    istore_1                                {[], [], [], []} | {[iconst_1]}
002:    iconst_2                                {[], [istore_1], [], []} | {}
003:    istore_2                                {[], [istore_1], [], []} | {[iconst_2]}
004:    iload_1                                 {[], [istore_1], [istore_2], []} | {}
005:    iload_2                                 {[], [istore_1], [istore_2], []} | {[iload_1]}
006:    iadd                                    {[], [istore_1], [istore_2], []} | {[iload_1], [iload_2]}
007:    istore_3                                {[], [istore_1], [istore_2], []} | {[iadd]}
008:    getstatic System.out                    {[], [istore_1], [istore_2], [istore_3]} | {}
009:    iload_3                                 {[], [istore_1], [istore_2], [istore_3]} | {[getstatic System.out]}
010:    invokevirtual PrintStream.println       {[], [istore_1], [istore_2], [istore_3]} | {[getstatic System.out], [iload_3]}
011:    return                                  {[], [istore_1], [istore_2], [istore_3]} | {}
================================================================
```

##### if

```java
public class HelloWorld {
  public void test(int a, int b) {
    Object obj;
    int c = a + b;
    if (c > 10) {
      obj = Integer.valueOf(c);
    }
    else {
      obj = Float.valueOf(c);
    }
    System.out.println(obj);
  }
}
```

### 4.10.4 总结

本文内容总结如下：

- 第一点，**`SourceValue`类是“记录”instruction（`AbstractInsnNode`）与Frame当中的值（`SourceValue`）之间的关系，而`SourceInterpreter`类是“建立”两者之间的联系**。
- 第二点，`SourceInterpreter`类是属于`Interpreter`的部分，它使用`SourceValue`的逻辑分成三个部分：
  - 在指令（instruction）执行之前，计算方法的初始Frame（initial frame），this、方法的参数和未初始化的值，它们对应的`SourceValue`对象不与任何的指令（instruction）相关，对应于`SourceValue`第1个构造方法。
  - 在指令（instruction）开始执行之后，如果某一条指令（instruction）改变了local variable和operand stack上的值，那么对应的`SourceValue`对象就记录该instruction（`AbstractInsnNode`）与`SourceValue`对象的联系，对应于`SourceValue`第2个构造方法。
  - 在指令（instruction）开始执行之后，control flow有分支（brach），也就意味着将来的合并（merge）；在合并（merge）时对应的`SourceValue`对象就记录该多个instruction（`Set<AbstractInsnNode>`）与`SourceValue`对象的联系，对应于`SourceValue`第3个构造方法。

## 4.11 SourceValue-SourceInterpreter示例：反编译-方法参数

### 4.11.1 如何反编译方法参数

#### 提出问题

我们在学习Java的过程中，多多少少都会用到[Java Decompiler](http://java-decompiler.github.io/)工具，它可以将具体的`.class`文件转换成相应的Java代码。

假如有一个`HelloWorld`类：

```java
public class HelloWorld {
  public void test(int a, int b) {
    int sum = Math.addExact(a, b);
    int diff = Math.subtractExact(a, b);
    int result = Math.multiplyExact(sum, diff);
    System.out.println(result);
  }
}
```

上面的`HelloWorld.java`经过编译之后会生成`HelloWorld.class`文件，然后可以查看其包含的instruction内容：

```shell
$ javap -v sample.HelloWorld
  Compiled from "HelloWorld.java"
public class sample.HelloWorld
{
...
  public void test(int, int);
    descriptor: (II)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=6, args_size=3
         0: iload_1
         1: iload_2
         2: invokestatic  #2                  // Method java/lang/Math.addExact:(II)I
         5: istore_3
         6: iload_1
         7: iload_2
         8: invokestatic  #3                  // Method java/lang/Math.subtractExact:(II)I
        11: istore        4
        13: iload_3
        14: iload         4
        16: invokestatic  #4                  // Method java/lang/Math.multiplyExact:(II)I
        19: istore        5
        21: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        24: iload         5
        26: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
        29: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      30     0  this   Lsample/HelloWorld;
            0      30     1     a   I
            0      30     2     b   I
            6      24     3   sum   I
           13      17     4  diff   I
           21       9     5 result   I
}
```

那么，我们能不能利用Java ASM帮助我们做一些反编译的工作呢？

#### 整体思路

我们的整体思路就是，结合`SourceInterpreter`类和`LocalVariableTable`来对invoke（方法调用）相关的指令进行反编译。

使用`SourceInterpreter`类输出Frame变化信息：

```shell
test:(II)V
000:                                 iload_1    {[], [], [], [], [], []} | {}
001:                                 iload_2    {[], [], [], [], [], []} | {[iload_1]}
002:              invokestatic Math.addExact    {[], [], [], [], [], []} | {[iload_1], [iload_2]}
003:                                istore_3    {[], [], [], [], [], []} | {[invokestatic Math.addExact]}
004:                                 iload_1    {[], [], [], [istore_3], [], []} | {}
005:                                 iload_2    {[], [], [], [istore_3], [], []} | {[iload_1]}
006:         invokestatic Math.subtractExact    {[], [], [], [istore_3], [], []} | {[iload_1], [iload_2]}
007:                                istore 4    {[], [], [], [istore_3], [], []} | {[invokestatic Math.subtractExact]}
008:                                 iload_3    {[], [], [], [istore_3], [istore 4], []} | {}
009:                                 iload 4    {[], [], [], [istore_3], [istore 4], []} | {[iload_3]}
010:         invokestatic Math.multiplyExact    {[], [], [], [istore_3], [istore 4], []} | {[iload_3], [iload 4]}
011:                                istore 5    {[], [], [], [istore_3], [istore 4], []} | {[invokestatic Math.multiplyExact]}
012:                    getstatic System.out    {[], [], [], [istore_3], [istore 4], [istore 5]} | {}
013:                                 iload 5    {[], [], [], [istore_3], [istore 4], [istore 5]} | {[getstatic System.out]}
014:       invokevirtual PrintStream.println    {[], [], [], [istore_3], [istore 4], [istore 5]} | {[getstatic System.out], [iload 5]}
015:                                  return    {[], [], [], [istore_3], [istore 4], [istore 5]} | {}
================================================================
```

### 4.11.2 示例：方法参数反编译

#### 预期目标

我们想对`HelloWorld.class`中的`test`方法内的invoke相关的instruction进行反编译。

```java
public class HelloWorld {
  public void test(int a, int b) {
    int sum = Math.addExact(a, b);
    int diff = Math.subtractExact(a, b);
    int result = Math.multiplyExact(sum, diff);
    System.out.println(result);
  }
}
```

预期目标：将方法调用的参数进行反编译。

例如，将下面的instructions反编译成`Math.addExact(a, b)`。

```shell
0: iload_1
1: iload_2
2: invokestatic  #2                  // Method java/lang/Math.addExact:(II)I
```

#### 编码实现

```java
import org.objectweb.asm.Type;
import org.objectweb.asm.tree.*;
import org.objectweb.asm.tree.analysis.*;

import java.util.ArrayList;
import java.util.List;

public class ReverseEngineerMethodArgumentsDiagnosis {
  private static final String UNKNOWN_VARIABLE_NAME = "unknown";

  public static void diagnose(String className, MethodNode mn) throws AnalyzerException {
    // 第一步，获取Frame信息
    Analyzer<SourceValue> analyzer = new Analyzer<>(new SourceInterpreter());
    Frame<SourceValue>[] frames = analyzer.analyze(className, mn);

    // 第二步，获取LocalVariableTable信息
    List<LocalVariableNode> localVariables = mn.localVariables;
    if (localVariables == null || localVariables.size() < 1) {
      System.out.println("LocalVariableTable is Empty");
      return;
    }

    // 第三步，获取instructions，并找到与invoke相关的指令
    InsnList instructions = mn.instructions;
    int[] methodInsnArray = findMethodInvokes(instructions);

    // 第四步，对invoke相关的指令进行反编译
    for (int methodInsn : methodInsnArray) {
      // (1) 获取方法的参数
      MethodInsnNode methodInsnNode = (MethodInsnNode) instructions.get(methodInsn);
      Type methodType = Type.getMethodType(methodInsnNode.desc);
      Type[] argumentTypes = methodType.getArgumentTypes();
      int argNum = argumentTypes.length;

      // (2) 从Frame当中获取指令，并将指令转换LocalVariableTable当中的变量名
      Frame<SourceValue> f = frames[methodInsn];
      int stackSize = f.getStackSize();
      List<String> argList = new ArrayList<>();
      for (int i = 0; i < argNum; i++) {
        int stackIndex = stackSize - argNum + i;
        SourceValue stackValue = f.getStack(stackIndex);
        AbstractInsnNode insn = stackValue.insns.iterator().next();
        String argName = getMethodVariableName(insn, localVariables);
        argList.add(argName);
      }

      // (3) 将反编译的结果打印出来
      String line = String.format("%s.%s(%s)", methodInsnNode.owner, methodInsnNode.name, argList);
      System.out.println(line);
    }
  }

  public static String getMethodVariableName(AbstractInsnNode insn, List<LocalVariableNode> localVariables) {
    if (insn instanceof VarInsnNode) {
      VarInsnNode varInsnNode = (VarInsnNode) insn;
      int localIndex = varInsnNode.var;

      for (LocalVariableNode node : localVariables) {
        if (node.index == localIndex) {
          return node.name;
        }
      }

      return String.format("locals[%d]", localIndex);
    }
    return UNKNOWN_VARIABLE_NAME;
  }

  public static int[] findMethodInvokes(InsnList instructions) {
    int size = instructions.size();
    boolean[] methodArray = new boolean[size];
    for (int i = 0; i < size; i++) {
      AbstractInsnNode node = instructions.get(i);
      if (node instanceof MethodInsnNode) {
        methodArray[i] = true;
      }
    }

    int count = 0;
    for (boolean flag : methodArray) {
      if (flag) {
        count++;
      }
    }

    int[] array = new int[count];
    int j = 0;
    for (int i = 0; i < size; i++) {
      boolean flag = methodArray[i];
      if (flag) {
        array[j] = i;
        j++;
      }
    }
    return array;
  }
}
```

#### 进行分析

在`HelloWorldAnalysisTree`类当中，要注意：不能使用`ClassReader.SKIP_DEBUG`，因为我们要使用到`MethodNode.localVariables`字段的信息。

```java
public class HelloWorldAnalysisTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);

    int parsingOptions = ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）进行分析
    String className = cn.name;
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);
    ReverseEngineerMethodArgumentsDiagnosis.diagnose(className, mn);
  }
}
```

输出结果：

```shell
java/lang/Math.addExact([a, b])
java/lang/Math.subtractExact([a, b])
java/lang/Math.multiplyExact([sum, diff])
java/io/PrintStream.println([result])
```

### 4.11.3 总结

本文内容总结如下：

- 第一点，整体的思路，是利用`SourceInterpreter`类和LocalVariableTable来实现的。
- 第二点，代码示例。如何编码实现对于方法的参数进行反编译。

## 4.12 SourceValue-SourceInterpreter示例：查找相关的指令

在StackOverflow上有这样一个问题：[Find all instructions belonging to a specific method-call](https://stackoverflow.com/questions/60969392/java-asm-bytecode-find-all-instructions-belonging-to-a-specific-method-call)。在解决方法当中，就用到了`SourceInterpreter`和`SourceValue`类。在本文当中，我们将这个问题和解决思路简单地进行介绍。

### 4.12.1 实现思路

#### 回顾SourceInterpreter

首先，我们来回顾一下`SourceInterpreter`的作用：记录指令（instruction）与Frame当中值（`SourceValue`）的关联关系。

```java
public class HelloWorld {
  public void test(int a, int b) {
    int c = a + b;
    System.out.println(c);
  }
}
```

那么，借助于`SourceInterpreter`类查看`test`方法内某一条instuction将某一个`SourceValue`加载（入栈）到Frame上：

```shell
test:(II)V
000:                                 iload_1    {[], [], [], []} | {}
001:                                 iload_2    {[], [], [], []} | {[iload_1]}
002:                                    iadd    {[], [], [], []} | {[iload_1], [iload_2]}
003:                                istore_3    {[], [], [], []} | {[iadd]}
004:                    getstatic System.out    {[], [], [], [istore_3]} | {}
005:                                 iload_3    {[], [], [], [istore_3]} | {[getstatic System.out]}
006:       invokevirtual PrintStream.println    {[], [], [], [istore_3]} | {[getstatic System.out], [iload_3]}
007:                                  return    {[], [], [], [istore_3]} | {}
================================================================
```

由此，我们可以模仿一下，进一步记录某一条instuction将某一个`SourceValue`从Frame上消耗掉（出栈）：

```shell
test:(II)V
000:                                 iload_1    {[], [], [], []} | {}
001:                                 iload_2    {[], [], [], []} | {[iadd]}
002:                                    iadd    {[], [], [], []} | {[iadd], [iadd]}
003:                                istore_3    {[], [], [], []} | {[istore_3]}
004:                    getstatic System.out    {[], [], [], []} | {}
005:                                 iload_3    {[], [], [], []} | {[invokevirtual PrintStream.println]}
006:       invokevirtual PrintStream.println    {[], [], [], []} | {[invokevirtual PrintStream.println], [invokevirtual PrintStream.println]}
007:                                  return    {[], [], [], []} | {}
================================================================
```

那么，我们怎么实现这样一个功能呢？用一个`DestinationInterpreter`类来实现。

#### DestinationInterpreter

首先，我们编写一个`DestinationInterpreter`类。对于这个类，我们从两点来把握：

- 第一点，抽象功能。要实现什么的功能呢？记录operand stack上某一个元素是被哪一个指令（instruction）消耗掉（出栈）的。这个功能正好与`SourceInterpreter`类的功能相反。
- 第二点，具体实现。如何进行编码实现呢？`DestinationInterpreter`类，是模仿着`SourceInterpreter`类实现的。

```java
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.tree.*;
import org.objectweb.asm.tree.analysis.AnalyzerException;
import org.objectweb.asm.tree.analysis.Interpreter;
import org.objectweb.asm.tree.analysis.SourceValue;

import java.util.HashSet;
import java.util.List;

public class DestinationInterpreter extends Interpreter<SourceValue> implements Opcodes {
  public DestinationInterpreter() {
    super(ASM9);
    if (getClass() != DestinationInterpreter.class) {
      throw new IllegalStateException();
    }
  }

  protected DestinationInterpreter(final int api) {
    super(api);
  }

  @Override
  public SourceValue newValue(Type type) {
    if (type == Type.VOID_TYPE) {
      return null;
    }
    return new SourceValue(type == null ? 1 : type.getSize(), new HashSet<>());
  }

  @Override
  public SourceValue newOperation(AbstractInsnNode insn) {
    int size;
    switch (insn.getOpcode()) {
      case LCONST_0:
      case LCONST_1:
      case DCONST_0:
      case DCONST_1:
        size = 2;
        break;
      case LDC:
        Object value = ((LdcInsnNode) insn).cst;
        size = value instanceof Long || value instanceof Double ? 2 : 1;
        break;
      case GETSTATIC:
        size = Type.getType(((FieldInsnNode) insn).desc).getSize();
        break;
      default:
        size = 1;
        break;
    }
    return new SourceValue(size, new HashSet<>());
  }

  @Override
  public SourceValue copyOperation(AbstractInsnNode insn, SourceValue value) throws AnalyzerException {
    int opcode = insn.getOpcode();
    if (opcode >= ISTORE && opcode <= ASTORE) {
      value.insns.add(insn);
    }

    return new SourceValue(value.getSize(), new HashSet<>());
  }

  @Override
  public SourceValue unaryOperation(AbstractInsnNode insn, SourceValue value) throws AnalyzerException {
    value.insns.add(insn);

    int size;
    switch (insn.getOpcode()) {
      case LNEG:
      case DNEG:
      case I2L:
      case I2D:
      case L2D:
      case F2L:
      case F2D:
      case D2L:
        size = 2;
        break;
      case GETFIELD:
        size = Type.getType(((FieldInsnNode) insn).desc).getSize();
        break;
      default:
        size = 1;
        break;
    }
    return new SourceValue(size, new HashSet<>());
  }

  @Override
  public SourceValue binaryOperation(AbstractInsnNode insn, SourceValue value1, SourceValue value2) throws AnalyzerException {
    value1.insns.add(insn);
    value2.insns.add(insn);

    int size;
    switch (insn.getOpcode()) {
      case LALOAD:
      case DALOAD:
      case LADD:
      case DADD:
      case LSUB:
      case DSUB:
      case LMUL:
      case DMUL:
      case LDIV:
      case DDIV:
      case LREM:
      case DREM:
      case LSHL:
      case LSHR:
      case LUSHR:
      case LAND:
      case LOR:
      case LXOR:
        size = 2;
        break;
      default:
        size = 1;
        break;
    }
    return new SourceValue(size, new HashSet<>());
  }

  @Override
  public SourceValue ternaryOperation(AbstractInsnNode insn, SourceValue value1, SourceValue value2, SourceValue value3) throws AnalyzerException {
    value1.insns.add(insn);
    value2.insns.add(insn);
    value3.insns.add(insn);

    return new SourceValue(1, new HashSet<>());
  }

  @Override
  public SourceValue naryOperation(AbstractInsnNode insn, List<? extends SourceValue> values) throws AnalyzerException {
    if (values != null) {
      for (SourceValue v : values) {
        v.insns.add(insn);
      }
    }

    int size;
    int opcode = insn.getOpcode();
    if (opcode == MULTIANEWARRAY) {
      size = 1;
    }
    else if (opcode == INVOKEDYNAMIC) {
      size = Type.getReturnType(((InvokeDynamicInsnNode) insn).desc).getSize();
    }
    else {
      size = Type.getReturnType(((MethodInsnNode) insn).desc).getSize();
    }
    return new SourceValue(size, new HashSet<>());
  }

  @Override
  public void returnOperation(AbstractInsnNode insn, SourceValue value, SourceValue expected) throws AnalyzerException {
    // Nothing to do.
  }

  // 不像 SourceInterpreter需要标记Value来源的可能值集合，这里Value被消耗一定是被具体某一指令消耗
  @Override
  public SourceValue merge(final SourceValue value1, final SourceValue value2) {
    return new SourceValue(Math.min(value1.size, value2.size), new HashSet<>());
  }
}
```

### 4.12.2 示例：查找相关的指令

#### 预期目标

假如有一个`HelloWorld`类：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = 3;
    int d = 4;
    int sum = add(a, b);
    int diff = sub(c, d);
    int result = mul(sum, diff);
    System.out.println(result);
  }

  public int add(int a, int b) {
    return a + b;
  }

  public int sub(int a, int b) {
    return a - b;
  }

  public int mul(int a, int b) {
    return a * b;
  }
}
```

我们对`test`方法进行分析，想实现的预期目标：如果想删除对某一个方法的调用，有哪些指令会变得无效呢？

例如，想删除`add(a, b)`方法调用，直接使用`int sum = 10;`，那么`a`和`b`两个变量就不需要了：

```java
public class HelloWorld {
  public void test() {
    int c = 3;
    int d = 4;
    int sum = 10;
    int diff = sub(c, d);
    int result = mul(sum, diff);
    System.out.println(result);
  }

  // ...
}
```

#### 编码实现

```java
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.AbstractInsnNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.VarInsnNode;
import org.objectweb.asm.tree.analysis.*;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class RelatedInstructionDiagnosis {
  public static int[] diagnose(String className, MethodNode mn, int insnIndex) throws AnalyzerException {
    // 第一步，判断insnIndex范围是否合理
    InsnList instructions = mn.instructions;
    int size = instructions.size();
    if (insnIndex < 0 || insnIndex >= size) {
      String message = String.format("the 'insnIndex' argument should in range [0, %d]", size - 1);
      throw new IllegalArgumentException(message);
    }

    // 第二步，获取两个Frame
    Frame<SourceValue>[] sourceFrames = getSourceFrames(className, mn);
    Frame<SourceValue>[] destinationFrames = getDestinationFrames(className, mn);

    // 第三步，循环处理，所有结果记录到这个intArrayList变量中
    TIntArrayList intArrayList = new TIntArrayList();
    // 循环tmpInsnList
    List<AbstractInsnNode> tmpInsnList = new ArrayList<>();
    AbstractInsnNode insnNode = instructions.get(insnIndex);
    tmpInsnList.add(insnNode);
    for (int i = 0; i < tmpInsnList.size(); i++) {
      AbstractInsnNode currentNode = tmpInsnList.get(i);
      int opcode = currentNode.getOpcode();

      int index = instructions.indexOf(currentNode);
      intArrayList.add(index);

      // 第一种情况，处理load相关的opcode情况
      Frame<SourceValue> srcFrame = sourceFrames[index];
      if (opcode >= Opcodes.ILOAD && opcode <= Opcodes.ALOAD) {
        VarInsnNode varInsnNode = (VarInsnNode) currentNode;
        int localIndex = varInsnNode.var;
        SourceValue value = srcFrame.getLocal(localIndex);
        for (AbstractInsnNode insn : value.insns) {
          if (!tmpInsnList.contains(insn)) {
            tmpInsnList.add(insn);
          }
        }
      }

      // 第二种情况，从dstFrame到srcFrame查找
      Frame<SourceValue> dstFrame = destinationFrames[index];
      int stackSize = dstFrame.getStackSize();
      for (int j = 0; j < stackSize; j++) {
        SourceValue value = dstFrame.getStack(j);
        if (value.insns.contains(currentNode)) {
          for (AbstractInsnNode insn : srcFrame.getStack(j).insns) {
            if (!tmpInsnList.contains(insn)) {
              tmpInsnList.add(insn);
            }
          }
        }
      }
    }

    // 第四步，将intArrayList变量转换成int[]，并进行排序
    int[] array = intArrayList.toNativeArray();
    Arrays.sort(array);
    return array;
  }


  private static Frame<SourceValue>[] getSourceFrames(String className, MethodNode mn) throws AnalyzerException {
    Analyzer<SourceValue> analyzer = new Analyzer<>(new SourceInterpreter());
    return analyzer.analyze(className, mn);
  }

  private static Frame<SourceValue>[] getDestinationFrames(String className, MethodNode mn) throws AnalyzerException {
    Analyzer<SourceValue> analyzer = new Analyzer<>(new DestinationInterpreter());
    return analyzer.analyze(className, mn);
  }
}
```

#### 进行分析

```java
public class HelloWorldAnalysisTree {
  public static void main(String[] args) throws Exception {
    String relative_path = "sample/HelloWorld.class";
    String filepath = FileUtils.getFilePath(relative_path);
    byte[] bytes = FileUtils.readBytes(filepath);

    //（1）构建ClassReader
    ClassReader cr = new ClassReader(bytes);

    //（2）生成ClassNode
    int api = Opcodes.ASM9;
    ClassNode cn = new ClassNode(api);

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cn, parsingOptions);

    //（3）进行分析
    List<MethodNode> methods = cn.methods;
    MethodNode mn = methods.get(1);
    int[] array = RelatedInstructionDiagnosis.diagnose(cn.name, mn, 11);
    System.out.println(Arrays.toString(array));
    BoxDrawingUtils.printInstructionLinks(mn.instructions, array);
  }
}
```

输出结果：

```shell
[0, 1, 2, 3, 8, 9, 10, 11]
┌──── 000: iconst_1
├──── 001: istore_1
├──── 002: iconst_2
├──── 003: istore_2
│     004: iconst_3
│     005: istore_3
│     006: iconst_4
│     007: istore 4
├──── 008: aload_0
├──── 009: iload_1
├──── 010: iload_2
└──── 011: invokevirtual HelloWorld.add
      012: istore 5
      013: aload_0
      014: iload_3
      015: iload 4
      016: invokevirtual HelloWorld.sub
      017: istore 6
      018: aload_0
      019: iload 5
      020: iload 6
      021: invokevirtual HelloWorld.mul
      022: istore 7
      023: getstatic System.out
      024: iload 7
      025: invokevirtual PrintStream.println
      026: return
```

### 4.12.3 测试用例

#### switch-yes

```java
public class HelloWorld {
  public void test(int val) {
    double doubleValue = Math.random();
    switch (val) {
      case 10:
        System.out.println("val = 10");
        break;
      case 20:
        System.out.println("val = 20");
        break;
      case 30:
        System.out.println("val = 30");
        break;
      case 40:
        System.out.println("val = 40");
        break;
      default:
        System.out.println("val is unknown");
    }
    System.out.println(doubleValue); // 分析这条语句
  }
}
```

输出结果：

```shell
[0, 1, 29, 30, 31]
┌──── 000: invokestatic Math.random
├──── 001: dstore_2
│     002: iload_1
│     003: lookupswitch {
│              10: L0
│              20: L1
│              30: L2
│              40: L3
│              default: L4
│          }
│     004: L0
│     005: getstatic System.out
│     006: ldc "val = 10"
│     007: invokevirtual PrintStream.println
│     008: goto L5
│     009: L1
│     010: getstatic System.out
│     011: ldc "val = 20"
│     012: invokevirtual PrintStream.println
│     013: goto L5
│     014: L2
│     015: getstatic System.out
│     016: ldc "val = 30"
│     017: invokevirtual PrintStream.println
│     018: goto L5
│     019: L3
│     020: getstatic System.out
│     021: ldc "val = 40"
│     022: invokevirtual PrintStream.println
│     023: goto L5
│     024: L4
│     025: getstatic System.out
│     026: ldc "val is unknown"
│     027: invokevirtual PrintStream.println
│     028: L5
├──── 029: getstatic System.out
├──── 030: dload_2
└──── 031: invokevirtual PrintStream.println
      032: return
```

#### switch-no

在当前的解决思路中，还不能很好的处理创建对象（`new`）的情况。

```java
public class HelloWorld {
  public void test(int val) {
    Random rand = new Random(); // 注意，这里创建了Random对象
    double doubleValue = rand.nextDouble();
    switch (val) {
      case 10:
        System.out.println("val = 10");
        break;
      case 20:
        System.out.println("val = 20");
        break;
      case 30:
        System.out.println("val = 30");
        break;
      case 40:
        System.out.println("val = 40");
        break;
      default:
        System.out.println("val is unknown");
    }
    System.out.println(doubleValue); // 分析这条语句
  }
}
```

输出结果：

```shell
[0, 3, 4, 5, 6, 34, 35, 36]
┌──── 000: new Random                              // new + dup + invokespecial三个指令一起来创建对象
│     001: dup                                     // 当前的方法只分析出了new，而没有分析出dup和invokespecial
│     002: invokespecial Random.<init>
├──── 003: astore_2
├──── 004: aload_2
├──── 005: invokevirtual Random.nextDouble
├──── 006: dstore_3
│     007: iload_1
│     008: lookupswitch {
│              10: L0
│              20: L1
│              30: L2
│              40: L3
│              default: L4
│          }
│     009: L0
│     010: getstatic System.out
│     011: ldc "val = 10"
│     012: invokevirtual PrintStream.println
│     013: goto L5
│     014: L1
│     015: getstatic System.out
│     016: ldc "val = 20"
│     017: invokevirtual PrintStream.println
│     018: goto L5
│     019: L2
│     020: getstatic System.out
│     021: ldc "val = 30"
│     022: invokevirtual PrintStream.println
│     023: goto L5
│     024: L3
│     025: getstatic System.out
│     026: ldc "val = 40"
│     027: invokevirtual PrintStream.println
│     028: goto L5
│     029: L4
│     030: getstatic System.out
│     031: ldc "val is unknown"
│     032: invokevirtual PrintStream.println
│     033: L5
├──── 034: getstatic System.out
├──── 035: dload_3
└──── 036: invokevirtual PrintStream.println
      037: return
```

### 4.12.4 总结

本文内容总结如下：

- 第一点，模拟`SourceInterpreter`类来编写一个`DestinationInterpreter`类，这两个类的作用是相反的。
- 第二点，结合`SourceInterpreter`和`DestinationInterpreter`类，用来查找相关的指令。

## 4.13 Interpreter和Value的精妙之处

### 4.13.1 内容回顾

首先，我们要强调：**在`asm-analysis.jar`当中，最重要的类就是`Analyzer`、`Frame`、`Interpreter`和`Value`四个类**。

其次，贯穿本章内容的核心就是下面两行代码：

```java
   ┌── Analyzer
   │        ┌── Value                                   ┌── Interpreter
   │        │                                           │
Analyzer<BasicValue> analyzer = new Analyzer<>(new BasicInterpreter());
Frame<BasicValue>[] frames = analyzer.analyze(owner, mn);
   │        │
   │        └── Value
   └── Frame
```

在这两行代码当中，用到了`Analyzer`、`Frame`、`Interpreter`和`Value`这四个类：

- 第一行代码，使用`Interpreter`来创建`Analyzer`对象。
- 第二行代码，调用`Analyzer.analyze`方法生成`Frame<?>[]`信息，之后可以进行各种不同类型的分析。

再者，这四个类当中，`Analyzer`和`Frame`是相对固定的，而`Interpreter`和`Value`是变化的：

```pseudocode
┌──────────┬─────────────┐
│          │  Analyzer   │
│  Fixed   ├─────────────┤
│          │    Frame    │
├──────────┼─────────────┤
│          │ Interpreter │
│ Variable ├─────────────┤
│          │    Value    │
└──────────┴─────────────┘
```

最后，`Interpreter`和`Value`类有不同的具体表现形式：

```pseudocode
┌───┬───────────────────┬─────────────┬───────┐
│ 0 │    Interpreter    │    Value    │ Range │
├───┼───────────────────┼─────────────┼───────┤
│ 1 │ BasicInterpreter  │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 2 │   BasicVerifier   │ BasicValue  │   7   │
├───┼───────────────────┼─────────────┼───────┤
│ 3 │  SimpleVerifier   │ BasicValue  │   N   │
├───┼───────────────────┼─────────────┼───────┤
│ 4 │ SourceInterpreter │ SourceValue │   N   │
└───┴───────────────────┴─────────────┴───────┘
```

回顾了这些内容之后，我们再来看一下`Interpreter`和`Value`的精妙之处。

### 4.13.2 Interpreter类的精妙之处

`Interpreter`类的精妙之处体现在它的功能上，我们从三点来把握，来欣赏`Interpreter`类的“艺术和美”。

第一点，从功能的角度来讲，`Interpreter`类与`Frame`类的功能分割恰到好处：**入栈和出栈的操作交给了`Frame`来进行处理，而具体的入栈和出栈的值是由`Interpreter`来处理的**。举一个生活中的例子，国有资本（Government Capital，相当于`Frame`类）新建了一个高铁站，进入车站的车辆和离开车站的车辆是它来负责的；具体的运作交给民营资本（Private Investment，相当于`Interpreter`类），每辆车上装载的是货物还是人是由它来决定的。

第二点，从opcode的角度来讲，它将原本200多个opcode分类成（浓缩成）了7个方法，分类的依据就是使用的（或者说，消耗的、出栈的、入栈的）元素数量来决定的。

```java
public abstract class Interpreter {
  public abstract Value newOperation(AbstractInsnNode insn) throws AnalyzerException;

  public abstract Value copyOperation(AbstractInsnNode insn, Value value) throws AnalyzerException;

  public abstract Value unaryOperation(AbstractInsnNode insn, Value value) throws AnalyzerException;

  public abstract Value binaryOperation(AbstractInsnNode insn, Value value1, Value value2) throws AnalyzerException;

  public abstract Value ternaryOperation(AbstractInsnNode insn, Value value1, Value value2, Value value3) throws AnalyzerException;

  public abstract Value naryOperation(AbstractInsnNode insn, List<Value> values) throws AnalyzerException;

  public abstract void returnOperation(AbstractInsnNode insn, Value value, Value expected) throws AnalyzerException;
}
```

第三点，从`Value`的角度来讲，`Interpreter`类决定了`Value`类对象的数量、表达能力。一般来说，`Value`类对象的数量越多，它的表达能力越丰富。举个生活当中的例子，这有点类似于“计划生育”，`Interpreter`类决定了`Value`类的数量。

### 4.13.3 Value类的表达能力

在Frame当中，我们可以存储的`Value`值有两种：**类型**（type或者class）和**实例对象**（object或者instance）。

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    String str1 = "Hello";
    String str2 = "World";
    Object obj1 = new Object();
    Object obj2 = new Object();
  }
}
```

如果使用`BasicInterpreter`类，那么`BasicValue`值的表达能力（**类型**）如下：每个类型

```shell
test:()V
000:    iconst_1                                {R, ., ., ., ., ., .} | {}
001:    istore_1                                {R, ., ., ., ., ., .} | {I}
002:    iconst_2                                {R, I, ., ., ., ., .} | {}
003:    istore_2                                {R, I, ., ., ., ., .} | {I}
004:    ldc "Hello"                             {R, I, I, ., ., ., .} | {}
005:    astore_3                                {R, I, I, ., ., ., .} | {R}
006:    ldc "World"                             {R, I, I, R, ., ., .} | {}
007:    astore 4                                {R, I, I, R, ., ., .} | {R}
008:    new Object                              {R, I, I, R, R, ., .} | {}
009:    dup                                     {R, I, I, R, R, ., .} | {R}
010:    invokespecial Object.<init>             {R, I, I, R, R, ., .} | {R, R}
011:    astore 5                                {R, I, I, R, R, ., .} | {R}
012:    new Object                              {R, I, I, R, R, R, .} | {}
013:    dup                                     {R, I, I, R, R, R, .} | {R}
014:    invokespecial Object.<init>             {R, I, I, R, R, R, .} | {R, R}
015:    astore 6                                {R, I, I, R, R, R, .} | {R}
016:    return                                  {R, I, I, R, R, R, R} | {}
================================================================
```

如果使用`SimpleVerifier`类，那么`BasicValue`值的表达能力（**对象实例**）如下：每个不同的对象对应一个`BasicValue`值（看着好像是一样的，实际下面出现的两个String，三个Object各自都是新`BasicValue`对象）。

```shell
test:()V
000:    iconst_1                                {HelloWorld, ., ., ., ., ., .} | {}
001:    istore_1                                {HelloWorld, ., ., ., ., ., .} | {I}
002:    iconst_2                                {HelloWorld, I, ., ., ., ., .} | {}
003:    istore_2                                {HelloWorld, I, ., ., ., ., .} | {I}
004:    ldc "Hello"                             {HelloWorld, I, I, ., ., ., .} | {}
005:    astore_3                                {HelloWorld, I, I, ., ., ., .} | {String}
006:    ldc "World"                             {HelloWorld, I, I, String, ., ., .} | {}
007:    astore 4                                {HelloWorld, I, I, String, ., ., .} | {String}
008:    new Object                              {HelloWorld, I, I, String, String, ., .} | {}
009:    dup                                     {HelloWorld, I, I, String, String, ., .} | {Object}
010:    invokespecial Object.<init>             {HelloWorld, I, I, String, String, ., .} | {Object, Object}
011:    astore 5                                {HelloWorld, I, I, String, String, ., .} | {Object}
012:    new Object                              {HelloWorld, I, I, String, String, Object, .} | {}
013:    dup                                     {HelloWorld, I, I, String, String, Object, .} | {Object}
014:    invokespecial Object.<init>             {HelloWorld, I, I, String, String, Object, .} | {Object, Object}
015:    astore 6                                {HelloWorld, I, I, String, String, Object, .} | {Object}
016:    return                                  {HelloWorld, I, I, String, String, Object, Object} | {}
================================================================
```

如果使用`SourceInterpreter`类，那么`SourceValue`值的表达能力（**对象实例**）如下：差不多每个instruction都会生成一个新的`SourceValue`对象

```shell
test:()V
000:    iconst_1                                {[], [], [], [], [], [], []} | {}
001:    istore_1                                {[], [], [], [], [], [], []} | {[iconst_1]}
002:    iconst_2                                {[], [istore_1], [], [], [], [], []} | {}
003:    istore_2                                {[], [istore_1], [], [], [], [], []} | {[iconst_2]}
004:    ldc "Hello"                             {[], [istore_1], [istore_2], [], [], [], []} | {}
005:    astore_3                                {[], [istore_1], [istore_2], [], [], [], []} | {[ldc "Hello"]}
006:    ldc "World"                             {[], [istore_1], [istore_2], [astore_3], [], [], []} | {}
007:    astore 4                                {[], [istore_1], [istore_2], [astore_3], [], [], []} | {[ldc "World"]}
008:    new Object                              {[], [istore_1], [istore_2], [astore_3], [astore 4], [], []} | {}
009:    dup                                     {[], [istore_1], [istore_2], [astore_3], [astore 4], [], []} | {[new Object]}
010:    invokespecial Object.<init>             {[], [istore_1], [istore_2], [astore_3], [astore 4], [], []} | {[new Object], [dup]}
011:    astore 5                                {[], [istore_1], [istore_2], [astore_3], [astore 4], [], []} | {[new Object]}
012:    new Object                              {[], [istore_1], [istore_2], [astore_3], [astore 4], [astore 5], []} | {}
013:    dup                                     {[], [istore_1], [istore_2], [astore_3], [astore 4], [astore 5], []} | {[new Object]}
014:    invokespecial Object.<init>             {[], [istore_1], [istore_2], [astore_3], [astore 4], [astore 5], []} | {[new Object], [dup]}
015:    astore 6                                {[], [istore_1], [istore_2], [astore_3], [astore 4], [astore 5], []} | {[new Object]}
016:    return                                  {[], [istore_1], [istore_2], [astore_3], [astore 4], [astore 5], [astore 6]} | {}
================================================================
```

我们暂时不考虑primitive type（`int`、`float`、`long`和`double`类型），只考虑reference type：

```pseudocode
模拟类型（抽象）: R - BasicInterpreter, BasicVerifier
模拟类型（具体）: Integer, String, Object - StackMapTable
模拟对象（具体）: Integer(1), Integer(2), String("AAA"), String("BBB"), Object(obj1), Object(obj2) - SimpleVerifier
模拟指令（超级具体）: 每个指令对应一个Value对象 - SourceInterpreter
```

那么`Value`的表达能力可以描述成这样：

```
         ┌─── abstract type
         │
         ├─── concrete type
Value ───┤
         ├─── concrete object
         │
         └─── instruction-based object
```

### 4.14.4 总结

本文内容总结如下：

- 第一点，把握`Interpreter`类的精妙之处就是结合`Frame`（功能切割）、opcode（方法定义）和`Value`（表达能力）来理解。
- 第二点，把握`Value`类的精妙之处在于理解四个不同层次的表达。

## 4.14 示例：检测潜在的NullPointerException
