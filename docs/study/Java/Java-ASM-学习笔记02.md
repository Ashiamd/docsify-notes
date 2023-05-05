# Java-ASM-学习笔记02

> 视频：[Java ASM系列：（101）课程介绍-lsieun-lsieun-哔哩哔哩视频 (bilibili.com)](https://www.bilibili.com/list/1321054247?bvid=BV1Bo4y1S7U3&oid=377071342)
> 视频对应的gitee：[learn-java-asm: 学习Java ASM (gitee.com)](https://gitee.com/lsieun/learn-java-asm)
> 视频对应的文章：[Java ASM系列二：OPCODE | lsieun](https://lsieun.github.io/java/asm/java-asm-season-02.html) <= 笔记内容大部分都摘自该文章，下面不再重复声明
>
> ASM官网：[ASM (ow2.io)](https://asm.ow2.io/)
> ASM中文手册：[ASM4 使用手册(中文版) (yuque.com)](https://www.yuque.com/mikaelzero/asm)
>
> Java语言规范和JVM规范-官方文档：[Java SE Specifications (oracle.com)](https://docs.oracle.com/javase/specs/index.html)

# 1. 基础

## 1.1 课程研究主题

《Java ASM系列二：OPCODE》的研究主题是围绕着三个事物来展开：

- instruction
- `MethodVisitor.visitXxxInsn()`方法
- Stack Frame

那么，这三个事物之间是有什么样的关系呢？详细的来说，instruction、`MethodVisitor.visitXxxInsn()`方法和Stack Frame三个事物之间有内在的关联关系：

- 从一个具体`.class`文件的视角来说，它定义的每一个方法当中都包含实现代码逻辑的instruction内容。
- 从ASM的视角来说，ASM可以通过Class Generation或Class Transformation操作来生成一个具体的`.class`文件；那么，对于方法当中的instruction内容，应该使用哪些`MethodVisitor.visitXxxInsn()`方法来生成呢？
- 从JVM的视角来说，一个具体的`.class`文件需要加载到JVM当中来运行；在方法的运行过程当中，每一条instruction的执行，会对Stack Frame有哪些影响呢？

![Java ASM系列：（045）课程研究主题_ASM](https://s2.51cto.com/images/20210805/1628104966874244.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在[The Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)中，它对于具体`.class`文件提供了[ClassFile](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)结构的支持，对于JVM Execution Engine提供了[Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)的支持。在以后的学习当中，我们需要经常参照[Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)部分的内容。

## 1.2 ClassFile和Instruction

### 1.2.1 ClassFile对方法的约束

从ClassFile的角度来说，它对于方法接收的参数数量、方法体的大小做了约束。

#### 方法参数的数量（255）

在一个方法当中，方法接收的参数最多有255个。我们分成三种情况来进行说明：

- 第一种情况，对于non-static方法来说，`this`也要占用1个参数位置，因此接收的参数最多有254个参数。
- 第二种情况，对于static方法来说，不需要存储`this`变量，因此接收的参数最多有255个参数。
- 第三种情况，不管是non-static方法，还是static方法，`long`类型或`double`类型占据2个参数位置，所以实际的参数数量要小于255。

```java
public class HelloWorld {
  public void test(int a, int b) {
    // do nothing
  }

  public static void main(String[] args) {
    for (int i = 1; i <= 255; i++) {
      String item = String.format("int val%d,", i);
      System.out.println(item);
    }
  }
}
```

- 问题：能否在文档中找到依据呢？
- 回答：能。

在[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的[4.11. Limitations of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.11)部分对方法参数的数量进行了限制：

The number of method parameters is limited to **255** by the definition of a method descriptor ([§4.3.3](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3)), where the limit includes one unit for `this` in the case of instance or interface method invocations.

Note that a method descriptor is defined in terms of a notion of method parameter length in which a parameter of type `long` or `double` contributes two units to the length, so parameters of these types further reduce the limit.

#### 方法体的大小（65535）

对于方法来说，方法体并不是想写多少代码就写多少代码，它的大小也有一个限制：方法体内最多包含65535个字节。

当方法体的代码超过65535字节（bytes）时，会出现编译错误：code too large。

- 问题：能否在文档中找到依据呢？
- 回答：能。

在[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的[4.7.3. The Code Attribute](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.3)部分定义了`Code`属性：

```shell
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

- `code_length`: The value of the `code_length` item gives the number of bytes in the `code` array for this method. The value of `code_length` must be greater than **zero** (as the `code` array must not be empty) and less than **65536**.

### 1.2.2 Instruction VS. Opcode

严格的来说，instruction和opcode这两个概念是有区别的。相对来说，instruction是一个比较大的概念，而opcode是一个比较小的概念：

```
instruction = opcode + operands
```

- 问题：能否在文档中找到依据呢？
- 回答：能。

A Java Virtual Machine **instruction** consists of a one-byte **opcode** specifying the operation to be performed, followed by zero or more **operands** supplying arguments or data that are used by the operation. **Many instructions have no operands and consist only of an opcode.** （本段内容来自于[2.11. Instruction Set Summary](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11)的第一段）

粗略的来说，在现实生活当中谈论技术问题，我们经常混用instruction和opcode这两个概念，不作区分，可以将两者当成一回事儿。

### 1.2.3 Opcodes

#### 一个字节的容量（256）

opcode占用空间的大小为1 byte。在1 byte中，包含8个bit，因此1 byte最多可以表示256个值（即0~255）。

#### opcode的数量（205）

> [Chapter 7. Opcode Mnemonics by Opcode (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-7.html) 官方文档有完整的Opcode清单

虽然一个字节（byte)当中可以存储256个值（即0~255），但是在Java 8这个版本中定义的opcode数量只有205个。因此，opcode的内容就是一个“数值”，位于0~255之间。

#### opcode和mnemonic symbol

要记住每个opcode对应数值的含义，是非常困难的。为了方便人们记忆opcode的作用，就给每个opcode起了一个名字，叫作mnemonic symbol（助记符号）。

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 0      | nop             | 64     | lstore_1        | 128    | ior             | 192    | checkcast       |
| 1      | aconst_null     | 65     | lstore_2        | 129    | lor             | 193    | instanceof      |
| 2      | iconst_m1       | 66     | lstore_3        | 130    | ixor            | 194    | monitorenter    |
| 3      | iconst_0        | 67     | fstore_0        | 131    | lxor            | 195    | monitorexit     |
| 4      | iconst_1        | 68     | fstore_1        | 132    | iinc            | 196    | wide            |
| 5      | iconst_2        | 69     | fstore_2        | 133    | i2l             | 197    | multianewarray  |
| 6      | iconst_3        | 70     | fstore_3        | 134    | i2f             | 198    | ifnull          |
| 7      | iconst_4        | 71     | dstore_0        | 135    | i2d             | 199    | ifnonnull       |
| 8      | iconst_5        | 72     | dstore_1        | 136    | l2i             | 200    | goto_w          |
| 9      | lconst_0        | 73     | dstore_2        | 137    | l2f             | 201    | jsr_w           |
| 10     | lconst_1        | 74     | dstore_3        | 138    | l2d             | 202    | breakpoint      |
| 11     | fconst_0        | 75     | astore_0        | 139    | f2i             | 203    |                 |
| 12     | fconst_1        | 76     | astore_1        | 140    | f2l             | 204    |                 |
| 13     | fconst_2        | 77     | astore_2        | 141    | f2d             | 205    |                 |
| 14     | dconst_0        | 78     | astore_3        | 142    | d2i             | 206    |                 |
| 15     | dconst_1        | 79     | iastore         | 143    | d2l             | 207    |                 |
| 16     | bipush          | 80     | lastore         | 144    | d2f             | 208    |                 |
| 17     | sipush          | 81     | fastore         | 145    | i2b             | 209    |                 |
| 18     | ldc             | 82     | dastore         | 146    | i2c             | 210    |                 |
| 19     | ldc_w           | 83     | aastore         | 147    | i2s             | 211    |                 |
| 20     | ldc2_w          | 84     | bastore         | 148    | lcmp            | 212    |                 |
| 21     | iload           | 85     | castore         | 149    | fcmpl           | 213    |                 |
| 22     | lload           | 86     | sastore         | 150    | fcmpg           | 214    |                 |
| 23     | fload           | 87     | pop             | 151    | dcmpl           | 215    |                 |
| 24     | dload           | 88     | pop2            | 152    | dcmpg           | 216    |                 |
| 25     | aload           | 89     | dup             | 153    | ifeq            | 217    |                 |
| 26     | iload_0         | 90     | dup_x1          | 154    | ifne            | 218    |                 |
| 27     | iload_1         | 91     | dup_x2          | 155    | iflt            | 219    |                 |
| 28     | iload_2         | 92     | dup2            | 156    | ifge            | 220    |                 |
| 29     | iload_3         | 93     | dup2_x1         | 157    | ifgt            | 221    |                 |
| 30     | lload_0         | 94     | dup2_x2         | 158    | ifle            | 222    |                 |
| 31     | lload_1         | 95     | swap            | 159    | if_icmpeq       | 223    |                 |
| 32     | lload_2         | 96     | iadd            | 160    | if_icmpne       | 224    |                 |
| 33     | lload_3         | 97     | ladd            | 161    | if_icmplt       | 225    |                 |
| 34     | fload_0         | 98     | fadd            | 162    | if_icmpge       | 226    |                 |
| 35     | fload_1         | 99     | dadd            | 163    | if_icmpgt       | 227    |                 |
| 36     | fload_2         | 100    | isub            | 164    | if_icmple       | 228    |                 |
| 37     | fload_3         | 101    | lsub            | 165    | if_acmpeq       | 229    |                 |
| 38     | dload_0         | 102    | fsub            | 166    | if_acmpne       | 230    |                 |
| 39     | dload_1         | 103    | dsub            | 167    | goto            | 231    |                 |
| 40     | dload_2         | 104    | imul            | 168    | jsr             | 232    |                 |
| 41     | dload_3         | 105    | lmul            | 169    | ret             | 233    |                 |
| 42     | aload_0         | 106    | fmul            | 170    | tableswitch     | 234    |                 |
| 43     | aload_1         | 107    | dmul            | 171    | lookupswitch    | 235    |                 |
| 44     | aload_2         | 108    | idiv            | 172    | ireturn         | 236    |                 |
| 45     | aload_3         | 109    | ldiv            | 173    | lreturn         | 237    |                 |
| 46     | iaload          | 110    | fdiv            | 174    | freturn         | 238    |                 |
| 47     | laload          | 111    | ddiv            | 175    | dreturn         | 239    |                 |
| 48     | faload          | 112    | irem            | 176    | areturn         | 240    |                 |
| 49     | daload          | 113    | lrem            | 177    | return          | 241    |                 |
| 50     | aaload          | 114    | frem            | 178    | getstatic       | 242    |                 |
| 51     | baload          | 115    | drem            | 179    | putstatic       | 243    |                 |
| 52     | caload          | 116    | ineg            | 180    | getfield        | 244    |                 |
| 53     | saload          | 117    | lneg            | 181    | putfield        | 245    |                 |
| 54     | istore          | 118    | fneg            | 182    | invokevirtual   | 246    |                 |
| 55     | lstore          | 119    | dneg            | 183    | invokespecial   | 247    |                 |
| 56     | fstore          | 120    | ishl            | 184    | invokestatic    | 248    |                 |
| 57     | dstore          | 121    | lshl            | 185    | invokeinterface | 249    |                 |
| 58     | astore          | 122    | ishr            | 186    | invokedynamic   | 250    |                 |
| 59     | istore_0        | 123    | lshr            | 187    | new             | 251    |                 |
| 60     | istore_1        | 124    | iushr           | 188    | newarray        | 252    |                 |
| 61     | istore_2        | 125    | lushr           | 189    | anewarray       | 253    |                 |
| 62     | istore_3        | 126    | iand            | 190    | arraylength     | 254    | impdep1         |
| 63     | lstore_0        | 127    | land            | 191    | athrow          | 255    | impdep2         |

在上面表格当中，大家如果对任何一个opcode或mnemonic symbol感兴趣，都可以参阅[Chapter 6. The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)。

#### mnemonic symbol和类型信息

对于大多数的opcode，它的mnemonic symbol名字当中带有类型信息（type information）。一般情况下，规律如下：

- `i`表示`int`类型
- `l`表示`long`类型。
- `s`表示`short`类型
- `b`表示`byte`类型
- `c`表示`char`类型
- `f`表示`float`类型
- `d`表示`double`类型。
- `a`表示`reference`类型。`a`可能是address的首字母，表示指向内存空间的一个地址信息。

有一些opcode，它的mnemonic symbol名字不带有类型信息（type information），但是可以操作多种类型的数据，例如：

- `arraylength`，无论是`int[]`类型的数组，还是`String[]`类型的数组，获取数组的长度都是使用`arraylength`。
- `ldc`、`ldc_w`和`ldc2_w`表示**l**oa**d** **c**onstant的缩写，可以加载各种常量值。
- 与stack相关的指令，可以操作多种数据类型。`pop`表示出栈操作，`dup`相关的指令是duplicate单词的缩写，表示“复制”操作；`swap`表示“交换”。

还有一些opcode，它的mnemonic symbol名字不带有类型信息（type information），它也不处理任何类型的数据。例如：

- `goto`
- 问题：能否在文档中找到依据呢？
- 回答：能。

For the majority of **typed instructions**, the instruction type is represented explicitly in the opcode mnemonic by a letter: `i` for an `int` operation, `l` for long, `s` for `short`, `b` for `byte`, `c` for `char`, `f` for `float`, `d` for `double`, and `a` for `reference`. Some instructions for which the type is unambiguous do not have a type letter in their mnemonic. For instance, `arraylength` always operates on an object that is an array. Some instructions, such as `goto`, an unconditional control transfer, do not operate on typed operands.（本段内容来自于[2.11.1. Types and the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.1)的第二段）

### 1.2.4 总结

本文内容总结如下：

- 第一点，在ClassFile结构中，对于方法接收的参数和方法体的大小是有数量限制的。
- 第二点，instruction = opcode + operands
- 第三点，opcode占用1个byte大小，目前定义了205个；为了方便记忆，JVM文档为opcode提供了名字（即mnemonic symbol），大多数的名字中带有类型信息。
- 第四点，学习opcode是一个长期积累的过程，不能一蹴而就。所以，推荐大家依据感兴趣的内容，进行有选择性的学习。

## 1.3 ASM的MethodVisitor类

使用ASM，可以生成一个`.class`文件当中各个部分的内容。

```java
public class HelloWorld {
  public void test(String name, int age) {
    String line = String.format("name = '%s', age = %d", name, age);
    System.out.println(line);
  }
}
```

在这里，我们只关心方法的部分：

- 对于方法头的部分，我们可以使用`ClassVisitor.visitMethod(int access, String name, String descriptor, String signature, String[] exceptions)`方法来提供。
  - 其中的`access`参数提供访问标识信息，例如`public`
  - 其中的`name`参数提供方法的名字，例如`test`
  - 其中的`descriptor`参数提供方法的参数类型和返回值的类型
- 对于方法体的部分，我们可能通过使用`MethodVisitor`类来实现。
  - 如何得到一个`MethodVisitor`对象呢？`ClassVisitor.visitMethod()`的返回值是一个`MethodVisitor`类型的实例。
  - 方法体的instructions是如何添加的呢？通过调用`MethodVisitor.visitXxxInsn()`方法来提供的

对于`MethodVisitor`类来说，我们从两个方面来把握（回顾`MethodVisitor`类，可以参考[此处](https://lsieun.github.io/java-asm-01/method-visitor-intro.html)）：

- 第一方面，就是`MethodVisitor`类的`visitXxx()`方法的调用顺序。
- 第二方面，就是`MethodVisitor`类的`visitXxxInsn()`方法具体有哪些。

注意：`visitXxx()`方法表示`MethodVisitor`类当中所有以`visit`开头的方法，包含的方法比较多；而`visitXxxInsn()`方法是`visitXxx()`方法当中的一小部分，包含的方法比较较少。

### 1.3.1 方法的调用顺序

`MethodVisitor`类的`visitXxx()`方法要遵循一定的调用顺序：

```pseudocode
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

### 1.3.2 visitXxxInsn()方法

粗略的来说，`MethodVisitor`类有15个`visitXxxInsn()`方法。但严格的来说，有13个`visitXxxInsn()`方法，再加上`visitLabel()`和`visitTryCatchBlock()`这2个方法。

那么，这15个`visitXxxInsn()`方法，可以用来生成将近200个左右的opcode，也就是用来生成方法体的内容。

```java
public abstract class MethodVisitor {
    // (1)
    public void visitInsn(int opcode);
    // (2)
    public void visitIntInsn(int opcode, int operand);
    // (3)
    public void visitVarInsn(int opcode, int var);
    // (4)
    public void visitTypeInsn(int opcode, String type);
    // (5)
    public void visitFieldInsn(int opcode, String owner, String name, String descriptor);
    // (6)
    public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface);
    // (7)
    public void visitInvokeDynamicInsn(String name, String descriptor, Handle bootstrapMethodHandle, Object... bootstrapMethodArguments);
    // (8)
    public void visitJumpInsn(int opcode, Label label);
    // (9) 这里并不是严格的visitXxxInsn()方法
    public void visitLabel(Label label);
    // (10)
    public void visitLdcInsn(Object value);
    // (11)
    public void visitIincInsn(int var, int increment);
    // (12)
    public void visitTableSwitchInsn(int min, int max, Label dflt, Label... labels);
    // (13)
    public void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels);
    // (14)
    public void visitMultiANewArrayInsn(String descriptor, int numDimensions);
    // (15) 这里也并不是严格的visitXxxInsn()方法
    public void visitTryCatchBlock(Label start, Label end, Label handler, String type);
}
```

### 1.3.3 总结

本文主要是对ASM中`MethodVisitor`类进行回顾，内容总结如下：

- 第一点，`MethodVisitor`类的`visitXxx()`方法要遵循一定的调用顺序。
- 第二点，`MethodVisitor`类有15个`visitXxxInsn()`方法，用来生成将近200个左右的opcode。

## 1.4 JVM Architecture

### 1.4.1 JVM的组成部分

从JVM组成的角度来说，它由Class Loader SubSystem、Runtime Data Areas和Execution Engine三个部分组成：

- **类加载子系统（Class Loader SubSystem）**，负责加载具体的`.class`文件。
- **运行时数据区（Runtime Data Areas）**，主要负责为执行引擎（Execution Engine）提供“空间维度”的支持，为类（Class）、对象实例（object instance）、局部变量（local variable）提供存储空间。
- **执行引擎（Execution Engine）**，主要负责方法体里的instruction内容，它是JVM的核心部分。

![Java ASM系列：（048）JVM Architecture_Java](https://s2.51cto.com/images/20210811/1628667916643758.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在JVM当中，数据类型分成primitive type和reference type两种，那么ClassLoader负责加载哪些类型呢？

- <u>对于primitive type是JVM内置的类型，不需要ClassLoader加载</u>。
- 对于reference type来说，它又分成类（class types）、接口（interface types）和数组（array types）三种子类型。
  - **ClassLoader只负责加载类（class types）和接口（interface types）**。
  - <u>JVM内部会帮助我们创建数组（array types）</u>。

### 1.4.2 JVM Execution Engine

At the core of any Java Virtual Machine implementation is its **execution engine**.

#### JVM文档：指令集

在[The Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)中，它没有明确的提到Execution Engine的内容：打开JVM文档的每一章内容，搜索“execution engine”，会发现没有相关内容。

那么，这是不是意味着JVM文档与Execution Engine两者之间没有关系呢？其实，两者之间是有关系的。

可能是JVM文档描述的比较含蓄，它执行引擎（Execution Engine）的描述是通过指令集（Instruction Set）来体现的。

In the Java Virtual Machine specification, **the behavior of the execution engine** is defined in terms of **an instruction set**.

#### 执行引擎：三种解读

The term “**execution engine**” can also be used in any of three senses: **an abstract specification**, **a concrete implementation**, or **a runtime instance**.

- **The abstract specification** defines the behavior of an execution engine in terms of the instruction set.
- **Concrete implementations**, which may use a variety of techniques, are either software, hardware, or a combination of both.
- **A runtime instance** of an execution engine is a thread.

<u>**Each thread of a running Java application is a distinct instance of the virtual machine’s execution engine**. From the beginning of its lifetime to the end, a thread is either executing bytecodes or native methods. A thread may execute bytecodes directly, by interpreting or executing natively in silicon, or indirectly, by just-in-time compiling and executing the resulting native code</u>.

A Java Virtual Machine implementation may use other threads invisible to the running application, such as a thread that performs garbage collection. Such threads need not be “instances” of the implementation’s execution engine. **All threads that belong to the running application, however, are execution engines in action.**

### 1.4.3 Runtime Data Areas: JVM Stack

在现实生活当中，我们生活在一个三维的空间，在这个空间维度里，可以确定一个事物的具体位置；同时，也有一个时间维度，随着时间的流逝，这个事物的状态也会发生变化。**简单来说，对于一个具体事物，空间维度上就是看它占据一个什么位置，时间维度上就看它如何发生变化。** 接下来，我们把“空间维度”和“时间维度”的视角带入到JVM当中。

在JVM当中，是怎么体现“空间维度”视角和“时间维度”两个视角的呢？

- 时间维度。上面谈到执行引擎（Execution Engine），一条一条的执行instruction的内容，会引起相应事物的状态发生变化，这就是“时间维度”的视角。
- 空间维度。接下来要讲的JVM Stack和Stack Frame，它们都是运行时数据区（Runtime Data Areas）具体的内存空间分配，用于存储相应的数据，这就是“空间维度”视角。

![Java ASM系列：（048）JVM Architecture_Java](https://s2.51cto.com/images/20210811/1628667916643758.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

#### thread对应于JVM Stack

上面提到，线程（thread）是执行引擎（Execution Engine）的运行实例，那么线程（thread）就是一个“时间维度”的视角。JVM Stack是运行时数据区（Runtime Data Areas）的一部分，是“空间维度”的视角。两者之间是什么样的关系呢？

**Each Java Virtual Machine thread has a private Java Virtual Machine stack, created at the same time as the thread.**

那么，对于线程（thread）来说，它就同时具有“时间维度”（Execution Engine）和“空间维度”(JVM Stack)。

A Java Virtual Machine stack stores frames.

#### method对应于Stack Frame

接着，我们如何看待方法（method）呢？或者说方法（method）是什么呢？方法（method），是一组instruction内容的有序集合；而instruction的执行，就对应着时间的流逝，所以方法（method）也可以理解成一个“时间片段”。

那么，线程（thread）和方法（method）是什么关系呢？线程（thread），从本质上来说，就是不同方法（method）之间的调用。所以，线程（thread）是更大的“时间片段”，而方法（method）是较小的“时间片段”。

上面描述，体现方法（method）是在“时间维度”的考量，在“空间维度”上有哪些体现呢？在“空间维度”上，就体现为Stack Frame。

- **A new frame is created each time a method is invoked.**
- **A frame is destroyed when its method invocation completes, whether that completion is normal or abrupt (it throws an uncaught exception).**

接下来，介绍current frame、current method和current class三个概念。

Only one frame, the frame for the executing method, is active at any point in a given thread of control. This frame is referred to as the **current frame**, and its method is known as the **current method**. The class in which the current method is defined is the **current class**.

一个方法会调用另外一个方法，另一个方法也有执行结束的时候。那么，current frame是如何变换的呢？

- A frame ceases to be current if its method invokes another method or if its method completes.
- When a method is invoked, a new frame is created and becomes current when control transfers to the new method.
- On method return, the current frame passes back the result of its method invocation, if any, to the previous frame.
- The current frame is then discarded as the previous frame becomes the current one.

> [Chapter 2. The Structure of the Java Virtual Machine (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.6) => 2.6. Frames 有介绍上面这段话
>
> Note that a frame created by a thread is local to that thread and cannot be referenced by any other thread.

### 1.4.4 Runtime Data Areas: Stack Frame

当一个具体的`.class`加载到JVM当中之后，方法的执行会对应于JVM当中一个Stack Frame内存空间。

#### Stack Frame的内部结构

对于Stack Frame内存空间，主要分成三个子区域：

- 第一个子区域，operand stack，是一个栈结构，遵循后进先出（LIFO）的原则；它的大小是由`Code`属性中的`max_stack`来决定的。
- 第二个子区域，local variables，是一个数组结构，通过索引值来获取和设置里面的数据，其索引值是从`0`开始；它的大小是由`Code`属性中的`max_locals`来决定的。
- 第三个子区域，Frame Data，它用来存储与当前方法相关的数据。例如，一个指向runtime constant pool的引用、出现异常时的处理逻辑（exception table）。

在Frame Data当中，我们也列出了其中两个重要数据信息：

- 第一个数据，`instructions`，它表示指令集合，是由`Code`属性中的`code[]`解析之后的结果。（准确的来说，这里是不对的，instructions可能是位于Method Area当中。我们为了看起来方便，把它放到了这里。）
- 第二个数据，`ref`，它是一个指向runtime constant pool的引用。这个runtime constant pool是由具体`.class`文件中的constant pool解析之后的结果。

![Java ASM系列：（048）JVM Architecture_ASM_03](https://s2.51cto.com/images/20210811/1628668011678650.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

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

#### Stack Frame内的数据类型

在这里，我们要区分开两个概念：**存储时的类型** 和 **运行时的类型**。

将一个具体`.class`文件加载进JVM当中，存放数据的地方有两个主要区域：堆（Heap Area）和栈（Stack Area）。

- 在堆（Heap Area）上，存放的就是Actual type，就是“存储时的类型”。例如，`byte`类型就是占用1个byte，`int`类型就是占用4个byte。
- 在栈（Stack Area）上，更确切的说，就是Stack Frame当中的operand stack和local variables区域，存放的就是Computational type。这个时候，类型就发生了变化，`boolean`、`byte`、`char`、`short`都会被转换成`int`类型来进行计算。

在方法执行的时候，或者说方法里的instruction在执行的时候，需要将相关的数据加载到Stack Frame里；更进一步的说，就是将数据加载到operand stack和local variables两个子区域当中。

![Java ASM系列：（048）JVM Architecture_Java_04](https://s2.51cto.com/images/20210811/1628668049730555.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

当数据加载到operand stack和local variables当中时，需要注意三点：

- **第一点，`boolean`, `byte`, `short`, `char`, or `int`，这几种类型，在local variable和operand stack当中，都是作为`int`类型进行处理。**
- **第二点，`int`、`float`和`reference`类型，在local variable和operand stack当中占用1个位置**
- **第三点，`long`和`double`类型，在local variable和operand stack当中占用2个位置**

<u>举个例子，在一个类当中，有一个`byte`类型的字段。将该类加载进JVM当中，然后创建该类的对象实例，那么该对象实例是存储在堆（Heap Area）上的，其中的字段就是`byte`类型。当程序运行过程中，会使用到该对象的字段，这个时候就要将`byte`类型的值转换成`int`类型进行计算；计算完成之后，需要将值存储到该对象的字段当中，这个时候就会将`int`类型再转换成`byte`类型进行存储</u>。

另外，对于Category为`1`的类型，在operand stack和local variables当中占用1个slot的位置；对于对于Category为`2`的类型，在operand stack和local variables当中占用2个slot的位置。

下表的内容是来自于[Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)的[Table 2.11.1-B. Actual and Computational types in the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.1-320)部分。

| Actual type | Computational type | Category |
| ----------- | ------------------ | -------- |
| boolean     | int                | 1        |
| byte        | int                | 1        |
| char        | int                | 1        |
| short       | int                | 1        |
| int         | int                | 1        |
| float       | float              | 1        |
| reference   | reference          | 1        |
| long        | long               | 2        |
| double      | double             | 2        |

在[2.11.1. Types and the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.1)谈到：most operations on values of **actual types** `boolean`, `byte`, `char`, and `short` are correctly performed by instructions operating on values of **computational type** `int`.

对于local variable是这样描述的：（内容来自于[2.6.1. Local Variables](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.1)）

- A single local variable can hold a value of type `boolean`, `byte`, `char`, `short`, `int`, `float`, `reference`, or `returnAddress`.
- A pair of local variables can hold a value of type `long` or `double`.

对于operand stack是这样描述的：（内容来自于[2.6.2. Operand Stacks](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.2)）

- Each entry on the operand stack can hold a value of any Java Virtual Machine type, including a value of type `long` or type `double`.
- At any point in time, an operand stack has an associated depth, where a value of type `long` or `double` contributes two units to the depth and a value of any other type contributes one unit.

### 1.4.5 总结

本文内容总结如下：

- 第一点，**Execution Engine是JVM的核心；JVM文档中的Instruction Set就是对Execution Engine的行为描述；线程就是Execution Engine的运行实例**。
- 第二点，**线程所对应的内存空间是JVM Stack，方法所对应的内存空间是Stack Frame**。
- 第三点，在Stack Frame当中，分成local variable、operand stack和frame data三个子区域；在local variable和operand stack中，要注意数据的计算类型和占用的空间大小。

## 1.5 JVM Execution Model

### 1.5.1 Execution Model

#### 什么是Execution Model

在[asm4-guide.pdf](https://asm.ow2.io/asm4-guide.pdf)文档的`3.1.1. Execution Model`部分提到了Execution Model。

那么，Execution Model是什么呢？其实，**Execution Model就是指Stack Frame简化之后的模型**。如何“简化”呢？也就是，我们不需要去考虑Stack Frame的技术实现细节，把它想像一个理想的模型就可以了。

针对Execution Model或Stack Frame，我们可以理解成它由local variable和operand stack两个部分组成，或者说理解成它由local variable、operand stack和frame data三个部分组成。换句话说，local variable和operand stack是两个必不可少的部分，而frame data是一个相对来说不那么重要的部分。在一般的描述当中，都是将Stack Frame描述成local variable和operand stack两个部分；但是，如果我们为了知识的完整性，就可以考虑添加上frame data这个部分。

![Java ASM系列：（049）JVM Execution Model_Opcode](https://s2.51cto.com/images/20210819/1629358156974803.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

另外，方法的执行与Stack Frame之间有一个非常紧密的关系：

- 一个方法的调用开始，就对应着Stack Frame的内存空间的分配。
- 一个方法的执行结束，无论正常结束（return），还是异常退出（throw exception），都表示着相应的Stack Frame内存空间被销毁。

接下来，我们就通过两个方面来把握Stack Frame的状态：

- 第一个方面，方法刚进入的时候，任何的instruction都没有执行，那么Stack Frame是一个什么样的状态呢？
- 第二个方面，在方法开始执行后，这个时候instruction开始执行，每一条instruction的执行，会对Stack Frame的状态产生什么样的影响呢？

#### 方法的初始状态

在方法进入的时候，会生成相应的Stack Frame内存空间。那么，Stack Frame的初始状态是什么样的呢？（点击[这里](https://lsieun.github.io/java-asm-01/method-initial-frame.html)查看之前内容）

在Stack Frame当中，operand stack是空的，而local variables则需要考虑三方面的因素：

- 当前方法是否为static方法。
  - 如果当前方法是non-static方法，则需要在local variables索引为`0`的位置存在一个`this`变量，后续的内容从`1`开始存放。
  - 如果当前方法是static方法，则不需要存储`this`，因此后续的内容从`0`开始存放。
- 当前方法是否接收参数。方法接收的参数，会按照参数的声明顺序放到local variables当中。
- 方法参数是否包含`long`或`double`类型。如果方法的参数是`long`或`double`类型，那么它在local variables当中占用两个位置。
- 问题：能否在文档中找到依据呢？
- 回答：能。

The Java Virtual Machine uses **local variables** to pass parameters on **method invocation**. On **class method invocation**, any parameters are passed in consecutive local variables starting from local variable `0`. On **instance method invocation**, local variable `0` is always used to pass a reference to the object on which the instance method is being invoked (`this` in the Java programming language). Any parameters are subsequently passed in consecutive local variables starting from local variable `1`.（内容来自于[2.6.1. Local Variables](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.1)）

**The operand stack is empty** when the frame that contains it is created.（内容来自于[2.6.2. Operand Stacks](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.2)）

#### 方法的后续变化

方法的后续变化，就是在方法初始状态的基础上，随着instruction的执行而对local variable和operand stack的状态产生影响。

当方法执行时，就是将instruction一条一条的执行：

- 第一步，获取instruction。每一条instruction都是从`instructions`内存空间中取出来的。
- 第二步，执行instruction。对于instruction的执行，就会引起operand stack和local variables的状态变化。
  - 在执行instruction过程中，需要获取相关资源。通过`ref`可以获取runtime constant pool的“资源”，例如一个字符串的内容，一个指向方法的物理内存地址。

在[Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html)的[2.11. Instruction Set Summary](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11)部分，对程序的执行进行了如下描述：

Ignoring exceptions, the inner loop of a Java Virtual Machine interpreter is effectively:

```pseudocode
do {
    atomically calculate pc and fetch opcode at pc;
    if (operands) fetch operands;
    execute the action for the opcode;
} while (there is more to do);
```

![Java ASM系列：（049）JVM Execution Model_Opcode](https://s2.51cto.com/images/20210819/1629358156974803.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

需要注意的是，虽然local variable和operand stack是Stack Frame当中两个最重要的结构，两者是处于一个平等的地位上，缺少任何一个都无法正常工作；但是，从使用频率的角度来说，两者还是有很大的差别。先举个生活当中的例子，operand stack类似于“公司”，local variables类似于“临时租的房子”，虽然说“公司”和“临时租的房子”是我们经常待的场所，是两个非常重要的地方，但是我们工作的时间大部是在“公司”进行的，少部分工作时间是在“家”里进行。也就是说，<u>大多数情况下，都是先把数据加载到operand stack上，在operand stack上进行运算，最后可能将数据存储到local variables当中。只有少部分的操作（例如`iinc`），只需要在local variable上就能完成</u>。所以从使用频率的角度来说，**operand stack是进行工作的“主战场”，使用频率就比较高，大多数工作都是在它上面完成；而local variable使用频率就相对较低，它只是提供一个临时的数据存储区域**。

### 1.5.2 查看方法的Stack Frame变化

在这个部分，我们介绍一下如何使用`HelloWorldFrameCore02`类查看方法对应的Stack Frame的变化。

#### 查看Frame变化的工具类

在课程代码中，查看方法对应的Stack Frame的变化，有两个类的版本：

- 第一个版本，是`HelloWorldFrameCore`类。它是在《Java ASM系列一：Core API》阶段引入的类，可以用来打印方法的Stack Frame的变化。为了保证与以前内容的一致性，我们保留了这个类的代码逻辑不变动。
- 第二个版本，是`HelloWorldFrameCore02`类。它是在《Java ASM系列二：OPCODE》阶段引入的类，在第一个版本的基础上进行了改进：引入了instruction部分，精简了Stack Frame的类型显示。

我们在使用的时候，直接使用第二个版本就可以了，也就是使用`HelloWorldFrameCore02`类。

#### Frame变化过程输出

我们在执行`HelloWorldFrameCore02`类之后，输出结果分成三个部分：

我们在执行`HelloWorldFrameCore02`类之后，输出结果分成三个部分：

- 第一部分，是offset，它表示某一条instruction的具体位置或偏移量。
- 第二部分，是instructions，它表示方法里包含的所有指令信息。
- 第三部分，是local variable和operand stack中存储的具体数据类型。
  - 格式：`{local variable types} | {operand stack types}`
  - 第一行的local variable和operand stack表示“方法的初始状态”。
  - 其后每一行instruction
    - 上一行的local variable和operand stack表示该instruction执行之前的状态
    - 与该instruction位于同一行local variable和operand stack表示该instruction执行之后的状态

![Java ASM系列：（049）JVM Execution Model_Java_03](https://s2.51cto.com/images/20210819/1629358215407158.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在上面的输出结果中，我们会看到local variable和operand stack为`{} | {}`的情况，这是四种特殊情况。

#### Frame清空LVR和OS的四种情况

在`HelloWorldFrameCore02`类当中，会间接使用到`AnalyzerAdapter`类。在`AnalyzerAdapter`类的代码中，将`locals`和`stack`字段的取值设置为`null`的情况，就会有上面`{} | {}`的情况。

在`AnalyzerAdapter`类的代码中，有四个方法会将`locals`和`stack`字段设置为`null`：

- 在`AnalyzerAdapter.visitInsn(int opcode)`方法中，当`opcode`为`return`或`athrow`的情况
- 在`AnalyzerAdapter.visitJumpInsn(int opcode, Label label)`方法中，当`opcode`为`goto`的情况
- 在`AnalyzerAdapter.visitTableSwitchInsn(int min, int max, Label dflt, Label... labels)`方法中
- 在`AnalyzerAdapter.visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels)`方法中

#### 五种理解代码的视角

在后续内容中，我们介绍代码示例的时候，一般都会从五个视角来学习：

- 第一个视角，Java语言的视角，就是`sample.HelloWorld`里的代码怎么编写。
- 第二个视角，Instruction的视角，就是`javap -c sample.HelloWorld`，这里给出的就是标准的opcode内容。
- 第三个视角，ASM的视角，就是编写ASM代码实现某种功能，这里主要是对`visitXxxInsn()`方法的调用，与实际的opcode可能相同，也可能有差异。
- 第四个视角，Frame的视角，就是JVM内存空间的视角，就是local variable和operand stack的变化。
- 第五个视角，JVM Specification的视角，参考JVM文档，它是怎么说的。

第一个视角，Java语言的视角。假如我们有一个`sample.HelloWorld`类，代码如下：

```java
package sample;

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

第二个视角，Instruction的视角。我们可以通过`javap -c sample.HelloWorld`命令查看方法包含的instruction内容：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(boolean);
    Code:
       0: iload_1
       1: ifeq          15 (计算之后的值)
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String value is true
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: goto          23 (计算之后的值)
      15: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #5                  // String value is false
      20: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: return
}
```

第三个视角，ASM的视角。运行`ASMPrint`类，可以查看ASM代码，可以查看某一个opcode具体对应于哪一个`MethodVisitor.visitXxxInsn()`方法：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitJumpInsn(IFEQ, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("value is true");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("value is false");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

第四个视角，Frame的视角。我们可以通过运行`HelloWorldFrameCore02`类来查看方法对应的Stack Frame的变化：

```shell
test:(Z)V
                               // {this, int} | {}
0000: iload_1                  // {this, int} | {int}
0001: ifeq            14(真实值)// {this, int} | {}
0004: getstatic       #2       // {this, int} | {PrintStream}
0007: ldc             #3       // {this, int} | {PrintStream, String}
0009: invokevirtual   #4       // {this, int} | {}
0012: goto            11(真实值)// {} | {}
                               // {this, int} | {}
0015: getstatic       #2       // {this, int} | {PrintStream}
0018: ldc             #5       // {this, int} | {PrintStream, String}
0020: invokevirtual   #4       // {this, int} | {}
                               // {this, int} | {}
0023: return                   // {} | {}
```

> 注意，`.class`存储的是跳转的偏移量，需要将当前offset+跳转的偏移量，才能得到实际跳转的位置

第五个视角，JVM Specification的视角。我们可以参考[Chapter 6. The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)文档，查看具体的opcode的内容，主要是查看opcode的Format和Operand Stack。

Format

```
mnemonic
operand1
operand2
...
```

Operand Stack

```
..., value1, value2 →

..., value3
```

### 1.5.3 总结

本文内容总结如下：

- 第一点，Execution Model就是对Stack Frame进行简单、理想化之后的模型；对于Stack Frame来说，我们要关注方法的初始状态和方法的后续变化。
- 第二点，通过运行`HelloWorldFrameCore02`类可以查看具体方法的Stack Frame变化。

# 2. OPCODE

在JVM文档中，一共定义了205个[opcode](https://lsieun.github.io/static/java/opcode.html)，内容比较多，我们可以根据自己的兴趣进行有选择性的学习。在下面文章的标题后面都带有`(m/n/sum)`标识，其中，`m`表示当前文章当中介绍多少个opcode，`n`表示到目前为止介绍了多少个opcode，`sum`表示一共有多少个opcode。

## 2.1 opcode: return (6/6/205)

### 2.1.1 概览

从Instruction的角度来说，与return相关的opcode有6个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 172    | ireturn         | 174    | freturn         | 176    | areturn         |
| 173    | lreturn         | 175    | dreturn         | 177    | return          |

从ASM的角度来说，这些opcode是通过`MethodVisitor.visitInsn(int opcode)`方法来调用的。

### 2.1.2 return void type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    // do nothing
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(0, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

[empty]
```

The current method must have return type `void`.

- If no exception is thrown, **any values on the operand stack of the current frame are discarded.**
- The interpreter then returns control to the invoker of the method, reinstating the frame of the invoker.

### 2.1.3 return primitive type

#### ireturn

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public int test() {
    return 0;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public int test();
    Code:
       0: iconst_0
       1: ireturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(IRETURN);
methodVisitor.visitMaxs(1, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_0                 // {this} | {int}
0001: ireturn                  // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
..., value →

[empty]
```

- The current method must have return type `boolean`, `byte`, `short`, `char`, or `int`. The `value` must be of type `int`.
- **If no exception is thrown, `value` is popped from the operand stack of the current frame and pushed onto the operand stack of the frame of the invoker.**
- Any other values on the operand stack of the current method are discarded.
- The interpreter then returns control to the invoker of the method, reinstating the frame of the invoker.

注意：这里只要是`boolean`, `byte`, `short`, `char`, or `int`，都会以相同的指令被处理（即被转成int处理）

#### freturn

freturn、lreturn、dreturn描述都类似，就是指令和要求的类型不同，这里省略

#### lreturn

freturn、lreturn、dreturn描述都类似，就是指令和要求的类型不同，这里省略

#### dreturn

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public double test() {
        return 0;
    }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public double test();
    Code:
       0: dconst_0
       1: dreturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(DCONST_0);
methodVisitor.visitInsn(DRETURN);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: dconst_0                 // {this} | {double, top}
0001: dreturn                  // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
..., value →

[empty]
```

- The current method must have return type `double`. The `value` must be of type `double`.
- If no exception is thrown, `value` is popped from the operand stack of the current frame. The `value` is pushed onto the operand stack of the frame of the invoker.
- Any other values on the operand stack of the current method are discarded.
- The interpreter then returns control to the invoker of the method, reinstating the frame of the invoker.

### 2.1.4 return reference type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public Object test() {
    return null;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public java.lang.Object test();
    Code:
       0: aconst_null
       1: areturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ACONST_NULL);
methodVisitor.visitInsn(ARETURN);
methodVisitor.visitMaxs(1, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: aconst_null              // {this} | {null}
0001: areturn                  // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
..., objectref →

[empty]
```

- **The `objectref` must be of type `reference` and must refer to an object of a type that is assignment compatible with the type represented by the return descriptor of the current method.**
- If no exception is thrown, `objectref` is popped from the operand stack of the current frame and pushed onto the operand stack of the frame of the invoker.
- Any other values on the operand stack of the current method are discarded.
- The interpreter then reinstates the frame of the invoker and returns control to the invoker.

## 2.2 opcode: constant (20/26/205)

### 2.2.1 概述

从Instruction的角度来说，与constant相关的opcode有20个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 1      | aconst_null     | 6      | iconst_3        | 11     | fconst_0        | 16     | bipush          |
| 2      | iconst_m1       | 7      | iconst_4        | 12     | fconst_1        | 17     | sipush          |
| 3      | iconst_0        | 8      | iconst_5        | 13     | fconst_2        | 18     | ldc             |
| 4      | iconst_1        | 9      | lconst_0        | 14     | dconst_0        | 19     | ldc_w           |
| 5      | iconst_2        | 10     | lconst_1        | 15     | dconst_1        | 20     | ldc2_w          |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`: `aconst_null`, `iconst_<i>`, `lconst_<l>`, `fconst_<f>`, `dconst_<d>`
- `MethodVisitor.visitIntInsn()`: `bipush`, `sipush`
- `MethodVisitor.visitLdcInsn()`: `ldc`, `ldc_w`, `ldc2_w`

> 在B站视频[Java ASM系列：（116）constant04](https://www.bilibili.com/video/BV1L64y1v7jB#reply94499748336)当中， [伊ㅅ钦ㅇ](https://space.bilibili.com/430685850)同学说：
>
> **`ldc_w`这个`w`，如果是按王爽老师的8086汇编来看，可以理解为一个字（word）2个字节，可能联想记忆比较好记点**。

### 2.2.2 int

注意，下面几个小节里面，iconst、bipush、sipush、ldc都列举了一个范围，是因为在指定范围内，JVM会优先使用对应的指令，减少`.class`文件的大小，节省内存占用。

#### iconst_`<i>`: -1~5

> i即int，i const 即 int 常量

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int int_m1 = -1;
    int int_0 = 0;
    int int_1 = 1;
    int int_2 = 2;
    int int_3 = 3;
    int int_4 = 4;
    int int_5 = 5;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_m1 // m 即 minus 负数的意思
       1: istore_1 // non-static方法，local variables 0位置是this指针，所以方法体内第一个变量存到1的位置
       2: iconst_0
       3: istore_2
       4: iconst_1
       5: istore_3
       6: iconst_2
       7: istore        4
       9: iconst_3
      10: istore        5
      12: iconst_4
      13: istore        6
      15: iconst_5
      16: istore        7
      18: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_M1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 4);
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitVarInsn(ISTORE, 5);
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitVarInsn(ISTORE, 6);
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitVarInsn(ISTORE, 7);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 8);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

> 注意，这里看着好像local variables的空间越变越大，但是实际一开始就是申请了8个位置的大小，只是后续才填装数据而已

```shell
                               // {this} | {}
0000: iconst_m1                // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_0                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iconst_1                 // {this, int, int} | {int}
0005: istore_3                 // {this, int, int, int} | {}
0006: iconst_2                 // {this, int, int, int} | {int}
0007: istore          4        // {this, int, int, int, int} | {}
0009: iconst_3                 // {this, int, int, int, int} | {int}
0010: istore          5        // {this, int, int, int, int, int} | {}
0012: iconst_4                 // {this, int, int, int, int, int} | {int}
0013: istore          6        // {this, int, int, int, int, int, int} | {}
0015: iconst_5                 // {this, int, int, int, int, int, int} | {int}
0016: istore          7        // {this, int, int, int, int, int, int, int} | {}
0018: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., <i>
```

Push the `int` constant `<i>` (-1, 0, 1, 2, 3, 4 or 5) onto the operand stack.

#### bipush: -128~127

> b即byte，i即int，即byte数据被当作int，然后push到operand stack

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int int_m128 = -128;
    int int_127 = 127;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: bipush        -128
       2: istore_1
       3: bipush        127
       5: istore_2
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitIntInsn(BIPUSH, -128);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitIntInsn(BIPUSH, 127);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: bipush          -128     // {this} | {int} 
0002: istore_1                 // {this, int} | {} // 可以看到在 local variables 和 operand stack中被当作int处理
0003: bipush          127      // {this, int} | {int}
0005: istore_2                 // {this, int, int} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

The immediate byte is sign-extended to an `int` value. That `value` is pushed onto the operand stack.

> 即实际加载到operand stack和放入local variables时，会被当作`int`进行处理

Format

```
bipush
byte
```

#### sipush: -32768~32767

> s 即 shot，i 即 int，short数据被当作int处理，push到operand stack

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int int_m32768 = -32768;
    int int_32767 = 32767;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: sipush        -32768
       3: istore_1
       4: sipush        32767
       7: istore_2
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitIntInsn(SIPUSH, -32768);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitIntInsn(SIPUSH, 32767);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: sipush          -32768   // {this} | {int} 
0003: istore_1                 // {this, int} | {}
0004: sipush          32767    // {this, int} | {int} // 可以看到在 local variables 和 operand stack中被当作int处理
0007: istore_2                 // {this, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```
... →

..., value
```

Format

> short数据占用2字节，所以用 byte1 和 byte 两个字节表示数值

```shell
sipush
byte1
byte2
```

The immediate unsigned `byte1` and `byte2` values are assembled into an intermediate `short`, where the `value` of the `short` is `(byte1 << 8) | byte2`. The intermediate `value` is then sign-extended to an `int` value. That `value` is pushed onto the operand stack.

#### ldc: MIN~MAX

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int int_min = Integer.MIN_VALUE;
    int int_max = Integer.MAX_VALUE;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意这里ldc加载的是Frame Data中指向常量池数值数据的指针，这里会根据指针获取到具体的数值

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #3                  // int -2147483648
       2: istore_1
       3: ldc           #4                  // int 2147483647
       5: istore_2
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

> 注意ASM代码并没有要求我们计算数据对应的常量池索引位置是什么，因为ASM生成`.class`的时候会自动帮我们计算

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Integer(-2147483648));
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitLdcInsn(new Integer(2147483647));
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc             #3       // {this} | {int}
0002: istore_1                 // {this, int} | {}
0003: ldc             #4       // {this, int} | {int}
0005: istore_2                 // {this, int, int} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

Format

> 注意，这里index是个指向运行时常量池中数据的指针，而不是具体的数值

```shell
ldc
index
```

The `index` is an unsigned byte that must be a valid index into the run-time constant pool of the current class. The run-time constant pool entry at `index` either must be a run-time constant of type `int` or `float`, or **a reference to a string literal**, or **a symbolic reference to a class, method type, or method handle**.

### 2.2.3 long

#### lconst_`<l>`: 0~1

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long long_0 = 0;
    long long_1 = 1;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: lconst_0
       1: lstore_1
       2: lconst_1
       3: lstore_3
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(LCONST_0);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitInsn(LCONST_1);
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: lconst_0                 // {this} | {long, top}
0001: lstore_1                 // {this, long, top} | {}
0002: lconst_1                 // {this, long, top} | {long, top} // 注意 long在 常量池中、local variables、operand stack都是占用2个位置（从javap指令可以看出来long的下一个常量池的标记是+2位置，而不是连续的+1）
0003: lstore_3                 // {this, long, top, long, top} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., <l>
```

Push the `long` constant `<l>` (0 or 1) onto the operand stack.

#### ldc2_w: MIN~MAX

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long long_min = Long.MIN_VALUE;
    long long_max = Long.MAX_VALUE;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc2_w        #3                  // long -9223372036854775808l
       3: lstore_1
       4: ldc2_w        #5                  // long 9223372036854775807l
       7: lstore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Long(-9223372036854775808L));
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitLdcInsn(new Long(9223372036854775807L));
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc2_w          #3       // {this} | {long, top}
0003: lstore_1                 // {this, long, top} | {}
0004: ldc2_w          #5       // {this, long, top} | {long, top}
0007: lstore_3                 // {this, long, top, long, top} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

The numeric `value` of that run-time constant is pushed onto the operand stack as a `long` or `double`, respectively.

Format

> 注意，long和double都是占用2个槽位（2 * 32 bit），这里indexbyte1和indexbyte2是 ldc_2w这个opcode操作码的operands操作数（常量池中的索引）

```shell
ldc2_w
indexbyte1
indexbyte2
```

The unsigned `indexbyte1` and `indexbyte2` are assembled into an unsigned 16-bit `index` into the run-time constant pool of the current class, where the value of the `index` is calculated as `(indexbyte1 << 8) | indexbyte2`. The `index` must be a valid index into the run-time constant pool of the current class.

### 2.2.4 float

#### fconst_`<f>`: 0~2

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    float float_0 = 0;
    float float_1 = 1;
    float float_2 = 2;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: fconst_0
       1: fstore_1
       2: fconst_1
       3: fstore_2
       4: fconst_2
       5: fstore_3
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(FCONST_0);
methodVisitor.visitVarInsn(FSTORE, 1);
methodVisitor.visitInsn(FCONST_1);
methodVisitor.visitVarInsn(FSTORE, 2);
methodVisitor.visitInsn(FCONST_2);
methodVisitor.visitVarInsn(FSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: fconst_0                 // {this} | {float}
0001: fstore_1                 // {this, float} | {}
0002: fconst_1                 // {this, float} | {float}
0003: fstore_2                 // {this, float, float} | {}
0004: fconst_2                 // {this, float, float} | {float}
0005: fstore_3                 // {this, float, float, float} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., <f>
```

Push the `float` constant `<f>` (0.0, 1.0, or 2.0) onto the operand stack.

#### ldc: MIN~MAX

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    float float_min = Float.MIN_VALUE;
    float float_max = Float.MAX_VALUE;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #3                  // float 1.4E-45f
       2: fstore_1
       3: ldc           #4                  // float 3.4028235E38f
       5: fstore_2
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Float("1.4E-45"));
methodVisitor.visitVarInsn(FSTORE, 1);
methodVisitor.visitLdcInsn(new Float("3.4028235E38"));
methodVisitor.visitVarInsn(FSTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc             #3       // {this} | {float}
0002: fstore_1                 // {this, float} | {}
0003: ldc             #4       // {this, float} | {float}
0005: fstore_2                 // {this, float, float} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```
... →

..., value
```

Format

```shell
ldc
index
```

The `index` is an unsigned byte that must be a valid index into the run-time constant pool of the current class. The run-time constant pool entry at `index` either must be a run-time constant of type `int` or `float`, or **a reference to a string literal**, or **a symbolic reference to a class, method type, or method handle**.

### 2.2.5 double

#### dconst_`<d>`: 0~1

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    double double_0 = 0;
    double double_1 = 1;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: dconst_0
       1: dstore_1
       2: dconst_1
       3: dstore_3
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(DCONST_0);
methodVisitor.visitVarInsn(DSTORE, 1);
methodVisitor.visitInsn(DCONST_1);
methodVisitor.visitVarInsn(DSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: dconst_0                 // {this} | {double, top}
0001: dstore_1                 // {this, double, top} | {}
0002: dconst_1                 // {this, double, top} | {double, top}
0003: dstore_3                 // {this, double, top, double, top} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., <d>
```

Push the `double` constant `<d>` (0.0 or 1.0) onto the operand stack.

#### ldc2_w: MIN~MAX

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    double double_min = Double.MIN_VALUE;
    double double_max = Double.MAX_VALUE;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc2_w        #3                  // double 4.9E-324d
       3: dstore_1
       4: ldc2_w        #5                  // double 1.7976931348623157E308d
       7: dstore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Double("4.9E-324"));
methodVisitor.visitVarInsn(DSTORE, 1);
methodVisitor.visitLdcInsn(new Double("1.7976931348623157E308"));
methodVisitor.visitVarInsn(DSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc2_w          #3       // {this} | {double, top}
0003: dstore_1                 // {this, double, top} | {}
0004: ldc2_w          #5       // {this, double, top} | {double, top}
0007: dstore_3                 // {this, double, top, double, top} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

The numeric `value` of that run-time constant is pushed onto the operand stack as a `long` or `double`, respectively.

### 2.2.6 reference type

#### null: aconst_null

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object obj_null = null;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: aconst_null
       1: astore_1
       2: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ACONST_NULL);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: aconst_null              // {this} | {null}
0001: astore_1                 // {this, null} | {}
0002: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., null
```

Push the `null` object reference onto the operand stack.

#### String: ldc

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    String str = "GoodChild";
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #2                  // String GoodChild
       2: astore_1
       3: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn("GoodChild");
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc             #2       // {this} | {String}
0002: astore_1                 // {this, String} | {}
0003: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

Format

```shell
ldc
index
```

The `index` is an unsigned byte that must be a valid index into the run-time constant pool of the current class. The run-time constant pool entry at `index` either must be a run-time constant of type `int` or `float`, or **a reference to a string literal**, or **a symbolic reference to a class, method type, or method handle**.

#### Class`<?>`: ldc

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Class<?> clazz = HelloWorld.class;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> `.`分隔是Full Qualified Class Name，用于`.java`
>
> `/`分隔是 Internal Name，用于`.class`
>
> `L{用/分隔的类名};`是Descriptor，用于`.class`

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #2                  // class sample/HelloWorld
       2: astore_1
       3: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(Type.getType("Lsample/HelloWorld;"));
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc             #2       // {this} | {Class}
0002: astore_1                 // {this, Class} | {}
0003: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

Format

```shell
ldc
index
```

The `index` is an unsigned byte that must be a valid index into the run-time constant pool of the current class. The run-time constant pool entry at `index` either must be a run-time constant of type `int` or `float`, or **a reference to a string literal**, or **a symbolic reference to a class, method type, or method handle**.

### 2.2.7 ldc and ldc_w

> 简言之，一个字节大小的index（0～255）能够索引常量池，则使用`ldc`，需要两个字节大小的index（256～65535）索引常量池，则使用`ldc_w`。
>
> 但就功能方面，`ldc_w`等价于`ldc`

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  private String str001 = "str001";
  // ... ... 省略255个
  private String str257 = "str257";

  public void test() {
    String str = "str258";
  }

  public static void main(String[] args) {
    String format = "private String str%03d = \"str%03d\";";
    for (int i = 1; i < 258; i++) {
      String line = String.format(format, i, i);
      System.out.println(line);
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 前面0～255还是会使用`ldc`，而非`ldc_w` 

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc_w         #516                // String str258
       3: astore_1
       4: return
...
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn("str258");
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc_w           #516     // {this} | {String}
0003: astore_1                 // {this, String} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，Operand Stack的变化如下：

```shell
... →

..., value
```

Format

> 这里需要16bit才能标记常量池索引时，就需要indexbyte1和indexbyte2

```shell
ldc_w
indexbyte1
indexbyte2
```

The unsigned `indexbyte1` and `indexbyte2` are assembled into an unsigned 16-bit index into the run-time constant pool of the current class, where the value of the `index` is calculated as `(indexbyte1 << 8) | indexbyte2`. The index must be a valid index into the run-time constant pool of the current class. The run-time constant pool entry at the index either must be a run-time constant of type `int` or `float`, or **a reference to a string literal**, or **a symbolic reference to a class, method type, or method handle**.

## 2.3 opcode: transfer values (50/76/205)