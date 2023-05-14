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
