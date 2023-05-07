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

### 2.3.1 概览

从Instruction的角度来说，与transfer values相关的opcode有50个。

其中，与load相关的opcode有25个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 21     | iload           | 28     | iload_2         | 35     | fload_1         | 42     | aload_0         |
| 22     | lload           | 29     | iload_3         | 36     | fload_2         | 43     | aload_1         |
| 23     | fload           | 30     | lload_0         | 37     | fload_3         | 44     | aload_2         |
| 24     | dload           | 31     | lload_1         | 38     | dload_0         | 45     | aload_3         |
| 25     | aload           | 32     | lload_2         | 39     | dload_1         | 46     |                 |
| 26     | iload_0         | 33     | lload_3         | 40     | dload_2         | 47     |                 |
| 27     | iload_1         | 34     | fload_0         | 41     | dload_3         | 48     |                 |

其中，与store相关的opcode有25个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 54     | istore          | 61     | istore_2        | 68     | fstore_1        | 75     | astore_0        |
| 55     | lstore          | 62     | istore_3        | 69     | fstore_2        | 76     | astore_1        |
| 56     | fstore          | 63     | lstore_0        | 70     | fstore_3        | 77     | astore_2        |
| 57     | dstore          | 64     | lstore_1        | 71     | dstore_0        | 78     | astore_3        |
| 58     | astore          | 65     | lstore_2        | 72     | dstore_1        | 79     |                 |
| 59     | istore_0        | 66     | lstore_3        | 73     | dstore_2        | 80     |                 |
| 60     | istore_1        | 67     | fstore_0        | 74     | dstore_3        | 81     |                 |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitVarInsn()`:
  - `iload`, `istore`, `iload_<n>`, `istore_<n>`.
  - `lload`, `lstore`, `lload_<n>`, `lstore_<n>`.
  - `fload`, `fstore`, `fload_<n>`, `fstore_<n>`.
  - `dload`, `dstore`, `dload_<n>`, `dstore_<n>`.
  - `aload`, `astore`, `aload_<n>`, `astore_<n>`.

**注意: Constant Pool、operand stack和local variables，对于`long`和`double`类型的数据，都占用2个位置。（Javap看常量池的索引，会发现如果索引数值对应long或double，下一个索引位置是当前+2，而非当前+1）**

**注意：load将数据从local variables加载到operand stack；store将数据从operand stack出栈后存储到local variables中**

### 2.3.2 primitive type

#### int

> 注意，boolean、byte、short、char，在local variables、operand stack中，都会被当作int处理

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = a;
    int c = b;
    int d = c;
    int e = d;
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
       0: iconst_1
       1: istore_1
       2: iload_1
       3: istore_2
       4: iload_2
       5: istore_3
       6: iload_3
       7: istore        4
       9: iload         4
      11: istore        5
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitVarInsn(ILOAD, 1); // 虽然 JVM的操作码有iload_1和i_store_1，但是ASM中还是拆成iload index的形式，调用visitVarInsn()方法，当然生成的.class代码是iload_1 而非 iload 1
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitVarInsn(ILOAD, 3);
methodVisitor.visitVarInsn(ISTORE, 4);
methodVisitor.visitVarInsn(ILOAD, 4);
methodVisitor.visitVarInsn(ISTORE, 5);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 6);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iload_1                  // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_2                  // {this, int, int} | {int}
0005: istore_3                 // {this, int, int, int} | {}
0006: iload_3                  // {this, int, int, int} | {int}
0007: istore          4        // {this, int, int, int, int} | {}
0009: iload           4        // {this, int, int, int, int} | {int}
0011: istore          5        // {this, int, int, int, int, int} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`iload`指令对应的Operand Stack的变化如下：

```shell
... →

..., value
```

Format:

> Instruction = opcode + operands，这里iload就是操作码opcode，index就是操作数operands

```shell
iload
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The local variable at `index` must contain an `int`.
- The `value` of the local variable at `index` is pushed onto the operand stack.

从JVM规范的角度来看，`istore`指令对应的Operand Stack的变化如下：

```
..., value →

...
```

Format:

```
istore
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The `value` on the top of the operand stack must be of type `int`.
- It is popped from the operand stack, and the value of the local variable at `index` is set to `value`.

#### float

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    float a = 1;
    float b = a;
    float c = b;
    float d = c;
    float e = d;
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
       0: fconst_1
       1: fstore_1
       2: fload_1
       3: fstore_2
       4: fload_2
       5: fstore_3
       6: fload_3
       7: fstore        4
       9: fload         4
      11: fstore        5
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitInsn(FCONST_1);
methodVisitor.visitVarInsn(FSTORE, 1);
methodVisitor.visitVarInsn(FLOAD, 1);
methodVisitor.visitVarInsn(FSTORE, 2);
methodVisitor.visitVarInsn(FLOAD, 2);
methodVisitor.visitVarInsn(FSTORE, 3);
methodVisitor.visitVarInsn(FLOAD, 3);
methodVisitor.visitVarInsn(FSTORE, 4);
methodVisitor.visitVarInsn(FLOAD, 4);
methodVisitor.visitVarInsn(FSTORE, 5);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 6);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: fconst_1                 // {this} | {float}
0001: fstore_1                 // {this, float} | {}
0002: fload_1                  // {this, float} | {float}
0003: fstore_2                 // {this, float, float} | {}
0004: fload_2                  // {this, float, float} | {float}
0005: fstore_3                 // {this, float, float, float} | {}
0006: fload_3                  // {this, float, float, float} | {float}
0007: fstore          4        // {this, float, float, float, float} | {}
0009: fload           4        // {this, float, float, float, float} | {float}
0011: fstore          5        // {this, float, float, float, float, float} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`fload`指令对应的Operand Stack的变化如下：

```shell
... →

..., value
```

Format:

```shell
fload
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The local variable at `index` must contain a `float`.
- The `value` of the local variable at `index` is pushed onto the operand stack.

从JVM规范的角度来看，`fstore`指令对应的Operand Stack的变化如下：

```shell
..., value →

...
```

Format:

```shell
fstore
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The `value` on the top of the operand stack must be of type `float`.
- It is popped from the operand stack and undergoes value set conversion, resulting in `value'`. The value of the local variable at `index` is set to `value'`.

#### long

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long a = 1;
    long b = a;
    long c = b;
    long d = c;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意long和double占用2个槽位，所以这里lstore和lload都间隔2个位置。

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: lconst_1
       1: lstore_1
       2: lload_1
       3: lstore_3
       4: lload_3
       5: lstore        5
       7: lload         5
       9: lstore        7
      11: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(LCONST_1);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitVarInsn(LLOAD, 3);
methodVisitor.visitVarInsn(LSTORE, 5);
methodVisitor.visitVarInsn(LLOAD, 5);
methodVisitor.visitVarInsn(LSTORE, 7);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 9);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: lconst_1                 // {this} | {long, top}
0001: lstore_1                 // {this, long, top} | {}
0002: lload_1                  // {this, long, top} | {long, top}
0003: lstore_3                 // {this, long, top, long, top} | {}
0004: lload_3                  // {this, long, top, long, top} | {long, top}
0005: lstore          5        // {this, long, top, long, top, long, top} | {}
0007: lload           5        // {this, long, top, long, top, long, top} | {long, top}
0009: lstore          7        // {this, long, top, long, top, long, top, long, top} | {}
0011: return                   // {} | {}
```

从JVM规范的角度来看，`lload`指令对应的Operand Stack的变化如下：

```shell
... →

..., value
```

Format:

```shell
lload
index
```

- The `index` is an unsigned byte. Both `index` and `index+1` must be indices into the local variable array of the current frame.
- The local variable at `index` must contain a `long`.
- The `value` of the local variable at `index` is pushed onto the operand stack.

从JVM规范的角度来看，`lstore`指令对应的Operand Stack的变化如下：

```shell
..., value →

...
```

Format:

```shell
lstore
index
```

- The `index` is an unsigned byte. Both `index` and `index+1` must be indices into the local variable array of the current frame.
- The `value` on the top of the operand stack must be of type `long`.
- It is popped from the operand stack, and the local variables at `index` and `index+1` are set to `value`.

#### double

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    double a = 1;
    double b = a;
    double c = b;
    double d = c;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
...
  public void test();
    Code:
       0: dconst_1
       1: dstore_1
       2: dload_1
       3: dstore_3
       4: dload_3
       5: dstore        5
       7: dload         5
       9: dstore        7
      11: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(DCONST_1);
methodVisitor.visitVarInsn(DSTORE, 1);
methodVisitor.visitVarInsn(DLOAD, 1);
methodVisitor.visitVarInsn(DSTORE, 3);
methodVisitor.visitVarInsn(DLOAD, 3);
methodVisitor.visitVarInsn(DSTORE, 5);
methodVisitor.visitVarInsn(DLOAD, 5);
methodVisitor.visitVarInsn(DSTORE, 7);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 9);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: dconst_1                 // {this} | {double, top}
0001: dstore_1                 // {this, double, top} | {}
0002: dload_1                  // {this, double, top} | {double, top}
0003: dstore_3                 // {this, double, top, double, top} | {}
0004: dload_3                  // {this, double, top, double, top} | {double, top}
0005: dstore          5        // {this, double, top, double, top, double, top} | {}
0007: dload           5        // {this, double, top, double, top, double, top} | {double, top}
0009: dstore          7        // {this, double, top, double, top, double, top, double, top} | {}
0011: return                   // {} | {}
```

从JVM规范的角度来看，`dload`指令对应的Operand Stack的变化如下：

```shell
... →

..., value
```

Format:

```shell
dload
index
```

- The `index` is an unsigned byte. Both `index` and `index+1` must be indices into the local variable array of the current frame.
- The local variable at `index` must contain a `double`.
- The `value` of the local variable at `index` is pushed onto the operand stack.

从JVM规范的角度来看，`dstore`指令对应的Operand Stack的变化如下：

```shell
..., value →

...
```

Format:

```shell
dstore
index
```

- The `index` is an unsigned byte. Both `index` and `index+1` must be indices into the local variable array of the current frame.
- The `value` on the top of the operand stack must be of type `double`.
- It is popped from the operand stack and undergoes value set conversion, resulting in `value'`. The local variables at `index` and `index+1` are set to `value'`.

### 2.3.3 reference type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object a = null;
    Object b = a;
    Object c = b;
    Object d = c;
    Object e = d;
  }
}
```

在上面的代码中，我们也可以将`null`替换成`String`或`Object`类型的对象。

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
       2: aload_1
       3: astore_2
       4: aload_2
       5: astore_3
       6: aload_3
       7: astore        4
       9: aload         4
      11: astore        5
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```shell
methodVisitor.visitCode();
methodVisitor.visitInsn(ACONST_NULL);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitVarInsn(ASTORE, 2);
methodVisitor.visitVarInsn(ALOAD, 2);
methodVisitor.visitVarInsn(ASTORE, 3);
methodVisitor.visitVarInsn(ALOAD, 3);
methodVisitor.visitVarInsn(ASTORE, 4);
methodVisitor.visitVarInsn(ALOAD, 4);
methodVisitor.visitVarInsn(ASTORE, 5);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 6);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: aconst_null              // {this} | {null}
0001: astore_1                 // {this, null} | {}
0002: aload_1                  // {this, null} | {null}
0003: astore_2                 // {this, null, null} | {}
0004: aload_2                  // {this, null, null} | {null}
0005: astore_3                 // {this, null, null, null} | {}
0006: aload_3                  // {this, null, null, null} | {null}
0007: astore          4        // {this, null, null, null, null} | {}
0009: aload           4        // {this, null, null, null, null} | {null}
0011: astore          5        // {this, null, null, null, null, null} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`aload`指令对应的Operand Stack的变化如下：

```shell
... →

..., objectref
```

Format:

```shell
aload
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The local variable at `index` must contain a reference.
- The `objectref` in the local variable at `index` is pushed onto the operand stack.

从JVM规范的角度来看，`astore`指令对应的Operand Stack的变化如下：

```shell
..., objectref →

...
```

Format:

```shell
astore
index
```

- The `index` is an unsigned byte that must be an index into the local variable array of the current frame.
- The `objectref` on the top of the operand stack must be of type `returnAddress` or of type `reference`.
- It is popped from the operand stack, and the `value` of the local variable at `index` is set to `objectref`.

## 2.4 opcode: math (52/128/205)

### 2.4.1 概览

从Instruction的角度来说，与math相关的opcode有52个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 96     | iadd            | 109    | ldiv            | 122    | ishr            | 135    | i2d             |
| 97     | ladd            | 110    | fdiv            | 123    | lshr            | 136    | l2i             |
| 98     | fadd            | 111    | ddiv            | 124    | iushr           | 137    | l2f             |
| 99     | dadd            | 112    | irem            | 125    | lushr           | 138    | l2d             |
| 100    | isub            | 113    | lrem            | 126    | iand            | 139    | f2i             |
| 101    | lsub            | 114    | frem            | 127    | land            | 140    | f2l             |
| 102    | fsub            | 115    | drem            | 128    | ior             | 141    | f2d             |
| 103    | dsub            | 116    | ineg            | 129    | lor             | 142    | d2i             |
| 104    | imul            | 117    | lneg            | 130    | ixor            | 143    | d2l             |
| 105    | lmul            | 118    | fneg            | 131    | lxor            | 144    | d2f             |
| 106    | fmul            | 119    | dneg            | 132    | iinc            | 145    | i2b             |
| 107    | dmul            | 120    | ishl            | 133    | i2l             | 146    | i2c             |
| 108    | idiv            | 121    | lshl            | 134    | i2f             | 147    | i2s             |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`:
  - `iadd`, `isub`, `imul`, `idiv`, `irem`, `ineg`
  - `ladd`, `lsub`, `lmul`, `ldiv`, `lrem`, `lneg`
  - `fadd`, `fsub`, `fmul`, `fdiv`, `frem`, `fneg`
  - `dadd`, `dsub`, `dmul`, `ddiv`, `drem`, `dneg`
  - `ishl`, `ishr`, `iushr`, `iand`, `ior`, `ixor` （int类型的位操作）
  - `lshl`, `lshr`, `lushr`, `land`, `lor`, `lxor` （long类型的位操作）
  - `i2l`, `i2f`, `i2d`, `i2b`, `i2c`, `i2s`
  - `l2i`, `l2f`, `l2d`
  - `f2i`, `f2l`, `f2d`
  - `d2i`, `d2l`, `d2f`
- `MethodVisitor.visitIincInsn()`: `iinc`

### 2.4.2 Arithmetic

#### int: add/sub/mul/div/rem

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a + b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1 // a 加载到 operand stack
       5: iload_2 // b 加载到 operand stack
       6: iadd // b和a出栈，相加后的结果入栈 operand stack
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(IADD);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: iadd                     // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`iadd`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

#### long: add/sub/mul/div/rem

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long a = 1;
    long b = 2;
    long c = a + b;
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
       0: lconst_1
       1: lstore_1
       2: ldc2_w        #2                  // long 2l
       5: lstore_3
       6: lload_1 // long占用2位置，a加载到operand stack
       7: lload_3 // long占用2位置，b加载到operand stack
       8: ladd // b和a出栈，相加后的结果入栈
       9: lstore        5
      11: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(LCONST_1);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitLdcInsn(new Long(2L));
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitVarInsn(LLOAD, 3);
methodVisitor.visitInsn(LADD);
methodVisitor.visitVarInsn(LSTORE, 5);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(4, 7);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: lconst_1                 // {this} | {long, top}
0001: lstore_1                 // {this, long, top} | {}
0002: ldc2_w          #2       // {this, long, top} | {long, top}
0005: lstore_3                 // {this, long, top, long, top} | {}
0006: lload_1                  // {this, long, top, long, top} | {long, top}
0007: lload_3                  // {this, long, top, long, top} | {long, top, long, top}
0008: ladd                     // {this, long, top, long, top} | {long, top}
0009: lstore          5        // {this, long, top, long, top, long, top} | {}
0011: return                   // {} | {}
```

从JVM规范的角度来看，`ladd`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `long`. The values are popped from the operand stack. The long `result` is `value1 + value2`. The `result` is pushed onto the operand stack.

#### int: ineg

> [neg（汇编指令）_百度百科 (baidu.com)](https://baike.baidu.com/item/neg/2832712?fr=aladdin) neg即取反

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = -a;
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
       0: iconst_1
       1: istore_1
       2: iload_1
       3: ineg
       4: istore_2
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitInsn(INEG);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iload_1                  // {this, int} | {int}
0003: ineg                     // {this, int} | {int}
0004: istore_2                 // {this, int, int} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`ineg`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., result
```

The `value` must be of type `int`. It is popped from the operand stack. The int `result` is the arithmetic negation of `value`, `-value`. The `result` is pushed onto the operand stack.

#### long: lneg

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long a = 1;
    long b = -a;
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
       0: lconst_1
       1: lstore_1
       2: lload_1
       3: lneg
       4: lstore_3
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(LCONST_1);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitInsn(LNEG);
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: lconst_1                 // {this} | {long, top}
0001: lstore_1                 // {this, long, top} | {}
0002: lload_1                  // {this, long, top} | {long, top}
0003: lneg                     // {this, long, top} | {long, top}
0004: lstore_3                 // {this, long, top, long, top} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`lneg`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., result
```

The `value` must be of type `long`. It is popped from the operand stack. The long `result` is the arithmetic negation of `value`, `-value`. The `result` is pushed onto the operand stack.

### 2.4.3 iinc

> 操作数必须是能转int处理的boolean、byte、short、char，或int本身

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int i = 0;
    i++;
    i += 10;
    i -= 5;
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
       0: iconst_0
       1: istore_1
       2: iinc          1, 1
       5: iinc          1, 10
       8: iinc          1, -5 // 注意就算是减法，也是用iinc
      11: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitIincInsn(1, 1);
methodVisitor.visitIincInsn(1, 10);
methodVisitor.visitIincInsn(1, -5);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_0                 // {this} | {int}
0001: istore_1                 // {this, int} | {} // iinc只需要在local variables上进行，不需要先把操作数加载到operand stack
0002: iinc       1    1        // {this, int} | {}
0005: iinc       1    10       // {this, int} | {}
0008: iinc       1    -5       // {this, int} | {}
0011: return                   // {} | {}
```

从JVM规范的角度来看，`iinc`指令对应的Operand Stack的变化如下：

```pseudocode
No change
```

### 2.4.4 Bit Shift

#### shift left

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a << b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: ishl // 将b和a出栈后，a左移b位，然后重新入栈
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(ISHL); // 
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: ishl                     // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`ishl`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

> 这里 s 只取low低5bits是因为int就32bit，2^5=32

Both `value1` and `value2` must be of type `int`. The values are popped from the operand stack. An int `result` is calculated by shifting `value1` left by `s` bit positions, where `s` is the value of the low 5 bits of `value2`. The `result` is pushed onto the operand stack.

#### arithmetic shift right

> **>>** 表示右移，如果该数为正，则高位补0，若为负数，则高位补1**>>** 表示右移，如果该数为正，则高位补0，若为负数，则高位补1

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a >> b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: ishr
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```shell
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(ISHR);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: ishr                     // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`ishr`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. The values are popped from the operand stack. An int `result` is calculated by shifting `value1` right by `s` bit positions, with **sign extension**, where `s` is the value of the low 5 bits of `value2`. The `result` is pushed onto the operand stack.

#### logical shift right

> **>>>** 表示**无符号右移**，也叫逻辑右移，即若该数为正，则高位补0，而若该数为负数，则右移后高位同样补0。

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a >>> b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: iushr
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(IUSHR);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: iushr                    // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`iushr`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. The values are popped from the operand stack. An int `result` is calculated by shifting `value1` right by `s` bit positions, with **zero extension**, where `s` is the value of the low 5 bits of `value2`. The `result` is pushed onto the operand stack.

### 2.4.5 Bit Logic

#### and

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a & b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: iand
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(IAND);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: iand                     // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`iand`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. They are popped from the operand stack. An int `result` is calculated by taking the bitwise AND (conjunction) of `value1` and `value2`. The `result` is pushed onto the operand stack.

#### or

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a | b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: ior
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(IOR);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: ior                      // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`ior`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. They are popped from the operand stack. An int `result` is calculated by taking the bitwise inclusive OR of `value1` and `value2`. The `result` is pushed onto the operand stack.

#### xor

> [异或_百度百科 (baidu.com)](https://baike.baidu.com/item/异或/10993677?fr=aladdin) 异或运算

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 1;
    int b = 2;
    int c = a ^ b;
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
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: ixor
       7: istore_3
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(IXOR);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_2                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: iload_1                  // {this, int, int} | {int}
0005: iload_2                  // {this, int, int} | {int, int}
0006: ixor                     // {this, int, int} | {int}
0007: istore_3                 // {this, int, int, int} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`ixor`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. They are popped from the operand stack. An int `result` is calculated by taking the bitwise exclusive OR of `value1` and `value2`. The `result` is pushed onto the operand stack.

#### not

> [NOT运算_百度百科 (baidu.com)](https://baike.baidu.com/item/NOT运算/22801356) 
>
> 在逻辑中，NOT运算是一种操作，它将命题P带到另一个命题“非P”，写为¬P，当P为假时直观地解释为真，而当P为真时则为假。
>
> **not在java中并没有直接对应的指令，java通过 `iconst_m1` , `ixor` 组成完成not操作**

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a = 0;
    int b = ~a;
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
       0: iconst_0
       1: istore_1
       2: iload_1
       3: iconst_m1 // -1 加载到 operand stack
       4: ixor // 将  -1 和 a变量出栈，进行 异或操作，结果入栈
       5: istore_2
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```shell
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitInsn(ICONST_M1);
methodVisitor.visitInsn(IXOR);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_0                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iload_1                  // {this, int} | {int}
0003: iconst_m1                // {this, int} | {int, int}
0004: ixor                     // {this, int} | {int}
0005: istore_2                 // {this, int, int} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，`ixor`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `int`. They are popped from the operand stack. An int `result` is calculated by taking the bitwise exclusive OR of `value1` and `value2`. The `result` is pushed onto the operand stack.

### 2.4.6 Type Conversion

#### int to long

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int intValue = 0;
    long longValue = intValue; // 占位少的转占位多的，不需要显式 (long)
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
       0: iconst_0
       1: istore_1
       2: iload_1
       3: i2l // 实际指令中会有 int to long的过程
       4: lstore_2
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitInsn(I2L);
methodVisitor.visitVarInsn(LSTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_0                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iload_1                  // {this, int} | {int} // 原本是int
0003: i2l                      // {this, int} | {long, top} // int出栈后转long再入栈
0004: lstore_2                 // {this, int, long, top} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`i2l`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., result
```

The value on the top of the operand stack must be of type `int`. It is popped from the operand stack and sign-extended to a `long` result. That `result` is pushed onto the operand stack.

#### long to int

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long longValue = 0;
    int intValue = (int) longValue;
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
       2: lload_1
       3: l2i
       4: istore_3
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(LCONST_0);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitInsn(L2I);
methodVisitor.visitVarInsn(ISTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: lconst_0                 // {this} | {long, top}
0001: lstore_1                 // {this, long, top} | {}
0002: lload_1                  // {this, long, top} | {long, top}
0003: l2i                      // {this, long, top} | {int}
0004: istore_3                 // {this, long, top, int} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`l2i`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., result
```

The `value` on the top of the operand stack must be of type `long`. It is popped from the operand stack and converted to an int `result` by taking the low-order 32 bits of the long `value` and discarding the high-order 32 bits. The `result` is pushed onto the operand stack.

## 2.5 opcode: object (3/131/205)

### 2.5.1 概览

从Instruction的角度来说，与type相关的opcode有3个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 187    | new             | 192    | checkcast       | 193    | instanceof      |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitTypeInsn()`: `new`, `checkcast`, `instanceof`

### 2.5.2 Create Object

#### create instance with no args

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object obj = new Object();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 一般看到 `new` => `dup` => ... => `invokespecial` 的指令组合，那大概率就是在创建对象（new）

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "java/lang/Object");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_Object}
0003: dup                      // {this} | {uninitialized_Object, uninitialized_Object}
0004: invokespecial   #1       // {this} | {Object} // 调用构造方法后，原本operand stack栈顶两个对象指针指向的Heap堆内存空间被初始化
0007: astore_1                 // {this, Object} | {}
0008: return                   // {} | {}
```

从JVM规范的角度来看，`new`指令对应的Operand Stack的变化如下：

```pseudocode
... →

..., objectref
```

Memory for a new instance of that class is allocated from the garbage-collected heap, and the instance variables of the new object are initialized to their default initial values ([§2.3](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.3), [§2.4](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.4)). The *objectref*, a `reference` to the instance, is pushed onto the operand stack.

> 这里说的在Heap中分配对象所需的内存，初始化对象的默认值，这里默认值不是说构造方法传入的参数，而是指成员变量如果是primitive type则会赋予0等值，而对象类型则是null。

从JVM规范的角度来看，`dup`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., value, value
```

Duplicate the top `value` on the operand stack and push the duplicated value onto the operand stack.

The `dup` instruction must not be used unless `value` is a value of a **category 1 computational type**.

> 这里dup只能复制operand stack栈顶1个位置的数据，如果是long、double则需要用dup2

从JVM规范的角度来看，`invokespecial`指令对应的Operand Stack的变化如下：

```
..., objectref, [arg1, [arg2 ...]] →

...
```

The `objectref` must be of type reference and must be followed on the operand stack by `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the selected instance method.

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dup2) 具体描述规则很长，可以自行查阅
>
> invokespecial: Invoke instance method; direct invocation of instance initialization methods and methods of the current class and its supertypes

#### create instance with args

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  private String name;
  private int age;

  public HelloWorld(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public void test() {
    HelloWorld instance = new HelloWorld("tomcat", 10);
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
       0: new           #4                  // class sample/HelloWorld
       3: dup
       4: ldc           #5                  // String tomcat
       6: bipush        10
       8: invokespecial #6                  // Method "<init>":(Ljava/lang/String;I)V
      11: astore_1
      12: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "sample/HelloWorld");
methodVisitor.visitInsn(DUP);
methodVisitor.visitLdcInsn("tomcat");
methodVisitor.visitIntInsn(BIPUSH, 10);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "sample/HelloWorld", "<init>", "(Ljava/lang/String;I)V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(4, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: new             #4       // {this} | {uninitialized_HelloWorld}
0003: dup                      // {this} | {uninitialized_HelloWorld, uninitialized_HelloWorld}
0004: ldc             #5       // {this} | {uninitialized_HelloWorld, uninitialized_HelloWorld, String}
0006: bipush          10       // {this} | {uninitialized_HelloWorld, uninitialized_HelloWorld, String, int}
0008: invokespecial   #6       // {this} | {HelloWorld}
0011: astore_1                 // {this, HelloWorld} | {}
0012: return                   // {} | {}
```

从JVM规范的角度来看，`invokespecial`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref, [arg1, [arg2 ...]] →

...
```

The `objectref` must be of type reference and must be followed on the operand stack by `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the selected instance method.

### 2.5.3 Type Check

#### checkcast

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object obj = new Object();
    String str = (String) obj;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 这里checkcast会检查是否能进行转换，如果类型不符合会抛出`ClassCastException`异常
>
> 这里checkcast转换类型后，operand stack指向Heap内存中的指针并没有发生改变

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1
       8: aload_1
       9: checkcast     #3                  // class java/lang/String
      12: astore_2
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```shell
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "java/lang/Object");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitTypeInsn(CHECKCAST, "java/lang/String");
methodVisitor.visitVarInsn(ASTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_Object}
0003: dup                      // {this} | {uninitialized_Object, uninitialized_Object}
0004: invokespecial   #1       // {this} | {Object}
0007: astore_1                 // {this, Object} | {}
0008: aload_1                  // {this, Object} | {Object} // 调用checkcast之前
0009: checkcast       #3       // {this, Object} | {String} // 调用checkcast后，类型转变，但是operand stack内指针指向Heap内存中的位置并没有改变
0012: astore_2                 // {this, Object, String} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`checkcast`指令对应的Operand Stack的变化如下：

```pse\
..., objectref →

..., objectref
```

The `objectref` must be of type `reference`.

- **If `objectref` is `null`, then the operand stack is unchanged.**
- **If `objectref` can be cast to the resolved class, array, or interface type, the operand stack is unchanged; otherwise, the `checkcast` instruction throws a `ClassCastException`.**

**The `checkcast` instruction is very similar to the `instanceof` instruction. It differs in its treatment of `null`, its behavior when its test fails (`checkcast` throws an exception, `instanceof` pushes a `result` code), and its effect on the operand stack.**

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dup2) 更详细的描述和规则见官方文档
>
> 值得注意的就是数组类型，这里也有转换的规则，和泛型不一样
>
> The following rules are used to determine whether an *objectref* that is not `null` can be cast to the resolved type. If S is the type of the object referred to by *objectref*, and T is the resolved class, array, or interface type, then *checkcast* determines whether *objectref* can be cast to type T as follows:
>
> - If S is a class type, then:
>   - If T is a class type, then S must be the same class as T, or S must be a subclass of T;
>   - If T is an interface type, then S must implement interface T.
> - If S is an array type SC`[]`, that is, an array of components of type SC, then:
>   - If T is a class type, then T must be `Object`.
>   - If T is an interface type, then T must be one of the interfaces implemented by arrays (JLS §4.10.3).
>   - If T is an array type TC`[]`, that is, an array of components of type TC, then one of the following must be true:
>     - TC and SC are the same primitive type.
>     - **TC and SC are reference types, and type SC can be cast to TC by recursive application of these rules.**

#### instanceof

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
    public void test() {
        Object obj = new Object();
        boolean flag = obj instanceof String;
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
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1
       8: aload_1
       9: instanceof    #3                  // class java/lang/String
      12: istore_2
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "java/lang/Object");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitTypeInsn(INSTANCEOF, "java/lang/String");
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_Object}
0003: dup                      // {this} | {uninitialized_Object, uninitialized_Object}
0004: invokespecial   #1       // {this} | {Object}
0007: astore_1                 // {this, Object} | {}
0008: aload_1                  // {this, Object} | {Object}
0009: instanceof      #3       // {this, Object} | {int} // 注意这里 instanceof 如果为true则放入1，为false则放入0
0012: istore_2                 // {this, Object, int} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`instanceof`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref →

..., result
```

The `objectref`, which must be of type `reference`, is popped from the operand stack.

- **If `objectref` is `null`, the `instanceof` instruction pushes an int `result` of `0` as an `int` on the operand stack.**
- **If `objectref` is an instance of the resolved class or array or implements the resolved interface, the `instanceof` instruction pushes an int `result` of `1` as an `int` on the operand stack; otherwise, it pushes an int `result` of `0`.**

**The `instanceof` instruction is very similar to the `checkcast` instruction. It differs in its treatment of `null`, its behavior when its test fails (`checkcast` throws an exception, `instanceof` pushes a `result` code), and its effect on the operand stack.**

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.instanceof)
>
> 下面规则和checkcast的描述是一样的，主要是描述java中怎么判断S是否允许被转成T
>
> The following rules are used to determine whether an *objectref* that is not `null` is an instance of the resolved type. If S is the type of the object referred to by *objectref*, and T is the resolved class, array, or interface type, then *instanceof* determines whether *objectref* is an instance of T as follows:
>
> - If S is a class type, then:
>   - If T is a class type, then S must be the same class as T, or S must be a subclass of T;
>   - If T is an interface type, then S must implement interface T.
> - If S is an array type SC`[]`, that is, an array of components of type SC, then:
>   - If T is a class type, then T must be `Object`.
>   - If T is an interface type, then T must be one of the interfaces implemented by arrays (JLS §4.10.3).
>   - If T is an array type TC`[]`, that is, an array of components of type TC, then one of the following must be true:
>     - TC and SC are the same primitive type.
>     - TC and SC are reference types, and type SC can be cast to TC by these run-time rules.

## 2.6 opcode: field (4/135/205)

### 2.6.1 概览

从Instruction的角度来说，与field相关的opcode有4个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 178    | getstatic       | 179    | putstatic       | 180    | getfield        | 181    | putfield        |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitFieldInsn()`: `getstatic`, `putstatic`, `getfield`, `putfield`.

注意点：

+ getstatic和putstatic对应类的静态成员变量，所以不需要加载实例指针到operand stack，getfield和putfield对应类的实例成员变量，则需要先加载实例指针到operand stack

### 2.6.2 non-static field

#### getfield

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public int value;

  public void test() {
    int i = this.value;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  public int value;
...
  public void test();
    Code:
       0: aload_0 // 需要先把对象实例指针加载到operand stack
       1: getfield      #2                  // Field value:I
       4: istore_1
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitFieldInsn(GETFIELD, "sample/HelloWorld", "value", "I");
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: aload_0                  // {this} | {this} // 需要先加载对象指针到operand stack
0001: getfield        #2       // {this} | {int} // 指定对象的成员变量加载到operand stack中
0004: istore_1                 // {this, int} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`getfield`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref →

..., value
```

The `objectref`, which must be of type `reference`, is popped from the operand stack. The value of the referenced field in `objectref` is fetched and pushed onto the operand stack.

#### putfield

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public int value;

  public void test() {
    this.value = 0;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  public int value;
...

  public void test();
    Code:
       0: aload_0 // 先加载对象指针到 operand stack
       1: iconst_0 // 0用于赋值
       2: putfield      #2                  // Field value:I
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitFieldInsn(PUTFIELD, "sample/HelloWorld", "value", "I");
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: aload_0                  // {this} | {this}
0001: iconst_0                 // {this} | {this, int}
0002: putfield        #2       // {this} | {} // putfield 消耗栈顶的数据和对象指针，完成对指定成员变量的赋值
0005: return                   // {} | {}
```

从JVM规范的角度来看，`putfield`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref, value →

...
```

The `value` and `objectref` are popped from the operand stack. The `objectref` must be of type `reference`. The `value` undergoes value set conversion, resulting in `value'`, and the referenced field in `objectref` is set to `value'`.

- If the field descriptor type is `boolean`, `byte`, `char`, `short`, or `int`, then the `value` must be an `int`.
- If the field descriptor type is `float`, `long`, or `double`, then the `value` must be a `float`, `long`, or `double`, respectively.
- If the field descriptor type is a `reference` type, then the `value` must be of a type that is assignment compatible with the field descriptor type.
- <u>If the field is `final`, it must be declared in the current class, and the instruction must occur in an **instance initialization method** (`<init>`) of the current class.</u>

需要注意的就是最后一行，final成员变量要求在构造方法里就完成赋值

### 2.6.3 static field

#### getstatic

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public static int staticValue;

  public void test() {
    int i = HelloWorld.staticValue;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意：因为是类的静态成员变量，所以不需要加载实例指针到operand stack，直接获取

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  public static int staticValue;
...

  public void test();
    Code:
       0: getstatic     #2                  // Field staticValue:I
       3: istore_1
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitFieldInsn(GETSTATIC, "sample/HelloWorld", "staticValue", "I");
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: getstatic       #2       // {this} | {int} // 静态成员变量，可直接到方法区(Method Area)的类信息中取静态成员变量，而不需要先加载实例指针到operand stack
0003: istore_1                 // {this, int} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，`getstatic`指令对应的Operand Stack的变化如下：

```pseudocode
..., →

..., value
```

The `value` of the class or interface field is fetched and pushed onto the operand stack.

#### putstatic

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public static int staticValue;

  public void test() {
    HelloWorld.staticValue = 1;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  public static int staticValue;
...

  public void test();
    Code:
       0: iconst_1
       1: putstatic     #2                  // Field staticValue:I
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitFieldInsn(PUTSTATIC, "sample/HelloWorld", "staticValue", "I");
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: putstatic       #2       // {this} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，`putstatic`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

...
```

The `value` is popped from the operand stack and undergoes value set conversion, resulting in `value'`. The class field is set to `value'`.

The type of a `value` stored by a `putstatic` instruction must be compatible with the descriptor of the referenced field.

- If the field descriptor type is `boolean`, `byte`, `char`, `short`, or `int`, then the `value` must be an `int`.
- If the field descriptor type is `float`, `long`, or `double`, then the `value` must be a `float`, `long`, or `double`, respectively.
- If the field descriptor type is a `reference` type, then the `value` must be of a type that is assignment compatible with the field descriptor type.
- <u>If the field is `final`, it must be declared in the current class, and the instruction must occur in the `<clinit>` method of the current class.</u>

需要注意的同样是最后一句话，如果是final修饰的静态成员变量，那么需要在`<clint>`类的静态初始化块中就完成赋值。（当然直接在声明成员变量的时候就赋值也是ok的，可以用javap对比一下）

## 2.7 opcode: method (5/140/205)

### 2.7.1 概览

从Instruction的角度来说，与method相关的opcode有5个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 182    | invokevirtual   | 184    | invokestatic    | 186    | invokedynamic   |
| 183    | invokespecial   | 185    | invokeinterface | 187    |                 |

![Java ASM系列：（056）opcode: method_Java](https://s2.51cto.com/images/20210821/1629535019505703.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitMethodInsn()`: `invokevirtual`, `invokespecial`, `invokestatic`, `invokeinterface`
- `MethodVisitor.visitInvokeDynamicInsn()`: `invokedynamic`

另外，我们要注意：

- 方法调用，是先把方法所需要的参数加载到operand stack上，最后再进行方法的调用。
- static方法，在local variables索引为`0`的位置，存储的可能是方法的第一个参数或方法体内定义的局部变量。

### 2.7.2 invokevirtual

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void publicMethod(String name, int age) {
    // do nothing
  }

  protected void protectedMethod() {
    // do nothing
  }

  void packageMethod() {
    // do nothing
  }

  public void test() {
    publicMethod("tomcat", 10);
    protectedMethod();
    packageMethod();
    String str = toString();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意，这里没有显式Override父类Object的toString方法，所以还是调用Object类中的toString方法，但是加载到operand stack的实例指针是当前对象的指针（毕竟当前类是Object的子类）

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: aload_0
       1: ldc           #2                  // String tomcat
       3: bipush        10
       5: invokevirtual #3                  // Method publicMethod:(Ljava/lang/String;I)V
       8: aload_0
       9: invokevirtual #4                  // Method protectedMethod:()V
      12: aload_0
      13: invokevirtual #5                  // Method packageMethod:()V
      16: aload_0
      17: invokevirtual #6                  // Method java/lang/Object.toString:()Ljava/lang/String;
      20: astore_1
      21: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitLdcInsn("tomcat");
methodVisitor.visitIntInsn(BIPUSH, 10);
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "sample/HelloWorld", "publicMethod", "(Ljava/lang/String;I)V", false);
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "sample/HelloWorld", "protectedMethod", "()V", false);
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "sample/HelloWorld", "packageMethod", "()V", false);
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Object", "toString", "()Ljava/lang/String;", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: aload_0                  // {this} | {this}
0001: ldc             #2       // {this} | {this, String}
0003: bipush          10       // {this} | {this, String, int}
0005: invokevirtual   #3       // {this} | {}
0008: aload_0                  // {this} | {this}
0009: invokevirtual   #4       // {this} | {}
0012: aload_0                  // {this} | {this}
0013: invokevirtual   #5       // {this} | {}
0016: aload_0                  // {this} | {this}
0017: invokevirtual   #6       // {this} | {String}
0020: astore_1                 // {this, String} | {}
0021: return                   // {} | {}
```

从JVM规范的角度来看，`invokevirtual`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref, [arg1, [arg2 ...]] →

...
```

The `objectref` must be followed on the operand stack by `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the selected instance method.

> 因为调用的是实例方法，所以需要先加载对象指针到operand stack，后续才是加载方法所需要的参数

### 2.7.3 invokespecial

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dup2)
>
> invokespecial 在官方文档中有详细的规则和描述，可以自行查阅

从JVM规范的角度来看，`invokespecial`指令对应的Operand Stack的变化如下：

> incokespecial同样调用的也都是实例方法，所以也需要先加载对象指针到operand stack

```pseudocode
..., objectref, [arg1, [arg2 ...]] →

...
```

The `objectref` must be of type reference and must be followed on the operand stack by `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the selected instance method.

#### invoke constructor

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    HelloWorld instance = new HelloWorld();
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
       0: new           #2                  // class sample/HelloWorld
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "sample/HelloWorld");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "sample/HelloWorld", "<init>", "()V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_HelloWorld}
0003: dup                      // {this} | {uninitialized_HelloWorld, uninitialized_HelloWorld}
0004: invokespecial   #3       // {this} | {HelloWorld}
0007: astore_1                 // {this, HelloWorld} | {}
0008: return                   // {} | {}
```

#### invoke private method

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  private void privateMethod() {
    // do nothing
  }

  public void test() {
    privateMethod();
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
       0: aload_0
       1: invokespecial #2                  // Method privateMethod:()V
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "sample/HelloWorld", "privateMethod", "()V", false);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: aload_0                  // {this} | {this}
0001: invokespecial   #2       // {this} | {}
0004: return                   // {} | {}
```

#### invoke super method

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    String str = super.toString();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意：同样是调用toString方法，但是这里声明了是通过super来调用，则会变成invokespecial（前面invokevirtual的例子中没有super关键字）

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: aload_0
       1: invokespecial #2                  // Method java/lang/Object.toString:()Ljava/lang/String;
       4: astore_1
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "toString", "()Ljava/lang/String;", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: aload_0                  // {this} | {this}
0001: invokespecial   #2       // {this} | {String}
0004: astore_1                 // {this, String} | {}
0005: return                   // {} | {}
```

### 2.7.4 invokestatic

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public static void staticPublicMethod(String name, int age) {
    // do nothing
  }

  protected static void staticProtectedMethod() {
    // do nothing
  }

  static void staticPackageMethod() {
    // do nothing
  }

  private static void staticPrivateMethod() {
    // do nothing
  }

  public void test() {
    staticPublicMethod("tomcat", 10);
    staticProtectedMethod();
    staticPackageMethod();
    staticPrivateMethod();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$  javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #2                  // String tomcat
       2: bipush        10
       4: invokestatic  #3                  // Method staticPublicMethod:(Ljava/lang/String;I)V
       7: invokestatic  #4                  // Method staticProtectedMethod:()V
      10: invokestatic  #5                  // Method staticPackageMethod:()V
      13: invokestatic  #6                  // Method staticPrivateMethod:()V
      16: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn("tomcat");
methodVisitor.visitIntInsn(BIPUSH, 10);
methodVisitor.visitMethodInsn(INVOKESTATIC, "sample/HelloWorld", "staticPublicMethod", "(Ljava/lang/String;I)V", false);
methodVisitor.visitMethodInsn(INVOKESTATIC, "sample/HelloWorld", "staticProtectedMethod", "()V", false);
methodVisitor.visitMethodInsn(INVOKESTATIC, "sample/HelloWorld", "staticPackageMethod", "()V", false);
methodVisitor.visitMethodInsn(INVOKESTATIC, "sample/HelloWorld", "staticPrivateMethod", "()V", false);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: ldc             #2       // {this} | {String}
0002: bipush          10       // {this} | {String, int}
0004: invokestatic    #3       // {this} | {}
0007: invokestatic    #4       // {this} | {}
0010: invokestatic    #5       // {this} | {}
0013: invokestatic    #6       // {this} | {}
0016: return                   // {} | {}
```

从JVM规范的角度来看，`invokestatic`指令对应的Operand Stack的变化如下：

> 因为是调用static方法，所以不需要加载对象指针到operand stack

```pseudocode
..., [arg1, [arg2 ...]] →

...
```

The operand stack must contain `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the resolved method.

### 2.7.5 invokeinterface

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    MyInterface instance = new MyInterface() {
      @Override
      public void targetMethod() {
        // do nothing
      }
    };
    instance.defaultMethod();
    instance.targetMethod();
    MyInterface.staticMethod();
  }
}

interface MyInterface {
  static void staticMethod() {
    // do nothing
  }

  default void defaultMethod() {
    // do nothing
  }

  void targetMethod();
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意：调用接口中的static方法，还是使用invokespecial

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class sample/HelloWorld$1
       3: dup
       4: aload_0
       5: invokespecial #3                  // Method sample/HelloWorld$1."<init>":(Lsample/HelloWorld;)V
       8: astore_1
       9: aload_1
      10: invokeinterface #4,  1            // InterfaceMethod sample/MyInterface.defaultMethod:()V
      15: aload_1
      16: invokeinterface #5,  1            // InterfaceMethod sample/MyInterface.targetMethod:()V
      21: invokestatic  #6                  // InterfaceMethod sample/MyInterface.staticMethod:()V
      24: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "sample/HelloWorld$1");
methodVisitor.visitInsn(DUP);
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "sample/HelloWorld$1", "<init>", "(Lsample/HelloWorld;)V", false);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitMethodInsn(INVOKEINTERFACE, "sample/MyInterface", "defaultMethod", "()V", true);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitMethodInsn(INVOKEINTERFACE, "sample/MyInterface", "targetMethod", "()V", true);
methodVisitor.visitMethodInsn(INVOKESTATIC, "sample/MyInterface", "staticMethod", "()V", true);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_HelloWorld$1}
0003: dup                      // {this} | {uninitialized_HelloWorld$1, uninitialized_HelloWorld$1}
0004: aload_0                  // {this} | {uninitialized_HelloWorld$1, uninitialized_HelloWorld$1, this}
0005: invokespecial   #3       // {this} | {HelloWorld$1}
0008: astore_1                 // {this, HelloWorld$1} | {}
0009: aload_1                  // {this, HelloWorld$1} | {HelloWorld$1}
0010: invokeinterface #4  1    // {this, HelloWorld$1} | {}
0015: aload_1                  // {this, HelloWorld$1} | {HelloWorld$1}
0016: invokeinterface #5  1    // {this, HelloWorld$1} | {}
0021: invokestatic    #6       // {this, HelloWorld$1} | {}
0024: return                   // {} | {}
```

从JVM规范的角度来看，`invokeinterface`指令对应的Operand Stack的变化如下：

> 因为接口方法除了static修饰的以外，默认都是抽象方法，所以需要实例实现接口后，再通过实例调用接口方法

```pseudocode
..., objectref, [arg1, [arg2 ...]] →

...
```

The `objectref` must be of type `reference` and must be followed on the operand stack by `nargs` argument values, where the number, type, and order of the values must be consistent with the descriptor of the resolved interface method.

### 2.7.6 invokedynamic

> **就整体使用上而言，invokedynamic比较类似于invokestatic，因为都不需要实例指针**

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
import java.util.function.Consumer;

public class HelloWorld {
  public void test() {
    Consumer<String> c = System.out::println;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意：这里多出来的`dup` => `invokevirtual` => `pop`，是java编译时自动生成的，为的是确保被使用的类一定会加载到JVM中（原可能未加载or已经不在JVM中存在了）。
>
> 这里即保证PrintStream被加载到JVM中，这样System.out的类方法`println()`才能被正常使用

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: dup
       4: invokevirtual #3                  // Method java/lang/Object.getClass:()Ljava/lang/Class;
       7: pop
       8: invokedynamic #4,  0              // InvokeDynamic #0:accept:(Ljava/io/PrintStream;)Ljava/util/function/Consumer;
      13: astore_1
      14: return
}
```

从ASM的视角来看，方法体对应的内容如下：

> **可以看出来使用ASM生成lambda表达式比较复杂容易出错，一般不建议直接ASM生成，而是将拥有lambda表达式的代码封装到一个method中，然后直接用ASM调用该method。**

```java
methodVisitor.visitCode();
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Object", "getClass", "()Ljava/lang/Class;", false);
methodVisitor.visitInsn(POP);
methodVisitor.visitInvokeDynamicInsn("accept", "(Ljava/io/PrintStream;)Ljava/util/function/Consumer;", 
    new Handle(Opcodes.H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory", 
        "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false), 
    new Object[]{Type.getType("(Ljava/lang/Object;)V"), 
    new Handle(Opcodes.H_INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false), 
    Type.getType("(Ljava/lang/String;)V")});
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: getstatic       #2       // {this} | {PrintStream}
0003: dup                      // {this} | {PrintStream, PrintStream}
0004: invokevirtual   #3       // {this} | {PrintStream, Class}
0007: pop                      // {this} | {PrintStream}
0008: invokedynamic   #4       // {this} | {Consumer}
0013: astore_1                 // {this, Consumer} | {}
0014: return                   // {} | {}
```

从JVM规范的角度来看，`invokedynamic`指令对应的Operand Stack的变化如下：

> 和invokestatic类似，不需要对象指针，即可完成方法调用

```pseudocode
..., [arg1, [arg2 ...]] →

...
```

### 2.7.7 小结

+ 除了private、构造方法`<init>`、`super.xxx()`使用invokespecial，其他实例方法的调用，都是使用invokevirtual（所以大部分情况都是使用invokevirtual）
+ 只要是static方法，不管是定义在类还是接口中，都是使用invokestatic
+ 非static修饰的其他定义在接口中的方法，调用时使用invokeinterface
+ lambada表达式调用方法，使用invokedynamic（ASM生成lambda表达式逻辑，复杂容易出错，一般建议将lambda封装到一个method中，然后ASM直接调用该method）

## 2.8 opcode: array (20/160/205)

### 2.8.1 概览

从Instruction的角度来说，与array相关的opcode有20个，内容如下：

> **注意，boolean数组没有单独的指令，和byte数组共用`bastore`和`baload`**

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 188    | newarray        | 189    | anewarray       | 190    | arraylength     | 197    | multianewarray  |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 46     | iaload          | 48     | faload          | 50     | aaload          | 52     | caload          |
| 47     | laload          | 49     | daload          | 51     | baload          | 53     | saload          |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 79     | iastore         | 81     | fastore         | 83     | aastore         | 85     | castore         |
| 80     | lastore         | 82     | dastore         | 84     | bastore         | 86     | sastore         |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitIntInsn()`: `newarray`
- `MethodVisitor.visitTypeInsn()`: `anewarray`
- `MethodVisitor.visitMultiANewArrayInsn()`: `multianewarray`
- `MethodVisitor.visitInsn()`:
  - `arraylength`
  - `iaload`, `iastore`
  - `laload`, `lastore`
  - `faload`, `fastore`
  - `daload`, `dastore`
  - `aaload`, `aastore`
  - `baload`, `bastore`
  - `caload`, `castore`
  - `saload`, `sastore`

### 2.8.2 create array

#### newarray: primitive type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    byte[] byteArray = new byte[5];
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
       0: iconst_5
       1: newarray       byte // 这里.class实际是数字8表示byte，只是javap解析后直接显示为byte方便阅读
       3: astore_1
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitIntInsn(NEWARRAY, T_BYTE);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

> 这里可以看到`.class`实际用8表示byte，而不是直接存储字符串"byte"

```pseudocode
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray   8 (byte)      // {this} | {[B} // newarray 消耗一个表示数组大小的数值，然后入栈数组指针
0003: astore_1                 // {this, [B} | {}
0004: return                   // {} | {}
```

从JVM规范的角度来看，`newarray`指令对应的Operand Stack的变化如下：

```pseudocode
..., count →

..., arrayref
```

The `count` must be of type `int`. It is popped off the operand stack. The `count` represents the number of elements in the array to be created.

另外，`newarray`指令对应的Format如下：

```
newarray
atype
```

The [`atype`](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.newarray) is a code that indicates the type of array to create. It must take one of the following values:

| Array Type | atype |
| ---------- | ----- |
| T_BOOLEAN  | 4     |
| T_CHAR     | 5     |
| T_FLOAT    | 6     |
| T_DOUBLE   | 7     |
| T_BYTE     | 8     |
| T_SHORT    | 9     |
| T_INT      | 10    |
| T_LONG     | 11    |

#### anewarray: reference type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object[] objArray = new Object[5];
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
       0: iconst_5
       1: anewarray     #2                  // class java/lang/Object
       4: astore_1
       5: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitTypeInsn(ANEWARRAY, "java/lang/Object");
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: anewarray       #2       // {this} | {[Object}
0004: astore_1                 // {this, [Object} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`anewarray`指令对应的Operand Stack的变化如下：

```pseudocode
..., count →

..., arrayref
```

- The `count` must be of type `int`. It is popped off the operand stack. The `count` represents the number of components of the array to be created.
- A new array with components of that type, of length `count`, is allocated from the garbage-collected heap, and a reference `arrayref` to this new array object is pushed onto the operand stack.
- All components of the new array are initialized to `null`, the default value for reference types.

#### multianewarray

> 多维数组使用multianewarray

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object[][] array = new Object[3][4];
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> `multianewarray #2,  2`，第二个"2"表示数组的维度是2

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_3
       1: iconst_4
       2: multianewarray #2,  2             // class "[[Ljava/lang/Object;"
       6: astore_1
       7: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitMultiANewArrayInsn("[[Ljava/lang/Object;", 2);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_3                 // {this} | {int}
0001: iconst_4                 // {this} | {int, int} // 因为需要多个维度的数值，所以加载多个数值到operand stack
0002: multianewarray  #2  2    // {this} | {[[Object}
0006: astore_1                 // {this, [[Object} | {}
0007: return                   // {} | {}
```

从JVM规范的角度来看，`multianewarray`指令对应的Operand Stack的变化如下：

```pseudocode
..., count1, [count2, ...] →

..., arrayref
```

- All of the count values are popped off the operand stack.
- A new multidimensional array of the array type is allocated from the garbage-collected heap. A reference `arrayref` to the new array is pushed onto the operand stack.

另外，`multianewarray`指令对应的Format如下：

```pseudocode
multianewarray
indexbyte1
indexbyte2
dimensions
```

The `dimensions` operand is an unsigned byte that must be greater than or equal to 1. It represents the number of dimensions of the array to be created. The operand stack must contain `dimensions` values. Each such value represents the number of components in a dimension of the array to be created, must be of type `int`, and must be non-negative. The `count1` is the desired length in the first dimension, `count2` in the second, etc.

#### special

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    // 注意，这里是一个二维数据，但只提供了第一维的大小
    int[][] array = new int[10][];
    // 这里array并不能直接使用，使用时需要如下操作
    // array[0] = new int[10];
    // array[0][1] = 1;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 可以看到这里虽然声明二维数组，但是由于第二维还没有初始化，所以实际还是被当作一维处理

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: bipush        10
       2: anewarray     #2                  // class "[I"
       5: astore_1
       6: return
}
```

### 2.8.3 array element

#### int array

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int[] intArray = new int[5];
    intArray[0] = 10;
    int i = intArray[0];
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
       0: iconst_5
       1: newarray       int
       3: astore_1
       4: aload_1
       5: iconst_0
       6: bipush        10
       8: iastore
       9: aload_1
      10: iconst_0
      11: iaload
      12: istore_2
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitIntInsn(NEWARRAY, T_INT);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitIntInsn(BIPUSH, 10);
methodVisitor.visitInsn(IASTORE);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(IALOAD);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray        int      // {this} | {[I}
0003: astore_1                 // {this, [I} | {}
0004: aload_1                  // {this, [I} | {[I}
0005: iconst_0                 // {this, [I} | {[I, int}
0006: bipush          10       // {this, [I} | {[I, int, int}
0008: iastore                  // {this, [I} | {} // 将数组下标0的元素赋值10
0009: aload_1                  // {this, [I} | {[I}
0010: iconst_0                 // {this, [I} | {[I, int}
0011: iaload                   // {this, [I} | {int} // 将数组下标0的元素加载到operand stack
0012: istore_2                 // {this, [I, int} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`iastore`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index, value →

...
```

The `arrayref` must be of type `reference` and must refer to an array whose components are of type `int`. Both `index` and `value` must be of type `int`. The `arrayref`, `index`, and `value` are popped from the operand stack. The `int` value is stored as the component of the array indexed by `index`.

从JVM规范的角度来看，`iaload`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index →

..., value
```

The `arrayref` must be of type `reference` and must refer to an array whose components are of type `int`. The `index` must be of type `int`. Both `arrayref` and `index` are popped from the operand stack. The `int` value in the component of the array at `index` is retrieved and pushed onto the operand stack.

#### Object Array

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Object[] objArray = new Object[2];
    objArray[0] = null;
    Object obj = objArray[0];
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
       0: iconst_2
       1: anewarray     #2                  // class java/lang/Object
       4: astore_1
       5: aload_1
       6: iconst_0
       7: aconst_null
       8: aastore
       9: aload_1
      10: iconst_0
      11: aaload
      12: astore_2
      13: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitTypeInsn(ANEWARRAY, "java/lang/Object");
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(ACONST_NULL);
methodVisitor.visitInsn(AASTORE);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(AALOAD);
methodVisitor.visitVarInsn(ASTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_2                 // {this} | {int}
0001: anewarray       #2       // {this} | {[Object}
0004: astore_1                 // {this, [Object} | {}
0005: aload_1                  // {this, [Object} | {[Object}
0006: iconst_0                 // {this, [Object} | {[Object, int}
0007: aconst_null              // {this, [Object} | {[Object, int, null}
0008: aastore                  // {this, [Object} | {}
0009: aload_1                  // {this, [Object} | {[Object}
0010: iconst_0                 // {this, [Object} | {[Object, int}
0011: aaload                   // {this, [Object} | {Object}
0012: astore_2                 // {this, [Object, Object} | {}
0013: return                   // {} | {}
```

从JVM规范的角度来看，`aastore`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index, value →

...
```

The `arrayref` must be of type `reference` and must refer to an array whose components are of type `reference`. The `index` must be of type `int` and `value` must be of type `reference`. The `arrayref`, `index`, and `value` are popped from the operand stack. The reference `value` is stored as the component of the array at `index`.

从JVM规范的角度来看，`aaload`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index →

..., value
```

The `arrayref` must be of type `reference` and must refer to an array whose components are of type `reference`. The `index` must be of type `int`. Both `arrayref` and `index` are popped from the operand stack. The reference `value` in the component of the array at `index` is retrieved and pushed onto the operand stack.

#### multi-dimensions array

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int[][][] array = new int[4][5][6];
    array[1][2][3] = 10;
    int val = array[1][2][3];
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
       0: iconst_4
       1: iconst_5
       2: bipush        6
       4: multianewarray #2,  3             // class "[[[I"
       8: astore_1
       9: aload_1
      10: iconst_1
      11: aaload
      12: iconst_2
      13: aaload
      14: iconst_3
      15: bipush        10
      17: iastore
      18: aload_1
      19: iconst_1
      20: aaload
      21: iconst_2
      22: aaload
      23: iconst_3
      24: iaload
      25: istore_2
      26: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitIntInsn(BIPUSH, 6);
methodVisitor.visitMultiANewArrayInsn("[[[I", 3);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitInsn(AALOAD);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitInsn(AALOAD);
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitIntInsn(BIPUSH, 10);
methodVisitor.visitInsn(IASTORE);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitInsn(AALOAD);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitInsn(AALOAD);
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitInsn(IALOAD);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

> 整体思路即，读写元素时需要先将多维数组层层解析至1维数组，然后再操作

```pseudocode
                               // {this} | {}
0000: iconst_4                 // {this} | {int}
0001: iconst_5                 // {this} | {int, int}
0002: bipush          6        // {this} | {int, int, int}
0004: multianewarray  #2  3    // {this} | {[[[I} // 创建3维int数组
0008: astore_1                 // {this, [[[I} | {}
0009: aload_1                  // {this, [[[I} | {[[[I}
0010: iconst_1                 // {this, [[[I} | {[[[I, int}
0011: aaload                   // {this, [[[I} | {[[I} // 加载array[1]，解析成2维数组
0012: iconst_2                 // {this, [[[I} | {[[I, int}
0013: aaload                   // {this, [[[I} | {[I} // 加载array[1][2]，解析成1维数组
0014: iconst_3                 // {this, [[[I} | {[I, int}
0015: bipush          10       // {this, [[[I} | {[I, int, int}
0017: iastore                  // {this, [[[I} | {} // 将10赋值到 arrya[1][2][3]
0018: aload_1                  // {this, [[[I} | {[[[I}
0019: iconst_1                 // {this, [[[I} | {[[[I, int}
0020: aaload                   // {this, [[[I} | {[[I}
0021: iconst_2                 // {this, [[[I} | {[[I, int}
0022: aaload                   // {this, [[[I} | {[I}
0023: iconst_3                 // {this, [[[I} | {[I, int}
0024: iaload                   // {this, [[[I} | {int}
0025: istore_2                 // {this, [[[I, int} | {}
0026: return                   // {} | {}
```

### 2.8.4 array length

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int[] intArray = new int[5];
    int length = intArray.length;
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
       0: iconst_5
       1: newarray       int
       3: astore_1
       4: aload_1
       5: arraylength
       6: istore_2
       7: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitIntInsn(NEWARRAY, T_INT);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ARRAYLENGTH);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray        int      // {this} | {[I}
0003: astore_1                 // {this, [I} | {}
0004: aload_1                  // {this, [I} | {[I}
0005: arraylength              // {this, [I} | {int}
0006: istore_2                 // {this, [I, int} | {}
0007: return                   // {} | {}
```

从JVM规范的角度来看，`arraylength`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref →

..., length
```

The `arrayref` must be of type reference and must refer to an array. It is popped from the operand stack. The `length` of the array it references is determined. That `length` is pushed onto the operand stack as an `int`.

### 2.8.5 boolean

在[Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html)当中，有如下几段描述：

- Like the Java programming language, the Java Virtual Machine operates on two kinds of types: **primitive types** and **reference types**.
- The **primitive data types** supported by the Java Virtual Machine are **the numeric types**, **the boolean type**, and **the returnAddress type**.
- There are three kinds of **reference types**: **class types**, **array types**, and **interface types**.

经过整理，我们可以将JVM当中的类型表示成如下形式：

- JVM Types
  - primitive types
    - numeric types
      - integral types: `byte`, `short`, `int`, `long`, `char`
      - floating-point types: `float`, `double`
    - `boolean` type
    - `returnAddress` type
  - reference types
    - class types
    - array types
    - interface types

**在这里，我们关注`boolean`、`byte`、`short`、`char`、`int`这几个primitive types；它们在local variable和operand stack当中都是作为`int`类型来进行处理。**

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

为什么我们要关注`boolean`、`byte`、`short`、`char`、`int`这几个primitive types呢？因为我们想把`boolean`单独拿出来，与`byte`、`short`、`char`、`int`进行一下对比：

- 第一次对比，作为primitive types进行对比：`boolean`、`byte`、`short`、`char`、`int`。（处理相同）
- 第二次对比，作为array types(reference types)进行对比：`boolean[]`、`byte[]`、`short[]`、`char[]`、`int[]`。（处理出现差异）

#### boolean type

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    boolean flag_false = false;
    boolean flag_true = true;
  }
}
```

或者代码如下（`byte`类型可以替换成`short`、`char`、`int`类型）：

```java
public class HelloWorld {
  public void test() {
    byte val0 = 0;
    byte val1 = 1;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 可以看出来上面举例的byte和boolean在operand stack、local variables都被当作int处理，使用i开头的指令

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_0
       1: istore_1
       2: iconst_1
       3: istore_2
       4: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_0                 // {this} | {int}
0001: istore_1                 // {this, int} | {}
0002: iconst_1                 // {this, int} | {int}
0003: istore_2                 // {this, int, int} | {}
0004: return                   // {} | {}
```

#### boolean array

我们先来看一个`int[]`类型的示例。从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int[] array = new int[5];
    array[0] = 1;
    int val = array[1];
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
       0: iconst_5
       1: newarray       int
       3: astore_1
       4: aload_1
       5: iconst_0
       6: iconst_1
       7: iastore
       8: aload_1
       9: iconst_1
      10: iaload
      11: istore_2
      12: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray        int      // {this} | {[I}
0003: astore_1                 // {this, [I} | {}
0004: aload_1                  // {this, [I} | {[I}
0005: iconst_0                 // {this, [I} | {[I, int}
0006: iconst_1                 // {this, [I} | {[I, int, int}
0007: iastore                  // {this, [I} | {}
0008: aload_1                  // {this, [I} | {[I}
0009: iconst_1                 // {this, [I} | {[I, int}
0010: iaload                   // {this, [I} | {int}
0011: istore_2                 // {this, [I, int} | {}
0012: return                   // {} | {}
```

|          | int[]   | char[]  | short[] | byte[]  | boolean[]   |
| -------- | ------- | ------- | ------- | ------- | ----------- |
| 加载数据 | iaload  | caload  | saload  | baload  | **baload**  |
| 存储数据 | iastore | castore | sastore | bastore | **bastore** |

接下来，我们来验证一下`boolean[]`。从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    boolean[] boolArray = new boolean[5];
    boolArray[0] = true;
    boolean flag = boolArray[1];
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 可以看出来，在boolean数组读写操作中，使用和byte数组一致的**bastore**和**baload**

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_5
       1: newarray       boolean
       3: astore_1
       4: aload_1
       5: iconst_0
       6: iconst_1
       7: bastore
       8: aload_1
       9: iconst_1
      10: baload
      11: istore_2
      12: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_5);
methodVisitor.visitIntInsn(NEWARRAY, T_BOOLEAN);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitInsn(BASTORE);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitInsn(BALOAD);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(3, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray        boolean  // {this} | {[Z}
0003: astore_1                 // {this, [Z} | {}
0004: aload_1                  // {this, [Z} | {[Z}
0005: iconst_0                 // {this, [Z} | {[Z, int}
0006: iconst_1                 // {this, [Z} | {[Z, int, int}
0007: bastore                  // {this, [Z} | {} // 可以看出来虽然Z对应boolean，但是使用指令bastore(和byte数组一致)
0008: aload_1                  // {this, [Z} | {[Z}
0009: iconst_1                 // {this, [Z} | {[Z, int}
0010: baload                   // {this, [Z} | {int}
0011: istore_2                 // {this, [Z, int} | {}
0012: return                   // {} | {}
```

从JVM规范的角度来看，`bastore`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index, value →

...
```

**The `arrayref` must be of type `reference` and must refer to an array whose components are of type `byte` or of type `boolean`.** The `index` and the `value` must both be of type `int`. The `arrayref`, `index`, and `value` are popped from the operand stack. The int `value` is truncated to a `byte` and stored as the component of the array indexed by `index`.

从JVM规范的角度来看，`baload`指令对应的Operand Stack的变化如下：

```pseudocode
..., arrayref, index →

..., value
```

**The `arrayref` must be of type `reference` and must refer to an array whose components are of type `byte` or of type `boolean`.** The `index` must be of type `int`. Both `arrayref` and `index` are popped from the operand stack. The byte `value` in the component of the array at `index` is retrieved, sign-extended to an `int` value, and pushed onto the top of the operand stack.

------

在[Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html)的[2.3.4. The boolean Type](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3.4)对`boolean`类型进行了如下描述：

Although the Java Virtual Machine defines a `boolean` type, it only provides very limited support for it.

- **There are no Java Virtual Machine instructions solely dedicated to operations on `boolean` values**.（没有相应的opcode）
- Instead, expressions in the Java programming language that operate on `boolean` values are compiled to use values of the Java Virtual Machine `int` data type.

The Java Virtual Machine does directly support **boolean arrays**.

- Its `newarray` instruction enables creation of boolean arrays.（创建数组）
- Arrays of type boolean are accessed and modified using the byte array instructions `baload` and `bastore`.（存取数据）

In Oracle’s Java Virtual Machine implementation, **boolean arrays** in the Java programming language are encoded as Java Virtual Machine **byte arrays**, using 8 bits per boolean element.

The Java Virtual Machine encodes `boolean` array components using `1` to represent `true` and `0` to represent `false`. Where Java programming language `boolean` values are mapped by compilers to values of Java Virtual Machine type `int`, the compilers must use the same encoding.

## 2.9 opcode: jump (25/185/205)

### 2.9.1 概览

从Instruction的角度来说，与jump相关的opcode有25个，内容如下：

> lcmp即两个long比较，a>b返回1，a=b返回0，a<b返回-1
>
> **java里float、double、long的比较，没有类似ifXXX直接跳转的指令，而是先比较得到一个int值，再调用ifXX跳转**
>
> fcmpl和fcmpg都是float的比较，区别仅对于NaN的处理 => [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dup2) 
>
> `fcmp<op>`
>
> - Otherwise, at least one of *value1*' or *value2*' is NaN. The *fcmpg* instruction pushes the `int` value 1 onto the operand stack and the *fcmpl* instruction pushes the `int` value -1 onto the operand stack.
>
> ifeq即int和0比较，相等则跳转
>
> if_ifcmple即int的a和b比较，a<=b则跳转
>
> tableswitch和loopupswitch都是java中的switch跳转，简单理解的话前者是case中数字连续的情况，后则是case中数字非连续的情况
>
> goto地址2字节，goto_w地址4字节

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 148    | lcmp            | 153    | ifeq            | 158    | ifle            | 163    | if_icmpgt       |
| 149    | fcmpl           | 154    | ifne            | 159    | if_icmpeq       | 164    | if_icmple       |
| 150    | fcmpg           | 155    | iflt            | 160    | if_icmpne       | 165    | if_acmpeq       |
| 151    | dcmpl           | 156    | ifge            | 161    | if_icmplt       | 166    | if_acmpne       |
| 152    | dcmpg           | 157    | ifgt            | 162    | if_icmpge       | 167    | goto            |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 170    | tableswitch     | 171    | lookupswitch    |        |                 |        |                 |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 198    | ifnull          | 199    | ifnonnull       | 200    | goto_w          |        |                 |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`: `lcmp`, `fcmpl`, `fcmpg`, `dcmpl`, `dcmpg`
- `MethodVisitor.visitJumpInsn()`:
  - `ifeq`, `ifne`, `iflt`, `ifge`, `ifgt`, `ifle`
  - `if_icmpeq`, `if_icmpne`, `if_icmplt`, `if_icmpge`, `if_icmpgt`, `if_icmple`, `if_acmpeq`, `if_acmpne`
  - `ifnull`, `ifnonnull`
  - `goto`, `goto_w`
- `MethodVisitor.visitTableSwitchInsn()`: `tableswitch`, `lookupswitch`

### 2.9.2 if and goto

#### compare int with zero

从Java语言的视角，有一个`HelloWorld`类，代码如下：

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

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: iload_1
       1: ifne          15 // 注意生成的指令是 ifne 而非 ifeq
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

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitJumpInsn(IFNE, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("val is 0");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("val is not 0");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this, int} | {}
0000: iload_1                  // {this, int} | {int}
0001: ifne            14       // {this, int} | {} // 注意`.class`内存跳转的偏移量，1+14=15
0004: getstatic       #2       // {this, int} | {PrintStream}
0007: ldc             #3       // {this, int} | {PrintStream, String}
0009: invokevirtual   #4       // {this, int} | {}
0012: goto            11       // {} | {}
                               // {this, int} | {}
0015: getstatic       #2       // {this, int} | {PrintStream}
0018: ldc             #5       // {this, int} | {PrintStream, String}
0020: invokevirtual   #4       // {this, int} | {}
                               // {this, int} | {}
0023: return                   // {} | {}
```

从JVM规范的角度来看，`ifne`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

...
```

The `value` must be of type `int`. It is popped from the operand stack and compared against **zero**. All comparisons are signed. The results of the comparisons are as follows:

- `ifeq` succeeds if and only if `value = 0`
- `ifne` succeeds if and only if `value ≠ 0`
- `iflt` succeeds if and only if `value < 0`
- `ifle` succeeds if and only if `value ≤ 0`
- `ifgt` succeeds if and only if `value > 0`
- `ifge` succeeds if and only if `value ≥ 0`

#### compare int with non-zero

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int a, int b) {
    if (a > b) {
      System.out.println("a > b");
    }
    else {
      System.out.println("a <= b");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: if_icmple     16 // 如果 a <= b 则跳转
       5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: ldc           #3                  // String a > b
      10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: goto          24
      16: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      19: ldc           #5                  // String a <= b
      21: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      24: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitJumpInsn(IF_ICMPLE, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("a > b");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("a <= b");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, int, int} | {}
0000: iload_1                  // {this, int, int} | {int}
0001: iload_2                  // {this, int, int} | {int, int}
0002: if_icmple       14       // {this, int, int} | {}
0005: getstatic       #2       // {this, int, int} | {PrintStream}
0008: ldc             #3       // {this, int, int} | {PrintStream, String}
0010: invokevirtual   #4       // {this, int, int} | {}
0013: goto            11       // {} | {}
                               // {this, int, int} | {}
0016: getstatic       #2       // {this, int, int} | {PrintStream}
0019: ldc             #5       // {this, int, int} | {PrintStream, String}
0021: invokevirtual   #4       // {this, int, int} | {}
                               // {this, int, int} | {}
0024: return                   // {} | {}
```

从JVM规范的角度来看，`if_icmple`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

...
```

Both `value1` and `value2` must be of type `int`. They are both popped from the operand stack and compared. All comparisons are signed. The results of the comparison are as follows:

- `if_icmpeq` succeeds if and only if `value1 = value2`
- `if_icmpne` succeeds if and only if `value1 ≠ value2`
- `if_icmplt` succeeds if and only if `value1 < value2`
- `if_icmple` succeeds if and only if `value1 ≤ value2`
- `if_icmpgt` succeeds if and only if `value1 > value2`
- `if_icmpge` succeeds if and only if `value1 ≥ value2`

#### compare long

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(long a, long b) {
    if (a > b) {
      System.out.println("a > b");
    }
    else {
      System.out.println("a <= b");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

> 注意：因为long、float、double没有直接比较后就进行跳转的指令，所以lcmp比较后先得到int入栈，然后再借助ifle判断a<=b是否成立，成立则跳转到java代码中对应else语句块中

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(long, long);
    Code:
       0: lload_1
       1: lload_3
       2: lcmp
       3: ifle          17
       6: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #3                  // String a > b
      11: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: goto          25
      17: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      20: ldc           #5                  // String a <= b
      22: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      25: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitVarInsn(LLOAD, 3);
methodVisitor.visitInsn(LCMP);
methodVisitor.visitJumpInsn(IFLE, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("a > b");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("a <= b");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(4, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, long, top, long, top} | {}
0000: lload_1                  // {this, long, top, long, top} | {long, top}
0001: lload_3                  // {this, long, top, long, top} | {long, top, long, top}
0002: lcmp                     // {this, long, top, long, top} | {int} // lcmp消耗栈顶2个long，得到int结果
0003: ifle            14       // {this, long, top, long, top} | {}
0006: getstatic       #2       // {this, long, top, long, top} | {PrintStream}
0009: ldc             #3       // {this, long, top, long, top} | {PrintStream, String}
0011: invokevirtual   #4       // {this, long, top, long, top} | {}
0014: goto            11       // {} | {}
                               // {this, long, top, long, top} | {}
0017: getstatic       #2       // {this, long, top, long, top} | {PrintStream}
0020: ldc             #5       // {this, long, top, long, top} | {PrintStream, String}
0022: invokevirtual   #4       // {this, long, top, long, top} | {}
                               // {this, long, top, long, top} | {}
0025: return                   // {} | {}
```

从JVM规范的角度来看，`lcmp`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

..., result
```

Both `value1` and `value2` must be of type `long`. They are both popped from the operand stack, and a signed integer comparison is performed. If `value1` is greater than `value2`, the int value `1` is pushed onto the operand stack. If `value1` is equal to `value2`, the int value `0` is pushed onto the operand stack. If `value1` is less than `value2`, the int value `-1` is pushed onto the operand stack.

#### compare obj with null

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(Object obj) {
    if (obj == null) {
      System.out.println("obj is null");
    }
    else {
      System.out.println("obj is not null");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(java.lang.Object);
    Code:
       0: aload_1
       1: ifnonnull     15
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String obj is null
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: goto          23
      15: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #5                  // String obj is not null
      20: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitJumpInsn(IFNONNULL, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("obj is null");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("obj is not null");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, Object} | {}
0000: aload_1                  // {this, Object} | {Object}
0001: ifnonnull       14       // {this, Object} | {}
0004: getstatic       #2       // {this, Object} | {PrintStream}
0007: ldc             #3       // {this, Object} | {PrintStream, String}
0009: invokevirtual   #4       // {this, Object} | {}
0012: goto            11       // {} | {}
                               // {this, Object} | {}
0015: getstatic       #2       // {this, Object} | {PrintStream}
0018: ldc             #5       // {this, Object} | {PrintStream, String}
0020: invokevirtual   #4       // {this, Object} | {}
                               // {this, Object} | {}
0023: return                   // {} | {}
```

从JVM规范的角度来看，`ifnonnull`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

...
```

另外，`ifnonnull`指令对应的Format如下：

```pseudocode
ifnonnull
branchbyte1
branchbyte2
```

The `value` must be of type `reference`. It is popped from the operand stack. If `value` is not `null`, the unsigned `branchbyte1` and `branchbyte2` are used to construct a signed 16-bit offset, where the offset is calculated to be `(branchbyte1 << 8) | branchbyte2`. Execution then proceeds at that offset from the address of the opcode of this `ifnonnull` instruction. The target address must be that of an opcode of an instruction within the method that contains this `ifnonnull` instruction.

Otherwise, execution proceeds at the address of the instruction following this `ifnonnull` instruction.

#### compare objA with objB

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(Object objA, Object objB) {
    if (objA == objB) {
      System.out.println("objA == objB");
    }
    else {
      System.out.println("objA != objB");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(java.lang.Object, java.lang.Object);
    Code:
       0: aload_1
       1: aload_2
       2: if_acmpne     16
       5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: ldc           #3                  // String objA == objB
      10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: goto          24
      16: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      19: ldc           #5                  // String objA != objB
      21: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      24: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();

methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitVarInsn(ALOAD, 2);
methodVisitor.visitJumpInsn(IF_ACMPNE, label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("objA == objB");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label1);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("objA != objB");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, Object, Object} | {}
0000: aload_1                  // {this, Object, Object} | {Object}
0001: aload_2                  // {this, Object, Object} | {Object, Object}
0002: if_acmpne       14       // {this, Object, Object} | {}
0005: getstatic       #2       // {this, Object, Object} | {PrintStream}
0008: ldc             #3       // {this, Object, Object} | {PrintStream, String}
0010: invokevirtual   #4       // {this, Object, Object} | {}
0013: goto            11       // {} | {}
                               // {this, Object, Object} | {}
0016: getstatic       #2       // {this, Object, Object} | {PrintStream}
0019: ldc             #5       // {this, Object, Object} | {PrintStream, String}
0021: invokevirtual   #4       // {this, Object, Object} | {}
                               // {this, Object, Object} | {}
0024: return                   // {} | {}
```

从JVM规范的角度来看，`if_acmpne`指令对应的Operand Stack的变化如下：

```pseudocode
..., value1, value2 →

...
```

Both `value1` and `value2` must be of type `reference`. They are both popped from the operand stack and compared. The results of the comparison are as follows:

- `if_acmpeq` succeeds if and only if `value1 = value2`
- `if_acmpne` succeeds if and only if `value1 ≠ value2`

### 2.9.3 switch

#### tableswitch

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    int result = 0;

    switch (val) {
      case 1:
        result = 1;
        break;
      case 2:
        result = 2;
        break;
      case 3:
        result = 3;
        break;
      default:
        result = 4;
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: iconst_0
       1: istore_2
       2: iload_1
       3: tableswitch   { // 1 to 3
                     1: 28
                     2: 33
                     3: 38
               default: 43
          }
      28: iconst_1
      29: istore_2
      30: goto          45
      33: iconst_2
      34: istore_2
      35: goto          45
      38: iconst_3
      39: istore_2
      40: goto          45
      43: iconst_4
      44: istore_2
      45: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();
Label label2 = new Label();
Label label3 = new Label();
Label label4 = new Label();

methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitTableSwitchInsn(1, 3, label3, new Label[] { label0, label1, label2 });

methodVisitor.visitLabel(label0);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label2);
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label3);
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitVarInsn(ISTORE, 2);

methodVisitor.visitLabel(label4);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, int} | {}
0000: iconst_0                 // {this, int} | {int}
0001: istore_2                 // {this, int, int} | {}
0002: iload_1                  // {this, int, int} | {int}
0003: tableswitch              // {} | {} // 这里 出栈数字，如果数字是1，则3+25=28跳转到28的位置，其他同理
      {
              1: 25
              2: 30
              3: 35
        default: 40
      }
                               // {this, int, int} | {}
0028: iconst_1                 // {this, int, int} | {int}
0029: istore_2                 // {this, int, int} | {}
0030: goto            15       // {} | {}
                               // {this, int, int} | {}
0033: iconst_2                 // {this, int, int} | {int}
0034: istore_2                 // {this, int, int} | {}
0035: goto            10       // {} | {}
                               // {this, int, int} | {}
0038: iconst_3                 // {this, int, int} | {int}
0039: istore_2                 // {this, int, int} | {}
0040: goto            5        // {} | {}
                               // {this, int, int} | {}
0043: iconst_4                 // {this, int, int} | {int}
0044: istore_2                 // {this, int, int} | {}
                               // {this, int, int} | {}
0045: return                   // {} | {}
```

从JVM规范的角度来看，`tableswitch`指令对应的Operand Stack的变化如下：

```pseudocode
..., index →

...
```

另外，`tableswitch`指令对应的Format如下：

```pseudocode
tableswitch
<0-3 byte pad>
defaultbyte1
defaultbyte2
defaultbyte3
defaultbyte4
lowbyte1
lowbyte2
lowbyte3
lowbyte4
highbyte1
highbyte2
highbyte3
highbyte4
jump offsets...
```

The `index` must be of type `int` and is popped from the operand stack.

- If `index` is less than `low` or `index` is greater than `high`, then a target address is calculated by adding `default` to the address of the opcode of this `tableswitch` instruction.
- Otherwise, the offset at position `index - low` of the jump table is extracted. The target address is calculated by adding that offset to the address of the opcode of this `tableswitch` instruction. Execution then continues at the target address.

#### lookupswitch

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    int result = 0;

    switch (val) {
      case 10:
        result = 1;
        break;
      case 20:
        result = 2;
        break;
      case 30:
        result = 3;
        break;
      default:
        result = 4;
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```pseudocode
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: iconst_0
       1: istore_2
       2: iload_1
       3: lookupswitch  { // 3
                    10: 36
                    20: 41
                    30: 46
               default: 51
          }
      36: iconst_1
      37: istore_2
      38: goto          53
      41: iconst_2
      42: istore_2
      43: goto          53
      46: iconst_3
      47: istore_2
      48: goto          53
      51: iconst_4
      52: istore_2
      53: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();
Label label2 = new Label();
Label label3 = new Label();
Label label4 = new Label();

methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitLookupSwitchInsn(label3, new int[] { 10, 20, 30 }, new Label[] { label0, label1, label2 });

methodVisitor.visitLabel(label0);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label2);
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label3);
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitVarInsn(ISTORE, 2);

methodVisitor.visitLabel(label4);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, int} | {}
0000: iconst_0                 // {this, int} | {int}
0001: istore_2                 // {this, int, int} | {}
0002: iload_1                  // {this, int, int} | {int}
0003: lookupswitch             // {} | {}
      {
             10: 33
             20: 38
             30: 43
        default: 48
      }
                               // {this, int, int} | {}
0036: iconst_1                 // {this, int, int} | {int}
0037: istore_2                 // {this, int, int} | {}
0038: goto            15       // {} | {}
                               // {this, int, int} | {}
0041: iconst_2                 // {this, int, int} | {int}
0042: istore_2                 // {this, int, int} | {}
0043: goto            10       // {} | {}
                               // {this, int, int} | {}
0046: iconst_3                 // {this, int, int} | {int}
0047: istore_2                 // {this, int, int} | {}
0048: goto            5        // {} | {}
                               // {this, int, int} | {}
0051: iconst_4                 // {this, int, int} | {int}
0052: istore_2                 // {this, int, int} | {}
                               // {this, int, int} | {}
0053: return                   // {} | {}
```

从JVM规范的角度来看，`lookupswitch`指令对应的Operand Stack的变化如下：

```pseudocode
..., key →

...
```

另外，`lookupswitch`指令对应的Format如下：

```pseudocode
lookupswitch
<0-3 byte pad>
defaultbyte1
defaultbyte2
defaultbyte3
defaultbyte4
npairs1
npairs2
npairs3
npairs4
match-offset pairs...
```

The `key` must be of type `int` and is popped from the operand stack. The `key` is compared against the match values.

- If it is equal to one of them, then a target address is calculated by adding the corresponding offset to the address of the opcode of this `lookupswitch` instruction.
- If the `key` does not match any of the match values, the target address is calculated by adding `default` to the address of the opcode of this `lookupswitch` instruction. Execution then continues at the target address.

## 2.10 opcode: stack (9/194/205)

### 2.10.1 概览

从Instruction的角度来说，与stack相关的opcode有9个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 87     | pop             | 90     | dup_x1          | 93     | dup2_x1         |
| 88     | pop2            | 91     | dup_x2          | 94     | dup2_x2         |
| 89     | dup             | 92     | dup2            | 95     | swap            |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`:
  - `pop`, `pop2`
  - `dup`, `dup_x1`, `dup_x2`
  - `dup2`, `dup2_x1`, `dup2_x2`
  - `swap`

### 2.10.2 pop

#### pop

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Math.max(3, 4);
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
       0: iconst_3
       1: iconst_4
       2: invokestatic  #2                  // Method java/lang/Math.max:(II)I
       5: pop // 因为没有本地变量接收 Math.max的返回值，所以生成pop指令
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_3);
methodVisitor.visitInsn(ICONST_4);
methodVisitor.visitMethodInsn(INVOKESTATIC, "java/lang/Math", "max", "(II)I", false);
methodVisitor.visitInsn(POP);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_3                 // {this} | {int}
0001: iconst_4                 // {this} | {int, int}
0002: invokestatic    #2       // {this} | {int}
0005: pop                      // {this} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，`pop`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

...
```

Pop the top `value` from the operand stack. The `pop` instruction must not be used unless `value` is a value of a category 1 computational type.

#### pop2

> 弹出栈顶两个位置的数据（常见用于long、double，因为这两个占用2位置）

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    Math.max(3L, 4L);
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
       0: ldc2_w        #2                  // long 3l
       3: ldc2_w        #4                  // long 4l
       6: invokestatic  #6                  // Method java/lang/Math.max:(JJ)J
       9: pop2 // 因为没有本地变量接收Math.max的返回值，所以栈顶弹出
      10: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Long(3L));
methodVisitor.visitLdcInsn(new Long(4L));
methodVisitor.visitMethodInsn(INVOKESTATIC, "java/lang/Math", "max", "(JJ)J", false);
methodVisitor.visitInsn(POP2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(4, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: ldc2_w          #2       // {this} | {long, top}
0003: ldc2_w          #4       // {this} | {long, top, long, top}
0006: invokestatic    #6       // {this} | {long, top}
0009: pop2                     // {this} | {}
0010: return                   // {} | {}
```

从JVM规范的角度来看，`pop2`指令对应的Operand Stack的变化如下：

Form 1:

```pseudocode
..., value2, value1 →

...
```

where each of `value1` and `value2` is a value of a category 1 computational type.

Form 2:

```pseudocode
..., value →

...
```

where `value` is a value of a category 2 computational type.

### 2.10.3 dup

#### dup

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int a;
    int b;
    b = a = 2;
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
       0: iconst_2
       1: dup
       2: istore_1
       3: istore_2
       4: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitInsn(DUP);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitVarInsn(ISTORE, 2);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: iconst_2                 // {this} | {int}
0001: dup                      // {this} | {int, int} // 赋值栈顶一个位置的数据
0002: istore_1                 // {this, int} | {int} // 给变量a赋值
0003: istore_2                 // {this, int, int} | {} // 给变量b赋值
0004: return                   // {} | {}
```

从JVM规范的角度来看，`dup`指令对应的Operand Stack的变化如下：

```pseudocode
..., value →

..., value, value
```

Duplicate the top value on the operand stack and push the duplicated value onto the operand stack.

The `dup` instruction must not be used unless `value` is a value of a category 1 computational type.

#### dup_x1

> 复制栈顶的1个位置数据，并且是插入到当前栈顶往下再间隔1个位置的地方

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  private int num = 0;

  public static int test(HelloWorld instance, int val) {
    return instance.num = val;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  private int num;
...

  public static int test(sample.HelloWorld, int);
    Code:
       0: aload_0
       1: iload_1
       2: dup_x1
       3: putfield      #2                  // Field num:I
       6: ireturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitInsn(DUP_X1);
methodVisitor.visitFieldInsn(PUTFIELD, "sample/HelloWorld", "num", "I");
methodVisitor.visitInsn(IRETURN);
methodVisitor.visitMaxs(3, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {HelloWorld, int} | {}
0000: aload_0                  // {HelloWorld, int} | {HelloWorld}
0001: iload_1                  // {HelloWorld, int} | {HelloWorld, int}
0002: dup_x1                   // {HelloWorld, int} | {int, HelloWorld, int}
0003: putfield        #2       // {HelloWorld, int} | {int}
0006: ireturn                  // {} | {}
```

从JVM规范的角度来看，`dup_x1`指令对应的Operand Stack的变化如下：

```pseudocode
..., value2, value1 →

..., value1, value2, value1
```

Duplicate the top value on the operand stack and insert the duplicated `value` two values down in the operand stack.

**The `dup_x1` instruction must not be used unless both `value1` and `value2` are values of a category 1 computational type**.

#### dup_x2

> 复制栈顶一个位置数据，并且在原栈顶再往下两个间隔的位置插入该数据

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public static int test(int[] array, int i, int value) {
    return array[i] = value;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public static int test(int[], int, int);
    Code:
       0: aload_0
       1: iload_1
       2: iload_2
       3: dup_x2
       4: iastore
       5: ireturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(ILOAD, 2);
methodVisitor.visitInsn(DUP_X2);
methodVisitor.visitInsn(IASTORE);
methodVisitor.visitInsn(IRETURN);
methodVisitor.visitMaxs(4, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {[I, int, int} | {}
0000: aload_0                  // {[I, int, int} | {[I}
0001: iload_1                  // {[I, int, int} | {[I, int}
0002: iload_2                  // {[I, int, int} | {[I, int, int}
0003: dup_x2                   // {[I, int, int} | {int, [I, int, int}
0004: iastore                  // {[I, int, int} | {int}
0005: ireturn                  // {} | {}
```

从JVM规范的角度来看，`dup_x2`指令对应的Operand Stack的变化如下：

Form 1:

```
..., value3, value2, value1 →

..., value1, value3, value2, value1
```

where `value1`, `value2`, and `value3` are all values of a category 1 computational type.

Form 2:

```
..., value2, value1 →

..., value1, value2, value1
```

where `value1` is a value of a category 1 computational type and `value2` is a value of a category 2 computational type.

### 2.10.4 dup2

#### dup2

> 复制栈顶两个位置数据

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    long a;
    long b;
    b = a = 2;
  }
}
```

```java
public class HelloWorld {
  public void test() {
    int[] array = new int[5];
    array[0]++; // 这里也会生成dup2指令
  }
}
/*
test:()V
                               // {this} | {}
0000: iconst_5                 // {this} | {int}
0001: newarray   10 (int)      // {this} | {[I}
0003: astore_1                 // {this, [I} | {}
0004: aload_1                  // {this, [I} | {[I}
0005: iconst_0                 // {this, [I} | {[I, int}
0006: dup2                     // {this, [I} | {[I, int, [I, int}
0007: iaload                   // {this, [I} | {[I, int, int}
0008: iconst_1                 // {this, [I} | {[I, int, int, int}
0009: iadd                     // {this, [I} | {[I, int, int}
0010: iastore                  // {this, [I} | {}
0011: return                   // {} | {}
*/
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc2_w        #2                  // long 2l
       3: dup2
       4: lstore_1
       5: lstore_3
       6: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitLdcInsn(new Long(2L));
methodVisitor.visitInsn(DUP2);
methodVisitor.visitVarInsn(LSTORE, 1);
methodVisitor.visitVarInsn(LSTORE, 3);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(4, 5);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: ldc2_w          #2       // {this} | {long, top}
0003: dup2                     // {this} | {long, top, long, top}
0004: lstore_1                 // {this, long, top} | {long, top}
0005: lstore_3                 // {this, long, top, long, top} | {}
0006: return                   // {} | {}
```

从JVM规范的角度来看，`dup2`指令对应的Operand Stack的变化如下：

Form 1:

```pseudocode
..., value2, value1 →

..., value2, value1, value2, value1
```

where both `value1` and `value2` are values of a category 1 computational type.

Form 2:

```pseudocode
..., value →

..., value, value
```

where `value` is a value of a category 2 computational type.

#### dup2_x1

> 复制栈顶2位置数据，并且在原栈顶位置往下间隔1个位置后再插入

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  private long num = 0;

  public static long test(HelloWorld instance, long val) {
    return instance.num = val;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
  private long num;
...

  public static long test(sample.HelloWorld, long);
    Code:
       0: aload_0
       1: lload_1
       2: dup2_x1
       3: putfield      #2                  // Field num:J
       6: lreturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitVarInsn(LLOAD, 1);
methodVisitor.visitInsn(DUP2_X1);
methodVisitor.visitFieldInsn(PUTFIELD, "sample/HelloWorld", "num", "J");
methodVisitor.visitInsn(LRETURN);
methodVisitor.visitMaxs(5, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {HelloWorld, long, top} | {}
0000: aload_0                  // {HelloWorld, long, top} | {HelloWorld}
0001: lload_1                  // {HelloWorld, long, top} | {HelloWorld, long, top}
0002: dup2_x1                  // {HelloWorld, long, top} | {long, top, HelloWorld, long, top}
0003: putfield        #2       // {HelloWorld, long, top} | {long, top}
0006: lreturn                  // {} | {}
```

从JVM规范的角度来看，`dup2_x1`指令对应的Operand Stack的变化如下：

Form 1:

```pseudocode
..., value3, value2, value1 →

..., value2, value1, value3, value2, value1
```

where `value1`, `value2`, and `value3` are all values of a category 1 computational type.

Form 2:

```pseudocode
..., value2, value1 →

..., value1, value2, value1
```

where `value1` is a value of a category 2 computational type and `value2` is a value of a category 1 computational type.

#### dup2_x2

> 复制栈顶2个位置数据，并且原栈顶位置向下2个位置间隔后再插入

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public static long test(long[] array, int i, long value) {
    return array[i] = value;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public static long test(long[], int, long);
    Code:
       0: aload_0
       1: iload_1
       2: lload_2
       3: dup2_x2
       4: lastore
       5: lreturn
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitVarInsn(ALOAD, 0);
methodVisitor.visitVarInsn(ILOAD, 1);
methodVisitor.visitVarInsn(LLOAD, 2);
methodVisitor.visitInsn(DUP2_X2);
methodVisitor.visitInsn(LASTORE);
methodVisitor.visitInsn(LRETURN);
methodVisitor.visitMaxs(6, 4);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {[J, int, long, top} | {}
0000: aload_0                  // {[J, int, long, top} | {[J}
0001: iload_1                  // {[J, int, long, top} | {[J, int}
0002: lload_2                  // {[J, int, long, top} | {[J, int, long, top}
0003: dup2_x2                  // {[J, int, long, top} | {long, top, [J, int, long, top}
0004: lastore                  // {[J, int, long, top} | {long, top}
0005: lreturn                  // {} | {}
```

从JVM规范的角度来看，`dup2_x2`指令对应的Operand Stack的变化如下：

Form 1:

```pseudocode
..., value4, value3, value2, value1 →

..., value2, value1, value4, value3, value2, value1
```

where `value1`, `value2`, `value3`, and `value4` are all values of a category 1 computational type.

Form 2:

```pseudocode
..., value3, value2, value1 →

..., value1, value3, value2, value1
```

where `value1` is a value of a category 2 computational type and `value2` and `value3` are both values of a category 1 computational type.

Form 3:

```pseudocode
..., value3, value2, value1 →

..., value2, value1, value3, value2, value1
```

where `value1` and `value2` are both values of a category 1 computational type and `value3` is a value of a category 2 computational type.

Form 4:

```pseudocode
..., value2, value1 →

..., value1, value2, value1
```

where `value1` and `value2` are both values of a category 2 computational type.

### 2.10.5 swap

> 交换栈顶2个位置的数据，这里讲师没有找到和swap相关的代码，所以通过ASM生成代码

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    System.out.println("Hello ASM");
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String Hello ASM
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("Hello ASM");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: getstatic       #2       // {this} | {PrintStream}
0003: ldc             #3       // {this} | {PrintStream, String}
0005: invokevirtual   #4       // {this} | {}
0008: return                   // {} | {}
```

为了使用`swap`指令，我们编写如下ASM代码：

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
      mv2.visitLdcInsn("Hello ASM");
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitInsn(SWAP);
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

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: ldc           #11                 // String Hello ASM
       2: getstatic     #17                 // Field java/lang/System.out:Ljava/io/PrintStream;
       5: swap
       6: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       9: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: ldc             #11      // {this} | {String}
0002: getstatic       #17      // {this} | {String, PrintStream}
0005: swap                     // {this} | {PrintStream, String}
0006: invokevirtual   #23      // {this} | {}
0009: return                   // {} | {}
```

从JVM规范的角度来看，`swap`指令对应的Operand Stack的变化如下：

```pseudocode
..., value2, value1 →

..., value1, value2
```

Swap the top two values on the operand stack.

The `swap` instruction must not be used unless `value1` and `value2` are both values of a category 1 computational type.

## 2.11 opcode: wide (1/195/205)

> [Chapter 6. The Java Virtual Machine Instruction Set (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.wide) wide本身不能直接使用，而是需要配合下一个指令使用
>
> The *wide* instruction modifies the behavior of another instruction. It takes one of two formats, depending on the instruction being modified. The first form of the *wide* instruction modifies one of the instructions *iload*, *fload*, *aload*, *lload*, *dload*, *istore*, *fstore*, *astore*, *lstore*, *dstore*, or *ret* ([§*iload*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.iload), [§*fload*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.fload), [§*aload*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.aload), [§*lload*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.lload), [§*dload*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dload), [§*istore*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.istore), [§*fstore*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.fstore), [§*astore*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.astore), [§*lstore*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.lstore), [§*dstore*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.dstore), [§*ret*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.ret)). The second form applies only to the *iinc* instruction ([§*iinc*](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.iinc)).
>
> In either case, the *wide* opcode itself is followed in the compiled code by the opcode of the instruction *wide* modifies. In either form, two unsigned bytes *indexbyte1* and *indexbyte2* follow the modified opcode and are assembled into a 16-bit unsigned index to a local variable in the current frame ([§2.6](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-2.html#jvms-2.6)), where the value of the index is (*indexbyte1* `<<` 8) | *indexbyte2*. The calculated index must be an index into the local variable array of the current frame. Where the *wide* instruction modifies an *lload*, *dload*, *lstore*, or *dstore* instruction, the index following the calculated index (index + 1) must also be an index into the local variable array. In the second form, two immediate unsigned bytes *constbyte1* and *constbyte2* follow *indexbyte1* and *indexbyte2* in the code stream. Those bytes are also assembled into a signed 16-bit constant, where the constant is (*constbyte1* `<<` 8) | *constbyte2*.
>
> The widened bytecode operates as normal, except for the use of the wider index and, in the case of the second form, the larger increment range.

### 2.11.1 概览

> + wide主要作用即让下一个紧接着的指令原本接收的参数的字节数翻倍，即原本接收一个字节参数变成接收两个字节参数
> + wide本身不影响Frame内的operand stack
>
> + ASM代码中不需要传递wide，ASM在检测到代码需要使用wide指令时会自动补上

从Instruction的角度来说，与wide相关的opcode有1个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 196    | wide            |        |                 |        |                 |        |                 |

从ASM的角度来说，这`wide`指令并没有与特定的`MethodVisitor.visitXxxInsn()`方法对应。

The `wide` instruction modifies the behavior of another instruction. It takes one of two formats, depending on the instruction being modified.

The first form of the `wide` instruction modifies one of the instructions `iload`, `fload`, `aload`, `lload`, `dload`, `istore`, `fstore`, `astore`, `lstore`, `dstore`, or `ret`.（ret是已经废弃的指令，这里可以忽略）

```pseudocode
wide
<opcode>
indexbyte1
indexbyte2
```

where `<opcode>` is one of `iload`, `fload`, `aload`, `lload`, `dload`, `istore`, `fstore`, `astore`, `lstore`, `dstore`, or `ret`.

The second form of the `wide` instruction applies only to the `iinc` instruction.

```pseudocode
wide
iinc
indexbyte1
indexbyte2
constbyte1
constbyte2
```

另外，与`wide`相关的opcode有`ldc_w`、`ldc2_w`、`goto_w`和`jsr_w`（已经弃用）。

### 2.11.2 wide: istore

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    int var001 = 1;
    // ... ... 省略
    int var257 = 257;
  }

  public static void main(String[] args) {
    String format = "int var%03d = %d;";
    for (int i = 1; i < 258; i++) {
      String line = String.format(format, i, i);
      System.out.println(line);
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       ...
    1140: sipush        256
    1143: istore_w      256 // 实际不存在 istore_w 这个指令，而是 wide 紧接着 istore，只是javap以这种形式展示而已
    1147: sipush        257
    1150: istore_w      257
    1154: return

...
}
```

但是，`istore_w`并不是真实存在的opcode，它其实是`wide`和`istore`指令的组合之后的结果，如下所示：

```shell
=== === ===  === === ===  === === ===
0000: iconst_1             // 04
0001: istore_1             // 3C
0002: iconst_2             // 05
0003: istore_2             // 3D
...
1140: sipush          256  // 110100
1143: wide                 // C4
1144: istore          256  // 360100
1147: sipush          257  // 110101
1150: wide                 // C4
1151: istore          257  // 360101
1154: return               // B1
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitVarInsn(ISTORE, 1);
methodVisitor.visitInsn(ICONST_2);
methodVisitor.visitVarInsn(ISTORE, 2);
...
methodVisitor.visitIntInsn(SIPUSH, 255);
methodVisitor.visitVarInsn(ISTORE, 255);
methodVisitor.visitIntInsn(SIPUSH, 256); 
methodVisitor.visitVarInsn(ISTORE, 256); // istore 原本只能0～255索引，需要加上wide才能索引超过255的位置，这里ASM自动补齐wide指令
methodVisitor.visitIntInsn(SIPUSH, 257);
methodVisitor.visitVarInsn(ISTORE, 257);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(1, 258);
methodVisitor.visitEnd();
```

### 2.11.3 wide: iinc

当使用`iinc`时，如果local variable值的变化范围大于`127`或者小于`-128`，就会生成`wide`指令。（同样的，ASM自动补齐）

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int val) {
    val += 128;
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int);
    Code:
       0: iinc_w        1, 128 // 同样并不存在 iinc_w 指令，实际是 wide 紧接着 iinc
       6: return
}
```

但是，`iinc_w`并不是真实存在的opcode，它其实是`wide`和`iinc`指令的组合之后的结果，如下所示：

```shell
=== === ===  === === ===  === === ===
0000: wide                 // C4
0001: iinc       1    128  // 8400010080
0006: return               // B1
=== === ===  === === ===  === === ===
LocalVariableTable:
index  start_pc  length  name_and_type
    0         0       7  this:Lsample/HelloWorld;
    1         0       7  val:I
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitIincInsn(1, 128); // ASM自动会补齐wide指令
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(0, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this, int} | {}
0000: wide                
0001: iinc       1    128      // {this, int} | {}
0006: return                   // {} | {}
```

## 2.12 opcode: exception (1/196/205)

### 2.12.1 概览

> 注意： catch并没有单独的指令，只有抛出异常有
>
> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html#jvms-4.7.3)
>
> 4.7.3. The Code Attribute 回顾一下，异常处理逻辑的判断和跳转逻辑在 exception_table 中
>
> ```java
> Code_attribute {
>     u2 attribute_name_index;
>     u4 attribute_length;
>     u2 max_stack;
>     u2 max_locals;
>     u4 code_length;
>     u1 code[code_length];
>     u2 exception_table_length;
>     {   u2 start_pc;
>         u2 end_pc;
>         u2 handler_pc;
>         u2 catch_type;
>     } exception_table[exception_table_length];
>     u2 attributes_count;
>     attribute_info attributes[attributes_count];
> }
> ```

从Instruction的角度来说，与exception相关的opcode有1个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 191    | athrow          |        |                 |        |                 |        |                 |

从ASM的角度来说，这个`athrow`指令与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`: `athrow`

### 2.12.2 throw exception

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    throw new RuntimeException();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class java/lang/RuntimeException
       3: dup
       4: invokespecial #3                  // Method java/lang/RuntimeException."<init>":()V
       7: athrow // 抛出异常RuntimeException
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
methodVisitor.visitCode();
methodVisitor.visitTypeInsn(NEW, "java/lang/RuntimeException");
methodVisitor.visitInsn(DUP);
methodVisitor.visitMethodInsn(INVOKESPECIAL, "java/lang/RuntimeException", "<init>", "()V", false);
methodVisitor.visitInsn(ATHROW);
methodVisitor.visitMaxs(2, 1);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_RuntimeException}
0003: dup                      // {this} | {uninitialized_RuntimeException, uninitialized_RuntimeException}
0004: invokespecial   #3       // {this} | {RuntimeException}
0007: athrow                   // {} | {} // 消耗栈顶的 异常指针
```

从JVM规范的角度来看，`athrow`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref →

objectref
```

> 下面简言之，即抛出异常后，如果当前Frame内能找到对应的handle处理，那么pc程序计数器跳转到handle代码执行逻辑；
>
> 如果当前Frame找不到handle，则当前Frame出栈，前一个Frame内继续查找handle处理，如果一直没有，重复上抛过程，直到当前thread线程无Frame，则线程exit退出

The `objectref` must be of type `reference` and must refer to an object that is an instance of class `Throwable` or of a subclass of `Throwable`. It is popped from the operand stack. The `objectref` is then thrown by searching the current method for the first exception handler that matches the class of `objectref`.

- If an exception handler that matches `objectref` is found, it contains the location of the code intended to handle this exception. The `pc` register is reset to that location, the operand stack of the current frame is cleared, `objectref` is pushed back onto the operand stack, and execution continues.
- If no matching exception handler is found in the current frame, that frame is popped. Finally, the frame of its invoker is reinstated, if such a frame exists, and the `objectref` is rethrown. If no such frame exists, the current thread exits.

### 2.12.3 catch exception

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    try {
      int val = 1 / 0;
    }
    catch (ArithmeticException e1) {
      System.out.println("catch ArithmeticException");
    }
    catch (Exception e2) {
      System.out.println("catch Exception");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: iconst_1
       1: iconst_0
       2: idiv
       3: istore_1
       4: goto          28
       7: astore_1
       8: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: ldc           #4                  // String catch ArithmeticException
      13: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      16: goto          28
      19: astore_1
      20: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: ldc           #7                  // String catch Exception
      25: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      28: return
    Exception table: // 可以看到 catch 对应的handle跳转记录在 exception table中，即[0,4)代码执行时如果出现 ArithmeticException 异常则跳转到7执行，遇到Exception则跳转到19执行。这里7和19执行astore_1是为了存储异常指针
       from    to  target type
           0     4     7   Class java/lang/ArithmeticException
           0     4    19   Class java/lang/Exception
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();
Label label2 = new Label();
Label label3 = new Label();
Label label4 = new Label();

methodVisitor.visitCode();
methodVisitor.visitTryCatchBlock(label0, label1, label2, "java/lang/ArithmeticException");
methodVisitor.visitTryCatchBlock(label0, label1, label3, "java/lang/Exception");

methodVisitor.visitLabel(label0);
methodVisitor.visitInsn(ICONST_1);
methodVisitor.visitInsn(ICONST_0);
methodVisitor.visitInsn(IDIV);
methodVisitor.visitVarInsn(ISTORE, 1);

methodVisitor.visitLabel(label1);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label2);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("catch ArithmeticException");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label3);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("catch Exception");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

methodVisitor.visitLabel(label4);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 2);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```shell
                               // {this} | {}
0000: iconst_1                 // {this} | {int}
0001: iconst_0                 // {this} | {int, int}
0002: idiv                     // {this} | {int}
0003: istore_1                 // {this, int} | {}
0004: goto            24       // {} | {}
                               // {this} | {ArithmeticException}
0007: astore_1                 // {this, ArithmeticException} | {} // 可以看到如果跳转到这里，则存储 ArithmeticException 异常信息
0008: getstatic       #3       // {this, ArithmeticException} | {PrintStream}
0011: ldc             #4       // {this, ArithmeticException} | {PrintStream, String}
0013: invokevirtual   #5       // {this, ArithmeticException} | {}
0016: goto            12       // {} | {}
                               // {this} | {Exception}
0019: astore_1                 // {this, Exception} | {} // 可以看到如果跳转到这里，则存储 Exception 异常信息
0020: getstatic       #3       // {this, Exception} | {PrintStream}
0023: ldc             #7       // {this, Exception} | {PrintStream, String}
0025: invokevirtual   #5       // {this, Exception} | {}
                               // {this} | {}
0028: return                   // {} | {}
```

## 2.13 opcode: monitor (2/198/205)

### 2.13.1 概览

> + monitor指令对应加锁、解锁操作
>
> + 每一个Heap中的java对象关联一个monitor，而该monitor同时只能被一个thread占有
>
> + thread可以反复monitorenter，使monitor的计数从0一直不断+1
>
> + 只有当monitorexit执行多次直到monitor的计数恢复到0之后，该thread就不再是monitor的拥有者

从Instruction的角度来说，与monitor相关的opcode有2个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 194    | monitorenter    | 195    | monitorexit     |        |                 |        |                 |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`: `monitorenter`, `monitorexit`

### 2.13.2 示例

从Java语言的视角，有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    synchronized (System.out) {
      System.out.println("Hello World");
    }
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: dup
       4: astore_1
       5: monitorenter
       6: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #3                  // String Hello World
      11: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: aload_1
      15: monitorexit
      16: goto          24
      19: astore_2 // 将[6,16)发生的异常存储到local variables的下标2位置
      20: aload_1
      21: monitorexit
      22: aload_2
      23: athrow
      24: return
    Exception table:
       from    to  target type
           6    16    19   any // any表示捕获所有类型的异常,这里[6,16)出现任何异常就会跳转到19，这里21是避免已经monitorenter了，结果代码出错没monitorexit，所以这里再执行一下monitorexit
          19    22    19   any // [19,22)，即如果捕获异常后monitorexit生效前还是出现异常，就跳转回19，一直重复直到确保monitor释放
}
```

从ASM的视角来看，方法体对应的内容如下：

```java
Label label0 = new Label();
Label label1 = new Label();
Label label2 = new Label();
Label label3 = new Label();
Label label4 = new Label();

methodVisitor.visitCode();
methodVisitor.visitTryCatchBlock(label0, label1, label2, null); // 这里传递null表示捕获所有类型的异常
methodVisitor.visitTryCatchBlock(label2, label3, label2, null);

methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitInsn(DUP);
methodVisitor.visitVarInsn(ASTORE, 1);
methodVisitor.visitInsn(MONITORENTER);

methodVisitor.visitLabel(label0);
methodVisitor.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
methodVisitor.visitLdcInsn("Hello World");
methodVisitor.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(MONITOREXIT);

methodVisitor.visitLabel(label1);
methodVisitor.visitJumpInsn(GOTO, label4);

methodVisitor.visitLabel(label2);
methodVisitor.visitVarInsn(ASTORE, 2);
methodVisitor.visitVarInsn(ALOAD, 1);
methodVisitor.visitInsn(MONITOREXIT);

methodVisitor.visitLabel(label3);
methodVisitor.visitVarInsn(ALOAD, 2);
methodVisitor.visitInsn(ATHROW);

methodVisitor.visitLabel(label4);
methodVisitor.visitInsn(RETURN);
methodVisitor.visitMaxs(2, 3);
methodVisitor.visitEnd();
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: getstatic       #2       // {this} | {PrintStream}
0003: dup                      // {this} | {PrintStream, PrintStream}
0004: astore_1                 // {this, PrintStream} | {PrintStream}
0005: monitorenter             // {this, PrintStream} | {} // 消耗栈顶对象指针,将当前thread和该对象的monitor关联起来
0006: getstatic       #2       // {this, PrintStream} | {PrintStream}
0009: ldc             #3       // {this, PrintStream} | {PrintStream, String}
0011: invokevirtual   #4       // {this, PrintStream} | {}
0014: aload_1                  // {this, PrintStream} | {PrintStream}
0015: monitorexit              // {this, PrintStream} | {}
0016: goto            8        // {} | {}
                               // {this, Object} | {Throwable}
0019: astore_2                 // {this, Object, Throwable} | {}
0020: aload_1                  // {this, Object, Throwable} | {Object}
0021: monitorexit              // {this, Object, Throwable} | {} // monitorexit 需要消耗 栈顶对象指针
0022: aload_2                  // {this, Object, Throwable} | {Throwable}
0023: athrow                   // {} | {}
                               // {this} | {}
0024: return                   // {} | {}
```

从JVM规范的角度来看，`monitorenter`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref →

...
```

> 下面简言之，每个对象都有关联一个monitor，同一时间只能有一个owner（线程thread）能lock这个monitor：
>
> + monitor起初计数0，每次thread执行完monitorenter后，计数+1
> + 同一个owner重复monitorenter，则计数每次进入时+1
> + 当且仅当monitor的count计数为0时，其他线程才能有机会成为monitor的下一个owner

The `objectref` must be of type `reference`.

**Each object is associated with a monitor.** A monitor is locked if and only if it has an owner. The thread that executes `monitorenter` attempts to gain ownership of the monitor associated with `objectref`, as follows:

- If the entry count of the monitor associated with `objectref` is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
- If the thread already owns the monitor associated with `objectref`, it reenters the monitor, incrementing its entry count.
- If another thread already owns the monitor associated with `objectref`, the thread blocks until the monitor’s entry count is zero, then tries again to gain ownership.

从JVM规范的角度来看，`monitorexit`指令对应的Operand Stack的变化如下：

```pseudocode
..., objectref →

...
```

> 简言之，只有owner线程能够执行monitorexit使关联对象的monitor的计数在`monitorexit`之后-1

The `objectref` must be of type `reference`.

**The thread that executes `monitorexit` must be the owner of the monitor associated with the instance referenced by `objectref`.**

The thread decrements the entry count of the monitor associated with `objectref`.

- If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner.
- Other threads that are blocking to enter the monitor are allowed to attempt to do so.

## 2.14 opcode: unused (7/205/205)

### 2.14.1 概览

> + nop本身不产生任何作用，正常java代码不会生成nop指令（如果用ASM编写不符合规范的`.class`，会生成nop，比如在`visitMax()`方法结束后仍然执行逻辑，就会被替换成nop指令）
> + jsr、ret、jsr_w在java7之后废弃无用
> + breakpoint、impdep1、impdep2是java保留的指令，本身不允许编译器生成，只能JVM自己使用

从Instruction的角度来说，余下的opcode有7个，内容如下：

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 0      | nop             |        |                 |        |                 |        |                 |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 168    | jsr             | 169    | ret             | 201    | jsr_w           |        |                 |

| opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol | opcode | mnemonic symbol |
| ------ | --------------- | ------ | --------------- | ------ | --------------- | ------ | --------------- |
| 202    | breakpoint      |        |                 | 254    | impdep1         | 255    | impdep2         |

从ASM的角度来说，这些opcode与`MethodVisitor.visitXxxInsn()`方法对应关系如下：

- `MethodVisitor.visitInsn()`: `nop`
- `MethodVisitor.visitJumpInsn()`: `jsr`, `jsr_w`
- `MethodVisitor.visitVarInsn()`: `ret`

### 2.14.2 nop

Do nothing

使用ASM编写一段包含`nop`指向的代码如下：

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
      mv2.visitInsn(NOP);
      mv2.visitInsn(NOP);
      mv2.visitInsn(NOP);
      mv2.visitInsn(NOP);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(0, 1);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c -p sample.HelloWorld
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: nop
       1: nop
       2: nop
       3: nop
       4: nop
       5: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: nop                      // {this} | {}
0001: nop                      // {this} | {}
0002: nop                      // {this} | {}
0003: nop                      // {this} | {}
0004: nop                      // {this} | {}
0005: return                   // {} | {}
```

从JVM规范的角度来看，`nop`指令对应的Operand Stack的变化如下：

```pseudocode
No change
```

### 2.14.3 Deprecated Opcodes

**If the class file version number is 51.0(Java 7) or above, then neither the `jsr` opcode or the `jsr_w` opcode may appear in the `code` array.**

- jsr
- jsr_w
- ret

```java
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length]; // code array
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

### 2.14.4 Reserved Opcodes

Three opcodes are reserved for internal use by a Java Virtual Machine implementation. If the instruction set of the Java Virtual Machine is extended in the future, these reserved opcodes are guaranteed not to be used.

Two of the reserved opcodes, numbers `254` (`0xfe`) and `255` (`0xff`), have the mnemonics `impdep1` and `impdep2`, respectively. These instructions are intended to provide “back doors” or traps to implementation-specific functionality implemented in software and hardware, respectively. The third reserved opcode, number `202` (`0xca`), has the mnemonic `breakpoint` and is intended to be used by debuggers to implement breakpoints.

Although these opcodes have been reserved, they may be used only inside a Java Virtual Machine implementation. They cannot appear in valid class files.

# 3. 难点解析

## 3.1 调用方法

### 3.1.1 细微的差异

在本文当中，主要是介绍一下`.java`文件和`.class`文件中方法调用的差异。

在`.java`文件中，一般的方法由三部分组成，其书写步骤如下：

- 第一步，写对象的变量名
- 第二步，写方法的名字
- 第三步，写方法的参数

![Java ASM系列：（064）调用方法_ClassFile](https://s2.51cto.com/images/20210828/1630130358776142.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在`.class`文件中，一般方法也是由三部分组成，但其指令执行顺序有所不同：

- 第一步，加载对象
- 第二步，加载方法的参数
- 第三步，对方法进行调用 （第一步和第二步，就是在准备数据，此时，万事俱备，只差“方法调用”了）

![img](https://s2.51cto.com/images/20210828/1630130313596345.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp)

简单来说，两者的差异如下：

- Java文件：instance –> method –> parameters
- Class文件: instance –> parameters –> method

### 3.1.2 举例说明

假如有一个`GoodChild`类，其代码如下：

```java
public class GoodChild {
  private String name;
  private int age;

  public GoodChild(String name, int age) {
    this.name = name;
    this.age = age;
  }

  public void study(String subject, int minutes) {
    String str = String.format("%s who is %d years old has studied %s for %d minutes", name, age, subject, minutes);
    System.out.println(str);
  }

  @Override
  public String toString() {
    return String.format("GoodChild{name='%s', age=%d}", name, age);
  }
}
```

假如有一个`HelloWorld`类，其代码如下：

```java
public class HelloWorld {
  public void test() {
    GoodChild child = new GoodChild("Lucy", 8);
    child.study("Math", 30);
  }
}
```

假如有一个`HelloWorldRun`类，其代码如下：

```java
import sample.HelloWorld;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test();
  }
}
```

对于上面的三个类，我们只关注其中的`HelloWorld`类。通过`javap -c sample.HelloWorld`命令，我们可以查看其`test()`方法对应的instructions内容：

```shell
$  javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class sample/GoodChild
       3: dup
       4: ldc           #3                  // String Lucy
       6: bipush        8
       8: invokespecial #4                  // Method sample/GoodChild."<init>":(Ljava/lang/String;I)V
      11: astore_1
      12: aload_1
      13: ldc           #5                  // String Math
      15: bipush        30
      17: invokevirtual #6                  // Method sample/GoodChild.study:(Ljava/lang/String;I)V
      20: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_GoodChild}
0003: dup                      // {this} | {uninitialized_GoodChild, uninitialized_GoodChild}
0004: ldc             #3       // {this} | {uninitialized_GoodChild, uninitialized_GoodChild, String}
0006: bipush          8        // {this} | {uninitialized_GoodChild, uninitialized_GoodChild, String, int}
0008: invokespecial   #4       // {this} | {GoodChild}
0011: astore_1                 // {this, GoodChild} | {}
0012: aload_1                  // {this, GoodChild} | {GoodChild}
0013: ldc             #5       // {this, GoodChild} | {GoodChild, String}
0015: bipush          30       // {this, GoodChild} | {GoodChild, String, int}
0017: invokevirtual   #6       // {this, GoodChild} | {}
0020: return                   // {} | {}
```

## 3.2 创建对象

在字节码层面，创建对象，会用到`new/dup/invokespecial`指令的集合。本文主要介绍三个问题：

- 第一个问题：为什么要使用dup指令呢？
- 第二个问题：是否可以将dup指令替换成别的指令（`astore`）呢？
- 第三个问题：是否可以打印未初始化的对象？

### 3.2.1 为什么要用dup指令

假如有一个`GoodChild`类，代码如下：

```java
package sample;

public class GoodChild {
  private String name;
  private int age;

  public GoodChild(String name, int age) {
    this.name = name;
    this.age = age;
  }

  @Override
  public String toString() {
    return String.format("GoodChild{name='%s', age=%d}", name, age);
  }
}
```

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test() {
    GoodChild child = new GoodChild("tom", 10);
    System.out.println(child);
  }
}
```

假如有一个`HelloWorldRun`类，其代码如下：

```java
import sample.HelloWorld;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$  javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #2                  // class sample/GoodChild
       3: dup
       4: ldc           #3                  // String tom
       6: bipush        10
       8: invokespecial #4                  // Method sample/GoodChild."<init>":(Ljava/lang/String;I)V
      11: astore_1
      12: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      15: aload_1
      16: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      19: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

> dup是为了保留初始化后对象的指针，不然Heap堆中对象初始化了，结果没有指针指向该对象，后面岂不是又直接被垃圾回收了

```pseudocode
                               // {this} | {}
0000: new             #2       // {this} | {uninitialized_GoodChild}
0003: dup                      // {this} | {uninitialized_GoodChild, uninitialized_GoodChild}
0004: ldc             #3       // {this} | {uninitialized_GoodChild, uninitialized_GoodChild, String}
0006: bipush          10       // {this} | {uninitialized_GoodChild, uninitialized_GoodChild, String, int}
0008: invokespecial   #4       // {this} | {GoodChild} // 消耗栈顶的对象指针和构造方法所需要的参数，对Heap中对象进行初始化
0011: astore_1                 // {this, GoodChild} | {}
0012: getstatic       #5       // {this, GoodChild} | {PrintStream}
0015: aload_1                  // {this, GoodChild} | {PrintStream, GoodChild}
0016: invokevirtual   #6       // {this, GoodChild} | {}
0019: return                   // {} | {}
```

### 3.2.2 使用astore替换dup指令

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
      mv2.visitTypeInsn(NEW, "sample/GoodChild");
      mv2.visitVarInsn(ASTORE, 1);
      mv2.visitVarInsn(ALOAD, 1);
      mv2.visitLdcInsn("tom");
      mv2.visitIntInsn(BIPUSH, 10);
      mv2.visitMethodInsn(INVOKESPECIAL, "sample/GoodChild", "<init>", "(Ljava/lang/String;I)V", false);

      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ALOAD, 1);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(4, 2);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

从Instruction的视角来看，方法体对应的内容如下：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test();
    Code:
       0: new           #11                 // class sample/GoodChild
       3: astore_1
       4: aload_1
       5: ldc           #13                 // String tom
       7: bipush        10
       9: invokespecial #16                 // Method sample/GoodChild."<init>":(Ljava/lang/String;I)V
      12: getstatic     #22                 // Field java/lang/System.out:Ljava/io/PrintStream;
      15: aload_1
      16: invokevirtual #28                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      19: return
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #11      // {this} | {uninitialized_GoodChild}
0003: astore_1                 // {this, uninitialized_GoodChild} | {}
0004: aload_1                  // {this, uninitialized_GoodChild} | {uninitialized_GoodChild}
0005: ldc             #13      // {this, uninitialized_GoodChild} | {uninitialized_GoodChild, String}
0007: bipush          10       // {this, uninitialized_GoodChild} | {uninitialized_GoodChild, String, int}
0009: invokespecial   #16      // {this, GoodChild} | {}
0012: getstatic       #22      // {this, GoodChild} | {PrintStream}
0015: aload_1                  // {this, GoodChild} | {PrintStream, GoodChild}
0016: invokevirtual   #28      // {this, GoodChild} | {}
0019: return                   // {} | {}
```

### 3.2.3 打印未初始化对象

#### 生成instructions

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
      mv2.visitTypeInsn(NEW, "sample/GoodChild");
      mv2.visitVarInsn(ASTORE, 1);

      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ALOAD, 1);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(4, 2);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

从Frame的视角来看，local variable和operand stack的变化：

```pseudocode
                               // {this} | {}
0000: new             #11      // {this} | {uninitialized_GoodChild}
0003: astore_1                 // {this, uninitialized_GoodChild} | {}
0004: getstatic       #17      // {this, uninitialized_GoodChild} | {PrintStream}
0007: aload_1                  // {this, uninitialized_GoodChild} | {PrintStream, uninitialized_GoodChild}
0008: invokevirtual   #23      // {this, uninitialized_GoodChild} | {}
0011: return                   // {} | {}
```

#### 执行结果（出现错误）

运行`HelloWorldRun`类之后，得到如下结果：

```shell
Exception in thread "main" java.lang.VerifyError: Bad type on operand stack
Exception Details:
  Location:
    sample/HelloWorld.test()V @8: invokevirtual
  Reason: // 这里println的参数要求Object，但是传入的是一个 uninitialized 未初始化对象的指针，不兼容，所以报错
    Type uninitialized 0 (current frame, stack[1]) is not assignable to 'java/lang/Object'
  Current Frame:
    bci: @8 // byte code index 8 位置报错
    flags: { }
    locals: { 'sample/HelloWorld', uninitialized 0 } // local variables 状态
    stack: { 'java/io/PrintStream', uninitialized 0 } // operand stack 状态
  Bytecode:
    0x0000000: bb00 0b4c b200 112b b600 17b1          

	at run.HelloWorldRun.main(HelloWorldRun.java:7)
```

#### 分析原因

我们可以使用`BytecodeRun`来还原instructions的内容：

```shell
0x0000000: bb00 0b4c b200 112b b600 17b1
==================================================
0000: new             #11 
0003: astore_1            
0004: getstatic       #17 
0007: aload_1             
0008: invokevirtual   #23 // 报错的位置
0011: return 
```

## 3.3 Exception处理

<u>当方法内部的instructions执行的时候，可能会遇到异常情况，那么JVM会将其封装成一个具体的Exception子类的对象，然后把这个“异常对象”放在operand stack上</u>，接下来要怎么处理呢？

对于operand stack上的“异常对象”，有两种处理情况：

- 第一种，当前方法处理不了，只能再交给别的方法进行处理
- 第二种，当前方法能够处理，就按照自己的处理逻辑来进行处理了

在本文当中，我们就只关注第二种情况，也就是当前方法能够这个“异常对象”。

在当前方法中，处理异常的机制，在`Code`属性结构当中，分成了两个部分进行存储：

- 第一部分，是位于`exception_table[]`内，它类似于中国古代打仗当中“军师”（运筹帷幄），它会告诉我们应该怎么处理这个“异常对象”。
- 第二部分，是位于`code[]`内，它类似于中国古代打仗当中的“将军和士兵”（决胜千里），它是具体的来执行对“异常对象”处理的操作。

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

### 3.3.1 异常处理的顺序

#### 代码举例

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(int a, int b) {
    try {
      int val = a / b;
      System.out.println(val);
    }
    catch (ArithmeticException ex1) {
      System.out.println("catch ArithmeticException");
    }
    catch (Exception ex2) {
      System.out.println("catch Exception");
    }
  }
}
```

接着，我们来写一个`HelloWorldRun`类，来对`HelloWorld.test()`方法进行调用。

```java
import sample.HelloWorld;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test(10, 0);
  }
}
```

我们来运行一下`HelloWorldRun`类，查看一下输出结果：

```
catch ArithmeticException
```

#### exception table结构

那么，异常处理的逻辑是存储在哪里呢？

从ClassFile的角度来看，异常处理的逻辑，是存在`Code`属性的`exception_table`部分。

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

接着，我们对`HelloWorld.class`文件进行分析，其`test()`方法的`Code`结构当中各个条目具体取值如下：

```java
Code_attribute {
    attribute_name_index='000D' (#13)
    attribute_length='000000C1' (193)
    max_stack='0002' (2)
    max_locals='0004' (4)
    code_length='00000024' (36)
    code: 1B1C6C3EB200021DB60003A700184EB200021205B60006A7000C4EB200021208B60006B1
    exception_table_length='0002' (2)        // 注意，这里的数量是2
    exception_table[0] {                     // 这里是第1个异常处理
        start_pc='0000' (0)
        end_pc='000B' (11)
        handler_pc='000E' (14)
        catch_type='0004' (#4)               // 这里表示捕获的异常是ArithmeticException
    }
    exception_table[1] {                     // 这里是第2个异常处理
        start_pc='0000' (0)
        end_pc='000B' (11)
        handler_pc='001A' (26)
        catch_type='0007' (#7)               // 这里表示捕获的异常是Exception类型
    }
    attributes_count='0003' (3)
    LineNumberTable: 000E00000026...
    LocalVariableTable: 000F0000003E...
    StackMapTable: 001C0000000B...
}
```

再接下来，我们结合着`javap`指令来理解一下exception table的含义：

```shell
$ javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: idiv
       3: istore_3
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_3
       8: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
      11: goto          35
      14: astore_3 // 将异常对象指针存储到 local variables 下标3位置(0 this指针，1和2对应方法参数)
      15: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #5                  // String catch ArithmeticException
      20: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: goto          35
      26: astore_3
      27: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      30: ldc           #8                  // String catch Exception
      32: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      35: return
    Exception table:
       from    to  target type
           0    11    14   Class java/lang/ArithmeticException // [0,11) 出现ArithmeticException异常，则跳转到14
           0    11    26   Class java/lang/Exception // [0,11) 出现Exception异常，则跳转到26
}
```

#### 调换exception table顺序

在这里，我们想调换一下exception table顺序：先捕获`Exception`异常，再捕获`ArithmeticException`异常。

我们想通过Java语言来实现调换一下exception table顺序，会发现代码会报错：

```java
public class HelloWorld {
  public void test(int a, int b) {
    try {
      int val = a / b;
      System.out.println(val);
    }
    catch (Exception ex2) {
      System.out.println("catch Exception");
    }
    catch (ArithmeticException ex1) { // Exception 'java.lang.ArithmeticException' has already been caught
      System.out.println("catch ArithmeticException");
    }
  }
}
```

那么，应该怎么做呢？我们就通过ASM来帮助我们实现调换一下exception table顺序。

首先，我们来看一下，原本`HelloWorld`类的代码如何通过ASM代码来进行实现：

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
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "(II)V", null, null);
      Label startLabel = new Label();
      Label endLabel = new Label();
      Label exceptionHandler01 = new Label();
      Label exceptionHandler02 = new Label();
      Label returnLabel = new Label();

      mv2.visitCode();
      mv2.visitTryCatchBlock(startLabel, endLabel, exceptionHandler01, "java/lang/ArithmeticException");
      mv2.visitTryCatchBlock(startLabel, endLabel, exceptionHandler02, "java/lang/Exception");

      mv2.visitLabel(startLabel);
      mv2.visitVarInsn(ILOAD, 1);
      mv2.visitVarInsn(ILOAD, 2);
      mv2.visitInsn(IDIV);
      mv2.visitVarInsn(ISTORE, 3);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ILOAD, 3);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(I)V", false);

      mv2.visitLabel(endLabel);
      mv2.visitJumpInsn(GOTO, returnLabel);

      mv2.visitLabel(exceptionHandler01);
      mv2.visitVarInsn(ASTORE, 3);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("catch ArithmeticException");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
      mv2.visitJumpInsn(GOTO, returnLabel);

      mv2.visitLabel(exceptionHandler02);
      mv2.visitVarInsn(ASTORE, 3);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitLdcInsn("catch Exception");
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

      mv2.visitLabel(returnLabel);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(2, 4);
      mv2.visitEnd();
    }
    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

接下来，我们调换一下上述ASM代码中两个`MethodVisitor.visitTryCatchBlock()`方法调用的顺序：

```java
mv2.visitTryCatchBlock(startLabel, endLabel, exceptionHandler02, "java/lang/Exception");
mv2.visitTryCatchBlock(startLabel, endLabel, exceptionHandler01, "java/lang/ArithmeticException");
```

那么，我们执行一下`HelloWorldGenerateCore`类，就可以生成`HelloWorld.class`文件。然后，我们执行一下`javap -c sample.HelloWorld`命令，来验证一下：

```shell
$  javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: idiv
       3: istore_3
       4: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
       7: iload_3
       8: invokevirtual #26                 // Method java/io/PrintStream.println:(I)V
      11: goto          35
      14: astore_3
      15: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      18: ldc           #28                 // String catch ArithmeticException
      20: invokevirtual #31                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      23: goto          35
      26: astore_3
      27: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      30: ldc           #33                 // String catch Exception
      32: invokevirtual #31                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      35: return
    Exception table:
       from    to  target type
           0    11    26   Class java/lang/Exception // 发现exception table中记录顺序变化了
           0    11    14   Class java/lang/ArithmeticException
}
```

再接着，我们来运行一下`HelloWorldRun`类，查看一下输出结果：

```shell
catch Exception
```

这样的一个输出结果，说明：**exception table就是根据先后顺序来判定的，而不是找到最匹配的异常类型**。

### 3.3.2 为整个方法添加异常处理的逻辑

#### 代码举例

假如有一个`HelloWorld`类，代码如下：

```java
public class HelloWorld {
  public void test(String name, int age) {
    try {
      int length = name.length();
      System.out.println("length = " + length);
    }
    catch (NullPointerException ex) {
      System.out.println("name is null");
    }

    int val = div(10, age);
    System.out.println("val = " + val);
  }

  public int div(int a, int b) {
    return a / b;
  }
}
```

我们想实现的预期目标：将整个`test()`方法添加一个try-catch语句。

首先，我们来运行一下`HelloWorldRun`类：

```java
import sample.HelloWorld;

public class HelloWorldRun {
  public static void main(String[] args) throws Exception {
    HelloWorld instance = new HelloWorld();
    instance.test(null, 0);
  }
}
```

其输出结果如下：

```shell
name is null
Exception in thread "main" java.lang.ArithmeticException: / by zero
	at sample.HelloWorld.div(HelloWorld.java:18)
	at sample.HelloWorld.test(HelloWorld.java:13)
	at run.HelloWorldRun.main(HelloWorldRun.java:8)
```

使用`javap`命令查看exception table的内容：

```shell
$  javap -c sample.HelloWorld
Compiled from "HelloWorld.java"
public class sample.HelloWorld {
...
  public void test(java.lang.String, int);
    Code:
       0: aload_1
       1: invokevirtual #2                  // Method java/lang/String.length:()I
       4: istore_3
       5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: new           #4                  // class java/lang/StringBuilder
      11: dup
      12: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
      15: ldc           #6                  // String length =
      17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: iload_3
      21: invokevirtual #8                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      24: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      27: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      30: goto          42
      33: astore_3
      34: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      37: ldc           #12                 // String name is null
      39: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: aload_0
      43: bipush        10
      45: iload_2
      46: invokevirtual #13                 // Method div:(II)I
      49: istore_3
      50: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      53: new           #4                  // class java/lang/StringBuilder
      56: dup
      57: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
      60: ldc           #14                 // String val =
      62: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      65: iload_3
      66: invokevirtual #8                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      69: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      72: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      75: return
    Exception table:
       from    to  target type
           0    30    33   Class java/lang/NullPointerException

...
}
```

#### 正确的异常捕获处理

```java
import org.objectweb.asm.*;

import static org.objectweb.asm.Opcodes.*;

public class MethodWithWholeTryCatchVisitor extends ClassVisitor {
  private final String methodName;
  private final String methodDesc;

  public MethodWithWholeTryCatchVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && methodName.equals(name) && methodDesc.equals(descriptor)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodWithWholeTryCatchAdapter(api, mv, access, descriptor);
      }
    }
    return mv;
  }

  private static class MethodWithWholeTryCatchAdapter extends MethodVisitor {
    private final int methodAccess;
    private final String methodDesc;

    private final Label startLabel = new Label();
    private final Label endLabel = new Label();
    private final Label handlerLabel = new Label();

    public MethodWithWholeTryCatchAdapter(int api, MethodVisitor methodVisitor, int methodAccess, String methodDesc) {
      super(api, methodVisitor);
      this.methodAccess = methodAccess;
      this.methodDesc = methodDesc;
    }

    public void visitCode() {
      // 首先，处理自己的代码逻辑
      // (1) startLabel
      super.visitLabel(startLabel);

      // 其次，调用父类的方法实现
      super.visitCode();
    }

    @Override
    public void visitMaxs(int maxStack, int maxLocals) {
      // 首先，处理自己的代码逻辑
      // (2) endLabel
      super.visitLabel(endLabel);

      // (3) handlerLabel
      super.visitLabel(handlerLabel);
      int localIndex = getLocalIndex(); // 计算获取当前local variables可存储异常指针的位置
      super.visitVarInsn(ASTORE, localIndex); // 存储异常对象指针
      super.visitVarInsn(ALOAD, localIndex); // 加载异常对象指针
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "(Ljava/io/PrintStream;)V", false);
      super.visitVarInsn(ALOAD, localIndex);
      super.visitInsn(Opcodes.ATHROW);

      // (4) visitTryCatchBlock
      super.visitTryCatchBlock(startLabel, endLabel, handlerLabel, "java/lang/Exception");

      // 其次，调用父类的方法实现
      super.visitMaxs(maxStack, maxLocals);
    }

    private int getLocalIndex() {
      Type t = Type.getType(methodDesc);
      Type[] argumentTypes = t.getArgumentTypes();

      boolean isStaticMethod = ((methodAccess & ACC_STATIC) != 0);
      int localIndex = isStaticMethod ? 0 : 1;
      for (Type argType : argumentTypes) {
        localIndex += argType.getSize();
      }
      return localIndex;
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
    ClassVisitor cv = new MethodWithWholeTryCatchVisitor(api, cw, "test", "(Ljava/lang/String;I)V");


    //（4）结合ClassReader和ClassVisitor
    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    //（5）生成byte[]
    byte[] bytes2 = cw.toByteArray();

    FileUtils.writeBytes(filepath, bytes2);
  }
}
```

转换完成之后，再次运行`HelloWorldRun`类，得到如下输出内容：

```shell
name is null
java.lang.ArithmeticException: / by zero // 增加的ASM异常捕获逻辑，逻辑中只是打印一下，然后继续上抛
	at sample.HelloWorld.div(Unknown Source)
	at sample.HelloWorld.test(Unknown Source)
	at run.HelloWorldRun.main(HelloWorldRun.java:8)
Exception in thread "main" java.lang.ArithmeticException: / by zero // 上抛后JVM打印的异常信息
	at sample.HelloWorld.div(Unknown Source)
	at sample.HelloWorld.test(Unknown Source)
	at run.HelloWorldRun.main(HelloWorldRun.java:8)
```

使用`javap`命令查看exception table的内容：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(java.lang.String, int);
    Code:
       0: aload_1
       1: invokevirtual #18                 // Method java/lang/String.length:()I
       4: istore_3
       5: getstatic     #24                 // Field java/lang/System.out:Ljava/io/PrintStream;
       8: new           #26                 // class java/lang/StringBuilder
      11: dup
      12: invokespecial #27                 // Method java/lang/StringBuilder."<init>":()V
      15: ldc           #29                 // String length =
      17: invokevirtual #33                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: iload_3
      21: invokevirtual #36                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      24: invokevirtual #40                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      27: invokevirtual #46                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      30: goto          42
      33: astore_3
      34: getstatic     #24                 // Field java/lang/System.out:Ljava/io/PrintStream;
      37: ldc           #48                 // String name is null
      39: invokevirtual #46                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: aload_0
      43: bipush        10
      45: iload_2
      46: invokevirtual #52                 // Method div:(II)I
      49: istore_3
      50: getstatic     #24                 // Field java/lang/System.out:Ljava/io/PrintStream;
      53: new           #26                 // class java/lang/StringBuilder
      56: dup
      57: invokespecial #27                 // Method java/lang/StringBuilder."<init>":()V
      60: ldc           #54                 // String val =
      62: invokevirtual #33                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      65: iload_3
      66: invokevirtual #36                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      69: invokevirtual #40                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      72: invokevirtual #46                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      75: return
      76: astore_3
      77: aload_3
      78: getstatic     #24                 // Field java/lang/System.out:Ljava/io/PrintStream;
      81: invokevirtual #60                 // Method java/lang/Exception.printStackTrace:(Ljava/io/PrintStream;)V
      84: aload_3
      85: athrow
    Exception table:
       from    to  target type
           0    30    33   Class java/lang/NullPointerException
           0    76    76   Class java/lang/Exception // ASM在visitMax()中添加的异常处理逻辑，排到第2个

...
}
```

#### 错误的异常捕获处理

```java
import org.objectweb.asm.*;

import static org.objectweb.asm.Opcodes.*;

public class MethodWithWholeTryCatchVisitor extends ClassVisitor {
  private final String methodName;
  private final String methodDesc;

  public MethodWithWholeTryCatchVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && methodName.equals(name) && methodDesc.equals(descriptor)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodWithWholeTryCatchAdapter(api, mv, access, descriptor);
      }
    }
    return mv;
  }

  private static class MethodWithWholeTryCatchAdapter extends MethodVisitor {
    private final int methodAccess;
    private final String methodDesc;

    private final Label startLabel = new Label();
    private final Label endLabel = new Label();
    private final Label handlerLabel = new Label();

    public MethodWithWholeTryCatchAdapter(int api, MethodVisitor methodVisitor, int methodAccess, String methodDesc) {
      super(api, methodVisitor);
      this.methodAccess = methodAccess;
      this.methodDesc = methodDesc;
    }

    public void visitCode() {
      // 首先，处理自己的代码逻辑
      // (1) visitTryCatchBlock和startLabel （注意，修改的是visitTryCatchBlock方法的位置）
      super.visitTryCatchBlock(startLabel, endLabel, handlerLabel, "java/lang/Exception");
      super.visitLabel(startLabel);

      // 其次，调用父类的方法实现
      super.visitCode();
    }

    @Override
    public void visitMaxs(int maxStack, int maxLocals) {
      // 首先，处理自己的代码逻辑
      // (2) endLabel
      super.visitLabel(endLabel);

      // (3) handlerLabel
      super.visitLabel(handlerLabel);
      int localIndex = getLocalIndex();
      super.visitVarInsn(ASTORE, localIndex);
      super.visitVarInsn(ALOAD, localIndex);
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "(Ljava/io/PrintStream;)V", false);
      super.visitVarInsn(ALOAD, localIndex);
      super.visitInsn(Opcodes.ATHROW);

      // 其次，调用父类的方法实现
      super.visitMaxs(maxStack, maxLocals);
    }

    private int getLocalIndex() {
      Type t = Type.getType(methodDesc);
      Type[] argumentTypes = t.getArgumentTypes();

      boolean isStaticMethod = ((methodAccess & ACC_STATIC) != 0);
      int localIndex = isStaticMethod ? 0 : 1;
      for (Type argType : argumentTypes) {
        localIndex += argType.getSize();
      }
      return localIndex;
    }
  }
}
```

转换完成之后，我们运行`HelloWorldRun`来查看一下输出内容：

```shell
// 发现第一个null异常处理后，后续代码不执行了，因为 ASM中把Exception异常处理提前注册到了exception table中，null异常被消耗后，后续逻辑也就是直接上抛异常了
java.lang.NullPointerException
	at sample.HelloWorld.test(Unknown Source)
	at run.HelloWorldRun.main(HelloWorldRun.java:8)
Exception in thread "main" java.lang.NullPointerException
	at sample.HelloWorld.test(Unknown Source)
	at run.HelloWorldRun.main(HelloWorldRun.java:8)
```

我们使用`javap`命令查看exception table的内容：

```shell
$ javap -c sample.HelloWorld
public class sample.HelloWorld {
...
  public void test(java.lang.String, int);
    Code:
       0: aload_1
       1: invokevirtual #20                 // Method java/lang/String.length:()I
       4: istore_3
       5: getstatic     #26                 // Field java/lang/System.out:Ljava/io/PrintStream;
       8: new           #28                 // class java/lang/StringBuilder
      11: dup
      12: invokespecial #29                 // Method java/lang/StringBuilder."<init>":()V
      15: ldc           #31                 // String length =
      17: invokevirtual #35                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: iload_3
      21: invokevirtual #38                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      24: invokevirtual #42                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      27: invokevirtual #48                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      30: goto          42
      33: astore_3
      34: getstatic     #26                 // Field java/lang/System.out:Ljava/io/PrintStream;
      37: ldc           #50                 // String name is null
      39: invokevirtual #48                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: aload_0
      43: bipush        10
      45: iload_2
      46: invokevirtual #54                 // Method div:(II)I
      49: istore_3
      50: getstatic     #26                 // Field java/lang/System.out:Ljava/io/PrintStream;
      53: new           #28                 // class java/lang/StringBuilder
      56: dup
      57: invokespecial #29                 // Method java/lang/StringBuilder."<init>":()V
      60: ldc           #56                 // String val =
      62: invokevirtual #35                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      65: iload_3
      66: invokevirtual #38                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      69: invokevirtual #42                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      72: invokevirtual #48                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      75: return
      76: astore_3
      77: aload_3
      78: getstatic     #26                 // Field java/lang/System.out:Ljava/io/PrintStream;
      81: invokevirtual #60                 // Method java/lang/Exception.printStackTrace:(Ljava/io/PrintStream;)V
      84: aload_3
      85: athrow
    Exception table:
       from    to  target type
           0    76    76   Class java/lang/Exception // Exception异常处理被提前注册，所以下面NullPointerException就不会再触发了
           0    30    33   Class java/lang/NullPointerException
...
}
```

### 3.3.3 操作数栈上的异常对象

当方法执行过程中，遇到异常的时候，在operand stack上会有一个“异常对象”。

这个“异常对象”，可能有对应的代码处理逻辑；但是，我们想对这个“异常对象”进行多次处理，那下一步要怎么处理呢？

有两种处理方式：

- 第一种方式，就是将“异常对象”存储到local variable内；后续使用的时候，再将“异常对象”加载到operand stack上。
- 第二种方式，就是在operand stack上，要使用的时候，就进行一次`dup`（复制）操作。

#### 存储异常对象

对于第一种情况，我们要将“异常对象”存储到local variable当中，但是，我们应该把这个“异常对象”存储到哪个位置上呢？换句话说，我们怎么计算“异常对象”在local variable当中的位置呢？我们需要考虑三个因素：

- 第一个因素，是否需要存储`this`变量。
- 第二个因素，是否需要存储方法的参数（method parameters）。
- 第三个因素，是否需要存储方法内部定义的局部变量（method local var）。

考虑这三个因素之后，我们就可以确定“异常对象”应该存储的一个什么位置上了。

在上面的`MethodWithWholeTryCatchAdapter`当中，有一个`getLocalIndex()`方法。这个方法就是考虑了`this`和方法的参数（method parameters），之后就是“异常对象”可以存储的位置。再者，方法内部定义的局部变量（method local var），有的时候有，有的时候没有，大家根据实际的情况来决定是否添加。

```java
class MethodWithWholeTryCatchAdapter extends MethodVisitor {
  @Override
  public void visitMaxs(int maxStack, int maxLocals) {
    // 首先，处理自己的代码逻辑
    // (2) endLabel
    super.visitLabel(endLabel);

    // (3) handlerLabel
    super.visitLabel(handlerLabel);
    int localIndex = getLocalIndex();
    super.visitVarInsn(ASTORE, localIndex);
    super.visitVarInsn(ALOAD, localIndex);
    super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
    super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "(Ljava/io/PrintStream;)V", false);
    super.visitVarInsn(ALOAD, localIndex);
    super.visitInsn(Opcodes.ATHROW);

    // 其次，调用父类的方法实现
    super.visitMaxs(maxStack, maxLocals);
  }    

  private int getLocalIndex() {
    Type t = Type.getType(methodDesc);
    Type[] argumentTypes = t.getArgumentTypes();

    boolean isStaticMethod = ((methodAccess & ACC_STATIC) != 0);
    int localIndex = isStaticMethod ? 0 : 1;
    for (Type argType : argumentTypes) {
      localIndex += argType.getSize();
    }
    return localIndex;
  }
}
```

#### 操作数栈上复制

第二种方式，就是不依赖于local variable，也就不用去计算一个具体的位置了。那么，我们想对“异常对象”进行多次的处理，直接在operand stack进行`dup`（复制），就能够进行多次处理了。

```java
import org.objectweb.asm.*;

import static org.objectweb.asm.Opcodes.*;

public class MethodWithWholeTryCatchVisitor extends ClassVisitor {
  private final String methodName;
  private final String methodDesc;

  public MethodWithWholeTryCatchVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc) {
    super(api, classVisitor);
    this.methodName = methodName;
    this.methodDesc = methodDesc;
  }

  @Override
  public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if (mv != null && !"<init>".equals(name) && methodName.equals(name) && methodDesc.equals(descriptor)) {
      boolean isAbstractMethod = (access & ACC_ABSTRACT) != 0;
      boolean isNativeMethod = (access & ACC_NATIVE) != 0;
      if (!isAbstractMethod && !isNativeMethod) {
        mv = new MethodWithWholeTryCatchAdapter(api, mv, access, descriptor);
      }
    }
    return mv;
  }

  private static class MethodWithWholeTryCatchAdapter extends MethodVisitor {
    private final int methodAccess;
    private final String methodDesc;

    private final Label startLabel = new Label();
    private final Label endLabel = new Label();
    private final Label handlerLabel = new Label();

    public MethodWithWholeTryCatchAdapter(int api, MethodVisitor methodVisitor, int methodAccess, String methodDesc) {
      super(api, methodVisitor);
      this.methodAccess = methodAccess;
      this.methodDesc = methodDesc;
    }

    public void visitCode() {
      // 首先，处理自己的代码逻辑
      // (1) startLabel
      super.visitLabel(startLabel);

      // 其次，调用父类的方法实现
      super.visitCode();
    }

    @Override
    public void visitMaxs(int maxStack, int maxLocals) {
      // 首先，处理自己的代码逻辑
      // (2) endLabel
      super.visitLabel(endLabel);

      // (3) handlerLabel
      super.visitLabel(handlerLabel);
      super.visitInsn(DUP);              // 注意，这里使用了DUP指令
      super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      super.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "(Ljava/io/PrintStream;)V", false);
      super.visitInsn(Opcodes.ATHROW);

      // (4) visitTryCatchBlock
      super.visitTryCatchBlock(startLabel, endLabel, handlerLabel, "java/lang/Exception");

      // 其次，调用父类的方法实现
      super.visitMaxs(maxStack, maxLocals);
    }
  }
}
```

## 3.4 Java 8 Lambda

在《[Java ASM系列一：Core API](https://lsieun.github.io/java/asm/java-asm-season-01.html)》当中，主要是对Core API进行了介绍，但是并没有使用Core API来生成Java 8 Lambda表达式。

在本文当中，我们主要对Java 8 Lambda的两方面进行介绍：

- 第一方面，如何使用ASM生成Lambda表达式？
- 第二方面，探究Lambda表达式的实现原理是什么？

### 3.4.1 使用ASM生成Lambda

#### 预期目标

我们的预期目标是生成一个`HelloWorld`类，代码如下：

```java
import java.util.function.BiFunction;

public class HelloWorld {
  public void test() {
    // 这里实际上 会生成一个类实例，该类实现 BiFunction 接口，然后apply方法中使用Math类的静态方法max
    BiFunction<Integer, Integer, Integer> func = Math::max;
    Integer result = func.apply(10, 20);
    System.out.println(result);
  }
}
```

#### 编码实现

在下面的代码中，我们重点关注两个点：

- 第一点，是`Handle`实例的创建。
- 第二点，是`MethodVisitor.visitInvokeDynamicInsn()`方法。

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
      mv1.visitMaxs(0, 0);
      mv1.visitEnd();
    }

    {
      MethodVisitor mv2 = cw.visitMethod(ACC_PUBLIC, "test", "()V", null, null);
      mv2.visitCode();
      // 第1点，Handle实例的创建
      Handle bootstrapMethodHandle = new Handle(H_INVOKESTATIC, "java/lang/invoke/LambdaMetafactory", "metafactory",
                                                "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;" +
                                                "Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;", false);
      // 第2点，MethodVisitor.visitInvokeDynamicInsn()方法的调用
      mv2.visitInvokeDynamicInsn("apply", "()Ljava/util/function/BiFunction;",
                                 bootstrapMethodHandle,
                                 // 因为java泛型擦除，所以实际方法签名不带有泛型参数，取而代之的是Object（实际方法签名）
                                 Type.getType("(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;"),
                                 new Handle(H_INVOKESTATIC, "java/lang/Math", "max", "(II)I", false),
                                 // 这个参数是用于说明实际传参时的参数类型（伪方法签名）
                                 Type.getType("(Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;")
                                );
      mv2.visitVarInsn(ASTORE, 1);
      mv2.visitVarInsn(ALOAD, 1);
      mv2.visitIntInsn(BIPUSH, 10);
      mv2.visitMethodInsn(INVOKESTATIC, "java/lang/Integer", "valueOf", "(I)Ljava/lang/Integer;", false);
      mv2.visitIntInsn(BIPUSH, 20);
      mv2.visitMethodInsn(INVOKESTATIC, "java/lang/Integer", "valueOf", "(I)Ljava/lang/Integer;", false);
      mv2.visitMethodInsn(INVOKEINTERFACE, "java/util/function/BiFunction", "apply",
                          "(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;", true);
      mv2.visitTypeInsn(CHECKCAST, "java/lang/Integer");
      mv2.visitVarInsn(ASTORE, 2);
      mv2.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
      mv2.visitVarInsn(ALOAD, 2);
      mv2.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/Object;)V", false);
      mv2.visitInsn(RETURN);
      mv2.visitMaxs(0, 0);
      mv2.visitEnd();
    }

    cw.visitEnd();

    // (3) 调用toByteArray()方法
    return cw.toByteArray();
  }
}
```

#### 验证结果

接下来，我们来验证生成的Lambda表达式是否能够正常运行。

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

### 3.4.2 探究Lambda的实现原理

简单来说，Lambda表达式的内部原理，是借助于Java ASM生成匿名内部类来实现的。

#### 追踪Lambda

首先，我们使用`javap -v -p sample.HelloWorld`命令查看输出结果：

```text
$ javap -v -p sample.HelloWorld
...
BootstrapMethods:
  0: #28 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
        Ljava/lang/invoke/MethodHandles$Lookup;
        Ljava/lang/String;Ljava/lang/invoke/MethodType;
        Ljava/lang/invoke/MethodType;
        Ljava/lang/invoke/MethodHandle;
        Ljava/lang/invok e/MethodType;
      )Ljava/lang/invoke/CallSite;
    Method arguments:
      #29 (Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
      #30 invokestatic java/lang/Math.max:(II)I
      #31 (Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
```

通过上面的输出结果，我们定位到BootstrapMethods的部分，可以看到它使用了`java.lang.invoke.LambdaMetafactory`类的`metafactory()`方法。

至此，我们想表达的意思：Lambda表达式是与`LambdaMetafactory.metafactory()`方法有关联关系的。

------

在IDE当中，我们可以查看`LambdaMetafactory.metafactory()`方法，其内容如下：

![Java ASM系列：（067）Java 8 Lambda原理探究_ClassFile](https://s2.51cto.com/images/20210831/1630383548682537.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在上图中，我们可以看到`mf`指向一个`InnerClassLambdaMetafactory`的实例，并在最后调用了`buildCallSite()`方法。

------

接着，我们跳转到`java.lang.invoke.InnerClassLambdaMetafactory`类的`buildCallSite()`方法

![Java ASM系列：（067）Java 8 Lambda原理探究_Java_02](https://s2.51cto.com/images/20210831/1630383583438639.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

在`java.lang.invoke.InnerClassLambdaMetafactory`类的`spinInnerClass()`方法中，找到如下代码：

```java
final byte[] classBytes = cw.toByteArray();
```

在其语句上，打一个断点：

![Java ASM系列：（067）Java 8 Lambda原理探究_ASM_03](https://s2.51cto.com/images/20210831/1630383608841146.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

调试运行，然后使用如下表达式来查看`classBytes`的值：

```java
Arrays.toString(classBytes)
```

其实，这里的`classBytes`就是生成的类文件的字节码内容，这样的字符串内容就是该字节码内容的另一种表现形式。那么，拿到这样一个字符串内容之后，我们应该如何处理呢？

#### 查看生成的类

在[项目代码](https://gitee.com/lsieun/learn-java-asm)中，有一个`PrintASMTextLambda`类，它的作用就是将上述字符串内容的类信息打印出来。

```java
import lsieun.utils.StringUtils;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.util.Printer;
import org.objectweb.asm.util.Textifier;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.IOException;
import java.io.PrintWriter;

public class PrintASMTextLambda {
  public static void main(String[] args) throws IOException {
    // (1) 设置参数
    String str = "[-54, -2, -70, -66, ...]";
    byte[] bytes = StringUtils.array2Bytes(str);
    int parsingOptions = ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG;

    // (2) 打印结果
    Printer printer = new Textifier();
    PrintWriter printWriter = new PrintWriter(System.out, true);
    TraceClassVisitor traceClassVisitor = new TraceClassVisitor(null, printer, printWriter);
    new ClassReader(bytes).accept(traceClassVisitor, parsingOptions);
  }
}
```

我们可以将代表字节码内容的字符串放到上面代码的`str`变量中，然后运行`PrintASMTextLambda`可以得到如下结果：

```java
// class version 52.0 (52)
// access flags 0x1030
final synthetic class sample/HelloWorld$$Lambda$1 implements java/util/function/BiFunction {


  // access flags 0x2
  private <init>()V
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x1 
  public apply(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
  @Ljava/lang/invoke/LambdaForm$Hidden;()
    ALOAD 1
    CHECKCAST java/lang/Integer
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    ALOAD 2
    CHECKCAST java/lang/Integer
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    INVOKESTATIC java/lang/Math.max (II)I
    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
    ARETURN
    MAXSTACK = 2
    MAXLOCALS = 3
}
```

通过上面的输出结果，我们可以看到：

- 第一点，当前类的名字叫`sample/HelloWorld$$Lambda$1`

  - 当前类带有`final`和`synthetic`标识。(synthetic是正常java代码不会有的，只有生成的才有这个标识)
  - 当前类实现了`java/util/function/BiFunction`接口。

- 第二点，在`sample/HelloWorld$$Lambda$1`类当中，它定义了一个构造方法（`<init>()V`）。

- 第三点，在`sample/HelloWorld$$Lambda$1`

  类当中，它定义了一个`apply`方法（`apply(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;`）。

  - 这个`apply`方法正是`BiFunction`接口中定义的方法。
  - `apply`方法的内部代码逻辑，是通过调用`Math.max()`方法来实现的；而`Math::max`正是Lambda表达式的内容。

**至此，我们可以知道：Lambda表达式的实现，是由JVM调用ASM来实现的。也就是说，JVM使用ASM创建一个匿名内部类（`sample/HelloWorld$$Lambda$1`），让该匿名内部类实现特定的接口（`java/util/function/BiFunction`），并在接口定义的方法（`apply`）实现中包含Lambda表达式的内容。**

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-4.html)
>
> 4.1. The ClassFile Structure
>
> **The `ACC_SYNTHETIC` flag indicates that this class or interface was generated by a compiler and does not appear in source code.**
>
> Table 4.1-B. Class access and property modifiers
>
> | Flag Name        | Value  | Interpretation                                               |
> | ---------------- | ------ | ------------------------------------------------------------ |
> | `ACC_PUBLIC`     | 0x0001 | Declared `public`; may be accessed from outside its package. |
> | `ACC_FINAL`      | 0x0010 | Declared `final`; no subclasses allowed.                     |
> | `ACC_SUPER`      | 0x0020 | Treat superclass methods specially when invoked by the *invokespecial* instruction. |
> | `ACC_INTERFACE`  | 0x0200 | Is an interface, not a class.                                |
> | `ACC_ABSTRACT`   | 0x0400 | Declared `abstract`; must not be instantiated.               |
> | `ACC_SYNTHETIC`  | 0x1000 | **Declared synthetic; not present in the source code.**      |
> | `ACC_ANNOTATION` | 0x2000 | Declared as an annotation type.                              |
> | `ACC_ENUM`       | 0x4000 | Declared as an `enum` type.                                  |
> | `ACC_MODULE`     | 0x8000 | Is a module, not a class or interface.                       |
