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

## 1.3 