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

[^6]: