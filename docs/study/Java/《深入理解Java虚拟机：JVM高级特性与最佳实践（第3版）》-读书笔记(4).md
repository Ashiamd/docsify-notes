# 《深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）》-读书笔记(4)

# 第四部分 程序编译与代码优化

# 10 前端编译与优化

## 10.1 概述

​	在Java技术下谈“编译期”而没有具体上下文语境的话，其实是一句很含糊的表述，因为它可能是指一个前端编译器（叫“编译器的前端”更准确一些）把`*.java`文件转变成`*.class`文件的过程；也可能是指Java虚拟机的即时编译器（常称JIT编译器，Just In Time Compiler）运行期把字节码转变成本地机器码的过程；还可能是指使用静态的提前编译器（常称AOT编译器，Ahead Of Time Compiler）直接把程序编译成与目标机器指令集相关的二进制代码的过程。下面笔者列举了这3类编译过程里一些比较有代表性的编译器产品：

+ 前端编译器：JDK的Javac、Eclipse JDT中的增量式编译器（ECJ）
+ 即时编译器：HotSpot虚拟机的C1、C2编译器，Graal编译器
+ 提前编译器：JDK的Jaotc、GNU Compiler for the Java（GCJ）、Excelsior JET。

​	这3类过程中最符合普通程序员对Java程序编译认知的应该是第一类，本章标题中的“前端”指的也是这种由前端编译器完成的编译行为。在本章后续的讨论里，笔者提到的全部“编译期”和“编译器”都仅限于第一类编译过程，我们会把第二、三类编译过程留到第11章中去讨论。限制了“编译期”的范围后，我们对于“优化”二字的定义也需要放宽一些，因为Javac这类前端编译器对代码的运行效率几乎没有任何优化措施可言（在JDK 1.3之后，Javac的-O优化参数就不再有意义），哪怕是编译器真的采取了优化措施也不会产生什么实质的效果。因为<u>Java虚拟机设计团队选择把对性能的优化全部集中到运行期的即时编译器中，这样可以让那些不是由Javac产生的Class文件（如JRuby、Groovy等语言的Class文件）也同样能享受到编译器优化措施所带来的性能红利</u>。

## 10.2 Javac编译器

​	分析源码是了解一项技术的实现内幕最彻底的手段，Javac编译器不像HotSpot虚拟机那样使用C++语言（包含少量C语言）实现，它本身就是一个由Java语言编写的程序，这为纯Java的程序员了解它的编译过程带来了很大的便利。

### 10.2.1 Javac的源码与调试

​	从Javac代码的总体结构来看，编译过程大致可以分为1个准备过程和3个处理过程，它们分别如下所示。

1）准备过程：初始化插入式注解处理器。

2）解析与填充符号表过程，包括：

+ 词法、语法分析。将源代码的字符流转变为标记集合，构造出抽象语法树。

+ 填充符号表。产生符号地址和符号信息。

3）插入式注解处理器的注解处理过程：插入式注解处理器的执行阶段，本章的实战部分会设计一个插入式注解处理器来影响Javac的编译行为。

4）分析与字节码生成过程，包括：

+ 标注检查。对语法的静态信息进行检查。

+ 数据流及控制流分析。对程序动态运行过程进行检查。

+ 解语法糖。将简化代码编写的语法糖还原为原有的形式。

+ 字节码生成。将前面各个步骤所生成的信息转化成字节码。

​	上述3个处理过程里，执行插入式注解时又可能会产生新的符号，如果有新的符号产生，就必须转回到之前的解析、填充符号表的过程中重新处理这些新符号，从总体来看，三者之间的关系与交互顺序如图10-4所示。

![img](https://img-blog.csdnimg.cn/653010de7534436eb7a80b1b01a6a303.png)

​		图10-4　Javac的编译过程

​	我们可以把上述处理过程对应到代码中，Javac编译动作的入口是com.sun.tools.javac.main.JavaCompiler类，上述3个过程的代码逻辑集中在这个类的compile()和compile2()方法里，其中主体代码如图10-5所示，整个编译过程主要的处理由图中标注的8个方法来完成。

![img](https://img-blog.csdnimg.cn/4889616ce44c4beeb83a61e4bd9e55e5.png)

​	图10-5　Javac编译过程的主体代码

​	接下来，我们将对照Javac的源代码，逐项讲解上述过程。

### 10.2.2 解析与填充符号表

​	解析过程由图10-5中的parseFiles()方法（图10-5中的过程1.1）来完成，解析过程包括了经典程序编译原理中的词法分析和语法分析两个步骤。

#### 1. 词法、语法分析

​	词法分析是将源代码的字符流转变为标记（Token）集合的过程，单个字符是程序编写时的最小元素，但标记才是编译时的最小元素。关键字、变量名、字面量、运算符都可以作为标记，如“inta=b+2”这句代码中就包含了6个标记，分别是int、a、=、b、+、2，虽然关键字int由3个字符构成，但是它只是一个独立的标记，不可以再拆分。**在Javac的源码中，词法分析过程由com.sun.tools.javac.parser.Scanner类来实现**。

​	**语法分析是根据标记序列构造抽象语法树的过程，抽象语法树（Abstract Syntax Tree，AST）是一种用来描述程序代码语法结构的树形表示方式，抽象语法树的每一个节点都代表着程序代码中的一个语法结构（Syntax Construct），例如包、类型、修饰符、运算符、接口、返回值甚至连代码注释等都可以是一种特定的语法结构**。

​	<u>经过词法和语法分析生成语法树以后，编译器就不会再对源码字符流进行操作了，后续的操作都建立在抽象语法树之上</u>。

#### 2. 填充符号表

​	完成了语法分析和词法分析之后，下一个阶段是对符号表进行填充的过程，也就是图10-5中enterTrees()方法（图10-5中注释的过程1.2）要做的事情。符号表（Symbol Table）是由一组符号地址和符号信息构成的数据结构，读者可以把它类比想象成哈希表中键值对的存储形式（实际上符号表不一定是哈希表实现，可以是有序符号表、树状符号表、栈结构符号表等各种形式）。<u>符号表中所登记的信息在编译的不同阶段都要被用到</u>。譬如在语义分析的过程中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的声明是否一致）和产生中间代码，在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的直接依据。

​	在Javac源代码中，填充符号表的过程由com.sun.tools.javac.comp.Enter类实现，该过程的产出物是一个待处理列表，其中包含了每一个编译单元的抽象语法树的顶级节点，以及package-info.java（如果存在的话）的顶级节点。

### 10.2.3 注解处理器

> [JVM系列之：你知道Lombok是如何工作的吗 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/485004057)
>
> [从JSR269到Lombok，学习注解处理器Annotation Processor Tool（附Demo） - 掘金 (juejin.cn)](https://juejin.cn/post/6960544470635347999#heading-11)

​	JDK 5之后，Java语言提供了对注解（Annotations）的支持，注解在设计上原本是与普通的Java代码一样，都只会在程序运行期间发挥作用的。但<u>在JDK 6中又提出并通过了JSR-269提案，该提案设计了一组被称为“插入式注解处理器”的标准API，可以提前至编译期对代码中的特定注解进行处理，从而影响到前端编译器的工作过程。我们可以把插入式注解处理器看作是一组编译器的插件，当这些插件工作时，允许读取、修改、添加抽象语法树中的任意元素。如果这些插件在处理注解期间对语法树进行过修改，编译器将回到解析及填充符号表的过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止，每一次循环过程称为一个轮次（Round），这也就对应着图10-4的那个回环过程</u>。

​	有了编译器注解处理的标准API后，程序员的代码才有可能干涉编译器的行为，由于语法树中的任意元素，甚至包括代码注释都可以在插件中被访问到，所以通过插入式注解处理器实现的插件在功能上有很大的发挥空间。只要有足够的创意，程序员能使用插入式注解处理器来实现许多原本只能在编码中由人工完成的事情。**譬如Java著名的编码效率工具Lombok，它可以通过注解来实现自动产生getter/setter方法、进行空置检查、生成受查异常表、产生equals()和hashCode()方法，等等，帮助开发人员消除Java的冗长代码，这些都是依赖插入式注解处理器来实现的，本章最后会设计一个如何使用插入式注解处理器的简单实战**。

​	<u>在Javac源码中，插入式注解处理器的初始化过程是在initPorcessAnnotations()方法中完成的，而它的执行过程则是在processAnnotations()方法中完成。这个方法会判断是否还有新的注解处理器需要执行，如果有的话，通过com.sun.tools.javac.processing.JavacProcessing-Environment类的doProcessing()方法来生成一个新的JavaCompiler对象，对编译的后续步骤进行处理</u>。

### 10.2.4 语义分析与字节码生成

​	<u>经过语法分析之后，编译器获得了程序代码的抽象语法树表示，抽象语法树能够表示一个结构正确的源程序，但无法保证源程序的语义是符合逻辑的</u>。而语义分析的主要任务则是对结构上正确的源程序进行上下文相关性质的检查，譬如进行类型检查、控制流检查、数据流检查，等等。举个简单的例子，假设有如下3个变量定义语句：

```java
int a = 1;
boolean b = false;
char c = 2;
```

​	后续可能出现的赋值运算：

```java
int d = a + c;
int d = b + c;
char d = a + c;
```

​	后续代码中如果出现了如上3种赋值运算的话，那它们都能构成结构正确的抽象语法树，但是只有第一种的写法在语义上是没有错误的，能够通过检查和编译。其余两种在Java语言中是不合逻辑的，无法编译（是否合乎语义逻辑必须限定在具体的语言与具体的上下文环境之中才有意义。如在C语言中，a、b、c的上下文定义不变，第二、三种写法都是可以被正确编译的）。**我们编码时经常能在IDE中看到由红线标注的错误提示，其中绝大部分都是来源于语义分析阶段的检查结果**。

#### 1. 标注检查
​	Javac在编译过程中，语义分析过程可分为**标注检查**和**数据及控制流分析**两个步骤，分别由图10-5的attribute()和flow()方法（分别对应图10-5中的过程3.1和过程3.2）完成。

​	<u>标注检查步骤要检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配，等等</u>，刚才3个变量定义的例子就属于标注检查的处理范畴。在标注检查中，还会顺便进行一个称为**常量折叠（Constant Folding）**的代码优化，这是Javac编译器会对源代码做的极少量优化措施之一（<u>代码优化几乎都在即时编译器中进行</u>）。如果我们在Java代码中写下如下所示的变量定义：

```java
int a = 1 + 2;
```

​	则在抽象语法树上仍然能看到字面量“1”“2”和操作符“+”号，但是在经过常量折叠优化之后，它们将会被折叠为字面量“3”，如图10-7所示，这个<u>插入式表达式（Infix Expression）</u>的值已经在语法树上标注出来了（ConstantExpressionValue：3）。<u>由于编译期间进行了常量折叠，所以在代码里面定义“a=1+2”比起直接定义“a=3”来，并不会增加程序运行期哪怕仅仅一个处理器时钟周期的处理工作量</u>。

​	标注检查步骤在Javac源码中的实现类是com.sun.tools.javac.comp.Attr类和com.sun.tools.javac.comp.Check类。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fe089f3d0e384ee7b19f5a6c2873b463.png)

#### 2. 数据及控制流分析

​	**数据流分析和控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题**。<u>编译时期的数据及控制流分析与类加载时的数据及控制流分析的目的基本上可以看作是一致的，但校验范围会有所区别，有一些校验项只有在编译期或运行期才能进行</u>。下面举一个关于final修饰符的数据及控制流分析的例子，见代码清单10-1所示。

​	代码清单10-1　final语义校验

```java
// 方法一带有final修饰
public void foo(final int arg) {
  final int var = 0;
  // do something
}
// 方法二没有final修饰
public void foo(int arg) {
  int var = 0;
  // do something
}
```

​	在这两个foo()方法中，一个方法的参数和局部变量定义使用了final修饰符，另外一个则没有，在代码编写时程序肯定会受到final修饰符的影响，不能再改变arg和var变量的值，但是<u>如果观察这两段代码编译出来的字节码，会发现它们是没有任何一点区别的，每条指令，甚至每个字节都一模一样</u>。通过第6章对Class文件结构的讲解我们已经知道，局部变量与类的字段（实例变量、类变量）的存储是有显著差别的，**<u>局部变量在常量池中并没有CONSTANT_Fieldref_info的符号引用，自然就不可能存储有访问标志（access_flags）的信息，甚至可能连变量名称都不一定会被保留下来（这取决于编译时的编译器的参数选项），自然在Class文件中就不可能知道一个局部变量是不是被声明为final了。因此，可以肯定地推断出把局部变量声明为final，对运行期是完全没有影响的，变量的不变性仅仅由Javac编译器在编译期间来保障，这就是一个只能在编译期而不能在运行期中检查的例子</u>**。在Javac的源码中，数据及控制流分析的入口是图10-5中的flow()方法（图10-5中的过程3.2），具体操作由com.sun.tools.javac.comp.Flow类来完成。

#### 3. 解语法糖

​	语法糖（Syntactic Sugar），也称糖衣语法，是由英国计算机科学家Peter J.Landin发明的一种编程术语，指的是在计算机语言中添加的某种语法，这种语法对语言的编译结果和功能并没有实际影响，但是却能更方便程序员使用该语言。通常来说使用语法糖能够减少代码量、增加程序的可读性，从而减少程序代码出错的机会。

​	Java在现代编程语言之中已经属于“低糖语言”（相对于C#及许多其他Java虚拟机语言来说），尤其是JDK 5之前的Java。“低糖”的语法让Java程序实现相同功能的代码量往往高于其他语言，通俗地说就是会显得比较“啰嗦”，这也是Java语言一直被质疑是否已经“落后”了的一个浮于表面的理由。

​	Java中最常见的语法糖包括了前面提到过的泛型（其他语言中泛型并不一定都是语法糖实现，如C#的泛型就是直接由CLR支持的）、变长参数、自动装箱拆箱，等等，**Java虚拟机运行时并不直接支持这些语法，它们在编译阶段被还原回原始的基础语法结构，这个过程就称为解语法糖**。Java的这些语法糖是如何实现的、被分解后会是什么样子，都将在10.3节中详细讲述。

​	在Javac的源码中，解语法糖的过程由desugar()方法触发，在com.sun.tools.javac.comp.TransTypes类和com.sun.tools.javac.comp.Lower类中完成。

#### 4. 字节码生成

​	**字节码生成是Javac编译过程的最后一个阶段，在Javac源码里面由com.sun.tools.javac.jvm.Gen类来完成**。<u>字节码生成阶段不仅仅是把前面各个步骤所生成的信息（语法树、符号表）转化成字节码指令写到磁盘中，编译器还进行了少量的代码添加和转换工作</u>。

​	**例如前文多次登场的实例构造器`<init>()`方法和类构造器`<clinit>()`方法就是在这个阶段被添加到语法树之中的**。请注意这里的实例构造器并不等同于默认构造函数，如果用户代码中没有提供任何构造函数，那编译器将会添加一个没有参数的、可访问性（public、protected、private或`<package>`）与当前类型一致的默认构造函数，这个工作在填充符号表阶段中就已经完成。**`<init>()`和`<clinit>()`这两个构造器的产生实际上是一种代码收敛的过程，编译器会把语句块（对于实例构造器而言是“{}”块，对于类构造器而言是“static{}”块）、变量初始化（实例变量和类变量）、调用父类的实例构造器（仅仅是实例构造器，`<clinit>()`方法中无须调用父类的`<clinit>()`方法，Java虚拟机会自动保证父类构造器的正确执行，但在`<clinit>()`方法中经常会生成调用java.lang.Object的`<init>()`方法的代码）等操作收敛到`<init>()`和`<clinit>()`方法之中，并且保证无论源码中出现的顺序如何，都一定是按先执行父类的实例构造器，然后初始化变量，最后执行语句块的顺序进行，上面所述的动作由Gen::normalizeDefs()方法来实现**。<u>除了生成构造器以外，还有其他的一些代码替换工作用于优化程序某些逻辑的实现方式，如把字符串的加操作替换为StringBuffer或StringBuilder（取决于目标代码的版本是否大于或等于JDK 5）的append()操作，等等</u>。

​	完成了对语法树的遍历和调整之后，就会把填充了所有所需信息的符号表交到com.sun.tools.javac.jvm.ClassWriter类手上，由这个类的writeClass()方法输出字节码，生成最终的Class文件，到此，整个编译过程宣告结束。

## 10.3 Java语法糖的味道

​	几乎所有的编程语言都或多或少提供过一些语法糖来方便程序员的代码开发，这些语法糖虽然不会提供实质性的功能改进，但是它们或能提高效率，或能提升语法的严谨性，或能减少编码出错的机会。现在也有一种观点认为语法糖并不一定都是有益的，大量添加和使用含糖的语法，容易让程序员产生依赖，无法看清语法糖的糖衣背后，程序代码的真实面目。

​	总而言之，语法糖可以看作是前端编译器实现的一些“小把戏”，这些“小把戏”可能会使效率得到“大提升”，但我们也应该去了解这些“小把戏”背后的真实面貌，那样才能利用好它们，而不是被它们所迷惑。

### 10.3.1 泛型

​	泛型的本质**是参数化类型（Parameterized Type）**或者**参数化多态（Parametric Polymorphism）**的应用，即可以将操作的数据类型指定为方法签名中的一种特殊参数，这种参数类型能够用在类、接口和方法的创建中，分别构成泛型类、泛型接口和泛型方法。泛型让程序员能够针对泛化的数据类型编写相同的算法，这极大地增强了编程语言的类型系统及抽象能力。

#### 1. Java与C#的泛型

​	**Java选择的泛型实现方式叫作“类型擦除式泛型”（Type Erasure Generics），而C#选择的泛型实现方式是“具现化式泛型”（Reified Generics）**。具现化和特化、偏特化这些名词最初都是源于C++模版语法中的概念，如果读者本身不使用C++的话，在本节的阅读中可不必太纠结其概念定义，把它当一个技术名词即可，只需要知道<u>C#里面泛型无论在程序源码里面、编译后的中间语言表示（IntermediateLanguage，这时候泛型是一个占位符）里面，抑或是运行期的CLR里面都是切实存在的，`List<int>`与`List<string>`就是两个不同的类型，它们由系统在运行期生成，有着自己独立的虚方法表和类型数据</u>。而**Java语言中的泛型则不同，它只在程序源码中存在，在编译后的字节码文件中，全部泛型都被替换为原来的裸类型（Raw Type，稍后我们会讲解裸类型具体是什么）了，并且在相应的地方插入了强制转型代码**，因此对于运行期的Java语言来说，`ArrayList<int>`与`ArrayList<String>`其实是同一个类型，由此读者可以想象“类型擦除”这个名字的含义和来源，这也是为什么笔者会把Java泛型安排在语法糖里介绍的原因。

​	代码清单10-2　Java中不支持的泛型用法

```java
public class TypeErasureGenerics<E> {
  public void doSomething(Object item) {
    if (item instanceof E) { // 不合法，无法对泛型进行实例判断
      ...
    }
    E newItem = new E(); // 不合法，无法使用泛型创建对象
    E[] itemArray = new E[10]; // 不合法，无法使用泛型创建数组
  }
}
```

​	上面这些是Java泛型在编码阶段产生的不良影响，如果说这种使用层次上的差别还可以通过多写几行代码、方法中多加一两个类型参数来解决的话，性能上的差距则是难以用编码弥补的。C#2.0引入了泛型之后，带来的显著优势之一便是对比起Java在执行性能上的提高，因为在使用平台提供的容器类型（如`List<T>`，`Dictionary<TKey，TValue>`）时，无须像Java里那样不厌其烦地**拆箱和装箱**，如果在Java中要避免这种损失，就必须构造一个与数据类型相关的容器类（譬如IntFloatHashMap这样的容器）。显然，这除了引入更多代码造成复杂度提高、复用性降低之外，更是丧失了泛型本身的存在价值。

​	**Java的类型擦除式泛型无论在使用效果上还是运行效率上，几乎是全面落后于C#的具现化式泛型，而它的唯一优势是在于实现这种泛型的影响范围上：擦除式泛型的实现几乎只需要在Javac编译器上做出改进即可，不需要改动字节码、不需要改动Java虚拟机，也保证了以前没有使用泛型的库可以直接运行在Java 5.0之上**。但这种听起来节省工作量甚至可以说是有偷工减料嫌疑的优势就显得非常短视，真的能在当年Java实现泛型的利弊权衡中胜出吗？答案的确是它胜出了，但我们必须在那时的泛型历史背景中去考虑不同实现方式带来的代价。

#### 2. 泛型的历史背景

​	泛型思想早在C++语言的模板（Template）功能中就开始生根发芽，而在Java语言中加入泛型的首次尝试是出现在1996年。Martin Odersky（后来Scala语言的缔造者）当时是德国卡尔斯鲁厄大学编程理论的教授，他想设计一门能够支持函数式编程的程序语言，又不想从头把编程语言的所有功能都再做一遍，所以就注意到了刚刚发布一年的Java，并在它上面实现了函数式编程的3大特性：泛型、高阶函数和模式匹配，形成了Scala语言的前身Pizza语言。后来，Java的开发团队找到了MartinOdersky，表示对Pizza语言的泛型功能很感兴趣，他们就一起建立了一个叫作“Generic Java”的新项目，目标是把Pizza语言的泛型单独拎出来移植到Java语言上，其最终成果就是Java 5.0中的那个泛型实现，但是移植的过程并不是一开始就朝着类型擦除式泛型去的，事实上Pizza语言中的泛型更接近于现在C#的泛型。Martin Odersky自己在采访自述中提到，进行Generic Java项目的过程中他受到了重重约束，甚至多次让他感到沮丧，最紧、最难的约束来源于被迫要完全向后兼容无泛型Java，即保证“**二进制向后兼容性**”（Binary Backwards Compatibility）。<u>二进制向后兼容性是明确写入《Java语言规范》中的对Java使用者的严肃承诺，譬如一个在JDK 1.2中编译出来的Class文件，必须保证能够在JDK 12乃至以后的版本中也能够正常运行</u>。这样，既然Java到1.4.2版之前都没有支持过泛型，而到Java 5.0突然要支持泛型了，还要让以前编译的程序在新版本的虚拟机还能正常运行，就意味着以前没有的限制不能突然间冒出来。

​	举个例子，在没有泛型的时代，由于Java中的数组是支持**协变（Covariant）**的，对应的集合类也可以存入不同类型的元素，类似于代码清单10-3这样的代码尽管不提倡，但是完全可以正常编译成Class文件。

​	代码清单10-3　以下代码可正常编译为Class

```java
Object[] array = new String[10];
array[0] = 10; // 编译期不会有问题，运行时会报错
ArrayList things = new ArrayList();
things.add(Integer.valueOf(10)); //编译、运行时都不会报错
things.add("hello world");
```

​	为了保证这些编译出来的Class文件可以在Java 5.0引入泛型之后继续运行，设计者面前大体上有两条路可以选择：

1）需要泛型化的类型（主要是容器类型），以前有的就保持不变，然后平行地加一套泛型化版本的新类型。

2）**直接把已有的类型泛型化，即让所有需要泛型化的已有类型都原地泛型化，不添加任何平行于已有类型的泛型版**。

​	在这个分叉路口，C#走了第一条路，添加了一组System.Collections.Generic的新容器，以前的System.Collections以及System.Collections.Specialized容器类型继续存在。C#的开发人员很快就接受了新的容器，倒也没出现过什么不适应的问题，唯一的不适大概是许多.NET自身的标准库已经把老容器类型当作方法的返回值或者参数使用，这些方法至今还保持着原来的老样子。

​	但如果相同的选择出现在Java中就很可能不会是相同的结果了，要知道当时.NET才问世两年，而Java已经有快十年的历史了，再加上各自流行程度的不同，两者遗留代码的规模根本不在同一个数量级上。而且更大的问题是Java并不是没有做过第一条路那样的技术决策，在JDK 1.2时，遗留代码规模尚小，Java就引入过新的集合类，并且保留了旧集合类不动。这导致了直到现在标准类库中还有Vector（老）和ArrayList（新）、有Hashtable（老）和HashMap（新）等两套容器代码并存，如果当时再摆弄出像Vector（老）、ArrayList（新）、`Vector<T>`（老但有泛型）、`ArrayList<T>`（新且有泛型）这样的容器集合，可能叫骂声会比今天听到的更响更大。

​	到了这里，相信读者已经能稍微理解为什么当时Java只能选择第二条路了。但第二条路也并不意味着一定只能使用类型擦除来实现，如果当时有足够的时间好好设计和实现，是完全有可能做出更好的泛型系统的，否则也不会有今天的Valhalla项目来还以前泛型偷懒留下的技术债了。下面我们就来看看当时做的类型擦除式泛型的实现时到底哪里偷懒了，又带来了怎样的缺陷。

#### 3. 类型擦除

​	我们继续以ArrayList为例来介绍Java泛型的类型擦除具体是如何实现的。由于Java选择了第二条路，直接把已有的类型泛型化。要让所有需要泛型化的已有类型，譬如ArrayList，原地泛型化后变成了`ArrayList<T>`，而且保证以前直接用ArrayList的代码在泛型新版本里必须还能继续用这同一个容器，这就<u>必须让所有泛型化的实例类型，譬如`ArrayList<Integer>`、`ArrayList<String>`这些全部自动成为ArrayList的子类型才能可以，否则类型转换就是不安全的</u>。由此就引出了**“裸类型”（Raw Type）的概念，裸类型应被视为所有该类型泛型化实例的共同父类型（Super Type）**，只有这样，像代码清单10-4中的赋值才是被系统允许的从子类到父类的安全转型。

​	代码清单10-4　裸类型赋值

```java
ArrayList<Integer> ilist = new ArrayList<Integer>();
ArrayList<String> slist = new ArrayList<String>();
ArrayList list; // 裸类型
list = ilist;
list = slist;
```

​	接下来的问题是该如何实现裸类型。这里又有了两种选择：一种是在运行期由Java虚拟机来自动地、真实地构造出`ArrayList<Integer>`这样的类型，并且自动实现从`ArrayList<Integer>`派生自ArrayList的继承关系来满足裸类型的定义；另外一种是索性简单粗暴地直接在编译时把`ArrayList<Integer>`还原回ArrayList，只在元素访问、修改时自动插入一些强制类型转换和检查指令，这样看起来也是能满足需要，这两个选择的最终结果大家已经都知道了。代码清单10-5是一段简单的Java泛型例子，我们可以看一下它编译后的实际样子是怎样的。

​	代码清单10-5　泛型擦除前的例子

```java
public static void main(String[] args) {
  Map<String, String> map = new HashMap<String, String>();
  map.put("hello", "你好");
  map.put("how are you?", "吃了没？");
  System.out.println(map.get("hello"));
  System.out.println(map.get("how are you?"));
}
```

​	把这段Java代码编译成Class文件，然后再用字节码反编译工具进行反编译后，将会发现泛型都不见了，程序又变回了Java泛型出现之前的写法，泛型类型都变回了裸类型，只在元素访问时插入了从Object到String的强制转型代码，如代码清单10-6所示。

​	代码清单10-6　泛型擦除后的例子

```java
public static void main(String[] args) {
  Map map = new HashMap(); // 我这里jdk11反编译这一行和书本不同，是 Map<String, String> map = new HashMap();
  map.put("hello", "你好");
  map.put("how are you?", "吃了没？");
  System.out.println((String) map.get("hello"));
  System.out.println((String) map.get("how are you?"));
}
```

​	类型擦除带来的缺陷前面已经提到过一些，为了系统性地讲述，笔者在此再举3个例子，把前面与C#对比时简要提及的擦除式泛型的缺陷做更具体的说明。

​	首先，使用擦除法实现泛型直接导致了对原始类型（Primitive Types）数据的支持又成了新的麻烦，譬如将代码清单10-2稍微修改一下，变成代码清单10-7这个样子。

​	代码清单10-7　原始类型的泛型（目前的Java不支持）

```java
ArrayList<int> ilist = new ArrayList<int>();
ArrayList<long> llist = new ArrayList<long>();
ArrayList list;
list = ilist;
list = llist;
```

​	这种情况下，一旦把泛型信息擦除后，到要插入强制转型代码的地方就没办法往下做了，<u>因为不支持int、long与Object之间的强制转型。当时Java给出的解决方案一如既往的简单粗暴：既然没法转换那就索性别支持原生类型的泛型了吧，你们都用`ArrayList<Integer>`、`ArrayList<Long>`，反正都做了自动的强制类型转换，遇到原生类型时把装箱、拆箱也自动做了得了</u>。这个决定后面导致了<u>无数构造包装类和装箱、拆箱的开销，成为Java泛型慢的重要原因</u>，也成为今天Valhalla项目要重点解决的问题之一。

​	第二，运行期无法取到泛型类型信息，会让一些代码变得相当啰嗦，譬如代码清单10-2中罗列的几种Java不支持的泛型用法，都是由于运行期Java虚拟机无法取得泛型类型而导致的。像代码清单10-8这样，我们去写一个泛型版本的从List到数组的转换方法，**由于不能从List中取得参数化类型T，所以不得不从一个额外参数中再传入一个数组的组件类型进去**，实属无奈。

​	代码清单10-8　不得不加入的类型参数

```java
public static <T> T[] convert(List<T> list, Class<T> componentType) {
  T[] array = (T[])Array.newInstance(componentType, list.size());
  ...
}
```

​	最后，笔者认为通过擦除法来实现泛型，还丧失了一些面向对象思想应有的优雅，带来了一些模棱两可的模糊状况，例如代码清单10-9的例子。

​	代码清单10-9　当泛型遇见重载1

```java
public class GenericTypes {
  public static void method(List<String> list) {
    System.out.println("invoke method(List<String> list)");
  }
  public static void method(List<Integer> list) {
    System.out.println("invoke method(List<Integer> list)");
  }
}
```

​	请读者思考一下，上面这段代码是否正确，能否编译执行？也许你已经有了答案，这段代码是不能被编译的，因为参数`List<Integer>`和`List<String>`编译之后都被擦除了，变成了同一种的裸类型List，**类型擦除导致这两个方法的特征签名变得一模一样**。初步看来，无法重载的原因已经找到了，但是真的就是如此吗？其实这个例子中泛型擦除成相同的裸类型只是无法重载的其中一部分原因，请再接着看一看代码清单10-10中的内容。

​	代码清单10-10　当泛型遇见重载2

```java
public class GenericTypes {
  public static String method(List<String> list) {
    System.out.println("invoke method(List<String> list)");
    return "";
  }
  public static int method(List<Integer> list) {
    System.out.println("invoke method(List<Integer> list)");
    return 1;
  }
  public static void main(String[] args) {
    method(new ArrayList<String>());
    method(new ArrayList<Integer>());
  }
}
```

​	执行结果：

```shell
# 我本地jdk11是编译不通过的 => method(java.util.List<java.lang.Integer>) and method(java.util.List<java.lang.String>) have the same erasure
invoke method(List<String> list)
invoke method(List<Integer> list)
```

​	代码清单10-9与代码清单10-10的差别，是两个method()方法添加了不同的返回值，由于这两个返回值的加入，方法重载居然成功了，即这段代码可以被编译和执行了。这是我们对Java语言中返回值不参与重载选择的基本认知的挑战吗？

​	代码清单10-10中的重载当然不是根据返回值来确定的，之所以这次能编译和执行成功，是因为两个method()方法加入了不同的返回值后才能共存在一个Class文件之中。**第6章介绍Class文件方法表（method_info）的数据结构时曾经提到过，方法重载要求方法具备不同的特征签名，返回值并不包含在方法的特征签名中，所以返回值不参与重载选择，但是在Class文件格式之中，只要描述符不是完全一致的两个方法就可以共存**。<u>也就是说两个方法如果有相同的名称和特征签名，但返回值不同，那它们也是可以合法地共存于一个Class文件中的</u>。

​	由于Java泛型的引入，各种场景（虚拟机解析、反射等）下的方法调用都可能对原有的基础产生影响并带来新的需求，如在泛型类中如何获取传入的参数化类型等。所以JCP组织对《Java虚拟机规范》做出了相应的修改，引入了诸如Signature、LocalVariableTypeTable等新的属性用于解决伴随泛型而来的参数类型的识别问题，**Signature是其中最重要的一项属性，它的作用就是存储一个方法在字节码层面的特征签名，这个属性中保存的参数类型并不是原生类型，而是包括了参数化类型的信息**。修改后的虚拟机规范要求所有能识别49.0以上版本的Class文件的虚拟机都要能正确地识别Signature参数。

​	*特征签名最重要的任务就是作为方法独一无二不可重复的ID，在Java代码中的方法特征签名只包括了方法名称、参数顺序及参数类型，而<u>在字节码中的特征签名还包括方法返回值及受查异常表</u>，本书中如果指的是字节码层面的方法签名，笔者会加入限定语进行说明，也请读者根据上下文语境注意区分。*

​	从上面的例子中可以看到擦除法对实际编码带来的不良影响，由于`List<String>`和`List<Integer>`擦除后是同一个类型，我们只能添加两个并不需要实际使用到的返回值才能完成重载，这是一种毫无优雅和美感可言的解决方案，并且存在一定语意上的混乱，譬如上面脚注中提到的，必须用JDK 6的Javac才能编译成功，其他版本或者是ECJ编译器都有可能拒绝编译。

​	**另外，从Signature属性的出现我们还可以得出结论，擦除法所谓的擦除，仅仅是对方法的Code属性中的字节码进行擦除，<u>实际上元数据中还是保留了泛型信息</u>，这也是我们在编码时能通过反射手段取得参数化类型的根本依据**。

#### 4. 值类型与未来的泛型

​	在2014年，刚好是Java泛型出现的十年之后，Oracle建立了一个名为Valhalla的语言改进项目，希望改进Java语言留下的各种缺陷（解决泛型的缺陷就是项目主要目标其中之一）。原本这个项目是计划在JDK 10中完成的，但在笔者撰写本节时（2019年8月，下个月JDK 13正式版都要发布了）也只有少部分目标（譬如VarHandle）顺利实现并发布出去。它现在的技术预览版LW2（L-World 2）是基于未完成的JDK 14 EarlyAccess来运行的，所以本节内容很可能在将来会发生变动，请读者阅读时多加注意。

​	在Valhalla项目中规划了几种不同的新泛型实现方案，被称为Model 1到Model 3，在这些新的泛型设计中，泛型类型有可能被具现化，也有可能继续维持类型擦除以保持兼容（取决于采用哪种实现方案），即使是继续采用类型擦除的方案，泛型的参数化类型也可以选择不被完全地擦除掉，而是相对完整地记录在Class文件中，能够在运行期被使用，也可以指定编译器默认要擦除哪些类型。相对于使用不同方式实现泛型，目前比较明确的是未来的Java应该会提供“值类型”（Value Type）的语言层面的支持。

​	说起值类型，这点也是C#用户攻讦Java语言的常用武器之一，C#并没有Java意义上的原生数据类型，在C#中使用的int、bool、double关键字其实是对应了一系列在.NET框架中预定义好的结构体（Struct），如Int32、Boolean、Double等。在C#中开发人员也可以定义自己值类型，只要继承于ValueType类型即可，而ValueType也是统一基类Object的子类，所以并不会遇到Java那样int不自动装箱就无法转型为Object的尴尬。

​	值类型可以与引用类型一样，具有构造函数、方法或是属性字段，等等，而它与引用类型的区别在于它在赋值的时候通常是整体复制，而不是像引用类型那样传递引用的。更为关键的是，值类型的实例很容易实现分配在方法的调用栈上的，这意味着值类型会随着当前方法的退出而自动释放，不会给垃圾收集子系统带来任何压力。

​	<u>在Valhalla项目中，Java的值类型方案被称为“内联类型”，计划通过一个新的关键字inline来定义，字节码层面也有专门与原生类型对应的以Q开头的新的操作码（譬如iload对应qload）来支撑</u>。现在的预览版可以通过一个特制的解释器来保证这些未来可能加入的字节码指令能够被执行，要即时编译的话，现在只支持C2编译器。即时编译器场景中是使用逃逸分析优化（见第11章）来处理内联类型的，通过编码时标注以及内联类实例所具备的不可变性，可以很好地解决逃逸分析面对传统引用类型时难以判断（没有足够的信息，或者没有足够的时间做全程序分析）对象是否逃逸的问题。

### 10.3.2 自动装箱、拆箱与遍历循环

​	就纯技术的角度而论，自动装箱、自动拆箱与遍历循环（for-each循环）这些语法糖，无论是实现复杂度上还是其中蕴含的思想上都不能和10.3.1节介绍的泛型相提并论，两者涉及的难度和深度都有很大差距。专门拿出一节来讲解它们只是因为这些是Java语言里面被使用最多的语法糖。我们通过代码清单10-11和代码清单10-12中所示的代码来看看这些语法糖在编译后会发生什么样的变化。

​	代码清单10-11　自动装箱、拆箱与遍历循环

```java
public static void main(String[] args) {
  List<Integer> list = Arrays.asList(1, 2, 3, 4);
  int sum = 0;
  for (int i : list) {
    sum += i;
  }
  System.out.println(sum);
}
```

​	代码清单10-12　自动装箱、拆箱与遍历循环编译之后

```java
public static void main(String[] args) {
  List list = Arrays.asList( new Integer[] {
    Integer.valueOf(1),
    Integer.valueOf(2),
    Integer.valueOf(3),
    Integer.valueOf(4) });
  int sum = 0;
  for (Iterator localIterator = list.iterator(); localIterator.hasNext(); ) {
    int i = ((Integer)localIterator.next()).intValue();
    sum += i;
  }
  System.out.println(sum);
}
/**
// 我这边 jdk11编译后反编译结果如下
public class TestClass {
    public TestClass() {
    }

    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);
        int sum = 0;

        int i;
        for(Iterator var3 = list.iterator(); var3.hasNext(); sum += i) {
            i = (Integer)var3.next();
        }

        System.out.println(sum);
    }
}
*/
```

​	代码清单10-11中一共包含了泛型、自动装箱、自动拆箱、遍历循环与变长参数5种语法糖，代码清单10-12则展示了它们在编译前后发生的变化。泛型就不必说了，自动装箱、拆箱在编译之后被转化成了对应的包装和还原方法，如本例中的Integer.valueOf()与Integer.intValue()方法，而<u>遍历循环则是把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口的原因</u>。最后再看看**变长参数，它在调用的时候变成了一个数组类型的参数，在变长参数出现之前，程序员的确也就是使用数组来完成类似功能的**。

​	这些语法糖虽然看起来很简单，但也不见得就没有任何值得我们特别关注的地方，代码清单10-13演示了自动装箱的一些错误用法。

​	代码清单10-13　自动装箱的陷阱

```java
public static void main(String[] args) {
  Integer a = 1;
  Integer b = 2;
  Integer c = 3;
  Integer d = 3;
  Integer e = 321;
  Integer f = 321;
  Long g = 3L;
  System.out.println(c == d); // true
  System.out.println(e == f); // false
  System.out.println(c == (a + b)); // 我搞错了，原来是true
  System.out.println(c.equals(a + b)); // true
  System.out.println(g == (a + b)); // 我搞错了，原来是 true
  System.out.println(g.equals(a + b)); // 我搞错了，原来是 false
}
```

​	读者阅读完代码清单10-13，不妨思考两个问题：一是这6句打印语句的输出是什么？二是这6句打印语句中，解除语法糖后参数会是什么样子？这两个问题的答案都很容易试验出来，笔者就暂且略去答案，希望不能立刻做出判断的读者自己上机实践一下。**无论读者的回答是否正确，鉴于包装类的“==”运算在不遇到算术运算的情况下不会自动拆箱，以及它们equals()方法不处理数据转型的关系，笔者建议在实际编码中尽量避免这样使用自动装箱与拆箱**。

### 10.3.3 条件编译

​	*许多程序设计语言都提供了条件编译的途径，如C、C++中使用预处理器指示符（#ifdef）来完成条件编译。C、C++的预处理器最初的任务是解决编译时的代码依赖关系（如极为常用的#include预处理命令），而<u>在Java语言之中并没有使用预处理器，因为Java语言天然的编译方式（编译器并非一个个地编译Java文件，而是将所有编译单元的语法树顶级节点输入到待处理列表后再进行编译，因此各个文件之间能够互相提供符号信息）就无须使用到预处理器</u>。那Java语言是否有办法实现条件编译呢？*

​	**Java语言当然也可以进行条件编译，方法就是使用条件为常量的if语句**。如代码清单10-14所示，该代码中的if语句不同于其他Java代码，它在编译阶段就会被“运行”，生成的字节码之中只包括“System.out.println("block 1")；”一条语句，并不会包含if语句及另外一个分子中的“System.out.println("block 2")；”

​	代码清单10-14　Java语言的条件编译

```java
public static void main(String[] args) {
  if (true) {
    System.out.println("block 1");
  } else {
    System.out.println("block 2");
  }
}
```

​	该代码编译后Class文件的反编译结果：

```java
public static void main(String[] args) {
  System.out.println("block 1");
}
```

​	只能使用条件为常量的if语句才能达到上述效果，如果使用常量与其他带有条件判断能力的语句搭配，则可能在控制流分析中提示错误，被拒绝编译，如代码清单10-15所示的代码就会被编译器拒绝编译。

​	代码清单10-15　不能使用其他条件语句来完成条件编译

```java
public static void main(String[] args) {
  // 编译器将会提示“Unreachable code”
  while (false) {
    System.out.println("");
  }
}
```

​	**Java语言中条件编译的实现，也是Java语言的一颗语法糖，根据布尔常量值的真假，编译器将会把分支中不成立的代码块消除掉，这一工作将在编译器解除语法糖阶段（com.sun.tools.javac.comp.Lower类中）完成**。由于这种条件编译的实现方式使用了if语句，所以它必须遵循最基本的Java语法，只能写在方法体内部，因此它只能实现语句基本块（Block）级别的条件编译，而没有办法实现根据条件调整整个Java类的结构。

​	除了本节中介绍的泛型、自动装箱、自动拆箱、遍历循环、变长参数和条件编译之外，Java语言还有不少其他的语法糖，如内部类、枚举类、断言语句、数值字面量、对枚举和字符串的switch支持、try语句中定义和关闭资源（这3个从JDK 7开始支持）、Lambda表达式（从JDK 8开始支持，Lambda不能算是单纯的语法糖，但在前端编译器中做了大量的转换工作），等等，读者可以通过跟踪Javac源码、反编译Class文件等方式了解它们的本质实现，囿于篇幅，笔者就不再一一介绍了。

## 10.4 实战: 插入式注解处理器

​	但是笔者丝毫不认为相对于前两部分介绍的内存管理子系统和字节码执行子系统，编译子系统就不那么重要了。一套编程语言中编译子系统的优劣，很大程度上决定了程序运行性能的好坏和编码效率的高低，尤其在Java语言中，运行期即时编译与虚拟机执行子系统非常紧密地互相依赖、配合运作（第11章我们将主要讲解这方面的内容）。了解JDK如何编译和优化代码，有助于我们写出适合Java虚拟机自优化的程序。话题说远了，下面我们回到本章的实战中来，看看插入式注解处理器API能为我们实现什么功能。

### 10.4.1 实战目标

​	通过阅读Javac编译器的源码，我们知道前端编译器在把Java程序源码编译为字节码的时候，会对Java程序源码做各方面的检查校验。这些校验主要是以程序“写得对不对”为出发点，虽然也会产生一些警告和提示类的信息，但总体来讲还是较少去校验程序“写得好不好”。有鉴于此，业界出现了许多针对程序“写得好不好”的辅助校验工具，如CheckStyle、FindBug、Klocwork等。这些代码校验工具有一些是基于Java的源码进行校验，有一些是通过扫描字节码来完成，在本节的实战中，我们将会使用注解处理器API来编写一款拥有自己编码风格的校验工具：NameCheckProcessor。

​	当然，由于我们的实战都是为了学习和演示技术原理，而且篇幅所限，不可能做出一款能媲美CheckStyle等工具的产品来，所以NameCheckProcessor的目标也仅定为对Java程序命名进行检查。根据《Java语言规范》中6.8节的要求，Java程序命名推荐（而不是强制）应当符合下列格式的书写规范。

+ 类（或接口）：符合驼式命名法，首字母大写。

+ 方法：符合驼式命名法，首字母小写。

+ 字段：

  + 类或实例变量。符合驼式命名法，首字母小写。

  + 常量。要求全部由大写字母或下划线构成，并且第一个字符不能是下划线。

​	上文提到的驼式命名法（Camel Case Name），正如它的名称所表示的那样，是指混合使用大小写字母来分割构成变量或函数的名字，犹如驼峰一般，这是当前Java语言中主流的命名规范，我们的实战目标就是为Javac编译器添加一个额外的功能，在编译程序时检查程序名是否符合上述对类（或接口）、方法、字段的命名要求。

### 10.4.2 代码实现

​	要通过注解处理器API实现一个编译器插件，首先需要了解这组API的一些基本知识。我们实现注解处理器的代码需要继承抽象类javax.annotation.processing.AbstractProcessor，这个抽象类中只有一个子类必须实现的抽象方法：“process()”，它是Javac编译器在执行注解处理器代码时要调用的过程，我们可以从这个方法的第一个参数“annotations”中获取到此注解处理器所要处理的注解集合，从第二个参数“roundEnv”中访问到当前这个轮次（Round）中的抽象语法树节点，**每个语法树节点在这里都表示为一个Element。在javax.lang.model.ElementKind中定义了18类Element，已经包括了Java代码中可能出现的全部元素，如：“包（PACKAGE）、枚举（ENUM）、类（CLASS）、注解（ANNOTATION_TYPE）、接口（INTERFACE）、枚举值（ENUM_CONSTANT）、字段（FIELD）、参数（PARAMETER）、本地变量（LOCAL_VARIABLE）、异常（EXCEPTION_PARAMETER）、方法（METHOD）、构造函数（CONSTRUCTOR）、静态语句块（STATIC_INIT，即static{}块）、实例语句块（INSTANCE_INIT，即{}块）、参数化类型（TYPE_PARAMETER，泛型尖括号内的类型）、资源变量（RESOURCE_VARIABLE，try-resource中定义的变量）、模块（MODULE）和未定义的其他语法树节点（OTHER）**”。除了process()方法的传入参数之外，还有一个很重要的实例变量“**processingEnv**”，它是AbstractProcessor中的一个protected变量，在注解处理器初始化的时候（init()方法执行的时候）创建，继承了AbstractProcessor的注解处理器代码可以直接访问它。**它代表了注解处理器框架提供的一个上下文环境，要创建新的代码、向编译器输出信息、获取其他工具类等都需要用到这个实例变量**。

​	<u>注解处理器除了process()方法及其参数之外，还有两个经常配合着使用的注解，分别是：@SupportedAnnotationTypes和@SupportedSourceVersion，前者代表了这个注解处理器对哪些注解感兴趣，可以使用星号“*”作为通配符代表对所有的注解都感兴趣，后者指出这个注解处理器可以处理哪些版本的Java代码</u>。

​	**每一个注解处理器在运行时都是单例的**，如果不需要改变或添加抽象语法树中的内容，process()方法就可以返回一个值为false的布尔值，通知编译器这个轮次中的代码未发生变化，无须构造新的JavaCompiler实例，在这次实战的注解处理器中只对程序命名进行检查，不需要改变语法树的内容，因此process()方法的返回值一律都是false。

​	关于注解处理器的API，笔者就简单介绍这些，对这个领域有兴趣的读者可以阅读相关的帮助文档。我们来看看注解处理器NameCheckProcessor的具体代码，如代码清单10-16所示。

​	代码清单10-16　注解处理器NameCheckProcessor

```java
// 可以用"*"表示支持所有Annotations
@SupportedAnnotationTypes("*")
// 只支持JDK 6的Java代码
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class NameCheckProcessor extends AbstractProcessor {
  private NameChecker nameChecker;
  /**
* 初始化名称检查插件
*/
  @Override
  public void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    nameChecker = new NameChecker(processingEnv);
  }
  /**
* 对输入的语法树的各个节点进行名称检查
*/
  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    if (!roundEnv.processingOver()) {
      for (Element element : roundEnv.getRootElements())
        nameChecker.checkNames(element);
    }
    return false;
  }
}
```

​	从代码清单10-16中可以看到NameCheckProcessor能处理基于JDK 6的源码，它不限于特定的注解，对任何代码都“感兴趣”，而在process()方法中是把当前轮次中的每一个RootElement传递到一个名为NameChecker的检查器中执行名称检查逻辑，NameChecker的代码如代码清单10-17所示。

​	代码清单10-17　命名检查器NameChecker

```java
import static javax.tools.Diagnostic.Kind.*;
import static javax.lang.model.element.ElementKind.*;
import static javax.lang.model.element.Modifier.*;

import java.util.EnumSet;
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.lang.model.element.Element;
import javax.lang.model.element.ExecutableElement;
import javax.lang.model.element.Name;
import javax.lang.model.element.TypeElement;
import javax.lang.model.element.VariableElement;
import javax.lang.model.util.ElementScanner6;

/**
 * 程序名称规范的编译器插件：<br>
 * 如果程序命名不合规范，将会输出一个编译器的 WARNING 信息
 *
 */
public class NameChecker {

  private final Messager messager;

  NameCheckScanner nameCheckScanner = new NameCheckScanner();

  public NameChecker(ProcessingEnvironment processingEnv) {
    this.messager = processingEnv.getMessager();
  }

  /**
	 * 对 Java 程序命名进行检查, 根据《Java 语言规范（第 3 版）》第 6.8 节的要求，Java 程序命名
	 * 应当符合下列格式：
	 * 
	 * <ul>
	 * <li>类或接口：符合驼式命名法，首字母大写。
	 * <li>方法：符合驼式命名法，首字母小写。
	 * <li>字段：
	 * <ul>
	 * <li>类、实例变量：符合驼式命名法，首字母小写。
	 * <li>常量：要求全部大写。
	 * </ul>
	 * </ul>
	 */
  public void checkNames(Element element) {
    nameCheckScanner.scan(element);
  }

  /**
	 * 名称检查器实现类，继承了 JDK 1.6 中新提供的 ElementScanner6<br>
	 * 将会以 Visitor 模式访问抽象语法树中的元素
	 *
	 */
  private class NameCheckScanner extends ElementScanner6<Void, Void> {

    /**
		 * 此方法用于检查 Java 类
		 */
    @Override
    public Void visitType(TypeElement e, Void p) {
      scan(e.getTypeParameters(), p);
      checkCamelCase(e, true);
      super.visitType(e, p);
      return null;
    }

    /**
		 * 检查方法命名是否合法
		 */
    @Override
    public Void visitExecutable(ExecutableElement e, Void p) {
      if (e.getKind() == METHOD) {
        Name name = e.getSimpleName();
        if (name.contentEquals(e.getEnclosingElement().getSimpleName())) {
          messager.printMessage(WARNING, "一个普通方法 [" + name + "] 不应当与类名重复，避免与构造函数产生混淆", e);
        }
        checkCamelCase(e, false);
      }
      super.visitExecutable(e, p);
      return null;
    }

    /**
		 * 检查变量名是否合法
		 */
    @Override
    public Void visitVariable(VariableElement e, Void p) {
      // 如果这个 Variable 是枚举或常量，则按大写命名检查；否则按照驼式命名法则检查
      if (e.getKind() == ENUM_CONSTANT || e.getConstantValue() != null || 
          heuristicallyConstant(e)) {
        checkAllCaps(e);
      } else {
        checkCamelCase(e, false);
      }
      return null;
    }

    private boolean heuristicallyConstant(VariableElement e) {
      if (e.getEnclosingElement().getKind() == INTERFACE) 
        return true;
      else if (e.getKind() == FIELD && e.getModifiers().containsAll(EnumSet.of(PUBLIC, STATIC, FINAL)))
        return true;
      else {
        return false;
      }
    }

    /**
		 * 检查传入的 Element 是否符合驼式命名法，如果不符合，则输出警告信息
		 */
    private void checkCamelCase(Element e, boolean initialCaps) {
      String name = e.getSimpleName().toString();
      boolean previousUpper = false;
      boolean conventional = true;
      int firstCodePoint = name.codePointAt(0);

      if (Character.isUpperCase(firstCodePoint)) {
        previousUpper = true;
        if (!initialCaps) {
          messager.printMessage(WARNING, " 名称 [" + name + "] 应当以小写字母开头", e);
          return;
        }
      } else if(Character.isLowerCase(firstCodePoint)) {
        if (initialCaps) {
          messager.printMessage(WARNING, " 名称 [" + name + "] 应当以大写字母开头", e);
          return;
        }
      } else {
        conventional = false;
      }

      if (conventional) {
        int cp = firstCodePoint;

        for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
          cp = name.codePointAt(i);
          if (Character.isUpperCase(cp)) {
            if (previousUpper) {
              conventional = false;
              break;
            }

            previousUpper = true;
          } else {
            previousUpper = false;
          }
        }
      }

      if (!conventional) {
        messager.printMessage(WARNING, " 名称 [" + name + "] 应当符合驼式命名法 (Camel Case Name)", e);
      }
    }

    /**
		 * 大写命名检查，要求第一个字母必须是大写的英文字母，其余部分可以是下划线或大写字母
		 */
    private void checkAllCaps(Element e) {
      String name = e.getSimpleName().toString();
      boolean conventional = true;
      int firstCodePoint = name.codePointAt(0);

      if (!Character.isUpperCase(firstCodePoint)) {
        conventional = false;
      } else {
        boolean previousUnderscore = false;
        int cp = firstCodePoint;
        for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
          cp = name.codePointAt(i);
          if (cp == (int) '_') {
            if (previousUnderscore) {
              conventional = false;
              break;
            }
            previousUnderscore = true;
          } else {
            previousUnderscore = false;
            if (!Character.isUpperCase(cp) && !Character.isDigit(cp)) {
              conventional = false;
              break;
            }
          }
        }
      }

      if (!conventional) {
        messager.printMessage(WARNING, "常量 [" + name + "] 应当全部以大写字母或下划线命名，并且以字母开头", e);
      }
    }
  }
}
```

​	NameChecker的代码看起来有点长，但实际上注释占了很大一部分，而且即使算上注释也不到190行。它通过一个继承于javax.lang.model.util.ElementScanner6的NameCheckScanner类，以Visitor模式来完成对语法树的遍历，分别执行visitType()、visitVariable()和visitExecutable()方法来访问类、字段和方法，这3个`visit*()`方法对各自的命名规则做相应的检查，checkCamelCase()与checkAllCaps()方法则用于实现驼式命名法和全大写命名规则的检查。

​	整个注解处理器只需NameCheckProcessor和NameChecker两个类就可以全部完成，为了验证我们的实战成果，代码清单10-18中提供了一段命名规范的“反面教材”代码，其中的每一个类、方法及字段的命名都存在问题，但是使用普通的Javac编译这段代码时不会提示任意一条警告信息。

​	代码清单10-18　包含了多处不规范命名的代码样例

```java
public class BADLY_NAMED_CODE {

  enum colors {
    red, blue, green;
  }

  static final int _FORTY_TWO = 42;

  public static int NOT_A_CONSTANT = _FORTY_TWO;

  protected void BADLY_NAMED_CODE() {
    return;
  }

  public void NOTcamelCASEmethodName() {
    return;
  }
}
```

### 10.4.3 运行与测试

> [早期（编译期）优化_clinit 比对编译前和编译后的代码_wiljm的博客-CSDN博客](https://blog.csdn.net/u013678930/article/details/52032328#)

​	我们可以通过Javac命令的“-processor”参数来执行编译时需要附带的注解处理器，如果有多个注解处理器的话，用逗号分隔。还可以使用-XprintRounds和-XprintProcessorInfo参数来查看注解处理器运作的详细信息，本次实战中的NameCheckProcessor的编译及执行过程如代码清单10-19所示。

​	代码清单10-19　注解处理器的运行过程

![img](https://img-blog.csdn.net/20160726172157651)

### 10.4.4 其他应用案例

​	NameCheckProcessor的实战例子只演示了JSR-269嵌入式注解处理API其中的一部分功能，基于这组API支持的比较有名的项目还有用于校验Hibernate标签使用正确性的Hibernate Validator AnnotationProcessor（本质上与NameCheckProcessor所做的事情差不多）、自动为字段生成getter和setter方法等辅助内容的Lombok（根据已有元素生成新的语法树元素）等，读者有兴趣的话可以参考它们官方站点的相关内容。

## 10.5 本章小结

​	在本章中，我们从Javac编译器源码实现的层次上学习了Java源代码编译为字节码的过程，分析了Java语言中泛型、主动装箱拆箱、条件编译等多种语法糖的前因后果，并实战练习了如何使用插入式注解处理器来完成一个检查程序命名规范的编译器插件。如本章概述中所说的，在前端编译器中，“优化”手段主要用于提升程序的编码效率，之所以把Javac这类将Java代码转变为字节码的编译器称作“前端编译器”，是因为它只完成了从程序到抽象语法树或中间字节码的生成，而在此之后，还有一组内置于Java虚拟机内部的“后端编译器”来完成代码优化以及从字节码生成本地机器码的过程，即前面多次提到的即时编译器或提前编译器，这个<u>后端编译器的编译速度及编译结果质量高低，是衡量Java虚拟机性能最重要的一个指标</u>。在第11章中，我们将会一探后端编译器的运作和优化过程。

# 11. 后端编译与优化

## 11.1 概述

​	如果我们把字节码看作是程序语言的一种中间表示形式（Intermediate Representation，IR）的话，那编译器无论在何时、在何种状态下把Class文件转换成与本地基础设施（硬件指令集、操作系统）相关的二进制机器码，它都可以视为整个编译过程的后端。如果读者阅读过本书的第2版，可能会发现本章的标题已经从“运行期编译与优化”悄然改成了“后端编译与优化”，这是因为在2012年的Java世界里，虽然提前编译（Ahead Of Time，AOT）早已有所应用，但相对而言，即时编译（Just In Time，JIT）才是占绝对主流的编译形式。**不过，最近几年编译技术发展出现了一些微妙的变化，提前编译不仅逐渐被主流JDK所支持，而且在Java编译技术的前沿研究中又重新成了一个热门的话题，所以再继续只提“运行期”和“即时编译”就显得不够全面了，在本章中它们两者都是主角**。

​	无论是提前编译器抑或即时编译器，都不是Java虚拟机必需的组成部分，《Java虚拟机规范》中从来没有规定过虚拟机内部必须要包含这些编译器，更没有限定或指导这些编译器应该如何去实现。但是，后端编译器编译性能的好坏、代码优化质量的高低却是衡量一款商用虚拟机优秀与否的关键指标之一，它们也是商业Java虚拟机中的核心，是最能体现技术水平与价值的功能。在本章中，我们将走进Java虚拟机的内部，探索后端编译器的运作过程和原理。

​	既然《Java虚拟机规范》没有具体的约束规则去限制后端编译器应该如何实现，那这部分功能就完全是与虚拟机具体实现相关的内容，如无特殊说明，本章中所提及的即时编译器都是特指HotSpot虚拟机内置的即时编译器，虚拟机也是特指HotSpot虚拟机。不过，本章虽然有大量的内容涉及了特定的虚拟机和编译器的实现层面，但主流Java虚拟机中后端编译器的行为会有很多相似相通之处，因此对其他虚拟机来说也具备一定的类比参考价值。

## 11.2 即时编译器

​	目前主流的两款商用Java虚拟机（HotSpot、OpenJ9）里，<u>Java程序最初都是通过解释器（Interpreter）进行解释执行的，当虚拟机发现某个方法或代码块的运行特别频繁，就会把这些代码认定为“热点代码”（Hot Spot Code），为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成本地机器码，并以各种手段尽可能地进行代码优化</u>，运行时完成这个任务的后端编译器被称为即时编译器。本节我们将会了解HotSpot虚拟机内的即时编译器的运作过程，此外，我们还将解决以下几个问题：

+ **为何HotSpot虚拟机要使用解释器与即时编译器并存的架构**？

+ 为何HotSpot虚拟机要实现两个（或三个）不同的即时编译器？

+ 程序何时使用解释器执行？何时使用编译器执行？

+ 哪些程序代码会被编译为本地代码？如何编译本地代码？

+ 如何从外部观察到即时编译器的编译过程和编译结果？

### 11.2.1 解释器与编译器

​	尽管并不是所有的Java虚拟机都采用解释器与编译器并存的运行架构，但目前主流的商用Java虚拟机，譬如HotSpot、OpenJ9等，内部都同时包含解释器与编译器，解释器与编译器两者各有优势：当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即运行。**当程序启动后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码，这样可以减少解释器的中间损耗，获得更高的执行效率**。<u>当程序运行环境中内存资源限制较大，可以使用解释执行节约内存（如部分嵌入式系统中和大部分的JavaCard应用中就只有解释器的存在），反之可以使用编译执行来提升效率</u>。**同时，解释器还可以作为编译器激进优化时后备的“逃生门”（如果情况允许，HotSpot虚拟机中也会采用不进行激进优化的客户端编译器充当“逃生门”的角色），让编译器根据概率选择一些不能保证所有情况都正确，但大多数时候都能提升运行速度的优化手段，当激进优化的假设不成立，如加载了新类以后，类型继承结构出现变化、出现“罕见陷阱”（Uncommon Trap）时可以通过逆优化（Deoptimization）退回到解释状态继续执行**，因此在整个Java虚拟机执行架构里，解释器与编译器经常是相辅相成地配合工作，其交互关系如图11-1所示。

![img](https://pic3.zhimg.com/80/v2-9eff369a0af4dbc39aa63997f457621e_1440w.webp)

​	HotSpot虚拟机中内置了两个（或三个）即时编译器，其中有两个编译器存在已久，分别被称为“客户端编译器”（Client Compiler）和“服务端编译器”（Server Compiler），或者简称为C1编译器和C2编译器（部分资料和JDK源码中C2也叫Opto编译器），第三个是在JDK 10时才出现的、长期目标是代替C2的Graal编译器。Graal编译器目前还处于实验状态，本章将安排出专门的小节对它讲解与实战，在本节里，我们将重点关注传统的C1、C2编译器的工作过程。

​	**在分层编译（Tiered Compilation）的工作模式出现以前，HotSpot虚拟机通常是采用解释器与其中一个编译器直接搭配的方式工作**，程序使用哪个编译器，只取决于虚拟机运行的模式，HotSpot虚拟机会根据自身版本与宿主机器的硬件性能自动选择运行模式，用户也可以使用“-client”或“-server”参数去强制指定虚拟机运行在客户端模式还是服务端模式。

​	**无论采用的编译器是客户端编译器还是服务端编译器，解释器与编译器搭配使用的方式在虚拟机中被称为“混合模式”（Mixed Mode）**，用户也可以使用参数“-Xint”强制虚拟机运行于“解释模式”（Interpreted Mode），这时候编译器完全不介入工作，全部代码都使用解释方式执行。另外，也可以使用参数“-Xcomp”强制虚拟机运行于“编译模式”（Compiled Mode），这时候将优先采用编译方式执行程序，但是解释器仍然要在编译无法进行的情况下介入执行过程。可以通过虚拟机的“-version”命令的输出结果显示出这三种模式，内容如代码清单11-1所示，请读者注意黑体字部分。

​	代码清单11-1　虚拟机执行模式

```shell
$java -version
java version "11.0.3" 2019-04-16 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.3+12-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.3+12-LTS, mixed mode)

$java -Xint -version
java version "11.0.3" 2019-04-16 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.3+12-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.3+12-LTS, interpreted mode)

$java -Xcomp -version
java version "11.0.3" 2019-04-16 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.3+12-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.3+12-LTS, compiled mode)
```

​	<u>由于即时编译器编译本地代码需要占用程序运行时间，通常要编译出优化程度越高的代码，所花费的时间便会越长；而且想要编译出优化程度更高的代码，解释器可能还要替编译器收集性能监控信息，这对解释执行阶段的速度也有所影响</u>。为了在程序启动响应速度与运行效率之间达到最佳平衡，HotSpot虚拟机在编译子系统中加入了**分层编译**的功能，分层编译的概念其实很早就已经提出，但直到JDK 6时期才被初步实现，后来一直处于改进阶段，最终在JDK 7的服务端模式虚拟机中作为默认编译策略被开启。<u>分层编译根据编译器编译、优化的规模与耗时，划分出不同的编译层次</u>，其中包括：

+ 第0层。程序纯解释执行，并且解释器不开启性能监控功能（Profiling）。

+ 第1层。使用客户端编译器将字节码编译为本地代码来运行，进行简单可靠的稳定优化，不开启性能监控功能。

+ 第2层。仍然使用客户端编译器执行，仅开启方法及回边次数统计等有限的性能监控功能。

+ 第3层。仍然使用客户端编译器执行，开启全部性能监控，除了第2层的统计信息外，还会收集如分支跳转、虚方法调用版本等全部的统计信息。

+ 第4层。使用服务端编译器将字节码编译为本地代码，相比起客户端编译器，服务端编译器会启用更多编译耗时更长的优化，还会根据性能监控信息进行一些不可靠的激进优化。

​	以上层次并不是固定不变的，根据不同的运行参数和版本，虚拟机可以调整分层的数量。各层次编译之间的交互、转换关系如图11-2所示。

​	**实施分层编译后，解释器、客户端编译器和服务端编译器就会同时工作，热点代码都可能会被多次编译，用客户端编译器获取更高的编译速度，用服务端编译器来获取更好的编译质量，在解释执行的时候也无须额外承担收集性能监控信息的任务，而在服务端编译器采用高复杂度的优化算法时，客户端编译器可先采用简单优化来为它争取更多的编译时间**。

![img](https://pic2.zhimg.com/80/v2-168a69b7f0376ceebf36c4d5b2a1bb91_1440w.webp)

### 11.2.2 编译对象与触发条件

​	在本章概述中提到了在运行过程中会被即时编译器编译的目标是“热点代码”，这里所指的热点代码主要有两类，包括：

+ 被多次调用的方法。
+ **被多次执行的循环体**。

​	前者很好理解，一个方法被调用得多了，方法体内代码执行的次数自然就多，它成为“热点代码”是理所当然的。而后者则是为了解决当一个方法只被调用过一次或少量的几次，但是方法体内部存在循环次数较多的循环体，这样循环体的代码也被重复执行多次，因此这些代码也应该认为是“热点代码”。

​	**对于这两种情况，编译的目标对象都是整个方法体，而不会是单独的循环体**。第一种情况，由于是依靠方法调用触发的编译，那编译器理所当然地会以整个方法作为编译对象，这种编译也是虚拟机中标准的即时编译方式。**而对于后一种情况，尽管编译动作是由循环体所触发的，热点只是方法的一部分，但编译器依然必须以整个方法作为编译对象，只是<u>执行入口（从方法第几条字节码指令开始执行）会稍有不同，编译时会传入执行入口点字节码序号（Byte Code Index，BCI）</u>**。这种编译方式因为编译发生在方法执行的过程中，因此被很形象地称为“栈上替换”（On Stack Replacement，OSR），即方法的栈帧还在栈上，方法就被替换了。

​	读者可能还会有疑问，在上面的描述里，无论是“多次执行的方法”，还是“多次执行的代码块”，所谓“多次”只定性不定量，并不是一个具体严谨的用语，那到底多少次才算“多次”呢？还有一个问题，就是Java虚拟机是如何统计某个方法或某段代码被执行过多少次的呢？解决了这两个问题，也就解答了即时编译被触发的条件。

​	要知道某段代码是不是热点代码，是不是需要触发即时编译，这个行为称为**“热点探测”（HotSpot Code Detection）**，其实进行热点探测并不一定要知道方法具体被调用了多少次，目前主流的热点探测判定方式有两种，分别是：

+ **基于采样的热点探测（Sample Based Hot Spot Code Detection）**。采用这种方法的虚拟机会周期性地检查各个线程的调用栈顶，如果发现某个（或某些）方法经常出现在栈顶，那这个方法就是“热点方法”。基于采样的热点探测的好处是实现简单高效，还可以很容易地获取方法调用关系（将调用堆栈展开即可），缺点是很难精确地确认一个方法的热度，容易因为受到线程阻塞或别的外界因素的影响而扰乱热点探测。
+ **基于计数器的热点探测（Counter Based Hot Spot Code Detection）**。采用这种方法的虚拟机会为每个方法（甚至是代码块）建立计数器，统计方法的执行次数，如果执行次数超过一定的阈值就认为它是“热点方法”。这种统计方法实现起来要麻烦一些，需要为每个方法建立并维护计数器，而且不能直接获取到方法的调用关系。但是它的统计结果相对来说更加精确严谨。

​	**这两种探测手段在商用Java虚拟机中都有使用到，譬如J9用过第一种采样热点探测，而在HotSpot虚拟机中使用的是第二种基于计数器的热点探测方法，为了实现热点计数，HotSpot为每个方法准备了两类计数器：方法调用计数器（Invocation Counter）和回边计数器（Back Edge Counter，“回边”的意思就是指在循环边界往回跳转）。当虚拟机运行参数确定的前提下，这两个计数器都有一个明确的阈值，计数器阈值一旦溢出，就会触发即时编译**。

​	我们首先来看看方法调用计数器。顾名思义，这个计数器就是用于统计方法被调用的次数，它的默认阈值在客户端模式下是1500次，在服务端模式下是10000次，这个阈值可以通过虚拟机参数`-XX:CompileThreshold`来人为设定。<u>当一个方法被调用时，虚拟机会先检查该方法是否存在被即时编译过的版本，如果存在，则优先使用编译后的本地代码来执行。如果不存在已被编译过的版本，则将该方法的调用计数器值加一，然后判断方法调用计数器与回边计数器值之和是否超过方法调用计数器的阈值。一旦已超过阈值的话，将会向即时编译器提交一个该方法的代码编译请求</u>。

​	**如果没有做过任何设置，执行引擎默认不会同步等待编译请求完成，而是继续进入解释器按照解释方式执行字节码，直到提交的请求被即时编译器编译完成**。当编译工作完成后，这个方法的调用入口地址就会被系统自动改写成新值，下一次调用该方法时就会使用已编译的版本了，整个即时编译的交互过程如图11-3所示。

​	**在默认设置下，方法调用计数器统计的并不是方法被调用的绝对次数，而是一个相对的执行频率，即一段时间之内方法被调用的次数**。**当超过一定的时间限度，如果方法的调用次数仍然不足以让它提交给即时编译器编译，那该方法的调用计数器就会被减少一半，这个过程被称为方法调用计数器热度的衰减（Counter Decay），而这段时间就称为此方法统计的半衰周期（Counter Half Life Time），<u>进行热度衰减的动作是在虚拟机进行垃圾收集时顺便进行的</u>**，可以使用虚拟机参数`-XX:-UseCounterDecay`来关闭热度衰减，让方法计数器统计方法调用的绝对次数，这样只要系统运行时间足够长，程序中绝大部分方法都会被编译成本地代码。另外还可以使用`-XX:CounterHalfLifeTime`参数设置半衰周期的时间，单位是秒。

![img](https://img-blog.csdnimg.cn/2020052618514515.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQyMDU0MzQ=,size_16,color_FFFFFF,t_70)

​	图11-3　方法调用计数器触发即时编译

​	现在我们再来看看另外一个计数器——回边计数器，它的作用是统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令就称为“回边（Back Edge）”，很显然建立回边计数器统计的目的是为了触发栈上的替换编译。

​	关于回边计数器的阈值，虽然HotSpot虚拟机也提供了一个类似于方法调用计数器阈值`-XX:CompileThreshold`的参数`-XX:BackEdgeThreshold`供用户设置，但是当前的HotSpot虚拟机实际上并未使用此参数，我们必须设置另外一个参数`-XX:OnStackReplacePercentage`来间接调整回边计数器的阈值，其计算公式有如下两种。

+ 虚拟机运行在客户端模式下，回边计数器阈值计算公式为：方法调用计数器阈值（`-XX:CompileThreshold`）乘以OSR比率（`-XX:OnStackReplacePercentage`）除以100。其中`-XX:OnStackReplacePercentage`默认值为933，如果都取默认值，那客户端模式虚拟机的回边计数器的阈值为13995。

+ 虚拟机运行在服务端模式下，回边计数器阈值的计算公式为：方法调用计数器阈值（`-XX:CompileThreshold`）乘以（OSR比率（`-XX:OnStackReplacePercentage`）减去解释器监控比率（`-XX:InterpreterProfilePercentage`）的差值）除以100。其中`-XX:OnStack ReplacePercentage`默认值为140，`-XX:InterpreterProfilePercentage`默认值为33，如果都取默认值，那服务端模式虚拟机回边计数器的阈值为10700。

​	**当解释器遇到一条回边指令时，会先查找将要执行的代码片段是否有已经编译好的版本，如果有的话，它将会优先执行已编译的代码，否则就把回边计数器的值加一，然后判断方法调用计数器与回边计数器值之和是否超过回边计数器的阈值**。当超过阈值的时候，将会提交一个栈上替换编译请求，并且把回边计数器的值稍微降低一些，以便继续在解释器中执行循环，等待编译器输出编译结果，整个执行过程如图11-4所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/501663078171426c9da265690b97ba7a.png)

​	**与方法计数器不同，回边计数器没有计数热度衰减的过程，因此这个计数器统计的就是该方法循环执行的绝对次数。当计数器溢出的时候，它还会把方法计数器的值也调整到溢出状态，这样下次再进入该方法的时候就会执行标准编译过程**。

​	最后还要提醒一点，图11-2和图11-3都仅仅是描述了客户端模式虚拟机的即时编译方式，对于服务端模式虚拟机来说，执行情况会比上面描述还要复杂一些。从理论上了解过编译对象和编译触发条件后，我们还可以从HotSpot虚拟机的源码中简单观察一下这两个计数器，在MehtodOop.hpp（一个methodOop对象代表了一个Java方法）中，定义了Java方法在虚拟机中的内存布局，如下所示：

![img](https://img-blog.csdn.net/20151106000917089)

​	在这段注释所描述的方法内存布局里，每一行表示占用32个比特，从中我们可以清楚看到方法调用计数器和回边计数器所在的位置和数据宽度，另外还有from_compiled_entry和from_interpreted_entry两个方法入口所处的位置。

### 11.2.3 编译过程

​	在默认条件下，无论是方法调用产生的标准编译请求，还是栈上替换编译请求，虚拟机在编译器还未完成编译之前，都仍然将按照解释方式继续执行代码，而编译动作则在后台的编译线程中进行。用户可以通过参数`-XX:-BackgroundCompilation`来禁止后台编译，后台编译被禁止后，当达到触发即时编译的条件时，执行线程向虚拟机提交编译请求以后将会一直阻塞等待，直到编译过程完成再开始执行编译器输出的本地代码。

​	那在后台执行编译的过程中，编译器具体会做什么事情呢？服务端编译器和客户端编译器的编译过程是有所差别的。对于客户端编译器来说，它是一个相对简单快速的三段式编译器，主要的关注点在于局部性的优化，而放弃了许多耗时较长的全局优化手段。

​	在第一个阶段，一个平台独立的前端将字节码构造成一种高级中间代码表示（High-LevelIntermediate Representation，HIR，即与目标机器指令集无关的中间表示）。HIR使用静态单分配（Static Single Assignment，SSA）的形式来代表代码值，这可以使得一些在HIR的构造过程之中和之后进行的优化动作更容易实现。在此之前编译器已经会在字节码上完成一部分基础优化，如方法内联、常量传播等优化将会在字节码被构造成HIR之前完成。

​	在第二个阶段，一个平台相关的后端从HIR中产生低级中间代码表示（Low-Level IntermediateRepresentation，LIR，即与目标机器指令集相关的中间表示），而在此之前会在HIR上完成另外一些优化，如空值检查消除、范围检查消除等，以便让HIR达到更高效的代码表示形式。

​	最后的阶段是在平台相关的后端使用线性扫描算法（Linear Scan Register Allocation）在LIR上分配寄存器，并在LIR上做窥孔（Peephole）优化，然后产生机器代码。客户端编译器大致的执行过程如图11-5所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/ae9b5c767c8044ed9144a5284beaac25.png)

​	而服务端编译器则是专门面向服务端的典型应用场景，并为服务端的性能配置针对性调整过的编译器，也是一个能容忍很高优化复杂度的高级编译器，几乎能达到GNU C++编译器使用`-O2`参数时的优化强度。它会执行大部分经典的优化动作，如：无用代码消除（Dead Code Elimination）、循环展开（Loop Unrolling）、循环表达式外提（Loop Expression Hoisting）、消除公共子表达式（Common Subexpression Elimination）、常量传播（Constant Propagation）、基本块重排序（Basic Block Reordering）等，还会实施一些与Java语言特性密切相关的优化技术，如范围检查消除（Range Check Elimination）、空值检查消除（Null Check Elimination，不过并非所有的空值检查消除都是依赖编译器优化的，有一些是代码运行过程中自动优化了）等。另外，还可能根据解释器或客户端编译器提供的性能监控信息，进行一些不稳定的预测性激进优化，如守护内联（Guarded Inlining）、分支频率预测（Branch Frequency Prediction）等，本章的下半部分将会挑选上述的一部分优化手段进行分析讲解，在此就先不做展开。

​	<u>服务端编译采用的寄存器分配器是一个全局图着色分配器，它可以充分利用某些处理器架构（如RISC）上的大寄存器集合。以即时编译的标准来看，服务端编译器无疑是比较缓慢的，但它的编译速度依然远远超过传统的静态优化编译器，而且它相对于客户端编译器编译输出的代码质量有很大提高，可以大幅减少本地代码的执行时间，从而抵消掉额外的编译时间开销，所以也有很多非服务端的应用选择使用服务端模式的HotSpot虚拟机来运行</u>。

​	在本节中出现了许多编译原理和代码优化中的概念名词，没有这方面基础的读者，可能阅读起来会感觉到很抽象、很理论化。有这种感觉并不奇怪，一方面，即时编译过程本来就是一个虚拟机中最能体现技术水平也是最复杂的部分，很难在几页纸的篇幅中介绍得面面俱到；另一方面，这个过程对Java开发者来说是完全透明的，程序员平时无法感知它的存在。所幸，HotSpot虚拟机提供了两个可视化的工具，让我们可以“看见”即时编译器的优化过程。下面笔者将实践演示这个过程。

### 11.2.4 实战: 查看及分析即时编译结果

> [[深入理解Java虚拟机\]第十一章 程序编译与代码优化-晚期(运行期)优化_Coding-lover的博客-CSDN博客](https://blog.csdn.net/coslay/article/details/49691473)
>
> [Ideal Graph Visualizer (oracle.com)](https://docs.oracle.com/en/graalvm/enterprise/22/docs/tools/igv/)

​	本节中提到的部分运行参数需要FastDebug或SlowDebug优化级别的HotSpot虚拟机才能够支持，Product级别的虚拟机无法使用这部分参数。如果读者使用的是根据第1章的教程自己编译的JDK，请注意将“`--with-debug-level`”参数设置为“fastdebug”或者“slowdebug”。现在Oracle和OpenJDK网站上都已经不再直接提供FastDebug的JDK下载了（从JDK 6 Update 25之后官网上就没有再提供下载），所以要完成本节全部测试内容，读者除了自己动手编译外，就只能到网上搜索非官方编译的版本了。本次实战中所有的测试都基于代码清单11-2所示的Java代码来进行。

​	代码清单11-2　测试代码

```java
public static final int NUM = 15000;
public static int doubleValue(int i) {
  // 这个空循环用于后面演示JIT代码优化过程
  for(int j=0; j<100000; j++);
  return i * 2;
}
public static long calcSum() {
  long sum = 0;
  for (int i = 1; i <= 100; i++) {
    sum += doubleValue(i);
  }
  return sum;
}
public static void main(String[] args) {
  for (int i = 0; i < NUM; i++) {
    calcSum();
  }
}
```

​	我们首先来运行这段代码，并且确认这段代码是否触发了即时编译。要知道某个方法是否被编译过，可以使用参数-XX：+PrintCompilation要求虚拟机在即时编译时将被编译成本地代码的方法名称打印出来，如代码清单11-3所示（其中带有“%”的输出说明是由回边计数器触发的栈上替换编译）。

​	代码清单11-3　被即时编译的代码

```shell
VM option '+PrintCompilation'
		310 	1 		java.lang.String::charAt (33 bytes)
		329 	2 		org.fenixsoft.jit.Test::calcSum (26 bytes)
		329 	3 		org.fenixsoft.jit.Test::doubleValue (4 bytes)
		332 	1% 		org.fenixsoft.jit.Test::main @ 5 (20 bytes)
```

​	从代码清单11-3输出的信息中可以确认，main()、calcSum()和doubleValue()方法已经被编译，我们还可以加上参数-XX：+PrintInlining要求虚拟机输出方法内联信息，如代码清单11-4所示。

​	代码清单11-4　内联信息

```shell
VM option '+PrintCompilation'
VM option '+PrintInlining'
		273 	1 		java.lang.String::charAt (33 bytes)
		291 	2 		org.fenixsoft.jit.Test::calcSum (26 bytes)
		@ 		9 		org.fenixsoft.jit.Test::doubleValue inline (hot)
		294 	3 		org.fenixsoft.jit.Test::doubleValue (4 bytes)
		295 	1% 		org.fenixsoft.jit.Test::main @ 5 (20 bytes)
		@ 		5 		org.fenixsoft.jit.Test::calcSum inline (hot)
		@ 		9 		org.fenixsoft.jit.Test::doubleValue inline (hot)
```

​	从代码清单11-4的输出日志中可以看到，doubleValue()方法已被内联编译到calcSum()方法中，而calcSum()方法又被内联编译到main()方法里面，所以虚拟机再次执行main()方法的时候（举例而已，main()方法当然不会运行两次），calcSum()和doubleValue()方法是不会再被实际调用的，没有任何方法分派的开销，它们的代码逻辑都被直接内联到main()方法里面了。

​	除了查看哪些方法被编译之外，我们还可以更进一步看到即时编译器生成的机器码内容。不过如果得到的是即时编译器输出一串0和1，对于我们人类来说是没法阅读的，机器码至少要反汇编成基本的汇编语言才可能被人类阅读。虚拟机提供了一组通用的反汇编接口，可以接入各种平台下的反汇编适配器，如使用32位x86平台应选用hsdis-i386适配器，64位则需要选用hsdis-amd64，其余平台的适配器还有如hsdis-sparc、hsdis-sparcv9和hsdis-aarch64等，读者可以下载或自己编译出与自己机器相符合的反汇编适配器，之后将其放置在JAVA_HOME/lib/amd64/server下，只要与jvm.dll或libjvm.so的路径相同即可被虚拟机调用。为虚拟机安装了反汇编适配器之后，我们就可以使用`-XX:+PrintAssembly`参数要求虚拟机打印编译方法的汇编代码了，关于HSDIS插件更多的操作介绍，可以参考第4章的相关内容。

​	如果没有HSDIS插件支持，也可以使用`-XX:+PrintOptoAssembly`（用于服务端模式的虚拟机）或`-XX:+PrintLIR`（用于客户端模式的虚拟机）来输出比较接近最终结果的中间代码表示，代码清单11-2所示代码被编译后部分反汇编（使用`-XX:+PrintOptoAssembly`）的输出结果如代码清单11-5所示。对于阅读来说，使用`-XX:+PrintOptoAssembly`参数输出的伪汇编结果包含了更多的信息（主要是注释），有利于人们阅读、理解虚拟机即时编译器的优化结果。

​	代码清单11-5　本地机器码反汇编信息（部分）

```java
…… ……
000 B1: 	# 			N1 <- BLOCK HEAD IS JUNK Freq: 1
000 			pushq 	rbp
					subq 		rsp, #16 # Create frame
					nop 		# nop for patch_verified_entry
006 			movl 		RAX, RDX # spill
008 			sall 		RAX, #1
00a 			addq 		rsp, 16 # Destroy frame
					popq 		rbp
					testl 	rax, [rip + #offset_to_poll_page] # Safepoint: poll for GC
…… ……
```

​	前面提到的使用-XX：+PrintAssembly参数输出反汇编信息需要FastDebug或SlowDebug优化级别的HotSpot虚拟机才能直接支持，如果使用Product版的虚拟机，则需要加入参数`-XX:+UnlockDiagnosticVMOptions`打开虚拟机诊断模式。

​	如果除了本地代码的生成结果外，还想再进一步跟踪本地代码生成的具体过程，那可以使用参数`-XX:+PrintCFGToFile`（用于客户端编译器）或`-XX:PrintIdealGraphFile`（用于服务端编译器）要求Java虚拟机将编译过程中各个阶段的数据（譬如对客户端编译器来说包括字节码、HIR生成、LIR生成、寄存器分配过程、本地代码生成等数据）输出到文件中。然后使用Java HotSpot Client CompilerVisualizer（用于分析客户端编译器）或Ideal Graph Visualizer（用于分析服务端编译器）打开这些数据文件进行分析。接下来将以使用服务端编译器为例，讲解如何分析即时编译的代码生成过程。这里先把重点放在编译整体过程阶段及Ideal Graph Visualizer功能介绍上，在稍后在介绍Graal编译器的实战小节里，我们会使用Ideal Graph Visualizer来详细分析虚拟机进行代码优化和生成时的执行细节，届时我们将重点关注编译器是如何实现这些优化的。

​	服务端编译器的中间代码表示是一种名为理想图（Ideal Graph）的程序依赖图（ProgramDependence Graph，PDG），在运行Java程序的FastDebug或SlowDebug优化级别的虚拟机上的参数中加入“`-XX:PrintIdealGraphLevel=2-XX:PrintIdeal-GraphFile=ideal.xml`”，即时编译后将会产生一个名为ideal.xml的文件，它包含了服务端编译器编译代码的全过程信息，可以使用Ideal Graph Visualizer对这些信息进行分析。

![这里写图片描述](https://img-blog.csdn.net/20170618190357990?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjEyNDQzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	图11-6　编译过的方法列表

​	Ideal Graph Visualizer加载ideal.xml文件后，在Outline面板上将显示程序运行过程中编译过的方法列表，如图11-6所示。这里列出的方法是代码清单11-2中所示的测试代码，其中doubleValue()方法出现了两次，这是由于该方法的编译结果存在标准编译和栈上替换编译两个版本。在代码清单11-2中，专门为doubleValue()方法增加了一个空循环，这个循环对方法的运算结果不会产生影响，但如果没有任何优化，执行该循环就会耗费处理器时间。<u>直到今天还有不少程序设计的入门教程会把空循环当作程序延时的手段来介绍，下面我们就来看看在Java语言中这样的做法是否真的能起到延时的作用</u>。

​	展开方法根节点，可以看到下面罗列了方法优化过程的各个阶段（根据优化措施的不同，每个方法所经过的阶段也会有所差别）的理想图，我们先打开“After Parsing”这个阶段。前面提到，即时编译器编译一个Java方法时，首先要把字节码解析成某种中间表示形式，然后才可以继续做分析和优化，最终生成代码。“After Parsing”就是服务端编译器刚完成解析，还没有做任何优化时的理想图表示。打开这个图后，读者会看到其中有很多有颜色的方块，如图11-7所示。每一个方块代表了一个程序的基本块（Basic Block）。基本块是指程序按照控制流分割出来的最小代码块，它的特点是只有唯一的一个入口和唯一的一个出口，只要基本块中第一条指令被执行了，那么基本块内所有指令都会按照顺序全部执行一次。

![这里写图片描述](https://img-blog.csdn.net/20170618190818280?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjEyNDQzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	图11-7　基本块图示（1）

​	代码清单11-2所示的doubleValue()方法虽然只有简单的两行字，但是按基本块划分后，形成的图形结构却要比想象中复杂得多，这是因为一方面要满足Java语言所定义的安全需要（如类型安全、空指针检查）和Java虚拟机的运作需要（如Safepoint轮询），另一方面有些程序代码中一行语句就可能形成几个基本块（例如循环语句）。对于例子中的doubleValue()方法，如果忽略语言安全检查的基本块，可以简单理解为按顺序执行了以下几件事情：

1）程序入口，建立栈帧。

2）设置j=0，进行安全点（Safepoint）轮询，跳转到4的条件检查。

3）执行j++。

4）条件检查，如果j<100000，跳转到3。

5）设置i=i*2，进行安全点轮询，函数返回。

​	以上几个步骤反映到Ideal Graph Visualizer生成的图形上，就是图11-8所示的内容。这样我们若想看空循环是否被优化掉，或者何时被优化掉，只要观察代表循环的基本块是否被消除掉，以及何时被优化掉就可以了。

![这里写图片描述](https://img-blog.csdn.net/20170618191456305?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjEyNDQzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	图11-8　基本块图示（2）

​	要观察这一点，可以在Outline面板上右击“Difference to current graph”，让软件自动分析指定阶段与当前打开的理想图之间的差异，如果基本块被消除了，将会以红色显示。对“AfterParsing”和“PhaseIdealLoop 1”阶段的理想图进行差异分析，会发现在“PhaseIdealLoop 1”阶段循环操作就被消除了，如图11-9所示，这也就**说明空循环在最终的本地代码里实际上是不会被执行的**。

![img](https://img-blog.csdn.net/20151106010308045)

​	图11-9　基本块图示（3）

​	从“After Parsing”阶段开始，一直到最后的“Final Code”阶段都可以看到doubleValue()方法的理想图从繁到简的变迁过程，这也反映了Java虚拟机即时编译器尽力优化代码的过程。到了最后的“FinalCode”阶段，**不仅空循环的开销被消除了，许多语言安全保障措施和GC安全点的轮询操作也被一起消除了，因为编译器判断到即使不做这些保障措施，程序也能得到相同的结果，不会有可观察到的副作用产生，虚拟机的运行安全也不会受到威胁**。

## 11.3 提前编译器

​	在1998年，GNU组织公布了著名的GCC家族（GNU CompilerCollection）的新成员GNU Compiler for Java（GCJ，2018年从GCC家族中除名），这也是一款Java的提前编译器，而且曾经被广泛应用。在OpenJDK流行起来之前，各种Linux发行版带的Java实现通常就是GCJ。

​	但是提前编译很快又在Java世界里沉寂了下来，因为当时Java的一个核心优势是平台中立性，其宣传口号是“一次编译，到处运行”，这与平台相关的提前编译在理念上就是直接冲突的。GCJ出现之后在长达15年的时间里，提前编译这条故事线上基本就再没有什么大的新闻和进展了。类似的状况一直持续至2013年，直到在Android的世界里，剑走偏锋使用提前编译的ART（Android Runtime）横空出世。ART一诞生马上就把使用即时编译的Dalvik虚拟机按在地上使劲蹂躏，仅经过Android 4.4一个版本的短暂交锋之后，ART就迅速终结了Dalvik的性命，把它从Android系统里扫地出门。

### 11.3.1 提前编译的优劣得失

​	现在提前编译产品和对其的研究有着两条明显的分支：

+ **一条分支是做与传统C、C++编译器类似的，在程序运行之前把程序代码编译成机器码的静态翻译工作**；
+ **另外一条分支是把原本即时编译器在运行时要做的编译工作提前做好并保存下来，下次运行到这些代码（譬如公共库代码在被同一台机器其他Java进程使用）时直接把它加载进来使用**。

​	*我们先来说第一条，这是传统的提前编译应用形式，它在Java中存在的价值直指即时编译的最大弱点：即时编译要占用程序运行时间和运算资源。即使现在先进的即时编译器已经足够快，以至于能够容忍相当高的优化复杂度了（譬如Azul公司基于LLVM的Falcon JIT，就能够以相当于Clang-O3的优化级别进行即时编译；又譬如OpenJ9的即时编译器Testarossa，它的静态版本同时也作为C、C++语言的提前编译器使用，优化的复杂度自然也支持得非常高）；即使现在先进的即时编译器架构有了分层编译的支持，可以先用快速但低质量的即时编译器为高质量的即时编译器争取出更多编译时间，但是，无论如何，即时编译消耗的时间都是原本可用于程序运行的时间，消耗的运算资源都是原本可用于程序运行的资源，这个约束从未减弱，更不会消失，始终是悬在即时编译头顶的达摩克利斯之剑。*

​	这里举个更具体的例子来帮助读者理解这种约束：在编译过程中最耗时的优化措施之一是通过“过程间分析”（Inter-Procedural Analysis，IPA，也经常被称为全程序分析，即Whole Program Analysis）来获得诸如某个程序点上某个变量的值是否一定为常量、某段代码块是否永远不可能被使用、在某个点调用的某个虚方法是否只能有单一版本等的分析结论。这些信息对生成高质量的优化代码有着极为巨大的价值，但是要精确（譬如对流敏感、对路径敏感、对上下文敏感、对字段敏感）得到这些信息，必须在全程序范围内做大量极耗时的计算工作，目前所有常见的Java虚拟机对过程间分析的支持都相当有限，要么借助大规模的方法内联来打通方法间的隔阂，以过程内分析（Intra-Procedural Analysis，只考虑过程内部语句，不考虑过程调用的分析）来模拟过程间分析的部分效果；要么借助可假设的激进优化，不求得到精确的结果，只求按照最可能的状况来优化，有问题再退回来解析执行。<u>但如果是在程序运行之前进行的静态编译，这些耗时的优化就可以放心大胆地进行了，譬如Graal VM中的Substrate VM，在创建本地镜像的时候，就会采取许多原本在HotSpot即时编译中并不会做的全程序优化措施以获得更好的运行时性能，反正做镜像阶段慢一点并没有什么大影响</u>。同理，这也是ART打败Dalvik的主要武器之一，连副作用也是相似的。在Android 5.0和6.0版本，安装一个稍微大一点的Android应用都是按分钟来计时的，以至于从Android 7.0版本起重新启用了解释执行和即时编译（但这已与Dalvik无关，它彻底凉透了），等空闲时系统再在后台自动进行提前编译。

​	<u>关于提前编译的第二条路径，本质是给即时编译器做缓存加速，去改善Java程序的启动时间，以及需要一段时间预热后才能到达最高性能的问题</u>。**这种提前编译被称为动态提前编译（Dynamic AOT）或者索性就大大方方地直接叫即时编译缓存（JIT Caching）**。<u>在目前的Java技术体系里，这条路径的提前编译已经完全被主流的商用JDK支持</u>。在商业应用中，这条路径最早出现在JDK 6版本的IBM J9虚拟机上，那时候在它的CDS（Class Data Sharing）功能的缓存中就有一块是即时编译缓存。不过这个缓存和CDS缓存一样是虚拟机运行时自动生成的，直接来源于J9的即时编译器，而且为了进程兼容性，很多激进优化都不能肆意运用，所以编译输出的代码质量反而要低于即时编译器。**真正引起业界普遍关注的是OpenJDK/OracleJDK 9中所带的Jaotc提前编译器，这是一个基于Graal编译器实现的新工具，目的是让用户可以针对目标机器，为应用程序进行提前编译。HotSpot运行时可以直接加载这些编译的结果，实现加快程序启动速度，减少程序达到全速运行状态所需时间的目的**。这里面确实有比较大的优化价值，试想一下，各种Java应用最起码会用到Java的标准类库，如java.base等模块，如果能够将这个类库提前编译好，并进行比较高质量的优化，显然能够节约不少应用运行时的编译成本。关于这点，我们将在下一节做一个简单的实战练习，而<u>在此要说明的是，这的确是很好的想法，但实际应用起来并不是那么容易，原因是这种提前编译方式不仅要和目标机器相关，甚至还必须与HotSpot虚拟机的运行时参数绑定</u>。譬如虚拟机运行时采用了不同的垃圾收集器，这原本就需要即时编译子系统的配合（典型的如生成内存屏障代码，见第3章相关介绍）才能正确工作，要做提前编译的话，自然也要把这些配合的工作平移过去。至于前面提到过的提前编译破坏平台中立性、字节膨胀等缺点当然还存在，这里就不重复了。<u>尽管还有许多困难，但提前编译无疑已经成为一种极限榨取性能（启动、响应速度）的手段，且被官方JDK关注，相信日后会更加灵活、更加容易使用，就如已经相当成熟的CDS（AppCDS需要用户参与）功能那样，几乎不需要用户介入，可自动完成</u>。

​	最后，我们还要思考一个问题：提前编译的代码输出质量，一定会比即时编译更高吗？提前编译因为没有执行时间和资源限制的压力，能够毫无顾忌地使用重负载的优化手段，这当然是一个极大的优势，但即时编译难道就没有能与其竞争的强项了吗？当然是有的，尽管即时编译在时间和运算资源方面的劣势是无法忽视的，但其依然有自己的优势。接下来便要开始即时编译器的绝地反击了，笔者将简要介绍三种即时编译器相对于提前编译器的天然优势。

​	首先，是**性能分析制导优化**（Profile-Guided Optimization，PGO）。上一节介绍HotSpot的即时编译器时就多次提及在解释器或者客户端编译器运行过程中，会不断收集性能监控信息，譬如某个程序点抽象类通常会是什么实际类型、条件判断通常会走哪条分支、方法调用通常会选择哪个版本、循环通常会进行多少次等，这些数据一般在静态分析时是无法得到的，或者不可能存在确定且唯一的解，最多只能依照一些启发性的条件去进行猜测。但在动态运行时却能看出它们具有非常明显的偏好性。如果一个条件分支的某一条路径执行特别频繁，而其他路径鲜有问津，那就可以把热的代码集中放到一起，集中优化和分配更好的资源（分支预测、寄存器、缓存等）给它。

​	其次，是**激进预测性优化**（Aggressive Speculative Optimization），这也已经成为很多即时编译优化措施的基础。静态优化无论如何都必须保证优化后所有的程序外部可见影响（不仅仅是执行结果）与优化前是等效的，不然优化之后会导致程序报错或者结果不对，若出现这种情况，则速度再快也是没有价值的。然而，相对于提前编译来说，即时编译的策略就可以不必这样保守，如果性能监控信息能够支持它做出一些正确的可能性很大但无法保证绝对正确的预测判断，就已经可以大胆地按照高概率的假设进行优化，万一真的走到罕见分支上，大不了退回到低级编译器甚至解释器上去执行，并不会出现无法挽救的后果。只要出错概率足够低，这样的优化往往能够大幅度降低目标程序的复杂度，输出运行速度非常高的代码。**譬如在Java语言中，默认方法都是虚方法调用，部分C、C++程序员（甚至一些老旧教材）会说虚方法是不能内联的，但如果Java虚拟机真的遇到虚方法就去查虚表而不做内联的话，Java技术可能就已经因性能问题而被淘汰很多年了。实际上虚拟机会通过类继承关系分析等一系列激进的猜测去做去虚拟化（Devitalization），以保证绝大部分有内联价值的虚方法都可以顺利内联。内联是最基础的一项优化措施，本章稍后还会对专门的Java虚拟机具体如何做虚方法内联进行详细讲解**。

​	最后，是**链接时优化**（Link-Time Optimization，LTO），<u>Java语言天生就是动态链接的，一个个Class文件在运行期被加载到虚拟机内存当中，然后在即时编译器里产生优化后的本地代码，这类事情在Java程序员眼里看起来毫无违和之处</u>。但如果类似的场景出现在使用提前编译的语言和程序上，譬如C、C++的程序要调用某个动态链接库的某个方法，就会出现很明显的边界隔阂，还难以优化。这是因为主程序与动态链接库的代码在它们编译时是完全独立的，两者各自编译、优化自己的代码。这些代码的作者、编译的时间，以及编译器甚至很可能都是不同的，当出现跨链接库边界的调用时，那些理论上应该要做的优化——譬如做对调用方法的内联，就会执行起来相当的困难。如果刚才说的虚方法内联让C、C++程序员理解还算比较能够接受的话（其实C++编译器也可以通过一些技巧来做到虚方法内联），那这种跨越动态链接库的方法内联在他们眼里可能就近乎于离经叛道了（但实际上依然是可行的）。

​	经过以上的讨论，读者应该能够理解提前编译器的价值与优势所在了，但忽略具体的应用场景就说它是万能的银弹，那肯定是有失偏颇的，提前编译有它的应用场景，也有它的弱项与不足，相信未来很长一段时间内，即时编译和提前编译都会是Java后端编译技术的共同主角。

### 11.3.2 实战: Jaotc的提前编译

> [jaotc_ifredom的博客-CSDN博客](https://blog.csdn.net/win7583362/article/details/126958751) <= 基本就是按照书本来的示例，也可以直接看这个文章

​	JDK 9引入了用于支持对Class文件和模块进行提前编译的工具Jaotc，以减少程序的启动时间和到达全速性能的预热时间，但由于这项功能必须针对特定物理机器和目标虚拟机的运行参数来使用，加之限制太多，Java开发人员对此了解、使用普遍比较少，本节我们将用Jaotc来编译Java SE的基础库（java.base模块），以改善本机Java环境的执行效率。

​	...省略代码和指令操作演示Jaotc的使用示例（建议这块直接看书）

​	目前状态的Jaotc还有许多需要完善的地方，仍难以直接编译SpringBoot、MyBatis这些常见的第三方工具库，甚至在众多Java标准模块中，能比较顺利编译的也只有java.base模块而已。不过随着Graal编译器的逐渐成熟，相信Jaotc前途还是可期的。

​	此外，本书虽然选择Jaotc来进行实战，但同样有发展潜力的Substrate VM也不应被忽视。Jaotc做的提前编译属于本节开头所说的“第二条分支”，即做<u>即时编译的缓存</u>；而Substrate VM则是选择的“第一条分支”，做的是传统的<u>静态提前编译</u>，关于Substrate VM的实战，建议读者自己去尝试一下。

## 11.4 编译器优化技术

​	经过前面对即时编译、提前编译的讲解，读者应该已经建立起一个认知：编译器的目标虽然是做由程序代码翻译为本地机器码的工作，但其实难点并不在于能不能成功翻译出机器码，输出代码优化质量的高低才是决定编译器优秀与否的关键。在本章之前的内容里出现过许多优化措施的专业名词，有一些是编译原理中的基础知识，譬如<u>方法内联</u>，只要是计算机专业毕业的读者至少都有初步的概念；但也有一些专业性比较强的名词，譬如<u>逃逸分析</u>，可能不少读者只听名字很难想象出来这个优化会做什么事情。本节将介绍几种HotSpot虚拟机的即时编译器在生成代码时采用的代码优化技术，以小见大，见微知著，让读者对编译器代码优化有整体理解。

### 11.4.1 优化技术概览

​	OpenJDK的官方Wiki上，HotSpot虚拟机设计团队列出了一个相对比较全面的、即时编译器中采用的优化技术列表，如表11-1所示，其中有不少经典编译器的优化手段，也有许多针对Java语言，或者说针对运行在Java虚拟机上的所有语言进行的优化。本节先对这些技术进行概览，在后面几节中，将挑选若干最重要或最典型的优化，与读者一起看看优化前后的代码发生了怎样的变化。

​	表11-1　即时编译器优化技术一览

![img](https://img2020.cnblogs.com/blog/1240544/202201/1240544-20220116152056600-1406973553.png)

![img](https://img2020.cnblogs.com/blog/1240544/202201/1240544-20220116152114753-886120889.png)

​	不过首先<u>需要明确一点，即时编译器对这些代码优化变换是建立在代码的中间表示或者是机器码之上的，绝不是直接在Java源码上去做的</u>，这里只是笔者为了方便讲解，使用了Java语言的语法来表示这些优化技术所发挥的作用。

​	第一步，从原始代码开始，如代码清单11-6所示。

​	代码清单11-6　优化前的原始代码

```java
static class B {
  int value;
  final int get() {
    return value;
  }
}
public void foo() {
  y = b.get();
  // ...do stuff...
  z = b.get();
  sum = y + z;
}
```

​	代码清单11-6所示的内容已经非常简化了，但是仍有不少优化的空间。首先，第一个要进行的优化是**方法内联**，它的主要目的有两个：一是去除方法调用的成本（如查找方法版本、建立栈帧等）；<u>二是为其他优化建立良好的基础。方法内联膨胀之后可以便于在更大范围上进行后续的优化手段，可以获取更好的优化效果</u>。因此**各种编译器一般都会把内联优化放在优化序列最靠前的位置**。内联后的代码如代码清单11-7所示。

​	代码清单11-7　内联后的代码

```java
public void foo() {
  y = b.value;
  // ...do stuff...
  z = b.value;
  sum = y + z;
}
```

​	第二步进行**冗余访问消除**（Redundant Loads Elimination），假设代码中间注释掉的“…do stuff…”所代表的操作不会改变b.value的值，那么就可以把“z=b.value”替换为“z=y”，因为上一句“y=b.value”已经保证了变量y与b.value是一致的，这样就可以不再去访问对象b的局部变量了。如果把b.value看作一个表达式，那么也可以把这项优化看作一种**公共子表达式消除**（Common Subexpression Elimination），优化后的代码如代码清单11-8所示。

​	代码清单11-8　冗余存储消除的代码

```java
public void foo() {
  y = b.value;
  // ...do stuff...
  z = y;
  sum = y + z;
}
```

​	第三步进行**复写传播**（Copy Propagation），因为这段程序的逻辑之中没有必要使用一个额外的变量z，它与变量y是完全相等的，因此我们可以使用y来代替z。复写传播之后的程序如代码清单11-9所示。

​	代码清单11-9　复写传播的代码

```java
public void foo() {
  y = b.value;
  // ...do stuff...
  y = y;
  sum = y + y;
}
```

​	第四步进行**无用代码消除**（Dead Code Elimination），无用代码可能是永远不会被执行的代码，也可能是完全没有意义的代码。因此它又被很形象地称为“Dead Code”，在代码清单11-9中，“y=y”是没有意义的，把它消除后的程序如代码清单11-10所示。

​	代码清单11-10　进行无用代码消除的代码

```java
public void foo() {
  y = b.value;
  // ...do stuff...
  sum = y + y;
}
```

​	经过四次优化之后，代码清单11-10所示代码与代码清单11-6所示代码所达到的效果是一致的，但是前者比后者省略了许多语句，体现在字节码和机器码指令上的差距会更大，执行效率的差距也会更高。<u>编译器的这些优化技术实现起来也许确实复杂，但是要理解它们的行为，对于一个初学者来说都是没有什么困难的，完全不需要有任何的恐惧心理</u>。

​	接下来，笔者挑选了四项有代表性的优化技术，与大家一起观察它们是如何运作的。它们分别是：

+ **最重要的优化技术之一：方法内联。**

+ **最前沿的优化技术之一：逃逸分析。**

+ **语言无关的经典优化技术之一：公共子表达式消除。**

+ **语言相关的经典优化技术之一：数组边界检查消除。**

### 11.4.2 方法内联

​	在前面的讲解中，我们多次提到方法内联，说它是编译器最重要的优化手段，甚至都可以不加上“之一”。**<u>内联被业内戏称为优化之母，因为除了消除方法调用的成本之外，它更重要的意义是为其他优化手段建立良好的基础</u>**，代码清单11-11所示的简单例子就揭示了内联对其他优化手段的巨大价值：没有内联，多数其他优化都无法有效进行。<u>例子里testInline()方法的内部全部是无用的代码，但如果不做内联，后续即使进行了无用代码消除的优化，也无法发现任何“Dead Code”的存在。如果分开来看，foo()和testInline()两个方法里面的操作都有可能是有意义的</u>。

​	代码清单11-11　未作任何优化的字节码

```java
public static void foo(Object obj) {
  if (obj != null) {
    System.out.println("do something");
  }
}
public static void testInline(String[] args) {
  Object obj = null;
  foo(obj);
}
```

​	方法内联的优化行为理解起来是没有任何困难的，不过就是把目标方法的代码原封不动地“复制”到发起调用的方法之中，避免发生真实的方法调用而已。但实际上Java虚拟机中的内联过程却远没有想象中容易，甚至如果不是即时编译器做了一些特殊的努力，**按照经典编译原理的优化理论，大多数的Java方法都无法进行内联（不讨论编程语言，通常只有非虚方法才可以内联）**。

​	无法内联的原因其实在第8章中讲解Java方法解析和分派调用的时候就已经解释过：**只有使用invokespecial指令调用的私有方法、实例构造器、父类方法和使用invokestatic指令调用的静态方法才会在编译期进行解析**。<u>除了上述四种方法之外（最多再除去被final修饰的方法这种特殊情况，尽管它使用invokevirtual指令调用，但也是非虚方法，《Java语言规范》中明确说明了这点），其他的Java方法调用都必须在运行时进行方法接收者的多态选择，它们都有可能存在多于一个版本的方法接收者，简而言之，Java语言中默认的实例方法是虚方法</u>。

​	**对于一个虚方法，编译器静态地去做内联的时候很难确定应该使用哪个方法版本**，<u>以将代码清单11-7中所示b.get()直接内联为b.value为例，如果不依赖上下文，是无法确定b的实际类型是什么的</u>。假如有ParentB和SubB是两个具有继承关系的父子类型，并且子类重写了父类的get()方法，那么b.get()是执行父类的get()方法还是子类的get()方法，这应该是根据实际类型动态分派的，而实际类型必须在实际运行到这一行代码时才能确定，编译器很难在编译时得出绝对准确的结论。

​	更糟糕的情况是，由于Java提倡使用面向对象的方式进行编程，而Java对象的方法默认就是虚方法，可以说Java间接鼓励了程序员使用大量的虚方法来实现程序逻辑。**根据上面的分析可知，内联与虚方法之间会产生“矛盾”，那是不是为了提高执行性能，就应该默认给每个方法都使用final关键字去修饰呢？<u>C和C++语言的确是这样做的，默认的方法是非虚方法，如果需要用到多态，就用virtual关键字来修饰，但Java选择了在虚拟机中解决这个问题</u>**。

​	**<u>为了解决虚方法的内联问题，Java虚拟机首先引入了一种名为类型继承关系分析（Class Hierarchy Analysis，CHA）的技术</u>**，这是整个应用程序范围内的类型分析技术，用于确定在目前已加载的类中，某个接口是否有多于一种的实现、某个类是否存在子类、某个子类是否覆盖了父类的某个虚方法等信息。<u>这样，编译器在进行内联时就会分不同情况采取不同的处理：如果是非虚方法，那么直接进行内联就可以了，这种的内联是有百分百安全保障的；如果遇到虚方法，则会向CHA查询此方法在当前程序状态下是否真的有多个目标版本可供选择，如果查询到只有一个版本，那就可以假设“应用程序的全貌就是现在运行的这个样子”来进行内联，这种内联被称为**守护内联（Guarded Inlining）**</u>。不过由于Java程序是动态连接的，说不准什么时候就会加载到新的类型从而改变CHA结论，因此这种内联属于激进预测性优化，必须预留好“逃生门”，即当假设条件不成立时的“退路”（Slow Path）。假如在程序的后续执行过程中，虚拟机一直没有加载到会令这个方法的接收者的继承关系发生变化的类，那这个内联优化的代码就可以一直使用下去。**如果加载了导致继承关系发生变化的新类，那么就必须抛弃已经编译的代码，退回到解释状态进行执行，或者重新进行编译**。

​	<u>假如向CHA查询出来的结果是该方法确实有多个版本的目标方法可供选择，那即时编译器还将进行最后一次努力，使用**内联缓存（Inline Cache）**的方式来缩减方法调用的开销</u>。这种状态下方法调用是真正发生了的，但是比起直接查虚方法表还是要快一些。<u>内联缓存是一个建立在目标方法正常入口之前的缓存，它的工作原理大致为：在未发生方法调用之前，内联缓存状态为空，当第一次调用发生后，缓存记录下方法接收者的版本信息，并且每次进行方法调用时都比较接收者的版本。如果以后进来的每次调用的方法接收者版本都是一样的，那么这时它就是一种单**态内联缓存（Monomorphic Inline Cache）**。通过该缓存来调用，比用不内联的非虚方法调用，仅多了一次类型判断的开销而已。但如果真的出现方法接收者不一致的情况，就说明程序用到了虚方法的多态特性，这时候会退化成**超多态内联缓存（Megamorphic Inline Cache）**，其开销相当于真正查找虚方法表来进行方法分派</u>。

​	所以说，**<u>在多数情况下Java虚拟机进行的方法内联都是一种激进优化。事实上，激进优化的应用在高性能的Java虚拟机中比比皆是，极为常见</u>**。除了方法内联之外，对于出现概率很小（通过经验数据或解释器收集到的性能监控信息确定概率大小）的隐式异常、使用概率很小的分支等都可以被激进优化“移除”，如果真的出现了小概率事件，这时才会从“逃生门”回到解释状态重新执行。

### 11.4.3 逃逸分析

​	逃逸分析（Escape Analysis）是目前Java虚拟机中比较前沿的优化技术，它与类型继承关系分析一样，<u>并不是直接优化代码的手段，而是为其他优化措施提供依据的分析技术</u>。

​	**逃逸分析的基本原理是：分析对象动态作用域，当一个对象在方法里面被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他方法中，这种称为方法逃逸；甚至还有可能被外部线程访问到，譬如赋值给可以在其他线程中访问的实例变量，这种称为线程逃逸；从不逃逸、方法逃逸到线程逃逸，称为对象由低到高的不同逃逸程度**。

​	如果能证明一个对象不会逃逸到方法或线程之外（换句话说是别的方法或线程无法通过任何途径访问到这个对象），或者逃逸程度比较低（只逃逸出方法而不会逃逸出线程），则可能为这个对象实例采取不同程度的优化，如：

+ **栈上分配（Stack Allocations）**：在Java虚拟机中，Java堆上分配创建对象的内存空间几乎是Java程序员都知道的常识，Java堆中的对象对于各个线程都是共享和可见的，只要持有这个对象的引用，就可以访问到堆中存储的对象数据。虚拟机的垃圾收集子系统会回收堆中不再使用的对象，但回收动作无论是标记筛选出可回收对象，还是回收和整理内存，都需要耗费大量资源。<u>如果确定一个对象不会逃逸出线程之外，那让这个对象在栈上分配内存将会是一个很不错的主意，对象所占用的内存空间就可以随栈帧出栈而销毁。在一般应用中，完全不会逃逸的局部对象和不会逃逸出线程的对象所占的比例是很大的，如果能使用栈上分配，那大量的对象就会随着方法的结束而自动销毁了，垃圾收集子系统的压力将会下降很多</u>。**栈上分配可以支持方法逃逸，但不能支持线程逃逸**。
+ **标量替换（Scalar Replacement）**：<u>若一个数据已经无法再分解成更小的数据来表示了，Java虚拟机中的原始数据类型（int、long等数值类型及reference类型等）都不能再进一步分解了，那么这些数据就可以被称为**标量**</u>。<u>相对的，如果一个数据可以继续分解，那它就被称为**聚合量（Aggregate）**</u>，Java中的对象就是典型的聚合量。如果把一个Java对象拆散，根据程序访问的情况，将其用到的成员变量恢复为原始类型来访问，这个过程就称为标量替换。**<u>假如逃逸分析能够证明一个对象不会被方法外部访问，并且这个对象可以被拆散，那么程序真正执行的时候将可能不去创建这个对象，而改为直接创建它的若干个被这个方法使用的成员变量来代替</u>。将对象拆分后，除了可以让对象的成员变量在栈上（栈上存储的数据，很大机会被虚拟机分配至物理机器的高速寄存器中存储）分配和读写之外，还可以为后续进一步的优化手段创建条件。标量替换可以视作栈上分配的一种特例，实现更简单（不用考虑整个对象完整结构的分配），但对逃逸程度的要求更高，它不允许对象逃逸出方法范围内**。

+ **同步消除（Synchronization Elimination）**：线程同步本身是一个相对耗时的过程，如果逃逸分析能够确定一个变量不会逃逸出线程，无法被其他线程访问，那么这个变量的读写肯定就不会有竞争，<u>对这个变量实施的同步措施也就可以安全地消除掉</u>。

​	关于逃逸分析的研究论文早在1999年就已经发表，但直到JDK 6，HotSpot才开始支持初步的逃逸分析，而且到现在这项优化技术尚未足够成熟，仍有很大的改进余地。不成熟的原因主要是逃逸分析的计算成本非常高，甚至不能保证逃逸分析带来的性能收益会高于它的消耗。如果要百分之百准确地判断一个对象是否会逃逸，需要进行一系列复杂的数据流敏感的过程间分析，才能确定程序各个分支执行时对此对象的影响。<u>前面介绍即时编译、提前编译优劣势时提到了过程间分析这种大压力的分析算法正是即时编译的弱项</u>。可以试想一下，如果逃逸分析完毕后发现几乎找不到几个不逃逸的对象，那这些运行期耗用的时间就白白浪费了，所以目前虚拟机只能采用不那么准确，但时间压力相对较小的算法来完成分析。

​	**C和C++语言里面原生就支持了栈上分配（不使用new操作符即可），而C#也支持值类型，可以很自然地做到标量替换（但并不会对引用类型做这种优化）。在灵活运用栈内存方面，确实是Java的一个弱项。在现在仍处于实验阶段的Valhalla项目里，设计了新的inline关键字用于定义Java的内联类型，目的是实现与C#中值类型相对标的功能。有了这个标识与约束，以后逃逸分析做起来就会简单很多**。

​	下面笔者将通过一系列Java伪代码的变化过程来模拟逃逸分析是如何工作的，向读者展示逃逸分析能够实现的效果。初始代码如下所示：

```java
// 完全未优化的代码
public int test(int x) {
  int xx = x + 2;
  Point p = new Point(xx, 42);
  return p.getX();
}
```

​	此处笔者省略了Point类的代码，这就是一个包含x和y坐标的POJO类型，读者应该很容易想象它的样子。

​	第一步，将Point的构造函数和getX()方法进行内联优化：

```java
// 步骤1：构造函数内联后的样子
public int test(int x) {
  int xx = x + 2;
  Point p = point_memory_alloc(); // 在堆中分配P对象的示意方法
  p.x = xx; // Point构造函数被内联后的样子
  p.y = 42
    return p.x; // Point::getX()被内联后的样子
}
```

​	第二步，经过逃逸分析，发现在整个test()方法的范围内Point对象实例不会发生任何程度的逃逸，这样可以对它进行标量替换优化，把其内部的x和y直接置换出来，分解为test()方法内的局部变量，从而避免Point对象实例被实际创建，优化后的结果如下所示：

```java
// 步骤2：标量替换后的样子
public int test(int x) {
  int xx = x + 2;
  int px = xx;
  int py = 42
    return px;
}
```

​	第三步，通过数据流分析，发现py的值其实对方法不会造成任何影响，那就可以放心地去做无效代码消除得到最终优化结果，如下所示：

```java
// 步骤3：做无效代码消除后的样子
public int test(int x) {
  return x + 2;
}
```

​	<u>从测试结果来看，实施逃逸分析后的程序在MicroBenchmarks中往往能得到不错的成绩，但是在实际的应用程序中，尤其是大型程序中反而发现实施逃逸分析可能出现效果不稳定的情况，或分析过程耗时但却无法有效判别出非逃逸对象而导致性能（即时编译的收益）下降</u>，所以曾经在很长的一段时间里，即使是服务端编译器，也默认不开启逃逸分析，甚至在某些版本（如JDK 6 Update 18）中还曾经完全禁止了这项优化，**一直到JDK 7时这项优化才成为服务端编译器默认开启的选项**。如果有需要，或者确认对程序运行有益，用户也可以使用参数`-XX:+DoEscapeAnalysis`来手动开启逃逸分析，开启之后可以通过参数`-XX:+PrintEscapeAnalysis`来查看分析结果。有了逃逸分析支持之后，用户可以使用参数`-XX:+EliminateAllocations`来开启标量替换，使用`-XX:+EliminateLocks`来开启同步消除，使用参数`-XX:+PrintEliminateAllocations`查看标量的替换情况。

​	尽管目前逃逸分析技术仍在发展之中，未完全成熟，但它是即时编译器优化技术的一个重要前进方向，在日后的Java虚拟机中，逃逸分析技术肯定会支撑起一系列更实用、有效的优化技术。

### 11.4.4 公共子表达式消除

​	公共子表达式消除是一项非常经典的、普遍应用于各种编译器的优化技术，它的含义是：**如果一个表达式E之前已经被计算过了，并且从先前的计算到现在E中所有变量的值都没有发生变化，那么E的这次出现就称为公共子表达式**。对于这种表达式，没有必要花时间再对它重新进行计算，只需要直接用前面计算过的表达式结果代替E。

+ 如果这种优化仅限于程序基本块内，便可称为**局部公共子表达式消除（Local Common Subexpression Elimination）**
+ 如果这种优化的范围涵盖了多个基本块，那就称为**全局公共子表达式消除（Global Common Subexpression Elimination）**

​	下面举个简单的例子来说明它的优化过程，假设存在如下代码：

```java
int d = (c * b) * 12 + a + (a + b * c);
```

​	**如果这段代码交给Javac编译器则不会进行任何优化**，那生成的代码将如代码清单11-12所示，是完全遵照Java源码的写法直译而成的。

​	代码清单11-12　未作任何优化的字节码

```shell
iload_2 // b
imul // 计算b*c
bipush 12 // 推入12
imul // 计算(c * b) * 12
iload_1 // a
iadd // 计算(c * b) * 12 + a
iload_1 // a
iload_2 // b
iload_3 // c
imul // 计算b * c
iadd // 计算a + b * c
iadd // 计算(c * b) * 12 + a + a + b * c
istore 4
```

​	当这段代码进入虚拟机即时编译器后，它将进行如下优化：编译器检测到`c*b`与`b*c`是一样的表达式，而且在计算期间b与c的值是不变的。

​	因此这条表达式就可能被视为：

```java
int d = E * 12 + a + (a + E);
```

​	这时候，编译器还可能（取决于哪种虚拟机的编译器以及具体的上下文而定）进行另外一种优化——**代数化简（Algebraic Simplification）**，在E本来就有乘法运算的前提下，把表达式变为：

```java
int d = E * 13 + a + a;
```

​	表达式进行变换之后，再计算起来就可以节省一些时间了。如果读者还对其他的经典编译优化技术感兴趣，可以参考《编译原理》（俗称龙书）中的相关章节。

### 11.4.5 数组边界检查消除

​	**数组边界检查消除（Array Bounds Checking Elimination）**是即时编译器中的一项语言相关的经典优化技术。我们知道<u>Java语言是一门动态安全的语言，对数组的读写访问也不像C、C++那样实质上就是裸指针操作</u>。如果有一个数组`foo[]`，在Java语言中访问数组元素`foo[i]`的时候系统将会自动进行上下界的范围检查，即i必须满足“`i>=0&&i<foo.length`”的访问条件，否则将抛出一个运行时异常：java.lang.ArrayIndexOutOfBoundsException。这对软件开发者来说是一件很友好的事情，即使程序员没有专门编写防御代码，也能够避免大多数的溢出攻击。但是**对于虚拟机的执行子系统来说，每次数组元素的读写都带有一次隐含的条件判定操作，对于拥有大量数组访问的程序代码，这必定是一种性能负担**。

​	无论如何，为了安全，数组边界检查肯定是要做的，但数组边界检查是不是必须在运行期间一次不漏地进行则是可以“商量”的事情。例如下面这个简单的情况：数组下标是一个常量，如`foo[3]`，只要在编译期根据数据流分析来确定foo.length的值，并判断下标“3”没有越界，执行的时候就无须判断了。更加常见的情况是，数组访问发生在循环之中，并且使用循环变量来进行数组的访问。<u>如果编译器只要通过数据流分析就可以判定循环变量的取值范围永远在区间`[0，foo.length)`之内，那么在循环中就可以把整个数组的上下界检查消除掉，这可以节省很多次的条件判断操作</u>。

​	把这个数组边界检查的例子放在更高的视角来看，大量的安全检查使编写Java程序比编写C和C++程序容易了很多，比如：数组越界会得到ArrayIndexOutOfBoundsException异常；空指针访问会得到NullPointException异常；除数为零会得到ArithmeticException异常……在C和C++程序中出现类似的问题，一个不小心就会出现Segment Fault信号或者Windows编程中常见的“XXX内存不能为Read/Write”之类的提示，处理不好程序就直接崩溃退出了。但这些安全检查也导致出现相同的程序，从而使Java比C和C++要做更多的事情（各种检查判断），这些事情就会导致一些隐式开销，如果不处理好它们，就很可能成为一项“Java语言天生就比较慢”的原罪。<u>为了消除这些隐式开销，除了如数组边界检查优化这种尽可能把运行期检查提前到编译期完成的思路之外，还有一种避开的处理思路——**隐式异常处理**，Java中空指针检查和算术运算中除数为零的检查都采用了这种方案</u>。举个例子，程序中访问一个对象（假设对象叫foo）的某个属性（假设属性叫value），那以Java伪代码来表示虚拟机访问foo.value的过程为：

```java
if (foo != null) {
  return foo.value;
}else{
  throw new NullPointException();
}
```

​	在使用隐式异常优化之后，虚拟机会把上面的伪代码所表示的访问过程变为如下伪代码：

```java
try {
  return foo.value;
} catch (segment_fault) {
  uncommon_trap();
}
```

​	<u>虚拟机会注册一个Segment Fault信号的异常处理器（伪代码中的uncommon_trap()，务必注意这里是指**进程层面的异常处理器**，并非真的Java的try-catch语句的异常处理器），这样当foo不为空的时候，对value的访问是不会有任何额外对foo判空的开销的，而代价就是当foo真的为空时，必须转到**异常处理器中恢复中断**并抛出NullPointException异常</u>。**进入异常处理器的过程涉及进程从用户态转到内核态中处理的过程，结束后会再回到用户态，速度远比一次判空检查要慢得多**。<u>当foo极少为空的时候，隐式异常优化是值得的，但假如foo经常为空，这样的优化反而会让程序更慢。幸好HotSpot虚拟机足够聪明，它会根据运行期收集到的性能监控信息自动选择最合适的方案</u>。

​	与语言相关的其他消除操作还有不少，如**自动装箱消除（Autobox Elimination）**、**安全点消除（Safepoint Elimination）**、**消除反射（Dereflection）**等，这里就不再一一介绍了。

## 11.5 实战: 深入理解Graal编译器

​	在本书刚开始介绍HotSpot即时编译器的时候曾经说过，从JDK 10起，HotSpot就同时拥有三款不同的即时编译器。此前我们已经介绍了经典的客户端编译器和服务端编译器，在本节，我们将把目光聚焦到**HotSpot即时编译器以及提前编译器共同的最新成果——Graal编译器**身上。

### 11.5.1 历史背景

> [jdk14之jdk工具——jaotc命令_早睡的叶子的博客-CSDN博客](https://blog.csdn.net/sexyluna/article/details/106533743)

​	在第1章展望Java技术的未来时，我们就听说过Graal虚拟机以及Graal编译器仍在实验室中尚未商用，但未来其有望代替或成为HotSpot下一代技术基础。Graal编译器最初是在Maxine虚拟机中作为C1X编译器的下一代编译器而设计的，所以它理所当然地使用于Java语言来编写。2012年，Graal编译器从Maxine虚拟机项目中分离，成为一个独立发展的Java编译器项目，<u>Oracle Labs希望它最终能够成为一款高编译效率、高输出质量、支持提前编译和即时编译，同时支持应用于包括HotSpot在内的不同虚拟机的编译器</u>。由于这个编译器使用Java编写，代码清晰，又继承了许多来自HotSpot的服务端编译器的高质量优化技术，所以无论是科技企业还是高校研究院，都愿意在它上面研究和开发新编译技术。HotSpot服务端编译器的创造者Cliff Click自己就对Graal编译器十分推崇，并且公开表示再也不会用C、C++去编写虚拟机和编译器了。Twitter的Java虚拟机团队也曾公开说过C2目前犹如一潭死水，亟待一个替代品，因为在它上面开发、改进实在太困难了。

​	**Graal编译器在JDK 9时以Jaotc提前编译工具的形式首次加入到官方的JDK中，从JDK 10起，Graal编译器可以替换服务端编译器，成为HotSpot分层编译中最顶层的即时编译器。这种可替换的即时编译器架构的实现，得益于HotSpot编译器接口的出现**。

​	<u>早期的Graal曾经同C1及C2一样，与HotSpot的协作是紧耦合的，这意味着每次编译Graal均需重新编译整个HotSpot。JDK 9时发布的JEP 243：**Java虚拟机编译器接口（Java-Level JVM Compiler Interface，JVMCI）**使得Graal可以从HotSpot的代码中分离出来</u>。

​	JVMCI主要提供如下三种功能：

+ 响应HotSpot的编译请求，并将该请求分发给Java实现的即时编译器。

+ 允许编译器访问HotSpot中与即时编译相关的数据结构，包括类、字段、方法及其性能监控数据等，并提供了一组这些数据结构在Java语言层面的抽象表示。

+ 提供HotSpot代码缓存（Code Cache）的Java端抽象表示，允许编译器部署编译完成的二进制机器码。

​	综合利用上述三项功能，我们就可以把一个在HotSpot虚拟机外部的、用Java语言实现的即时编译器（不局限于Graal）集成到HotSpot中，响应HotSpot发出的最顶层的编译请求，并将编译后的二进制代码部署到HotSpot的代码缓存中。<u>此外，单独使用上述第三项功能，又可以绕开HotSpot的即时编译系统，让该编译器直接为应用的类库编译出二进制机器码，将该编译器当作一个提前编译器去使用（如Jaotc）</u>。

​	Graal和JVMCI的出现，为不直接从事Java虚拟机和编译器开发，但对Java虚拟机技术充满好奇心的读者们提供一条窥探和尝试编译器技术的良好途径，现在我们就将开始基于Graal来实战HotSpot虚拟机的即时编译与代码优化过程。

### 11.5.2 构建编译调试环境

​	跟着书步骤走

### 11.5.3 JVMCI编译器接口

​	现在请读者来思考一下，如果让您来设计JVMCI编译器接口，它应该是怎样的？既然JVMCI面向的是Java语言的编译器接口，那它至少在形式上是与我们已经见过无数次的Java接口是一样的。我们来考虑即时编译器的输入是什么。答案当然是要编译的方法的字节码。既然叫字节码，顾名思义它就应该是“用一个字节数组表示的代码”。那接下来它输出什么？这也很简单，即时编译器应该输出与方法对应的二进制机器码，二进制机器码也应该是“用一个字节数组表示的代码”。这样的话，JVMCI接口就应该看起来类似于下面这种样子：

```java
interface JVMCICompiler {
  byte[] compileMethod(byte[] bytecode);
}
```

​	事实上JVMCI接口只比上面这个稍微复杂一点点，因为其输入除了字节码外，HotSpot还会向编译器提供各种该方法的相关信息，譬如局部变量表中变量槽的个数、<u>操作数栈的最大深度</u>，还有分层编译在底层收集到的统计信息等。因此JVMCI接口的核心内容实际就是代码清单11-13总所示的这些。

​	代码清单11-13　JVMCI接口

```java
interface JVMCICompiler {
  void compileMethod(CompilationRequest request);
}
interface CompilationRequest {
  JavaMethod getMethod();
}
interface JavaMethod {
  byte[] getCode();
  int getMaxLocals();
  int getMaxStackSize();
  ProfilingInfo getProfilingInfo();
  ... // 省略其他方法
}
```

​	我们在Eclipse中找到JVMCICompiler接口，通过继承关系分析，可以清楚地看到有一个实现类HotSpotGraalCompiler实现了JVMCI，如图11-12所示，这个就是我们要分析的代码的入口。

![在这里插入图片描述](https://img-blog.csdnimg.cn/0b95c6ce45894da68f3b15237856a9ec.png)

​	图11-12　JVMCI接口的继承关系

​	为了后续调试方便，我们先准备一段简单的代码，并让它触发HotSpot的即时编译，以便我们跟踪观察编译器是如何工作对的。具体代码如清单11-14所示。

​	代码清单11-14　触发即时编译的示例代码

```java
public class Demo {
  public static void main(String[] args) {
    while (true) {
      workload(14, 2);
    }
  }
  private static int workload(int a, int b) {
    return a + b;
  }
}
```

​	由于存在无限循环，workload()方法肯定很快就会被虚拟机发现是热点代码因而进行编译。实际上除了workload()方法以外，这段简单的代码还会导致相当多的其他方法的编译，因为一个最简单的Java类的加载和运行也会触发数百个类的加载。为了避免干扰信息太多，笔者加入了参数`-XX:CompileOnly`来限制只允许workload()方法被编译。先采用以下命令，用标准的服务端编译器来运行清单11-14中所示的程序。

```shell
$ javac Demo.java
$ java \
-XX:+PrintCompilation \
-XX:CompileOnly=Demo::workload \
Demo
...
193 1 3 Demo::workload (4 bytes)
199 2 1 Demo::workload (4 bytes)
199 1 3 Demo::workload (4 bytes) made not entrant
...
```

​	上面显示wordload()方法确实被分层编译了多次，“made not entrant”的输出就表示了方法的某个已编译版本被丢弃过。从这段信息中我们清楚看到，分层编译机制及最顶层的服务端编译都已经正常工作了，下一步就是用我们在Eclipse中的Graal编译器代替HotSpot的服务端编译器。

​	为简单起见，笔者加上`-XX:-TieredCompilation`关闭分层编译，让虚拟机只采用有一个JVMCI编译器而不是由客户端编译器和JVMCI混合分层。然后使用参数`-XX:+EnableJVMCI`、`-XX:+UseJVMCICompiler`来启用JVMCI接口和JVMCI编译器。由于这些目前尚属实验阶段的功能，需要再使用`-XX:+UnlockExperimentalVMOptions`参数进行解锁。最后，也是最关键的一个问题，如何让HotSpot找到Graal编译器的位置呢？

​	如果采用特殊版的JDK 8，那虚拟机将会自动去查找JAVA_HOME/jre/lib/jvmci目录。假如这个目录不存在，那就会从`-Djvmci.class.path.append`参数中搜索。它查找的目标，即Graal编译器的JAR包，刚才我们已经通过`mx build`命令成功编译出来，所以在JDK 8下笔者使用的启动参数如代码清单11-15所示。

​	代码清单11-15　JDK8的运行配置

```shell
-Djvmci.class.path.append=~/graal/compiler/mxbuild/dists/jdk1.8/graal.jar:~/graal/sdk/mxbuild/dists/jdk1.8/graal--XX:+UnlockExperimentalVMOptions
-XX:+EnableJVMCI
-XX:+UseJVMCICompiler
-XX:-TieredCompilation
-XX:+PrintCompilation
-XX:CompileOnly=Demo::workload
```

​	如果读者采用JDK 9或以上版本，那原本的Graal编译器是实现在jdk.internal.vm.compiler模块中的，我们只要用`--upgrade-module-path`参数指定这个模块的升级包即可，具体如代码清单11-16所示。

​	代码清单11-16　JDK 9或以上版本的运行配置

```shell
--module-path=~/graal/sdk/mxbuild/dists/jdk11/graal.jar
--upgrade-module-path=~graal/compiler/mxbuild/dists/jdk11/jdk.internal.vm.compiler.jar
-XX:+UnlockExperimentalVMOptions
-XX:+EnableJVMCI
-XX:+UseJVMCICompiler
-XX:-TieredCompilation
-XX:+PrintCompilation
-XX:CompileOnly=Demo::workload
```

​	通过上述参数，HotSpot就能顺利找到并应用我们编译的Graal编译器了。为了确认效果，我们对HotSpotGraalCompiler类的compileMethod()方法做一个简单改动，输出编译的方法名称和编译耗时，具体如下（黑色加粗代码是笔者在源码中额外添加的内容）：

```java
public CompilationRequestResult compileMethod(CompilationRequest request) {
  long time = System.currentTimeMillis();
  CompilationRequestResult result = compileMethod(request, true, graalRuntime.getOptions());
  System.out.println("compile method:" + request.getMethod().getName());
  System.out.println("time used:" + (System.currentTimeMillis() - time));
  return result;
}
```

​	在Eclipse里面运行这段代码，不需要重新运行mx build，马上就可以看到类似如下所示的输出结果：

```shell
97 1 Demo::workload (4 bytes)
……
compile method:workload
time used:4081
```

### 11.5.4 代码中间表示

> 建议这部分直接看书，主要描述构造理想图。

​	Graal编译器在设计之初就刻意采用了与HotSpot服务端编译器一致（略有差异但已经非常接近）的中间表示形式，也即是被称为Sea-of-Nodes的中间表示，或者与其等价的被称为**理想图（IdealGraph，在代码中称为Structured Graph）的程序依赖图（Program Dependence Graph，PDG）形式**。在11.2节即时编译器的实战中，我们已经通过可视化工具Ideal Graph Visualizer看到过在理想图上翻译和优化输入代码的整体过程，<u>从编译器内部来看即：字节码→理想图→优化→机器码（以Mach NodeGraph表示）的转变过程</u>。在那个实战里面，我们着重分析的是理想图转换优化的整体过程，对于多数读者，尤其是不熟悉编译原理与编译器设计的读者，可能会不太容易读懂每个阶段所要做的工作。

​	理想图是一种有向图，用节点来表示程序中的元素，譬如变量、操作符、方法、字段等，而用边来表示数据或者控制流。

​	...

​	代码清单11-17　公共子表达式被消除的应用范围

```java
// 以下代码的公共子表达式能够被消除
int workload(int a, int b) {
  return (a + b) * (a + b);
}
// 以下代码的公共子表达式是不可以被消除的
int workload() {
  return (getA() + getB()) * (getA() + getB());
}
```

​	对于第一段代码，a+b是公共子表达式，可以通过优化使其只计算一次而不会有任何的副作用。但是<u>对于第二段代码，由于getA()和getB()方法内部所蕴含的操作是不确定的，它是否被调用、调用次数的不同都可能会产生不同返回值或者其他影响程序状态的副作用（譬如改变某个全局的状态变量），这种代码只能内联了getA()和getB()方法之后才能考虑更进一步的优化措施，仍然保持函数调用的情况下是无法做公共子表达式消除的</u>。

### 11.5.5 代码优化与生成

> 建议直接看书

​	相信读者现在已经能够基本看明白Graal理想图的中间表示了，那对应到代码上，Graal编译器是如何从字节码生成理想图？又如何在理想图基础上进行代码优化的呢？这时候就充分体现出了Graal编译器在使用Java编写时对普通Java程序员来说具有的便捷性了，在Outline视图中找到创建理想图的方法是greateGraph()，我们可以从Call Hierarchy视图中轻易地找到从JVMCI的入口方法compileMethod()到greateGraph()之间的调用关系。

​	greateGraph()方法的代码也很清晰，里面调用了StructuredGraph::Builder()构造器来创建理想图。

​	第一是理想图本身的数据结构。它是一组不为空的节点的集合，它的节点都是用ValueNode的不同类型的子类节点来表示的。仍然以x+y表达式为例，譬如其中的加法操作，就由AddNode节点来表示，从图11-19所示的Type Hierarchy视图中可以清楚地看到加法操作是二元算术操作节点（BinaryArithmeticNode`<OP>`）的一种，而二元算术操作节点又是二元操作符（BinaryNode）的一种，以此类推直到所有操作符的共同父类ValueNode（表示可以返回数据的节点）。

​	第二就是如何从字节码转换到理想图。该过程被封装在BytecodeParser类中，这个解析器我们可以按照字节码解释器的思路去理解它。如果这真的是一个字节码解释器，执行一个整数加法操作，按照《Java虚拟机规范》所定义的iadd操作码的规则，应该从栈帧中出栈两个操作数，然后相加，再将结果入栈。而从BytecodeParser::genArithmeticOp()方法上我们可以看到，其实现与规则描述没有什么差异，如图11-20所示。

....

​	其中，genIntegerAdd()方法中就只有一行代码，即调用AddNode节点的create()方法，将两个操作数作为参数传入，创建出AddNode节点，如下所示：

```java
protected ValueNode genIntegerAdd(ValueNode x, ValueNode y) {
  return AddNode.create(x, y, NodeView.DEFAULT);
}
```

​	每一个理想图的节点都有两个共同的主要操作，一个**是规范化（Canonicalisation）**，另一个是**生成机器码（Generation）**。生成机器码顾名思义，就不必解释了，<u>规范化则是指如何缩减理想图的规模，也即在理想图的基础上优化代码所要采取的措施</u>。这两个操作对应了编译器两项最根本的任务：**代码优化与代码翻译**。

​	AddNode节点的规范化是实现在canonical()方法中的，机器码生成则是实现在generate()方法中的，从AddNode的创建方法上可以看到，在节点创建时会调用canonical()方法尝试进行规范化缩减图的规模，如下所示：

```java
public static ValueNode create(ValueNode x, ValueNode y, NodeView view) {
  BinaryOp<Add> op = ArithmeticOpTable.forStamp(x.stamp(view)).getAdd();
  Stamp stamp = op.foldStamp(x.stamp(view), y.stamp(view));
  ConstantNode tryConstantFold = tryConstantFold(op, x, y, stamp, view);
  if (tryConstantFold != null) {
    return tryConstantFold;
  }
  if (x.isConstant() && !y.isConstant()) {
    return canonical(null, op, y, x, view);
  } else {
    return canonical(null, op, x, y, view);
  }
}
```

​	从AddNode的canonical()方法中我们可以看到为了缩减理想图的规模而做的相当多的努力，即使只是两个整数相加那么简单的操作，也尝试过了**常量折叠（如果两个操作数都为常量，则直接返回一个常量节点）**、**算术聚合（聚合树的常量子节点，譬如将(a+1)+2聚合为a+3）**、**符号合并（聚合树的相反符号子节点，譬如将(a-b)+b或者b+(a-b)直接合并为a）**等多种优化，canonical()方法的内容较多，请读者自行参考源码，为节省版面这里就不贴出了。

​	**对理想图的规范化并不局限于单个操作码的局部范围之内，很多的优化都是要立足于全局来进行的**，这类操作在CanonicalizerPhase类中完成。仍然以上一节的公共子表达式消除为例，这就是一个全局性的优化，实现在CanonicalizerPhase::tryGlobalValueNumbering()方法中，其逻辑看起来已经非常清晰了：如果理想图中发现了可以进行消除的算术子表达式，那就找出重复的节点，然后替换、删除。具体代码如下所示：

```java
public boolean tryGlobalValueNumbering(Node node, NodeClass<?> nodeClass) {
  if (nodeClass.valueNumberable()) {
    Node newNode = node.graph().findDuplicate(node);
    if (newNode != null) {
      assert !(node instanceof FixedNode || newNode instanceof FixedNode);
      node.replaceAtUsagesAndDelete(newNode);
      COUNTER_GLOBAL_VALUE_NUMBERING_HITS.increment(debug);
      debug.log("GVN applied and new node is %1s", newNode);
      return true;
    }
  }
  return false;
}
```

​	**至于代码生成，Graal并不是直接由理想图转换到机器码，而是和其他编译器一样，会先生成低级中间表示（LIR，与具体机器指令集相关的中间表示），然后再由HotSpot统一后端来产生机器码**。譬如涉及算术运算加法的操作，就在ArithmeticLIRGeneratorTool接口的emitAdd()方法里完成。从低级中间表示的实现类上，我们可以看到Graal编译器能够支持的目标平台，目前它只提供了三种目标平台的指令集（SPARC、x86-AMD64、ARMv8-AArch64）的低级中间表示，所以现在Graal编译器也就只能支持这几种目标平台，如图11-21所示。

​	为了验证代码阅读的成果，现在我们来对AddNode的代码生成做一些小改动，将原本生成加法汇编指令修改为生成减法汇编指令，即按如下方式修改AddNode::generate()方法：

```java
class AddNode {
  void generate(...) {
    ... gen.emitSub(op1, op2, false) ... // 原来这个方法是emitAdd()
  }
}
```

​	然后在虚拟机运行参数中加上`-XX:+PrintAssembly`参数，因为**从低级中间表示到真正机器码的转换是由HotSpot统一负责的**，所以11.2节中用到的HSDIS插件仍然能发挥作用，帮助我们输出汇编代码。从输出的汇编中可以看到，在没有修改之前，AddNode节点输出的汇编代码如下所示：

```assembly
0x000000010f71cda0: nopl 0x0(%rax,%rax,1)
0x000000010f71cda5: add %edx,%esi ;*iadd {reexecute=0 rethrow=0 return_oop=0}
; - Demo::workload@2 (line 10)
0x000000010f71cda7: mov %esi,%eax ;*ireturn {reexecute=0 rethrow=0 return_oop=0}
; - Demo::workload@3 (line 10)
0x000000010f71cda9: test %eax,-0xcba8da9(%rip) # 0x0000000102b74006
; {poll_return}
0x000000010f71cdaf: vzeroupper
0x000000010f71cdb2: retq
```

​	而被我们修改后，编译的结果已经变为：

```assembly
0x0000000107f451a0: nopl 0x0(%rax,%rax,1)
0x0000000107f451a5: sub %edx,%esi ;*iadd {reexecute=0 rethrow=0 return_oop=0}
; - Demo::workload@2 (line 10)
0x0000000107f451a7: mov %esi,%eax ;*ireturn {reexecute=0 rethrow=0 return_oop=0}
; - Demo::workload@3 (line 10)
0x0000000107f451a9: test %eax,-0x1db81a9(%rip) # 0x000000010618d006
; {poll_return}
0x0000000107f451af: vzeroupper
0x0000000107f451b2: retq
```

​	我们的修改确实促使Graal编译器产生了不同的汇编代码，这也印证了我们代码分析的思路是正确的。写到这里，笔者忍不住感慨，Graal编译器的出现对学习和研究虚拟机代码编译技术实在有着不可估量的价值。在本书第2版编写时，只有C++编写的复杂无比的服务端编译器，要进行类似的实战是非常困难的，即使勉强写出来，也会因为过度烦琐而失去阅读价值。

## 11.6 本章小结

​	在本章中，我们学习了与提前编译和即时编译器两大后端编译器相关的知识，了解了提前编译器重新兴起的原因及其优劣势；还有与即时编译器相关的热点探测方法、编译触发条件及如何从虚拟机外部观察和分析即时编译的数据和结果；还选择了几种常见的编译器优化技术进行讲解，<u>对Java编译器的深入了解，有助于在工作中分辨哪些代码是编译器可以帮我们处理的，哪些代码需要自己调节以便更适合编译器的优化</u>。

# 第五部分 高效并发

# 12. Java内存模型与线程

​	并发处理的广泛应用是Amdahl定律代替摩尔定律成为计算机性能发展源动力的根本原因，也是人类压榨计算机运算能力的最有力武器。

> Amdahl定律通过系统中并行化与串行化的比重来描述多处理器系统能获得的运算加速能力，摩尔定律则用于描述处理器晶体管数量与运行效率之间的发展关系。这两个定律的更替代表了近年来硬件发展从追求处理器频率到追求多核心并行处理的发展过程。

## 12.1 概述

## 12.2 硬件的效率与一致性

​	基于高速缓存的存储交互很好地解决了处理器与内存速度之间的矛盾，但是也为计算机系统带来更高的复杂度，它引入了一个新的问题：**缓存一致性（Cache Coherence）**。在多路处理器系统中，每个处理器都有自己的高速缓存，而它们又共享同一主内存（Main Memory），这种系统称为**共享内存多核系统（Shared Memory Multiprocessors System）**，如图12-1所示。当多个处理器的运算任务都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致。如果真的发生这种情况，那同步回到主内存时该以谁的缓存数据为准呢？为了解决一致性的问题，需要各个处理器访问缓存时都遵循一些协议，在读写时要根据协议来进行操作，这类协议有MSI、MESI（Illinois Protocol）、MOSI、Synapse、Firefly及Dragon Protocol等。<u>从本章开始，我们将会频繁见到“内存模型”一词，它可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象</u>。不同架构的物理机器可以拥有不一样的内存模型，而**Java虚拟机也有自己的内存模型，并且与这里介绍的内存访问操作及硬件的缓存访问操作具有高度的可类比性**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20a4552da61040329110c7b177913d03.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5byg57Sr5aiD,size_20,color_FFFFFF,t_70,g_se,x_16)

​	除了增加高速缓存之外，为了使处理器内部的运算单元能尽量被充分利用，处理器可能会对输入代码进行**乱序执行（Out-Of-Order Execution）优化**，处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致，因此如果存在一个计算任务依赖另外一个计算任务的中间结果，那么其顺序性并不能靠代码的先后顺序来保证。<u>与处理器的乱序执行优化类似，Java虚拟机的即时编译器中也有**指令重排序（Instruction Reorder）优化**</u>。

## 12.3 Java内存模型

​	《Java虚拟机规范》中曾试图定义一种**“Java内存模型”（Java Memory Model，JMM）**来屏蔽各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。<u>在此之前，主流程序语言（如C和C++等）直接使用物理硬件和操作系统的内存模型。因此，由于不同平台上内存模型的差异，有可能导致程序在一套平台上并发完全正常，而在另外一套平台上并发访问却经常出错，所以在某些场景下必须针对不同的平台来编写程序</u>。

​	定义Java内存模型并非一件容易的事情，这个模型必须定义得足够严谨，才能让Java的并发内存访问操作不会产生歧义；但是<u>也必须定义得足够宽松，使得虚拟机的实现能有足够的自由空间去利用硬件的各种特性（寄存器、高速缓存和指令集中某些特有的指令）来获取更好的执行速度</u>。经过长时间的验证和修补，直至JDK 5（实现了JSR-133）发布后，Java内存模型才终于成熟、完善起来了。

### 12.3.1 主内存与工作内存

​	**Java内存模型的主要目的是定义程序中各种变量的访问规则，即关注在虚拟机中把变量值存储到内存和从内存中取出变量值这样的底层细节**。此处的变量（Variables）与Java编程中所说的变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但是不包括局部变量与方法参数，因为后者是线程私有的[^1]，不会被共享，自然就不会存在竞争问题。为了获得更好的执行效能，Java内存模型并没有限制执行引擎使用处理器的特定寄存器或缓存来和主内存进行交互，也没有限制即时编译器是否要进行调整代码执行顺序这类优化措施。

​	Java内存模型规定了所有的变量都存储在**主内存（Main Memory）**中（此处的主内存与介绍物理硬件时提到的主内存名字一样，两者也可以类比，但物理上它仅是虚拟机内存的一部分）。每条线程还有自己的**工作内存（Working Memory，可与前面讲的处理器高速缓存类比）**，<u>线程的工作内存中保存了被该线程使用的变量的主内存副本[^2]，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的数据(volatile也不例外)[^3]</u>。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如图12-2所示，注意与图12-1进行对比。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6638e3110d7f494ab441627f224bb562.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5byg57Sr5aiD,size_20,color_FFFFFF,t_70,g_se,x_16)

​	这里所讲的主内存、工作内存与第2章所讲的Java内存区域中的Java堆、栈、方法区等并不是同一个层次的对内存的划分，这两者基本上是没有任何关系的。如果两者一定要勉强对应起来，那么从变量、主内存、工作内存的定义来看，主内存主要对应于Java堆中的对象实例数据部分[^4]，而工作内存则对应于虚拟机栈中的部分区域。从更基础的层次上说，主内存直接对应于物理硬件的内存，而<u>为了获取更好的运行速度，虚拟机（或者是硬件、操作系统本身的优化措施）可能会让工作内存优先存储于寄存器和高速缓存中，因为程序运行时主要访问的是工作内存</u>。

[^1]: 此处请读者注意区分概念：如果局部变量是一个reference类型，它引用的对象在Java堆中可被各个线程共享，但是reference本身在Java栈的局部变量表中是线程私有的。
[^2]: 有部分读者会对这段描述中的“副本”提出疑问，如“假设线程中访问一个10MB大小的对象，也会把这10MB的内存复制一份出来吗？”，事实上并不会如此，这个对象的引用、对象中某个在线程访问到的字段是有可能被复制的，但不会有虚拟机把整个对象复制一次。
[^3]: 根据《Java虚拟机规范》的约定，volatile变量依然有工作内存的拷贝，但是由于它特殊的操作顺序性规定（后文会讲到），所以看起来如同直接在主内存中读写访问一般，因此这里的描述对于volatile也并不存在例外。
[^4]:除了实例数据，Java堆还保存了对象的其他信息，对于HotSpot虚拟机来讲，有Mark Word（存储对象哈希码、GC标志、GC年龄、同步锁等信息）、Klass Point（指向存储类型元数据的指针）及一些用于字节对齐补白的填充数据（如果实例数据刚好满足8字节对齐，则可以不存在补白）。

### 12.3.2 内存间交互操作

​	关于主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存这一类的实现细节，Java内存模型中定义了以下8种操作来完成。<u>Java虚拟机实现时必须保证下面提及的每一种操作都是**原子的**、不可再分的（对于double和long类型的变量来说，load、store、read和write操作在某些平台上允许有例外，这个问题在12.3.4节会专门讨论）</u>[^5]。

+ lock（锁定）：作用于主内存的变量，它把一个变量标识为一条<u>线程独占</u>的状态。

+ unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，<u>释放后的变量才可以被其他线程锁定</u>。

+ read（读取）：作用于主内存的变量，它<u>把一个变量的值从主内存传输到线程的工作内存中</u>，以便随后的load动作使用。

+ load（载入）：作用于工作内存的变量，它<u>把read操作从主内存中得到的变量值放入工作内存的变量副本中</u>。

+ use（使用）：作用于工作内存的变量，它<u>把工作内存中一个变量的值传递给执行引擎</u>，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作。

+ assign（赋值）：作用于工作内存的变量，它<u>把一个从执行引擎接收的值赋给工作内存的变量</u>，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。

+ store（存储）：作用于工作内存的变量，它<u>把工作内存中一个变量的值传送到主内存中</u>，以便随后的write操作使用。

+ write（写入）：作用于主内存的变量，它<u>把store操作从工作内存中得到的变量的值放入主内存的变量</u>中。

​	**如果要把一个变量从主内存拷贝到工作内存，那就要按顺序执行read和load操作，如果要把变量从工作内存同步回主内存，就要按顺序执行store和write操作**。<u>注意，Java内存模型只要求上述两个操作必须按顺序执行，但不要求是连续执行。也就是说read与load之间、store与write之间是可插入其他指令的，如对主内存中的变量a、b进行访问时，一种可能出现的顺序是read a、read b、load b、load a</u>。

​	除此之外，**Java内存模型还规定了在执行上述8种基本操作时必须满足如下规则**：

+ **不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者工作内存发起回写了但主内存不接受的情况出现**。

+ 不允许一个线程丢弃它最近的assign操作，即**变量在工作内存中改变了之后必须把该变化同步回主内存**。

+ **<u>不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中</u>**。

+ **一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use、store操作之前，必须先执行assign和load操作**。

+ 一个变量在同一个时刻只允许一条线程对其进行lock操作，但**lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁**。

+ **<u>如果对一个变量执行lock操作，那将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作以初始化变量的值</u>**。

+ <u>如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量</u>。

+ **<u>对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）</u>**。

​	这8种内存访问操作以及上述规则限定，再加上稍后会介绍的专门针对volatile的一些特殊规定，就已经能准确地描述出Java程序中哪些内存访问操作在并发下才是安全的。这种定义相当严谨，但也是极为烦琐，实践起来更是无比麻烦。可能部分读者阅读到这里已经对多线程开发产生恐惧感了，**后来Java设计团队大概也意识到了这个问题，将Java内存模型的操作简化为read、write、lock和unlock四种，但这只是语言描述上的等价化简，Java内存模型的基础设计并未改变，即使是这四操作种，对于普通用户来说阅读使用起来仍然并不方便**。不过读者对此无须过分担忧，除了进行虚拟机开发的团队外，大概没有其他开发人员会以这种方式来思考并发问题，我们只需要理解Java内存模型的定义即可。12.3.6节将介绍这种定义的一个等效判断原则——**先行发生原则**，用来确定一个操作在并发环境下是否安全的。

[^5]:基于理解难度和严谨性考虑，最新的JSR-133文档中，已经放弃了采用这8种操作去定义Java内存模型的访问协议，缩减为4种（仅是描述方式改变了，Java内存模型并没有改变）。

### 12.3.3 对于volatile型变量的特殊规则

​	关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制，但是它并不容易被正确、完整地理解，以至于许多程序员都习惯去避免使用它，遇到需要处理多线程数据竞争问题的时候一律使用synchronized来进行同步。了解volatile变量的语义对后面理解多线程操作的其他特性很有意义，在本节中我们将多花费一些篇幅介绍volatile到底意味着什么。

​	Java内存模型为volatile专门定义了一些特殊的访问规则，在介绍这些比较拗口的规则定义之前，先用一些不那么正式，但通俗易懂的语言来介绍一下这个关键字的作用。

​	当一个变量被定义成volatile之后，它将具备两项特性：第一项是**保证此变量对所有线程的可见性**，这里的“可见性”是指<u>当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量并不能做到这一点，普通变量的值在线程间传递时均需要通过主内存来完成</u>。比如，线程A修改一个普通变量的值，然后向主内存进行回写，另外一条线程B在线程A回写完成了之后再对主内存进行读取操作，新变量值才会对线程B可见。

​	关于volatile变量的可见性，经常会被开发人员误解，他们会误以为下面的描述是正确的：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反映到其他线程之中。换句话说，volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是线程安全的”。这句话的论据部分并没有错，但是由其论据并不能得出“基于volatile变量的运算在并发下是线程安全的”这样的结论。volatile变量在各个线程的工作内存中是不存在一致性问题的（**从物理存储的角度看，各个线程的工作内存中volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性问题**），但是<u>Java里面的运算操作符并非原子操作，这导致volatile变量的运算在并发下一样是不安全的</u>，我们可以通过一段简单的演示来说明原因，请看代码清单12-1中演示的例子。

​	代码清单12-1　volatile的运算

```java
/**
* volatile变量自增运算测试
*
* @author zzm
*/
public class VolatileTest {
  public static volatile int race = 0;
  public static void increase() {
    race++;
  }
  private static final int THREADS_COUNT = 20;
  public static void main(String[] args) {
    Thread[] threads = new Thread[THREADS_COUNT];
    for (int i = 0; i < THREADS_COUNT; i++) {
      threads[i] = new Thread(new Runnable() {
        @Override
        public void run() {
          for (int i = 0; i < 10000; i++) {
            increase();
          }
        }
      });
      threads[i].start();
    }
    // 等待所有累加线程都结束
    while (Thread.activeCount() > 1)
      Thread.yield();
    System.out.println(race);
  }
}
```

​	这段代码发起了20个线程，每个线程对race变量进行10000次自增操作，如果这段代码能够正确并发的话，最后输出的结果应该是200000。读者运行完这段代码之后，并不会获得期望的结果，而且会发现每次运行程序，输出的结果都不一样，都是一个小于200000的数字。这是为什么呢？

​	问题就出在自增运算“race++”之中，我们用Javap反编译这段代码后会得到代码清单12-2所示，发现只有一行代码的increase()方法在Class文件中是由4条字节码指令构成（return指令不是由race++产生的，这条指令可以不计算），从字节码层面上已经很容易分析出并发失败的原因了：<u>当getstatic指令把race的值取到操作栈顶时，volatile关键字保证了race的值在此时是正确的，但是在执行iconst_1、iadd这些指令的时候，其他线程可能已经把race的值改变了，而操作栈顶的值就变成了过期的数据，所以putstatic指令执行后就可能把较小的race值同步回主内存之中</u>。

​	代码清单12-2　VolatileTest的字节码

```shell
public static void increase();
	Code:
      Stack=2, Locals=0, Args_size=0
      0: getstatic #13; //Field race:I
      3: iconst_1
      4: iadd
      5: putstatic #13; //Field race:I
      8: return
	LineNumberTable:
      line 14: 0
      line 15: 8
```

​	实事求是地说，笔者使用字节码来分析并发问题仍然是不严谨的，因为**<u>即使编译出来只有一条字节码指令，也并不意味执行这条指令就是一个原子操作。一条字节码指令在解释执行时，解释器要运行许多行代码才能实现它的语义</u>**。如果是编译执行，一条字节码指令也可能转化成若干条本地机器码指令。此处使用`-XX:+PrintAssembly`参数输出反汇编来分析才会更加严谨一些，但是考虑到读者阅读的方便性，并且字节码已经能很好地说明问题，所以此处使用字节码来解释。

​	**由于<u>volatile变量只能保证可见性</u>，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用synchronized、java.util.concurrent中的锁或原子类）来保证原子性**：

+ 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
+ <u>变量不需要与其他的状态变量共同参与不变约束</u>。

​	而在像代码清单12-3所示的这类场景中就很适合使用volatile变量来控制并发，当`shutdown()`方法被调用时，能保证所有线程中执行的`doWork()`方法都立即停下来。

​	代码清单12-3　volatile的使用场景

```java
volatile boolean shutdownRequested;
public void shutdown() {
  shutdownRequested = true;
}
public void doWork() {
  while (!shutdownRequested) {
    // 代码的业务逻辑
  }
}
```

​	**使用volatile变量的第二个语义是禁止指令重排序优化**，<u>普通的变量仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致</u>。因为在同一个线程的方法执行过程中无法感知到这点，这就是Java内存模型中描述的所谓**“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics）**。

​	上面描述仍然比较拗口难明，我们还是继续通过一个例子来看看为何指令重排序会干扰程序的并发执行。演示程序如代码清单12-4所示。

​	代码清单12-4　指令重排序

```java
Map configOptions;
char[] configText;
// 此变量必须定义为volatile
volatile boolean initialized = false;

// 假设以下代码在线程A中执行
// 模拟读取配置信息，当读取完成后
// 将initialized设置为true,通知其他线程配置可用
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;

// 假设以下代码在线程B中执行
// 等待initialized为true，代表线程A已经把配置信息初始化完成
while (!initialized) {
sleep();
}
// 使用线程A中初始化好的配置信息
doSomethingWithConfig();
```

​	代码清单12-4中所示的程序是一段伪代码，其中描述的场景是开发中常见配置读取过程，只是我们在处理配置文件时一般不会出现并发，所以没有察觉这会有问题。读者试想一下，如果定义initialized变量时没有使用volatile修饰，就可能会由于指令重排序的优化，导致位于线程A中最后一条代码“initialized=true”被提前执行（这里虽然使用Java作为伪代码，但所指的重排序优化是机器级的优化操作，提前执行是指这条语句对应的汇编代码被提前执行），这样在线程B中使用配置信息的代码就可能出现错误，而volatile关键字则可以避免此类情况的发生[^6]。

​	指令重排序是并发编程中最容易导致开发人员产生疑惑的地方之一，除了上面伪代码的例子之外，笔者再举一个可以实际操作运行的例子来分析volatile关键字是如何禁止指令重排序优化的。代码清单12-5所示是一段标准的**双锁检测（Double Check Lock，DCL）单例**[^7]代码，可以观察加入volatile和未加入volatile关键字时所生成的汇编代码的差别（如何获得即时编译的汇编代码？请参考第4章关于HSDIS插件的介绍）。

​	代码清单12-5　DCL单例模式

```java
public class Singleton {
  private volatile static Singleton instance;
  public static Singleton getInstance() {
    if (instance == null) {
      synchronized (Singleton.class) {
        if (instance == null) {
          instance = new Singleton();
        }
      }
    }
    return instance;
  }
  public static void main(String[] args) {
    Singleton.getInstance();
  }
}
```

​	编译后，这段代码对instance变量赋值的部分如代码清单12-6所示。

​	代码清单12-6　对instance变量赋值

```shell
0x01a3de0f: mov 	$0x3375cdb0,%esi 			;...beb0cd75 33
																				; {oop('Singleton')}
0x01a3de14: mov 	%eax,0x150(%esi) 			;...89865001 0000
0x01a3de1a: shr 	$0x9,%esi 						;...c1ee09
0x01a3de1d: movb 	$0x0,0x1104800(%esi) 	;...c6860048 100100
0x01a3de24: lock 	addl $0x0,(%esp) 			;...f0830424 00
																				;*putstatic instance
																				; - Singleton::getInstance@24
```

​	通过对比发现，关键变化在于有volatile修饰的变量，赋值后（前面`mov %eax，0x150(%esi)`这句便是赋值操作）多执行了一个“`lock  addl $0x0，(%esp)`”操作，这个操作的作用相当于一个**内存屏障（Memory Barrier或Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置，注意不要与第3章中介绍的垃圾收集器用于捕获变量访问的内存屏障互相混淆）**，<u>只有一个处理器访问内存时，并不需要内存屏障；但如果有两个或更多处理器访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了</u>。

​	<u>这句指令中的“`addl $0x0，(%esp)`”（把ESP寄存器的值加0）显然是一个空操作，之所以用这个空操作而不是空操作专用指令nop，是因为IA32手册规定lock前缀不允许配合nop指令使用</u>。**这里的关键在于lock前缀，查询IA32手册可知，它的作用是<u>将本处理器的缓存写入了内存，该写入动作也会引起别的处理器或者别的内核无效化（Invalidate）其缓存，这种操作相当于对缓存中的变量做了一次前面介绍Java内存模式中所说的“store和write”操作</u>**。所以**通过这样一个空操作，可让前面volatile变量的修改对其他处理器立即可见**。

​	那为何说它禁止指令重排序呢？<u>从硬件架构上讲，指令重排序是指处理器采用了允许将多条指令不按程序规定的顺序分开发送给各个相应的电路单元进行处理。但并不是说指令任意重排，处理器必须能正确处理指令依赖情况保障程序能得出正确的执行结果</u>。譬如指令1把地址A中的值加10，指令2把地址A中的值乘以2，指令3把地址B中的值减去3，这时指令1和指令2是有依赖的，它们之间的顺序不能重排——`(A+10)*2`与`A*2+10`显然不相等，但指令3可以重排到指令1、2之前或者中间，只要保证处理器执行后面依赖到A、B值的操作时能获取正确的A和B值即可。所以<u>在同一个处理器中，重排序过的代码看起来依然是有序的。因此，`lock addl$0x0，(%esp)`指令把修改同步到内存时，意味着所有之前的操作都已经执行完成，这样便形成了“指令重排序无法越过内存屏障”的效果</u>。

​	解决了volatile的语义问题，再来看看在众多保障并发安全的工具中选用volatile的意义——它能让我们的代码比使用其他的同步工具更快吗？在某些情况下，volatile的同步机制的性能确实要优于锁（使用synchronized关键字或java.util.concurrent包里面的锁），但是由于虚拟机对锁实行的许多消除和优化，使得我们很难确切地说volatile就会比synchronized快上多少。如果让volatile自己与自己比较，那可以确定一个原则：volatile变量读操作的性能消耗与普通变量几乎没有什么差别，但是写操作则可能会慢上一些，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。不过即便如此，大多数场景下volatile的总开销仍然要比锁来得更低。**我们在volatile与锁中选择的唯一判断依据仅仅是volatile的语义能否满足使用场景的需求**。

​	本节的最后，我们再回头来看看Java内存模型中对volatile变量定义的特殊规则的定义。假定T表示一个线程，V和W分别表示两个volatile型变量，那么在进行read、load、use、assign、store和write操作时需要满足如下规则：

+ 只有当线程T对变量V执行的前一个动作是load的时候，线程T才能对变量V执行use动作；并且，只有当线程T对变量V执行的后一个动作是use的时候，线程T才能对变量V执行load动作。线程T对变量V的use动作可以认为是和线程T对变量V的load、read动作相关联的，必须连续且一起出现。

​	<u>这条规则要求在工作内存中，每次使用V前都必须先从主内存刷新最新的值，用于保证能看见其他线程对变量V所做的修改</u>。

+ 只有当线程T对变量V执行的前一个动作是assign的时候，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行的后一个动作是store的时候，线程T才能对变量V执行assign动作。线程T对变量V的assign动作可以认为是和线程T对变量V的store、write动作相关联的，必须连续且一起出现。

​	<u>这条规则要求在工作内存中，每次修改V后都必须立刻同步回主内存中，用于保证其他线程可以看到自己对变量V所做的修改</u>。

+ 假定动作A是线程T对变量V实施的use或assign动作，假定动作F是和动作A相关联的load或store动作，假定动作P是和动作F相应的对变量V的read或write动作；与此类似，假定动作B是线程T对变量W实施的use或assign动作，假定动作G是和动作B相关联的load或store动作，假定动作Q是和动作G相应的对变量W的read或write动作。如果A先于B，那么P先于Q。

​	**这条规则要求volatile修饰的变量不会被指令重排序优化，从而保证代码的执行顺序与程序的顺序相同**。

[^6]:volatile屏蔽指令重排序的语义在JDK 5中才被完全修复，此前的JDK中即使将变量声明为volatile也仍然不能完全避免重排序所导致的问题（主要是volatile变量前后的代码仍然存在重排序问题），这一点也是在JDK 5之前的Java中无法安全地使用DCL（双锁检测）来实现单例模式的原因。
[^7]:双重锁定检查是一种在许多语言中都广泛流传的单例构造模式。

### 12.3.4 针对long和double型变量的特殊规则

> [浮点处理单元_百度百科 (baidu.com)](https://baike.baidu.com/item/浮点处理单元/54846560?fromtitle=FPU&fromid=3412317&fr=aladdin)
>
> **浮点处理单元**（floating point unit，缩写FPU）是运行[浮点运算](https://baike.baidu.com/item/浮点运算?fromModule=lemma_inlink)的结构。一般是用[电路](https://baike.baidu.com/item/电路/33197?fromModule=lemma_inlink)来实现，应用在[计算机](https://baike.baidu.com/item/计算机?fromModule=lemma_inlink)芯片中。是整数运算器之后的一大发展，因为在浮点运算器发明之前，计算机中的浮点运算都是用[整数](https://baike.baidu.com/item/整数?fromModule=lemma_inlink)运算来模拟的，效率十分不良。浮点运算器一定会有[误差](https://baike.baidu.com/item/误差/738024?fromModule=lemma_inlink)，但科学及工程计算仍大量的依靠浮点运算器——只是在程序设计时就必需考虑[精确度](https://baike.baidu.com/item/精确度/1612899?fromModule=lemma_inlink)问题。

​	Java内存模型要求lock、unlock、read、load、assign、use、store、write这八种操作都具有原子性，但是对于64位的数据类型（long和double），在模型中特别定义了一条宽松的规定：允许虚拟机将没有被volatile修饰的64位数据的读写操作划分为两次32位的操作来进行，即**允许虚拟机实现自行选择是否要保证64位数据类型的load、store、read和write这四个操作的原子性，这就是所谓的“long和double的非原子性协定”（Non-Atomic Treatment of double and long Variables）**。

​	如果有多个线程共享一个并未声明为volatile的long或double类型的变量，并且同时对它们进行读取和修改操作，那么某些线程可能会读取到一个既不是原值，也不是其他线程修改值的代表了“半个变量”的数值。不过这种读取到“半个变量”的情况是非常罕见的，<u>经过实际测试，在目前主流平台下商用的64位Java虚拟机中并不会出现非原子性访问行为，但是对于32位的Java虚拟机，譬如比较常用的32位x86平台下的HotSpot虚拟机，对long类型的数据确实存在非原子性访问的风险</u>。从JDK 9起，HotSpot增加了一个实验性的参数`-XX:+AlwaysAtomicAccesses`（这是JEP 188对Java内存模型更新的一部分内容）来约束虚拟机对所有数据类型进行原子性的访问。而**针对double类型，由于现代中央处理器中一般都包含专门用于处理浮点数据的<u>浮点运算器（Floating Point Unit，FPU）</u>，用来专门处理单、双精度的浮点数据，所以哪怕是32位虚拟机中通常也不会出现非原子性访问的问题，实际测试也证实了这一点**。

​	**笔者的看法是，在实际开发中，除非该数据有明确可知的线程竞争，否则我们在编写代码时一般不需要因为这个原因刻意把用到的long和double变量专门声明为volatile**。

### 12.3.5 原子性、可见性与有序性

​	介绍完Java内存模型的相关操作和规则后，我们再整体回顾一下这个模型的特征。Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这三个特征来建立的，我们逐个来看一下哪些操作实现了这三个特性。

#### 1. 原子性（Atomicity）

​	由Java内存模型来直接保证的原子性变量操作包括read、load、assign、use、store和write这六个，我们大致可以认为，基本数据类型的访问、读写都是具备原子性的（例外就是long和double的非原子性协定，读者只要知道这件事情就可以了，无须太过在意这些几乎不会发生的例外情况）。

​	**如果应用场景需要一个更大范围的原子性保证（经常会遇到），Java内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐式地使用这两个操作。这两个字节码指令反映到Java代码中就是同步块——synchronized关键字，因此在synchronized块之间的操作也具备原子性**。

#### 2. 可见性（Visibility）

​	可见性就是指当一个线程修改了共享变量的值时，其他线程能够立即得知这个修改。上文在讲解volatile变量的时候我们已详细讨论过这一点。<u>**Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此**</u>。普通变量与volatile变量的区别是，volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。因此我们可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。

​	除了volatile之外，Java还有两个关键字能实现可见性，它们是synchronized和final。**同步块的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）”这条规则获得的**。**<u>而final关键字的可见性是指：被final修饰的字段在构造器中一旦被初始化完成，并且构造器没有把“this”的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到“初始化了一半”的对象），那么在其他线程中就能看见final字段的值</u>**。如代码清单12-7所示，变量i与j都具备可见性，它们无须同步就能被其他线程正确访问。

​	代码清单12-7　final与可见性

```java
public static final int i;
public final int j;
static {
  i = 0;
  // 省略后续动作
}
{
  // 也可以选择在构造函数中初始化
  j = 0;
  // 省略后续动作
}
```

#### 3. 有序性（Ordering）

​	Java内存模型的有序性在前面讲解volatile时也比较详细地讨论过了，<u>Java程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程，所有的操作都是无序的。前半句是指“线程内似表现为串行的语义”（Within-Thread As-If-Serial Semantics），后半句是指“指令重排序”现象和“工作内存与主内存同步延迟”现象</u>。

​	Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入。

​	介绍完并发中三种重要的特性，读者是否发现synchronized关键字在需要这三种特性的时候都可以作为其中一种的解决方案？看起来很“万能”吧？的确，绝大部分并发控制操作都能使用synchronized来完成。synchronized的“万能”也间接造就了它被程序员滥用的局面，越“万能”的并发控制，通常会伴随着越大的性能影响，关于这一点我们将在下一章讲解虚拟机锁优化时再细谈。

### 12.3.6 先行发生原则

​	如果Java内存模型中所有的有序性都仅靠volatile和synchronized来完成，那么有很多操作都将会变得非常啰嗦，但是我们在编写Java并发代码的时候并没有察觉到这一点，这是因为Java语言中有一个**“先行发生”（Happens-Before）的原则**。这个原则非常重要，它是判断数据是否存在竞争，线程是否安全的非常有用的手段。依赖这个原则，我们可以通过几条简单规则一揽子解决并发环境下两个操作之间是否可能存在冲突的所有问题，而不需要陷入Java内存模型苦涩难懂的定义之中。

​	现在就来看看“先行发生”原则指的是什么。**先行发生是Java内存模型中定义的两项操作之间的<u>偏序关系</u>，比如说操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被操作B观察到，“影响”包括修改了内存中共享变量的值、发送了消息、调用了方法等**。这句话不难理解，但它意味着什么呢？我们可以举个例子来说明一下。如代码清单12-8所示的这三条伪代码。

​	代码清单12-8　先行发生原则示例1

```java
// 以下操作在线程A中执行
i = 1;
// 以下操作在线程B中执行
j = i;
// 以下操作在线程C中执行
i = 2;
```

​	假设线程A中的操作“i=1”先行发生于线程B的操作“j=i”，那我们就可以确定在线程B的操作执行后，变量j的值一定是等于1，得出这个结论的依据有两个：一是根据先行发生原则，“i=1”的结果可以被观察到；二是线程C还没登场，线程A操作结束之后没有其他线程会修改变量i的值。现在再来考虑线程C，我们依然保持线程A和B之间的先行发生关系，而C出现在线程A和B的操作之间，但是C与B没有先行发生关系，那j的值会是多少呢？答案是不确定！1和2都有可能，因为线程C对变量i的影响可能会被线程B观察到，也可能不会，这时候线程B就存在读取到过期数据的风险，不具备多线程安全性。

​	**<u>下面是Java内存模型下一些“天然的”先行发生关系，这些先行发生关系无须任何同步器协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从下列规则推导出来，则它们就没有顺序性保障，虚拟机可以对它们随意地进行重排序</u>**。

+ 程序次序规则（Program Order Rule）：在<u>一个线程内</u>，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。注意，这里说的是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。

+ 管程锁定规则（Monitor Lock Rule）：<u>一个unlock操作先行发生于后面对同一个锁的lock操作</u>。这里必须强调的是“同一个锁”，而“后面”是指时间上的先后。

+ volatile变量规则（Volatile Variable Rule）：<u>对一个volatile变量的写操作先行发生于后面对这个变量的读操作</u>，这里的“后面”同样是指时间上的先后。

+ 线程启动规则（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的每一个动作。

+ 线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread::join()方法是否结束、Thread::isAlive()的返回值等手段检测线程是否已经终止执行。

+ 线程中断规则（Thread Interruption Rule）：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread::interrupted()方法检测到是否有中断发生。

+ 对象终结规则（Finalizer Rule）：**一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始**。

+ **<u>传递性（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论</u>**。

​	Java语言无须任何同步手段保障就能成立的先行发生规则有且只有上面这些，下面演示一下如何使用这些规则去判定操作间是否具备顺序性，对于读写共享变量的操作来说，就是线程是否安全。读者还可以从下面这个例子中感受一下“时间上的先后顺序”与“先行发生”之间有什么不同。演示例子如代码清单12-9所示。

​	代码清单12-9　先行发生原则示例2

```java
private int value = 0;
pubilc void setValue(int value){
  this.value = value;
}
public int getValue(){
  return value;
}
```

​	代码清单12-9中显示的是一组再普通不过的getter/setter方法，假设存在线程A和B，线程A先（时间上的先后）调用了setValue(1)，然后线程B调用了同一个对象的getValue()，那么线程B收到的返回值是什么？

​	我们依次分析一下先行发生原则中的各项规则。由于两个方法分别由线程A和B调用，不在一个线程中，所以程序次序规则在这里不适用；由于没有同步块，自然就不会发生lock和unlock操作，所以管程锁定规则不适用；由于value变量没有被volatile关键字修饰，所以volatile变量规则不适用；后面的线程启动、终止、中断规则和对象终结规则也和这里完全没有关系。**因为没有一个适用的先行发生规则，所以最后一条传递性也无从谈起，因此我们可以判定，尽管线程A在操作时间上先于线程B，但是无法确定线程B中getValue()方法的返回结果，换句话说，这里面的操作不是线程安全的**。

​	那怎么修复这个问题呢？我们至少有两种比较简单的方案可以选择：要么把getter/setter方法都定义为synchronized方法，这样就可以套用管程锁定规则；<u>要么把value定义为volatile变量，由于setter方法对value的修改不依赖value的原值，满足volatile关键字使用场景，这样就可以套用volatile变量规则来实现先行发生关系</u>。

​	通过上面的例子，我们可以得出结论：一个操作“时间上的先发生”不代表这个操作会是“先行发生”。那<u>如果一个操作“先行发生”，是否就能推导出这个操作必定是“时间上的先发生”呢？很遗憾，这个推论也是不成立的。一个典型的例子就是多次提到的“指令重排序”</u>，演示例子如代码清单12-10所示。

​	代码清单12-10　先行发生原则示例3

```java
// 以下操作在同一个线程中执行
int i = 1;
int j = 2;
```

​	代码清单12-10所示的两条赋值语句在同一个线程之中，根据程序次序规则，“int i=1”的操作先行发生于“int j=2”，但是“int j=2”的代码完全可能先被处理器执行，这并不影响先行发生原则的正确性，因为我们在这条线程之中没有办法感知到这一点。

​	上面两个例子综合起来证明了一个结论：**时间先后顺序与先行发生原则之间基本没有因果关系，所以我们衡量并发安全问题的时候不要受时间顺序的干扰，一切必须以先行发生原则为准**。

## 12.4 Java与线程

### 12.4.1 线程的实现



