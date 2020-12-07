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
| Tand      |         |         | iand      | Land    |         |         |         |           |
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

​	加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈（见第2章关于内存区域的介绍）之间来回传输，这类指令包括：



[^13]: