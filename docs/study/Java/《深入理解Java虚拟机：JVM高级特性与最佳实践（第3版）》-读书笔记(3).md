# 《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》-读书笔记(3)

# 第三部分 虚拟机执行子系统

# 6. 第6章 类文件结构

​	代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。

## 6.1 概述

​	曾记得在第一堂计算机程序课上老师就讲过：“计算机只认识0和1，所以我们写的程序需要被编译器翻译成由0和1构成的二进制格式才能被计算机执行。”十多年过去了，今天的计算机仍然只能识别0和1，但由于最近十年内虚拟机以及大量建立在虚拟机之上的程序语言如雨后春笋般出现并蓬勃发展，把我们编写的程序编译成二进制本地机器码（Native Code）已不再是唯一的选择，**越来越多的程序语言选择了与操作系统和机器指令集无关的、平台中立的格式作为程序编译后的存储格式**。

## 6.2 无关性的基石

个人小结：

+ 实现语言无关性的基础：虚拟机和字节码存储格式。

---

> [2.class类文件结构](https://www.yuque.com/ltvc5b/java/rozzn1)

​	如果全世界所有计算机的指令集就只有x86一种，操作系统就只有Windows一种，那也许就不会有Java语言的出现。Java在刚刚诞生之时曾经提出过一个非常著名的宣传口号“一次编写，到处运行（Write Once，Run Anywhere）”，这句话充分表达了当时软件开发人员对冲破平台界限的渴求。在每时每刻都充满竞争的IT业界，不可能只有Wintel[^1]存在，我们也不希望出现只有Wintel而没有竞争者的世界，各种不同的硬件体系结构、各种不同的操作系统肯定将会长期并存发展。“与平台无关”的理想最终只有实现在操作系统以上的应用层：Oracle公司以及其他虚拟机发行商发布过许多可以运行在各种不同硬件平台和操作系统上的Java虚拟机，这些虚拟机都可以载入和执行同一种平台无关的字节码，从而实现了程序的“一次编写，到处运行”。

​	各种不同平台的Java虚拟机，以及所有平台都统一支持的程序存储格式——字节码（Byte Code）是构成平台无关性的基石，但本节标题中笔者刻意省略了“平台”二字，那是因为笔者注意到虚拟机的另外一种中立特性——**语言无关性**正在越来越被开发者所重视。直到今天，或许还有相当一部分程序员认为Java虚拟机执行Java程序是一件理所当然和天经地义的事情。但**在Java技术发展之初，设计者们就曾经考虑过并实现了让其他语言运行在Java虚拟机之上的可能性**，他们在发布规范文档的时候，也刻意把Java的规范拆分成了《Java语言规范》（The Java Language Specification）及《Java虚拟机规范》（The Java Virtual Machine Specification）两部分。并且早在1997年发表的第一版《Java虚拟机规范》中就曾经承诺过：“在未来，我们会对Java虚拟机进行适当的扩展，以便更好地支持其他语言运行于Java虚拟机之上”（In the future，we will consider bounded extensions to the Java virtual machine to provide better support for other languages）。Java虚拟机发展到今天，尤其是在2018年，基于HotSpot扩展而来的GraalVM公开之后，当年的虚拟机设计者们已经基本兑现了这个承诺。

​	时至今日，商业企业和开源机构已经在Java语言之外发展出一大批运行在Java虚拟机之上的语言，如Kotlin、Clojure、Groovy、JRuby、JPython、Scala等。相比起基数庞大的Java程序员群体，使用过这些语言的开发者可能还不是特别多，但是听说过的人肯定已经不少，随着时间的推移，谁能保证日后Java虚拟机在语言无关性上的优势不会赶上甚至超越它在平台无关性上的优势呢？	

​	**实现语言无关性的基础仍然是虚拟机和字节码存储格式**。Java虚拟机不与包括Java语言在内的任何程序语言绑定，它只与“Class文件”这种特定的二进制文件格式所关联，Class文件中包含了Java虚拟机指令集、符号表以及若干其他辅助信息。<u>基于安全方面的考虑，《Java虚拟机规范》中要求在Class文件必须应用许多强制性的语法和结构化约束，但图灵完备的字节码格式，保证了任意一门功能性语言都可以表示为一个能被Java虚拟机所接受的有效的Class文件</u>。作为一个通用的、与机器无关的执行平台，任何其他语言的实现者都可以将Java虚拟机作为他们语言的运行基础，以Class文件作为他们产品的交付媒介。例如，使用Java编译器可以把Java代码编译为存储字节码的Class文件，使用JRuby等其他语言的编译器一样可以把它们的源程序代码编译成Class文件。虚拟机丝毫不关心Class的来源是什么语言，它与程序语言之间的关系如下图所示。

​	Java语言中的各种语法、关键字、常量变量和运算符号的语义最终都会由多条字节码指令组合来表达，这决定了字节码指令所能提供的语言描述能力必须比Java语言本身更加强大才行。因此，有一些Java语言本身无法有效支持的语言特性并不代表在字节码中也无法有效表达出来，这为其他程序语言实现一些有别于Java的语言特性提供了发挥空间。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/153889/1602930445309-a2335ac9-d9e7-4d28-9a80-75e858a1ac3b.png)

[^1]:Wintel指微软的Windows与Intel的芯片相结合，曾经是业界最强大的联盟。

## 6.3 Class类文件的结构

个人小结：

+ *任何一个Class文件都对应着唯一的一个类或接口的定义信息2，但是反过来说，类或接口并不一定都得定义在文件里（譬如类或接口也可以动态生成，直接送入类加载器中）。*
+ <u>Class文件是一组**以8个字节为基础单位**的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。当遇到需要占用8个字节以上空间的数据项时，则会按照**高位在前**[^3]的方式分割成若干个8个字节进行存储。</u>
+ 根据《Java虚拟机规范》的规定，Class文件格式采用一种**类似于C语言结构体的伪结构**来存储数据，这种伪结构中只有两种数据类型：**“无符号数”和“表”**

---

> [《深入理解 Java 虚拟机》笔记——第6章 类文件结构](https://blog.csdn.net/bm1998/article/details/110451370#63_Class__31)

​	解析Class文件的数据结构是本章的最主要内容。笔者曾经在前言中阐述过本书的写作风格：力求在保证逻辑准确的前提下，用尽量通俗的语言和案例去讲述虚拟机中与开发关系最为密切的内容。但是，对文件格式、结构方面的学习，有点类似于“读字典”，读者阅读本章时，大概会不可避免地感到比较枯燥，但这部分内容又是Java虚拟机的重要基础之一，是了解虚拟机的必经之路，如果想比较深入地学习虚拟机相关知识，这部分是无法回避的。

​	Java技术能够一直保持着非常良好的向后兼容性，Class文件结构的稳定功不可没，任何一门程序语言能够获得商业上的成功，都不可能去做升级版本后，旧版本编译的产品就不再能够运行这种事情。<u>本章所讲述的关于Class文件结构的内容，绝大部分都是在第一版的《Java虚拟机规范》（1997年发布，对应于JDK 1.2时代的Java虚拟机）中就已经定义好的，内容虽然古老，但时至今日，Java发展经历了十余个大版本、无数小更新，那时定义的Class文件格式的各项细节几乎没有出现任何改变。尽管不同版本的《Java虚拟机规范》对Class文件格式进行了几次更新，但基本上只是在原有结构基础上新增内容、扩充功能，并未对已定义的内容做出修改</u>。

​	*注意　任何一个Class文件都对应着唯一的一个类或接口的定义信息[^2]，但是反过来说，类或接口并不一定都得定义在文件里（譬如类或接口也可以动态生成，直接送入类加载器中）。本章中，笔者只是通俗地将任意一个有效的类或接口所应当满足的格式称为“Class文件格式”，实际上它完全不需要以磁盘文件的形式存在。*

​	<u>Class文件是一组**以8个字节为基础单位**的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。当遇到需要占用8个字节以上空间的数据项时，则会按照高位在前[^3]的方式分割成若干个8个字节进行存储。</u>

​	<u>根据《Java虚拟机规范》的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：**“无符号数”和“表”**</u>。后面的解析都要以这两种数据类型为基础，所以这里笔者必须先解释清楚这两个概念。

+ **无符号数**属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述**数字**、**索引引用**、**数量值**或者**按照UTF-8编码构成字符串值**。

+ **表**是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上也可以视作是一张表，这张表由表6-1所示的数据项按严格顺序排列构成。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201201203746403.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JtMTk5OA==,size_16,color_FFFFFF,t_70#pic_center)

​	<u>无论是无符号数还是表，当需要描述同一类型但数量不定的多个数据时，经常会使用一个前置的**容量计数器**加若干个连续的数据项的形式，这时候称这一系列连续的某一类型的数据为某一类型的“集合”</u>。

​	本节结束之前，笔者（书籍作者）需要再强调一次，Class的结构不像XML等描述语言，由于它**没有任何分隔符号**，所以在表6-1中的数据项，无论是顺序还是数量，甚至于数据存储的字节序（Byte Ordering，Class文件中字节序为Big-Endian）这样的细节，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，全部都不允许改变。接下来，我们将一起看看这个表中各个数据项的具体含义。

[^2]:其实也有反例，譬如package-info.class、module-info.class这些文件就属于完全描述性的。
[^3]:这种顺序称为“Big-Endian”，具体顺序是指按高位字节在地址最低位，最低字节在地址最高位来存储数据，它是SPARC、PowerPC等处理器的默认多字节存储顺序，而x86等处理器则是使用了相反的“Little-Endian”顺序来存储数据。

### 6.3.1 魔数与Class文件的版本

个人小结：

+ **每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件**。

+ **紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）**。

  <small>高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后版本的Class文件，因为《Java虚拟机规范》在Class文件校验部分明确要求了即使文件格式并未发生任何变化，虚拟机也必须拒绝执行超过其版本号的Class文件</small>

+ *如果Class文件中使用了该版本JDK尚未列入正式特性清单中的预览功能，则必须把次版本号标识为65535，以便Java虚拟机在加载类文件时能够区分出来*

---

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	**每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件**。不仅是Class文件，很多文件格式标准中都有使用魔数来进行身份识别的习惯，譬如图片格式，如GIF或者JPEG等在文件头中都存有魔数。使用魔数而不是扩展名来进行识别主要是基于安全考虑，因为文件扩展名可以随意改动。文件格式的制定者可以自由地选择魔数值，只要这个魔数值还没有被广泛采用过而且不会引起混淆。Class文件的魔数取得很有“浪漫气息”，值为0xCAFEBABE（咖啡宝贝？）。这个魔数值在Java还被称作“Oak”语言的时候（大约是1991年前后）就已经确定下来了。它还有一段很有趣的历史，据Java开发小组最初的关键成员Patrick Naughton所说：“我们一直在寻找一些好玩的、容易记忆的东西，选择0xCAFEBABE是因为它象征着著名咖啡品牌Peet’s Coffee深受欢迎的Baristas咖啡。”[^4]这个魔数似乎也预示着日后“Java”这个商标名称的出现。

​	**紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）**。Java的版本号是从45开始的，JDK 1.1之后的每个JDK大版本发布主版本号向上加1（JDK 1.0～1.1使用了45.0～45.3的版本号），<u>高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后版本的Class文件，因为《Java虚拟机规范》在Class文件校验部分明确要求了即使文件格式并未发生任何变化，虚拟机也必须拒绝执行超过其版本号的Class文件</u>。

​	例如，JDK 1.1能支持版本号为45.0～45.65535的Class文件，无法执行版本号为46.0以上的Class文件，而JDK 1.2则能支持45.0～46.65535的Class文件。目前最新的JDK版本为13，可生成的Class文件主版本号最大值为57.0。

​	为了讲解方便，笔者准备了一段最简单的Java代码（如代码清单6-1所示），本章后面的内容都将以这段程序使用JDK 6编译输出的Class文件为基础来进行讲解，建议读者不妨用较新版本的JDK跟随本章的实验流程自己动手测试一遍。

​	代码清单6-1　简单的Java代码

```java
package org.fenixsoft.clazz;
public class TestClass {
  private int m;
  public int inc() {
    return m + 1;
  }
}
```

​	图6-2显示的是使用十六进制编辑器WinHex打开这个Class文件的结果，可以清楚地看见开头4个字节的十六进制表示是0xCAFEBABE，代表次版本号的第5个和第6个字节值为0x0000，而主版本号的值为0x0032，也即是十进制的50，该版本号说明这个是可以被JDK 6或以上版本虚拟机执行的Class文件。

![这里写图片描述](https://img-blog.csdn.net/20170801163853231)

下面是我jdk11进行javac的结果

```class
00000000: cafe babe 0000 0037 0013 0a00 0400 0f09  .......7........
00000010: 0003 0010 0700 1107 0012 0100 016d 0100  .............m..
00000020: 0149 0100 063c 696e 6974 3e01 0003 2829  .I...<init>...()
00000030: 5601 0004 436f 6465 0100 0f4c 696e 654e  V...Code...LineN
00000040: 756d 6265 7254 6162 6c65 0100 0369 6e63  umberTable...inc
00000050: 0100 0328 2949 0100 0a53 6f75 7263 6546  ...()I...SourceF
00000060: 696c 6501 000e 5465 7374 436c 6173 732e  ile...TestClass.
00000070: 6a61 7661 0c00 0700 080c 0005 0006 0100  java............
00000080: 1d6f 7267 2f66 656e 6978 736f 6674 2f63  .org/fenixsoft/c
00000090: 6c61 7a7a 2f54 6573 7443 6c61 7373 0100  lazz/TestClass..
000000a0: 106a 6176 612f 6c61 6e67 2f4f 626a 6563  .java/lang/Objec
000000b0: 7400 2100 0300 0400 0000 0100 0200 0500  t.!.............
000000c0: 0600 0000 0200 0100 0700 0800 0100 0900  ................
000000d0: 0000 1d00 0100 0100 0000 052a b700 01b1  ...........*....
000000e0: 0000 0001 000a 0000 0006 0001 0000 0002  ................
000000f0: 0001 000b 000c 0001 0009 0000 001f 0002  ................
00000100: 0001 0000 0007 2ab4 0002 0460 ac00 0000  ......*....`....
00000110: 0100 0a00 0000 0600 0100 0000 0500 0100  ................
00000120: 0d00 0000 0200 0e                        .......
```

​	表6-2列出了从JDK 1.1到13之间，主流JDK版本编译器输出的默认的和可支持的Class文件版本号。

![这里写图片描述](https://img-blog.csdn.net/20170801164013108)

​	*注：从JDK 9开始，Javac编译器不再支持使用-source参数编译版本号小于1.5的源码。*

​	关于次版本号，曾经在现代Java（即Java 2）出现前被短暂使用过，JDK 1.0.2支持的版本45.0～45.3（包括45.0～45.3）。JDK 1.1支持版本45.0～45.65535，从JDK 1.2以后，直到JDK 12之前次版本号均未使用，全部固定为零。而到了JDK 12时期，由于JDK提供的功能集已经非常庞大，有一些复杂的新特性需要以“公测”的形式放出，所以设计者重新启用了副版本号，将它用于标识“技术预览版”功能特性的支持。<u>如果Class文件中使用了该版本JDK尚未列入正式特性清单中的预览功能，则必须把次版本号标识为65535，以便Java虚拟机在加载类文件时能够区分出来</u>。

[^4]:根据Java之父James Gosling的解释，当时还定义了“CAFEDEAD”用作另一种对象持久化文件格式的魔数，只是后来该格式被废弃掉了，所以并未流传开来。

### 6.3.2 常量池

个人小结：

+ **紧接着主、次版本号之后的是常量池入口，常量池可以比喻为Class文件里的资源仓库，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一，另外，它还是在Class文件中第一个出现的表类型数据项目**。
+ **由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count）**。<u>与Java中语言习惯不同，这个容量计数是从1而不是0开始的。</u>这样做的目的在于，如果后面**某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义，可以把索引值设置为0来表示**
+ *Class文件结构中只有常量池的容量计数是从1开始，对于其他集合类型，包括接口索引集合、字段表集合、方法表集合等的容量计数都与一般习惯相同，是从0开始*
+ **常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）**
+ <u>Java代码在进行Javac编译的时候，并不像C和C++那样有“连接”这一步骤，而是在虚拟机加载Class文件的时候进行动态连接（具体见第7章）</u>。也就是说，**在Class文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用的**。
+ 常量池中每一项常量都是一个表
+ 由于Class文件中方法、字段等都需要引用CONSTANT_Utf8_info型常量来描述名称，所以CONSTANT_Utf8_info型常量的最大长度也就是Java中方法、字段名的最大长度。而这里的最大长度就是length的最大值，既u2类型能表达的最大值65535。所以Java程序中如果定义了超过64KB英文字符的变量或方法名，即使规则和全部字符都是合法的，也会无法编译

---

​	**紧接着主、次版本号之后的是常量池入口，常量池可以比喻为Class文件里的资源仓库，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一，另外，它还是在Class文件中第一个出现的表类型数据项目**。

​	**由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count）**。<u>与Java中语言习惯不同，这个容量计数是从1而不是0开始的</u>，如图6-3所示，常量池容量（偏移地址：0x00000008）为十六进制数0x0016，即十进制的22，这就代表常量池中有21项常量，索引值范围为1～21。在Class文件格式规范制定之时，设计者将第0项常量空出来是有特殊考虑的，这样做的目的在于，如果后面**某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义，可以把索引值设置为0来表示**。<u>Class文件结构中只有常量池的容量计数是从1开始，对于其他集合类型，包括接口索引集合、字段表集合、方法表集合等的容量计数都与一般习惯相同，是从0开始</u>。

![这里写图片描述](https://img-blog.csdn.net/20170801164959785)

​	**常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）**。字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。而符号引用则属于编译原理方面的概念，主要包括下面几类常量：

+ 被模块导出或者开放的包（Package）

+ 类和接口的全限定名（Fully Qualified Name）

+ 字段的名称和描述符（Descriptor）

+ 方法的名称和描述符

+ 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）

+ **动态调用点和动态常量**（Dynamically-Computed Call Site、Dynamically-Computed Constant）

​	<u>Java代码在进行Javac编译的时候，并不像C和C++那样有“连接”这一步骤，而是在虚拟机加载Class文件的时候进行动态连接（具体见第7章）</u>。也就是说，**在Class文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用的**。当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。关于类的创建和动态连接的内容，在下一章介绍虚拟机类加载过程时再详细讲解。

​	**常量池中每一项常量都是一个表**，最初常量表中共有11种结构各不相同的表结构数据，后来为了更好地支持动态语言调用，额外增加了4种动态语言相关的常量[^5]，为了支持Java模块化系统（Jigsaw），又加入了CONSTANT_Module_info和CONSTANT_Package_info两个常量，所以截至JDK13，常量表中分别有17种不同类型的常量。

​	这17类表都有一个共同的特点，表结构起始的第一位是个u1类型的标志位（tag，取值见表6-3中标志列），代表着当前常量属于哪种常量类型。17种常量类型所代表的具体含义如表6-3所示。

​	表6-3　常量池的项目类型

| 类型                              | 标志 | 描述                           |
| --------------------------------- | ---- | ------------------------------ |
| CONSTRANT_Utf8_info               | 1    | UTF-8编码的字符串              |
| CONSTRANT_Integer_info            | 3    | 整型字面量                     |
| CONSTRANT_Float_info              | 4    | 浮点型字面量                   |
| CONSTRANT_Long_info               | 5    | 长整型字面量                   |
| CONSTRANT_Double_info             | 6    | 双精度浮点型字面量             |
| CONSTRANT_Class_info              | 7    | 类或接口的符号引用             |
| CONSTRANT_String_info             | 8    | 字符串类型字面量               |
| CONSTRANT_Fieldref_info           | 9    | 字段的符号引用                 |
| CONSTRANT_Methodref_info          | 10   | 类中方法的符号引用             |
| CONSTRANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用           |
| CONSTRANT_NameAndType_info        | 12   | 字段或方法的部分符号引用       |
| CONSTRANT_MethodHandle_info       | 15   | 表示方法句柄                   |
| CONSTRANT_MethodType_info         | 16   | 表示方法类型                   |
| CONSTRANT_Dynamic_info            | 17   | 表示一个动态计算常量           |
| CONSTRANT_InvokeDynamic_info      | 18   | 表示一个动态方法调用点         |
| CONSTRANT_Module_info             | 19   | 表示一个模块                   |
| CONSTRANT_Package_info            | 20   | 表示一个模块中开放或者导出的包 |

​	之所以说常量池是最烦琐的数据，是因为这17种常量类型各自有着完全独立的数据结构，两两之间并没有什么共性和联系，因此只能逐项进行讲解。

​	请读者回头看看图6-3中常量池的第一项常量，它的标志位（偏移地址：0x0000000A）是0x07，查表6-3的标志列可知这个常量属于CONSTANT_Class_info类型，此类型的常量代表一个类或者接口的符号引用。CONSTANT_Class_info的结构比较简单，如表6-4所示。

​	表6-4　CONSTANT_Class_info型常量的结构

| 类型 | 名称       | 数量 |
| ---- | ---------- | ---- |
| u1   | tag        | 1    |
| u2   | name_index | 1    |

​	tag是标志位，上面已经讲过了，它用于区分常量类型；name_index是一个索引值，它指向常量池中一个CONSTANT_Utf8_info类型常量，此常量代表了这个类（或者接口）的全限定名，这里name_index值（偏移地址：0x0000000B）为0x0002，也即是指向了常量池中的第二项常量。继续从图6-3中查找第二项常量，它的标志位（地址：0x0000000D）是0x01，查表6-3可知确实是一个CONSTANT_Utf8_info类型的常量。CONSTANT_Utf8_info类型的结构见表6-5。

| 类型 | 名称   | 数量   |
| ---- | ------ | ------ |
| u1   | tag    | 1      |
| u2   | length | 1      |
| u1   | bytes  | length |

​	**length值说明了这个UTF-8编码的字符串长度是多少字节，它后面紧跟着的长度为length字节的连续数据是一个使用<u>UTF-8缩略编码</u>表示的字符串**。<u>UTF-8缩略编码与普通UTF-8编码的区别是：从'\u0001'到'\u007f'之间的字符（相当于1～127的ASCII码）的缩略编码使用一个字节表示，从'\u0080'到'\u07ff'之间的所有字符的缩略编码用两个字节表示，从'\u0800'开始到'\uffff'之间的所有字符的缩略编码就按照普通UTF-8编码规则使用三个字节表示。</u>

​	<u>顺便提一下，由于Class文件中方法、字段等都需要引用CONSTANT_Utf8_info型常量来描述名称，所以CONSTANT_Utf8_info型常量的最大长度也就是Java中方法、字段名的最大长度。而这里的最大长度就是length的最大值，既u2类型能表达的最大值65535。所以Java程序中如果定义了超过64KB英文字符的变量或方法名，即使规则和全部字符都是合法的，也会无法编译</u>。

​	本例中这个字符串的length值（偏移地址：0x0000000E）为0x001D，也就是长29个字节，往后29个字节正好都在1～127的ASCII码范围以内，内容为“org/fenixsoft/clazz/TestClass”，有兴趣的读者可以自己逐个字节换算一下，换算结果如图6-4中选中的部分所示。

![这里写图片描述](https://img-blog.csdn.net/20170801170511863?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	到此为止，我们仅仅分析了TestClass.class常量池中21个常量中的两个，还未提到的其余19个常量都可以通过类似的方法逐一计算出来，为了避免计算过程占用过多的版面篇幅，后续的19个常量的计算过程就不手工去做了，而借助计算机软件来帮忙完成。在JDK的bin目录中，Oracle公司已经为我们准备好一个专门用于分析Class文件字节码的工具：javap。代码清单6-2中列出了<u>使用javap工具的-verbose参数输出的TestClass.class文件字节码内容</u>（为节省篇幅，此清单中省略了常量池以外的信息）。笔者曾经提到过Class文件中还有很多数据项都要引用常量池中的常量，建议读者不妨在本页做个记号，因为代码清单6-2中的内容在后续的讲解之中会频繁使用到。

​	代码清单6-2　使用javap命令输出常量表

```java
C:\>javap -verbose TestClass
Compiled from "TestClass.java"
public class org.fenixsoft.clazz.TestClass extends java.lang.Object
SourceFile: "TestClass.java"
minor version: 0
major version: 50
Constant pool:
const #1 = class #2; // org/fenixsoft/clazz/TestClass
const #2 = Asciz org/fenixsoft/clazz/TestClass;
const #3 = class #4; // java/lang/Object
const #4 = Asciz java/lang/Object;
const #5 = Asciz m;
const #6 = Asciz I;
const #7 = Asciz <init>;
const #8 = Asciz ()V;
const #9 = Asciz Code;
const #10 = Method #3.#11; // java/lang/Object."<init>":()V
const #11 = NameAndType #7:#8;// "<init>":()V
const #12 = Asciz LineNumberTable;
const #13 = Asciz LocalVariableTable;
const #14 = Asciz this;
const #15 = Asciz Lorg/fenixsoft/clazz/TestClass;;
const #16 = Asciz inc;
const #17 = Asciz ()I;
const #18 = Field #1.#19; // org/fenixsoft/clazz/TestClass.m:I
const #19 = NameAndType #5:#6; // m:I
const #20 = Asciz SourceFile;
const #21 = Asciz TestClass.java;
```

​	从代码清单6-2中可以看到，计算机已经帮我们把整个常量池的21项常量都计算了出来，并且第1、2项常量的计算结果与我们手工计算的结果完全一致。仔细看一下会发现，其中有些常量似乎从来没有在代码中出现过，如“I”“V”“\<init\>”“LineNumberTable”“LocalVariableTable”等，这些看起来在源代码中不存在的常量是哪里来的？

​	这部分常量的确不来源于Java源代码，它们都是编译器自动生成的，会被后面即将讲到的**<u>字段表（field_info）、方法表（method_info）、属性表（attribute_info）</u>**所引用，它们将会被用来描述一些不方便使用“固定字节”进行表达的内容，譬如描述方法的返回值是什么，有几个参数，每个参数的类型是什么。因为Java中的“类”是无穷无尽的，无法通过简单的无符号数来描述一个方法用到了什么类，因此在描述方法的这些信息时，需要引用常量表中的符号引用进行表达。这部分内容将在后面进一步详细阐述。最后，笔者将17种常量项的结构定义总结为表6-6。

​	表6-6　常量池中的17种数据类型的结构总表

<table>
  <tr>
  	<th>常量</th>
    <th>项目</th>
    <th>类型</th>
    <th>描述</th>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_Utf8_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为1</td>
  </tr>
  <tr>
    <td>length</td>
    <td>u2</td>
    <td>UTF-8编码的字符串占用了字节数</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u1</td>
    <td>长度为length的UTF-8编码的字符串</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_Utf8_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为1</td>
  </tr>
  <tr>
    <td>length</td>
    <td>u2</td>
    <td>UTF-8编码的字符串占用了字节数</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u1</td>
    <td>长度为length的UTF-8编码的字符串</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Integer_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为3</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u4</td>
    <td>按照高位在前存储的int值</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Float_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为4</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u4</td>
    <td>按照高位在前存储的float值</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Long_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为5</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u8</td>
    <td>按照高位在前存储的long值</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Double_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为6</td>
  </tr>
  <tr>
    <td>bytes</td>
    <td>u8</td>
    <td>按照高位在前存储的double值</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Class_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为7</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向全限定名常量项的索引</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_String_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为8</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向字符串字面量的索引</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_Fieldref_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为9</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向声明字段的类或者接口描述符CONSTANT_Class_info的索引项</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向字段描述符CONSTANT_NameAndType的索引项</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_Methodref_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为10</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向声明方法的类描述符CONSTANT_Class_info的索引项</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向名称及类型描述符CONSTANT_NameAndType的索引项</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_InterfaceMethodref_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为11</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向声明方法的接口描述符CONSTANT_Class_info的索引项</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向名称及类型描述符CONSTANT_NameAndType的索引项</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_NameAndType_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为12</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向该字段或方法名称常量项的索引</td>
  </tr>
  <tr>
    <td>index</td>
    <td>u2</td>
    <td>指向该字段或方法描述符常量项的索引</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_MethodHandle_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为15</td>
  </tr>
  <tr>
    <td>reference_kind</td>
    <td>u1</td>
    <td>值必须在1至9之间(包括1和9)，它决定了方法句柄的类型。方法句柄类型的值表示方法句柄的字节码行为</td>
  </tr>
  <tr>
    <td>reference_index</td>
    <td>u2</td>
    <td>值必须是对常量池的有效索引</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_MethodType_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为16</td>
  </tr>
  <tr>
    <td>descriptor_index</td>
    <td>u2</td>
    <td>值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符</td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_Dynamic_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为17</td>
  </tr>
  <tr>
    <td>bootstrap_method_attr_index</td>
    <td>u2</td>
    <td>值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引</td>
  </tr>
  <tr>
    <td>name_and_type_index</td>
    <td>u2</td>
    <td>值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符/td>
  </tr>
  <tr>
  	<td rowspan="3">CONSTANT_InvokeDynamic_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为18</td>
  </tr>
  <tr>
    <td>bootstrap_method_attr_index</td>
    <td>u2</td>
    <td>值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引</td>
  </tr>
  <tr>
    <td>name_and_type_index</td>
    <td>u2</td>
    <td>值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符/td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Module_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为19</td>
  </tr>
  <tr>
    <td>name_index</td>
    <td>u2</td>
    <td>值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示模块名字</td>
  </tr>
  <tr>
  	<td rowspan="2">CONSTANT_Package_info</td>
    <td>tag</td>
    <td>u1</td>
    <td>值为20</td>
  </tr>
  <tr>
    <td>name_index</td>
    <td>u2</td>
    <td>值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示包名称</td>
  </tr>
</table>

[^5]:JDK 7时增加了前三种：CONSTANT_MethodHandle_info、CONSTANT_MethodType_info和CONSTANT_InvokeDynamic_info。出于性能和易用性的考虑（JDK 7设计时已经考虑到，预留了17个常量标志位），在JDK 11中又增加了第四种常量CONSTANT_Dynamic_info。本章不会涉及这4种新增的类型，留待第8章介绍字节码执行和方法调用时详细讲解。

### 6.3.3 访问标志

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	在常量池结束之后，紧接着的2个字节代表访问标志（access_flags），这个标志用于识别一些类或者接口层次的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final；等等。具体的标志位以及标志的含义见表6-7。

​	表6-7　访问标志

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 是否为public类型                                             |
| ACC_FINAL      | 0x0010 | 是否被声明为final，只有类可设置                              |
| ACC_SUPER      | 0x0020 | 是否允许使用invokespecial字节码指令的新语义，invokespecial指令的语义在JDK1.0.2发生过改变，为了区别这条指令使用哪种语义，JDK1.0.2之后编译出来的类的这个标志都必须为真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否为abstarct类型，对于接口或者抽象类来说，此标志值为真，其他类型值为假 |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类并非由用户代码产生的                               |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                             |
| ACC_ENUM       | 0x4000 | 标识这是一个枚举                                             |
| ACC_MODULE     | 0x8000 | 标识这是一个模块                                             |

​	access_flags中一共有16个标志位可以使用，当前只定义了其中9个[^6]，没有使用到的标志位要求一律为零。以代码清单6-1中的代码为例，TestClass是一个普通Java类，不是接口、枚举、注解或者模块，被public关键字修饰但没有被声明为final和abstract，并且它使用了JDK 1.2之后的编译器进行编译，因此它的ACC_PUBLIC、ACC_SUPER标志应当为真，而ACC_FINAL、ACC_INTERFACE、ACC_ABSTRACT、ACC_SYNTHETIC、ACC_ANNOTATION、ACC_ENUM、ACC_MODULE这七个标志应当为假，因此它的access_flags的值应为：0x0001|0x0020=0x0021。从图6-5中看到，access_flags标志（偏移地址：0x000000EF）的确为0x0021。

![这里写图片描述](https://img-blog.csdn.net/20170801171638177?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

[^6]:在原始的《Java虚拟机规范》初版中，只定义了开头5种标志。JDK 5中增加了后续3种。这些标志为在JSR-202规范之中声明，是对《Java虚拟机规范》第2版的补充。JDK 9发布之后，增加了第9种。

### 6.3.4 类索引、父类索引与接口索引集合

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	**类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，而接口索引集合（interfaces）是一组u2类型的数据的集合，Class文件中由这三项数据来确定该类型的继承关系**。

​	**类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名**。由于Java语言不允许多重继承，所以父类索引只有一个，<u>除了java.lang.Object之外，所有的Java类都有父类，因此除了java.lang.Object外，所有Java类的父类索引都不为0</u>。接口索引集合就用来描述这个类实现了哪些接口，这些被实现的接口将按implements关键字（如果这个Class文件表示的是一个接口，则应当是extends关键字）后的接口顺序从左到右排列在接口索引集合中。

​	<u>类索引、父类索引和接口索引集合都按顺序排列在访问标志之后，类索引和父类索引用两个u2类型的索引值表示，它们各自指向一个类型为CONSTANT_Class_info的类描述符常量，通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串</u>。图6-6演示了代码清单6-1中代码的类索引查找过程。

![这里写图片描述](https://img-blog.csdn.net/20170801172046574?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	**对于接口索引集合，入口的第一项u2类型的数据为接口计数器（interfaces_count），表示索引表的容量**。如果该类没有实现任何接口，则该计数器值为0，后面接口的索引表不再占用任何字节。代码清单6-1中的代码的类索引、父类索引与接口表索引的内容如图6-7所示。

​	从偏移地址0x000000F1开始的3个u2类型的值分别为0x0001、0x0003、0x0000，也就是类索引为1，父类索引为3，接口索引集合大小为0。查询前面代码清单6-2中javap命令计算出来的常量池，找出对应的类和父类的常量，结果如代码清单6-3所示。

​	代码清单6-3　部分常量池内容

```java
const #1 = class #2; // org/fenixsoft/clazz/TestClass
const #2 = Asciz org/fenixsoft/clazz/TestClass;
const #3 = class #4; // java/lang/Object
const #4 = Asciz java/lang/Object;
```

### 6.3.5 字段表集合

个人小结：

+ <u>在Java语言中字段是无法重载的，两个字段的数据类型、修饰符不管是否相同，都必须使用不一样的名称，但是对于Class文件格式来讲，只要两个字段的描述符不是完全相同，那字段重名就是合法的</u>

---

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	**字段表（field_info）用于描述接口或者类中声明的变量**。<u>Java语言中的“字段”（Field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量</u>。读者可以回忆一下在Java语言中描述一个字段可以包含哪些信息。字段可以包括的修饰符有字段的作用域（public、private、protected修饰符）、是实例变量还是类变量（static修饰符）、可变性（final）、并发可见性（volatile修饰符，是否强制从主内存读写）、可否被序列化（transient修饰符）、字段数据类型（基本类型、对象、数组）、字段名称。上述这些信息中，<u>各个修饰符都是布尔值，要么有某个修饰符，要么没有，很适合使用标志位来表示。而字段叫做什么名字、字段被定义为什么数据类型，这些都是无法固定的，只能引用常量池中的常量来描述</u>。表6-8中列出了字段表的最终格式。

​	表6-8　字段表结构

| 类型 | 名称             | 数量 | 类型           | 名称       | 数量             |
| ---- | ---------------- | ---- | -------------- | ---------- | ---------------- |
| u2   | access_flags     | 1    | u2             | attributes | 1                |
| u2   | name_index       | 1    | Attribute_info | attributes | Attributes_count |
| u2   | Descriptor_index | 1    |                |            |                  |

​	字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型，其中可以设置的标志位和含义，如表6-9所示。

​	表6-9　字段访问标志

| 标志名称      | 标志值 | 含义              | 标志名称   | 标志值 | 含义                     |
| ------------- | ------ | ----------------- | ---------- | ------ | ------------------------ |
| ACC_PUBLIC    | 0x0001 | 字段是否public    | ACC_PUBLIC | 0x0040 | 字段是否volatile         |
| ACC_PRIVATE   | 0x0002 | 字段是否private   | ACC_PUBLIC | 0x0080 | 字段是否transient        |
| ACC_PROTECTED | 0x0004 | 字段是否protected | ACC_PUBLIC | 0x1000 | 字段是否由编译器自动产生 |
| ACC_STATIC    | 0x0008 | 字段是否static    | ACC_PUBLIC | 0x4000 | 字段是否enum             |
| ACC_FINAL     | 0x0010 | 字段是否final     |            |        |                          |

​	很明显，由于语法规则的约束，ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED三个标志最多只能选择其一，ACC_FINAL、ACC_VOLATILE不能同时选择。接口之中的字段必须有ACC_PUBLIC、ACC_STATIC、ACC_FINAL标志，这些都是由Java本身的语言规则所导致的。

​	跟随access_flags标志的是两项索引值：name_index和descriptor_index。它们都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符。现在需要解释一下“简单名称”“描述符”以及前面出现过多次的“全限定名”这三种特殊字符串的概念。

​	全限定名和简单名称很好理解，以代码清单6-1中的代码为例，“org/fenixsoft/clazz/TestClass”是这个类的全限定名，仅仅是把类全名中的“.”替换成了“/”而已，为了使连续的多个全限定名之间不产生混淆，在使用时最后一般会加入一个“；”号表示全限定名结束。<u>简单名称则就是指没有类型和参数修饰的方法或者字段名称</u>，这个类中的inc()方法和m字段的简单名称分别就是“inc”和“m”。

​	相比于全限定名和简单名称，方法和字段的描述符就要复杂一些。<u>描述符的作用是用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值</u>。**根据描述符规则，基本数据类型（byte、char、double、float、int、long、short、boolean）以及代表无返回值的void类型都用一个大写字符来表示，而对象类型则用字符L加对象的全限定名来表示**，详见表6-10。

​	表6-10　描述符标识字符含义

| 标识字符 | 含义           | 标识字符       | 含义                            |
| -------- | -------------- | -------------- | ------------------------------- |
| B        | 基本类型byte   | J              | 基本类型long                    |
| C        | 基本类型char   | S              | 基本类型short                   |
| D        | 基本类型double | Z              | 基本类型boolean                 |
| F        | 基本类型float  | V<sup>㊀</sup> | 特殊类型void                    |
| I        | 基本类型int    | L              | 对象类型，如Ljava/lang/Object； |

​	*注：void类型在《Java虚拟机规范》之中单独列出为“VoidDescriptor”，笔者为了结构统一，将其列在基本数据类型中一起描述。*

​	对于数组类型，每一维度将使用一个前置的“[”字符来描述，如一个定义为“java.lang.String[][]”类型的二维数组将被记录成“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录成“[I”。

​	**用描述符来描述方法时，按照先参数列表、后返回值的顺序描述**，参数列表按照参数的严格顺序放在一组小括号“()”之内。如方法void inc()的描述符为“()V”，方法java.lang.String toString()的描述符为“()Ljava/lang/String；”，方法int indexOf(char[]source，int sourceOffset，int sourceCount，char[]target，int targetOffset，int targetCount，int fromIndex)的描述符为“([CII[CIII)I”。

​	对于代码清单6-1所编译的TestClass.class文件来说，字段表集合从地址0x000000F8开始，第一个u2类型的数据为容量计数器fields_count，如图6-8所示，其值为0x0001，说明这个类只有一个字段表数据。接下来紧跟着容量计数器的是access_flags标志，值为0x0002，代表private修饰符的ACC_PRIVATE标志位为真（ACC_PRIVATE标志的值为0x0002），其他修饰符为假。代表字段名称的name_index的值为0x0005，从代码清单6-2列出的常量表中可查得第五项常量是一个CONSTANT_Utf8_info类型的字符串，其值为“m”，代表字段描述符的descriptor_index的值为0x0006，指向常量池的字符串“I”。根据这些信息，我们可以推断出原代码定义的字段为“private int m；”。

![这里写图片描述](https://img-blog.csdn.net/20170802091413576?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	字段表所包含的固定数据项目到descriptor_index为止就全部结束了，不过在descrip-tor_index之后跟随着一个属性表集合，用于存储一些额外的信息，字段表可以在属性表中附加描述零至多项的额外信息。对于本例中的字段m，它的属性表计数器为0，也就是没有需要额外描述的信息，但是，如果将字段m的声明改为“final static int m=123；”，那就可能会存在一项名称为ConstantValue的属性，其值指向常量123。关于attribute_info的其他内容，将在6.3.7节介绍属性表的数据项目时再做进一步讲解。

​	**字段表集合中不会列出从父类或者父接口中继承而来的字段**，<u>但有可能出现原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，编译器就会自动添加指向外部类实例的字段</u>。另外，<u>在Java语言中字段是无法重载的，两个字段的数据类型、修饰符不管是否相同，都必须使用不一样的名称，但是对于Class文件格式来讲，只要两个字段的描述符不是完全相同，那字段重名就是合法的</u>。

### 6.3.6 方法表集合

个人小结：

+ 在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名[^8]。特征签名是指一个方法中各个参数在常量池中的字段符号引用的集合，也正是因为返回值不会包含在特征签名之中，所以Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的。
+ 但是在Class文件格式之中，特征签名的范围明显要更大一些，只要描述符不是完全一致的两个方法就可以共存。也就是说，如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件中的。
+ **对于每一个属性，它的名称都要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可**

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	如果理解了上一节关于字段表的内容，那本节关于方法表的内容将会变得很简单。Class文件存储格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，方法表的结构如同字段表一样，依次包括访问**标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）**几项，如表6-11所示。这些数据项目的含义也与字段表中的非常类似，仅在访问标志和属性表集合的可选项中有所区别。

​	表6-11　方法表结构

| 类型 | 名称             | 数量 | 类型           | 名称             | 数量             |
| ---- | ---------------- | ---- | -------------- | ---------------- | ---------------- |
| u2   | access_flags     | 1    | u2             | attributes_count | 1                |
| u2   | name_index       | 1    | attribute_info | attributes       | attributes_count |
| u2   | descriptor_index | 1    |                |                  |                  |

​	因为volatile关键字和transient关键字不能修饰方法，所以方法表的访问标志中没有了ACC_VOLATILE标志和ACC_TRANSIENT标志。与之相对，synchronized、native、strictfp和abstract关键字可以修饰方法，方法表的访问标志中也相应地增加了ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志。对于方法表，所有标志位及其取值可参见表6-12。

​	表6-12　方法访问标志

| 标志名称         | 标志值 | 含义                             |
| ---------------- | ------ | -------------------------------- |
| ACC_PUBLIC       | 0x0001 | 方法是否为public                 |
| ACC_PRIVATE      | 0x0002 | 方法是否为private                |
| ACC_PROTECTED    | 0x0004 | 方法是否为protected              |
| ACC_STATIC       | 0x0008 | 方法是否为static                 |
| ACC_FINAL        | 0x0010 | 方法是否为final                  |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否为synchronized           |
| ACC_BRIDGE       | 0x0040 | 方法是不是由编译器产生的桥接方法 |
| ACC_VARARGS      | 0x0080 | 方法是否接受不定参数             |
| ACC_NATIVE       | 0x0100 | 方法是否为native                 |
| ACC_ABSTRACT     | 0x0400 | 方法是否为abstract               |
| ACC_STRICT       | 0x0800 | 方法是否为strictfp               |
| ACC_SYNTHETIC    | 0x1000 | 方法是否由编译器自动产生         |

​	行文至此，也许有的读者会产生疑问，方法的定义可以通过访问标志、名称索引、描述符索引来表达清楚，但方法里面的代码去哪里了？**方法里的Java代码，经过Javac编译器编译成字节码指令之后，存放在方法属性表集合中一个名为“Code”的属性里面**，属性表作为Class文件格式中最具扩展性的一种数据项目，将在下一节中详细讲解。

​	我们继续以代码清单6-1中的Class文件为例对方法表集合进行分析。如图6-9所示，方法表集合的入口地址为0x00000101，第一个u2类型的数据（即计数器容量）的值为0x0002，代表集合中有两个方法，这两个方法为编译器添加的实例构造器\<init\>和源码中定义的方法inc()。第一个方法的访问标志值为0x0001，也就是只有ACC_PUBLIC标志为真，名称索引值为0x0007，查代码清单6-2的常量池得方法名为“\<init\>”，描述符索引值为0x0008，对应常量为“()V”，属性表计数器attributes_count的值为0x0001，表示此方法的属性表集合有1项属性，属性名称的索引值为0x0009，对应常量为“Code”，说明此属性是方法的字节码描述。

![这里写图片描述](https://img-blog.csdn.net/20170802092333403?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	与字段表集合相对应地，如果父类方法在子类中没有被重写（Override），方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器“\<clinit\>()”方法和实例构造器“\<init\>()”方法[^7]。

​	**在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名[^8]。特征签名是指一个方法中各个参数在常量池中的字段符号引用的集合，也正是因为返回值不会包含在特征签名之中，所以Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的**。<u>但是在Class文件格式之中，特征签名的范围明显要更大一些，只要描述符不是完全一致的两个方法就可以共存。也就是说，如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件中的</u>。

[^7]:＜init＞()和＜clinit＞()的详细内容见本书的下一部分“前端编译与优化”。
[^8]:在《Java虚拟机规范》第2版的4.4.4节及《Java语言规范》第3版的8.4.2节中分别都定义了字节码层面的方法特征签名以及Java代码层面的方法特征签名，Java代码的方法特征签名只包括方法名称、参数顺序及参数类型，而字节码的特征签名还包括方法返回值以及受查异常表，请读者根据上下文语境注意区分。

### 6.3.7 属性表集合

个人小结：

+ 《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性。
+ **Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性内**。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性
+ 关于code_length，有一件值得注意的事情，虽然它是一个u4类型的长度值，理论上最大值可以达到2的32次幂，但是《Java虚拟机规范》中明确限制了一个方法不允许超过65535条字节码指令，即它实际只使用了u2的长度，如果超过这个限制，Javac编译器就会拒绝编译。一般来讲，编写Java代码时只要不是刻意去编写一个超级长的方法来为难编译器，是不太可能超过这个最大值的限制的。但是，某些特殊情况，例如在编译一个很复杂的JSP文件时，某些JSP编译器会把JSP内容和页面输出的信息归并于一个方法之中，就有可能因为方法生成字节码超长的原因而导致编译失败
+ **Java语言里面的潜规则：在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象**。这个访问机制对Java程序的编写很重要，而它的实现非常简单，仅仅是通过在Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数而已。因此在实例方法的局部变量表中至少会存在一个指向当前对象实例的局部变量，局部变量表中也会预留出第一个变量槽位来存放对象实例的引用，所以实例方法参数值从1开始计算。这个处理只对实例方法有效。
+ 异常表实际上是Java代码的一部分，尽管字节码中有最初为处理异常而设计的跳转指令，但《Java虚拟机规范》中明确要求Java语言的编译器应当选择使用异常表而不是通过跳转指令来实现Java异常及finally处理机制[^10]
+ 虽然有final关键字才更符合“ConstantValue”的语义，但《Java虚拟机规范》中并没有强制要求字段必须设置ACC_FINAL标志，只要求有ConstantValue属性的字段必须设置ACC_STATIC标志而已，对final关键字的要求是Javac编译器自己加入的限制
+ 在JDK 5里面大幅增强了Java语言的语法，在此之后，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（ParameterizedType），则Signature属性会为它记录泛型签名信息。**之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是擦除法实现的伪泛型，字节码（Code属性）中所有的泛型信息编译（类型变量、参数化类型）在编译之后都通通被擦除掉**。使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。但坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同等对待，例如运行期做反射时无法获得泛型信息。Signature属性就是为了弥补这个缺陷而增设的，现在Java的反射API能够获取的泛型类型，最终的数据来源也是这个属性。
+ JDK 9的一个重量级功能是Java的模块化功能，因为模块描述文件（module-info.java）最终是要编译成一个独立的Class文件来存储的，所以，Class文件格式也扩展了Module、ModulePackages和ModuleMainClass三个属性用于支持Java模块化相关功能

---

> [《深入理解Java虚拟机》第6章 类文件结构](https://blog.csdn.net/huaxun66/article/details/76541493?utm_source=blogxgwz0)

​	属性表（attribute_info）在前面的讲解之中已经出现过数次，Class文件、字段表、方法表都可以携带自己的属性表集合，以描述某些场景专有的信息。

​	与Class文件中其他的数据项目要求严格的顺序、长度和内容不同，属性表集合的限制稍微宽松一些，不再要求各个属性表具有严格顺序，并且**《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性**。为了能正确解析Class文件，《Java虚拟机规范》最初只预定义了9项所有Java虚拟机实现都应当能识别的属性，而在最新的《Java虚拟机规范》的Java SE 12版本中，预定义属性已经增加到29项，这些属性具体见表6-13。后文中将对这些属性中的关键的、常用的部分进行讲解。

​	表6-13　虚拟机规范预定义的属性

| 属性名称                             | 使用位置                     | 含义                                                         |
| ------------------------------------ | ---------------------------- | ------------------------------------------------------------ |
| Code                                 | 方发表                       | Java代码编译成的字节码指令                                   |
| ConstantValue                        | 字段表                       | 由final关键字定义的常量值                                    |
| Deprecated                           | 类、方法表、字段表           | 被声明为deprecated的方法和字段                               |
| Exceptions                           | 方法表                       | 方法抛出的异常列表                                           |
| EnclosingMethod                      | 类文件                       | 仅当一个类为局部类或者匿名类时才能拥有这个属性，这个属性用于标示这个类所在的外围方法 |
| InnerClasses                         | 类文件                       | 内部类列表                                                   |
| LineNumberTable                      | Code属性                     | Java源码的行号与字节码指令的对应关系                         |
| LocalNumberTable                     | Code属性                     | 方法的局部变量描述                                           |
| StackMapTable                        | Code属性                     | JDK 6 中新增的属性，供新的类型检查验证器(Type Checker)检查和处理目标的局部变量和操作数栈的类型是否匹配 |
| Signature                            | 类、方法表、字段表           | JDK 5 中新增的属性，用于支持泛型情况下的方法签名。在Java语言中，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variables）或参数化类型（Parameterized Types），则Signature属性会为它记录泛型签名信息。由于Java的泛型采用擦除法实现，为了避免类型信息被擦除后导致签名混乱，需要这个属性记录泛型中的相关信息 |
| SourceFile                           | 类文件                       | 记录源文件名称                                               |
| SourceDebugExtension                 | 类文件                       | JDK 5 中新增的属性，用于存储额外的调试信息。譬如在进行JSP文件调试时，无法通过Java堆栈来定位到JSP文件的行号，JSR 45 提案为这些非Java语言编写，却需要编译成字节码并运行在Java虚拟机中的程序提供了一个进行调试的标准机制，使用该属性就可以用于存储这个标准所新加入的调试信息 |
| Synthetic                            | 类、方法表、字段表           | 标识方法或字段为编译器自动生成的                             |
| LocalVariableTypeTable               | 类                           | JDK 5 中新增的属性，它使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加 |
| RuntimeVisibleAnnotations            | 类、方法表、字段表           | JDK 5 中新增的属性，为动态注解提供支持。该属性用于指明哪些注解是运行时（实际上运行时就是进行反射调用）可见的 |
| RuntimeInvisibleAnnotations          | 类、方法表、字段表           | JDK 5 中新增的属性，与RuntimeVisibleAnnotations属性作用刚好相反，用于指明哪些注解是运行时不可见的 |
| RuntimeVisibleParameterAnnotations   | 方法表                       | JDK 5 中新增的属性，作用与RuntimeVisibleAnnotations属性类似，只不过作用对象为方法参数 |
| RuntimeInvisibleParameterAnnotations | 方法表                       | JDK 5 中新增的属性，作用与RuntimeInvisibleAnnotations属性类似，只不过作用对象为方法参数 |
| AnnotationDefault                    | 方法表                       | JDK 5 中新增的属性，用于记录注解类元素的默认值               |
| BootstrapMethods                     | 类文件                       | JDK 7 中新增的属性，用于保存invokedynamic指令引用的引导方法限定符 |
| RuntimeVisibleTypeAnnotations        | 类、方法表、字段表、Code属性 | JDK 8 中新增的属性，为实现JSR 308 中新增的类型注解提供的支持，用于指明哪些类注解是运行时（实际上运行时就是进行反射调用）可见的 |
| RuntimeInvisibleTypeAnnotations      | 类、方法表、字段表、Code属性 | JDK 8 中新增的属性，为实现JSR 308 中新增的类型注解提供的支持，与RuntimeVisibleTypeAnnotations属性作用刚好相反，用于指明哪些注解是运行时不可见的 |
| MethodParameters                     | 方法表                       | JDK 8 中新增的属性，用于支持（编译时加上-parameters参数）将方法名称编译进Class文件中，并可运行时获取。此前要获取方法名称（典型的如IDE的代码提示）只能通过JavaDoc中得到 |
| Module                               | 类                           | JDK 9 中新增的属性，用于记录一个Module的名称以及相关信息（requires、exports、opens、uses、provides） |
| ModulePackages                       | 类                           | JDK 9 中新增的属性，用于记录一个模块中所有被exports或者opens的包 |
| ModuleMainClass                      | 类                           | JDK 9 中新增的属性，用于指定一个模块的主类                   |
| NestHost                             | 类                           | JDK 11 中新增的属性，用于支持嵌套类（Java中的内部类）的反射和访问控制的API，一个内部类通过属性得知自己的宿主类 |
| NestMembers                          | 类                           | JDK 11 中新增的属性，用于支持嵌套类（Java中的内部类）的反射和访问控制的API，一个宿主类通过该属性得知自己有哪些内部类 |

​	**对于每一个属性，它的名称都要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可**。一个符合规则的属性表应该满足表6-14中所定义的结构。

​	表6-14　属性表结构

| 类型 | 名称                 | 数量             |
| ---- | -------------------- | ---------------- |
| u2   | attribute_name_index | 1                |
| u4   | attribute_length     | 1                |
| u1   | info                 | attribute_length |

#### 1. Code属性

​	**Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性**，如果方法表有Code属性存在，那么它的结构将如表6-15所示。

​	表6-15　Code属性表的结构

| 类型           | 名称                   | 数量                   |
| -------------- | ---------------------- | ---------------------- |
| u2             | attribute_name_index   | 1                      |
| u4             | attribute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_table        | exception_table_length |
| u2             | attributes_count       | 1                      |
| attribute_info | attributes             | attributes_count       |

​	attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为“Code”，它代表了该属性的属性名称，attribute_length指示了属性值的长度，由于属性名称索引与属性长度一共为6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。

​	**max_stack代表了操作数栈（Operand Stack）深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个深度。虚拟机运行的时候需要根据这个值来分配栈帧（Stack Frame）中的操作栈深度**。

​	max_locals代表了局部变量表所需的存储空间。在这里，max_locals的单位是变量槽（Slot），变量槽是虚拟机为局部变量分配内存所使用的最小单位。对于byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用一个变量槽，而double和long这两种64位的数据类型则需要两个变量槽来存放。方法参数（包括实例方法中的隐藏参数“this”）、显式异常处理程序的参数（Exception Handler Parameter，就是try-catch语句中catch块中所定义的异常）、方法体中定义的局部变量都需要依赖局部变量表来存放。注意，并不是在方法中用了多少个局部变量，就把这些局部变量所占变量槽数量之和作为max_locals的值，操作数栈和局部变量表直接决定一个该方法的栈帧所耗费的内存，不必要的操作数栈深度和变量槽数量会造成内存的浪费。**Java虚拟机的做法是将局部变量表中的变量槽进行重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占的变量槽可以被其他局部变量所使用，Javac编译器会根据变量的作用域来分配变量槽给各个变量使用，根据同时生存的最大局部变量数量和类型计算出max_locals的大小**。

​	code_length和code用来存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code是用于存储字节码指令的一系列字节流。既然叫字节码指令，那顾名思义每个指令就是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应找出这个字节码代表的是什么指令，并且可以知道这条指令后面是否需要跟随参数，以及后续的参数应当如何解析。我们知道一个u1数据类型的取值范围为0x00～0xFF，对应十进制的0～255，也就是一共可以表达256条指令。目前，《Java虚拟机规范》已经定义了其中约200条编码值对应的指令含义，编码与指令之间的对应关系可查阅本书的附录C“虚拟机字节码指令表”。

​	**关于code_length，有一件值得注意的事情，虽然它是一个u4类型的长度值，理论上最大值可以达到2的32次幂，但是《Java虚拟机规范》中明确限制了一个方法不允许超过65535条字节码指令，即它实际只使用了u2的长度，如果超过这个限制，Javac编译器就会拒绝编译。一般来讲，编写Java代码时只要不是刻意去编写一个超级长的方法来为难编译器，是不太可能超过这个最大值的限制的。但是，某些特殊情况，例如在编译一个很复杂的JSP文件时，某些JSP编译器会把JSP内容和页面输出的信息归并于一个方法之中，就有可能因为方法生成字节码超长的原因而导致编译失败**。

​	Code属性是Class文件中最重要的一个属性，如果把一个Java程序中的信息分为代码（Code，方法体里面的Java代码）和元数据（Metadata，包括类、字段、方法定义及其他信息）两部分，那么在整个Class文件里，Code属性用于描述代码，所有的其他数据项目都用于描述元数据。了解Code属性是学习后面两章关于字节码执行引擎内容的必要基础，能直接阅读字节码也是工作中分析Java代码语义问题的必要工具和基本技能，为此，笔者准备了一个比较详细的实例来讲解虚拟机是如何使用这个属性的。

![这里写图片描述](https://img-blog.csdn.net/20170802095243330?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	继续以代码清单6-1的TestClass.class文件为例，如图6-10所示，这是上一节分析过的实例构造器“\<init\>()”方法的Code属性。它的操作数栈的最大深度和本地变量表的容量都为0x0001，字节码区域所占空间的长度为0x0005。虚拟机读取到字节码区域的长度后，按照顺序依次读入紧随的5个字节，并根据字节码指令表翻译出所对应的字节码指令。翻译“2A B7 00 0A B1”的过程为：

1. 读入2A，查表得0x2A对应的指令为aload_0，这个指令的含义是将第0个变量槽中为reference类型的本地变量推送到操作数栈顶。

2. 读入B7，查表得0xB7对应的指令为invokespecial，这条指令的作用是以栈顶的reference类型的数据所指向的对象作为方法接收者，调用此对象的实例构造器方法、private方法或者它的父类的方法。这个方法有一个u2类型的参数说明具体调用哪一个方法，它指向常量池中的一个CONSTANT_Methodref_info类型常量，即此方法的符号引用。

3. 读入000A，这是invokespecial指令的参数，代表一个符号引用，查常量池得0x000A对应的常量为实例构造器“\<init\>()”方法的符号引用。
4. 读入B1，查表得0xB1对应的指令为return，含义是从方法的返回，并且返回值为void。这条指令执行后，当前方法正常结束。

​	这段字节码虽然很短，但我们可以从中看出它<u>执行过程中的数据交换、方法调用等操作都是基于栈（操作数栈）的</u>。我们可以初步猜测，Java虚拟机执行字节码应该是基于栈的体系结构。但又发现与通常基于栈的指令集里都是无参数的又不太一样，某些指令（如invokespecial）后面还会带有参数，关于虚拟机字节码执行的讲解是后面两章的话题，我们不妨把这里的疑问放到第8章去解决。

​	我们再次使用javap命令把此Class文件中的另一个方法的字节码指令也计算出来，结果如代码清单6-4所示。

​	代码清单6-4　用Javap命令计算字节码指令

```java
// 原始Java代码
public class TestClass {
  private int m;
  public int inc() {
    return m + 1;
  }
}
C:\>javap -verbose TestClass
  // 常量表部分的输出见代码清单6-1，因版面原因这里省略掉
{
 public org.fenixsoft.clazz.TestClass();
 	 Code:
	 	Stack=1, Locals=1, Args_size=1
   	0: aload_0
    1: invokespecial #10; //Method java/lang/Object."<init>":()V
 		4: return
   LineNumberTable:
 		line 3: 0
   LocalVariableTable:
 		Start Length Slot Name Signature
   	0 		5 			0 	this Lorg/fenixsoft/clazz/TestClass;
 public int inc();
 		Code:
 			Stack=2, Locals=1, Args_size=1
   		0: aload_0
     	1: getfield #18; //Field m:I
 			4: iconst_1
  	  5: iadd
     	6: ireturn
 		LineNumberTable:
 			line 8: 0
   	LocalVariableTable:
 			Start Length Slot Name Signature
   		0 		7 		 0 		this Lorg/fenixsoft/clazz/TestClass;
}
```

​	如果大家注意到javap中输出的“Args_size”的值，可能还会有疑问：这个类有两个方法——实例构造器\<init\>()和inc()，这两个方法很明显都是没有参数的，为什么Args_size会为1？而且无论是在参数列表里还是方法体内，都没有定义任何局部变量，那Locals又为什么会等于1？如果有这样疑问的读者，大概是忽略了一条Java语言里面的潜规则：<u>**在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象。这个访问机制对Java程序的编写很重要，而它的实现非常简单，仅仅是通过在Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数而已。因此在实例方法的局部变量表中至少会存在一个指向当前对象实例的局部变量，局部变量表中也会预留出第一个变量槽位来存放对象实例的引用，所以实例方法参数值从1开始计算**。这个处理只对实例方法有效，如果代码清单6-1中的inc()方法被声明为static，那Args_size就不会等于1而是等于0了</u>。

​	在字节码指令之后的是这个方法的显式异常处理表（下文简称“异常表”）集合，异常表对于Code属性来说并不是必须存在的，如代码清单6-4中就没有异常表生成。

​	<u>如果存在异常表，那它的格式应如表6-16所示，包含四个字段，这些字段的含义为：如果当字节码从第start_pc行[^9]到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向一个CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转到handler_pc处进行处理</u>。

​	表6-16　属性表结构

| 类型 | 名称     | 数量 | 类型 | 名称       | 数量 |
| ---- | -------- | ---- | ---- | ---------- | ---- |
| u2   | start_pc | 1    | u2   | handler_pc | 1    |
| u2   | end_pc   | 1    | u2   | catch_type | 1    |

​	**异常表实际上是Java代码的一部分，尽管字节码中有最初为处理异常而设计的跳转指令，但《Java虚拟机规范》中明确要求Java语言的编译器应当选择使用异常表而不是通过跳转指令来实现Java异常及finally处理机制[^10]**。

​	代码清单6-5是一段演示异常表如何运作的例子，这段代码主要演示了在字节码层面try-catchfinally是如何体现的。阅读字节码之前，大家不妨先看看下面的Java源码，想一下这段代码的返回值在出现异常和不出现异常的情况下分别应该是多少？

​	代码清单6-5　异常表运作演示

```java
// Java源码
public int inc() {
  int x;
  try {
    x = 1;
    return x;
  } catch (Exception e) {
    x = 2;
    return x;
  } finally {
    x = 3;
  }
}
// 编译后的ByteCode字节码及异常表
public int inc();
		Code:
			Stack=1, Locals=5, Args_size=1
  		0: iconst_1 // try块中的x=1
   		1: istore_1
      2: iload_1 // 保存x到returnValue中，此时x=1
      3: istore 4
      5: iconst_3 // finaly块中的x=3
      6: istore_1
      7: iload 4 // 将returnValue中的值放到栈顶，准备给ireturn返回
      9: ireturn
      10: astore_2 // 给catch中定义的Exception e赋值，存储在变量槽 2中
      11: iconst_2 // catch块中的x=2
      12: istore_1
      13: iload_1 // 保存x到returnValue中，此时x=2
      14: istore 4
      16: iconst_3 // finaly块中的x=3
      17: istore_1
      18: iload 4 // 将returnValue中的值放到栈顶，准备给ireturn返回
      20: ireturn
      21: astore_3 // 如果出现了不属于java.lang.Exception及其子类的异常才会走到这里
      22: iconst_3 // finaly块中的x=3
      23: istore_1
      24: aload_3 // 将异常放置到栈顶，并抛出
      25: athrow
    Exception table:
		from to target type
  	0 		5 	10 		Class java/lang/Exception
  	0 		5 	21 		any
  	10 	 16 	21	  any
```

​	编译器为这段Java源码生成了三条异常表记录，对应三条可能出现的代码执行路径。从Java代码的语义上讲，这三条执行路径分别为：

+ 如果try语句块中出现属于Exception或其子类的异常，转到catch语句块处理；
+ 如果try语句块中出现不属于Exception或其子类的异常，转到finally语句块处理；
+ 如果catch语句块中出现任何异常，转到finally语句块处理。

​	返回到我们上面提出的问题，这段代码的返回值应该是多少？熟悉Java语言的读者应该很容易说出答案：如果没有出现异常，返回值是1；如果出现了Exception异常，返回值是2；如果出现了Exception以外的异常，方法非正常退出，没有返回值。我们一起来分析一下字节码的执行过程，从字节码的层面上看看为何会有这样的返回结果。

​	字节码中第0～4行所做的操作就是将整数1赋值给变量x，并且将此时x的值**复制一份副本**到最后一个**本地变量表的变量槽**中（这个变量槽里面的值在ireturn指令执行前将会被重新读到**操作栈顶**，作为方法返回值使用。为了讲解方便，笔者给这个变量槽起个名字：returnValue）。如果这时候没有出现异常，则会继续走到第5～9行，将变量x赋值为3，然后将之前保存在returnValue中的整数1读入到**操作栈顶**，最后ireturn指令会以int形式返回**操作栈顶中的值**，方法结束。如果出现了异常，**PC寄存器指针**转到第10行，第10～20行所做的事情是将2赋值给变量x，然后将变量x此时的值赋给returnValue，最后再将变量x的值改为3。方法返回前同样将returnValue中保留的整数2读到了**操作栈顶**。从第21行开始的代码，作用是将变量x的值赋为3，并将栈顶的异常抛出，方法结束。

​	尽管大家都知道这段代码出现异常的概率非常之小，但是并不影响它为我们演示异常表的作用。如果大家到这里仍然对字节码的运作过程比较模糊，其实也不要紧，关于虚拟机执行字节码的过程，本书第8章中将会有更详细的讲解。

#### 2. Exceptions属性

​	这里的Exceptions属性是在方法表中与Code属性平级的一项属性，读者不要与前面刚刚讲解完的异常表产生混淆。Exceptions属性的作用是列举出方法中可能抛出的受查异常（Checked Excepitons），也就是方法描述时在throws关键字后面列举的异常。它的结构见表6-17。

​	表6-17　Exceptions属性结构

| 类型 | 名称                  | 数量                 |
| ---- | --------------------- | -------------------- |
| u2   | attribute_name_index  | 1                    |
| u4   | attribute_length      | 1                    |
| u2   | number_of_exceptions  | 1                    |
| u2   | exception_index_table | number_of_exceptions |

​	此属性中的number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常，每一种受查异常使用一个exception_index_table项表示；exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引，代表了该受查异常的类型。

#### 3. LineNumberTable属性

​	<u>LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系</u>。它并不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中使用`-g：none`或`-g：lines`选项来取消或要求生成这项信息。<u>如果选择不生成LineNumberTable属性，对程序运行产生的最主要影响就是当抛出异常时，堆栈中将不会显示出错的行号，并且在调试程序的时候，也无法按照源码行来设置断点</u>。LineNumberTable属性的结构如表6-18所示。

​	表6-18　LineNumberTable属性结构

| 类型             | 名称                     | 数量                     |
| ---------------- | ------------------------ | ------------------------ |
| u2               | attribute_name_index     | 1                        |
| u4               | attribute_length         | 1                        |
| u2               | line_number_table_length | 1                        |
| line_number_info | line_number_table        | line_number_table_length |

​	line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合，line_number_info表包含start_pc和line_number两个u2类型的数据项，前者是字节码行号，后者是Java源码行号。

#### 4. LocalVariableTable及LocalVariableTypeTable属性

​	**LocalVariableTable属性用于描述栈帧中局部变量表的变量与Java源码中定义的变量之间的关系，它也不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中使用`-g：none`或`-g：vars`选项来取消或要求生成这项信息**。<u>如果没有生成这项属性，最大的影响就是当其他人引用这个方法时，所有的参数名称都将会丢失，譬如IDE将会使用诸如arg0、arg1之类的占位符代替原有的参数名，这对程序运行没有影响，但是会对代码编写带来较大不便，而且在调试期间无法根据参数名称从上下文中获得参数值</u>。LocalVariableTable属性的结构如表6-19所示。

​	表6-19　LocalVariableTable属性结构

| 类型                | 名称                        | 数量                        |
| ------------------- | --------------------------- | --------------------------- |
| u2                  | attribute_name_index        | 1                           |
| u4                  | attribute_length            | 1                           |
| u2                  | local_variable_table_length | 1                           |
| local_variable_info | local_variable_table        | local_variable_table_length |

​	其中local_variable_info项目代表了一个栈帧与源码中的局部变量的关联，结构如表6-20所示。

​	表6-20　local_variable_info项目结构

| 类型 | 名称             | 数量 |
| ---- | ---------------- | ---- |
| u2   | start_pc         | 1    |
| u2   | length           | 1    |
| u2   | name_index       | 1    |
| u2   | descriptor_index | 1    |
| u2   | index            | 1    |

​	start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围。

​	name_index和descriptor_index都是指向常量池中CONSTANT_Utf8_info型常量的索引，分别代表了局部变量的名称以及这个局部变量的描述符。

​	index是这个局部变量在栈帧的局部变量表中变量槽的位置。当这个变量数据类型是64位类型时（double和long），它占用的变量槽为index和index+1两个。

​	顺便提一下，在JDK 5引入泛型之后，LocalVariableTable属性增加了一个“姐妹属性”——LocalVariableTypeTable。这个新增的属性结构与LocalVariableTable非常相似，仅仅是把记录的字段描述符的descriptor_index替换成了字段的特征签名（Signature）。对于非泛型类型来说，描述符和特征签名能描述的信息是能吻合一致的，但是泛型引入之后，由于描述符中泛型的参数化类型被擦除掉[^11]，描述符就不能准确描述泛型类型了。因此出现了LocalVariableTypeTable属性，使用字段的特征签名来完成泛型的描述。

#### 5. SourceFile及SourceDebugExtension属性

​	**SourceFile属性用于记录生成这个Class文件的源码文件名称**。这个属性也是可选的，可以使用Javac的`-g：none`或`-g：source`选项来关闭或要求生成这项信息<u>。在Java中，对于大多数的类来说，类名和文件名是一致的，但是有一些特殊情况（如内部类）例外</u>。如果不生成这项属性，当抛出异常时，堆栈中将不会显示出错代码所属的文件名。这个属性是一个定长的属性，其结构如表6-21所示。

​	表6-21　SourceFile属性结构

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | sourcefile_index     | 1    |

​	**sourcefile_index数据项是指向常量池中CONSTANT_Utf8_info型常量的索引，常量值是源码文件的文件名**。

​	为了方便在编译器和动态生成的Class中加入供程序员使用的自定义内容，在JDK 5时，新增了SourceDebugExtension属性用于存储额外的代码调试信息。典型的场景是在进行JSP文件调试时，无法通过Java堆栈来定位到JSP文件的行号。JSR 45提案为这些非Java语言编写，却需要编译成字节码并运行在Java虚拟机中的程序提供了一个进行调试的标准机制，使用SourceDebugExtension属性就可以用于存储这个标准所新加入的调试信息，譬如让程序员能够快速从异常堆栈中定位出原始JSP中出现问题的行号。SourceDebugExtension属性的结构如表6-22所示。

​	表6-22　SourceDebugExtension属性结构

| 类型 | 名称                              | 数量 |
| ---- | --------------------------------- | ---- |
| u2   | attribute_name_index              | 1    |
| u4   | attribute_length                  | 1    |
| u1   | debug_extension[attribute_length] | 1    |

​	其中debug_extension存储的就是额外的调试信息，是一组通过变长UTF-8格式来表示的字符串。一个类中最多只允许存在一个SourceDebugExtension属性。

#### 6. ConstantValue属性

​	**ConstantValue属性的作用是通知虚拟机自动为静态变量赋值**。只有被static关键字修饰的变量（类变量）才可以使用这项属性。类似“int x=123”和“static int x=123”这样的变量定义在Java程序里面是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。对非static类型的变量（也就是实例变量）的赋值是在实例构造器\<init\>()方法中进行的；而对于类变量，则有两种方式可以选择：在类构造器\<clinit\>()方法中或者使用ConstantValue属性。目前Oracle公司实现的Javac编译器的选择是，<u>如果同时使用final和static来修饰一个变量（按照习惯，这里称“常量”更贴切），并且这个变量的数据类型是基本类型或者java.lang.String的话，就将会生成ConstantValue属性来进行初始化；如果这个变量没有被final修饰，或者并非基本类型及字符串，则将会选择在\<clinit\>()方法中进行初始化。</u>

​	**虽然有final关键字才更符合“ConstantValue”的语义，但《Java虚拟机规范》中并没有强制要求字段必须设置ACC_FINAL标志，只要求有ConstantValue属性的字段必须设置ACC_STATIC标志而已，对final关键字的要求是Javac编译器自己加入的限制**。而对ConstantValue的属性值只能限于基本类型和String这点，其实并不能算是什么限制，这是理所当然的结果。因为此属性的属性值只是一个常量池的索引号，由于Class文件格式的常量类型中只有与基本属性和字符串相对应的字面量，所以就算ConstantValue属性想支持别的类型也无能为力。ConstantValue属性的结构如表6-23所示。

​	表6-23　ConstantValue属性结构

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | constantvalue_index  | 1    |

​	从数据结构中可以看出ConstantValue属性是一个定长属性，它的attribute_length数据项值必须固定为2。constantvalue_index数据项代表了常量池中一个字面量常量的引用，根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。

#### 7. InnerClasses属性

​	InnerClasses属性用于记录内部类与宿主类之间的关联。如果一个类中定义了内部类，那编译器将会为它以及它所包含的内部类生成InnerClasses属性。InnerClasses属性的结构如表6-24所示。

​	表6-24　InnerClasses属性结构

| 类型               | 名称                 | 数量              |
| ------------------ | -------------------- | ----------------- |
| u2                 | attribute_name_index | 1                 |
| u4                 | attribute_length     | 1                 |
| u2                 | number_of_classes    | 1                 |
| inner_classes_info | inner_classes        | number_of_classes |

​	数据项number_of_classes代表需要记录多少个内部类信息，每一个内部类的信息都由一个inner_classes_info表进行描述。inner_classes_info表的结构如表6-25所示。

​	表6-25　inner_classes_info表的结构

| 类型 | 名称                     | 数量 |
| ---- | ------------------------ | ---- |
| u2   | inner_class_info_index   | 1    |
| u2   | out_class_info_index     | 1    |
| u2   | inner_name_index         | 1    |
| u2   | inner_class_access_flags | 1    |

​	**inner_class_info_index和outer_class_info_index都是指向常量池中CONSTANT_Class_info型常量的索引，分别代表了内部类和宿主类的符号引用**。

​	inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引，代表这个内部类的名称，如果是匿名内部类，这项值为0。

​	inner_class_access_flags是内部类的访问标志，类似于类的access_flags，它的取值范围如表6-26所示。

​	表6-26　inner_class_access_flags标志

| 标志名称       | 标志   | 含义                           |
| -------------- | ------ | ------------------------------ |
| ACC_PUBLIC     | 0x0001 | 内部类是否为public             |
| ACC_PRIVATE    | 0x0002 | 内部类是否为private            |
| ACC_PROTECTED  | 0x0004 | 内部类是否为protected          |
| ACC_STATIC     | 0x0008 | 内部类是否为static             |
| ACC_FINAL      | 0x0010 | 内部类是否为final              |
| ACC_INTERFACE  | 0x0020 | 内部类是否为接口               |
| ACC_ABSTRACT   | 0x0400 | 内部类是否为abstract           |
| ACC_SYNTHETIC  | 0x1000 | 内部类是否并非由用户代码产生的 |
| ACC_ANNOTATION | 0x2000 | 内部类是不是一个注解           |
| ACC_ENUM       | 0x4000 | 内部类是不是一个枚举           |

#### 8. Deprecated及Synthetic属性

​	Deprecated和Synthetic两个属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。

​	Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为不再推荐使用，它可以通过代码中使用“@deprecated”注解进行设置。

​	**Synthetic属性代表此字段或者方法并不是由Java源码直接产生的，而是由编译器自行添加的，在JDK 5之后，标识一个类、字段或者方法是编译器自动产生的，也可以设置它们访问标志中的ACC_SYNTHETIC标志位**。编译器通过生成一些在源代码中不存在的Synthetic方法、字段甚至是整个类的方式，实现了越权访问（越过private修饰器）或其他绕开了语言限制的功能，这可以算是一种早期优化的技巧，其中最典型的例子就是枚举类中自动生成的枚举元素数组和嵌套类的桥接方法（Bridge Method）。所有由不属于用户代码产生的类、方法及字段都应当至少设置Synthetic属性或者ACC_SYNTHETIC标志位中的一项，唯一的例外是实例构造器“\<init\>()”方法和类构造器“\<clinit\>()”方法。

​	Deprecated和Synthetic属性的结构非常简单，如表6-27所示。

​	表6-27　Deprecated及Synthetic属性结构

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |

​	其中attribute_length数据项的值必须为0x00000000，因为没有任何属性值需要设置。

#### 9. StackMapTable属性

​	StackMapTable属性在JDK 6增加到Class文件规范之中，它是一个相当复杂的变长属性，位于Code属性的属性表中。这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用（详见第7章字节码验证部分），目的在于代替以前比较消耗性能的基于数据流分析的类型推导验证器。

​	这个类型检查验证器最初来源于Sheng Liang（听名字似乎是虚拟机团队中的华裔成员）实现为JavaME CLDC实现的字节码验证器。<u>新的验证器在同样能保证Class文件合法性的前提下，省略了在运行期通过数据流分析去确认字节码的行为逻辑合法性的步骤，而在编译阶段将一系列的验证类型（Verification Type）直接记录在Class文件之中，通过检查这些验证类型代替了类型推导过程，从而大幅提升了字节码验证的性能</u>。<u>这个验证器在JDK 6中首次提供，并在JDK 7中强制代替原本基于类型推断的字节码验证器</u>。关于这个验证器的工作原理，《Java虚拟机规范》在Java SE 7版中新增了整整120页的篇幅来讲解描述，其中使用了庞大而复杂的公式化语言去分析证明新验证方法的严谨性，笔者在此就不展开赘述了。

​	StackMapTable属性中包含零至多个栈映射帧（Stack Map Frame），每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示执行到该字节码时局部变量表和操作数栈的验证类型。类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。StackMapTable属性的结构如表6-28所示。

​	表6-28　StackMapTable属性结构

| 类型            | 名称                    | 数量              |
| --------------- | ----------------------- | ----------------- |
| u2              | attribute_name_index    | 1                 |
| u4              | attribute_length        | 1                 |
| u2              | number_of_entries       | 1                 |
| stack_map_frame | stack_map_frame_entries | number_of_entries |

​	在Java SE 7版之后的《Java虚拟机规范》中，明确规定对于版本号大于或等于50.0的Class文件，如果方法的Code属性中没有附带StackMapTable属性，那就意味着它带有一个隐式的StackMap属性，这个StackMap属性的作用等同于number_of_entries值为0的StackMapTable属性。一个方法的Code属性最多只能有一个StackMapTable属性，否则将抛出ClassFormatError异常。

#### 10. Signature属性

​	Signature属性在JDK 5增加到Class文件规范之中，它是一个可选的定长属性，可以出现于类、字段表和方法表结构的属性表中。**在JDK 5里面大幅增强了Java语言的语法，在此之后，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（ParameterizedType），则Signature属性会为它记录泛型签名信息**。<u>之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是擦除法实现的伪泛型，字节码（Code属性）中所有的泛型信息编译（类型变量、参数化类型）在编译之后都通通被擦除掉</u>。**使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。但坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同等对待，例如运行期做反射时无法获得泛型信息。Signature属性就是为了弥补这个缺陷而增设的，现在Java的反射API能够获取的泛型类型，最终的数据来源也是这个属性**。关于Java泛型、Signature属性和类型擦除，在第10章讲编译器优化的时候我们会通过一个更具体的例子来讲解。Signature属性的结构如表6-29所示。

​	表6-29　Signature属性结构

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | signature_index      | 1    |

​	其中signature_index项的值必须是一个对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示类签名或方法类型签名或字段类型签名。<u>如果当前的Signature属性是类文件的属性，则这个结构表示类签名，如果当前的Signature属性是方法表的属性，则这个结构表示方法类型签名，如果当前Signature属性是字段表的属性，则这个结构表示字段类型签名</u>。

#### 11. BootstrapMethods属性

​	BootstrapMethods属性在JDK 7时增加到Class文件规范之中，它是一个复杂的变长属性，位于类文件的属性表中。这个属性用于保存invokedynamic指令引用的引导方法限定符。

​	根据《Java虚拟机规范》（从Java SE 7版起）的规定，如果某个类文件结构的常量池中曾经出现过CONSTANT_InvokeDynamic_info类型的常量，那么这个类文件的属性表中必须存在一个明确的BootstrapMethods属性，另外，即使CONSTANT_InvokeDynamic_info类型的常量在常量池中出现过多次，类文件的属性表中最多也只能有一个BootstrapMethods属性。BootstrapMethods属性和JSR-292中的InvokeDynamic指令和java.lang.Invoke包关系非常密切，要介绍这个属性的作用，必须先讲清楚InovkeDynamic指令的运作原理，笔者将在第8章专门花一整节篇幅去介绍它们，在此先暂时略过。

​	虽然JDK 7中已经提供了InovkeDynamic指令，但这个版本的Javac编译器还暂时无法支持InvokeDynamic指令和生成BootstrapMethods属性，必须通过一些非常规的手段才能使用它们。直到JDK 8中Lambda表达式和接口默认方法的出现，InvokeDynamic指令才算在Java语言生成的Class文件中有了用武之地。BootstrapMethods属性的结构如表6-30所示。

​	表6-30　BootstrapMethods属性结构

| 类型            | 名称                        | 数量                  |
| --------------- | --------------------------- | --------------------- |
| u2              | attribute_name_index        | 1                     |
| u4              | attribute_length            | 1                     |
| u2              | attribute_bootstrap_methods | 1                     |
| boostrap_method | bootstrap_methods           | num_bootstrap_methods |

​	其中引用到的bootstrap_method结构如表6-31所示。

​	表6-31　bootstrap_method属性结构

| 类型 | 名称                    | 数量                    |
| ---- | ----------------------- | ----------------------- |
| u2   | bootstrap_method_ref    | 1                       |
| u2   | num_bootstrap_arguments | 1                       |
| u2   | bootstrap_arguments     | num_bootstrap_arguments |

​	BootstrapMethods属性里，num_bootstrap_methods项的值给出了bootstrap_methods[]数组中的引导方法限定符的数量。而bootstrap_methods[]数组的每个成员包含了一个指向常量池CONSTANT_MethodHandle结构的索引值，它代表了一个引导方法。还包含了这个引导方法静态参数的序列（可能为空）。bootstrap_methods[]数组的每个成员必须包含以下三项内容：

+ bootstrap_method_ref：bootstrap_method_ref项的值必须是一个对常量池的有效索引。常量池在该索引处的值必须是一个CONSTANT_MethodHandle_info结构。

+ num_bootstrap_arguments：num_bootstrap_arguments项的值给出了bootstrap_argu-ments[]数组成员的数量。
+ bootstrap_arguments[]：bootstrap_arguments[]数组的每个成员必须是一个对常量池的有效索引。常量池在该索引出必须是下列结构之一：CONSTANT_String_info、CONSTANT_Class_info、CONSTANT_Integer_info、CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_MethodHandle_info或CONSTANT_MethodType_info。

#### 12. MethodParameters属性

​	MethodParameters是在JDK 8时新加入到Class文件格式中的，它是一个用在方法表中的变长属性。MethodParameters的作用是记录方法的各个形参名称和信息。

​	<u>最初，基于存储空间的考虑，Class文件默认是不储存方法参数名称的，因为给参数起什么名字对计算机执行程序来说是没有任何区别的，所以只要在源码中妥当命名就可以了</u>。随着Java的流行，这点确实为程序的传播和二次复用带来了诸多不便，由于Class文件中没有参数的名称，如果只有单独的程序包而不附加上JavaDoc的话，在IDE中编辑使用包里面的方法时是无法获得方法调用的智能提示的，这就阻碍了JAR包的传播。后来，“-g：var”就成为了Javac以及许多IDE编译Class时采用的默认值，这样会将方法参数的名称生成到LocalVariableTable属性之中。不过此时问题仍然没有全部解决，<u>LocalVariableTable属性是Code属性的子属性——没有方法体存在，自然就不会有局部变量表，但是对于其他情况，譬如抽象方法和接口方法，是理所当然地可以不存在方法体的，对于方法签名来说，还是没有找到一个统一完整的保留方法参数名称的地方</u>。所以JDK 8中新增的这个属性，使得编译器可以（编译时加上-parameters参数）将方法名称也写进Class文件中，而且MethodParameters是方法表的属性，与Code属性平级的，可以运行时通过反射API获取。MethodParameters的结构如表6-32所示。

​	表6-32　MethodParameters属性结构

| 类型      | 名称                 | 数量             |
| --------- | -------------------- | ---------------- |
| u2        | attribute_name_index | 1                |
| u4        | atrribute            | 1                |
| u1        | parameters_count     | 1                |
| parameter | parameters           | parameters_count |

​	其中，引用到的parameter结构如表6-33所示。

​	表6-33　parameter属性结构

| 类型 | 名称         | 数量 |
| ---- | ------------ | ---- |
| u2   | name_index   | 1    |
| u2   | access_flags | 1    |

​	其中，name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该参数的名称。而access_flags是参数的状态指示器，它可以包含以下三种状态中的一种或多种：

+ 0x0010（ACC_FINAL）：表示该参数被final修饰。
+ 0x1000（ACC_SYNTHETIC）：表示该参数并未出现在源文件中，是编译器自动生成的。
+ **0x8000（ACC_MANDATED）：表示该参数是在源文件中隐式定义的。Java语言中的典型场景是this关键字**。

#### 13. 模块化相关属性

​	**JDK 9的一个重量级功能是Java的模块化功能，因为模块描述文件（module-info.java）最终是要编译成一个独立的Class文件来存储的，所以，Class文件格式也扩展了Module、ModulePackages和ModuleMainClass三个属性用于支持Java模块化相关功能**。

​	Module属性是一个非常复杂的变长属性，除了表示该模块的名称、版本、标志信息以外，还存储了这个模块requires、exports、opens、uses和provides定义的全部内容，其结构如表6-34所示。

​	表6-34　Module属性结构

| 类型    | 名称                 | 数量           |
| ------- | -------------------- | -------------- |
| u2      | attribute_name_index | 1              |
| u4      | attribute_length     | 1              |
| u2      | module_name_index    | 1              |
| u2      | module_flags         | 1              |
| u2      | module_version_index | 1              |
| u2      | requires_count       | 1              |
| require | requires             | requires_count |
| u2      | exports_count        | 1              |
| export  | exports              | exports_count  |
| u2      | opens_count          | 1              |
| open    | opens                | opens_count    |
| u2      | uses_count           | 1              |
| use     | uses_index           | uses_count     |
| u2      | provides_count       | 1              |
| provide | provides             | provides_count |

​	其中，module_name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该模块的名称。而module_flags是模块的状态指示器，它可以包含以下三种状态中的一种或多种：

+ 0x0020（ACC_OPEN）：表示该模块是开放的。
+ 0x1000（ACC_SYNTHETIC）：表示该模块并未出现在源文件中，是编译器自动生成的。
+ 0x8000（ACC_MANDATED）：表示该模块是在源文件中隐式定义的。

​	module_version_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，代表了该模块的版本号。

​	后续的几个属性分别记录了模块的requires、exports、opens、uses和provides定义，由于它们的结构是基本相似的，为了节省版面，笔者仅介绍其中的exports，该属性结构如表6-35所示。

​	表6-35　exports属性结构

| 类型   | 名称             | 数量             |
| ------ | ---------------- | ---------------- |
| u2     | exports_index    | 1                |
| u2     | exports_flags    | 1                |
| u2     | exports_to_count | 1                |
| export | exports_to_index | exports_to_count |

​	exports属性的每一元素都代表一个被模块所导出的包，exports_index是一个指向常量池CONSTANT_Package_info常量的索引值，代表了被该模块导出的包。exports_flags是该导出包的状态指示器，它可以包含以下两种状态中的一种或多种：

+ 0x1000（ACC_SYNTHETIC）：表示该导出包并未出现在源文件中，是编译器自动生成的。
+ 0x8000（ACC_MANDATED）：表示该导出包是在源文件中隐式定义的。

​	exports_to_count是该导出包的限定计数器，如果这个计数器为零，这说明该导出包是无限定的（Unqualified），即完全开放的，任何其他模块都可以访问该包中所有内容。如果该计数器不为零，则后面的exports_to_index是以计数器值为长度的数组，每个数组元素都是一个指向常量池中CONSTANT_Module_info常量的索引值，代表着只有在这个数组范围内的模块才被允许访问该导出包的内容。

​	ModulePackages是另一个用于支持Java模块化的变长属性，它用于描述该模块中所有的包，不论是不是被export或者open的。该属性的结构如表6-36所示。

​	表6-36　ModulePackages属性结构

| 类型 | 名称                 | 数量          |
| ---- | -------------------- | ------------- |
| u2   | attribute_name_index | 1             |
| u4   | package_length       | 1             |
| u2   | package_count        | 1             |
| u2   | package_index        | package_count |

​	package_count是package_index数组的计数器，package_index中每个元素都是指向常量池CONSTANT_Package_info常量的索引值，代表了当前模块中的一个包。

​	最后一个ModuleMainClass属性是一个定长属性，用于确定该模块的主类（Main Class），其结构如表6-37所示。

​	表6-37　ModuleMainClass属性结构

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | main_class_index     | 1    |

​	其中，main_class_index是一个指向常量池CONSTANT_Class_info常量的索引值，代表了该模块的主类。

#### 14. 运行时注解相关属性

​	**早在JDK 5时期，Java语言的语法进行了多项增强，其中之一是提供了对注解（Annotation）的支持**。为了存储源码中注解信息，Class文件同步增加了RuntimeVisibleAnnotations、RuntimeInvisibleAnnotations、RuntimeVisibleParameterAnnotations和RuntimeInvisibleParameter-Annotations四个属性。到了JDK 8时期，进一步加强了Java语言的注解使用范围，又新增类型注解（JSR 308），所以Class文件中也同步增加了RuntimeVisibleTypeAnnotations和RuntimeInvisibleTypeAnnotations两个属性。由于这六个属性不论结构还是功能都比较雷同，因此我们把它们合并到一起，以RuntimeVisibleAnnotations为代表进行介绍。

​	RuntimeVisibleAnnotations是一个变长属性，它记录了类、字段或方法的声明上记录运行时可见注解，当我们使用反射API来获取类、字段或方法上的注解时，返回值就是通过这个属性来取到的。RuntimeVisibleAnnotations属性的结构如表6-38所示。

​	表6-38　RuntimeVisibleAnnotations属性结构

| 类型       | 名称                 | 数量            |
| ---------- | -------------------- | --------------- |
| u2         | attribute_name_index | 1               |
| u4         | attribute_length     | 1               |
| u2         | num_annotations      | 1               |
| annotation | annotations          | num_annotations |

​	num_annotations是annotations数组的计数器，annotations中每个元素都代表了一个运行时可见的注解，注解在Class文件中以annotation结构来存储，具体如表6-39所示。

​	表6-39　annotation属性结构

| 类型               | 名称                    | 数量                    |
| ------------------ | ----------------------- | ----------------------- |
| u2                 | type_index              | 1                       |
| u2                 | num_element_value_pairs | 1                       |
| element_value_pair | element_value_pairs     | num_element_value_pairs |

​	type_index是一个指向常量池CONSTANT_Utf8_info常量的索引值，该常量应以字段描述符的形式表示一个注解。num_element_value_pairs是element_value_pairs数组的计数器，element_value_pairs中每个元素都是一个键值对，代表该注解的参数和值。

[^9]:此处字节码的“行”是一种形象的描述，指的是字节码相对于方法体开始的偏移量，而不是Java源码的行号，下同。
[^10]:在JDK 1.4.2之前的Javac编译器采用了jsr和ret指令实现finally语句，但1.4.2之后已经改为编译器在每段分支之后都将finally语句块的内容冗余生成一遍来实现。从JDK 7起，已经完全禁止Class文件中出现jsr和ret指令，如果遇到这两条指令，虚拟机会在类加载的字节码校验阶段抛出异常。
[^11]:详见10.3节的内容。

## 6.4　字节码指令简介

个人小结：

+ Java虚拟机采用**面向操作数栈**而不是面向寄存器的架构（这两种架构的执行过程、区别和影响将在第8章中探讨），所以**大多数指令都不包含操作数，只有一个操作码，指令参数都存放在操作数栈中**。
+ 由于Class文件格式放弃了编译后代码的操作数长度对齐，这就意味着虚拟机在处理那些超过一个字节的数据时，不得不在运行时从字节中重建出具体数据的结构。
+ 字节码指令流基本上都是单字节对齐的，只有“tableswitch”和“lookupswitch”两条指令例外，由于它们的操作数比较特殊，是以4字节为界划分开的，所以这两条指令也需要预留出相应的空位填充来实现对齐。

---

​	Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作码，Opcode）以及跟随其后的零至多个代表此操作所需的参数（称为操作数，Operand）构成。由<u>于Java虚拟机采用**面向操作数栈**而不是面向寄存器的架构（这两种架构的执行过程、区别和影响将在第8章中探讨），所以**大多数指令都不包含操作数，只有一个操作码，指令参数都存放在操作数栈中**</u>

​	字节码指令集可算是一种具有鲜明特点、优势和劣势均很突出的指令集架构，由于限制了Java虚拟机操作码的长度为一个字节（即0～255），这意味着指令集的操作码总数不能够超过256条；又**<u>由于Class文件格式放弃了编译后代码的操作数长度对齐，这就意味着虚拟机在处理那些超过一个字节的数据时，不得不在运行时从字节中重建出具体数据的结构</u>**，譬如要将一个16位长度的无符号整数使用两个无符号字节存储起来（假设将它们命名为byte1和byte2），那它们的值应该是这样的：

```java
(byte1 << 8) | byte2
```

​	这种操作在某种程度上会导致解释执行字节码时将损失一些性能，但这样做的优势也同样明显：放弃了操作数长度对齐[^12]，就意味着可以省略掉大量的填充和间隔符号；用一个字节来代表操作码，也是为了尽可能获得短小精干的编译代码。这种追求尽可能小数据量、高传输效率的设计是由Java语言设计之初主要面向网络、智能家电的技术背景所决定的，并一直沿用至今。

​	如果不考虑异常处理的话，那Java虚拟机的解释器可以使用下面这段伪代码作为最基本的执行模型来理解，这个执行模型虽然很简单，但依然可以有效正确地工作：

```java
do {
  自动计算PC寄存器的值加1;
  根据PC寄存器指示的位置，从字节码流中取出操作码;
  if (字节码存在操作数) 从字节码流中取出操作数;
  执行操作码所定义的操作;
} while (字节码流长度 > 0);
```

[^12]:字节码指令流基本上都是单字节对齐的，只有“tableswitch”和“lookupswitch”两条指令例外，由于它们的操作数比较特殊，是以4字节为界划分开的，所以这两条指令也需要预留出相应的空位填充来实现对齐。

### 6.4.1 字节码与数据类型

个人小结：

+ 对于大部分与数据类型相关的字节码指令，它们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：i代表对int类型的数据操作，l代表long，s代表short，b代表byte，c代表char，f代表float，d代表double，a代表reference。
+ **Java虚拟机的操作码长度只有一字节**，非每种数据类型和每一种操作都有对应的指令
+ **大部分指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型**。<u>编译器会在**编译期**或**运行期**将byte和short类型的数据**带符号扩展（Sign-Extend）**为相应的int类型数据，将boolean和char类型数据**零位扩展（Zero-Extend）**为相应的int类型数据。</u>与之类似，在处理boolean、byte、short和char类型的**数组**时，也会转换为使用对应的int类型的字节码指令来处理
+ **大多数对于boolean、byte、short和char类型数据的操作，实际上都是使用相应的对int类型作为运算类型（Computational Type）来进行的。**

---

​	<u>在Java虚拟机的指令集中，大多数指令都包含其操作所对应的数据类型信息</u>。举个例子，iload指令用于从局部变量表中加载int型的数据到操作数栈中，而fload指令加载的则是float类型的数据。<u>这两条指令的操作在虚拟机内部可能会是由同一段代码来实现的，但在Class文件中它们必须拥有各自独立的操作码</u>。

​	**对于大部分与数据类型相关的字节码指令，它们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：i代表对int类型的数据操作，l代表long，s代表short，b代表byte，c代表char，f代表float，d代表double，a代表reference**。也有一些指令的助记符中没有明确指明操作类型的字母，例如arraylength指令，它没有代表数据类型的特殊字符，但操作数永远只能是一个数组类型的对象。还有另外一些指令，例如<u>无条件跳转指令goto则是与数据类型无关的指令</u>。

​	**因为Java虚拟机的操作码长度只有一字节，所以包含了数据类型的操作码就为指令集的设计带来了很大的压力**：如果每一种与数据类型相关的指令都支持Java虚拟机所有运行时数据类型的话，那么指令的数量恐怕就会超出一字节所能表示的数量范围了。因此，Java虚拟机的指令集对于特定的操作只提供了有限的类型相关指令去支持它，换句话说，指令集将会被故意设计成非完全独立的。（《Java虚拟机规范》中把这种特性称为“Not Orthogonal”，**即并非每种数据类型和每一种操作都有对应的指令**。）<u>有一些单独的指令可以在必要的时候用来将一些不支持的类型转换为可被支持的类型</u>。

​	表6-40列举了Java虚拟机所支持的与数据类型相关的字节码指令，通过使用数据类型列所代表的特殊字符替换opcode列的指令模板中的T，就可以得到一个具体的字节码指令。如果在表中指令模板与数据类型两列共同确定的格为空，则说明虚拟机不支持对这种数据类型执行这项操作。例如load指令有操作int类型的iload，但是没有操作byte类型的同类指令。

​	表6-40　Java虚拟机指令集所支持的数据类型

| opcode    | byte    | short   | int       | long    | float   | double  | char    | reference |
| --------- | ------- | ------- | --------- | ------- | ------- | ------- | ------- | --------- |
| Tipush    | bipush  | sipush  |           |         |         |         |         |           |
| Tconst    |         |         | iconst    | lconst  | fconst  | dconst  |         | aconst    |
| Tload     |         |         | iload     | lload   | fload   | dload   |         | aload     |
| Tstore    |         |         | istore    | lstore  | fstore  | dstroe  |         | astore    |
| Tinc      |         |         | iinc      |         |         |         |         |           |
| Taload    | baload  | saload  | iaload    | laload  | faload  | dload   | caload  | aaload    |
| Tastroe   | bastroe | sastroe | iastroe   | lastroe | fastroe | dastroe | castroe | aastroe   |
| Tadd      |         |         | iadd      | ladd    | fadd    | dadd    |         |           |
| Tsub      |         |         | isub      | lsub    | fsub    | dsub    |         |           |
| Tmul      |         |         | imul      | lmul    | fmul    | dmul    |         |           |
| Tdiv      |         |         | idiv      | ldiv    | fdiv    | ddiv    |         |           |
| Trem      |         |         | irem      | lrem    | frem    | drem    |         |           |
| Tneg      |         |         | ineg      | lneg    | fneg    | dneg    |         |           |
| Tshl      |         |         | ishl      | shll    |         |         |         |           |
| Tshr      |         |         | ishr      | lshr    |         |         |         |           |
| Tushr     |         |         | iushr     | lushr   |         |         |         |           |
| Tand      |         |         | iand      | land    |         |         |         |           |
| Tor       |         |         | ior       | lor     |         |         |         |           |
| Txor      |         |         | ixor      | lxor    |         |         |         |           |
| i2T       | i2b     | i2s     |           | i2l     | i2f     | I2d     |         |           |
| l2T       |         |         | l2i       |         | l2f     | l2d     |         |           |
| f2T       |         |         | f2i       | f2l     |         | f2d     |         |           |
| d2T       |         |         | d2i       | d2l     | D2f     |         |         |           |
| Tcmp      |         |         |           | Lamp    |         |         |         |           |
| Tcmpl     |         |         |           |         | fcmpl   | dcmpl   |         |           |
| Tcmpg     |         |         |           |         | fcmpg   | dcmpg   |         |           |
| if_TcmpOP |         |         | if_TcmpOP |         |         |         |         | if_TcmpOP |
| Treturn   |         |         | ireturn   | lreturn | freturn | dreturn |         | areturn   |

​	请注意，从表6-40中看来，**大部分指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型**。<u>编译器会在**编译期**或**运行期**将byte和short类型的数据**带符号扩展**（Sign-Extend）为相应的int类型数据，将boolean和char类型数据**零位扩展（Zero-Extend）**为相应的int类型数据</u>。<u>与之类似，在处理boolean、byte、short和char类型的数组时，也会转换为使用对应的int类型的字节码指令来处理</u>。因此，大多数对于boolean、byte、short和char类型数据的操作，实际上都是使用相应的对int类型作为运算类型（Computational Type）来进行的。

​	<small>在本书里，受篇幅所限，无法对字节码指令集中每条指令逐一讲解，但阅读字节码作为了解Java虚拟机的基础技能，是一项应当熟练掌握的能力。笔者(书籍作者)将字节码操作按用途大致分为9类，下面按照分类来为读者概略介绍这些指令的用法。如果读者希望了解更详细的信息，可以阅读由Oracle官方授权、由笔者翻译的《Java虚拟机规范（Java SE 7）》中文版（字节码的介绍可见此书第6章）。</small>

### 6.4.2 加载和存储指令

个人小结：

关键词：栈帧、局部变量表、操作数栈

+ **将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、iconst_m1、iconst\_\<i\>、lconst\_\<l\>、fconst\_\<f\>、dconst\_\<d\>**

---

​	<u>**加载**和**存储**指令用于将数据在**栈帧**中的**局部变量表**和**操作数栈**（见第2章关于内存区域的介绍）之间来回传输</u>，这类指令包括：

+ 将一个局部变量加载到操作栈：iload、iload\_\<n\>、lload、lload\_\<n\>、fload、fload\_\<n\>、dload、dload\_\<n\>、aload、aload\_\<n\>
+ 将一个数值从操作数栈存储到局部变量表：istore、istore\_\<n\>、lstore、lstore\_\<n\>、fstore、fstore\_\<n\>、dstore、dstore_\<n\>、astore、astore\_\<n\>
+ **将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、iconst_m1、iconst\_\<i\>、lconst\_\<l\>、fconst\_\<f\>、dconst\_\<d\>**
+ **扩充局部变量表的访问索引的指令：wide**

​	<u>存储数据的操作数栈和局部变量表主要由加载和存储指令进行操作，除此之外，还有少量指令，如访问对象的字段或数组元素的指令也会向操作数栈传输数据</u>。

​	上面所列举的指令助记符中，有一部分是以尖括号结尾的（例如iload\_\<n\>），这些指令助记符实际上代表了一组指令（例如iload\_\<n\>，它代表了iload_0、iload_1、iload_2和iload_3这几条指令）。这几组指令都是某个带有一个操作数的通用指令（例如iload）的特殊形式，**对于这几组特殊指令，它们省略掉了显式的操作数，不需要进行取操作数的动作，因为实际上操作数就隐含在指令中**。除了这点不同以外，它们的语义与原生的通用指令是完全一致的（**例如iload_0的语义与操作数为0时的iload指令语义完全一致**）。这种指令表示方法，在本书和《Java虚拟机规范》中都是通用的。

### 6.4.3 运算指令

个人小结：

+ **大体上运算指令可以分为两种：对整型数据进行运算的指令与对浮点型数据进行运算的指令**
+ 《Java虚拟机规范》中并没有明确定义过整型数据溢出具体会得到什么计算结果，仅规定了在处理整型数据时，只有除法指令（idiv和ldiv）以及求余指令（irem和lrem）中当出现除数为零时会导致虚拟机抛出ArithmeticException异常，其余任何整型数运算场景都不应该抛出运行时异常
+ Java虚拟机在处理浮点数运算时，不会抛出任何运行时异常（这里所讲的是Java语言中的异常，请读者勿与IEEE 754规范中的浮点异常互相混淆，IEEE 754的浮点异常是一种运算信号），当一个操作产生溢出时，将会使用有符号的无穷大来表示
+ 如果某个操作结果没有明确的数学定义的话，将会使用NaN（Not a Number）值来表示。所有使用NaN值作为操作数的算术操作，结果都会返回NaN
+ 在对long类型数值进行比较时，Java虚拟机采用带符号的比较方式，而对浮点数值进行比较时（dcmpg、dcmpl、fcmpg、fcmpl），虚拟机会采用IEEE 754规范所定义的无信号比较（Nonsignaling Comparison）方式进行

---

​	算术指令用于对两个操作数栈上的值进行某种特定运算，并**把结果重新存入到操作栈顶**。**大体上运算指令可以分为两种：对整型数据进行运算的指令与对浮点型数据进行运算的指令**。整数与浮点数的算术指令在溢出和被零除的时候也有各自不同的行为表现。<u>无论是哪种算术指令，均是使用Java虚拟机的算术类型来进行计算的，换句话说是不存在直接支持byte、short、char和boolean类型的算术指令，对于上述几种数据的运算，应使用操作int类型的指令代替</u>。所有的算术指令包括：

+ 加法指令：iadd、ladd、fadd、dadd
+ 减法指令：isub、lsub、fsub、dsub
+ 乘法指令：imul、lmul、fmul、dmul
+ 除法指令：idiv、ldiv、fdiv、ddiv
+ 求余指令：irem、lrem、frem、drem
+ 取反指令：ineg、lneg、fneg、dneg
+ 位移指令：ishl、ishr、iushr、lshl、lshr、lushr
+ 按位或指令：ior、lor
+ 按位与指令：iand、land
+ 按位异或指令：ixor、lxor
+ 局部变量自增指令：iinc
+ 比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp

​	Java虚拟机的指令集直接支持了在《Java语言规范》中描述的各种对整数及浮点数操作（详情参见《Java语言规范》4.2.2节和4.2.4节）的语义。数据运算可能会导致溢出，例如两个很大的正整数相加，结果可能会是一个负数，这种数学上不可能出现的溢出现象，对于程序员来说是很容易理解的，但<u>其实《Java虚拟机规范》中并没有明确定义过整型数据溢出具体会得到什么计算结果，仅规定了在处理整型数据时，只有除法指令（idiv和ldiv）以及求余指令（irem和lrem）中当出现除数为零时会导致虚拟机抛出ArithmeticException异常，其余任何整型数运算场景都不应该抛出运行时异常</u>。

​	《Java虚拟机规范》要求虚拟机实现在处理浮点数时，必须严格遵循IEEE 754规范中所规定行为和限制，也就是说Java虚拟机必须完全支持IEEE 754中定义的“非正规浮点数值”（DenormalizedFloating-Point Number）和“逐级下溢”（Gradual Underflow）的运算规则。这些规则将会使某些数值算法处理起来变得明确，不会出现模棱两可的困境。譬如以上规则要求Java虚拟机在进行浮点数运算时，<u>所有的运算结果都必须舍入到适当的精度，非精确的结果必须舍入为可被表示的最接近的精确值；如果有两种可表示的形式与该值一样接近，那将优先选择最低有效位为零的。这种舍入模式也是IEEE 754规范中的默认舍入模式，称为**向最接近数舍入模式**</u>。而<u>在把浮点数转换为整数时，Java虚拟机使用IEEE 754标准中的**向零舍入模式**，这种模式的舍入结果会导致数字被截断，所有小数部分的有效字节都会被丢弃掉</u>。<u>向零舍入模式将在目标数值类型中选择一个最接近，但是不大于原值的数字来作为最精确的舍入结果</u>。

​	另外，**Java虚拟机在处理浮点数运算时，不会抛出任何运行时异常（这里所讲的是Java语言中的异常，请读者勿与IEEE 754规范中的浮点异常互相混淆，IEEE 754的浮点异常是一种运算信号），当一个操作产生溢出时，将会使用有符号的无穷大来表示**；<u>如果某个操作结果没有明确的数学定义的话，将会使用NaN（Not a Number）值来表示。所有使用NaN值作为操作数的算术操作，结果都会返回NaN</u>。

​	<u>在对long类型数值进行比较时，Java虚拟机采用带符号的比较方式，而对浮点数值进行比较时（dcmpg、dcmpl、fcmpg、fcmpl），虚拟机会采用IEEE 754规范所定义的无信号比较（Nonsignaling Comparison）方式进行</u>。

### 6.4.4 类型转换指令

个人小结：

+ **处理窄化类型转换（Narrowing Numeric Conversion）时，就必须显式地使用转换指令来完成，这些转换指令包括i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f**。<u>窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况，转换过程很可能会导致数值的精度丢失。</u>

+ ​	**Java虚拟机将一个浮点值窄化转换为整数类型T（T限于int或long类型之一）的时候，必须遵循以下转换规则：**

  + **如果浮点值是NaN，那转换结果就是int或long类型的0。**

  + **如果浮点值不是无穷大的话，浮点值使用IEEE 754的向零舍入模式取整，获得整数值v。如果v在目标类型T（int或long）的表示范围之内，那转换结果就是v；否则，将根据v的符号，转换为T所能表示的最大或者最小正数。**

+ 尽管数据类型窄化转换可能会发生上限溢出、下限溢出和精度丢失等情况，但是**《Java虚拟机规范》中明确规定数值类型的窄化转换指令永远不可能导致虚拟机抛出运行时异常**。

---

​	类型转换指令可以将两种不同的数值类型相互转换，这些转换操作一般用于实现用户代码中的显式类型转换操作，或者用来处理本节开篇所提到的字节码指令集中数据类型相关指令无法与数据类型一一对应的问题。

​	Java虚拟机直接支持（即转换时无须显式的转换指令）以下数值类型的宽化类型转换（WideningNumeric Conversion，即小范围类型向大范围类型的安全转换）：

+ int类型到long、float或者double类型

+ long类型到float、double类型

+ float类型到double类型

​	与之相对的，**处理窄化类型转换（Narrowing Numeric Conversion）时，就必须显式地使用转换指令来完成，这些转换指令包括i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f**。<u>窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况，转换过程很可能会导致数值的精度丢失。</u>

​	在将int或long类型窄化转换为整数类型T的时候，转换过程仅仅是简单丢弃除最低位N字节以外的内容，N是类型T的数据类型长度，这将可能导致转换结果与输入值有不同的正负号。对于了解计算机数值存储和表示的程序员来说这点很容易理解，因为原来符号位处于数值的最高位，高位被丢弃之后，转换结果的符号就取决于低N字节的首位了。

​	**Java虚拟机将一个浮点值窄化转换为整数类型T（T限于int或long类型之一）的时候，必须遵循以下转换规则：**

+ **如果浮点值是NaN，那转换结果就是int或long类型的0。**

+ **如果浮点值不是无穷大的话，浮点值使用IEEE 754的向零舍入模式取整，获得整数值v。如果v在目标类型T（int或long）的表示范围之内，那转换结果就是v；否则，将根据v的符号，转换为T所能表示的最大或者最小正数。**

​	从double类型到float类型做窄化转换的过程与IEEE 754中定义的一致，通过IEEE 754向最接近数舍入模式舍入得到一个可以使用float类型表示的数字。如果转换结果的绝对值太小、无法使用float来表示的话，将返回float类型的正负零；如果转换结果的绝对值太大、无法使用float来表示的话，将返回float类型的正负无穷大。**对于double类型的NaN值将按规定转换为float类型的NaN值。**

​	**尽管数据类型窄化转换可能会发生上限溢出、下限溢出和精度丢失等情况，但是《Java虚拟机规范》中明确规定数值类型的窄化转换指令永远不可能导致虚拟机抛出运行时异常。**

### 6.4.5 对象创建与访问指令

​	虽然类实例和数组都是对象，但Java虚拟机对类实例和数组的创建与操作使用了不同的字节码指令（在下一章会讲到数组和普通类的类型创建过程是不同的）。对象创建后，就可以通过对象访问指令获取对象实例或者数组实例中的字段或者数组元素，这些指令包括：

+ 创建类实例的指令：new

+ **创建数组的指令：newarray、anewarray、multianewarray**

+ 访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getfield、putfield、getstatic、putstatic

+ 把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload

+ 将一个操作数栈的值储存到数组元素中的指令：bastore、castore、sastore、iastore、fastore、dastore、aastore

+ 取数组长度的指令：arraylength

+ **检查类实例类型的指令：instanceof、checkcast**

### 6.4.6 操作数栈管理指令

​	如同操作一个普通数据结构中的堆栈那样，Java虚拟机提供了一些用于直接操作操作数栈的指令，包括：

+ 将操作数栈的栈顶一个或两个元素出栈：pop、pop2

+ **复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2**

+ **将栈最顶端的两个数值互换：swap**

### 6.4.7 控制转移指令

个人小结：

+ 在Java虚拟机中有专门的指令集用来处理int和reference类型的条件分支比较操作，为了可以无须明显标识一个数据的值是否null，也有专门的指令用来检测null值。
+ **各种类型的比较最终都会转化为int类型的比较操作**，int类型比较是否方便、完善就显得尤为重要，而Java虚拟机提供的int类型的条件分支指令是最为丰富、强大的。

---

​	控制转移指令可以让Java虚拟机有条件或无条件地从指定位置指令（而不是控制转移指令）的下一条指令继续执行程序，从概念模型上理解，可以认为控制指令就是在有条件或无条件地修改PC寄存器的值。控制转移指令包括：

+ 条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne

+ 复合条件分支：tableswitch、lookupswitch

+ 无条件分支：goto、goto_w、jsr、jsr_w、ret

​	**在Java虚拟机中有专门的指令集用来处理int和reference类型的条件分支比较操作，为了可以无须明显标识一个数据的值是否null，也有专门的指令用来检测null值。**

​	与前面算术运算的规则一致，对于boolean类型、byte类型、char类型和short类型的条件分支比较操作，都使用int类型的比较指令来完成，而对于long类型、float类型和double类型的条件分支比较操作，则会先执行相应类型的比较运算指令（dcmpg、dcmpl、fcmpg、fcmpl、lcmp，见6.4.3节），运算指令会返回一个整型值到操作数栈中，随后再执行int类型的条件分支比较操作来完成整个分支跳转。由于**各种类型的比较最终都会转化为int类型的比较操作，int类型比较是否方便、完善就显得尤为重要，而Java虚拟机提供的int类型的条件分支指令是最为丰富、强大的。**

### 6.4.8 方法调用和返回指令

个人小结：

+ **方法调用指令与数据类型无关，而方法返回指令是根据返回值的类型区分的**，包括ireturn（当返回值是boolean、byte、char、short和int类型时使用）、lreturn、freturn、dreturn和areturn，另外还有一条return指令供声明为void的方法、实例初始化方法、类和接口的类初始化方法使用。

---

​	方法调用（分派、执行过程）将在第8章具体讲解，这里仅列举以下五条指令用于方法调用：

+ **invokevirtual指令：用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是Java语言中最常见的方法分派方式。**

+ invokeinterface指令：用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。

+ **invokespecial指令：用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。**

+ invokestatic指令：用于调用类静态方法（static方法）。

+ **invokedynamic指令：用于在运行时动态解析出调用点限定符所引用的方法。并执行该方法。前面四条调用指令的分派逻辑都固化在Java虚拟机内部，用户无法改变，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。**

​	**方法调用指令与数据类型无关，而方法返回指令是根据返回值的类型区分的，包括ireturn（当返回值是boolean、byte、char、short和int类型时使用）、lreturn、freturn、dreturn和areturn，另外还有一条return指令供声明为void的方法、实例初始化方法、类和接口的类初始化方法使用。**

### 6.4.9 异常处理指令

个人小结：

+ **在Java程序中显式抛出异常的操作（throw语句）都由athrow指令来实现**
+ **在Java虚拟机中，处理异常（catch语句）不是由字节码指令来实现的（很久之前曾经使用jsr和ret指令来实现，现在已经不用了），而是采用异常表来完成**

---

​	**在Java程序中显式抛出异常的操作（throw语句）都由athrow指令来实现**，除了用throw语句显式抛出异常的情况之外，《Java虚拟机规范》还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动抛出。例如前面介绍整数运算中，当除数为零时，虚拟机会在idiv或ldiv指令中抛出ArithmeticException异常。

​	**而在Java虚拟机中，处理异常（catch语句）不是由字节码指令来实现的（很久之前曾经使用jsr和ret指令来实现，现在已经不用了），而是采用异常表来完成**。

### 6.4.10 同步指令

个人小结：

+ **Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor，更常见的是直接将它称为“锁”）来实现的。**
+ **方法级的同步是隐式的，无须通过字节码指令来控制，它实现在方法调用和返回操作之中**。虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否被声明为同步方法。
+ **Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义**
+ 编译器必须确保无论方法通过何种方式完成，方法中调用过的每条monitorenter指令都必须有其对应的monitorexit指令，而无论这个方法是正常结束还是异常结束。

---

​	**Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor，更常见的是直接将它称为“锁”）来实现的**。

​	<u>**方法级的同步是隐式的，无须通过字节码指令来控制，它实现在方法调用和返回操作之中**。虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否被声明为同步方法。当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程持有了管程，其他任何线程都无法再获取到同一个管程。**如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的 管程将在异常抛到同步方法边界之外时自动释放**</u>。

​	同步一段指令集序列通常是由Java语言中的synchronized语句块来表示的，**Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义**，正确实现synchronized关键字需要Javac编译器与Java虚拟机两者共同协作支持，譬如有代码清单6-6所示的代码。

​	代码清单6-6　代码同步演示

```java
void onlyMe(Foo f) {
  synchronized(f) {
    doSomething();
  }
}
```

​	编译后，这段代码生成的字节码序列如下：

```java
Method void onlyMe(Foo)
0 aload_1 					// 将对象f入栈
1 dup 　　 					 // 复制栈顶元素（即f的引用）
2 astore_2 				  // 将栈顶元素存储到局部变量表变量槽 2中
3 monitorenter 			// 以栈顶元素（即f）作为锁，开始同步
4 aload_0 					// 将局部变量槽 0（即this指针）的元素入栈
5 invokevirtual #5 	// 调用doSomething()方法
8 aload_2 					// 将局部变量Slow 2的元素（即f）入栈
9 monitorexit 			// 退出同步
10 goto 18 					// 方法正常结束，跳转到18返回
13 astore_3 				// 从这步开始是异常路径，见下面异常表的Taget 13
14 aload_2 					// 将局部变量Slow 2的元素（即f）入栈
15 monitorexit 			// 退出同步
16 aload_3 					// 将局部变量Slow 3的元素（即异常对象）入栈
17 athrow 					// 把异常对象重新抛出给onlyMe()方法的调用者
18 return 					// 方法正常返回
Exception table:
FromTo Target Type
		4 		10 		13 any
		13 		16 		13 any
```

​	**编译器必须确保无论方法通过何种方式完成，方法中调用过的每条monitorenter指令都必须有其对应的monitorexit指令，而无论这个方法是正常结束还是异常结束。**

​	从代码清单6-6的字节码序列中可以看到，<u>为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理程序，这个异常处理程序声明可处理所有的异常，它的目的就是用来执行monitorexit指令</u>。

## 6.5 公有设计，私有实现

个人小结：

+ **任何一款Java虚拟机实现都必须能够读取Class文件并精确实现包含在其中的Java虚拟机代码的语义**。在满足《Java虚拟机规范》的约束下对具体实现做出修改和优化也是完全可行的，并且《Java虚拟机规范》中明确鼓励实现者这样去做。
+ 虚拟机实现的方式主要有以下两种：
  + **将输入的Java虚拟机代码在加载时或执行时翻译成另一种虚拟机的指令集；**
  + **将输入的Java虚拟机代码在加载时或执行时翻译成宿主机处理程序的本地指令集（即即时编译器代码生成技术）。**

---

​	<u>《Java虚拟机规范》描绘了Java虚拟机应有的共同程序存储格式：Class文件格式以及字节码指令集</u>。这些内容与硬件、操作系统和具体的Java虚拟机实现之间是完全独立的，虚拟机实现者可能更愿意把它们看作程序在各种Java平台实现之间互相安全地交互的手段。

​	理解公有设计与私有实现之间的分界线是非常有必要的，**任何一款Java虚拟机实现都必须能够读取Class文件并精确实现包含在其中的Java虚拟机代码的语义**。拿着《Java虚拟机规范》一成不变地逐字实现其中要求的内容当然是一种可行的途径，但一个优秀的虚拟机实现，在满足《Java虚拟机规范》的约束下对具体实现做出修改和优化也是完全可行的，并且《Java虚拟机规范》中明确鼓励实现者这样去做。只要优化以后Class文件依然可以被正确读取，并且包含在其中的语义能得到完整保持，那实现者就可以选择以任何方式去实现这些语义，虚拟机在后台如何处理Class文件完全是实现者自己的事情，只要它在外部接口上看起来与规范描述的一致即可[^13]。

​	虚拟机实现者可以使用这种伸缩性来让Java虚拟机获得更高的性能、更低的内存消耗或者更好的可移植性，选择哪种特性取决于Java虚拟机实现的目标和关注点是什么，虚拟机实现的方式主要有以下两种：

+ **将输入的Java虚拟机代码在加载时或执行时翻译成另一种虚拟机的指令集；**
+ **将输入的Java虚拟机代码在加载时或执行时翻译成宿主机处理程序的本地指令集（即即时编译器代码生成技术）。**

​	精确定义的虚拟机行为和目标文件格式，不应当对虚拟机实现者的创造性产生太多的限制，Java虚拟机是被设计成可以允许有众多不同的实现，并且各种实现可以在保持兼容性的同时提供不同的新的、有趣的解决方案。

[^13]:这里其实多少存在一些例外，譬如调试器（Debugger）、性能监视器（Profiler）和即时编译器（Just-In-Time Code Generator）等都可能需要访问一些通常被认为是“虚拟机后台”的元素。

## 6.6 Class文件结构的发展

​	Class文件结构自《Java虚拟机规范》初版订立以来，已经有超过二十年的历史。这二十多年间，Java技术体系有了翻天覆地的改变，JDK的版本号已经从1.0提升到了13。<u>相对于语言、API以及Java技术体系中其他方面的变化，Class文件结构一直处于一个相对比较稳定的状态，Class文件的主体结构、字节码指令的语义和数量几乎没有出现过变动[^14]，所有对Class文件格式的改进，都集中在访问标志、属性表这些设计上原本就是可扩展的数据结构中添加新内容</u>。

​	如果以《Java虚拟机规范（第2版）》（对应于JDK 1.4，是Java 2的奠基版本）为基准进行比较的话，在后续Class文件格式的发展过程中，访问标志新加入了ACC_SYNTHETIC、ACC_ANNOTATION、ACC_ENUM、ACC_BRIDGE、ACC_VARARGS共五个标志。属性表集合中，在JDK 5到JDK 12发展过程中一共增加了20项新属性，这些属性大部分是用于支持Java中许多新出现的语言特性，如枚举、变长参数、泛型、动态注解等。还有一些是为了支持性能改进和调试信息，譬如JDK 6的新类型校验器的StackMapTable属性和对非Java代码调试中用到的SourceDebugExtension属性。

​	Class文件格式所具备的平台中立（不依赖于特定硬件及操作系统）、紧凑、稳定和可扩展的特点，是Java技术体系实现平台无关、语言无关两项特性的重要支柱。

[^14]:二十余年间，字节码的数量和语义只发生过屈指可数的几次变动，例如JDK 1.0.2时改动过invokespecial指令的语义，JDK 7增加了invokedynamic指令，禁止了ret和jsr指令。

## 6.7 本章小结

​	Class文件是Java虚拟机执行引擎的数据入口，也是Java技术体系的基础支柱之一。了解Class文件的结构对后面进一步了解虚拟机执行引擎有很重要的意义。

​	本章详细讲解了Class文件结构中的各个组成部分，以及每个部分的定义、数据结构和使用方法。通过代码清单6-1的Java代码及其Class文件样例，以实战的方式演示了Class的数据是如何存储和访问的。从下一章开始，我们将以动态的、运行时的角度去看看字节码流在虚拟机执行引擎中是如何被解释执行的。

# 7. 第7章 虚拟机类加载机制

​	代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。

## 7.1 概述

​	上一章我们学习了Class文件存储格式的具体细节，在Class文件中描述的各类信息，最终都需要加载到虚拟机中之后才能被运行和使用。而虚拟机如何加载这些Class文件，Class文件中的信息进入到虚拟机后会发生什么变化，这些都是本章将要讲解的内容。

​	<u>Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制</u>。**与那些在编译时需要进行连接的语言不同，在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略让Java语言进行提前编译会面临额外的困难，也会让类加载时稍微增加一些性能开销，但是却为Java应用提供了极高的扩展性和灵活性，Java天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的**。例如，编写一个面向接口的应用程序，可以等到运行时再指定其实际的实现类，用户可以通过Java预置的或自定义类加载器，让某个本地的应用程序在运行时从网络或其他地方上加载一个二进制流作为其程序代码的一部分。这种动态组装应用的方式目前已广泛应用于Java程序之中，从最基础的Applet、JSP到相对复杂的OSGi技术，都依赖着Java语言运行期类加载才得以诞生。

​	为了避免语言表达中可能产生的偏差，在正式开始本章以前，笔者（书籍作者）先设立两个语言上的约定：

+ **第一，在实际情况中，每个Class文件都有代表着Java语言中的一个类或接口的可能，后文中直接对“类型”的描述都同时蕴含着类和接口的可能性，而需要对类和接口分开描述的场景，笔者会特别指明；**

+ **第二，与前面介绍Class文件格式时的约定一致，本章所提到的“Class文件”也并非特指某个存在于具体磁盘中的文件，而应当是一串二进制字节流，无论其以何种形式存在，包括但不限于磁盘文件、网络、数据库、内存或者动态产生等。**

## 7.2 类加载的时机

个人小结：

+ **一个类型从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期将会经历加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）七个阶段，其中<u>验证、准备、解析三个部分统称为连接（Linking）</u>**。
+ **主动引用**触发类型的初始化，**被动引用**不触发类型的初始化。
  + 通过子类引用父类的静态字段，不会导致子类初始化
  + 通过数组定义来引用类，不会触发此类的初始化
  + **常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的**
    **类的初始化**
+ 对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。
+ 接口与类真正有所区别的是前面讲述的六种“有且仅有”需要触发初始化场景中的第三种：**当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化**。

---

> [《深入理解Java虚拟机》第7章 虚拟机类加载机制](https://blog.csdn.net/huaxun66/article/details/77092308)

​	**一个类型从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期将会经历加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）七个阶段，其中<u>验证、准备、解析三个部分统称为连接（Linking）</u>**。这七个阶段的发生顺序如图7-1所示。

![这里写图片描述](https://img-blog.csdn.net/20170811102455051?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	图7-1中，<u>加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类型的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定特性（也称为动态绑定或晚期绑定）。请注意，这里笔者写的是按部就班地“开始”，而不是按部就班地“进行”或按部就班地“完成”，强调这点是因为这些阶段通常都是互相交叉地混合进行的，会在一个阶段执行的过程中调用、激活另一个阶段</u>。

​	<u>关于在什么情况下需要开始类加载过程的第一个阶段“加载”，《Java虚拟机规范》中并没有进行强制约束，这点可以交给虚拟机的具体实现来自由把握</u>。但是对于初始化阶段，**《Java虚拟机规范》则是严格规定了有且只有六种情况必须立即对类进行“初始化”**（而加载、验证、准备自然需要在此之前开始）：

1. **遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始化，则需要先触发其初始化阶段**。能够生成这四条指令的典型Java代码场景有：

   + 使用new关键字实例化对象的时候。

   + 读取或设置一个类型的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候。

   + 调用一个类型的静态方法的时候。

2. 使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。

3. **当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化**。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
5. **当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。**

6. **当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化**。

​	对于这六种会触发类型进行初始化的场景，《Java虚拟机规范》中使用了一个非常强烈的限定语——“有且只有”，这六种场景中的行为称为对一个类型进行**主动引用**。除此之外，所有引用类型的方式都不会触发初始化，称为**被动引用**。下面举三个例子来说明何为被动引用，分别见代码清单7-1、代码清单7-2和代码清单7-3。

​	代码清单7-1　被动引用的例子之一：

```java
package org.fenixsoft.classloading;
/**
* 被动使用类字段演示一：
* 通过子类引用父类的静态字段，不会导致子类初始化
**/
public class SuperClass {
  static {
    System.out.println("SuperClass init!");
  }
  public static int value = 123;
}
public class SubClass extends SuperClass {
  static {
    System.out.println("SubClass init!");
  }
}
/**
* 非主动使用类字段演示
**/
public class NotInitialization {
  public static void main(String[] args) {
    System.out.println(SubClass.value);
  }
}
```

​	上述代码运行之后，只会输出“SuperClass init！”，而不会输出“SubClass init！”。**对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化**。<u>至于是否要触发子类的加载和验证阶段，在《Java虚拟机规范》中并未明确规定，所以这点取决于虚拟机的具体实现。对于HotSpot虚拟机来说，可通过`-XX：+TraceClassLoading`参数观察到此操作是会导致子类加载的。</u>

​	代码清单7-2　被动引用的例子之二

```java
package org.fenixsoft.classloading;
/**
* 被动使用类字段演示二：
* 通过数组定义来引用类，不会触发此类的初始化
**/
public class NotInitialization {
  public static void main(String[] args) {
    SuperClass[] sca = new SuperClass[10];
  }
}
```

​	为了节省版面，这段代码复用了代码清单7-1中的SuperClass，运行之后发现没有输出“SuperClass init！”，说明并没有触发类org.fenixsoft.classloading.SuperClass的初始化阶段。但是<u>这段代码里面触发了另一个名为“[Lorg.fenixsoft.classloading.SuperClass”的类的初始化阶段，对于用户代码来说，这并不是一个合法的类型名称，它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建动作由字节码指令newarray触发</u>。

​	这个类代表了一个元素类型为org.fenixsoft.classloading.SuperClass的一维数组，数组中应有的属性和方法（用户可直接使用的只有被修饰为public的length属性和`clone()`方法）都实现在这个类里。<u>Java语言中对数组的访问要比C/C++相对安全，很大程度上就是因为这个类包装了数组元素的访问[^15]，而C/C++中则是直接翻译为对数组指针的移动</u>。<u>在Java语言里，当检查到发生数组越界时会抛出java.lang.ArrayIndexOutOfBoundsException异常，避免了直接造成非法内存访问</u>。

​	代码清单7-3　被动引用的例子之三

```java
package org.fenixsoft.classloading;
/**
* 被动使用类字段演示三：
* 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的
类的初始化
**/
public class ConstClass {
  static {
    System.out.println("ConstClass init!");
  }
  public static final String HELLOWORLD = "hello world";
}
/**
* 非主动使用类字段演示
**/
public class NotInitialization {
  public static void main(String[] args) {
    System.out.println(ConstClass.HELLOWORLD);
  }
}
```

​	上述代码运行之后，也没有输出“ConstClass init！”，这是因为虽然在Java源码中确实引用了ConstClass类的常量HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“helloworld”直接存储在NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用，实际都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口，这两个类在编译成Class文件后就已不存在任何联系了。

​	接口的加载过程与类加载过程稍有不同，针对接口需要做一些特殊说明：接口也有初始化过程，这点与类是一致的，上面的代码都是用静态语句块“static{}”来输出初始化信息的，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成“\<clinit\>()”类构造器[^16]，用于初始化接口中所定义的成员变量。**接口与类真正有所区别的是前面讲述的六种“有且仅有”需要触发初始化场景中的第三种：当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。**

[^15]: 准确地说，越界检查不是封装在数组元素访问的类中，而是封装在数组访问的xaload、xastore字节码指令中。
[^16]: 关于类构造器＜clinit＞()和方法构造器＜init＞()的生成过程和作用，可参见第10章的相关内容。

## 7.3 类加载的过程

​	接下来我们会详细了解Java虚拟机中类加载的全过程，即加载、验证、准备、解析和初始化这五个阶段所执行的具体动作。

### 7.3.1 加载

个人小结：

+ **数组类本身不通过类加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但数组类与类加载器仍然有很密切的关系，因为数组类的元素类型（Element Type，指的是数组去掉所有维度的类型）最终还是要靠类加载器来完成加载**
+ 加载阶段结束后，Java虚拟机外部的二进制字节流就按照虚拟机所设定的格式存储在方法区之中了，方法区中的数据存储格式完全由虚拟机实现自行定义，《Java虚拟机规范》未规定此区域的具体数据结构。类型数据妥善安置在**方法区**之后，会在Java**堆内存**中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的类型数据的外部接口。

---

​	“加载”（Loading）阶段是整个“类加载”（Class Loading）过程中的一个阶段，希望读者没有混淆这两个看起来很相似的名词。在加载阶段，Java虚拟机需要完成以下三件事情：

1. **通过一个类的全限定名来获取定义此类的二进制字节流。**

2. **将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。**

3. **在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。**

​	《Java虚拟机规范》对这三点要求其实并不是特别具体，留给虚拟机实现与Java应用的灵活度都是相当大的。例如“通过一个类的全限定名来获取定义此类的二进制字节流”这条规则，它并没有指明二进制字节流必须得从某个Class文件中获取，确切地说是根本没有指明要从哪里获取、如何获取。仅仅这一点空隙，Java虚拟机的使用者们就可以在加载阶段搭构建出一个相当开放广阔的舞台，Java发展历程中，充满创造力的开发人员则在这个舞台上玩出了各种花样，许多举足轻重的Java技术都建立在这一基础之上，例如：

+ 从ZIP压缩包中读取，这很常见，最终成为日后JAR、EAR、WAR格式的基础。

+ 从网络中获取，这种场景最典型的应用就是Web Applet。

+ **运行时计算生成，这种场景使用得最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass()来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流。**

+ 由其他文件生成，典型场景是JSP应用，由JSP文件生成对应的Class文件。

+ 从数据库中读取，这种场景相对少见些，例如有些中间件服务器（如SAP Netweaver）可以选择把程序安装到数据库中来完成程序代码在集群间的分发。

+ **可以从加密文件中获取，这是典型的防Class文件被反编译的保护措施，通过加载时解密Class文件来保障程序运行逻辑不被窥探。**

+ ……

​	相对于类加载过程的其他阶段，非数组类型的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的阶段。加载阶段既可以使用Java虚拟机里内置的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员通过定义自己的类加载器去控制字节流的获取方式（重写一个类加载器的`findClass()`或`loadClass()`方法），实现根据自己的想法来赋予应用程序获取运行代码的动态性。

​	对于数组类而言，情况就有所不同，**数组类本身不通过类加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但数组类与类加载器仍然有很密切的关系，因为数组类的元素类型（Element Type，指的是数组去掉所有维度的类型）最终还是要靠类加载器来完成加载**，一个数组类（下面简称为C）创建过程遵循以下规则：

+ <u>如果数组的组件类型（Component Type，指的是数组去掉一个维度的类型，注意和前面的元素类型区分开来）是引用类型，那就递归采用本节中定义的加载过程去加载这个组件类型，数组C将被标识在加载该组件类型的类加载器的类名称空间上（这点很重要，在7.4节会介绍，**一个类型必须与类加载器一起确定唯一性**）。</u>
+ **如果数组的组件类型不是引用类型（例如int[]数组的组件类型为int），Java虚拟机将会把数组C标记为与引导类加载器关联**。
+ 数组类的可访问性与它的组件类型的可访问性一致，如果组件类型不是引用类型，它的数组类的可访问性将默认为public，可被所有的类和接口访问到。

​	<u>加载阶段结束后，Java虚拟机外部的二进制字节流就按照虚拟机所设定的格式存储在方法区之中了，方法区中的数据存储格式完全由虚拟机实现自行定义，《Java虚拟机规范》未规定此区域的具体数据结构。类型数据妥善安置在**方法区**之后，会在Java**堆内存**中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的类型数据的外部接口。</u>

​	<u>加载阶段与连接阶段的部分动作（如一部分字节码文件格式验证动作）是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始，但这些夹在加载阶段之中进行的动作，仍然属于连接阶段的一部分，这两个阶段的开始时间仍然保持着固定的先后顺序</u>。

### 7.3.2 验证

个人小结：

+ 从整体上看，验证阶段大致上会完成下面四个阶段的检验动作：**文件格式验证、元数据验证、字节码验证和符号引用验证**

---

​	验证是连接阶段的第一步，这一阶段的目的是确保Class文件的字节流中包含的信息符合《Java虚拟机规范》的全部约束要求，保证这些信息被当作代码运行后不会危害虚拟机自身的安全。

​	Java语言本身是相对安全的编程语言（起码对于C/C++来说是相对安全的），使用纯粹的Java代码无法做到诸如访问数组边界以外的数据、将一个对象转型为它并未实现的类型、跳转到不存在的代码行之类的事情，如果尝试这样去做了，编译器会毫不留情地抛出异常、拒绝编译。但前面也曾说过，Class文件并不一定只能由Java源码编译而来，它可以使用包括靠键盘0和1直接在二进制编辑器中敲出Class文件在内的任何途径产生。上述Java代码无法做到的事情在字节码层面上都是可以实现的，至少语义上是可以表达出来的。Java虚拟机如果不检查输入的字节流，对其完全信任的话，很可能会因为载入了有错误或有恶意企图的字节码流而导致整个系统受攻击甚至崩溃，所以验证字节码是Java虚拟机保护自身的一项必要措施。

​	验证阶段是非常重要的，这个阶段是否严谨，直接决定了Java虚拟机是否能承受恶意代码的攻击，从代码量和耗费的执行性能的角度上讲，验证阶段的工作量在虚拟机的类加载过程中占了相当大的比重。但是《Java虚拟机规范》的早期版本（第1、2版）对这个阶段的检验指导是相当模糊和笼统的，规范中仅列举了一些对Class文件格式的静态和结构化的约束，要求虚拟机验证到输入的字节流如不符合Class文件格式的约束，就应当抛出一个java.lang.VerifyError异常或其子类异常，但具体应当检查哪些内容、如何检查、何时进行检查等，都没有足够具体的要求和明确的说明。<u>直到2011年《Java虚拟机规范（Java SE 7版）》出版，规范中大幅增加了验证过程的描述（篇幅从不到10页增加到130页），这时验证阶段的约束和验证规则才变得具体起来</u>。受篇幅所限，本书中无法逐条规则去讲解，但**从整体上看，验证阶段大致上会完成下面四个阶段的检验动作：文件格式验证、元数据验证、字节码验证和符号引用验证**。

1. 文件格式验证

   ​	第一阶段要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理。这一阶段可能包括下面这些验证点：

   + 是否以魔数0xCAFEBABE开头。
   + 主、次版本号是否在当前Java虚拟机接受范围之内。
   + 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）。
   + 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
   + **CONSTANT_Utf8_info型的常量中是否有不符合UTF-8编码的数据**。
   + Class文件中各个部分及文件本身是否有被删除的或附加的其他信息。
   + ……

   ​	实际上第一阶段的验证点还远不止这些，上面所列的只是从HotSpot虚拟机源码[^17]中摘抄的一小部分内容，<u>该验证阶段的主要目的是保证输入的字节流能正确地解析并存储于方法区之内，格式上符合描述一个Java类型信息的要求</u>。这阶段的验证是基于二进制字节流进行的，只有通过了这个阶段的验证之后，这段字节流才被允许进入Java虚拟机内存的方法区中进行存储，所以<u>后面的三个验证阶段全部是基于方法区的存储结构上进行的，不会再直接读取、操作字节流了</u>。

2. 元数据验证

   ​	第二阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合《Java语言规范》的要求，这个阶段可能包括的验证点如下：

   + 这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）。
   + 这个类的父类是否继承了不允许被继承的类（被final修饰的类）。
   + 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
   + 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。
   + ……

   第二阶段的主要目的是对类的元数据信息进行**语义校验**，保证不存在与《Java语言规范》定义相悖的元数据信息。

3. 字节码验证

   ​	第三阶段是整个验证过程中最复杂的一个阶段，主要目的是<u>通过数据流分析和控制流分析，确定程序**语义**是合法的、符合逻辑的</u>。在第二阶段对元数据信息中的数据类型校验完毕以后，这阶段就要对类的方法体（Class文件中的Code属性）进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为，例如：

   + 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似于“在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中”这样的情况。
   + 保证任何跳转指令都不会跳转到方法体以外的字节码指令上。
   + 保证方法体中的类型转换总是有效的，例如可以把一个子类对象赋值给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。
   + ……

   ​	如果一个类型中有方法体的字节码没有通过字节码验证，那它肯定是有问题的；但如果一个方法体通过了字节码验证，也仍然不能保证它一定就是安全的。即使字节码验证阶段中进行了再大量、再严密的检查，也依然不能保证这一点。这里涉及了离散数学中一个很著名的问题——“停机问题”（Halting Problem）[^18]，即不能通过程序准确地检查出程序是否能在有限的时间之内结束运行。在我们讨论字节码校验的上下文语境里，通俗一点的解释是通过程序去校验程序逻辑是无法做到绝对准确的，不可能用程序来准确判定一段程序是否存在Bug。

   ​	由于数据流分析和控制流分析的高度复杂性，Java虚拟机的设计团队为了避免过多的执行时间消耗在字节码验证阶段中，在JDK 6之后的Javac编译器和Java虚拟机里进行了一项联合优化，把尽可能多的校验辅助措施挪到Javac编译器里进行。具体做法是给方法体Code属性的属性表中新增加了一项名为“StackMapTable”的新属性，这项属性描述了方法体所有的基本块（Basic Block，指按照控制流拆分的代码块）开始时本地变量表和操作栈应有的状态，在字节码验证期间，Java虚拟机就不需要根据程序推导这些状态的合法性，只需要检查StackMapTable属性中的记录是否合法即可。这样就将字节码验证的类型推导转变为类型检查，从而节省了大量校验时间。理论上StackMapTable属性也存在错误或被篡改的可能，所以是否有可能在恶意篡改了Code属性的同时，也生成相应的StackMapTable属性来骗过虚拟机的类型校验，则是虚拟机设计者们需要仔细思考的问题。

   ​	JDK 6的HotSpot虚拟机中提供了`-XX：-UseSplitVerifier`选项来关闭掉这项优化，或者使用参数`-XX：+FailOverToOldVerifier`要求在类型校验失败的时候退回到旧的类型推导方式进行校验。而到了JDK 7之后，尽管虚拟机中仍然保留着类型推导验证器的代码，但是<u>对于主版本号大于50（对应JDK6）的Class文件，使用类型检查来完成数据流分析校验则是唯一的选择，不允许再退回到原来的类型推导的校验方式</u>。

4. 符号引用验证

   ​	最后一个阶段的校验行为发生在虚拟机将符号引用转化为直接引用[^19]的时候，这个转化动作将在连接的第三阶段——解析阶段中发生。符号引用验证可以看作是对类自身以外（常量池中的各种符号引用）的各类信息进行匹配性校验，通俗来说就是，该类是否缺少或者被禁止访问它依赖的某些外部类、方法、字段等资源。本阶段通常需要校验下列内容：

   + 符号引用中通过字符串描述的全限定名是否能找到对应的类。
   + 在指定类中是否存在符合方法的字段描述符及简单名称所描述的方法和字段。
   + 符号引用中的类、字段、方法的可访问性（private、protected、public、\<package\>）是否可被当前类访问。
   + ……

   ​	符号引用验证的主要目的是确保解析行为能正常执行，如果无法通过符号引用验证，Java虚拟机将会抛出一个java.lang.IncompatibleClassChangeError的子类异常，典型的如：java.lang.IllegalAccessError、java.lang.NoSuchFieldError、java.lang.NoSuchMethodError等。

   ​	<u>验证阶段对于虚拟机的类加载机制来说，是一个非常重要的、但却不是必须要执行的阶段，因为验证阶段只有通过或者不通过的差别，只要通过了验证，其后就对程序运行期没有任何影响了。如果程序运行的全部代码（包括自己编写的、第三方包中的、从外部加载的、动态生成的等所有代码）都已经被反复使用和验证过，在生产环境的实施阶段就可以考虑使用`-Xverify：none`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。</u>

[^17]:JDK 12源码中的位置：srcÄhotspotÄshareÄclassfileÄclassFileParser.cpp。
[^18]:停机问题就是判断任意一个程序是否会在有限的时间之内结束运行的问题。如果这个问题可以在有限的时间之内解决，可以有一个程序判断其本身是否会停机并做出相反的行为。这时候显然不管停机问题的结果是什么都不会符合要求，所以这是一个不可解的问题。具体的证明过程可参考链接http://zh.wikipedia.org/zh/停机问题。
[^19]:关于符号引用和直接引用的具体解释，见7.3.4节。

### 7.3.3 准备

个人小结：

+ **在JDK 7及之前，HotSpot使用永久代来实现方法区；而在JDK 8及之后，类变量则会随着Class对象一起存放在Java堆中，这时候“类变量在方法区”就完全是一种对逻辑概念的表述了。**
+ 如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量值就会被初始化为ConstantValue属性所指定的初始值

---

​	准备阶段是正式为类中定义的变量（即静态变量，被static修饰的变量）分配内存并设置类变量初始值的阶段，从概念上讲，这些变量所使用的内存都应当在方法区中进行分配，但必须注意到方法区本身是一个逻辑上的区域，**在JDK 7及之前，HotSpot使用永久代来实现方法区**时，实现是完全符合这种逻辑概念的；**而在JDK 8及之后，类变量则会随着Class对象一起存放在Java堆中，这时候“类变量在方法区”就完全是一种对逻辑概念的表述了**，关于这部分内容，笔者已在4.3.1节介绍并且验证过。

​	关于准备阶段，还有两个容易产生混淆的概念笔者需要着重强调，首先是这时候进行内存分配的仅包括类变量，而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。其次是这里所说的初始值“通常情况”下是数据类型的零值，假设一个类变量的定义为：

```java
public static int value = 123;
```

​	那变量<u>value在**准备阶段**过后的初始值为0而不是123，因为这时尚未开始执行任何Java方法，而把value赋值为123的putstatic指令是程序被编译后，存放于类构造器\<clinit\>()方法之中，所以把value赋值为123的动作要到类的**初始化**阶段才会被执行</u>。表7-1列出了Java中所有基本数据类型的零值。

​	表7-1　基本数据类型的零值

| 数据类型 | 零值      | 数据类型  | 零值  |
| -------- | --------- | --------- | ----- |
| int      | 0         | boolean   | false |
| long     | 0L        | float     | 0.0f  |
| short    | (short) 0 | double    | 0.0d  |
| char     | '\u0000'  | reference | null  |
| byte     | (byte) 0  |           |       |

​	上面提到在“通常情况”下初始值是零值，那言外之意是相对的会有某些“特殊情况”：**如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量值就会被初始化为ConstantValue属性所指定的初始值**，假设上面类变量value的定义修改为：

```java
public static final int value = 123;
```

​	编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据Con-stantValue的设置将value赋值为123。

### 7.3.4 解析

个人小结：

+ 解析阶段是Java虚拟机将常量池内的**符号引用**替换为**直接引用**的过程
+ **符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到虚拟机内存当中的内容。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在《Java虚拟机规范》的Class文件格式中。**
+ **直接引用（Direct References）：直接引用是可以直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局直接相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在虚拟机的内存中存在。**
+ 除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存，譬如在运行时直接引用常量池中的记录，并把常量标识为已解析状态，从而避免解析动作重复进行。无论是否真正执行了多次解析动作，Java虚拟机都需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直能够成功；同样地，如果第一次解析失败了，其他指令对这个符号的解析请求也应该收到相同的异常，哪怕这个请求的符号在后来已成功加载进Java虚拟机内存之中
+ ​	**如果我们说一个D拥有C的访问权限，那就意味着以下3条规则中至少有其中一条成立：**
  + **被访问类C是public的，并且与访问类D处于同一个模块。**
  + **被访问类C是public的，不与访问类D处于同一个模块，但是被访问类C的模块允许被访问类D的模块进行访问。**
  + **被访问类C不是public的，但是它与访问类D处于同一个包中。**
+ 在JDK 9之前，Java接口中的所有方法都默认是public的，也没有模块化的访问约束，所以不存在访问权限的问题，接口方法的符号解析就不可能抛出java.lang.IllegalAccessError异常。但在**JDK 9中增加了接口的静态私有方法，也有了模块化的访问约束，所以从JDK 9起，接口方法的访问也完全有可能因访问权限控制而出现java.lang.IllegalAccessError异常**

---

​	<u>解析阶段是Java虚拟机将常量池内的**符号引用**替换为**直接引用**的过程</u>，符号引用在第6章讲解Class文件格式的时候已经出现过多次，在Class文件中它以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现，那解析阶段中所说的直接引用与符号引用又有什么关联呢？

+ **符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到虚拟机内存当中的内容。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在《Java虚拟机规范》的Class文件格式中。**
+ **直接引用（Direct References）：直接引用是可以直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局直接相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在虚拟机的内存中存在。**

​	<u>《Java虚拟机规范》之中并未规定解析阶段发生的具体时间，只要求了在执行ane-warray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invoke-special、invokestatic、invokevirtual、ldc、ldc_w、ldc2_w、multianewarray、new、putfield和putstatic这17个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析</u>。所以虚拟机实现可以根据需要来自行判断，到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用将要被使用前才去解析它。

​	类似地，对方法或者字段的访问，也会在解析阶段中对它们的可访问性（public、protected、private、\<package\>）进行检查，至于其中的约束规则已经是Java语言的基本常识，笔者就不再赘述了。

​	对同一个符号引用进行多次解析请求是很常见的事情，<u>除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存，譬如在运行时直接引用常量池中的记录，并把常量标识为已解析状态，从而避免解析动作重复进行。无论是否真正执行了多次解析动作，Java虚拟机都需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直能够成功；同样地，如果第一次解析失败了，其他指令对这个符号的解析请求也应该收到相同的异常，哪怕这个请求的符号在后来已成功加载进Java虚拟机内存之中</u>。

​	不过对于invokedynamic指令，上面的规则就不成立了。当碰到某个前面已经由invokedynamic指令触发过解析的符号引用时，并不意味着这个解析结果对于其他invokedynamic指令也同样生效。因为invokedynamic指令的目的本来就是用于动态语言支持[^20]，它对应的引用称为“动态调用点限定符（Dynamically-Computed Call Site Specifier）”，这里“动态”的含义是指必须等到程序实际运行到这条指令时，解析动作才能进行。相对地，其余可触发解析的指令都是“静态”的，可以在刚刚完成加载阶段，还没有开始执行代码时就提前进行解析。

​	<u>解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这7类符号引用进行，分别对应于常量池的CONSTANT_Class_info、CON-STANT_Fieldref_info、CONSTANT_Methodref_info、CONSTANT_InterfaceMethodref_info、CONSTANT_MethodType_info、CONSTANT_MethodHandle_info、CONSTANT_Dyna-mic_info和CONSTANT_InvokeDynamic_info 8种常量类型[^21]</u>。下面笔者将讲解前4种引用的解析过程，对于后4种，它们都和动态语言支持密切相关，由于Java语言本身是一门静态类型语言，在没有讲解清楚invokedynamic指令的语意之前，我们很难将它们直观地和现在的Java语言语法对应上，因此笔者将延后到第8章介绍动态语言调用时一起分析讲解。

1. 类或接口的解析

   ​	假设当前代码所处的类为D，如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用，那虚拟机完成整个解析的过程需要包括以下3个步骤：

   1. 如果C不是一个数组类型，那虚拟机将会把代表N的全限定名传递给D的类加载器去加载这个类C。在加载过程中，由于元数据验证、字节码验证的需要，又可能触发其他相关类的加载动作，例如加载这个类的父类或实现的接口。一旦这个加载过程出现了任何异常，解析过程就将宣告失败。

   2. 如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似“[Ljava/lang/Integer”的形式，那将会按照第一点的规则加载数组元素类型。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表该数组维度和元素的数组对象。

   3. 如果上面两步没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成前还要进行符号引用验证，确认D是否具备对C的访问权限。如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

   ​	针对上面第3点访问权限验证，在JDK 9引入了模块化以后，一个public类型也不再意味着程序任何位置都有它的访问权限，我们还必须检查模块间的访问权限。

   ​	**如果我们说一个D拥有C的访问权限，那就意味着以下3条规则中至少有其中一条成立：**

   + **被访问类C是public的，并且与访问类D处于同一个模块。**
   + **被访问类C是public的，不与访问类D处于同一个模块，但是被访问类C的模块允许被访问类D的模块进行访问。**
   + **被访问类C不是public的，但是它与访问类D处于同一个包中。**

   ​	在后续涉及可访问性时，都必须考虑模块间访问权限隔离的约束，即以上列举的3条规则，这些内容在后面就不再复述了。

2. 字段解析

   ​	要解析一个未被解析过的字段符号引用，首先将会对字段表内class_index[^22]项中索引的CONSTANT_Class_info符号引用进行解析，也就是字段所属的类或接口的符号引用。如果在解析这个类或接口符号引用的过程中出现了任何异常，都会导致字段符号引用解析的失败。如果解析成功完成，那把这个字段所属的类或接口用C表示，《Java虚拟机规范》要求按照如下步骤对C进行后续字段的搜索：

   1. 如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
   2. 否则，如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
   3. 否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。
   4. 否则，查找失败，抛出java.lang.NoSuchFieldError异常。

   ​	如果查找过程成功返回了引用，将会对这个字段进行权限验证，如果发现不具备对字段的访问权限，将抛出java.lang.IllegalAccessError异常。

   ​	以上解析规则能够确保Java虚拟机获得字段唯一的解析结果，但<u>在实际情况中，Javac编译器往往会采取比上述规范更加严格一些的约束，譬如有一个同名字段同时出现在某个类的接口和父类当中，或者同时在自己或父类的多个接口中出现，按照解析规则仍是可以确定唯一的访问字段，但Javac编译器就可能直接拒绝其编译为Class文件</u>。在代码清单7-4中演示了这种情况，如果注释了Sub类中的“public static int A=4；”，接口与父类同时存在字段A，那Oracle公司实现的Javac编译器将提示“Thefield Sub.A is ambiguous”，并且会拒绝编译这段代码。

   ​	代码清单7-4　字段解析

   ```java
   package org.fenixsoft.classloading;
   public class FieldResolution {
     interface Interface0 {
       int A = 0;
     }
     interface Interface1 extends Interface0 {
       int A = 1;
     }
     interface Interface2 {
       int A = 2;
     }
     static class Parent implements Interface1 {
       public static int A = 3;
     }
     static class Sub extends Parent implements Interface2 {
       public static int A = 4;
     }
     public static void main(String[] args) {
       System.out.println(Sub.A);
     }
   }
   ```

3. 方法解析

   ​	方法解析的第一个步骤与字段解析一样，也是需要先解析出方法表的class_index[^23]项中索引的方法所属的类或接口的符号引用，如果解析成功，那么我们依然用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的方法搜索：

   1. 由于Class文件格式中类的方法和接口的方法符号引用的常量类型定义是分开的，如果在类的方法表中发现class_index中索引的C是个接口的话，那就直接抛出java.lang.IncompatibleClassChangeError异常。
   2. 如果通过了第一步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
   3. 否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
   4. 否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时候查找结束，抛出java.lang.AbstractMethodError异常。
   5. 否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。

   ​	最后，如果查找过程成功返回了直接引用，将会对这个方法进行权限验证，如果发现不具备对此方法的访问权限，将抛出java.lang.IllegalAccessError异常。

4. 接口方法解析

   ​	接口方法也是需要先解析出接口方法表的class_index[^24]项中索引的方法所属的类或接口的符号引用，如果解析成功，依然用C表示这个接口，接下来虚拟机将会按照如下步骤进行后续的接口方法搜索：

   1. 与类的方法解析相反，如果在接口方法表中发现class_index中的索引C是个类而不是接口，那么就直接抛出java.lang.IncompatibleClassChangeError异常。
   2. 否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
   3. 否则，在接口C的父接口中递归查找，直到java.lang.Object类（接口方法的查找范围也会包括Object类中的方法）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。
   4. 对于规则3，由于Java的接口允许多重继承，如果C的不同父接口中存有多个简单名称和描述符都与目标相匹配的方法，那将会从这多个方法中返回其中一个并结束查找，《Java虚拟机规范》中并没有进一步规则约束应该返回哪一个接口方法。但与之前字段查找类似地，不同发行商实现的Javac编译器有可能会按照更严格的约束拒绝编译这种代码来避免不确定性。
   5. 否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError异常。

   ​	<u>在JDK 9之前，Java接口中的所有方法都默认是public的，也没有模块化的访问约束，所以不存在访问权限的问题，接口方法的符号解析就不可能抛出java.lang.IllegalAccessError异常。但在JDK 9中增加了接口的静态私有方法，也有了模块化的访问约束，所以从JDK 9起，接口方法的访问也完全有可能因访问权限控制而出现java.lang.IllegalAccessError异常</u>。

[^20]:invokedynamic指令是在JDK 7时加入到字节码中的，当时确实只为了做动态语言（如JRuby、Scala）支持，Java语言本身并不会用到它。而到了JDK 8时代，Java有了Lambda表达式和接口的默认方法，它们在底层调用时就会用到invokedynamic指令，这时再提动态语言支持其实已不完全切合，我们就只把它当个代称吧。笔者将会在第8章中介绍这部分内容。
[^21]:严格来说，CONSTANT_String_info这种类型的常量也有解析过程，但是很简单而且直观，不再做独立介绍。
[^22]:参见第6章中关于CONSTANT_Fieldref_info常量的相关内容。
[^23]:参见第6章关于CONSTANT_Methodref_info常量的相关内容。
[^24]:参见第6章中关于CONSTANT_InterfaceMethodref_info常量的相关内容。

### 7.3.5 初始化

个人小结：

+ 进行准备阶段时，变量已经赋过一次系统要求的初始零值，而在初始化阶段，则会根据程序员通过程序编码制定的主观计划去初始化类变量和其他资源。
+ 初始化阶段就是执行类构造器\<clinit\>()方法的过程。\<clinit\>()并不是程序员在Java代码中直接编写的方法，它是Javac编译器的自动生成物。
+ \<clinit\>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定的，**静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问**
+ \<clinit\>()方法与类的构造函数（即在虚拟机视角中的实例构造器\<init\>()方法）不同，它不需要显式地调用父类构造器，**Java虚拟机会保证在子类的\<clinit\>()方法执行前，父类的\<clinit\>()方法已经执行完毕**。因此在Java虚拟机中第一个被执行的\<clinit\>()方法的类型肯定是java.lang.Object
+ **`<clinit>()`方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。 **
+ **接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成\<clinit\>()方法**。<u>但接口与类不同的是，执行接口的\<clinit>()方法不需要先执行父接口的\<clinit\>()方法，因为只有当父接口中定义的变量被使用时，父接口才会被初始化。此外，接口的实现类在初始化时也一样不会执行接口的\<clinit\>()方法</u>。
+ <u>**Java虚拟机必须保证一个类的\<clinit\>()方法在多线程环境中被正确地加锁同步，如果多个线程同时去初始化一个类，那么只会有其中一个线程去执行这个类的\<clinit\>()方法，其他线程都需要阻塞等待，直到活动线程执行完毕\<clinit\>()方法**</u>。**如果在一个类的\<clinit\>()方法中有耗时很长的操作，那就可能造成多个进程阻塞**[^26]，在实际应用中这种阻塞往往是很隐蔽的

---

​	类的初始化阶段是类加载过程的最后一个步骤，之前介绍的几个类加载的动作里，除了在加载阶段用户应用程序可以通过自定义类加载器的方式局部参与外，其余动作都完全由Java虚拟机来主导控制。直到初始化阶段，Java虚拟机才真正开始执行类中编写的Java程序代码，将主导权移交给应用程序。

​	**进行准备阶段时，变量已经赋过一次系统要求的初始零值，而在初始化阶段，则会根据程序员通过程序编码制定的主观计划去初始化类变量和其他资源**。我们也可以从另外一种更直接的形式来表达：**初始化阶段就是执行类构造器\<clinit\>()方法的过程。\<clinit\>()并不是程序员在Java代码中直接编写的方法，它是Javac编译器的自动生成物**，但我们非常有必要了解这个方法具体是如何产生的，以及\<clinit\>()方法执行过程中各种可能会影响程序运行行为的细节，这部分比起其他类加载过程更贴近于普通的程序开发人员的实际工作[^25]。

+ **\<clinit\>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问**，如代码清单7-5所示。

  ​	代码清单7-5　非法前向引用变量

  ```java
  public class Test {
    static {
      i = 0; // 给变量复制可以正常编译通过
      System.out.print(i); // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
  }
  ```

+ **\<clinit\>()方法与类的构造函数（即在虚拟机视角中的实例构造器\<init\>()方法）不同，它不需要显式地调用父类构造器，Java虚拟机会保证在子类的\<clinit\>()方法执行前，父类的\<clinit\>()方法已经执行完毕。因此在Java虚拟机中第一个被执行的\<clinit\>()方法的类型肯定是java.lang.Object**。

+ 由于父类的\<clinit\>()方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作，如代码清单7-6中，字段B的值将会是2而不是1。

  ​	代码清单7-6　\<clinit\>()方法执行顺序

  ```java
  static class Parent {
    public static int A = 1;
    static {
      A = 2;
    }
  }
  static class Sub extends Parent {
    public static int B = A;
  }
  public static void main(String[] args) {
    System.out.println(Sub.B);
  }
  ```

+ **\<clinit\>()方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成\<clinit\>()方法。**

+ **接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成\<clinit\>()方法**。<u>但接口与类不同的是，执行接口的\<clinit>()方法不需要先执行父接口的\<clinit\>()方法，因为只有当父接口中定义的变量被使用时，父接口才会被初始化。此外，接口的实现类在初始化时也一样不会执行接口的\<clinit\>()方法</u>。

+ **Java虚拟机必须保证一个类的\<clinit\>()方法在多线程环境中被正确地加锁同步，如果多个线程同时去初始化一个类，那么只会有其中一个线程去执行这个类的\<clinit\>()方法，其他线程都需要阻塞等待，直到活动线程执行完毕\<clinit\>()方法。如果在一个类的\<clinit\>()方法中有耗时很长的操作，那就可能造成多个进程阻塞[^26]，在实际应用中这种阻塞往往是很隐蔽的**。代码清单7-7演示了这种场景。

  ​	代码清单7-7　字段解析

  ```java
  static class DeadLoopClass {
    static {
      // 如果不加上这个if语句，编译器将提示“Initializer does not complete normally”,并拒绝编译
        if (true) {
          System.out.println(Thread.currentThread() + "init DeadLoopClass");
          while (true) {
          }
        }
    }
  }
  public static void main(String[] args) {
    Runnable script = new Runnable() {
      public void run() {
        System.out.println(Thread.currentThread() + "start");
        DeadLoopClass dlc = new DeadLoopClass();
        System.out.println(Thread.currentThread() + " run over");
      }
    };
    Thread thread1 = new Thread(script);
    Thread thread2 = new Thread(script);
    thread1.start();
    thread2.start();
  }
  ```

  ​	运行结果如下，一条线程在死循环以模拟长时间操作，另外一条线程在阻塞等待：

  ```java
  Thread[Thread-0,5,main]start
  Thread[Thread-1,5,main]start
  Thread[Thread-0,5,main]init DeadLoopClass
  ```

[^25]:这里的讨论只限于Java语言编译产生的Class文件，不包括其他Java虚拟机语言。
[^26]:需要注意，其他线程虽然会被阻塞，但如果执行＜clinit＞()方法的那条线程退出＜clinit＞()方法后，其他线程唤醒后则不会再次进入＜clinit＞()方法。同一个类加载器下，一个类型只会被初始化一次。

## 7.4 类加载器

> [Java类加载器ClassLoader总结 - 时间朋友 - 博客园 (cnblogs.com)](https://www.cnblogs.com/doit8791/p/5820037.html)

​	<u>Java虚拟机设计团队有意把类加载阶段中的“**通过一个类的全限定名来获取描述该类的二进制字节流**”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需的类。实现这个动作的代码被称为“类加载器”（Class Loader）</u>。

​	类加载器可以说是Java语言的一项创新，它是早期Java语言能够快速流行的重要原因之一。类加载器最初是为了满足Java Applet的需求而设计出来的，在今天用在浏览器上的Java Applet技术基本上已经被淘汰[^27]，但类加载器却在类层次划分、OSGi、程序热部署、代码加密等领域大放异彩，成为Java技术体系中一块重要的基石，可谓是失之桑榆，收之东隅。

[^27]:特指浏览器上的Java Applets，在其他领域，如智能卡上，Java Applets仍然有很广阔的市场。

### 7.4.1 类与类加载器

个人小结：

+ **对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间**。<u>这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等</u>

---

​	类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远超类加载阶段。**对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间**。<u>这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等</u>。

​	这里所指的“相等”，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括了使用instanceof关键字做对象所属关系判定等各种情况。如果没有注意到类加载器的影响，在某些情况下可能会产生具有迷惑性的结果，代码清单7-8中演示了不同的类加载器对instanceof关键字运算的结果的影响。

​	代码清单7-8　不同的类加载器对instanceof关键字运算的结果的影响

```java
/**
* 类加载器与instanceof关键字演示
*
* @author zzm
*/
public class ClassLoaderTest {
  public static void main(String[] args) throws Exception {
    ClassLoader myLoader = new ClassLoader() {
      @Override
        public Class<?> loadClass(String name) throws ClassNotFoundException {
        try {
          String fileName = name.substring(name.lastIndexOf(".") + 1)+".class";
          InputStream is = getClass().getResourceAsStream(fileName);
          if (is == null) {
            return super.loadClass(name);
          }
          byte[] b = new byte[is.available()];
          is.read(b);
          return defineClass(name, b, 0, b.length);
        } catch (IOException e) {
          throw new ClassNotFoundException(name);
        }
      }
    };
    Object obj = myLoader.loadClass("org.fenixsoft.classloading.ClassLoaderTest").newInstance();
    System.out.println(obj.getClass());
    System.out.println(obj instanceof org.fenixsoft.classloading.ClassLoaderTest);
  }
}
```

​	运行结果：

```java
class org.fenixsoft.classloading.ClassLoaderTest
false
```

​	代码清单7-8中构造了一个简单的类加载器，尽管它极为简陋，但是对于这个演示来说已经足够。它可以加载与自己在同一路径下的Class文件，我们使用这个类加载器去加载了一个名为“org.fenixsoft.classloading.ClassLoaderTest”的类，并实例化了这个类的对象。

​	两行输出结果中，从第一行可以看到这个对象确实是类org.fenixsoft.classloading.ClassLoaderTest实例化出来的，但在第二行的输出中却发现这个对象与类org.fenixsoft.classloading.ClassLoaderTest做所属类型检查的时候返回了false。这是因为Java虚拟机中同时存在了两个ClassLoaderTest类，一个是由虚拟机的应用程序类加载器所加载的，另外一个是由我们自定义的类加载器加载的，**虽然它们都来自同一个Class文件，但在Java虚拟机中仍然是两个互相独立的类，做对象所属类型检查时的结果自然为false**。

### 7.4.2 双亲委派模型

个人小结：

+ 站在Java虚拟机的角度来看，只存在两种不同的类加载器：
  + **一种是启动类加载器（BootstrapClassLoader），这个类加载器使用C++语言实现[^28]，是虚拟机自身的一部分；**
  + **另外一种就是其他所有的类加载器，这些类加载器都由Java语言实现，独立存在于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。**
+ 启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器去处理，那直接使用null代替即可。
+ 由于扩展类加载器是由Java代码实现的，开发者可以直接在程序中使用扩展类加载器来加载Class文件
+ 如果应用程序中没有自定义过自己的类加载器，一般情况下应用程序类加载器(Application Class Loader)就是程序中默认的类加载器
+ **双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合（Composition）关系来复用父加载器的代码**。
+ **双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载**

---

​	站在Java虚拟机的角度来看，只存在两种不同的类加载器：

+ **一种是启动类加载器（BootstrapClassLoader），这个类加载器使用C++语言实现[^28]，是虚拟机自身的一部分；**
+ **另外一种就是其他所有的类加载器，这些类加载器都由Java语言实现，独立存在于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。**

​	站在Java开发人员的角度来看，类加载器就应当划分得更细致一些。自JDK 1.2以来，Java一直保持着三层类加载器、双亲委派的类加载架构，尽管这套架构在Java模块化系统出现后有了一些调整变动，但依然未改变其主体结构，我们将在7.5节中专门讨论模块化系统下的类加载器。

​	本节内容将针对JDK 8及之前版本的Java来介绍什么是三层类加载器，以及什么是双亲委派模型。对于这个时期的Java应用，绝大多数Java程序都会使用到以下3个系统提供的类加载器来进行加载。

+ **启动类加载器（Bootstrap Class Loader）**：前面已经介绍过，这个类加载器负责加载存放在<JAVA_HOME>/lib目录，或者被`-Xbootclasspath`参数所指定的路径中存放的，而且是Java虚拟机能够识别的（按照文件名识别，如rt.jar、tools.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机的内存中。**启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器去处理，那直接使用null代替即可**，代码清单7-9展示的就是java.lang.ClassLoader.getClassLoader()方法的代码片段，其中的注释和代码实现都明确地说明了以null值来代表引导类加载器的约定规则。

​	代码清单7-9　ClassLoader.getClassLoader()方法的代码片段

```java
/**
Returns the class loader for the class. Some implementations may use null to represent the bootstrap class loader. */
public ClassLoader getClassLoader() {
  ClassLoader cl = getClassLoader0();
  if (cl == null)
    return null;
  SecurityManager sm = System.getSecurityManager();
  if (sm != null) {
    ClassLoader ccl = ClassLoader.getCallerClassLoader();
    if (ccl != null && ccl != cl && !cl.isAncestor(ccl)) {
      sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);
    }
  }
  return cl;
}
```

+ **扩展类加载器（Extension Class Loader）**：这个类加载器是在类sun.misc.Launcher$ExtClassLoader中以Java代码的形式实现的。它负责加载<JAVA_HOME>/lib/ext目录中，或者被java.ext.dirs系统变量所指定的路径中所有的类库。根据“扩展类加载器”这个名称，就可以推断出这是一种Java系统类库的扩展机制，JDK的开发团队允许用户将具有通用性的类库放置在ext目录里以扩展Java SE的功能，在JDK9之后，这种扩展机制被模块化带来的天然的扩展能力所取代。<u>由于扩展类加载器是由Java代码实现的，开发者可以直接在程序中使用扩展类加载器来加载Class文件</u>。

+ **应用程序类加载器（Application Class Loader）**：这个类加载器由sun.misc.Launcher$AppClassLoader来实现。由于应用程序类加载器是ClassLoader类中的getSystemClassLoader()方法的返回值，所以有些场合中也称它为“系统类加载器”。<u>它负责加载用户类路径（ClassPath）上所有的类库，开发者同样可以直接在代码中使用这个类加载器</u>。**如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。**

![这里写图片描述](https://img-blog.csdn.net/20170811111336362?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	JDK 9之前的Java应用都是由这三种类加载器互相配合来完成加载的，如果用户认为有必要，还可以加入自定义的类加载器来进行拓展，典型的如增加除了磁盘位置之外的Class文件来源，或者通过类加载器实现类的隔离、重载等功能。这些类加载器之间的协作关系“通常”会如图7-2所示。

​	图7-2中展示的各种类加载器之间的层次关系被称为类加载器的“双亲委派模型（Parents DelegationModel）”。**双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合（Composition）关系来复用父加载器的代码。**

​	读者可能注意到前面描述这种类加载器协作关系时，笔者专门用双引号强调这是“通常”的协作关系。类加载器的双亲委派模型在JDK 1.2时期被引入，并被广泛应用于此后几乎所有的Java程序中，但它并不是一个具有强制性约束力的模型，而是Java设计者们推荐给开发者的一种类加载器实现的最佳实践。

​	**双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载**。

​	使用双亲委派模型来组织类加载器之间的关系，一个显而易见的好处就是Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，<u>无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都能够保证是同一个类</u>。反之，如果没有使用双亲委派模型，都由各个类加载器自行去加载的话，如果用户自己也编写了一个名为java.lang.Object的类，并放在程序的ClassPath中，那系统中就会出现多个不同的Object类，Java类型体系中最基础的行为也就无从保证，应用程序将会变得一片混乱。如果读者有兴趣的话，可以尝试去写一个与rt.jar类库中已有类重名的Java类，将会发现它可以正常编译，但永远无法被加载运行[^29]。

​	双亲委派模型对于保证Java程序的稳定运作极为重要，但它的实现却异常简单，用以实现双亲委派的代码只有短短十余行，全部集中在java.lang.ClassLoader的loadClass()方法之中，如代码清单7-10所示。

​	代码清单7-10　双亲委派模型的实现

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
  // 首先，检查请求的类是否已经被加载过了
  Class c = findLoadedClass(name);
  if (c == null) {
    try {
      if (parent != null) {
        c = parent.loadClass(name, false);
      } else {
        c = findBootstrapClassOrNull(name);
      }
    } catch (ClassNotFoundException e) {
      // 如果父类加载器抛出ClassNotFoundException
      // 说明父类加载器无法完成加载请求
    }
    if (c == null) {
      // 在父类加载器无法加载时
      // 再调用本身的findClass方法来进行类加载
      c = findClass(name);
    }
  }
  if (resolve) {
    resolveClass(c);
  }
  return c;
}
```

​	这段代码的逻辑清晰易懂：<u>先检查请求加载的类型是否已经被加载过，若没有则调用父加载器的loadClass()方法，若父加载器为空则默认使用启动类加载器作为父加载器</u>。假如父类加载器加载失败，抛出ClassNotFoundException异常的话，才调用自己的findClass()方法尝试进行加载。

[^28]:这里只限于HotSpot，像MRP、Maxine这些虚拟机，整个虚拟机本身都是由Java编写的，自然BootstrapClassLoader也是由Java语言而不是C++实现的。退一步说，除了HotSpot外的其他两个高性能虚拟机JRockit和J9都有一个代表Bootstrap ClassLoader的Java类存在，但是关键方法的实现仍然是使用JNI回调到C（而不是C++）的实现上，这个Bootstrap ClassLoader的实例也无法被用户获取到。在JDK 9以后，HotSpot虚拟机也采用了类似的虚拟机与Java类互相配合来实现Bootstrap ClassLoader的方式，所以在JDK 9后HotSpot也有一个无法获取实例的代表Bootstrap ClassLoader的Java类存在了。
[^29]:即使自定义了自己的类加载器，强行用defineClass()方法去加载一个以“java.lang”开头的类也不会成功。如果读者尝试这样做的话，将会收到一个由Java虚拟机内部抛出的“java.lang.SecurityException：Prohibited package name：java.lang”异常。

### 7.4.3 破坏双亲委派模型

> [JNDI--百度百科](https://baike.baidu.com/item/JNDI/3792442?fr=aladdin)
>
> JNDI(Java Naming and Directory Interface)是一个[应用程序](https://baike.baidu.com/item/应用程序)设计的API，为开发人员提供了查找和访问各种命名和[目录服务](https://baike.baidu.com/item/目录服务/10413830)的通用、统一的接口，类似JDBC都是构建在抽象层上。现在JNDI已经成为J2EE的标准之一，所有的J2EE容器都必须提供一个JNDI的服务。

个人小结：

+ **在JDK 1.2之后的java.lang.ClassLoader中添加一个新的protected方法findClass()，并引导用户编写的类加载逻辑时尽可能去重写这个方法，而不是在loadClass()中编写代码**。按照loadClass()方法的逻辑，如果父类加载失败，会自动调用自己的findClass()方法来完成加载，这样既不影响用户按照自己的意愿去加载类，又可以保证新写出来的类加载器是符合双亲委派规则的。
+ <u>线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器</u>
+ **OSGi实现模块化热部署的关键是它自定义的类加载器机制的实现，每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换**。

---

​	上文提到过**双亲委派模型并不是一个具有强制性约束的模型，而是Java设计者推荐给开发者们的类加载器实现方式**。在Java的世界中大部分的类加载器都遵循这个模型，但也有例外的情况，直到Java模块化出现为止，双亲委派模型主要出现过3次较大规模“被破坏”的情况。

​	双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK 1.2面世以前的“远古”时代。由于双亲委派模型在JDK 1.2之后才被引入，但是类加载器的概念和抽象类java.lang.ClassLoader则在Java的第一个版本中就已经存在，面对已经存在的用户自定义类加载器的代码，Java设计者们引入双亲委派模型时不得不做出一些妥协，为了兼容这些已有代码，无法再以技术手段避免loadClass()被子类覆盖的可能性，只能**在JDK 1.2之后的java.lang.ClassLoader中添加一个新的protected方法findClass()，并引导用户编写的类加载逻辑时尽可能去重写这个方法，而不是在loadClass()中编写代码**。上节我们已经分析过loadClass()方法，双亲委派的具体逻辑就实现在这里面，<u>按照loadClass()方法的逻辑，如果父类加载失败，会自动调用自己的findClass()方法来完成加载，这样既不影响用户按照自己的意愿去加载类，又可以保证新写出来的类加载器是符合双亲委派规则的</u>。

​	双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷导致的，双亲委派很好地解决了各个类加载器协作时基础类型的一致性问题（越基础的类由越上层的加载器进行加载），基础类型之所以被称为“基础”，是因为它们总是作为被用户代码继承、调用的API存在，但程序设计往往没有绝对不变的完美规则，如果有基础类型又要调用回用户的代码，那该怎么办呢？

​	这并非是不可能出现的事情，一个典型的例子便是JNDI服务，JNDI现在已经是Java的标准服务，它的代码由启动类加载器来完成加载（在JDK 1.3时加入到rt.jar的），肯定属于Java中很基础的类型了。但<u>**JNDI存在的目的就是对资源进行查找和集中管理，它需要调用由其他厂商实现并部署在应用程序的ClassPath下的JNDI服务提供者接口（Service Provider Interface，SPI）的代码**</u>，现在问题来了，启动类加载器是绝不可能认识、加载这些代码的，那该怎么办？

​	为了解决这个困境，Java的设计团队只好引入了一个不太优雅的设计：<u>线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器</u>。

​	有了线程上下文类加载器，程序就可以做一些“舞弊”的事情了。JNDI服务使用这个线程上下文类加载器去加载所需的SPI服务代码，这是一种**父类加载器去请求子类加载器完成类加载**的行为，这种行为实际上是打通了双亲委派模型的层次结构来逆向使用类加载器，已经违背了双亲委派模型的一般性原则，但也是无可奈何的事情。<u>Java中涉及SPI的加载基本上都采用这种方式来完成，例如JNDI、JDBC、JCE、JAXB和JBI等</u>。不过，当SPI的服务提供者多于一个的时候，代码就只能根据具体提供者的类型来硬编码判断，为了消除这种极不优雅的实现方式，在JDK 6时，JDK提供了java.util.ServiceLoader类，以META-INF/services中的配置信息，辅以责任链模式，这才算是给SPI的加载提供了一种相对合理的解决方案。

​	双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的，这里所说的“动态性”指的是一些非常“热”门的名词：**代码热替换（Hot Swap）、模块热部署（Hot Deployment）**等。说白了就是希望Java应用程序能像我们的电脑外设那样，接上鼠标、U盘，不用重启机器就能立即使用，鼠标有问题或要升级就换个鼠标，不用关机也不用重启。对于个人电脑来说，重启一次其实没有什么大不了的，但对于一些生产系统来说，关机重启一次可能就要被列为生产事故，这种情况下热部署就对软件开发者，尤其是大型系统或企业级软件开发者具有很大的吸引力。

​	早在2008年，在Java社区关于模块化规范的第一场战役里，由Sun/Oracle公司所提出的JSR-294[^30]、JSR-277[^31]规范提案就曾败给以IBM公司主导的JSR-291（即OSGi R4.2）提案。尽管Sun/Oracle并不甘心就此失去Java模块化的主导权，随即又再拿出Jigsaw项目迎战，但此时OSGi已经站稳脚跟，成为业界“事实上”的Java模块化标准[^32]。曾经在很长一段时间内，IBM凭借着OSGi广泛应用基础让Jigsaw吃尽苦头，其影响一直持续到Jigsaw随JDK 9面世才算告一段落。而且即使Jigsaw现在已经是Java的标准功能了，它仍需小心翼翼地避开OSGi运行期动态热部署上的优势，仅局限于静态地解决模块间封装隔离和访问控制的问题，这部分内容笔者在7.5节中会继续讲解，现在我们先来简单看一看OSGi是如何通过类加载器实现热部署的。

​	**OSGi实现模块化热部署的关键是它自定义的类加载器机制的实现，每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换**。在OSGi环境下，类加载器不再双亲委派模型推荐的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载请求时，OSGi将按照下面的顺序进行类搜索：

1. 将以java.*开头的类，委派给父类加载器加载。

2. 否则，将委派列表名单内的类，委派给父类加载器加载。

3. 否则，将Import列表中的类，委派给Export这个类的Bundle的类加载器加载。

4. 否则，查找当前Bundle的ClassPath，使用自己的类加载器加载。

5. 否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载。

6. 否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载。

7. 否则，类查找失败。

​	上面的查找顺序中只有开头两点仍然符合双亲委派模型的原则，其余的类查找都是在平级的类加载器中进行的，关于OSGi的其他内容，笔者（书籍作者）就不再展开了。

​	本节中笔者（书籍作者）虽然使用了“被破坏”这个词来形容上述不符合双亲委派模型原则的行为，但这里“被破坏”并不一定是带有贬义的。只要有明确的目的和充分的理由，突破旧有原则无疑是一种创新。正如OSGi中的类加载器的设计不符合传统的双亲委派的类加载器架构，且业界对其为了实现热部署而带来的额外的高复杂度还存在不少争议，但对这方面有了解的技术人员基本还是能达成一个共识，认为<u>OSGi中对类加载器的运用是值得学习的，完全弄懂了OSGi的实现，就算是掌握了类加载器的精粹</u>。

[^30]:JSR-294：Improved Modularity Support in the Java Programming Language（Java编程语言中的改进模块性支持）。
[^31]:JSR-277：Java Module System（Java模块系统）。
[^32]:如果读者对Java模块化之争或者OSGi本身感兴趣，欢迎阅读笔者的另一本书《深入理解OSGi：Equinox原理、应用与最佳实践》。

## 7.5 Java模块化系统

个人小结：

+ JDK 9中的public类型不再意味着程序的所有地方的代码都可以随意访问到它们，模块提供了更精细的可访问性控制，必须明确声明其中哪一些public的类型可以被其他哪一些模块访问，这种访问控制也主要是在类加载过程中完成的。

---

​	在JDK 9中引入的Java模块化系统（Java Platform Module System，JPMS）是对Java技术的一次重要升级，为了能够实现模块化的关键目标——可配置的封装隔离机制，Java虚拟机对类加载架构也做出了相应的变动调整，才使模块化系统得以顺利地运作。JDK 9的模块不仅仅像之前的JAR包那样只是简单地充当代码的容器，除了代码外，Java的模块定义还包含以下内容：

+ 依赖其他模块的列表。
+ 导出的包列表，即其他模块可以使用的列表。
+ 开放的包列表，即其他模块可反射访问模块的列表。
+ 使用的服务列表。
+ 提供服务的实现列表。

​	可配置的封装隔离机制首先要解决JDK 9之前基于类路径（ClassPath）来查找依赖的可靠性问题。此前，如果类路径中缺失了运行时依赖的类型，那就只能等程序运行到发生该类型的加载、链接时才会报出运行的异常。而在JDK 9以后，如果启用了模块化进行封装，模块就可以声明对其他模块的显式依赖，这样Java虚拟机就能够在启动时验证应用程序开发阶段设定好的依赖关系在运行期是否完备，如有缺失那就直接启动失败，从而避免了很大一部分[^33]由于类型依赖而引发的运行时异常。

​	可配置的封装隔离机制还解决了原来类路径上跨JAR文件的public类型的可访问性问题。**JDK 9中的public类型不再意味着程序的所有地方的代码都可以随意访问到它们，模块提供了更精细的可访问性控制，必须明确声明其中哪一些public的类型可以被其他哪一些模块访问，这种访问控制也主要是在类加载过程中完成的**，具体内容笔者（书籍作者）在前文对解析阶段（7.3.4 解析）的讲解中已经介绍过。

[^33]:并不是说模块化下就不可能出现ClassNotFoundExcepiton这类异常了，假如将某个模块中的、原本公开的包中把某些类型移除，但不修改模块的导出信息，这样程序能够顺利启动，但仍然会在运行期出现类加载异常。

### 7.5.1 模块的兼容性

个人小结：

+ **为了使可配置的封装隔离机制能够兼容传统的类路径查找机制，JDK 9提出了与“类路径”（ClassPath）相对应的“模块路径”（ModulePath）的概念。简单来说，就是某个类库到底是模块还是传统的JAR包，只取决于它存放在哪种路径上。只要是放在类路径上的JAR文件，无论其中是否包含模块化信息（是否包含了module-info.class文件），它都会被当作传统的JAR包来对待；相应地，只要放在模块路径上的JAR文件，即使没有使用JMOD后缀，甚至说其中并不包含module-info.class文件，它也仍然会被当作一个模块来对待**。
+ 所有类路径下的JAR文件及其他资源文件，都被视为自动打包在一个匿名模块（Unnamed Module）里。
+ 模块在模块路径的访问规则：模块路径下的具名模块（Named Module）只能访问到它依赖定义中列明依赖的模块和包，匿名模块里所有的内容对具名模块来说都是不可见的，即具名模块看不见传统JAR包的内容。
+ JAR文件在模块路径的访问规则：如果把一个传统的、不包含模块定义的JAR文件放置到模块路径中，它就会变成一个自动模块（Automatic Module）。

---

​	**为了使可配置的封装隔离机制能够兼容传统的类路径查找机制，JDK 9提出了与“类路径”（ClassPath）相对应的“模块路径”（ModulePath）的概念。简单来说，就是某个类库到底是模块还是传统的JAR包，只取决于它存放在哪种路径上。只要是放在类路径上的JAR文件，无论其中是否包含模块化信息（是否包含了module-info.class文件），它都会被当作传统的JAR包来对待；相应地，只要放在模块路径上的JAR文件，即使没有使用JMOD后缀，甚至说其中并不包含module-info.class文件，它也仍然会被当作一个模块来对待**。

​	模块化系统将按照以下规则来保证使用传统类路径依赖的Java程序可以不经修改地直接运行在JDK 9及以后的Java版本上，即使这些版本的JDK已经使用模块来封装了Java SE的标准类库，模块化系统的这套规则也仍然保证了传统程序可以访问到所有标准类库模块中导出的包。

+ JAR文件在类路径的访问规则：**所有类路径下的JAR文件及其他资源文件，都被视为自动打包在一个匿名模块（Unnamed Module）里**，这个匿名模块几乎是没有任何隔离的，它可以看到和使用类路径上所有的包、JDK系统模块中所有的导出包，以及模块路径上所有模块中导出的包。

+ **模块在模块路径的访问规则：模块路径下的具名模块（Named Module）只能访问到它依赖定义中列明依赖的模块和包，匿名模块里所有的内容对具名模块来说都是不可见的，即具名模块看不见传统JAR包的内容。**

+ **JAR文件在模块路径的访问规则：如果把一个传统的、不包含模块定义的JAR文件放置到模块路径中，它就会变成一个自动模块（Automatic Module）**。尽管不包含module-info.class，但自动模块将默认依赖于整个模块路径中的所有模块，因此可以访问到所有模块导出的包，自动模块也默认导出自己所有的包。

​	以上3条规则保证了即使Java应用依然使用传统的类路径，升级到JDK 9对应用来说几乎（类加载器上的变动还是可能会导致少许可见的影响，将在下节介绍）不会有任何感觉，项目也不需要专门为了升级JDK版本而去把传统JAR包升级成模块。

​	除了向后兼容性外，随着JDK 9模块化系统的引入，更值得关注的是它本身面临的模块间的管理和兼容性问题：如果同一个模块发行了多个不同的版本，那只能由开发者在编译打包时人工选择好正确版本的模块来保证依赖的正确性。<u>Java模块化系统目前不支持在模块定义中加入版本号来管理和约束依赖，本身也不支持多版本号的概念和版本选择功能</u>。前面这句话引来过很多的非议，但它确实是Oracle官方对模块化系统的明确的目标说明[^34]。我们不论是在Java命令、Java类库的API抑或是《Java虚拟机规范》定义的Class文件格式里都能轻易地找到证据，表明模块版本应是编译、加载、运行期间都可以使用的。譬如输入“java--list-modules”，会得到明确带着版本号的模块列表：

```java
java.base@12.0.1
java.compiler@12.0.1
java.datatransfer@12.0.1
java.desktop@12.0.1
java.instrument@12.0.1
java.logging@12.0.1
java.management@12.0.1
....
```

​	<u>在JDK 9时加入Class文件格式的Module属性，里面有module_version_index这样的字段，用户可以在编译时使用“javac--module-version”来指定模块版本，在Java类库API中也存在java.lang.module.ModuleDescriptor.Version这样的接口可以在运行时获取到模块的版本号</u>。这一切迹象都证明了Java模块化系统对版本号的支持本可以不局限在编译期。而官方却在Jigsaw的规范文件、JavaOne大会的宣讲和与专家的讨论列表中，都反复强调“JPMS的目的不是代替OSGi”，“JPMS不支持模块版本”这样的话语。

​	*Oracle给出的理由是希望维持一个足够简单的模块化系统，避免技术过于复杂。但结合JCP执行委员会关于的Jigsaw投票中Oracle与IBM、RedHat的激烈冲突[^35]，实在很难让人信服这种设计只是单纯地基于技术原因，而不是厂家之间互相博弈妥协的结果。Jigsaw仿佛在刻意地给OSGi让出一块生存空间，以换取IBM支持或者说不去反对Jigsaw，其代价就是几乎宣告Java模块化系统不可能拥有像OSGi那样支持多版本模块并存、支持运行时热替换、热部署模块的能力，可这却往往是一个应用进行模块化的最大驱动力所在。如果要在JDK 9之后实现这种目的，就只能将OSGi和JPMS混合使用，如图7-4所示，这无疑带来了更高的复杂度。模块的运行时部署、替换能力没有内置在Java模块化系统和Java虚拟机之中，仍然必须通过类加载器去实现，实在不得不说是一个缺憾。*

​	其实Java虚拟机内置的JVMTI接口（java.lang.instrument.Instrumentation）提供了一定程度的运行时修改类的能力（RedefineClass、RetransformClass），但这种修改能力会受到很多限制[^36]，不可能直接用来实现OSGi那样的热替换和多版本并存，用在IntelliJ IDE、Eclipse这些IDE上做HotSwap（是指IDE编辑方法的代码后不需要重启即可生效）倒是非常的合适。也<u>曾经有一个研究性项目Dynamic CodeEvolution VM（DECVM）探索过在虚拟机内部支持运行时类型替换的可行性，允许任意修改已加载到内存中的Class，并不损失任何性能，但可惜已经很久没有更新了，最新版只支持到JDK 7</u>。

​	图7-4　OSGi与JPMS交互[^37]

![img](https://res.infoq.com/articles/java9-osgi-future-modularity-part-2/en/resources/2figure.jpg)

[^34]:源自Jigsaw本身的项目目标定义：http://openjdk.java.net/projects/jigsaw/goals-reqs/03#versioning。
[^35]:具体可参见1.3节对JDK 9期间描述的部分内容。
[^36]:譬如只能修改已有方法的方法体，而不能添加新成员、删除已有成员、修改已有成员的签名等。
[^37]:图片来源：https://www.infoq.com/articles/java9-osgi-future-modularity-part-2/。

### 7.5.2 模块化下的类加载器

> [深入理解Java虚拟机（八）：类加载器与双亲委派模型](https://blog.csdn.net/hu_zhiting/article/details/107320391/)

个人小结：

+ **JDK9中，扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代**。
+ JDK9中，**平台类加载器和应用程序类加载器都不再派生自java.net.URLClassLoader，如果有程序直接依赖了这种继承关系，或者依赖了URLClassLoader类的特定方法，那代码很可能会在JDK 9及更高版本的JDK中崩溃**
+ 

---

​	**为了保证兼容性，JDK 9并没有从根本上动摇从JDK 1.2以来运行了二十年之久的三层类加载器架构以及双亲委派模型**。但是为了模块化系统的顺利施行，模块化下的类加载器仍然发生了一些应该被注意到变动，主要包括以下几个方面。

​	首先，是**扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代。**这其实是一个很顺理成章的变动，既然整个JDK都基于模块化进行构建（原来的rt.jar和tools.jar被拆分成数十个JMOD文件），其中的Java类库就已天然地满足了可扩展的需求，那自然无须再保留<JAVA_HOME>/lib/ext目录，此前使用这个目录或者java.ext.dirs系统变量来扩展JDK功能的机制已经没有继续存在的价值了，用来加载这部分类库的扩展类加载器也完成了它的历史使命。类似地，在<u>新版的JDK中也取消了<JAVA_HOME>/jre目录，因为随时可以组合构建出程序运行所需的JRE来</u>，譬如假设我们只使用java.base模块中的类型，那么随时可以通过以下命令打包出一个“JRE”：

```java
jlink -p $JAVA_HOME/jmods --add-modules java.base --output jre
```

​	其次，**平台类加载器和应用程序类加载器都不再派生自java.net.URLClassLoader，如果有程序直接依赖了这种继承关系，或者依赖了URLClassLoader类的特定方法，那代码很可能会在JDK 9及更高版本的JDK中崩溃**。现在启动类加载器、平台类加载器、应用程序类加载器全都继承于jdk.internal.loader.BuiltinClassLoader，在BuiltinClassLoader中实现了新的模块化架构下类如何从模块中加载的逻辑，以及模块中资源可访问性的处理。两者的前后变化如图7-5和7-6所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713180720558.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMwMzQyMjM=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713180828363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMwMzQyMjM=,size_16,color_FFFFFF,t_70)

​	另外，读者可能已经注意到图7-6中有“BootClassLoader”存在，启动类加载器现在是在Java虚拟机内部和Java类库共同协作实现的类加载器，尽管有了BootClassLoader这样的Java类，但为了与之前的代码保持兼容，所有在获取启动类加载器的场景（譬如Object.class.getClassLoader()）中仍然会返回null来代替，而不会得到BootClassLoader的实例。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713181216470.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMwMzQyMjM=,size_16,color_FFFFFF,t_70)

​	最后，JDK 9中虽然仍然维持着三层类加载器和双亲委派的架构，但类加载的委派关系也发生了变动。当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责那个模块的加载器完成加载，也许这可以算是对双亲委派的第四次破坏。在JDK 9以后的三层类加载器的架构如图7-7所示，请读者对照图7-2进行比较。

​	在Java模块化系统明确规定了三个类加载器负责各自加载的模块，即前面所说的归属关系，如下所示。

+ 启动类加载器负责加载的模块：

  ```java
  java.base java.security.sasl
  java.datatransfer java.xml
  java.desktop jdk.httpserver
  java.instrument jdk.internal.vm.ci
  java.logging jdk.management
  java.management jdk.management.agent
  java.management.rmi jdk.naming.rmi
  java.naming jdk.net
  java.prefs jdk.sctp
  java.rmi jdk.unsupported
  ```

+ 平台类加载器负责加载的模块：

  ```java
  java.activation* jdk.accessibility
  java.compiler* jdk.charsets
  java.corba* jdk.crypto.cryptoki
  java.scripting jdk.crypto.ec
  java.se jdk.dynalink
  java.se.ee jdk.incubator.httpclient
  java.security.jgss jdk.internal.vm.compiler*
  java.smartcardio jdk.jsobject
  java.sql jdk.localedata
  java.sql.rowset jdk.naming.dns
  java.transaction* jdk.scripting.nashorn
  java.xml.bind* jdk.security.auth
  java.xml.crypto jdk.security.jgss
  java.xml.ws* jdk.xml.dom
  java.xml.ws.annotation* jdk.zipfs
  ```

+ 应用程序类加载器负责加载的模块：

  ```java
  jdk.aot jdk.jdeps
  jdk.attach jdk.jdi
  jdk.compiler jdk.jdwp.agent
  jdk.editpad jdk.jlink
  jdk.hotspot.agent jdk.jshell
  jdk.internal.ed jdk.jstatd
  jdk.internal.jvmstat jdk.pack
  jdk.internal.le jdk.policytool
  jdk.internal.opt jdk.rmic
  jdk.jartool jdk.scripting.nashorn.shell
  jdk.javadoc jdk.xml.bind*
  jdk.jcmd jdk.xml.ws*
  jdk.jconsole
  ```

## 7.6 本章小结

​	本章介绍了类加载过程的“加载”“验证”“准备”“解析”和“初始化”这5个阶段中虚拟机进行了哪些动作，还介绍了类加载器的工作原理及其对虚拟机的意义。

​	经过第6、7章的讲解，相信读者已经对如何在Class文件中定义类，以及如何将类加载到虚拟机之中这两个问题有了一个比较系统的了解，第8章我们将探索Java虚拟机的执行引擎，一起来看看虚拟机如何执行定义在Class文件里的字节码。

# 8. 第8章　虚拟机字节码执行引擎

代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。

## 8.1 概述

​	执行引擎是Java虚拟机核心的组成部分之一。“虚拟机”是一个相对于“物理机”的概念，这两种机器都有代码执行能力，其区别是物理机的执行引擎是直接建立在处理器、缓存、指令集和操作系统层面上的，而虚拟机的执行引擎则是由软件自行实现的，因此可以不受物理条件制约地定制指令集与执行引擎的结构体系，能够执行那些不被硬件直接支持的指令集格式。

​	在《Java虚拟机规范》中制定了Java虚拟机字节码执行引擎的概念模型，这个概念模型成为各大发行商的Java虚拟机执行引擎的统一外观（Facade）。在不同的虚拟机实现中，执行引擎在执行字节码的时候，通常会有**解释执行（通过解释器执行）**和**编译执行（通过即时编译器产生本地代码执行）**两种选择[^38]，也可能两者兼备，还可能会有同时包含几个不同级别的即时编译器一起工作的执行引擎。但从外观上来看，所有的Java虚拟机的执行引擎输入、输出都是一致的：输入的是字节码二进制流，处理过程是字节码解析执行的等效过程，输出的是执行结果，本章将主要从概念模型的角度来讲解虚拟机的方法调用和字节码执行。

[^38]: 有一些虚拟机（如Sun Classic VM）的内部只存在解释器，只能解释执行，另外一些虚拟机（如BEA JRockit）的内部只存在即时编译器，只能编译执行。

## 8.2 运行时栈帧结构

> [第8章 虚拟机字节码执行引擎](https://blog.csdn.net/limeOracle/article/details/114487353)

​	**Java虚拟机以方法作为最基本的执行单元，“栈帧”（Stack Frame）则是用于支持虚拟机进行方法调用和方法执行背后的数据结构，它也是虚拟机运行时数据区中的虚拟机栈（Virtual Machine Stack）[^39]的栈元素**。<u>栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息</u>，如果读者认真阅读过第6章，应该能从Class文件格式的方法表中找到以上大多数概念的静态对照物。每一个方法从调用开始至执行结束的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

​	<u>每一个栈帧都包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息</u>。

​	<u>在编译Java程序源码的时候，栈帧中需要多大的局部变量表，需要多深的操作数栈就已经被分析计算出来，并且写入到方法表的Code属性之中</u>[^40]。换言之，一个栈帧需要分配多少内存，并不会受到程序运行期变量数据的影响，而仅仅取决于程序源码和具体的虚拟机实现的栈内存布局形式。

​	一个线程中的方法调用链可能会很长，以Java程序的角度来看，同一时刻、同一条线程里面，在调用堆栈的所有方法都同时处于执行状态。而**对于执行引擎来讲，在活动线程中，只有位于栈顶的方法才是在运行的，只有位于栈顶的栈帧才是生效的，其被称为“当前栈帧”（Current Stack Frame），与这个栈帧所关联的方法被称为“当前方法”（Current Method）**。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作，在概念模型上，典型的栈帧结构如图8-1所示。

​	图8-1所示的就是虚拟机栈和栈帧的总体结构，接下来，我们将会详细了解栈帧中的局部变量表、操作数栈、动态连接、方法返回地址等各个部分的作用和数据结构。

![img](https://img-blog.csdnimg.cn/20210307192252740.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpbWVPcmFjbGU=,size_16,color_FFFFFF,t_70)

[^39]: 详细内容请参见2.2节的相关内容。
[^40]: 详细内容请参见6.3.7节的相关内容。

### 8.2.1 局部变量表

​	局部变量表（Local Variables Table）是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量。**在Java程序被编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需分配的局部变量表的最大容量**。

​	**局部变量表的容量以变量槽（Variable Slot）为最小单位**，《Java虚拟机规范》中并没有明确指出一个变量槽应占用的内存空间大小，只是很有导向性地说到每个变量槽都应该能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据，这8种数据类型，都可以使用32位或更小的物理内存来存储，但这种描述与明确指出“每个变量槽应占用32位长度的内存空间”是有本质差别的，**它允许变量槽的长度可以随着处理器、操作系统或虚拟机实现的不同而发生变化**，保证了即使在64位虚拟机中使用了64位的物理内存空间去实现一个变量槽，虚拟机仍要使用对齐和补白的手段让变量槽在外观上看起来与32位虚拟机中的一致。

​	<small>既然前面提到了Java虚拟机的数据类型，在此对它们再简单介绍一下。一个变量槽可以存放一个32位以内的数据类型，Java中占用不超过32位存储空间的数据类型有boolean、byte、char、short、int、float、reference[^41]和returnAddress这8种类型。前面6种不需要多加解释，读者可以按照Java语言中对应数据类型的概念去理解它们（仅是这样理解而已，Java语言和Java虚拟机中的基本数据类型是存在本质差别的），而第7种reference类型表示对一个对象实例的引用，《Java虚拟机规范》既没有说明它的长度，也没有明确指出这种引用应有怎样的结构。但是一般来说，虚拟机实现至少都应当能通过这个引用做到两件事情，一是从根据引用直接或间接地查找到对象在Java堆中的数据存放的起始地址或索引，二是根据引用直接或间接地查找到对象所属数据类型在方法区中的存储的类型信息，否则将无法实现《Java语言规范》中定义的语法约定[^42]。第8种returnAddress类型目前已经很少见了，它是为字节码指令jsr、jsr_w和ret服务的，指向了一条字节码指令的地址，某些很古老的Java虚拟机曾经使用这几条指令来实现异常处理时的跳转，但现在也已经全部改为采用异常表来代替了。</small>

​	**对于64位的数据类型，Java虚拟机会以高位对齐的方式为其分配两个连续的变量槽空间**。Java语言中明确的64位的数据类型只有long和double两种。这里把long和double数据类型分割存储的做法与“long和double的非原子性协定”中允许把一次long和double数据类型读写分割为两次32位读写的做法有些类似，读者阅读到本书关于Java内存模型的内容[^43]时可以进行对比。不过，**由于局部变量表是建立在线程堆栈中的，属于线程私有的数据，无论读写两个连续的变量槽是否为原子操作，都不会引起数据竞争和线程安全问题**。

​	**Java虚拟机通过索引定位的方式使用局部变量表，索引值的范围是从0开始至局部变量表最大的变量槽数量**。如果访问的是32位数据类型的变量，索引N就代表了使用第N个变量槽，如果访问的是64位数据类型的变量，则说明会同时使用第N和N+1两个变量槽。<u>对于两个相邻的共同存放一个64位数据的两个变量槽，虚拟机不允许采用任何方式单独访问其中的某一个，《Java虚拟机规范》中明确要求了如果遇到进行这种操作的字节码序列，虚拟机就应该在类加载的校验阶段中抛出异常</u>。

​	**当一个方法被调用时，Java虚拟机会使用局部变量表来完成参数值到参数变量列表的传递过程，即实参到形参的传递**。<u>如果执行的是实例方法（没有被static修饰的方法），那局部变量表中第0位索引的变量槽默认是用于传递方法所属对象实例的引用，在方法中可以通过关键字“this”来访问到这个隐含的参数</u>。其余参数则按照参数表顺序排列，占用从1开始的局部变量槽，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用域分配其余的变量槽。

​	**为了尽可能节省栈帧耗用的内存空间，局部变量表中的变量槽是可以重用的，方法体中定义的变量，其作用域并不一定会覆盖整个方法体，如果当前字节码PC计数器的值已经超出了某个变量的作用域，那这个变量对应的变量槽就可以交给其他变量来重用**。不过，这样的设计除了节省栈帧空间以外，还会伴随有少量额外的副作用，例如**在某些情况下变量槽的复用会直接影响到系统的垃圾收集行为**，请看代码清单8-1、代码清单8-2和代码清单8-3的3个演示。

​	代码清单8-1　局部变量表槽复用对垃圾收集的影响之一

```java
public static void main(String[] args)() {
  byte[] placeholder = new byte[64 * 1024 * 1024];
  System.gc();
}
```

​	代码清单8-1中的代码很简单，向内存填充了64MB的数据，然后通知虚拟机进行垃圾收集。我们在虚拟机运行参数中加上“-verbose：gc”来看看垃圾收集的过程，发现在System.gc()运行后并没有回收掉这64MB的内存，下面是运行的结果：

```shell
[GC 66846K->65824K(125632K), 0.0032678 secs]
[Full GC 65824K->65746K(125632K), 0.0064131 secs]
```

​	代码清单8-1的代码没有回收掉placeholder所占的内存是能说得过去，因为在执行System.gc()时，变量placeholder还处于作用域之内，虚拟机自然不敢回收掉placeholder的内存。那我们把代码修改一下，变成代码清单8-2的样子。

​	代码清单8-2　局部变量表Slot复用对垃圾收集的影响之二

```java
public static void main(String[] args)() {
  {
    byte[] placeholder = new byte[64 * 1024 * 1024];
  }
  System.gc();
}
```

​	加入了花括号之后，placeholder的作用域被限制在花括号以内，从代码逻辑上讲，在执行System.gc()的时候，placeholder已经不可能再被访问了，但执行这段程序，会发现运行结果如下，还是有64MB的内存没有被回收掉，这又是为什么呢？

```shell
[GC 66846K->65888K(125632K), 0.0009397 secs]
[Full GC 65888K->65746K(125632K), 0.0051574 secs]
```

​	在解释为什么之前，我们先对这段代码进行第二次修改，在调用System.gc()之前加入一行“int a=0；”，变成代码清单8-3的样子。

​	代码清单8-3　局部变量表Slot复用对垃圾收集的影响之三

```java
public static void main(String[] args)() {
  {
    byte[] placeholder = new byte[64 * 1024 * 1024];
  }
  int a = 0;
  System.gc();
}
```

​	这个修改看起来很莫名其妙，但运行一下程序，却发现这次内存真的被正确回收了：

```shell
[GC 66401K->65778K(125632K), 0.0035471 secs]
[Full GC 65778K->218K(125632K), 0.0140596 secs]
```

​	代码清单8-1至8-3中，**placeholder能否被回收的根本原因就是：局部变量表中的变量槽是否还存有关于placeholder数组对象的引用**。<u>第一次修改中，代码虽然已经离开了placeholder的作用域，但在此之后，再没有发生过任何对局部变量表的读写操作，placeholder原本所占用的变量槽还没有被其他变量所复用，所以**作为GC Roots一部分的局部变量表仍然保持着对它的关联**</u>。这种关联没有被及时打断，绝大部分情况下影响都很轻微。但如果遇到一个方法，其后面的代码有一些耗时很长的操作，而前面又定义了占用了大量内存但实际上已经不会再使用的变量，手动将其设置为null值（用来代替那句int a=0，把变量对应的局部变量槽清空）便不见得是一个绝对无意义的操作，**这种操作可以作为一种在极特殊情形（对象占用内存大、此方法的栈帧长时间不能被回收、方法调用次数达不到即时编译器的编译条件）下的“奇技”来使用**。Java语言的一本非常著名的书籍《Practical Java》中将把“不使用的对象应手动赋值为null”作为一条推荐的编码规则（笔者并不认同这条规则），但是并没有解释具体原因，很长时间里都有读者对这条规则感到疑惑。

​	虽然代码清单8-1至8-3的示例说明了赋null操作在某些极端情况下确实是有用的，但笔者的观点是不应当对赋null值操作有什么特别的依赖，更没有必要把它当作一个普遍的编码规则来推广。原因有两点，<u>从编码角度讲，以恰当的变量作用域来控制变量回收时间才是最优雅的解决方法</u>，如代码清单8-3那样的场景除了做实验外几乎毫无用处。更关键的是，从执行角度来讲，使用赋null操作来优化内存回收是建立在对字节码执行引擎概念模型的理解之上的，在第6章介绍完字节码之后，笔者在末尾还撰写了一个小结“公有设计、私有实现”（6.5节）来强调概念模型与实际执行过程是外部看起来等效，内部看上去则可以完全不同。当虚拟机使用解释器执行时，通常与概念模型还会比较接近，但经过即时编译器施加了各种编译优化措施以后，两者的差异就会非常大，只保证程序执行的结果与概念一致。<u>在实际情况中，即时编译才是虚拟机执行代码的主要方式，赋null值的操作在经过即时编译优化后几乎是一定会被当作无效操作消除掉的，这时候将变量设置为null就是毫无意义的行为</u>。字节码被即时编译为本地代码后，对GC Roots的枚举也与解释执行时期有显著差别，以前面的例子来看，经过第一次修改的代码清单8-2在经过即时编译后，System.gc()执行时就可以正确地回收内存，根本无须写成代码清单8-3的样子。

​	**关于局部变量表，还有一点可能会对实际开发产生影响，就是局部变量不像前面介绍的类变量那样存在“准备阶段”**。

​	**通过第7章的学习，我们已经知道类的字段变量有两次赋初始值的过程，一次在准备阶段，赋予系统初始值；另外一次在初始化阶段，赋予程序员定义的初始值**。因此即使在初始化阶段程序员没有为类变量赋值也没有关系，类变量仍然具有一个确定的初始值，不会产生歧义。

​	**但局部变量就不一样了，如果一个局部变量定义了但没有赋初始值，那它是完全不能使用的**。所以不要认为Java中任何情况下都存在诸如整型变量默认为0、布尔型变量默认为false等这样的默认值规则。如代码清单8-4所示，这段代码在Java中其实并不能运行（但是在其他语言，譬如C和C++中类似的代码是可以运行的），所幸编译器能在编译期间就检查到并提示出这一点，即便编译能通过或者手动生成字节码的方式制造出下面代码的效果，字节码校验的时候也会被虚拟机发现而导致类加载失败。

​	代码清单8-4　未赋值的局部变量

```java
public static void main(String[] args) {
  int a;
  System.out.println(a);
}
```

[^41]:Java虚拟机规范中没有明确规定reference类型的长度，它的长度与实际使用32位还是64位虚拟机有关，如果是64位虚拟机，还与是否开启某些对象指针压缩的优化有关，这里我们暂且只取32位虚拟机的reference长度。
[^42]:并不是所有语言的对象引用都能满足这两点，例如C++语言，默认情况下（不开启RTTI支持的情况），就只能满足第一点，而不满足第二点。这也是为何C++中无法提供Java语言里很常见的反射的根本原因。
[^43]:这是Java内存模型中定义的内容，关于原子操作与“long和double的非原子性协定”等问题，将在本书第12章中做详细讲解。

### 8.2.2 操作数栈

> [《深入理解java虚拟机》 第八章 虚拟机字节码执行引擎](https://blog.csdn.net/lik_lik/article/details/89322483)	<=	图片来源

​	**操作数栈（Operand Stack）也常被称为操作栈，它是一个后入先出（Last In First Out，LIFO）栈**。同局部变量表一样，操作数栈的最大深度也在编译的时候被写入到Code属性的max_stacks数据项之中。操作数栈的每一个元素都可以是包括long和double在内的任意Java数据类型。**32位数据类型所占的栈容量为1，64位数据类型所占的栈容量为2**。<u>Javac编译器的数据流分析工作保证了在方法执行的任何时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值。</u>

​	当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈和入栈操作。譬如在做算术运算的时候是通过将运算涉及的操作数栈压入栈顶后调用运算指令来进行的，又譬如在调用其他方法的时候是通过操作数栈来进行方法参数的传递。举个例子，例如整数加法的字节码指令iadd，这条指令在运行的时候要求操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值，当执行这个指令时，会把这两个int值出栈并相加，然后将相加的结果重新入栈。

​	<u>操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器必须要严格保证这一点，在类校验阶段的数据流分析中还要再次验证这一点</u>。再以上面的iadd指令为例，这个指令只能用于整型数的加法，它在执行时，最接近栈顶的两个元素的数据类型必须为int型，不能出现一个long和一个float使用iadd命令相加的情况。

​	**另外在概念模型中，两个不同栈帧作为不同方法的虚拟机栈的元素，是完全相互独立的**。**<u>但是在大多虚拟机的实现里都会进行一些优化处理，令两个栈帧出现一部分重叠</u>**。让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起，这样做不仅节约了一些空间，更重要的是在进行方法调用时就可以直接共用一部分数据，无须进行额外的参数复制传递了，重叠的过程如图8-2所示。

​	**Java虚拟机的解释执行引擎被称为“基于栈的执行引擎”，里面的“栈”就是操作数栈**。后文会对基于栈的代码执行过程进行更详细的讲解，介绍它与更常见的基于寄存器的执行引擎有哪些差别。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190415222347729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpa19saWs=,size_16,color_FFFFFF,t_70)

### 8.2.3 动态连接

​	**每个栈帧都包含一个指向运行时常量池[^44]中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接（Dynamic Linking）**。

​	<u>通过第6章的讲解，我们知道Class文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数。</u>

+ <u>这些符号引用一部分会在**类加载阶段或者第一次使用**的时候就被转化为直接引用，这种转化被称为**静态解析**</u>。
+ <u>另外一部分将在**每一次运行期间**都转化为直接引用，这部分就称为**动态连接**</u>。

​	关于这两个转化过程的具体过程，将在8.3节中再详细讲解。

[^44]:运行时常量池的相关内容详见第2章。

### 8.2.4 方法返回地址

​	当一个方法开始执行后，只有两种方式退出这个方法。

+ 第一种方式是执行引擎**遇到任意一个方法返回的字节码指令**，这时候可能会有返回值传递给上层的方法调用者（调用当前方法的方法称为调用者或者主调方法），方法是否有返回值以及返回值的类型将根据遇到何种方法返回指令来决定，这种退出方法的方式称为**“正常调用完成”（Normal Method Invocation Completion）**。

+ 另外一种退出方式是在方法执行的过程中**遇到了异常，并且这个异常没有在方法体内得到妥善处理**。无论是Java虚拟机内部产生的异常，还是代码中使用athrow字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种退出方法的方式称为**“异常调用完成（Abrupt Method Invocation Completion）”**。一个方法使用异常完成出口的方式退出，是不会给它的上层调用者提供任何返回值的。

​	<u>无论采用何种退出方式，在方法退出之后，都必须返回到最初方法被调用时的位置，程序才能继续执行，方法返回时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层主调方法的执行状态</u>。

+ 一般来说，**方法正常退出**时，**主调方法的PC计数器的值**就可以作为返回地址，**栈帧中很可能会保存这个计数器值**。
+ 而**方法异常退出**时，返回地址是要通过**异常处理器表**来确定的，**栈帧中就一般不会保存这部分信息**。

​	**方法退出的过程实际上等同于把当前栈帧出栈**（学过汇编就很好理解了），因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值（如果有的话）压入调用者栈帧的操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令等。笔者这里写的“可能”是由于这是基于概念模型的讨论，只有具体到某一款Java虚拟机实现，会执行哪些操作才能确定下来。

### 8.2.5 附加信息

​	《Java虚拟机规范》允许虚拟机实现增加一些规范里没有描述的信息到栈帧之中，例如与调试、性能收集相关的信息，这部分信息完全取决于具体的虚拟机实现，这里不再详述。

​	**在讨论概念时，一般会把动态连接、方法返回地址与其他附加信息全部归为一类，称为栈帧信息**。

## 8.3 方法调用

​	<u>方法调用并不等同于方法中的代码被执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还未涉及方法内部的具体运行过程</u>。

​	在程序运行时，进行方法调用是最普遍、最频繁的操作之一，但第7章中已经讲过，Class文件的编译过程中不包含传统程序语言编译的连接步骤，**一切方法调用在Class文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址（也就是之前说的直接引用）**。这个特性给Java带来了更强大的动态扩展能力，但也使得Java方法调用过程变得相对复杂，某些调用需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。

### 8.3.1 解析

​	<u>承接前面关于方法调用的话题，所有方法调用的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能够成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且**这个方法的调用版本在运行期是不可改变的**。换句话说，**调用目标在程序代码写好、编译器进行编译那一刻就已经确定下来**</u>。**这类方法的调用被称为解析（Resolution）**。

​	在Java语言中符合“编译期可知，运行期不可变”这个要求的方法，主要有静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，**这两种方法各自的特点决定了它们都不可能通过继承或别的方式重写出其他版本，因此它们都适合在类加载阶段进行解析**。

​	调用不同类型的方法，字节码指令集里设计了不同的指令。在Java虚拟机支持以下5条方法调用字节码指令，分别是：

+ invokestatic。用于调用静态方法。

+ invokespecial。用于调用实例构造器\<init\>\(\)方法、私有方法和父类中的方法。

+ invokevirtual。用于调用所有的虚方法。

+ invokeinterface。用于调用接口方法，会在运行时再确定一个实现该接口的对象。

+ invokedynamic。先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

​	<u>前面4条调用指令，分派逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户设定的引导方法来决定的</u>。

​	只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版本，Java语言里符合这个条件的方法共有**静态方法、私有方法、实例构造器、父类方法**4种，再加上**被final修饰的方法**（尽管它使用invokevirtual指令调用），**这5种方法调用会<u>在类加载的时候</u>就可以把符号引用解析为该方法的直接引用**。<u>这些方法统称为“非虚方法”（Non-Virtual Method），与之相反，其他方法就被称为“虚方法”（Virtual Method）</u>。

​	代码清单8-5演示了一种常见的解析调用的例子，该样例中，静态方法sayHello()只可能属于类型StaticResolution，没有任何途径可以覆盖或隐藏这个方法。

​	代码清单8-5　方法静态解析演示

```java
/**
* 方法静态解析演示
*
* @author zzm
*/
public class StaticResolution {
  public static void sayHello() {
    System.out.println("hello world");
  }
  public static void main(String[] args) {
    StaticResolution.sayHello();
  }
}
```

​	使用javap命令查看这段程序对应的字节码，会发现的确是通过invokestatic命令来调用sayHello()方法，而且其调用的方法版本已经在编译时就明确以常量池项的形式固化在字节码指令的参数之中（代码里的31号常量池项）：

```shell
javap -verbose StaticResolution
public static void main(java.lang.String[]);
Code:
Stack=0, Locals=1, Args_size=1
0: invokestatic #31; //Method sayHello:()V
3: return
LineNumberTable:
line 15: 0
line 16: 3
```

​	Java中的非虚方法除了使用invokestatic、invokespecial调用的方法之外还有一种，就是被final修饰的实例方法。虽然由于历史设计的原因，final方法是使用invokevirtual指令来调用的，但是因为它也无法被覆盖，没有其他版本的可能，所以也无须对方法接收者进行多态选择，又或者说多态选择的结果肯定是唯一的。**在《Java语言规范》中明确定义了被final修饰的方法是一种非虚方法。**

​	**解析调用一定是个静态的过程，在编译期间就完全确定，在类加载的解析阶段就会把涉及的符号引用全部转变为明确的直接引用，不必延迟到运行期再去完成**。

​	而另一种主要的方法调用形式：分派（Dispatch）调用则要复杂许多，它可能是静态的也可能是动态的，按照分派依据的宗量数可分为单分派和多分派[^45]。这两类分派方式两两组合就构成了静态单分派、静态多分派、动态单分派、动态多分派4种分派组合情况，下面我们来看看虚拟机中的方法分派是如何进行的。

[^45]:这里涉及的单分派、多分派及相关概念（如“宗量数”）在后续章节有详细解释，如没有这方面基础的读者暂时略过即可。

### 8.3.2 分派

​	众所周知，Java是一门面向对象的程序语言，因为**Java具备面向对象的3个基本特征：继承、封装和多态**。本节讲解的分派调用过程将会揭示多态性特征的一些最基本的体现，如“重载”和“重写”在Java虚拟机之中是如何实现的，这里的实现当然不是语法上该如何写，我们关心的依然是虚拟机如何确定正确的目标方法。

#### 1. 静态分派

​	在开始讲解静态分派[^46]前，笔者先声明一点，“分派”（Dispatch）这个词本身就具有动态性，一般不应用在静态语境之中，<u>这部分原本在英文原版的《Java虚拟机规范》和《Java语言规范》里的说法都是“Method Overload Resolution”</u>，即应该归入8.2节的“解析”里去讲解，<u>但部分其他外文资料和国内翻译的许多中文资料都将这种行为称为“静态分派”</u>，所以笔者在此特别说明一下，以免读者阅读英文资料时遇到这两种说法产生疑惑。

​	为了解释**静态分派**和**重载（Overload）**，笔者准备了一段经常出现在面试题中的程序代码，读者不妨先看一遍，想一下程序的输出结果是什么。后面的话题将围绕这个类的方法来编写重载代码，以分析虚拟机和编译器确定方法版本的过程。程序如代码清单8-6所示。

​	代码清单8-6　方法静态分派演示

```java
package org.fenixsoft.polymorphic;
/**
* 方法静态分派演示
* @author zzm
*/
public class StaticDispatch {
  static abstract class Human {
  }
  static class Man extends Human {
  }
  static class Woman extends Human {
  }
  public void sayHello(Human guy) {
    System.out.println("hello,guy!");
  }
  public void sayHello(Man guy) {
    System.out.println("hello,gentleman!");
  }
  public void sayHello(Woman guy) {
    System.out.println("hello,lady!");
  }
  public static void main(String[] args) {
    Human man = new Man();
    Human woman = new Woman();
    StaticDispatch sr = new StaticDispatch();
    sr.sayHello(man);
    sr.sayHello(woman);
  }
}
```

​	运行结果：

```shell
hello,guy!
hello,guy!
```

​	代码清单8-6中的代码实际上是在考验阅读者对重载的理解程度，相信对Java稍有经验的程序员看完程序后都能得出正确的运行结果，但为什么虚拟机会选择执行参数类型为Human的重载版本呢？在解决这个问题之前，我们先通过如下代码来定义两个关键概念：

```java
Human man = new Man();
```

​	<u>我们把上面代码中的“Human”称为变量的**“静态类型”（Static Type）**，或者叫**“外观类型”（Apparent Type）**，后面的“Man”则被称为变量的**“实际类型”（Actual Type）**或者叫**“运行时类型”（Runtime Type）**</u>。静态类型和实际类型在程序中都可能会发生变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的**静态类型是在编译期可知的**；而**实际类型变化的结果在运行期才可确定**，编译器在编译程序的时候并不知道一个对象的实际类型是什么。笔者猜想上面这段话读者大概会不太好理解，那不妨通过一段实际例子来解释，譬如有下面的代码：

```java
// 实际类型变化
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();
// 静态类型变化
sr.sayHello((Man) human)
sr.sayHello((Woman) human)
```

​	对象human的实际类型是可变的，编译期间它完全是个“薛定谔的人”，到底是Man还是Woman，必须等到程序运行到这行的时候才能确定。**而human的静态类型是Human，也可以在使用时（如sayHello()方法中的强制转型）临时改变这个类型，但这个改变仍是在编译期是可知的，两次sayHello()方法的调用，在编译期完全可以明确转型的是Man还是Woman**。

​	解释清楚了静态类型与实际类型的概念，我们就把话题再转回到代码清单8-6的样例代码中。main()里面的两次sayHello()方法调用，在方法接收者已经确定是对象“sr”的前提下，使用哪个重载版本，就完全取决于传入参数的数量和数据类型。<u>代码中故意定义了两个静态类型相同，而实际类型不同的变量，但虚拟机（或者准确地说是编译器）在重载时是通过参数的静态类型而不是实际类型作为判定依据的</u>。

​	**由于静态类型在编译期可知，所以在编译阶段，Javac编译器就根据参数的静态类型决定了会使用哪个重载版本，因此选择了sayHello(Human)作为调用目标，并把这个方法的符号引用写到main()方法里的两条invokevirtual指令的参数中**。

​	**所有依赖静态类型来决定方法执行版本的分派动作，都称为静态分派**。静态分派的最典型应用表现就是方法重载。<u>静态分派发生在编译阶段，因此确定静态分派的动作实际上不是由虚拟机来执行的，这点也是为何一些资料选择把它归入“解析”而不是“分派”的原因</u>。

​	需要注意Javac编译器虽然能确定出方法的重载版本，但在很多情况下这个重载版本并不是“唯一”的，往往只能确定一个“相对更合适的”版本。这种模糊的结论在由0和1构成的计算机世界中算是个比较稀罕的事件，产生这种模糊结论的主要原因是<u>字面量天生的模糊性，它不需要定义，所以字面量就没有显式的静态类型，它的静态类型只能通过语言、语法的规则去理解和推断</u>。代码清单8-7演示了何谓“更加合适的”版本。

​	代码清单8-7　重载方法匹配优先级

```java
package org.fenixsoft.polymorphic;
public class Overload {
  public static void sayHello(Object arg) {
    System.out.println("hello Object");
  }
  public static void sayHello(int arg) {
    System.out.println("hello int");
  }
  public static void sayHello(long arg) {
    System.out.println("hello long");
  }
  public static void sayHello(Character arg) {
    System.out.println("hello Character");
  }
  public static void sayHello(char arg) {
    System.out.println("hello char");
  }
  public static void sayHello(char... arg) {
    System.out.println("hello char ...");
  }
  public static void sayHello(Serializable arg) {
    System.out.println("hello Serializable");
  }
  public static void main(String[] args) {
    sayHello('a');
  }
}
```

​	上面的代码运行后会输出：

```shell
hello char
```

​	这很好理解，'a'是一个char类型的数据，自然会寻找参数类型为char的重载方法，如果注释掉sayHello(char arg)方法，那输出会变为：

```shell
hello int
```

​	这时发生了一次自动类型转换，'a'除了可以代表一个字符串，还可以代表数字97（字符'a'的Unicode数值为十进制数字97），因此参数类型为int的重载也是合适的。我们继续注释掉sayHello(int arg)方法，那输出会变为：

```shell
hello long
```

​	这时发生了两次自动类型转换，'a'转型为整数97之后，进一步转型为长整数97L，匹配了参数类型为long的重载。笔者在代码中没有写其他的类型如float、double等的重载，不过实际上自动转型还能继续发生多次，按照char>int>long>float>double的顺序转型进行匹配，但不会匹配到byte和short类型的重载，因为**char到byte或short的转型是不安全的**。我们继续注释掉sayHello(long arg)方法，那输出会变为：

```shell
hello Character
```

​	这时发生了一次自动装箱，'a'被包装为它的封装类型java.lang.Character，所以匹配到了参数类型为Character的重载，继续注释掉sayHello(Character arg)方法，那输出会变为：

```shell
hello Serializable
```

​	这个输出可能会让人摸不着头脑，一个字符或数字与序列化有什么关系？出现hello Serializable，是因为java.lang.Serializable是java.lang.Character类实现的一个接口，当自动装箱之后发现还是找不到装箱类，但是找到了装箱类所实现的接口类型，所以紧接着又发生一次自动转型。char可以转型成int，但是Character是绝对不会转型为Integer的，它只能安全地转型为它实现的接口或父类。Character还实现了另外一个接口java.lang.Comparable\<Character\>，<u>如果同时出现两个参数分别为Serializable和Comparable\<Character\>的重载方法，那它们在此时的优先级是一样的。编译器无法确定要自动转型为哪种类型，会提示“类型模糊”（Type Ambiguous），并拒绝编译</u>。程序必须在调用时显式地指定字面量的静态类型，如：sayHello((Comparable\<Character\>)'a')，才能编译通过。但是如果读者愿意花费一点时间，绕过Javac编译器，自己去构造出表达相同语义的字节码，将会发现这是能够通过Java虚拟机的类加载校验，而且能够被Java虚拟机正常执行的，但是会选择Serializable还是Comparable\<Character\>的重载方法则并不能事先确定，这是《Java虚拟机规范》所允许的，在第7章介绍接口方法解析过程时曾经提到过。

​	下面继续注释掉sayHello(Serializable arg)方法，输出会变为：

```shell
hello Object
```

​	这时是char装箱后转型为父类了，如果有多个父类，那将在继承关系中从下往上开始搜索，越接上层的优先级越低。即使方法调用传入的参数值为null时，这个规则仍然适用。我们把sayHello(Object arg)也注释掉，输出将会变为：

```shell
hello char ...
```

​	7个重载方法已经被注释得只剩1个了，可见**变长参数的重载优先级是最低的**，这时候字符'a'被当作了一个char[]数组的元素。笔者使用的是char类型的变长参数，读者在验证时还可以选择int类型、Character类型、Object类型等的变长参数重载来把上面的过程重新折腾一遍。但是要注意的是，**有一些在单个参数中能成立的自动转型，如char转型为int，在变长参数中是不成立的**[^47]。

​	代码清单8-7演示了编译期间选择静态分派目标的过程，这个过程也是Java语言实现方法重载的本质。演示所用的这段程序无疑是属于很极端的例子，除了用作面试题为难求职者之外，在实际工作中几乎不可能存在任何有价值的用途，笔者拿来做演示仅仅是用于讲解重载时目标方法选择的过程，对绝大多数下进行这样极端的重载都可算作真正的“关于茴香豆的茴有几种写法的研究”。

​	**无论对重载的认识有多么深刻，一个合格的程序员都不应该在实际应用中写这种晦涩的重载代码。**

​	另外还有一点读者可能比较容易混淆：笔者讲述的解析与分派这两者之间的关系并不是二选一的排他关系，它们是在不同层次上去筛选、确定目标方法的过程。<u>例如前面说过静态方法会在编译期确定、在类加载期就进行解析，而静态方法显然也是可以拥有重载版本的，选择重载版本的过程也是通过静态分派完成的</u>。

#### 2. 动态分派

​	了解了静态分派，我们接下来看一下Java语言里动态分派的实现过程，它与Java语言多态性的另外一个重要体现[^48]——**重写（Override）**有着很密切的关联。我们还是用前面的Man和Woman一起sayHello的例子来讲解动态分派，请看代码清单8-8中所示的代码。

​	代码清单8-8　方法动态分派演示

```java
package org.fenixsoft.polymorphic;
/**
* 方法动态分派演示
* @author zzm
*/
public class DynamicDispatch {
  static abstract class Human {
    protected abstract void sayHello();
  }
  static class Man extends Human {
    @Override
    protected void sayHello() {
      System.out.println("man say hello");
    }
  }
  static class Woman extends Human {
    @Override
    protected void sayHello() {
      System.out.println("woman say hello");
    }
  }
  public static void main(String[] args) {
    Human man = new Man();
    Human woman = new Woman();
    man.sayHello();
    woman.sayHello();
    man = new Woman();
    man.sayHello();
  }
}
```

​	运行结果：

```shell
man say hello
woman say hello
woman say hello
```

​	这个运行结果相信不会出乎任何人的意料，对于习惯了面向对象思维的Java程序员们会觉得这是完全理所当然的结论。我们现在的问题还是和前面的一样，Java虚拟机是如何判断应该调用哪个方法的？

​	显然这里选择调用的方法版本是不可能再根据静态类型来决定的，因为静态类型同样都是Human的两个变量man和woman在调用sayHello()方法时产生了不同的行为，甚至变量man在两次调用中还执行了两个不同的方法。导致这个现象的原因很明显，是因为这两个变量的实际类型不同，Java虚拟机是如何根据实际类型来分派方法执行版本的呢？我们使用javap命令输出这段代码的字节码，尝试从中寻找答案，输出结果如代码清单8-9所示。

​	代码清单8-9　main()方法的字节码

```shell
public static void main(java.lang.String[]);
Code:
Stack=2, Locals=3, Args_size=1
0: new #16; //class org/fenixsoft/polymorphic/DynamicDispatch$Man
3: dup
4: invokespecial #18; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Man."<init>":()V
7: astore_1
8: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
11: dup
12: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
15: astore_2
16: aload_1
17: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
20: aload_2
21: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
24: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
27: dup
28: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
31: astore_1
32: aload_1
33: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
36: return
```

​	0～15行的字节码是准备动作，作用是建立man和woman的内存空间、调用Man和Woman类型的实例构造器，将这两个实例的引用存放在第1、2个局部变量表的变量槽中，这些动作实际对应了Java源码中的这两行：

```shell
Human man = new Man();
Human woman = new Woman();
```

​	接下来的16～21行是关键部分，16和20行的aload指令分别把刚刚创建的两个对象的引用压到栈顶，这两个对象是将要执行的sayHello()方法的所有者，称为接收者（Receiver）；17和21行是方法调用指令，这两条调用指令单从字节码角度来看，无论是指令（都是invokevirtual）还是参数（都是常量池中第22项的常量，注释显示了这个常量是Human.sayHello()的符号引用）都完全一样，但是这两句指令最终执行的目标方法并不相同。那看来解决问题的关键还必须从invokevirtual指令本身入手，要弄清楚它是如何确定调用方法版本、如何实现多态查找来着手分析才行。根据《Java虚拟机规范》，invokevirtual指令的运行时解析过程[^49]大致分为以下几步：

1）找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C。

2）如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果

通过则返回这个方法的直接引用，查找过程结束；不通过则返回java.lang.IllegalAccessError异常。

3）否则，按照继承关系从下往上依次对C的各个父类进行第二步的搜索和验证过程。

4）如果始终没有找到合适的方法，则抛出java.lang.AbstractMethodError异常。

​	**正是因为invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型，所以两次调用中的invokevirtual指令并不是把常量池中方法的符号引用解析到直接引用上就结束了，还会根据方法接收者的实际类型来选择方法版本，这个过程就是Java语言中方法重写的本质**。<u>我们把这种在运行期根据实际类型确定方法执行版本的分派过程称为动态分派</u>。

​	既然这种多态性的根源在于虚方法调用指令invokevirtual的执行逻辑，那自然我们得出的结论就只会对方法有效，对字段是无效的，因为字段不使用这条指令。**事实上，在Java里面只有虚方法存在，字段永远不可能是虚的，换句话说，字段永远不参与多态，哪个类的方法访问某个名字的字段时，该名字指的就是这个类能看到的那个字段**。

​	**<u>当子类声明了与父类同名的字段时，虽然在子类的内存中两个字段都会存在，但是子类的字段会遮蔽父类的同名字段</u>**。

​	为了加深理解，笔者又编撰了一份“劣质面试题式”的代码片段，请阅读代码清单8-10，思考运行后会输出什么结果。

​	代码清单8-10　字段没有多态性

```java
package org.fenixsoft.polymorphic;
/**
* 字段不参与多态
* @author zzm
*/
public class FieldHasNoPolymorphic {
  static class Father {
    public int money = 1;
    public Father() {
      money = 2;
      showMeTheMoney();
    }
    public void showMeTheMoney() {
      System.out.println("I am Father, i have $" + money);
    }
  }
  static class Son extends Father {
    public int money = 3;
    public Son() {
      money = 4;
      showMeTheMoney();
    }
    public void showMeTheMoney() {
      System.out.println("I am Son, i have $" + money);
    }
  }
  public static void main(String[] args) {
    Father gay = new Son();
    System.out.println("This gay has $" + gay.money);
  }
}
```

​	运行后输出结果为：

```shell
I am Son, i have $0
I am Son, i have $4
This gay has $2
```

​	**输出两句都是“I am Son”，这是因为Son类在创建的时候，首先隐式调用了Father的构造函数，而Father构造函数中对showMeTheMoney()的调用是一次虚方法调用，实际执行的版本是Son::showMeTheMoney()方法，所以输出的是“I am Son”，这点经过前面的分析相信读者是没有疑问的了**。

​	**而这时候虽然父类的money字段已经被初始化成2了，但Son::showMeTheMoney()方法中访问的却是子类的money字段，这时候结果自然还是0，因为它要到子类的构造函数执行时才会被初始化**。

​	**main()的最后一句通过<u>静态类型访问</u>到了父类中的money，输出了2**。

#### 3. 单分派与多分派

​	**方法的接收者与方法的参数统称为方法的宗量，这个定义最早应该来源于著名的《Java与模式》一书**。

+ 根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。

+ **单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择**。

​	单分派和多分派的定义读起来拗口，从字面上看也比较抽象，不过对照着实例看并不难理解其含义，代码清单8-11中举了一个Father和Son一起来做出“一个艰难的决定[^50]”的例子。

​	代码清单8-11　单分派和多分派

```java
/**
* 单分派、多分派演示
* @author zzm
*/
public class Dispatch {
  static class QQ {}
  static class _360 {}
  public static class Father {
    public void hardChoice(QQ arg) {
      System.out.println("father choose qq");
    }
    public void hardChoice(_360 arg) {
      System.out.println("father choose 360");
    }
  }
  public static class Son extends Father {
    public void hardChoice(QQ arg) {
      System.out.println("son choose qq");
    }
    public void hardChoice(_360 arg) {
      System.out.println("son choose 360");
    }
  }
  public static void main(String[] args) {
    Father father = new Father();
    Father son = new Son();
    father.hardChoice(new _360());
    son.hardChoice(new QQ());
  }
}
```

​	运行结果：

```shell
father choose 360
son choose qq
```

​	在main()里调用了两次hardChoice()方法，这两次hardChoice()方法的选择结果在程序输出中已经显示得很清楚了。<u>我们关注的首先是编译阶段中编译器的选择过程，也就是静态分派的过程。这时候选择目标方法的依据有两点：一是**静态类型**是Father还是Son，二是**方法参数**是QQ还是360</u>。这次选择结果的最终产物是产生了两条invokevirtual指令，两条指令的参数分别为**常量池**中指向Father::hardChoice(360)及Father::hardChoice(QQ)**方法的符号引用**。

​	**因为是根据两个宗量进行选择，所以<u>Java语言的静态分派属于多分派类型</u>**。

​	再看看运行阶段中虚拟机的选择，也就是动态分派的过程。在执行“son.hardChoice(new QQ())”这行代码时，更准确地说，是在执行这行代码所对应的invokevirtual指令时，由于**编译期已经决定目标方法的签名必须为hardChoice(QQ)**，虚拟机此时不会关心传递过来的参数“QQ”到底是“腾讯QQ”还是“奇瑞QQ”，因为这时候参数的静态类型、实际类型都对方法的选择不会构成任何影响，唯一可以影响虚拟机选择的因素只有该方法的接受者的实际类型是Father还是Son。

​	**因为只有一个宗量作为选择依据，所以<u>Java语言的动态分派属于单分派类型</u>**。

​	根据上述论证的结果，我们可以总结一句：**如今（直至本书编写的Java 12和预览版的Java 13）的Java语言是一门静态多分派、动态单分派的语言**。强调“如今的Java语言”是因为这个结论未必会恒久不变，C#在3.0及之前的版本与Java一样是动态单分派语言，但在C#4.0中引入了dynamic类型后，就可以很方便地实现动态多分派。JDK 10时Java语法中新出现var关键字，但请读者切勿将其与C#中的dynamic类型混淆，事实上Java的var与C#的var才是相对应的特性，它们与dynamic有着本质的区别：**var是在编译时根据声明语句中赋值符右侧的表达式类型来静态地推断类型，这本质是一种语法糖**；<u>而dynamic在编译时完全不关心类型是什么，等到运行的时候再进行类型判断</u>。Java语言中与C#的dynamic类型功能相对接近（只是接近，并不是对等的）的应该是在JDK 9时通过JEP 276引入的jdk.dynalink模块[^51]，**使用jdk.dynalink可以实现在表达式中使用动态类型，Javac编译器会将这些动态类型的操作翻译为invokedynamic指令的调用点**。

​	**按照目前Java语言的发展趋势，它并没有直接变为动态语言的迹象，而是通过内置动态语言（如JavaScript）执行引擎、加强与其他Java虚拟机上动态语言交互能力的方式来间接地满足动态性的需求**。

​	<u>但是作为多种语言共同执行平台的Java虚拟机层面上则不是如此，早在JDK 7中实现的JSR-292[^52]里面就已经开始提供对动态语言的方法调用支持了，JDK 7中新增的invokedynamic指令也成为最复杂的一条方法调用的字节码指令，稍后笔者将在本章中专门开一节来讲解这个与Java调用动态语言密切相关的特性</u>。

#### 4. 虚拟机动态分派的实现

> [多态方法调用的解析和分派](https://www.cnblogs.com/wade-luffy/p/6058075.html)	<=	下方图片来源

​	前面介绍的分派过程，作为对Java虚拟机概念模型的解释基本上已经足够了，它已经解决了虚拟机在分派中“会做什么”这个问题。但如果问Java虚拟机“具体如何做到”的，答案则可能因各种虚拟机的实现不同会有些差别。

​	动态分派是执行非常频繁的动作，而且动态分派的方法版本选择过程需要运行时在接收者类型的方法元数据中搜索合适的目标方法，因此，<u>Java虚拟机实现基于执行性能的考虑，真正运行时一般不会如此频繁地去反复搜索类型元数据</u>。面对这种情况，**一种基础而且常见的优化手段是为类型在方法区中建立一个虚方法表（Virtual Method Table，也称为vtable，与此对应的，在invokeinterface执行时也会用到接口方法表——Interface Method Table，简称itable），使用虚方法表索引来代替元数据查找以提高性能**[^53]。我们先看看代码清单8-11所对应的虚方法表结构示例，如图8-3所示。

![img](https://images2015.cnblogs.com/blog/990532/201611/990532-20161113074119030-528815574.jpg)

​	虚方法表中存放着各个方法的实际入口地址。

+ **如果某个方法在子类中没有被重写，那子类的虚方法表中的地址入口和父类相同方法的地址入口是一致的，都指向父类的实现入口。**
+ **如果子类中重写了这个方法，子类虚方法表中的地址也会被替换为指向子类实现版本的入口地址**。

​	在图8-3中，Son重写了来自Father的全部方法，因此Son的方法表没有指向Father类型数据的箭头。但是Son和Father都没有重写来自Object的方法，所以它们的方法表中所有从Object继承来的方法都指向了Object的数据类型。

​	**为了程序实现方便，具有相同签名的方法，在父类、子类的虚方法表中都应当具有一样的索引序号**，<u>这样当类型变换时，仅需要变更查找的虚方法表，就可以从不同的虚方法表中按索引转换出所需的入口地址</u>。**虚方法表一般在类加载的连接阶段进行初始化，准备了类的变量初始值后，虚拟机会把该类的虚方法表也一同初始化完毕**。

​	<u>上文中笔者提到了查虚方法表是分派调用的一种优化手段，由于Java对象里面的方法默认（即不使用final修饰）就是虚方法，虚拟机除了使用虚方法表之外，为了进一步提高性能，还会使用类型继承关系分析（Class Hierarchy Analysis，CHA）、守护内联（Guarded Inlining）、内联缓存（InlineCache）等多种非稳定的激进优化来争取更大的性能空间，关于这几种优化技术的原理和运作过程，读者可以参考第11章中的相关内容</u>。

[^46]:维基百科中关于静态分派的解释：https://en.wikipedia.org/wiki/Static_dispatch。
[^47]:重载中选择最合适方法的过程，可参见《Java语言规范》15.12.2节的相关内容。
[^48]:重写肯定是多态性的体现，但对于重载算不算多态，有一些概念上的争议，有观点认为必须是多个不同类对象对同一签名的方法做出不同响应才算多态，也有观点认为只要使用同一形式的接口去实现不同类的行为就算多态。笔者看来这种争论并无太大意义，概念仅仅是说明问题的一种工具而已。
[^49]:指普通方法的解析过程，有一些特殊情况（签名多态性方法）的解析过程会稍有区别，但这是用于支持动态语言调用的，与本节话题关系不大。
[^50]:这是一个2010年诞生的老梗了，尽管当时很轰动，但现在可能很多人都并不了解事情始末，这并不会影响本文的阅读。如有兴趣具体可参考：https://zhuanlan.zhihu.com/p/19609988。
[^51]:JEP 276的Owner是Attila Szegedi，jdk.dynalink包实质就是把他自己写的开源项目dynalink变成了Java标准的API，所以读者对jdk.dynalink感兴趣的话可以参考：https://github.com/szegedi/dynalink。
[^52]:JSR-292：Supporting Dynamically Typed Languages on the Java Platform.（Java平台的动态语言支持）。
[^53]:这里的“提高性能”是相对于直接搜索元数据来说的，实际上在HotSpot虚拟机的实现中，直接去查itable和vtable已经算是最慢的一种分派，只在解释执行状态时使用，在即时编译执行时，会有更多的性能优化措施，具体可常见第11章关于方法内联的内容。

## 8.4 动态类型语言支持

​	Java虚拟机的字节码指令集的数量自从Sun公司的第一款Java虚拟机问世至今，**二十余年间只新增过一条指令，它就是随着JDK 7的发布的字节码首位新成员——invokedynamic指令**。

​	这条新增加的指令是JDK 7的项目目标：<u>实现动态类型语言（Dynamically Typed Language）</u>支持而进行的改进之一，也是为JDK 8里可以顺利实现Lambda表达式而做的技术储备。在本节中，我们将详细了解动态语言支持这项特性出现的前因后果和它的意义与价值。

### 8.4.1 动态类型语言

​	在介绍Java虚拟机的动态类型语言支持之前，我们要先弄明白动态类型语言是什么？它与Java语言、Java虚拟机有什么关系？了解Java虚拟机提供动态类型语言支持的技术背景，对理解这个语言特性是非常有必要的。

​	何谓动态类型语言[^54]？**动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期进行的**，满足这个特征的语言有很多，常用的包括：APL、Clojure、Erlang、Groovy、JavaScript、Lisp、Lua、PHP、Prolog、Python、Ruby、Smalltalk、Tcl，等等。**那相对地，在编译期就进行类型检查过程的语言，譬如C++和Java等就是最常用的静态类型语言**。

​	如果读者觉得上面的定义过于概念化，那我们不妨通过两个例子以最浅显的方式来说明什么是“类型检查”和什么叫“在编译期还是在运行期进行”。首先看下面这段简单的Java代码，思考一下它是否能正常编译和运行？

```java
public static void main(String[] args) {
  int[][][] array = new int[1][0][-1];
}
```

​	上面这段Java代码能够正常编译，但运行的时候会出现NegativeArraySizeException异常。在《Java虚拟机规范》中明确规定了NegativeArraySizeException是一个运行时异常（Runtime Exception），**通俗一点说，运行时异常就是指只要代码不执行到这一行就不会出现问题**。<u>与运行时异常相对应的概念是连接时异常，例如很常见的NoClassDefFoundError便属于连接时异常，即使导致连接时异常的代码放在一条根本无法被执行到的路径分支上，类加载时（第7章解释过Java的连接过程不在编译阶段，而在类加载阶段）也照样会抛出异常</u>。

​	不过，在C语言里，语义相同的代码就会在编译期就直接报错，而不是等到运行时才出现异常：

```java
int main(void) {
  int i[1][0][-1]; // GCC拒绝编译，报“size of array is negative”
  return 0;
}
```

​	由此看来，一门语言的哪一种检查行为要在运行期进行，哪一种检查要在编译期进行并没有什么必然的因果逻辑关系，关键是在语言规范中人为设立的约定。

​	解答了什么是“连接时、运行时”，笔者再举一个例子来解释什么是“类型检查”，例如下面这一句再普通不过的代码：

```java
obj.println("hello world");
```

​	虽然正在阅读本书的每一位读者都能看懂这行代码要做什么，但对于计算机来讲，这一行“没头没尾”的代码是无法执行的，它需要一个具体的上下文中（譬如程序语言是什么、obj是什么类型）才有讨论的意义。

​	现在先假设这行代码是在Java语言中，并且变量obj的静态类型为java.io.PrintStream，那变量obj的实际类型就必须是PrintStream的子类（实现了PrintStream接口的类）才是合法的。否则，哪怕obj属于一个确实包含有println(String)方法相同签名方法的类型，但只要它与PrintStream接口没有继承关系，代码依然不可能运行——因为类型检查不合法。

​	<u>但是相同的代码在ECMAScript（JavaScript）中情况则不一样，无论obj具体是何种类型，无论其继承关系如何，只要这种类型的方法定义中确实包含有println(String)方法，能够找到相同签名的方法，调用便可成功</u>。

​	产生这种差别产生的根本原因是<u>Java语言在编译期间却已将println(String)方法完整的符号引用（本例中为一项CONSTANT_InterfaceMethodref_info常量）生成出来</u>，并作为方法调用指令的参数存储到Class文件中，例如下面这个样子：

```shell
invokevirtual #4; //Method java/io/PrintStream.println:(Ljava/lang/String;)V
```

​	这个符号引用包含了该方法定义在哪个具体类型之中、方法的名字以及参数顺序、参数类型和方法返回值等信息，通过这个符号引用，Java虚拟机就可以翻译出该方法的直接引用。**而ECMAScript等动态类型语言与Java有一个核心的差异就是变量obj本身并没有类型，变量obj的值才具有类型，所以编译器在编译时最多只能确定方法名称、参数、返回值这些信息，而不会去确定方法所在的具体类型（即方法接收者不固定）**。

​	**“变量无类型而变量值才有类型”这个特点也是动态类型语言的一个核心特征。**

​	了解了动态类型和静态类型语言的区别后，也许读者的下一个问题就是动态、静态类型语言两者谁更好，或者谁更加先进呢？这种比较不会有确切答案，它们都有自己的优点，选择哪种语言是需要权衡的事情。

+ <u>静态类型语言能够在编译期确定变量类型，最显著的好处是编译器可以提供全面严谨的类型检查，这样与数据类型相关的潜在问题就能在编码时被及时发现，利于稳定性及让项目容易达到更大的规模。</u>
+ <u>而动态类型语言在运行期才确定类型，这可以为开发人员提供极大的灵活性，某些在静态类型语言中要花大量臃肿代码来实现的功能，由动态类型语言去做可能会很清晰简洁，清晰简洁通常也就意味着开发效率的提升</u>。

[^54]:注意，动态类型语言与动态语言、弱类型语言并不是一个概念，需要区别对待。

### 8.4.2 Java与动态类型

​	现在我们回到本节的主题，来看看Java语言、Java虚拟机与动态类型语言之间有什么关系。Java虚拟机毫无疑问是Java语言的运行平台，但它的使命并不限于此，早在1997年出版的《Java虚拟机规范》第1版中就规划了这样一个愿景：“在未来，我们会对Java虚拟机进行适当的扩展，以便更好地支持其他语言运行于Java虚拟机之上。”而目前确实已经有许多动态类型语言运行于Java虚拟机之上了，如Clojure、Groovy、Jython和JRuby等，能够在同一个虚拟机之上可以实现静态类型语言的严谨与动态类型语言的灵活，这的确是一件很美妙的事情。

​	但遗憾的是Java虚拟机层面对动态类型语言的支持一直都还有所欠缺，主要表现在方法调用方面：JDK 7以前的字节码指令集中，4条方法调用指令（invokevirtual、invokespecial、invokestatic、invokeinterface）的第一个参数都是被调用的方法的符号引用（CONSTANT_Methodref_info或者CONSTANT_InterfaceMethodref_info常量），前面已经提到过，**方法的符号引用在编译时产生，而动态类型语言只有在运行期才能确定方法的接收者**。这样，在Java虚拟机上实现的动态类型语言就不得不使用“曲线救国”的方式（如编译时留个占位符类型，运行时动态生成字节码实现具体类型到占位符类型的适配）来实现，但这样势必会让动态类型语言实现的复杂度增加，也会带来额外的性能和内存开销。内存开销是很显而易见的，方法调用产生的那一大堆的动态类就摆在那里。而其中最严重的性能瓶颈是在于动态类型方法调用时，由于无法确定调用对象的静态类型，而导致的方法内联无法有效进行。在第11章里我们会讲到方法内联的重要性，它是其他优化措施的基础，也可以说是最重要的一项优化。尽管也可以想一些办法（譬如调用点缓存）尽量缓解支持动态语言而导致的性能下降，但这种改善毕竟不是本质的。譬如有类似以下代码：

```java
var arrays = {"abc", new ObjectX(), 123, Dog, Cat, Car..}
for(item in arrays){
  item.sayHello();
}
```

​	<u>在动态类型语言下这样的代码是没有问题，但由于在运行时arrays中的元素可以是任意类型，即使它们的类型中都有sayHello()方法，也肯定无法在编译优化的时候就确定具体sayHello()的代码在哪里，编译器只能不停编译它所遇见的每一个sayHello()方法，并缓存起来供执行时选择、调用和内联，如果arrays数组中不同类型的对象很多，就势必会对内联缓存产生很大的压力，缓存的大小总是有限的，类型信息的不确定性导致了缓存内容不断被失效和更新，先前优化过的方法也可能被不断替换而无法重复使用</u>。

​	所以**这种动态类型方法调用的底层问题终归是应当在Java虚拟机层次上去解决才最合适。因此，在Java虚拟机层面上提供动态类型的直接支持就成为Java平台发展必须解决的问题**，这便是JDK 7时JSR-292提案中invokedynamic指令以及java.lang.invoke包出现的技术背景。

### 8.4.3 java.lang.invoke包

​	JDK 7时新加入的java.lang.invoke包[^55]是JSR 292的一个重要组成部分，这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这条路之外，提供一种新的**动态确定目标方法**的机制，称为**“方法句柄”（Method Handle）**。这个表达听起来也不好懂？那不妨把方法句柄与C/C++中的函数指针（Function Pointer），或者C#里面的委派（Delegate）互相类比一下来理解。举个例子，如果我们要实现一个带谓词（谓词就是由外部传入的排序时比较大小的动作）的排序函数，在C/C++中的常用做法是把谓词定义为函数，用函数指针来把谓词传递到排序方法，像这样：

```java
void sort(int list[], const int size, int (*compare)(int, int))
```

​	但**在Java语言中做不到这一点，没有办法单独把一个函数作为参数进行传递**。普遍的做法是设计一个带有compare()方法的Comparator接口，以实现这个接口的对象作为参数，例如Java类库中的Collections::sort()方法就是这样定义的：

```java
void sort(List list, Comparator c)
```

​	不过，<u>在拥有方法句柄之后，Java语言也可以拥有类似于函数指针或者委托的方法别名这样的工具了</u>。代码清单8-12演示了方法句柄的基本用法，无论obj是何种类型（临时定义的ClassA抑或是实现PrintStream接口的实现类System.out），都可以正确调用到println()方法。

​	代码清单8-12　方法句柄演示

```java
import static java.lang.invoke.MethodHandles.lookup;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;
/**
* JSR 292 MethodHandle基础用法演示
* @author zzm
*/
public class MethodHandleTest {
  static class ClassA {
    public void println(String s) {
      System.out.println(s);
    }
  }
  public static void main(String[] args) throws Throwable {
    Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
    // 无论obj最终是哪个实现类，下面这句都能正确调用到println方法。
    getPrintlnMH(obj).invokeExact("icyfenix");
  }
  private static MethodHandle getPrintlnMH(Object reveiver) throws Throwable {
    // MethodType：代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）和具体参数（methodType()第二个及以后的参数）。
    MethodType mt = MethodType.methodType(void.class, String.class);
    // lookup()方法来自于MethodHandles.lookup，这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
    // 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即this指向的对象，这个参数以前是放在参数列表中进行传递，现在提供了bindTo()方法来完成这件事情。
    return lookup().findVirtual(reveiver.getClass(), "println", mt).bindTo(reveiver);
  }
}
```

​	方法getPrintlnMH()中实际上是模拟了invokevirtual指令的执行过程，只不过它的分派逻辑并非固化在Class文件的字节码上，而是通过一个由用户设计的Java方法来实现。而这个方法本身的返回值（MethodHandle对象），可以视为对最终调用方法的一个“引用”。以此为基础，有了MethodHandle就可以写出类似于C/C++那样的函数声明了：

```java
void sort(List list, MethodHandle compare)
```

​	从上面的例子看来，使用MethodHandle并没有多少困难，不过看完它的用法之后，读者大概就会产生疑问，相同的事情，用反射不是早就可以实现了吗？

​	确实，仅站在Java语言的角度看，MethodHandle在使用方法和效果上与Reflection有众多相似之处。不过，它们也有以下这些区别：

+ **Reflection和MethodHandle机制本质上都是在模拟方法调用，但是Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用**。在MethodHandles.Lookup上的3个方法findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual（以及invokeinterface）和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。

+ Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息来得多。<u>前者是方法在Java端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含执行权限等的运行期信息。而后者仅包含执行该方法的相关信息</u>。用开发人员通俗的话来讲，**Reflection是重量级，而MethodHandle是轻量级**。

+ **由于MethodHandle是对字节码的方法指令调用的模拟，那理论上虚拟机在这方面做的各种优化（如方法内联），在MethodHandle上也应当可以采用类似思路去支持（但目前实现还在继续完善中），而通过反射去调用方法则几乎不可能直接去实施各类调用点优化措施**。

​	MethodHandle与Reflection除了上面列举的区别外，最关键的一点还在于去掉前面讨论施加的前提“仅站在Java语言的角度看”之后：**Reflection API的设计目标是只为Java语言服务的，而MethodHandle则设计为可服务于所有Java虚拟机之上的语言，其中也包括了Java语言而已，而且Java在这里并不是主角**。

[^55]:这个包曾经在不算短的时间里的名称是java.dyn，也曾经短暂更名为java.lang.mh，如果读者在其他资料上看到这两个包名，可以把它们与java.lang.invoke理解为同一种东西。

### 8.4.4 invokedynamic指令

​	8.4节一开始就提到了JDK 7为了更好地支持动态类型语言，引入了第五条方法调用的字节码指令invokedynamic，之后却一直没有再提起它，甚至把代码清单8-12使用MethodHandle的示例代码反编译后也完全找不到invokedynamic的身影，这实在与invokedynamic作为Java诞生以来唯一一条新加入的字节码指令的地位不相符，那么invokedynamic到底有什么应用呢？

​	某种意义上可以说invokedynamic指令与MethodHandle机制的作用是一样的，都是为了解决原有4条“invoke*”指令方法分派规则完全固化在虚拟机之中的问题，把如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之中，让用户（广义的用户，包含其他程序语言的设计者）有更高的自由度。而且，它们两者的思路也是可类比的，都是为了达成同一个目的，只是一个用上层代码和API来实现，另一个用字节码和Class中其他属性、常量来完成。因此，如果前面MethodHandle的例子看懂了，相信读者理解invokedynamic指令并不困难。

​	**每一处含有invokedynamic指令的位置都被称作“动态调用点（Dynamically-Computed Call Site）”**，这条指令的第一个参数不再是代表方法符号引用的CONSTANT_Methodref_info常量，而是变为JDK 7时新加入的CONSTANT_InvokeDynamic_info常量，从这个新常量中可以得到3项信息：引导方法（Bootstrap Method，该方法存放在新增的BootstrapMethods属性中）、方法类型（MethodType）和名称。引导方法是有固定的参数，并且返回值规定是java.lang.invoke.CallSite对象，这个对象代表了真正要执行的目标方法调用。根据CONSTANT_InvokeDynamic_info常量中提供的信息，虚拟机可以找到并且执行引导方法，从而获得一个CallSite对象，最终调用到要执行的目标方法上。我们还是照例不依赖枯燥的概念描述，改用一个实际例子来解释这个过程吧，如代码清单8-13所示。

​	代码清单8-13　InvokeDynamic指令演示

```java
import static java.lang.invoke.MethodHandles.lookup;
import java.lang.invoke.CallSite;
import java.lang.invoke.ConstantCallSite;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
public class InvokeDynamicTest{
  public static void main(String[]args)throws Throwable{
    INDY_BootstrapMethod().invokeExact("icyfenix");
  }
  public static void testMethod(String s){
    System.out.println("hello String："+s);
  }
  public static CallSite BootstrapMethod(MethodHandles.Lookup lookup,String name,MethodType mt)throws Throwable{
    return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class,name,mt));
  }
  private static MethodType MT_BootstrapMethod(){
    return MethodType.fromMethodDescriptorString("(Ljava/lang/invoke/MethodHandles $Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;"，null);
  }
  private static MethodHandle MH_BootstrapMethod()throws Throwable{
    return lookup().findStatic(InvokeDynamicTest.class，"BootstrapMethod"，MT_BootstrapMethod());
  }
  private static MethodHandle INDY_BootstrapMethod()throws Throwable{
    CallSite cs=(CallSite)MH_BootstrapMethod().invokeWithArguments(lookup()，"testMethod"，MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V"，null));
    return cs.dynamicInvoker();
  }
}
```

​	这段代码与前面MethodHandleTest的作用基本上是一样的，虽然笔者没有加以注释，但是阅读起来应当也不困难。要是真没读懂也不要紧，笔者没写注释的主要原因是这段代码并非写给人看的，只是为了方便编译器按照笔者的意愿来产生一段字节码而已。前文提到过，由于invokedynamic指令面向的主要服务对象并非Java语言，而是其他Java虚拟机之上的其他动态类型语言，因此，光靠Java语言的编译器Javac的话，在JDK 7时甚至还完全没有办法生成带有invokedynamic指令的字节码（曾经有一个java.dyn.InvokeDynamic的语法糖可以实现，但后来被取消了），而到JDK 8引入了Lambda表达式和接口默认方法后，Java语言才算享受到了一点invokedynamic指令的好处，但用Lambda来解释invokedynamic指令运作就比较别扭，也无法与前面MethodHandle的例子对应类比，所以笔者采用一些变通的办法：John Rose（JSR 292的负责人，以前Da Vinci Machine Project的Leader）编写过一个把程序的字节码转换为使用invokedynamic的简单工具INDY[^56]来完成这件事，我们要使用这个工具来产生最终需要的字节码，因此代码清单8-13中的方法名称不能随意改动，更不能把几个方法合并到一起写，因为它们是要被INDY工具读取的。

​	把上面的代码编译，再使用INDY转换后重新生成的字节码如代码清单8-14所示（结果使用javap输出，因版面原因，精简了许多无关的内容）。

​	代码清单8-14　InvokeDynamic指令演示（2）

```shell
Constant pool:
#121 = NameAndType #33:#30 // testMethod:(Ljava/lang/String;)V
#123 = InvokeDynamic #0:#121 // #0:testMethod:(Ljava/lang/String;)V
public static void main(java.lang.String[]) throws java.lang.Throwable;
Code:
stack=2, locals=1, args_size=1
0: ldc #23 // String abc
2: invokedynamic #123, 0 // InvokeDynamic #0:testMethod: (Ljava/lang/String;)V
7: nop
8: return
public static java.lang.invoke.CallSite BootstrapMethod(java.lang.invoke.Method Handles$Lookup, java.lang.String, Code:
stack=6, locals=3, args_size=3
0: new #63 // class java/lang/invoke/ConstantCallSite
3: dup
4: aload_0
5: ldc #1 // class org/fenixsoft/InvokeDynamicTest
7: aload_1
8: aload_2
9: invokevirtual #65 // Method java/lang/invoke/MethodHandles$ Lookup.findStatic:(Ljava/lang/Class;
12: invokespecial #71 // Method java/lang/invoke/ConstantCallSite. "<init>":(Ljava/lang/invoke/MethodHandle;)
15: areturn
```

​	从main()方法的字节码中可见，原本的方法调用指令已经被替换为invokedynamic了，它的参数为第123项常量（第二个值为0的参数在虚拟机中不会直接用到，这与invokeinterface指令那个的值为0的参数一样是占位用的，目的都是为了给常量池缓存留出足够的空间）：

```shell
2: invokedynamic #123, 0 // InvokeDynamic #0:testMethod:(Ljava/lang/String;)V
```

​	从常量池中可见，第123项常量显示“#123=InvokeDynamic#0：#121”说明它是一项CONSTANT_InvokeDynamic_info类型常量，常量值中前面“#0”代表引导方法取Bootstrap Methods属性表的第0项（javap没有列出属性表的具体内容，不过示例中仅有一个引导方法，即BootstrapMethod()），而后面的“#121”代表引用第121项类型为CONSTANT_NameAndType_info的常量，从这个常量中可以获取到方法名称和描述符，即后面输出的“testMethod：(Ljava/lang/String；)V”。

​	再看BootstrapMethod()，这个方法在Java源码中并不存在，是由INDY产生的，但是它的字节码很容易读懂，所有逻辑都是调用MethodHandles$Lookup的findStatic()方法，产生testMethod()方法的MethodHandle，然后用它创建一个ConstantCallSite对象。最后，这个对象返回给invokedynamic指令实现对testMethod()方法的调用，invokedynamic指令的调用过程到此就宣告完成了。

[^56]:INDY下载地址：http://blogs.oracle.com/jrose/entry/a_modest_tool_for_writing。

### 8.4.5 实战：掌控方法分派规则

​	invokedynamic指令与此前4条传统的“invoke*”指令的最大区别就是它的分派逻辑不是由虚拟机决定的，而是由程序员决定。在介绍Java虚拟机动态语言支持的最后一节中，笔者希望通过一个简单例子（如代码清单8-15所示），帮助读者理解程序员可以掌控方法分派规则之后，我们能做什么以前无法做到的事情。

​	代码清单8-15　方法调用问题

```java
class GrandFather {
  void thinking() {
    System.out.println("i am grandfather");
  }
}
class Father extends GrandFather {
  void thinking() {
    System.out.println("i am father");
  }
}
class Son extends Father {
  void thinking() {
    // 请读者在这里填入适当的代码（不能修改其他地方的代码）
    // 实现调用祖父类的thinking()方法，打印"i am grandfather"
  }
}
```

​	在Java程序中，可以通过“super”关键字很方便地调用到父类中的方法，但如果要访问祖类的方法呢？读者在往下阅读本书提供的解决方案之前，不妨自己思考一下，在JDK 7之前有没有办法解决这个问题。

​	在拥有invokedynamic和java.lang.invoke包之前，使用纯粹的Java语言很难处理这个问题（使用ASM等字节码工具直接生成字节码当然还是可以处理的，但这已经是在字节码而不是Java语言层面来解决问题了），原因是在Son类的thinking()方法中根本无法获取到一个实际类型是GrandFather的对象引用，而<u>invokevirtual指令的分派逻辑是固定的，只能按照方法接收者的实际类型进行分派，这个逻辑完全固化在虚拟机中，程序员无法改变</u>。如果是JDK 7 Update 9之前，使用代码清单8-16中的程序就可以直接解决该问题。

​	代码清单8-16　使用MethodHandle来解决问题

```java
import static java.lang.invoke.MethodHandles.lookup;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;
class Test {
  class GrandFather {
    void thinking() {
      System.out.println("i am grandfather");
    }
  }
  class Father extends GrandFather {
    void thinking() {
      System.out.println("i am father");
    }
  }
  class Son extends Father {
    void thinking() {
      try {
        MethodType mt = MethodType.methodType(void.class);
        MethodHandle mh = lookup().findSpecial(GrandFather.class,
                                               "thinking", mt, getClass());
        mh.invoke(this);
      } catch (Throwable e) {
      }
    }
  }
  public static void main(String[] args) {
    (new Test().new Son()).thinking();
  }
}
```

​	使用JDK 7 Update 9之前的HotSpot虚拟机运行，会得到如下运行结果：

```java
i am grandfather
```

​	<u>但是这个逻辑在JDK 7 Update 9之后被视作一个潜在的安全性缺陷修正了，原因是必须保证findSpecial()查找方法版本时受到的访问约束（譬如对访问控制的限制、对参数类型的限制）应与使用invokespecial指令一样，两者必须保持精确对等，包括在上面的场景中它只能访问到其直接父类中的方法版本</u>。所以在JDK 7 Update 10修正之后，运行以上代码只能得到如下结果：

```shell
i am father
```

​	由于本书的第2版是基于早期版本的JDK 7撰写的，所以印刷之后才发布的JDK更新就很难再及时地同步修正了，这导致不少读者重现这段代码的运行结果时产生了疑惑，也收到了很多热心读者的邮件，在此一并感谢。

​	那在新版本的JDK中，上面的问题是否能够得到解决呢？答案是可以的，如果读者去查看MethodHandles.Lookup类的代码，将会发现需要进行哪些访问保护，在该API实现时是预留了后门的。**访问保护是通过一个allowedModes的参数来控制，而且这个参数可以被设置成“TRUSTED”来绕开所有的保护措施**。**尽管这个参数只是在Java类库本身使用，没有开放给外部设置，但我们通过反射可以轻易打破这种限制**。由此，我们可以把代码清单8-16中子类的thinking()方法修改为如下所示的代码来解决问题：

```java
void thinking() {
  try {
    MethodType mt = MethodType.methodType(void.class);
    Field lookupImpl = MethodHandles.Lookup.class.getDeclaredField("IMPL_LOOKUP");
    lookupImpl.setAccessible(true);
    MethodHandle mh = ((MethodHandles.Lookup) lookupImpl.get(null)).findSpecial(GrandFather.class,"thinking", mh.invoke(this);
   } catch (Throwable e) {}
}
```

​	运行以上代码，在目前所有JDK版本中均可获得如下结果：

```shell
i am grandfather
```

## 8.5 基于栈的字节码解释执行引擎

​	关于Java虚拟机是如何调用方法、进行版本选择的内容已经全部讲解完毕，从本节开始，我们来探讨虚拟机是如何执行方法里面的字节码指令的。概述中曾提到过，许多Java虚拟机的执行引擎在执行Java代码的时候都有**解释执行（通过解释器执行）**和**编译执行（通过即时编译器产生本地代码执行）**两种选择，在本节中，我们将会分析在概念模型下的Java虚拟机解释执行字节码时，其执行引擎是如何工作的。笔者在本章多次强调了“概念模型”，是因为实际的虚拟机实现，譬如HotSpot的模板解释器工作的时候，并不是按照下文中的动作一板一眼地进行机械式计算，而是动态产生每条字节码对应的汇编代码来运行，这与概念模型中执行过程的差异很大，但是结果却能保证是一致的。

### 8.5.1 解释执行

> [本篇来自周志明的<<深入理解java虚拟机>>](https://www.kanzhun.com/jiaocheng/228363.html)	<=	下方图片来源

​	Java语言经常被人们定位为“解释执行”的语言，在Java初生的JDK 1.0时代，这种定义还算是比较准确的，但当主流的虚拟机中都包含了即时编译器后，Class文件中的代码到底会被解释执行还是编译执行，就成了只有虚拟机自己才能准确判断的事。再后来，Java也发展出可以直接生成本地代码的编译器（如Jaotc、GCJ[^57]，Excelsior JET），而C/C++语言也出现了通过解释器执行的版本（如CINT[^58]），这时候再笼统地说“解释执行”，对于整个Java语言来说就成了几乎是没有意义的概念，只有确定了谈论对象是某种具体的Java实现版本和执行引擎运行模式时，谈解释执行还是编译执行才会比较合理确切。

​	无论是解释还是编译，也无论是物理机还是虚拟机，对于应用程序，机器都不可能如人那样阅读、理解，然后获得执行能力。大部分的程序代码转换成物理机的目标代码或虚拟机能执行的指令集之前，都需要经过图8-4中的各个步骤。如果读者对大学**编译原理**的相关课程还有印象的话，很容易就会发现图8-4中下面的那条分支，就是传统编译原理中程序代码到目标机器代码的生成过程；而中间的那条分支，自然就是解释执行的过程。

![img](https://img.kanzhun.com/images/seo/mianshiti/20191125/9c7cd3d48f863e52da4017d9cee31ca0.jpg)

​	如今，基于物理机、Java虚拟机，或者是非Java的其他高级语言虚拟机（HLLVM）的代码执行过程，大体上都会遵循这种符合现代经典编译原理的思路，**在执行前先对程序源码进行词法分析和语法分析处理，把源码转化为抽象语法树（Abstract Syntax Tree，AST）**。

​	<u>对于一门具体语言的实现来说，词法、语法分析以至后面的优化器和目标代码生成器都可以选择独立于执行引擎，形成一个完整意义的编译器去实现，这类代表是C/C++语言。也可以选择把其中一部分步骤（如生成抽象语法树之前的步骤）实现为一个半独立的编译器，这类代表是Java语言。又或者把这些步骤和执行引擎全部集中封装在一个封闭的黑匣子之中，如大多数的JavaScript执行引擎。</u>

​	在Java语言中，Javac编译器完成了程序代码经过词法分析、语法分析到抽象语法树，再遍历语法树生成线性的字节码指令流的过程。因为这一部分动作是在Java虚拟机之外进行的，而解释器在虚拟机的内部，所以Java程序的编译就是半独立的实现。

[^57]:GCJ：http://gcc.gnu.org/java/。
[^58]:CINT：http://root.cern.ch/drupal/content/cint。

### 8.5.2 基于栈的指令集与基于寄存器的指令集

​	Javac编译器输出的字节码指令流，基本上[^59]是一种基于栈的指令集架构（Instruction SetArchitecture，ISA），<u>字节码指令流里面的指令大部分都是零地址指令，它们依赖操作数栈进行工作</u>。与之相对的另外一套常用的指令集架构是基于寄存器的指令集，最典型的就是x86的二地址指令集，如果说得更通俗一些就是现在我们主流PC机中物理硬件直接支持的指令集架构，这些指令依赖寄存器进行工作。那么，基于栈的指令集与基于寄存器的指令集这两者之间有什么不同呢？

​	举个最简单的例子，分别使用这两种指令集去计算“1+1”的结果，基于栈的指令集会是这样子的：

```shell
iconst_1
iconst_1
iadd
istore_0
```

​	两条iconst_1指令连续把两个常量1压入栈后，iadd指令把栈顶的两个值出栈、相加，然后把结果放回栈顶，最后istore_0把栈顶的值放到局部变量表的第0个变量槽中。这种指令流中的指令通常都是不带参数的，使用操作数栈中的数据作为指令的运算输入，指令的运算结果也存储在操作数栈之中。而如果用基于寄存器的指令集，那程序可能会是这个样子：

```shell
mov eax, 1
add eax, 1
```

​	mov指令把EAX寄存器的值设为1，然后add指令再把这个值加1，结果就保存在EAX寄存器里面。这种二地址指令是x86指令集中的主流，每个指令都包含两个单独的输入参数，依赖于寄存器来访问和存储数据。

​	了解了基于栈的指令集与基于寄存器的指令集的区别后，读者可能会有个进一步的疑问，这两套指令集谁更好一些呢？

​	应该说，既然两套指令集会同时并存和发展，那肯定是各有优势的，如果有一套指令集全面优于另外一套的话，就是直接替代而不存在选择的问题了。

​	**基于栈的指令集主要优点是可移植，因为寄存器由硬件直接提供**[^60]，**程序直接依赖这些硬件寄存器则不可避免地要受到硬件的约束**。例如现在32位80x86体系的处理器能提供了8个32位的寄存器，而ARMv6体系的处理器（在智能手机、数码设备中相当流行的一种处理器）则提供了30个32位的通用寄存器，其中前16个在用户模式中可以使用。如果使用栈架构的指令集，用户程序不会直接用到这些寄存器，那就可以由虚拟机实现来自行决定把一些访问最频繁的数据（程序计数器、栈顶缓存等）放到寄存器中以获取尽量好的性能，这样实现起来也更简单一些。**栈架构的指令集还有一些其他的优点，如代码相对更加紧凑（字节码中每个字节就对应一条指令，而多地址指令集中还需要存放参数）、编译器实现更加简单（不需要考虑空间分配的问题，所需空间都在栈上操作）等**。

​	**栈架构指令集的主要缺点是理论上执行速度相对来说会稍慢一些，所有主流物理机的指令集都是寄存器架构[^61]也从侧面印证了这点**。

​	**不过这里的执行速度是要局限在解释执行的状态下，如果经过即时编译器输出成物理机上的汇编指令流，那就与虚拟机采用哪种指令集架构没有什么关系了**。

​	在解释执行时，栈架构指令集的代码虽然紧凑，但是完成相同功能所需的指令数量一般会比寄存器架构来得更多，因为出栈、入栈操作本身就产生了相当大量的指令。**<u>更重要的是栈实现在内存中，频繁的栈访问也就意味着频繁的内存访问，相对于处理器来说，内存始终是执行速度的瓶颈</u>**。

​	**尽管虚拟机可以采取栈顶缓存的优化方法，把最常用的操作映射到寄存器中避免直接内存访问，但这也只是优化措施而不是解决本质问题的方法。因此由于指令数量和内存访问的原因，导致了栈架构指令集的执行速度会相对慢上一点**。

[^59]:使用“基本上”，是因为部分字节码指令会带有参数，而纯粹基于栈的指令集架构中应当全部都是零地址指令，也就是都不存在显式的参数。Java这样实现主要是考虑了代码的可校验性。
[^60]:这里说的是物理机器上的寄存器。也有基于寄存器的虚拟机，如Google Android平台的Dalvik虚拟机。即使是基于寄存器的虚拟机，也会希望把虚拟机寄存器尽量映射到物理寄存器上以获取尽可能高的性能。
[^61]:Intel x86架构早期的数学协处理器x87（譬如与8086搭配工作的8087）就是基于栈的，只操作栈顶的两个数据。但是实际常见的物理机处理器已经很久不用这种架构了。

### 8.5.3 基于栈的解释器执行过程

> [Java虚拟机（6）-----虚拟机字节码执行引擎](https://blog.csdn.net/zh328271057/article/details/81021627)	<=	下方图片来源

​	关于栈架构执行引擎的必要前置知识已经全部讲解完毕了，本节笔者准备了一段Java代码，以便向读者实际展示在虚拟机里字节码是如何执行的。前面笔者曾经举过一个计算“1+1”的例子，那种小学一年级的算数题目显然太过简单了，给聪明的读者练习的题目起码……嗯，笔者准备的是四则运算加减乘除法，大概能达到三年级左右的数学水平，请看代码清单8-17。

​	代码清单8-17　一段简单的算术代码

```java
public int calc() {
  int a = 100;
  int b = 200;
  int c = 300;
  return (a + b) * c;
}
```

​	这段代码从Java语言的角度没有任何谈论的必要，直接使用javap命令看看它的字节码指令，如代码清单8-18所示。

​	代码清单8-18　一段简单的算术代码的字节码表示

```shell
public int calc();
	Code:
    Stack=2, Locals=4, Args_size=1
    0: bipush 100
    2: istore_1
    3: sipush 200
    6: istore_2
    7: sipush 300
    10: istore_3
    11: iload_1
    12: iload_2
    13: iadd
    14: iload_3
    15: imul
    16: ireturn
}
```

​	javap提示这段代码需要深度为2的操作数栈和4个变量槽的局部变量空间，笔者就根据这些信息画了图8-5至图8-11共7张图片，来描述代码清单8-13执行过程中的代码、操作数栈和局部变量表的变化情况。

![img](https://img-blog.csdn.net/20160722150834157)

​	首先，执行偏移地址为0的指令，Bipush指令的作用是将单字节的整型常量值（-128～127）推入操作数栈顶，跟随有一个参数，指明推送的常量值，这里是100。

![img](https://img-blog.csdn.net/20160722150841294)

​	执行偏移地址为2的指令，istore_1指令的作用是将操作数栈顶的整型值出栈并存放到第1个局部变量槽中。后续4条指令（直到偏移为11的指令为止）都是做一样的事情，也就是在对应代码中把变量a、b、c赋值为100、200、300。这4条指令的图示略过。

![img](https://img-blog.csdn.net/20160722150847404)

​	执行偏移地址为11的指令，iload_1指令的作用是将局部变量表第1个变量槽中的整型值复制到操作数栈顶。

![img](https://img-blog.csdn.net/20160722150852326)

​	执行偏移地址为12的指令，iload_2指令的执行过程与iload_1类似，把第2个变量槽的整型值入栈。画出这个指令的图示主要是为了显示下一条iadd指令执行前的堆栈状况。

![img](https://img-blog.csdn.net/20160722150901360)

​	执行偏移地址为13的指令，iadd指令的作用是将操作数栈中头两个栈顶元素出栈，做整型加法，然后把结果重新入栈。在iadd指令执行完毕后，栈中原有的100和200被出栈，它们的和300被重新入栈。

![img](https://img-blog.csdn.net/20160722150906501)

​	执行偏移地址为14的指令，iload_3指令把存放在第3个局部变量槽中的300入栈到操作数栈中。这时操作数栈为两个整数300。下一条指令imul是将操作数栈中头两个栈顶元素出栈，做整型乘法，然后把结果重新入栈，与iadd完全类似，所以笔者省略图示。

![img](https://img-blog.csdn.net/20160722150911173)

​	执行偏移地址为16的指令，ireturn指令是方法返回指令之一，它将结束方法执行并将操作数栈顶的整型值返回给该方法的调用者。到此为止，这段方法执行结束。

​	再次强调上面的执行过程仅仅是一种概念模型，虚拟机最终会对执行过程做出一系列优化来提高性能，实际的运作过程并不会完全符合概念模型的描述。更确切地说，实际情况会和上面描述的概念模型差距非常大，差距产生的根本原因是虚拟机中解析器和即时编译器都会对输入的字节码进行优化，即使解释器中也不是按照字节码指令去逐条执行的。例如在HotSpot虚拟机中，就有很多以“fast_”开头的非标准字节码指令用于合并、替换输入的字节码以提升解释执行性能，即时编译器的优化手段则更是花样繁多[^62]。

​	不过我们从这段程序的执行中也可以看出栈结构指令集的一般运行过程，整个运算过程的中间变量都以操作数栈的出栈、入栈为信息交换途径，符合我们在前面分析的特点。

[^62]:具体可以参考第11章的相关内容。

## 8.6 本章小结

​	本章中，我们分析了虚拟机在执行代码时，如何找到正确的方法，如何执行方法内的字节码，以及执行代码时涉及的内存结构。在第6～8章里面，我们针对Java程序是如何存储的、如何载入（创建）的，以及如何执行的问题，把相关知识系统地介绍了一遍，第9章我们将一起看看这些理论知识在具体开发之中的典型应用。

# 9. 类加载及执行子系统的案例与实战



[^63]:
[^64]:
[^65]:
[^66]:
[^67]:
[^68]:
[^69]:
[^70]:
[^71]:

