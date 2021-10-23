# 《Effective-Java第三版》-读书笔记-02

> + `1、`、`2、`表示：第1章、第2章
>
> + `E1`、`E2`表示：第1条、第2条
> + `* X、`表示：个人认为第X章是重点（注意点）
> + `* EX`表示：个人认为第X条是重点（注意点）
>
> 全书共12章，90条目
>
> 下面提到的“设计模式”，指《Design Patterns - Elements of Reusable Object-Oriented Software》一书中提到的23种设计模式

# 6、枚举和注解

​	Java支持两种特殊用途的引用类型：一种是类，称作枚举类型(enum type)；一种是接口，称作注解类型(annotation type)。本章将讨论这两个新类型的最佳使用实践。

## E34 用enum代替int常量

> [Java 静态内部类的加载时机 - は問わない - 博客园 (cnblogs.com)](https://www.cnblogs.com/zouxiangzhongyan/p/10762540.html)

+ 引入enum之前的方案

  ​	在Java引入枚举类型之前，通常使用一组int常量来表示枚举类型，其中每一个int常量表示枚举类型的一个成员。但是int枚举模式，不具类型安全性，也几乎没有描述性可言。

  ​	采用int枚举模式的程序是十分脆弱的。因为**int枚举是编译时常量(constant variable)** [JLS，4.12.4]，它们的int值会被编译到使用它们的客户端中。如果与int枚举常量关联的值发生了变化，客户端必须重新编译。如果没有重新编译，客户端程序还是可以运行，不过其行为已经不再准确。

  ​	*很难将int枚举常量转换成可打印的字符串。就算将这种常量打印岀来，或者从调试器中将它显示出来，你所见到的也只是一个数字，这几乎没有什么用处。当需要遍历一个int枚举模式中的所有常量，以及获得int枚举数组的大小时，在int枚举模式中，几乎不存在可靠的方式。*

  ​	另一种类似的即String枚举模式（String enum pattern），同样效果不是很理想。

---

+ enum概述

  ​	**Java的枚举类型的本质是int值**。

  ​	Java枚举类型的基本想法非常简单：这些类通过**公有的静态final域为每个枚举常量导出一个实例**。**<u>枚举类型没有可以访问的构造器，所以它是真正的final类</u>**。客户端不能创建枚举类型的实例，也不能对它进行扩展，因此不存在实例，而只存在声明过的枚举常量。

  ​	换句话说，枚举类型是实例受控的（E6）。它们是单例(Singleton)（E3）的泛型化，本质上是单元素的枚举。

  ​	**枚举类型保证了编译时的类型安全**。例如声明参数的类型为Apple，它就能保证传到该参数上的任何非空的对象引用一定属于三个有效的Apple值之一，而其他任何试图传递类型错误的值都会导致编译时错误，就像<u>试图将某种枚举类型的表达式赋给另一种枚举类型的变量，或者试图利用\=\=操作符比较不同枚举类型的值都会导致编译时错误</u>。

  ​	**包含同名常量的多个枚举类型可以在一个系统中和平共处，因为每个类型都有自己的<u>命名空间</u>**。你可以增加或者重新排列枚举类型中的常量，而无须重新编译它的客户端代码，因为导出常量的域在枚举类型和它的客户端之间提供了一个隔离层：<u>常量值并没有被编译到客户端代码中，而是在int枚举模式之中。最终，可以通过调用toString方法，将枚举转换成可打印的字符串</u>。	

  ​	除了完善int枚举模式的不足之外，枚举类型还允许添加任意的方法和域，并实现任意的接口。它们提供了所有 Object方法（E3）的高级实现，实现了 Comparable（E14）和 Serializable接口(详见第12章)，并针对枚举类型的可任意改变性设计了序列化方式。

  ​	**为了将数据与枚举常量关联起来，得声明实例域，并编写一个带有数据并将数据保存在域中的构造器**。枚举天生就是不可变的，因此所有的域都应该为final的（E17）。它们可以是公有的，但最好将它们声明为私有的，并提供公有的访问方法（E16）。

  ​	**所有的枚举，都有一个静态的values方法，按照声明顺序返回它的值数组。toString方法返回每个枚举值的声明名称，使得println和printf的打印变得更加容易**。

  ​	如果将一个元素从一个枚举类型移除，那么没有引用该元素的任何客户端程序都会继续正常工作。<u>如果客户端引用了被删除的元素，重新编译客户端就会失败，并在引用被删除的枚举元素的那一条出现一条错误信息；如果没有重新编译客户端代码，在运行时就会在这一行抛出一个异常。这是你能期待的最佳行为了，远比使用int枚举模式时要好得多</u>。

  ​	有些与枚举常量相关的行为，可能只会用在枚举类型的定义类或者所在的包中，那么这些方法最好被实现成私有的或者包级私有的。于是每个枚举常量都带有一组隐藏的行为，这使得枚举类型的类或者所在的包能够运作得很好，**像其他的类一样，除非要将枚举方法导出至它的客户端，否则都应该声明为私有的，或者声明为包级私有的（E15）**。

  ​	<u>如果一个枚举具有普遍适用性，它就应该成为一个顶层类(top-level class)；如果它只是被用在一个特定的顶层类中，它就应该成为该顶层类的一个成员类（E24）</u>。例如，`java.math.RoundingMode`枚举表示十进制小数的舍入模式(rounding mode)。这些舍入模式被用于BigDecimal类，但是它们却不属于BigDecimal类的一个抽象。通过使RoundingMode变成一个顶层类，库的设计者鼓励任何需要舍入模式的程序员重用这个枚举，从而增强API之间的一致性。

  ​	**枚举类型中的抽象方法必须被它的所有常量中的具体方法所覆盖**。

  ​	<u>枚举类型有一个自动产生的`valueOf(String)`方法，它将常量的名字转成常量本身。如果在枚举类型中覆盖toString方法，要考虑编写一个fromString方法，将定制的字符串表示法变回相应的枚举</u>。

  ​	**除了编译时常量域（E34）之外，枚举构造器不可以访问枚举的静态域**。这一限制是有必要的，<u>因为构造器运行的时候，这些静态域还没有被初始化。这条限制只有一个特例：枚举常量无法通过构造器访问另一个构造器</u>。

  ​	那么什么时候应该使用枚举呢？**每当需要一组固定常量，并且在编译时就知道其成员的时候，就应该使用枚举**。当然，这包括"天然的枚举类型"，例如行星、一周的天数以及棋子的数目等。但它也包括你在编译时就知道其所有可能值的其他集合，例如菜单的选项、操作代码以及命令行标记等。**枚举类型中的常量集并不一定要始终保持不变**。专门设计枚举特性是考虑到枚举类型的二进制兼容演变。

---

+ 代码举例

  ​	下面代码可以为任何枚举完成一个String到枚举的转化，只要每个常量都有一个独特的字符串表示法：

  ```java
  // Implementing a fromString method on an enum type
  private static final Map<String, Operation> stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));
  
  // Return Operation for string, if any
  public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
  }
  ```

  ​	注意返回`Optional<Operation>`的fromString方法。它用该方法表明：传入的字符串并不代表一项有效的操作，并强制客户端面对这种可能性（E55）。

  ---

  ​	**策略枚举（strategy enum）**代码举例：

  ```java
  // The strategy enum pattern
  enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THUSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEDEND);
    
    private final PayType payType;
    
    PayrollDay(PayType payType) { this.payType = payType; }
    PayrollDay() { this(PayType.WEEKDAY); } // Default
    
    int pay(int minutesWorked, int payRate) {
      return payType.pay(minutesWorked, payRate);
    }
    
    // The strategy enum type
    private enum PayType {
      WEEKDAY {
        int overtimePay(int minsWorked, int payRate) {
          return minsWorked <= MINS_PER_SHIFT ? 0 :
          (minsWorked - MINS_PER_SHIFT) * payRate / 2;
        }
      },
      WEEKEND {
  			int overtimePay(int minsWorked, int payRate) {
          return minsWorked * payRate / 2;
        }
      };
      
      abstract int overtimePay(int mins, int payRate);
      
      private static final int MINS_PER_SHIFT = 8 * 60;
      
      int pay(int minsWorked, int payRate) {
        int basePay = minsWorked * payRate;
        return baseaPay + overtimePay(minsWorked, payRate);
      }
    }
  }
  ```

  ​	如果枚举中的switch语句不是在枚举中实现特定于常量的行为的一种很好的选择，那么它们还有什么用处呢？**枚举中的switch语句适合于给外部的枚举类型增加特定于常量的行为**。例如，假设 Operation枚举不受你的控制，你希望它有一个实例方法来返回每个运算的反运算。你可以用下列静态方法模拟这种效果：

  ```java
  // Switch on an enum to simulate a missing method
  public static Operation inverse(Operation op) {
    switch(op) {
      case PLUS: return Operation.MINUS;
      case MINUS: return Operation.PLUS;
      case TIMES: return Operation.DIVIDE;
      case DIVIDE: return Operation.TIMES;
      default: throw new AssertionError("Unknown op: " + op);
    }
  }
  ```

  > ​	一般来说，枚举通常在性能上与int常量相当。与int常量相比，枚举有个小小的性能缺点，即装载和初始化枚举时会需要空间和时间的成本，但在实践中几乎注意不到这个问题。

---

+ 小结

  ​	总而言之，与int常量相比，枚举类型的优势是不言而喻的。枚举的可读性更好，也更加安全，功能更加强大。

  ​	许多枚举都不需要显式的构造器或者成员，但许多其他枚举则受益于属性与每个常量的关联以及其行为受该属性影响的方法。

  ​	只有极少数的枚举受益于将多种行为与单个方法关联。在这种相对少见的情况下，特定于常量的方法要优先于启用自有值的枚举。

  ​	<u>如果多个(但非所有)枚举常量同时共享相同的行为，则要考虑策略枚举</u>。

## E35 用实例域代替序数

+ 概述

  ​	许多枚举天生就与一个单独的int值相关联。所有的枚举都有一个ordinal方法，它返回每个枚举常量在类型中的数字位置。你可以试着从序数中得到关联的int值：

  ```java
  // Enum.java 源码
  public abstract class Enum<E extends Enum<E>>
    implements Comparable<E>, Serializable { 
  	
    // ... 省略其他代码
    private final int ordinal;
  
    /**
       * Returns the ordinal of this enumeration constant (its position
       * in its enum declaration, where the initial constant is assigned
       * an ordinal of zero).
       *
       * Most programmers will have no use for this method.  It is
       * designed for use by sophisticated enum-based data structures, such
       * as {@link java.util.EnumSet} and {@link java.util.EnumMap}.
       *
       * @return the ordinal of this enumeration constant
       */
    public final int ordinal() {
      return ordinal;
    }
    
    // ... 省略其他代码
  }
  ```

  ​	**永远不要根据枚举的序数导出与它关联的值，而是要将它保存在一个实例域中**：

  ```java
  public enum Ensemble {
  	SOLO(1), DUET(2); // 比较懒，这里省略了余下枚举实例
    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
  }
  ```

  ​	Enum规范中谈及ordinal方法时写道："大多数程序员都不需要这个方法。它是设计用于像 EnumSet和 EnumMap这种基于枚举的通用数据结构的。" <u>除非你在编写的是这种数据结构，否则**最好完全避免使用ordinal方法**</u>。

## E36 用EnumSet代替位域

+ 概述

  ​	如果一个枚举类型的元素主要用在集合中，一般就使用int枚举模式（E34），比如将2的不同倍数赋予每个常量：

  ```java
  // Bit field enumeration constants - OBSOLETE!
  public class Text {
    public static final int STYLE_BOLD  = 1 << 0; // 1
    public static final int STYLE_ITALIC = 1 << 1; // 2
    public static final int STYLE_UNDERLINE = 1 << 2; // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
    
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
  }
  ```

---

+ 位域

  ​	通过OR位运算将几个常量合并到一个集合中，称作位域（bit field）：

  ```java
  text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
  ```

  优势：

  + 有效地执行像union（求并集）和intersection（求交集）的集合操作。

  缺点：

  + 具有int枚举常量的所有缺点。
  + 翻译位域比翻译简单的int枚举常量困难。
  + 遍历元素困难。
  + 需要预测最多需要多少位，选择相应类型（一般是int或者long）。

---

+ 替代位域的优选方案——EnumSet

  ​	需要传递多组常量集时，有些人回倾向于用位域代替枚举。

  ​	**`java.util.EnumSet`类能有效地表示从单个枚举类型中提取的多个值的多个集合，是位域的优选替代方案**。这个类实现了Set接口，提供了丰富功能和类型安全性，以及可以从其他Set实现中得到的互用性。

  + 在内部实现中，每个EnumSet内容都表示为位矢量。**如果底层的枚举类型有64个或者更少的元素（大多如此）整个EnumSet就使用单个long表示，因此它的性能比得上位域的性能**。
  + **批处理操作，如removeAll和retainAll，都是利用位算法来实现的，就像手工替代位域实现的那样**。但是可以避免手工操作时容易出现的错误以及丑陋的代码，因为EnumSet替你完成了这项艰巨的工作。

  ​	以下ø使用枚举Enum替代前面的位域代码，更加简洁、安全：

  ```java
  // EnumSet - a modern replacement for bit fields
  public class Text {
    public enum Style { BOLD, ITALIC, UNDELINE, STRIKETHROUGH }
    
    // Any Set could be passed in, but EnumSet is clearly best 
    public void applyStyles(Set<Style> styles) { ... }
  }
  ```

  ​	下面是将EnumSet实例传递给applyStyles方法的客户端代码。EnumSet提供了丰富的静态工厂来轻松创建集合，其中一个如下代码所示：

  ```java
  text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
  ```

  > 注意，applyStyles方法采用的是`Set<Sty1e>`而非`EnumSet<Sty1e>`。虽然看起来好像所有的客户端都可以将EnumSet传到这个方法，但是**最好还是接受接口类型而非接受实现类型**（E64）。这是考虑到可能会有特殊的客户端需要传递一些其他的Set实现。

---

+ 小结

  ​	**正是因为枚举类型要用在集合中，所以没有理由用位域来表示它**。

  + EnumSet比起位域更简洁，且带有（E34）中枚举类型的所有优点。
  + EnumSet缺点，**截止Java9发行版本，它都无法创建不可变的EnumSet**。可以用`Collections.unmodifiableSet将EnumSet`封装起来，但是简洁性和性能会受到影响。

> [EnumSet (Java SE 11 & JDK 11 ) (runoob.com)](https://www.runoob.com/manual/jdk11api/java.base/java/util/EnumSet.html)

## * E37 用EnumMap代替序数索引

+ 概述

  ​	有一种快速的Map实现专门用于枚举键，称作`java.util.EnumMap`。

  ​	**EnumMap构造器采用键类型的Class对象：这是一个有限制的类型令牌（bounded type token），它提供了运行时的泛型信息（E33）**。

---

+ 代码举例

  ​	假设有一个类表示用于烹饪的香草：

  ```java
  class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    
    final String name;
    final LifeCycle lifeCycle;
    
    Plant(String name, LifeCycle lifeCycle) {
      this.name = name;
      this.lifeCycle = lifeCycle;
    }
    
    @Override public String toString() {
      return name;
    }
  }
  ```

  ​	假设按照一年生、多年生或者多年生类别对植物进行分类，容易想到需要构建三个集合，每种类型一个，遍历整座花园，将每种香草放到相应的集合中。

  ​	有些程序员会讲这些集合放到一个按照<u>类型的序数</u>进行索引的数组中实现这一点：

  ```java
  // Using ordinal() to index into an array - DON't DO THIS!
  Set<Plant>[] plantsByLifeCycle = 
    (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
  for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifrCycle[i] = new HashSet<>();
  
  for(Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);
  
  // Print the results
  for(int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n",
                     Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
  }
  ```

  ​	上述代码虽可行，但是隐藏许多问题：

  + 数组不能与泛型（E28）兼容，程序需要进行未受检的转换，并且不能正确无误地进行编译。
  + 当你访问一个按照枚举的序数进行索引的数组时，使用正确的int值就是你的职责了。
  + int不能提供枚举的类型安全。

  ​	以下使用EnumMap对代码进行改进：

  ```java
  // Using an EnumMap to associate data with an enum
  Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle = 
    new EnumMap<>(Plant.LifeCycle.class);
  for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());
  for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
  System.out.println(plantsByLifeCycle);
  ```

  ​	这段程序更简短、更清楚，也更加安全，运行速度方面可以与使用序数的程序相媲美。它没有不安全的转换；不必手工标注这些索引的输出，因为映射键知道如何将自身翻译成可打印字符串的枚举；计算数组索引时也不可能出错。 

  ​	EnumMap在运行速度方面之所以能与通过序数索引的数组相媲美，正是**因为EnuMap在内部使用了这种数组。但是它对程序员隐藏了这种实现细节，集Map的丰富功能和类型安全与数组的快速于一身**。注意 Enummap构造器采用键类型的 Class对象：这是一个有限制的类型令牌(bounded type token)，它提供了运行时的泛型信息（E33）。

  ​	上一段程序可能比用 stream（E45）管理映射要简短得多。下面是基于 stream的最简单的代码，大量复制了上一个示例的行为：

  ```java
  // Naive stream-based approach - unlikely to produce an EnumMap!
  System.out.println(Arrays.stream(garden)
                     .collect(gropingBy(p -> p.lifeCycle)));
  ```

  ​	这段代码的问题在于它选择自己的映射实现，实际上不会是一个EnumMap，因此<u>与显式EnumMap版本的空间及时间性能并不吻合</u>。为了解决这个问题，要使用有三种参数形式的`Collectors.groupingBy`方法，它允许调用者利用mapFactory参数定义映射实现：

  ```java
  // Using a stream and an EnumMap to associate data with an enum
  System.out.println(Arrays.stream(garden))
    .collect(groupingBy(p -> p.lifeCycle,
                        () -> new EnumMap<>(LifeCycle.class),toSet()));
  ```

  ​	<u>在这样一个玩具程序中不值得进行这种优化，但是在大量使用映射的程序中就很重要了</u>。

  ​	基于 stream的代码版本的行为与EnumMap版本的稍有不同。 EnumMap版本总是给每一个植物生命周期都设计一个嵌套映射，基于 stream的版本则仅当花园中包含了一种或多种植物带有该生命周期时才会设计一个嵌套映射。因此，假如花园中包含了一年生和多年生植物，但没有两年生植物，plantByLifeCycle的数量在 EnumMap版本中应该是三种，在基于 stream的两个版本中则都是两种。

  ​	你还可能见到按照序数进行索引（两次）的数组的数组，该序数表示两个枚举值的映射。例如，下面这个程序就是使用这样一个数组将两个阶段映射到一个阶段过渡中（从液体到固体称作凝固，从液体到气体称作沸腾，诸如此类）：

  ```java
  // Using ordinal() to index array of arrays - DON'T DO THIS!
  public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
      MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;
      
      // Rows indexed by from-ordinal, cols by to-ordinal
      private static final Transition[][] TRANSITIONS = {
        {null, MELT, SUBLIME},
        {FREEZE, null, BOIL},
        {DEPOSIT, CONDENSE, null}
      };
      
      // Returns the phase transition from one phase to another
      public static Transition from(Phase from, Phase to) {
        return TRANSITIONS[from.ordinal()][to.ordinal()];
      }
    }
  }
  ```

  ​	这段程序可行，看起来也比较优雅，但是事实并非如此。就像上面那个比较简单的香草花园的示例一样，编译器无法知道序数和数组索引之间的关系。如果在过渡表中出了错，或者在修改`Phase`或者`Phase.Transition`枚举类型的时候忘记将它更新，程序就会在运行时失败。这种失败的形式可能为 ArrayIndexOutOfBoundsException、NullPointerException或者(更糟糕的是)没有任何提示的错误行为。这张表的大小是阶段数的平方，即使非空项的数量比较少。

  ​	同样，利用 EnumMap依然可以做得更好一些。因为每个阶段过渡都是通过一对阶段枚举进行索引的，最好将这种关系表示为一个映射，这个映射的键是一个枚举(起始阶段)，值为另一个映射，这第二个映射的键为第二个枚举(目标阶段，它的值为结果(阶段过渡，即形成了Map(起始阶段，Map(目标阶段，阶段过渡)这种形式。一个阶段过渡所关联的两个阶段，最好通过"数据与阶段过渡枚举之间的关系"获取，之后用该阶段过渡枚举来初始化嵌套的EnumMap：

  ```java
  // Using a nested EnumMap to associate data with enum pairs
  public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
      MELT(SOLID, LIQUID), FREEZE(LIQUID, SILID),
      BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
      SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
  
      private final Phase from;
      private final Phase to;
  
      Transition(Phase from, Phase to) {
        this.from = from;
        this.to = to;
      }
  
      // Initialize the phase transition map
      private static final Map<Phase, Map<Phase, Transition>>
        m = Stream.of(values()).collect(gropingBy(t -> t.from, () -> new EnumMap<>(Phase.class), toMap(t -> t.to, t -> t, (x, y) -> y, () -> new EnumMap<>(Phase.class))));
  		
      public static Transition from(Phase from, Phase to) {
        return m.get(from).get(to);
      }
    }
  }
  ```

  ​	初始化阶段过渡映射的代码看起来可能有点复杂。映射的类型为`Map<Phase，Map<Phase， Transition>>`，表示是由键为源Phase(即第一个阶段)、值为另一个映射组成的Map，其中组成值的Map是由键值对目标Phase(即第二个阶段)和Transition组成的。这个映射的映射是利用两个集合的级联顺序进行初始化的。第一个集合按源Phase对过渡进行分组，第二个集合利用从目标Phase到过渡之间的映射创建一个 EnumMap。第个集合中的merge函数`((x，y)->y)`没有用到；只有当我们因为想要获得一个 EnumMap而定义映射工厂时才需要用到它，同时Collectors提供了重叠工厂。本书第2版是利用显式迭代来初始化阶段过渡映射的。其代码更加烦琐，但是的确更易于理解。

  ​	现在假设想要给系统添加一个新的阶段：plasma(离子)或者电离气体。只有两个过渡与这个阶段关联：电离化(ionization)，它将气体变成离子；以及消电离化(deionization)，将离子变成气体。为了更新基于数组的程序，必须给Phase添加一种新常量，给`Phase.Transition`添加两种新常量，用一种新的16个元素的版本取代原来9个元素的数组的数组。如果给数组添加的元素过多或者过少，或者元素放置不妥当，可就麻烦了：程序可以编译，但是会在运行时失败。为了更新基于 EnumMap的版本，所要做的就是必须将PLASMA添加到 Phase列表，并将工IONIZE(GAS， PLASMA)和DEIONIZE(PLASMA，GAS)添加到`Phase.Transition`的列表中：

  ```java
  // Adding a new phase using the nested EnumMap implementation
  public enum Phase {
    SOLID, LIQUID, GAS, PLASMA;
    public enum Transition {
      MELT(SOLD, LIQUID), FREEZR(LIQUID, SOLID),
      BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
      SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
      IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
      ...// Remainder unchanged
    }
  }
  ```

  ​	程序会自行处理所有其他的事情，这样就几乎没有出错的可能。从内部来看，映射的映射被实现成了数组的数组，因此在提升了清晰性、安全性和易维护性的同时，在空间或者时间上也几乎没有多余的开销。

  ​	为了简洁起见，上述范例是用null表明状态没有变化(这里的to和from是相等的)。这并不是好的实践，可能在运行时导致NullPointerException异常。要给这个问题设计个整洁、优雅的解决方案，需要高超的技巧，得到的程序会很长，贬损了本条目的主要精神。

---

+ 小结

  ​	总而言之，**最好不要用序数来索引数组，而要使用 EnumMap**。

  ​	如果你所表示的这种关系是多维的，就使用`EnumMap<... , EnumMap<...>>`。应用程序的程序员在一般情况下都不使用`Enum.ordinal`方法，仅仅在极少数情况下才会使用，因此这是一种特殊情况（E35）。

## * E38 用接口模拟可拓展的枚举

+ 概述

  ​	从多方面来看，枚举类型优于本书第1版中描述的类型安全枚举模式。第1版所述的模式能实现让一个枚举类型去扩展另一个枚举类型；利用这种语言特性，则不可能做到。

  ​	**枚举的可伸缩性最后证明基本上都不是什么好点子**。

  ​	目前还没有很好的方法来枚举基本类型的所有元素及其扩展。最终，可伸缩性会导致设计和实现的许多方面变得复杂起来。

  ​	<u>对于可伸缩的枚举类型而言，至少有一种具有说服力的用例，这就是操作码(operation code)，也称作 opcode</u>。操作码是指这样的枚举类型：它的元素表示在某种机器上的那些操作，例如（E34）中的 Operation类型，它表示一个简单的计算器中的某些函数。有时要尽可能地让API的用户提供它们自己的操作，这样可以有效地扩展API所提供的操作集。

  ​	幸运的是，有一种很好的方法可以利用枚举类型来实现这种效果。由于枚举类型可以通过给操作码类型和（属于接口的标准实现的)枚举定义接口来实现任意接口，基本的想法就是利用这一事实。例如，以下是（E34）中的Operation类型的扩展版本：

  ```java
  public interface Operation {
    double apply(double x, double y);
  }
  
  public enum BasicOperation implements Operation {
    PLUS("+") {
      public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
      public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
      public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
      public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    
    BasicOepration(String symbol) {
      this.symbol = symbol;
    }
    
    @Override
    public String toString() {
      return symbol;
    }
  }
  ```

  ​	在可以使用基础操作的任何地方，现在都可以使用新的操作，只要API是写成采用接类型(Operation)而非实现(BasicOperation)。注意，在枚举中，不必像在不可扩展的枚举中所做的那样，利用特定于实例的方法实现（E34）来声明抽象的apply方法。因为抽象的方法(apply)是接口(Operation)的一部分。

  ​	不仅可以在任何需要"基本枚举"的地方单独传递一个"扩展枚举"的实例，而且除了那些基本类型的元素之外，还可以传递完整的扩展枚举类型，并使用它的元素。例如，通过(E34)的测试程序版本，体验一下上面定义过的所有扩展过的操作：

  ```java
  public static void main(String[] args) {
    double x = Double.parseDounle(args[0]);
    double y = Double.parseDouble(args[1]);
    test(ExtendedOperation.class, x, y);
  }
  
  private static <T extends Enum<T> & Operation> void test(
  Class<T> opEnumType, double x, double y) {
    for(Operation op : opEnumType.getEnumConstants())
      System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x,y));
  }
  ```

  ​	注意扩展过的操作类型的类的字面文字（ExtendedOperation.class）从main被传递给了test方法，来描述被扩展操作的集合。这个类的字面文字充当有限制的类型令牌(bounded type token)（E33）。opEnumType**参数中公认很复杂的声明(`<T extends Enum<T> & operation> C1ass<T>`)确保了Class对象既表示枚举又表示Operation的子类型，这正是遍历元素和执行与每个元素相关联的操作时所需要的**。

  ​	第二种方法是传入一个`Collection<? Extends Operation>`，这是个有限制的通配符类型(bounded wildcard type)（E31），而不是传递一个类对象：

  ```java
  public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
  	double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
  }
  
  private static void test(Collection<? extends Operation> opSet, double x, double y) {
    for (Operation op : opSet)
      System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
  }
  ```

  ​	这样得到的代码没有那么复杂，test方法也比较灵活一些：<u>它允许调用者将多个实现类型的操作合并到一起。另一方面，也放弃了在指定操作上使用 EnumSet（E36）和 EnumMap（E37）的功能</u>。

  ​	上面这两段程序运行时带上命令行参数4和2，都会产生如下输出：

  ```shell
  4.000000 ^ 2.000000 = 16.000000
  4.000000 % 2.000000 = 0.000000
  ```

  ​	<u>用接口模拟可伸缩枚举有个小小的不足，即无法将实现从一个枚举类型继承到另一个枚举类型</u>。如果实现代码不依赖于任何状态，就可以将缺省实现（E20）放在接口中。在上述 Operation的示例中，保存和获取与某项操作相关联的符号的逻辑代码，必须复制到BasicOperation和 ExtendedOperation中。在这个例子中是可以的，因为复制的代码非常少。如果共享功能比较多，则可以将它封装在一个辅助类或者静态辅助方法中来避免代码的复制工作。

  ​	本条目所述的模式也在Java类库中得到了应用。例如，`java.nio.file.LinkOption`枚举类型，它同时实现了CopyOption和 OpenOption接口。

---

+ 小结

  ​	总而言之，**虽然无法编写可扩展的枚举类型，却可以通过编写接口以及实现该接口的基础枚举类型来对它进行模拟**。这样允许客户端编写自己的枚举(或者其他类型)来实现接口。如果API是根据接口编写的，那么在可以使用基础枚举类型的任何地方，也都可以使用这些枚举。

## E39 注解优先于命名模式

+ 概述

  ​	**根据经验，一般使用命名模式（naming pattern）表明有些程序元素需要通过某种工具或者框架进行特殊处理**。

  ​	例如，在Java4发行版本之前，Jnit测试框架原本要求其用户定要用test作为测试方法名称的开头[Beck04]。**这种方法可行，但是有几个很严重的缺点**。

  + <u>文字拼写错误会导致失败，且没有任何提示</u>。

    例如，假设不小心将一个测试方法命名为 tsetSafetyOverride而不是 testSafetyOverride。 Junit3不会提示，但也不会执行测试，造成错误的安全感。

  + <u>无法确保它们只用于相应的程序元素上</u>。

    例如，假设将某个类称作 TestSafetyMechanisms，是希望JUnt3会自动地测试它所有的方法，而不管它们叫什么名称。 Junit3还是不会提示，但也同样不会执行测试。

  + <u>它们没有提供将参数值与程序元素关联起来的好方法</u>。

    例如，假设想要支持一种测试类别，它只在抛出特殊异常时才会成功。异常类型本质上是测试的一个参数。你可以利用某种具体的命名模式，将异常类型名称编码到测试方法名称中，但是这样的代码很不雅观，也很脆弱（E62）。编译器不知道要去检验准备命名异常的字符串是否真正命名成功。如果命名的类不存在，或者不是一个异常，你也要到试着运行测试时才会发现。

  ​	注解[JLS，9.7]很好地解决了所有这些问题，Jnit从Java4开始使用。<u>在本条目中，我们要编写自己的试验测试框架，展示一下注解的使用方法</u>。假设想要定义一个注解类型来指定简单的测试，它们自动运行，并在抛出异常时失败。以下就是这样的一个注解类型，命名为Test：

  ```java
  // Marker annotation type declaration
  import java.lang.annotation.*;
  /**
   * Indicates that the annotated method is a test method.
   * Use only on parameterless static methods.
   */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElememtType.METHOD)
  public @interface Test {
  }
  ```

  ​	Test注解类型的声明就是它自身通过@Retention和@Target注解进行了注解。

  ​	**注解类型声明中的这种注解被称作元注解(meta-annotation)**。

  + `@Retention(RetentionPolicy.RUNTIME)`元注解表明Test注解在运行时也应该存在，否则测试工具就无法知道Test注解。
  + `@Target(ElementType. METHOD)`元注解表明，Test注解只在方法声明中才是合法的：它不能运用到类声明、域声明或者其他程序元素上。

  ​	注意Test注解声明上方的注释："Use only on parameterless static method"(只用于无参的静态方法)。如果编译器能够强制这一限制最好，但是它做不到，除非编写一个**注解处理器(annotation processor)**，让它来完成。关于这个主题的更多信息，请参阅 Javax.annotation.processing的文档。在没有这类注解处理器的情况下，如果将Test注解放在实例方法的声明中，或者放在带有一个或者多个参数的方法中，测试程序还是可以编译，让测试工具在运行时来处理这个问题。

  ​	下面就是现实应用中的Test注解，称作**标记注解(marker annotation)**，因为它没有参数，只是"标注"被注解的元素。如果程序员拼错了Test，或者将Test注解应用到程序元素而非方法声明，程序就无法编译：

  ```java
  // Program containing marker annotations
  public class Sample {
    @Test
    public static void m1() { } // Test should pass
    public static void m2() { }
    @Test
    public static void m3() { // Test should fail
      throw new RuntimeException("Boom");
    }
    public static void m4() { }
    @Test
    public void m5() { } // INVALID USE: nonstatic method
    public static void m6() { }
    @Test
    public static void m7() { // Test should fail
      throw new RuntimeException("Cash");
    }
    public static void m8() { }
  }
  ```

  ​	Sample类有7个静态方法，其中4个被注解为测试。这4个中有2个抛出了异常m3和m7，另外两个则没有：m1和m5。但是其中一个没有抛出异常的被注解方法：m5，是个实例方法，因此不属于注解的有效使用。总之，Sample包含4项测试：一项会通过，两项会失败，另一项无效。没有用Test注解进行标注的另外4个方法会被测试工具忽略。

  ​	Test注解对 Sample类的语义没有直接的影响。它们只负责提供信息供相关的程序使用。更一般地讲，注解永远不会改变被注解代码的语义，但是使它可以通过工具进行特殊的处理，例如像这种简单的测试运行类：

  ```java
  // Program to process marker annotations
  import java.lang.reflect.*;
  
  public class RunTests {
    public static void main(String[] args) throws Exception {
      int tests = 0;
      int passed = 0;
      Class<?> testClass = Class.forName(args[0]);
      for (Method m : testClass.getDeclaredMethods()) {
        if(m.isAnnotationPresent(Test.class)) {
          tests++;
          try {
            m.invoke(null);
            passed++;
          } catch (InvocationTargetException wrappedExc) {
            Throwable exc = wrappedExc.getCause();
            System.out.println(m + " failed: " + exc);
          } catch (Exception exc) {
            System.out.println("Invalid @Test: " + m);
          }
        }
      }
      System.out.printf("Passed: %d, Failed: %d%n", passed, tests - passed);
    }
  }
  ```

  ​	测试运行工具在命令行上使用完全匹配的类名，并通过调用 Method.invoke反射式地运行类中所有标注了Test注解的方法。 isAnnotationPresent方法告知该工具要运行哪些方法。如果测试方法抛出异常，反射机制就会将它封装在InvocationtTargetException中。该工具捕捉到这个异常，并打印失败报告，包含测试方法抛出的原始异常，这些信息是通过getCause方法从InvocationTargetException中提取出来的。

  ​	<u>如果尝试通过反射调用测试方法时抛出InvocationTargetException之外的任何异常，表明编译时没有捕捉到Test注解的无效用法。这种用法包括实例方法的注解，或者带有一个或多个参数的方法的注解，或者不可访问的方法的注解。测试运行类中的第二个catch块捕捉到这些Test用法错误，并打印出相应的错误消息</u>。下面就是 RunTests在Sample上运行时打印的输出:

  ```shell
  public static void Sample.m3() failed: RuntimeException: Boom Invalid @Test: public void Sample.m5()
  public static void Sample.m7() failed: RuntimeException: Crash Passed : 1, Failed: 3
  ```

  ​	现在我们要针对只在抛出特殊异常时才成功的测试添加支持。为此需要一个新的注解类型：

  ```java
  // Annotation type with a parameter
  import java.lang.annotation.*;
  /**
  * Indicates that the annotated method is a test method that must throw the designated exception to succeed.
  */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTest {
    Class<? extends Throwable> value();
  }
  ```

  ​	<u>这个注解的参数类型是`Class<? extends Throwable>`。这个通配符类型有些绕口。它在英语中的意思是：某个扩展 Throwable的类的Class对象，它允许注解的用户指定任何异常(或错误)类型</u>。这种用法是有限制的类型令牌(bounded type token)（E33）的一个示例。下面就是实际应用中的这个注解。注意类名称被用作了注解参数的值：

  ```java
  // Program containing annotations with a parameter
  public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() { // Test should pass
      int i = 0;
      i = i / i;
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m2() { // Should fail (wrong exception)
      int[] a = new int[0];
      int i = a[i]; 
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m3() { } // Should fail(no exception)
  }
  ```

  ​	现在我们要修改一下测试运行工具来处理新的注解。这其中包括将以下代码添加到main方法中：

  ```java
  if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try {
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (InvocationTargetException wrappedEx) {
      Class<? extends Throwable> excType = 
        m.getAnnotation(ExceptionTest.class).value();
      if(exType.isInstance(exc)) {
        passed++;
      } else {
        System.out.printf("Test %s failed: expected %s, got %s%n", m, excType.getName(), exc);
      }
    } catch (Exception exc) {
      System.out.println("Invalid @Test: " + m);
    }
  }
  ```

  ​	这段代码类似于用来处理Test注解的代码，但有一处不同：这段代码提取了注解参数的值，并用它检验该测试抛出的异常是否为正确的类型。没有显式的转换，因此没有出现ClassCastException的危险。编译过的测试程序确保它的注解参数表示的是有效的异常类型，<u>需要提醒一点：有可能注解参数在编译时是有效的，但是表示特定异常类型的类文件在运行时却不存在。在这种希望很少出现的情况下，测试运行类会抛出 TypeNotPresentException异常</u>。

  ​	将上面的异常测试示例再深入一点，想象测试可以在抛出任何一种指定异常时都能够通过。注解机制有一种工具，使得支持这种用法变得十分容易。假设我们将ExceptionTest注解的参数类型改成Class对象的一个数组：

  ```java
  // Annotation type with an array parameter
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTest {
    Class<? extends Exception>[] value();
  }
  ```

  ​	注解中数组参数的语法十分灵活。它是进行过优化的单元素数组。使用了ExceptionTest新版的数组参数之后，<u>之前的所有 ExceptionTest注解仍然有效，并产生单元素的数组</u>。为了指定多元素的数组，要用花括号将元素包围起来，并用逗号将它们隔开：

  ```java
  // Code containing an annotation with an array parameter
  @ExceptionTest({ IndexOutOfBoundsException.class, NullPointerException.class })
  public static void doublyBad() {
    List<String> list = new ArrayList<>();
    
    // The spec permits this method to throw either
    // IndexOutOfBoundsException or NullPointerException
    list.addAll(5, null);
  }
  ```

  ​	修改测试运行工具来处理新的ExceptionTest相当简单。下面的代码代替了原本的代码：

  ```java
  if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try{
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
      Throwable exc = wrappedExc.getCause();
      int oldPassed = passed;
      Class<? extends Exception>[] excTypes = m.getAnnotation(ExceptionTest.class).value();
      for (Class<? extends Exception> excType : excTypes) {
        if (excType.isInstance(exc)) {
          passed++;
          break;
        }
      }
      if (passed == oldPassed)
        System.out.printf("Test %s failed: %s %s", m, exc);
    }
  }
  ```

  ​	<u>从Java8开始，还有另一种方法可以进行多值注解。它不是用一个数组参数声明一个注解类型，而是用@Repeatable元注解对注解的声明进行注解，表示该注解可以被重复地应用给单个元素。这个元注解只有一个参数，就是包含注解类型(containing annotation type)的类对象，它唯一的参数是一个注解类型数组[JLS，9.6.3]</u>。下面的注解声明就是把ExceptionTest注解改成使用这个方法之后的版本。注意包含的注解类型必须利用适当的保留策略和目标进行注解，否则声明将无法编译：

  ```java
  // Repeatable annotation type
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  @Repeatable(ExceptionTestContainer.class)
  public @interface ExceptionTest {
    Class<? extends Exception> value();
  }
  
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTestContainer {
    ExceptionTest[] value();
  }
  ```

  ​	下面是doublyBad测试方法用重复注解代替数组值注解之后的代码：

  ```java
  // Code containing a repeated annotation
  @RetentionTest(IndexOutOfBoundException.class)
  @RetentionTest(NullPonterException.class)
  public static void doublyBad() { ... }
  ```

  ​	**处理可重复的注解要非常小心。重复的注解会产生一个包含注解类型的合成注解**。

  ​	<u>getAnnotationsByType方法掩盖了这个事实，可以用于访问可重复注解类型的重复和非重复的注解</u>。<u>但isAnnotationPresent使它变成了显式的，即重复的注解不是注解类型(而是所包含的注解类型)的一部分</u>。

  ​	<u>如果一个元素具有某种类型的重复注解，并且用isAnnotationPresent方法检验该元素是否具有该类型的注解，会发现它没有。用这种方法检验是否存在注解类型，会导致程序默默地忽略掉重复的注解。同样地，用这种方法检验是否存在包含的注解类型，会导致程序默默地忽略掉非重复的注解</u>。

  ​	为了利用isAnnotationPresent检测重复和非重复的注解，必须检查注解类型及其包含的注解类型。下面是 Runtests程序改成使用ExceptionTest注解时有关部分的代码：

  ```java
  // Processing repeatable annotations
  if (m.isAnnotationPresent(ExceptionTest.class)
     || m.isAnnotationPresent(ExceptionTestContainer.class)) {
    tests++;
    try {
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
      Throwable exc = wrappedExc.getCause();
      int oldPassed = passed;
      ExceptionTest[] excTests = m.getAnnotationsByType(ExceptionTest.class);
      for (ExceptionTest excTest : excTests) {
        if  excTest.value().isInstance(exc)) {
          passed++;
          break;
        }
      }
      if(passed == oldPassed)
        System.out.printf("Test %s failed: %s %n", m, exc);
    }
  }
  ```

  ​	加入可重复的注解，提升了源代码的可读性，逻辑上是将同一个注解类型的多个实例应用到了一个指定的程序元素。如果你觉得它们增强了源代码的可读性就使用它们，但是记住<u>在声明和处理可重复注解的代码中会有更多的样板代码，并且处理可重复的注解容易出错</u>。

  ​	本条目中的测试框架只是一个试验，但它清楚地示范了注解之于命名模式的优越性这只是揭开了注解功能的冰山一角。如果是在编写一个需要程序员给源文件添加信息的工具，就要定义一组适当的注解类型。**既然有了注解，就完全没有理由再使用命名模式了**。

  ​	也就是说，除了"工具铁匠"( toolsmiths，即平台框架程序员)之外，大多数程序员都不必定义注解类型。但是**所有的程序员都应该使用Java平台所提供的预定义的注解类型**（E40和E27）。还要考虑使用IDE或者静态分析工具所提供的任何注解。这种注解可以提升由这些工具所提供的诊断信息的质量。但是要注意这些注解还没有标准化，因此如果变换工具或者形成标准，就有很多工作要做了。

## E40 坚持使用Override注解

P157

